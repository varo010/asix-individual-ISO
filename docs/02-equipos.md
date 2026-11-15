# T2 · Mis equipos: clase y casa

RA: FDM RA1. Inventario de los dos equipos, comparativa y cuellos de botella.

## Lo que pide la rúbrica

- [x] Inventariar el equipo de clase y el de casa.
- [x] Compararlos e identificar cuellos de botella.
- [x] Proponer soluciones ajustadas al hardware disponible.
- [ ] Ampliación: tests de memoria, disco y red en los dos equipos (casa: hecho; clase: pendiente).
- [x] Ampliación: registrar herramienta, condiciones y resultado.
- [ ] Ampliación: confirmar o descartar con datos los cuellos de botella previstos (casa: hecho; falta clase).

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

Herramienta:, igual en los dos equipos. Usa `openssl speed`, `dd`, `ping` y `curl`; con `sudo` añade `smartctl`, `fio` y `memtester` (instalados con `sudo apt install fio smartmontools memtester sysstat`).

| Equipo | Recurso | Herramienta | Condiciones | Resultado | Interpretación | Evidencia |
|---|---|---|---|---|---|---|
| Casa | CPU (1 hilo) | `openssl speed sha256` (bloques de 16 KB) | 15/11 · **con batería** · `powersave` · sin VM | 1,05 GB/s | Referencia con batería | [batería](../evidencias/t2-diagnostico-83NN-bateria-20261115.txt) |
| Casa | CPU (32 hilos) | `openssl speed -multi 32 sha256` | Igual; dos pruebas seguidas | 30,0 GB/s y después **18,3 GB/s** (CPU a 69 °C) | Con batería recorta potencia en cuanto la carga dura: un 39 % menos de una prueba a la siguiente | ídem |
| Casa | CPU (1 hilo) | `openssl speed sha256` | 15/11 · **con cargador** · sin VM | **1,82 GB/s** | Un 73 % más que con batería | [cargador](../evidencias/t2-diagnostico-83NN-20261115.txt) |
| Casa | CPU (32 hilos) | `openssl speed -multi 32 sha256` | Igual | 34,8 GB/s (pico de 46,9 GB/s con bloques de 8 KB) | Confirma que con batería el equipo no da todo su rendimiento | ídem |
| Casa | Memoria (velocidad) | `dd /dev/zero → /dev/null` (20 GiB) | Con batería / con cargador | 31 GB/s / **54 GB/s** | La memoria no limita la velocidad; limita la **cantidad** (27 GiB libres sin VM) | ídem |
| Casa | Memoria (errores) | `memtester 1024M 1` | Con cargador, 1 GiB, 1 pasada | Todas las pruebas `ok` | Sin errores en el GiB probado. No sustituye un test completo de arranque (MemTest86), que revisaría toda la RAM | ídem |
| Casa | Disco (secuencial) | `dd` 2 GiB sin caché (`oflag/iflag=direct`) | Con cargador | Escritura 362 MB/s · lectura 403 MB/s | Cerca del máximo práctico de USB 3.0 (5 Gb/s); el SSD por SATA directo llegaría a unos 500 MB/s en lectura (ficha de Kingston) | ídem |
| Casa | Disco (aleatorio) | `fio` lectura aleatoria 4 KiB, profundidad 32, 20 s | Con cargador | **41.200 IOPS** (161 MiB/s), latencia media 0,77 ms | Bien para varias VM a la vez. Comparación: el HDD del PC-01 del grupo hace 107 IOPS | ídem |
| Casa | Disco (salud) | `smartctl -d sat -H -A` | Con cargador | PASSED; 1572 h encendido; 483 encendidos; 1834 GiB escritos; 0 sectores reasignados; 26 °C. **Errores CRC: atributo 199 = 262185 (valor bruto) y 218 = 177** | El SSD está sano, pero hay **errores de transmisión** en el enlace SATA entre el SSD y el adaptador USB de la caja. Ver apartado 3.1 | ídem |
| Casa | Red (local) | `ping -c 10` a la puerta de enlace | Wi-Fi 5 GHz, señal 50/100 | 0 % de pérdida, media 4,6–6 ms | Normal en Wi-Fi; en cable sería menos de 1 ms | ídem |
| Casa | Red (Internet) | `ping` a 1.1.1.1 y descarga de 100 Mb | Igual | 0 % de pérdida, 14–31 ms; descarga entre 6,4 y 12,2 MB/s en dos pruebas | El Wi-Fi varía mucho de una prueba a otra; para descargar ISOs, mejor cable | ídem |

### 3.1 Posible avería: errores CRC en el disco USB

- **Qué indica:** un error CRC es un bloque que llega dañado por el cable y se tiene que volver a enviar. No se pierden datos (se reenvía), pero el disco va más lento mientras pasa. El SSD en sí está sano: 0 sectores reasignados y 0 errores incorregibles.
- **Causa probable:** el enlace SATA **dentro de la caja USB**: el conector entre el SSD y la placa adaptadora (JMicron JMS578), o el propio adaptador. Que el disco se desconecte de un portátil y se conecte a otro muchas veces (483 encendidos) favorece los falsos contactos.
- **Otra causa posible:** que el contador venga de antes (el disco tiene 1572 h de uso) y ya no esté subiendo.
- **Prueba que las distingue:** anotar los dos contadores, hacer una prueba de disco larga (por ejemplo `fio` 5 minutos) y volver a leerlos. Si **suben**, el problema es actual y está en la caja o el conector. Si **no cambian**, son errores antiguos.
- **Solución si suben:** volver a encajar el SSD en la caja o cambiar la caja USB (son baratas); en un equipo de sobremesa, cambiar el cable SATA.

## 4. Conclusión (provisional)

- El **cuello de botella común** es el **disco USB**, porque es el mismo en los dos equipos y todas las VM dependen de él: unos 400 MB/s en secuencial. En aleatorio rinde bien (41.200 IOPS), así que se **descarta** que sea un problema grave con unas pocas VM. Se comprobará con carga real en la T7.
- **Riesgo encontrado:** errores CRC en el enlace SATA de la caja USB (apartado 3.1). Hay que vigilarlos, porque todo el entorno está en ese disco.
- La **RAM** es lo que limita **cuántas VM** arrancan a la vez, igual en los dos equipos (30–31 GiB).
- **Confirmado:** el equipo de casa tiene casi el doble de CPU, pero solo lo aprovecha con el cargador conectado (+73 % en un núcleo).
- El entorno vive en el disco USB y funciona en los dos equipos, una vez resuelta la incidencia de Secure Boot en casa.

## 5. Comparación con el inventario del grupo

| Aspecto | Equipos del grupo (aula) | Mis equipos |
|---|---|---|
| Equipo con las VM | PC-01: i3-4160 (2 núcleos/4 hilos), 8 GB DDR3, HDD de 500 GB | Portátiles con 18 y 32 hilos, 30–31 GiB, SSD por USB |
| Cuello de botella que encontró el grupo | HDD con accesos aleatorios (107 IOPS en el PC-01) y red Gigabit con ficheros grandes | Disco USB compartido por todas las VM y RAM para el número de VM |
| Disco, lectura secuencial | HDD del PC-01: 181 MiB/s | SSD USB: 410 MB/s (~391 MiB/s) |
| Herramientas | `fio`, `smartctl`, `openssl speed`, `ethtool`, `iperf3`, `top` | `openssl speed`, `dd`, `fio`, `smartctl`, `memtester`, `ping`, `curl` |
| Disco, lectura aleatoria 4K | HDD del PC-01: 107 IOPS | SSD USB: 41.200 IOPS |
| Errores CRC | HDD del PC-01: 1 error | SSD USB: 177 (atributo 218), a vigilar |
