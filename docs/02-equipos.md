# T2 · Mis equipos: clase y casa

RA: FDM RA1. Inventario de los dos equipos, comparativa y cuellos de botella.

## Lo que pide la rúbrica

- [x] Inventariar el equipo de clase y el de casa.
- [x] Compararlos e identificar cuellos de botella.
- [x] Proponer soluciones ajustadas al hardware disponible.
- [x] Ampliación: tests de memoria, disco y red en los dos equipos.
- [x] Ampliación: registrar herramienta, condiciones y resultado.
- [x] Ampliación: confirmar o descartar con datos los cuellos de botella previstos.

## 0. Punto de partida: un solo disco, dos equipos

Mi sistema (Linux Mint 22.3 con VirtualBox y todas las VM) está en un **SSD Kingston A400 de 480 GB dentro de una caja USB** (la caja lleva una pequeña placa que pasa de SATA, la conexión del SSD, a USB). Ese mismo disco lo conecto al portátil de clase y al de casa. Por eso:

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
| Virtualización | Sí (Intel VT-x activado) | Sí (Intel VT-x activado) | `lscpu`, `lsmod` |
| RAM | 30 GiB visibles | 31 GiB visibles | `free -h` |
| Módulos de RAM y ampliación | N/C | N/C | `sudo dmidecode -t memory` |
| Disco del sistema y de las VM | El SSD USB, conectado a un USB 3.0 (5 Gb/s) | El mismo SSD USB, también a un USB 3.0 (5 Gb/s) | `lsblk -o NAME,TRAN,MODEL`, `lsusb -t` |
| Disco interno | NVMe Samsung 1 TB (otros sistemas, no se usa) | NVMe Samsung 1 TB (Windows con BitLocker, no se usa) | `lsblk` |
| Espacio libre para VM | ~227 GB en `/home` (el mismo disco) | ~227 GB en `/home` (el mismo disco) | `df -h` |
| Red cableada | Intel Ethernet, **conectada a 1000 Mb/s** (red del aula) | Realtek RTL8111 Gigabit, **sin cable** | `ip -br a`, `/sys/class/net/*/speed` |
| Wi-Fi | Realtek RTL8852BE (Wi-Fi 6), sin usar | Intel Wi-Fi 6, **en uso**: canal 44 (5 GHz), enlace 270 Mb/s, señal 50/100 | `lspci`, `nmcli dev wifi` |
| Gráficos | Intel Arc integrada | Intel UHD integrada + NVIDIA dedicada | `lspci` |
| Batería | ≈ 47 Wh de 50 Wh cuando era nueva (94 % de salud), 14 ciclos de carga | 74,2 Wh de 80 Wh cuando era nueva (93 % de salud) | `/sys/class/power_supply/BAT*` |
| Sistema operativo | Linux Mint 22.3, kernel 6.8 (el del disco USB) | Igual | `/etc/os-release`, `uname -r` |
| VirtualBox | 7.2.18 + Extension Pack, **funciona** (16/11: arrancan `r-mikrotik`, `pve1` con VT-x anidado y `srv-dc`) | 7.2.18 + Extension Pack, **no arranca VM** (ver incidencia) | `VBoxManage --version` |

Evidencias: [inventario de clase](../evidencias/t2-inventario-clase-20261112.txt) · [inventario de casa](../evidencias/t2-inventario-casa-20261115.txt).

### Incidencia encontrada: VirtualBox no funciona en casa

Con Secure Boot activado, el portátil solo deja cargar controladores firmados con una clave de confianza. Los controladores de VirtualBox están firmados con una clave que se creó al instalarlo, pero esa clave **solo la conoce el portátil de clase**: la lista de claves de confianza se guarda en la placa base de cada equipo, no en el disco. Detalle y solución en [INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md).

## 2. Cuellos de botella previstos

| Equipo | Recurso | Por qué podría limitar | Cómo lo compruebo | Solución posible |
|---|---|---|---|---|
| Los dos | **Disco (SSD por USB)** | Todas las VM leen y escriben en el mismo SSD, que va por USB (5 Gb/s como máximo). Con varias VM a la vez, el disco tiene que leer muchos trozos pequeños en sitios distintos | `dd` para ficheros grandes y `fio` para trozos pequeños (hechos); `iostat` con VM en marcha (T7) | No arrancar todas las VM a la vez; discos dinámicos; instantáneas en vez de copias completas; conectarlo a un USB más rápido (10 Gb/s) no ayuda, porque la caja solo llega a 5 Gb/s |
| Los dos | **RAM** | Las VM previstas suman unos 30 GB ([00-entorno](00-entorno.md)) y los equipos tienen 30–31 GiB, de los que el anfitrión necesita unos 4 GiB | Suma de la tabla de VM y `free -h` con VM arrancadas (T7) | Escenarios de trabajo con solo las VM necesarias; un solo router a la vez; ajustar la RAM de cada VM a lo medido |
| Clase | CPU | 18 hilos para repartir entre las VM y el anfitrión | `openssl speed` y `top` con VM en marcha (T7) | Asignar 1–2 vCPU por VM, nunca más hilos que los del equipo |
| Casa | **CPU con batería** | El i9 HX baja mucho su rendimiento sin cargador para ahorrar energía y no calentarse | Dos pruebas seguidas con batería: 30,0 GB/s y 18,3 GB/s (ver apartado 3) | Trabajar con el cargador conectado y el perfil de rendimiento |
| Casa | Red | Solo hay Wi-Fi (la tarjeta de cable no tiene cable). Afecta a descargas de ISOs y actualizaciones, no al tráfico entre VM (las VM se comunican dentro del propio portátil, sin pasar por el Wi-Fi) | `ping` y descarga de prueba (hecho) | Cable de red al router de casa para las descargas grandes |

## 3. Ampliación: tests de diagnóstico

Herramienta:, igual en los dos equipos. Usa `openssl speed`, `dd`, `ping` y `curl`; con `sudo` añade `smartctl`, `fio` y `memtester` (instalados con `sudo apt install fio smartmontools memtester sysstat`).

**Cómo leer los resultados:**

- **MB/s o GB/s:** cuántos datos se mueven por segundo. Más es mejor.
- **IOPS:** cuántas lecturas pequeñas hace el disco por segundo. Es lo que importa cuando varias VM usan el disco a la vez. Más es mejor.
- **ms (milisegundos):** cuánto tarda en responder. Menos es mejor.
- **CPU con `openssl speed`:** pone la CPU a calcular a tope y mide cuántos datos procesa por segundo. No es una medida «real», pero sirve para comparar dos CPU con la misma prueba.

| Equipo | Recurso | Herramienta | Condiciones | Resultado | Interpretación | Evidencia |
|---|---|---|---|---|---|---|
| Casa | CPU (1 hilo) | `openssl speed sha256` | 15/11 · **con batería** · modo ahorro de energía · sin VM | 1,05 GB/s | Referencia con batería | [batería](../evidencias/t2-diagnostico-83NN-bateria-20261115.txt) |
| Casa | CPU (32 hilos) | `openssl speed -multi 32 sha256` | Igual; dos pruebas seguidas | 30,0 GB/s y después **18,3 GB/s** (CPU a 69 °C) | Con batería, en cuanto el trabajo dura un poco, el portátil baja la potencia: un 39 % menos de una prueba a la siguiente | ídem |
| Casa | CPU (1 hilo) | `openssl speed sha256` | 15/11 · **con cargador** · sin VM | **1,82 GB/s** | Un 73 % más que con batería | [cargador](../evidencias/t2-diagnostico-83NN-20261115.txt) |
| Casa | CPU (32 hilos) | `openssl speed -multi 32 sha256` | Igual | 34,8 GB/s | Confirma que con batería el equipo no da todo su rendimiento | ídem |
| Casa | Memoria (velocidad) | `dd /dev/zero → /dev/null` (20 GiB) | Con batería / con cargador | 31 GB/s / **54 GB/s** | La velocidad de la memoria no es un problema; lo que limita es la **cantidad** (27 GiB libres sin VM) | ídem |
| Casa | Memoria (errores) | `memtester 1024M 1` | Con cargador, 1 GiB, 1 pasada | Todas las pruebas `ok` | Sin errores en el GiB probado. No revisa toda la RAM: para eso habría que arrancar un test completo como MemTest86 | ídem |
| Casa | Disco (ficheros grandes) | `dd` 2 GiB sin caché (así se mide el disco y no la RAM) | Con cargador | Escritura 362 MB/s · lectura 403 MB/s | Cerca del máximo real de un USB 3.0. El mismo SSD conectado por dentro (SATA) daría unos 500 MB/s, según Kingston | ídem |
| Casa | Disco (trozos pequeños) | `fio`: 20 s leyendo trozos pequeños (4 KiB) en sitios al azar | Con cargador | **41.200 IOPS**; cada lectura tarda 0,77 ms de media | Bien para varias VM a la vez. Comparación: el HDD del PC-01 del grupo hace 107 IOPS | ídem |
| Casa | Disco (salud) | `smartctl -d sat -H -A` | Con cargador | Estado bueno (PASSED); 1572 h encendido; 483 arranques; 1834 GiB escritos; ninguna zona dañada; 26 °C. **Dos contadores de errores de transmisión (CRC): 262185 y 177** | El SSD está sano, pero hay **errores de transmisión** en la conexión entre el SSD y la caja USB. Ver apartado 3.1 | ídem |
| Casa | Red (local) | `ping -c 10` a la puerta de enlace | Wi-Fi 5 GHz, señal 50/100 | 0 % de pérdida, media 4,6–6 ms | Normal en Wi-Fi; en cable sería menos de 1 ms | ídem |
| Casa | Red (Internet) | `ping` a 1.1.1.1 y descarga de 100 Mb | Igual | 0 % de pérdida, 14–31 ms; descarga entre 6,4 y 12,2 MB/s en dos pruebas | El Wi-Fi varía mucho de una prueba a otra; para descargar ISOs, mejor cable | ídem |
| Clase | CPU (1 hilo) | `openssl speed sha256` | 16/11 · con cargador · modo ahorro de energía · sin VM · **con una descarga de 7 GB en curso** | 1,91 GB/s | Igual que casa con cargador (1,82): en un solo núcleo los dos equipos rinden lo mismo | [clase](../evidencias/t2-diagnostico-ASUSEXPERTBOOKB3605CCA-20261116.txt) |
| Clase | CPU (18 hilos) | `openssl speed -multi 18 sha256` | Igual · **CPU a 98 °C** | **17,7 GB/s** | La mitad que casa (34,8): tiene menos hilos y, a 98 °C, la CPU ya baja su velocidad para no calentarse más. Confirma el cuello de botella previsto | ídem |
| Clase | Memoria (velocidad) | `dd /dev/zero → /dev/null` (20 GiB) | Igual | 45,4 GB/s | Algo menos que casa (54); tampoco limita | ídem |
| Clase | Memoria (errores) | `memtester 1024M 1` | Igual | Todas las pruebas `ok` | Sin errores en el GiB probado | ídem |
| Clase | Disco (ficheros grandes) | `dd` 2 GiB sin caché | Igual | Escritura 417 MB/s · lectura 419 MB/s | Casi igual que en casa (362/403): es el mismo disco y el límite es la caja USB, no el portátil | ídem |
| Clase | Disco (trozos pequeños) | `fio`: igual que en casa | Igual | **44.000 IOPS**; cada lectura tarda 0,72 ms de media | Igual que en casa (41.200): el disco rinde lo mismo en los dos equipos | ídem |
| Clase | Disco (salud) | `smartctl -d sat -H -A` | Igual · 1 día después de la prueba de casa | Estado bueno (PASSED); 1574 h; 486 arranques; 1861 GiB escritos; **contadores CRC: 262186 (+1) y 177 (igual)** | En un día con 3 arranques y 27 GiB escritos (las ISOs) el contador casi no se mueve: los errores CRC son **antiguos**. Ver apartado 3.1 | ídem |
| Clase | Red (local) | `ping -c 10` a la puerta de enlace | Cable a 1000 Mb/s | 0 % de pérdida, media **0,52 ms** | Diez veces menos que el Wi-Fi de casa (4,6–6 ms) | ídem |
| Clase | Red (Internet) | `ping` a 1.1.1.1 y descarga de 100 Mb | Igual, con la descarga en curso y la red del aula compartida | Ping: 100 % de pérdida. Descarga: **5,2 MB/s**, sin problemas | El ping no sale pero la web sí: la red del aula **bloquea el ping hacia Internet**, no es una avería. La descarga es más lenta que en casa porque la salida a Internet del aula se comparte | ídem |

### 3.1 Posible avería: errores CRC en el disco USB

- **Qué indica:** un error CRC es un bloque que llega dañado por el cable y se tiene que volver a enviar. No se pierden datos (se reenvía), pero el disco va más lento mientras pasa. El SSD en sí está sano: no tiene zonas dañadas ni errores de lectura.
- **Causa probable:** la conexión **dentro de la caja USB**: el conector entre el SSD y la placa de la caja, o la propia placa. Que el disco se desconecte de un portátil y se conecte a otro muchas veces (483 encendidos) favorece los falsos contactos.
- **Otra causa posible:** que el contador venga de antes (el disco tiene 1572 h de uso) y ya no esté subiendo.
- **Prueba que las distingue:** anotar los dos contadores, hacer una prueba de disco larga (por ejemplo `fio` 5 minutos) y volver a leerlos. Si **suben**, el problema es actual y está en la caja o el conector. Si **no cambian**, son errores antiguos.
- **Resultado (16/11, en clase):** un contador pasó de 262185 a 262186 (+1) y el otro sigue en 177, tras 3 arranques, 27 GiB escritos y todas las pruebas de disco. **Son errores antiguos**: la conexión funciona bien ahora. Se seguirá mirando en la T7.
- **Solución si suben:** volver a encajar el SSD en la caja o cambiar la caja USB (son baratas); en un equipo de sobremesa, cambiar el cable SATA.

## 4. Conclusión

- El **cuello de botella común** es el **disco USB**: todas las VM dependen de él y da lo mismo en los dos portátiles (unos 400 MB/s con ficheros grandes). Con muchos trozos pequeños también rinde bien (41.000–44.000 IOPS), así que con unas pocas VM no debería ser un problema grave. Se comprobará con VM en marcha en la T7.
- **Riesgo descartado por ahora:** los errores CRC del disco son antiguos; en un día casi no han subido (apartado 3.1). Se vigilan, porque todo el entorno está en ese disco.
- La **RAM** es lo que limita **cuántas VM** arrancan a la vez, igual en los dos equipos (30–31 GiB).
- **Confirmado:** el equipo de casa tiene el doble de CPU con todos los hilos (34,8 frente a 17,7 GB/s), pero solo lo aprovecha con el cargador (+73 % en un núcleo). En un núcleo los dos rinden igual. El de clase se calienta mucho (98 °C): para tareas largas, mejor no tener otras cargas a la vez.
- **Red:** en clase, por cable, el ping tarda 0,5 ms; en casa el Wi-Fi da 5 ms y velocidades variables. Para descargar ISOs grandes, mejor en clase o con cable.
- El entorno vive en el disco USB y funciona en los dos equipos, una vez resuelta la incidencia de Secure Boot en casa.

## 5. Comparación con el inventario del grupo

| Aspecto | Equipos del grupo (aula) | Mis equipos |
|---|---|---|
| Equipo con las VM | PC-01: i3-4160 (2 núcleos/4 hilos), 8 GB DDR3, HDD de 500 GB | Portátiles con 18 y 32 hilos, 30–31 GiB, SSD por USB |
| Cuello de botella que encontró el grupo | HDD lento con muchos trozos pequeños (107 IOPS en el PC-01) y red Gigabit con ficheros grandes | Disco USB compartido por todas las VM y RAM para el número de VM |
| Disco, ficheros grandes | HDD del PC-01: 181 MiB/s | SSD USB: 410 MB/s (~391 MiB/s) |
| Herramientas | `fio`, `smartctl`, `openssl speed`, `ethtool`, `iperf3`, `top` | `openssl speed`, `dd`, `fio`, `smartctl`, `memtester`, `ping`, `curl` |
| Disco, trozos pequeños | HDD del PC-01: 107 IOPS | SSD USB: 41.200 IOPS |
| Errores CRC | HDD del PC-01: 1 error | SSD USB: contadores altos pero que ya no suben (errores antiguos) |
