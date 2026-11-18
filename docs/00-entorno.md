# T0 · Mi entorno de pruebas

RA: FDM RA1. Enunciado, Tasca 0 individual: preparar un entorno de pruebas en mi equipo que simule, más o menos, el del grupo, y justificar sus limitaciones.

## Lo que pide la rúbrica

- [x] Preparar el entorno propio de pruebas (11 VM creadas: 10 el 15/11 y `pve3` el 18/11).
- [x] Identificar los recursos físicos y justificar sus limitaciones.
- [x] Ampliación: documentarlo en Markdown y Git ([repositorio en GitHub](https://github.com/varo010/asix-individual-ISO)).

## 1. Dónde vive el entorno

| Dato | Valor |
|---|---|
| Disco | SSD Kingston A400 de 480 GB en caja USB: es «el disco de clase» que recomienda el enunciado |
| Sistema anfitrión | Linux Mint 22.3 (en ese mismo disco) |
| Equipos donde funciona | Portátil de clase (ASUS ExpertBook) y portátil de casa (Lenovo Legion Pro 5). Ver [02-equipos](02-equipos.md) |
| Hipervisor | VirtualBox 7.2.18 + Extension Pack |
| Carpeta de las VM | `/home/batoi/VirtualBox VMs/` |
| Espacio libre | ~227 GB |

**Por qué así:** al llevar las VM en el disco USB, el mismo entorno funciona en clase y en casa sin copiar nada. El precio es que el disco USB es el componente más lento ([02-equipos](02-equipos.md#2-cuellos-de-botella-previstos)).

## 2. Máquinas virtuales previstas

Discos **dinámicos**: el tamaño es el máximo, pero solo ocupan lo que se escribe. vCPU = núcleos de CPU que ve la VM.

| VM | Función | SO | vCPU | RAM | Disco | Red | Estado |
|---|---|---|---|---|---|---|---|
| `r-mikrotik` | Router alternativa 1 | MikroTik RouterOS CHR 7.24.6 | 1 | 256 MB | Imagen CHR (~128 MB) | NAT + LAN | Instalado y configurado (T5) |
| `r-opnsense` | Router alternativa 2 | OPNsense 26.7 (actualizado a 26.7.9) | 2 | 2 GB | 20 GB | NAT + LAN | Instalado y configurado (T5) |
| `r-nftables` | Router alternativa 3 | Debian 13.7 sin escritorio + nftables | 1 | 1 GB (512 MB no bastan para el instalador) | 8 GB | NAT + LAN | Instalándose |
| `nas-truenas` | NAS y copias | TrueNAS Community 25.10.8 | 2 | 8 GB | 20 GB sistema + 20 GB copias | LAN | Instalada; IP fija 192.168.27.10; ISO sacada |
| `pve1` | Nodo Proxmox 1 | Proxmox VE 9.2 | 2 | 4 GB | 32 GB | LAN | Creada; ISO comprobada puesta; sin instalar |
| `pve2` | Nodo Proxmox 2 | Proxmox VE 9.2 | 2 | 4 GB | 32 GB | LAN | Creada; ISO comprobada puesta; sin instalar |
| `pve3` | Nodo Proxmox 3 (añadido el 18/11) | Proxmox VE 9.2 | 2 | 4 GB | 32 GB | LAN | Creada; ISO comprobada puesta; sin instalar |
| `srv-dc` | Dominio | Windows Server 2025 (evaluación 180 días) | 2 | 4 GB | 50 GB | LAN | Instalándose |
| `srv-linux` | Servidor de infraestructura (y RAID de la T6) | Debian 13.7 sin escritorio | 1 | 1 GB | 10 GB + 4 discos de 1 GB | LAN | Creada; ISO comprobada puesta; sin instalar |
| `cli-win11` | Cliente Windows | Windows 11 Pro (ISO oficial en español, compilación 26300) | 2 | 4 GB | 64 GB (TPM 2.0, UEFI, Secure Boot) | LAN | Creada; ISO comprobada puesta; sin instalar |
| `cli-linux` | Cliente GNU/Linux | Ubuntu Desktop 26.04.1 LTS | 4 | 4 GB (con 2 vCPU y 3 GB iba al 100 %) | 25 GB | LAN | Se usa en modo live (sin instalar) como cliente de pruebas |

Total si estuvieran todas encendidas: **unos 36 GB de RAM y 21 vCPU**.

Dentro de los nodos Proxmox irán un contenedor y una VM Debian sin escritorio para la migración (T5), con la RAM de los nodos.


### Cómo se crearon las VM y de dónde salen las imágenes

- Las 10 primeras VM se crearon el 15/11/2026 y `pve3` el 18/11/2026, con `VBoxManage` (VirtualBox por comandos), todas en el grupo `ISO-individual`. Así solo se crea el «hardware» virtual: la instalación se hace a mano dentro de cada VM (T5).
- Antes de poner una ISO en una VM, calculo su SHA256 con `sha256sum` (una «huella» del fichero: si cambia un solo bit, la huella es otra) y lo comparo con el que publica el fabricante. Solo se usan las que coinciden. Resultado: [evidencias/t5-isos-sha256-20261116.txt](../evidencias/t5-isos-sha256-20261116.txt).
- **Windows Server 2025 (evaluación):** Microsoft no publica la huella (hash). Se acepta porque viene de su servidor oficial por HTTPS y la descarga está completa; se apunta su huella para demostrar más adelante que es la misma.
- **Windows 11:** la `w11.iso` que había en el disco **no se usa**: no se sabe de dónde salió y es una versión de 2023. El 16/11/2026 descargué la actual (español, 64 bits) de <https://www.microsoft.com/es-es/software-download/windows11>. Microsoft publica su huella en la misma página («Verificar la descarga») y **coincide**: `98287037…72a14f9a`.
- Las VM aún no arrancan en el portátil de casa hasta resolver [INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md) (Secure Boot).

## 3. Qué puedo arrancar a la vez y qué no (limitaciones)

**RAM: es el límite principal.** Los dos portátiles tienen 30–31 GiB. El anfitrión (Linux Mint, navegador, VirtualBox) necesita unos 4 GiB, así que quedan unos **26 GiB para VM**. Las once VM juntas piden unos 36 GB: **no caben todas a la vez**.

- Solo un router a la vez: las tres alternativas usan la misma puerta de enlace ([04-red](04-red.md)) y nunca tiene sentido tener dos.
- Con el router (0,25–2 GB según cuál) y el NAS (8 GB) siempre encendidos, hasta 10 GB, el resto se arranca por escenarios. La RAM es el máximo, con el router más pesado (OPNsense):

| Escenario | VM encendidas | RAM aprox. |
|---|---|---|
| Comunicaciones (T5) | Router + NAS + cliente Linux | hasta 13 GB |
| Clúster y migración (T5) | Router + NAS + `pve1` + `pve2` + `pve3` | hasta 22 GB |
| Dominio y clientes (T5) | Router + NAS + `srv-dc` + `cli-win11` + `cli-linux` | hasta 21 GB |
| Servidor y RAID (T6) | Router + NAS + `srv-linux` | hasta 11 GB |
| Carga máxima (T7) | Se irán sumando VM hasta notar bloqueos | Medido en [07-rendimiento](07-rendimiento.md) |

**CPU: no es el límite.** 21 vCPU en total, pero nunca encendidas todas a la vez, frente a 18 hilos en clase y 32 en casa. En casa, con batería, la CPU rinde mucho menos ([02-equipos](02-equipos.md)): se trabajará con cargador.

**Disco: limita la velocidad, no la cantidad.** Todas las VM comparten el SSD por USB (~400 MB/s con ficheros grandes). Si todas escribieran el máximo de su disco, sumarían unos 280 GB, más de lo libre (~227 GB). Con discos dinámicos ocuparán bastante menos, pero hay que vigilar el espacio. Las VM antiguas de otras prácticas ocupan unos 64 GB y se pueden borrar si hace falta.

**Proxmox dentro de VirtualBox:** necesita *virtualización anidada*: que la VM pueda crear a su vez otras VM. Las VM de dentro de Proxmox irán despacio y con poca RAM: sirven para probar el clúster y la migración, no para trabajar con ellas.

**Firmware:** en el portátil de casa hay que registrar la clave de VirtualBox para que funcione con Secure Boot ([INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md)).

## 4. Comparación con el entorno del grupo

| Aspecto | Grupo 3 (aula) | Mi entorno individual |
|---|---|---|
| Dónde se ejecuta | Equipos físicos del aula + servidor Proxmox del centro (`1asixb`) | Un solo portátil con VirtualBox (el de clase o el de casa, con el mismo disco USB) |
| Router | MikroTik hEX **físico**; OPNsense y Debian en VM de VirtualBox | Los tres en VM (MikroTik como CHR) |
| Red exterior (WAN) | Red del aula 172.16.222.0/24 | NAT de VirtualBox, IP por DHCP |
| LAN | 192.168.30.0/24 con switches físicos | 192.168.27.0/24, red interna de VirtualBox |
| NAS | TrueNAS en un PC físico (SSD + HDD de 2 TB) | TrueNAS en VM con discos virtuales |
| Clúster Proxmox | 3 nodos anidados en el servidor del centro | 3 nodos anidados en VirtualBox (al principio eran 2; ver *Cambio de diseño*) |
| Clientes | Portátiles físicos de los compañeros | 2 VM (Windows 11 y Ubuntu) |
| Qué simulo y qué no | — | Simulo el switch (red interna) y el router físico (CHR). No hay hardware de red real |

### Cambio de diseño: de 2 a 3 nodos de Proxmox (18/11/2026)

Al principio puse **2 nodos** para ahorrar RAM. Lo cambié a **3** (`pve3`) por el **quórum**:

- Un clúster de Proxmox funciona por **votación**: cada nodo tiene 1 voto y, para hacer cambios (arrancar o migrar VM), hace falta **más de la mitad** de los votos. Eso es el **quórum**.
- **Con 2 nodos**, si cae uno queda 1 de 2: **no es mayoría** y el que queda se bloquea por seguridad. No sabe si el otro ha muerto o solo se ha cortado la red entre ellos; si los dos siguieran por separado, podrían arrancar la misma VM dos veces y estropear sus datos (*split-brain*, «cerebro partido»).
- **Con 3 nodos**, si cae uno quedan 2 de 3: **es mayoría** y el clúster sigue funcionando. Así se puede probar la **alta disponibilidad** (si un nodo muere, sus VM arrancan solas en otro) y demostrar el quórum apagando un nodo.

**¿Cabe?** Sí, en el escenario del clúster: router (hasta 2 GB) + NAS (8 GB) + 3 nodos × 4 GB = **unos 22 GB**, de los 26 GB que quedan para VM. Además, así tengo el mismo número de nodos que el grupo.

`pve3` se creó igual que `pve2`: 2 vCPU, 4 GB, disco de 32 GB, UEFI, virtualización anidada y la tarjeta en `lan-iso` con el modo promiscuo en *Permitir todo*.

## 5. Problemas encontrados

| Problema | Causa | Solución | Incidencia |
|---|---|---|---|
| VirtualBox no arranca VM en el portátil de casa | Secure Boot activado y la clave de los controladores de VirtualBox sin registrar en ese portátil | Registrar la clave en el portátil de casa | [INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md) |
