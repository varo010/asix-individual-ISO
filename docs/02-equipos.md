# T2 · Mis equipos: clase y casa

RA: FDM RA1. Inventario de los dos equipos, comparativa y cuellos de botella.

## Lo que pide la rúbrica

- [x] Inventariar el equipo de clase y el de casa.
- [x] Compararlos e identificar cuellos de botella.
- [x] Proponer soluciones ajustadas al hardware disponible.
- [ ] Ampliación: tests de memoria, disco y red en los dos equipos (casa: hecho en parte; clase: pendiente).
- [x] Ampliación: registrar herramienta, condiciones y resultado.
- [ ] Ampliación: confirmar o descartar con datos los cuellos de botella previstos.

## 0. Punto de partida: un solo disco, dos equipos

Mi sistema (Linux Mint 22.3 con VirtualBox y todas las VM) está en un **SSD Kingston A400 de 480 GB dentro de una caja USB** (puente SATA→USB JMicron JMS578). Ese mismo disco lo conecto al portátil de clase y al de casa. Por eso:

- El **disco, el sistema operativo y las VM son los mismos** en los dos equipos.
- Lo que cambia es el **anfitrión**: CPU, RAM, red, firmware y puerto USB.
- Ningún equipo usa su disco interno: en clase el NVMe tiene otros sistemas y en casa el NVMe tiene Windows cifrado con BitLocker. No se tocan.

## 1. Inventario y comparativa

N/C = no comprobado todavía.

| Dato | Equipo de clase | Equipo de casa | Cómo se comprueba |
|---|---|---|---|
| Modelo | Portátil ASUS ExpertBook B3605CCA | Portátil Lenovo Legion Pro 5 16IRX10 (83NN) | `/sys/class/dmi/id/` |
| Firmware | UEFI, BIOS B3605CCA.303 (06/11/2025) | UEFI, BIOS S9CN15WW (23/07/2025) | `/sys/class/dmi/id/bios_*` |
| Secure Boot | **Desactivado** | **Activado** | `mokutil --sb-state` |
| CPU | Intel Core Ultra 5 125H: 14 núcleos, 18 hilos, hasta 4,5 GHz, 18 MiB L3 | Intel Core i9-14900HX: 24 núcleos, 32 hilos, hasta 5,8 GHz, 36 MiB L3 | `lscpu` |
| Virtualización | VT-x, módulo `kvm_intel` | VT-x, módulo `kvm_intel` | `lscpu`, `lsmod` |
| RAM | 30 GiB visibles | 31 GiB visibles | `free -h` |
| Módulos de RAM y ampliación | N/C | N/C | `sudo dmidecode -t memory` |
| Disco del sistema y de las VM | El SSD USB (puerto USB a 5 Gb/s, driver UAS) | El mismo SSD USB (puerto USB a 5 Gb/s, driver UAS) | `lsblk -o NAME,TRAN,MODEL`, `lsusb -t` |
| Disco interno | NVMe Samsung 1 TB (otros sistemas, no se usa) | NVMe Samsung 1 TB (Windows con BitLocker, no se usa) | `lsblk` |
| Espacio libre para VM | ~227 GB en `/home` (el mismo disco) | ~227 GB en `/home` (el mismo disco) | `df -h` |
| Red cableada | Intel Ethernet, **conectada a 1000 Mb/s** (red del aula) | Realtek RTL8111 Gigabit, **sin cable** | `ip -br a`, `/sys/class/net/*/speed` |
| Wi-Fi | Realtek RTL8852BE (Wi-Fi 6), sin usar | Intel CNVi (Wi-Fi 6), **en uso**: canal 44 (5 GHz), enlace 270 Mb/s, señal 50/100 | `lspci`, `nmcli dev wifi` |
| Gráficos | Intel Arc integrada | Intel UHD integrada + NVIDIA dedicada | `lspci` |
| Batería | N/C | 74,2 Wh de 80 Wh de diseño (93 % de salud) | `/sys/class/power_supply/BAT*` |
| Sistema operativo | Linux Mint 22.3, kernel 6.8 (el del disco USB) | Igual | `/etc/os-release`, `uname -r` |
| VirtualBox | 7.2.18 + Extension Pack, **funciona** | 7.2.18 + Extension Pack, **no arranca VM** (ver incidencia) | `VBoxManage --version` |

Evidencias: [inventario de clase](../evidencias/t2-inventario-clase-20261112.txt) · [inventario de casa](../evidencias/t2-inventario-casa-20261115.txt).

### Incidencia encontrada: VirtualBox no funciona en casa

Con Secure Boot activado, el firmware solo deja cargar módulos del kernel firmados con una clave que conozca. Los módulos de VirtualBox (`vboxdrv`…) están firmados con una clave propia (MOK) que se creó en el disco, pero esa clave **solo está registrada en el firmware del portátil de clase**. El registro de claves se guarda en cada placa base, no en el disco. Detalle y solución en [INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md).

## 2. Cuellos de botella previstos

| Equipo | Recurso | Por qué podría limitar | Cómo lo compruebo | Solución posible |
|---|---|---|---|---|
| Los dos | **Disco (SSD por USB)** | Todas las VM leen y escriben en el mismo SSD, que va detrás de un puente USB de 5 Gb/s. Con varias VM a la vez los accesos son pequeños y aleatorios | `dd` secuencial (hecho) y `fio` aleatorio 4K (pendiente), y `iostat` con VM en marcha (T7) | No arrancar todas las VM a la vez; discos dinámicos; instantáneas en vez de copias completas; en casa, conectar a un puerto USB de 10 Gb/s no ayuda porque el puente es de 5 Gb/s |
| Los dos | **RAM** | Las VM previstas suman unos 30 GB ([00-entorno](00-entorno.md)) y los equipos tienen 30–31 GiB, de los que el anfitrión necesita unos 4 GiB | Suma de la tabla de VM y `free -h` con VM arrancadas (T7) | Escenarios de trabajo con solo las VM necesarias; un solo router a la vez; ajustar la RAM de cada VM a lo medido |
| Clase | CPU | 18 hilos para repartir entre las VM y el anfitrión | `openssl speed` y `top` con VM en marcha (T7) | Asignar 1–2 vCPU por VM, nunca más hilos que los del equipo |
| Casa | **CPU con batería** | El i9 HX baja mucho su rendimiento sin cargador para ahorrar energía y no calentarse | Dos pruebas seguidas con batería: 30,0 GB/s y 18,3 GB/s (ver apartado 3) | Trabajar con el cargador conectado y el perfil de rendimiento |
| Casa | Red | Solo hay Wi-Fi (la tarjeta de cable no tiene cable). Afecta a descargas de ISOs y actualizaciones, no al tráfico entre VM (la red interna va por memoria) | `ping` y descarga de prueba (hecho) | Cable de red al router de casa para las descargas grandes |

## 3. Ampliación: tests de diagnóstico

Herramienta:, igual en los dos equipos. Usa `openssl speed`, `dd`, `ping` y `curl`; con `sudo` y las herramientas instaladas añade `smartctl`, `fio` y `memtester`.

| Equipo | Recurso | Herramienta | Condiciones | Resultado | Interpretación | Evidencia |
|---|---|---|---|---|---|---|
| Casa | CPU (1 hilo) | `openssl speed sha256` | 15/11, con batería, gobernador `powersave`, sin VM, 4 min encendido | ~1,05 GB/s | Rendimiento de un solo núcleo | [diagnóstico](../evidencias/t2-diagnostico-83NN-20261115.txt) |
| Casa | CPU (32 hilos) | `openssl speed -multi 32 sha256` | Igual; dos pruebas seguidas | 30,0 GB/s la primera y **18,3 GB/s la segunda** (CPU a 69 °C) | Con batería, el equipo recorta potencia en cuanto la carga dura un poco: un 39 % menos. Hay que repetirlo con cargador | ídem |
| Casa | Memoria | `dd /dev/zero → /dev/null` (20 GiB) | Igual | ~31 GB/s; 27 GiB disponibles sin VM | La memoria no limita la velocidad; limita la **cantidad** | ídem |
| Casa | Disco (secuencial) | `dd` 2 GiB sin caché | Igual | Escritura 389 MB/s · lectura 410 MB/s | Cerca del máximo práctico de USB 3.0 (5 Gb/s); el SSD por SATA llegaría a unos 500 MB/s en lectura (ficha del fabricante): el puente USB recorta un poco | ídem |
| Casa | Red (local) | `ping -c 10` a la puerta de enlace | Wi-Fi 5 GHz, señal 50/100 | 0 % de pérdida, media 6 ms (máx. 24 ms) | Normal en Wi-Fi; en cable sería menos de 1 ms | ídem |
| Casa | Red (Internet) | `ping` a 1.1.1.1 y descarga de 100 Mb | Igual | 0 % de pérdida, media 31 ms; descarga 12,2 MB/s (~98 Mb/s) | Suficiente para descargar ISOs (una de 6 GB, unos 8–9 min) | ídem |
| Casa | Disco (aleatorio 4K), SMART y memtester | `fio`, `smartctl`, `memtester` | Pendiente: hacen falta `sudo apt install fio smartmontools memtester` y `sudo` | — | — | — |

## 4. Conclusión (provisional)

- El **cuello de botella común** es el **disco USB**, porque es el mismo en los dos equipos y todas las VM dependen de él. Falta medir el acceso aleatorio con `fio`, que es el que más sufre con varias VM.
- La **RAM** es lo que limita **cuántas VM** arrancan a la vez, igual en los dos equipos (30–31 GiB).
- El equipo de casa tiene casi el doble de CPU, pero solo lo aprovecha con el cargador conectado.
- El entorno vive en el disco USB y funciona en los dos equipos, una vez resuelta la incidencia de Secure Boot en casa.

## 5. Comparación con el inventario del grupo

| Aspecto | Equipos del grupo (aula) | Mis equipos |
|---|---|---|
| Equipo con las VM | PC-01: i3-4160 (2 núcleos/4 hilos), 8 GB DDR3, HDD de 500 GB | Portátiles con 18 y 32 hilos, 30–31 GiB, SSD por USB |
| Cuello de botella que encontró el grupo | HDD con accesos aleatorios (107 IOPS en el PC-01) y red Gigabit con ficheros grandes | Disco USB compartido por todas las VM y RAM para el número de VM |
| Disco, lectura secuencial | HDD del PC-01: 181 MiB/s | SSD USB: 410 MB/s (~391 MiB/s) |
| Herramientas | `fio`, `smartctl`, `openssl speed`, `ethtool`, `iperf3`, `top` | `openssl speed`, `dd`, `ping`, `curl` (+ `fio`, `smartctl` y `memtester` pendientes) |
