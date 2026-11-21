# T7 · Rendimiento y diagnóstico

RA: FDM RA1, ISO RA6. Recursos del equipo físico con distintas cargas de VM.

## Lo que pide la rúbrica

- [x] Medir CPU, RAM, disco y red durante las cargas previstas.
- [x] Interpretar las medidas e identificar los recursos saturados.
- [x] Analizar los recursos físicos con diferentes cargas de VM.
- [x] Determinar cuántas VM permiten trabajar sin bloquear el equipo.
- [x] Determinar la capacidad del hardware para ejecutar las VM.
- [x] Entregar el informe de medidas y conclusiones.
- [x] Ampliación: primera optimización aplicada a las VM.
- [x] Ampliación: segunda optimización aplicada a las VM.
- [x] Ampliación: comparar medidas antes y después.

## 0. El equipo y cómo medí

| Pieza | Valor |
|---|---|
| Equipo | Portátil Lenovo Legion Pro 5 (el de casa) |
| CPU | Intel Core i9-14900HX: **32 hilos** (24 núcleos) |
| RAM | **31,8 GB** + 8 GB de intercambio (*swap*) en fichero + 7,8 GB de *zram* (intercambio comprimido en RAM) |
| Disco del entorno | **SSD Kingston A400 de 480 GB por USB** (`sda`): ahí están Ubuntu, VirtualBox y todos los discos de las VM |
| Virtualización | VirtualBox 7.2.18 sobre Ubuntu |

**Herramientas** (paquete `sysstat`, de Linux), siempre con muestras cada 5 segundos:

| Herramienta | Qué mide | Qué miro |
|---|---|---|
| `free -m` | RAM | **«disponible»**: lo que aún pueden usar programas nuevos. «Libre» engaña: Linux usa la RAM sobrante como caché |
| `vmstat 5 6` | CPU, RAM, intercambio y disco | `r` (procesos esperando CPU), `si/so` (si se usa el intercambio), `bi/bo` (lectura/escritura de disco) |
| `mpstat` | CPU | `%idle`: el porcentaje de CPU **sin usar** |
| `iostat -dx sda` | El disco USB | `%util`: cuánto tiempo está ocupado el disco |
| `sar -n DEV` | Red | KB/s por la tarjeta del portátil |
| `iperf3` | Velocidad de red entre dos VM | Gbit/s |

Cada medida se guardó en un fichero de texto (columna «Evidencia»).

## 1. Línea base (equipo sin VM)

Con todas las VM apagadas, solo con el escritorio de Ubuntu y VS Code abiertos. → [t7-medida-01](<../evidencias/t7-medida-01-base-0vm-20261121.txt>)

| Recurso | Herramienta | Valor |
|---|---|---|
| CPU | `mpstat` | **95,8 % sin usar** |
| RAM | `free -m` | 7,5 GB usados, **24,3 GB disponibles** |
| Intercambio | `swapon --show` | 0 B usados |
| Disco USB | `iostat` | 0,1 % ocupado |
| Red | `sar` | 4 KB/s |

Los 7,5 GB de partida son del escritorio, el navegador, VS Code y la caché del sistema.

## 2. Medidas por escenario de carga

RAM de cada VM: router 1 GB, NAS 8 GB, nodos Proxmox 4 GB cada uno, `srv-dc` 4 GB, `srv-linux` 1 GB, `cli-win11` 4 GB, `cli-linux` 4 GB.

| # | Escenario (VM arrancadas) | CPU sin usar | RAM usada | RAM disponible | Disco USB (pico) | Intercambio | ¿Se nota lento? | Evidencia |
|---|---|---|---|---|---|---|---|---|
| 0 | Ninguna | 95,8 % | 7,5 GB | 24,3 GB | 0 MB/s | 0 | No | [01](<../evidencias/t7-medida-01-base-0vm-20261121.txt>) |
| 1 | Router | 95,7 % | 8,1 GB | 23,7 GB | 0 MB/s | 0 | No | [02](<../evidencias/t7-medida-02-carga-1vm-router-20261121.txt>) |
| 2 | Router + NAS + `srv-linux` + 3 nodos Proxmox, **en reposo** (6 VM) | 94,2 % | 21,6 GB | 10,2 GB | 1 MB/s | 0 | No | [03](<../evidencias/t7-medida-03-carga-6vm-reposo-20261121.txt>) |
| 3 | Las mismas 6, con los **3 Proxmox actualizándose** (225 paquetes cada uno) | 91,5 % | 24,5 GB | 7,3 GB | lee 48, escribe **97 MB/s** | 0 | No | [04](<../evidencias/t7-medida-04-carga-6vm-actualizando-20261121.txt>) |
| 4 | Las mismas 6, **migrando** la VM de prueba entre nodos | 86,1 % | 21,6 GB | 10,3 GB | lee 105, escribe **170 MB/s** | 0 | No | [05](<../evidencias/t7-medida-05-carga-6vm-migrando-20261121.txt>) |
| 5 | 8 VM (todas menos `cli-linux`), Proxmox actualizándose | 89,8 % | 27,9 GB | **3,9 GB** | escribe 10 MB/s, 26 % ocupado | 0 | No | [06](<../evidencias/t7-medida-06-carga-8vm-20261121.txt>) |
| 6 | **9 VM**: arrancando `srv-dc`, `cli-win11` y `cli-linux` **a la vez** | 92,7 % | 25,4 GB (subiendo) | 6,4 GB | lee **130 MB/s** | 0 | Arrancan los tres en menos de 1 minuto | [07](<../evidencias/t7-medida-07-carga-9vm-arrancando-20261121.txt>) |
| 7 | **9 VM** en reposo | 92,5 % | 27,5 GB | 4,3 GB | 16 MB/s | 0 | No | [08](<../evidencias/t7-medida-08-carga-9vm-reposo-20261121.txt>) |
| 8 | **9 VM + carga máxima**: migración en caliente + copias al NAS de `srv-linux` y `cli-win11` a la vez | 87,0 % | **28,2 GB** | **3,6 GB** (libre: 0,26 GB) | lee 97, escribe **159 MB/s** | 0 | No: SSH a `srv-linux` en 0,3 s y `ping` al NAS de 0,8 ms, **igual que en reposo** | [09](<../evidencias/t7-medida-09-carga-9vm-maxima-20261121.txt>) |

**Cómo comprobé «si se nota lento»:** además de usar el escritorio mientras tanto, medí el tiempo de abrir una sesión SSH en `srv-linux` (`time ssh srv-linux true`) y el `ping` al NAS, en reposo y con la carga máxima: **0,29-0,44 s y 0,8 ms en los dos casos**. La migración de la VM de prueba también tardó lo mismo con carga (35 s) que sin ella (31-37 s).

## 3. Interpretación: qué recurso se satura

| Recurso | Uso máximo medido | ¿Saturado? | Por qué |
|---|---|---|---|
| **RAM** | 28,2 de 31,8 GB (**89 %**), solo 3,6 GB disponibles | **Sí, es el límite** | Cada VM se queda con la RAM que va usando y no la devuelve. Con una VM más de 4 GB, Linux empezaría a usar el intercambio y **todo** iría lento |
| **Disco USB** | 170 MB/s de escritura en la migración; 130 MB/s de lectura al arrancar 3 VM | **Casi, en momentos** | Los discos de las 11 VM están en un SSD **por USB**. Es lo que más se nota en los arranques y migraciones (un SSD SATA da ~500 MB/s y uno NVMe más de 3000) |
| **CPU** | 14 % como mucho (86 % sin usar) | **No** | 32 hilos para 18 vCPU asignadas. Ninguna prueba se acercó |
| **Red** | 10 Gbit/s entre VM; Internet casi sin uso | **No** | La red entre VM es interna de VirtualBox: no pasa por un cable. Solo la usan las actualizaciones |
| **Intercambio** | 0 B en todas las pruebas | No se llegó a usar | Es la señal de alarma: si aparecen números en `si`/`so` de `vmstat`, falta RAM |

**Curioso:** las 9 VM tienen asignados **34 GB** de RAM entre todas, **más que los 31,8 GB del portátil**, y aun así funcionan. Es porque VirtualBox no reserva toda la RAM de golpe: la va dando **a medida que la VM la usa**. Una VM en reposo no llega a tocar toda la suya. Pero si todas la usaran a la vez (por ejemplo, el NAS llenando su caché ZFS), no cabrían.

### Qué satura cada VM

| VM | Qué consume más | Cuándo |
|---|---|---|
| `nas-truenas` | **RAM** (8 GB: mínimo oficial; ZFS usa la sobrante como caché) | Siempre |
| `srv-dc`, `cli-win11` | **RAM** y **disco** | Al arrancar (Windows lee mucho) y en las actualizaciones |
| `pve1-3` | **RAM** (4 GB cada uno) y **disco** | Al actualizar (97 MB/s) y al migrar (170 MB/s) |
| `cli-linux` | RAM y CPU | Con el navegador abierto (webs de OPNsense y TrueNAS) |
| `r-nftables`, `srv-linux` | Casi nada (1 GB, 1 vCPU) | Son lo más ligero del entorno |

## 4. Cuántas VM a la vez

**Respuesta: las 9 a la vez funcionan sin bloquear el portátil** (son las que necesita el proyecto, con un router cada vez). Con carga máxima aún quedan 3,6 GB disponibles y la CPU al 87 % sin usar.

| VM a la vez | Estado |
|---|---|
| Hasta 6 (router + NAS + servidor + Proxmox) | Sobrado: 10 GB disponibles |
| 9 (todo el entorno) | **Funciona bien**, pero **al límite de RAM** (3,6-4,3 GB disponibles) |
| 10 o más (por ejemplo, otro router o una VM más de 4 GB) | **No recomendable**: se acabaría la RAM y empezaría el intercambio en el disco USB, que es lento |

**Capacidad del hardware:**
- **La CPU sobra**: con 32 hilos podría con el doble de VM.
- **La RAM manda**: con 32 GB caben unos **34 GB asignados** gracias a que las VM no la usan entera. Para más VM hay que **ampliar a 64 GB** o asignar menos a cada VM.
- **El disco USB** es el segundo cuello de botella. Pasar los discos de las VM a un SSD interno (NVMe) haría más rápidos los arranques y las migraciones.

**Cómo lo organizo:** no arranco todo siempre. Por tareas: comunicaciones (router + `cli-linux` + NAS), servidores (router + NAS + servidores) y Proxmox (router + NAS + 3 nodos). Ver la tabla de escenarios de [00-entorno](00-entorno.md).

## 5. Ampliación: optimizaciones

| Optimización | Antes | Después | Mejora | Evidencia |
|---|---|---|---|---|
| **1. Tarjeta de red paravirtualizada** (`virtio-net`) en `srv-linux` y `pve1` | Intel PRO/1000 emulada: **5,65 Gbit/s**, usando 362 % de CPU (3,6 hilos) | `virtio-net`: **10,4 Gbit/s**, usando 329 % de CPU | **+84 % de velocidad con un 9 % menos de CPU** (la mitad de CPU por cada Gbit/s) | [opt-1](<../evidencias/t7-optimizacion-1-red-virtio-antes-despues-20261121.txt>) |
| **2. Globo de memoria** (*memory ballooning*) de 1,5 GB en `srv-dc` y `cli-win11` | 2.698 MB disponibles en el portátil | 2.781 MB disponibles | **Solo +83 MB**: no compensa | [opt-2](<../evidencias/t7-optimizacion-2-globo-memoria-antes-despues-20261121.txt>) |

### 1. Tarjeta de red paravirtualizada

- **Qué es:** VirtualBox, de serie, **imita** una tarjeta Intel real. Tiene que simular cada detalle del hardware, y eso gasta CPU. Con **`virtio`** la VM **sabe que es virtual** y habla directamente con VirtualBox, sin imitar nada (*paravirtualización*).
- **Cómo:** con la VM apagada, `VBoxManage modifyvm srv-linux --nictype1 virtio` (y lo mismo en `pve1`). Linux ya trae el controlador (`virtio_net`). La tarjeta conserva su nombre (`enp0s3` / `nic0`) y su MAC, así que la red siguió funcionando sin tocar nada más.
- **Prueba:** `iperf3` de `srv-linux` a `pve1` durante 20 segundos, midiendo a la vez la CPU de las dos VM con `pidstat`.
- **Nota:** en la medida de «antes» las dos VM tenían ventana; en la de «después», no (arranque sin ventana). La ventana gasta algo de CPU, así que la diferencia de CPU real puede ser un poco menor. La velocidad no depende de eso.
- **Dónde serviría más:** en el NAS (las copias van por la red) y en los nodos Proxmox (las migraciones). En Windows haría falta instalar el controlador `virtio` aparte.

### 2. Globo de memoria

- **Qué es:** un controlador de las Guest Additions **«infla un globo»** dentro de la VM: ocupa RAM de la VM para que, en teoría, VirtualBox se la devuelva al portátil. Sirve para quitar RAM a una VM **sin apagarla**.
- **Cómo:** `VBoxManage controlvm srv-dc guestmemoryballoon 1536` (y en `cli-win11`). Antes comprobé que cada Windows tenía 2,7 GB libres, para no dejarlo sin memoria.
- **Resultado:** dentro de Windows funcionó (el globo ocupó 1.536 MB y la memoria libre bajó de 2,7 a 1,3 GB), pero el **portátil solo ganó 83 MB**. La RAM que el globo «quitó» era memoria que Windows tenía libre y **que el portátil nunca había llegado a dar** (VirtualBox solo da la RAM cuando la VM la usa, punto 3). No había nada que devolver.
- **Conclusión:** en este equipo es mejor **asignar menos RAM desde el principio** a las VM que lo permitan que usar el globo. Lo dejé desactivado otra vez (`guestmemoryballoon 0`).

## 6. Conclusiones

1. **El portátil puede con el entorno completo (9 VM a la vez)** sin bloquearse, incluso con migraciones y copias al mismo tiempo.
2. **El recurso que manda es la RAM**: con todo arrancado queda un 11 % disponible. Es lo primero que ampliaría (64 GB).
3. **El segundo es el disco USB**: se nota en los arranques y las migraciones (picos de 130-170 MB/s). Mejor un SSD interno.
4. **La CPU y la red sobran**: la CPU nunca pasó del 14 % de uso.
5. **Optimización que sí vale la pena:** tarjetas de red `virtio` (casi el doble de velocidad gastando menos CPU). **La que no:** el globo de memoria en VirtualBox sobre Linux.
6. En el portátil de clase (ASUS), si tiene menos RAM, habría que arrancar **solo las VM de la tarea** que toque (tabla de escenarios de [00-entorno](00-entorno.md)).
