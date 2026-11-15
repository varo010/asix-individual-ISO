# T0 · Mi entorno de pruebas

RA: FDM RA1. Enunciado, Tasca 0 individual: preparar un entorno de pruebas en mi equipo que simule, más o menos, el del grupo, y justificar sus limitaciones.

## Lo que pide la rúbrica

- [ ] Preparar el entorno propio de pruebas (diseñado; VM pendientes de crear).
- [x] Identificar los recursos físicos y justificar sus limitaciones.
- [ ] Ampliación: documentarlo en Markdown y Git (repositorio local hecho; falta publicarlo en GitHub).

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

Discos **dinámicos**: el tamaño es el máximo, pero solo ocupan lo que se escribe.

| VM | Función | SO | vCPU | RAM | Disco | Red | Estado |
|---|---|---|---|---|---|---|---|
| `r-mikrotik` | Router alternativa 1 | MikroTik RouterOS CHR 7 | 1 | 256 MB | Imagen CHR (~128 MB) | NAT + LAN | Pendiente |
| `r-opnsense` | Router alternativa 2 | OPNsense | 2 | 2 GB | 20 GB | NAT + LAN | Pendiente |
| `r-nftables` | Router alternativa 3 | Debian 13 sin escritorio + nftables | 1 | 512 MB | 8 GB | NAT + LAN | Pendiente |
| `nas-truenas` | NAS y copias | TrueNAS Community 25.10 | 2 | 8 GB | 20 GB sistema + 20 GB copias | LAN | Pendiente |
| `pve1` | Nodo Proxmox 1 | Proxmox VE 9.2 | 2 | 4 GB | 32 GB | LAN | Pendiente |
| `pve2` | Nodo Proxmox 2 | Proxmox VE 9.2 | 2 | 4 GB | 32 GB | LAN | Pendiente |
| `srv-dc` | Dominio | Windows Server 2025 (evaluación 180 días) | 2 | 4 GB | 50 GB | LAN | Pendiente |
| `srv-linux` | Servidor de infraestructura (y RAID de la T6) | Debian 13 sin escritorio | 1 | 1 GB | 10 GB + 4 discos de 1 GB | LAN | Pendiente |
| `cli-win11` | Cliente Windows | Windows 11 Pro | 2 | 4 GB | 64 GB (TPM 2.0, UEFI, Secure Boot) | LAN | Pendiente |
| `cli-linux` | Cliente GNU/Linux | Ubuntu Desktop 26.04 LTS | 2 | 3 GB | 25 GB | LAN | Pendiente |

Total si estuvieran todas encendidas: **unos 31 GB de RAM y 17 vCPU**.

Dentro de los nodos Proxmox irán un contenedor y una VM Debian sin escritorio para la migración (T5), con la RAM de los nodos.

## 3. Qué puedo arrancar a la vez y qué no (limitaciones)

**RAM: es el límite principal.** Los dos portátiles tienen 30–31 GiB. El anfitrión (Linux Mint, navegador, VirtualBox) necesita unos 4 GiB, así que quedan unos **26 GiB para VM**. Las diez VM juntas piden unos 31 GB: **no caben todas a la vez**.

- Solo un router a la vez: las tres alternativas usan la misma puerta de enlace ([04-red](04-red.md)) y nunca tiene sentido tener dos.
- Con el router (0,25–2 GB según cuál) y el NAS (8 GB) siempre encendidos, hasta 10 GB, el resto se arranca por escenarios. La RAM es el máximo, con el router más pesado (OPNsense):

| Escenario | VM encendidas | RAM aprox. |
|---|---|---|
| Comunicaciones (T5) | Router + NAS + cliente Linux | hasta 13 GB |
| Clúster y migración (T5) | Router + NAS + `pve1` + `pve2` | hasta 18 GB |
| Dominio y clientes (T5) | Router + NAS + `srv-dc` + `cli-win11` + `cli-linux` | hasta 21 GB |
| Servidor y RAID (T6) | Router + NAS + `srv-linux` | hasta 11 GB |
| Carga máxima (T7) | Se irán sumando VM hasta notar bloqueos | Medido en [07-rendimiento](07-rendimiento.md) |

**CPU: no es el límite.** 17 vCPU frente a 18 hilos en clase y 32 en casa. En casa, con batería, la CPU rinde mucho menos ([02-equipos](02-equipos.md)): se trabajará con cargador.

**Disco: limita la velocidad, no la cantidad.** Todas las VM comparten el SSD por USB (~400 MB/s secuencial). Si todas escribieran el máximo de su disco, sumarían unos 280 GB, más de lo libre (~227 GB). Con discos dinámicos ocuparán bastante menos, pero hay que vigilar el espacio. Las VM antiguas de otras prácticas ocupan unos 64 GB y se pueden borrar si hace falta.

**Proxmox dentro de VirtualBox:** necesita *virtualización anidada* (VT-x anidado). Las VM de dentro de Proxmox irán despacio y con poca RAM: sirven para probar el clúster y la migración, no para trabajar con ellas.

**Firmware:** en el portátil de casa hay que registrar la clave de VirtualBox para que funcione con Secure Boot ([INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md)).

## 4. Comparación con el entorno del grupo

| Aspecto | Grupo 3 (aula) | Mi entorno individual |
|---|---|---|
| Dónde se ejecuta | Equipos físicos del aula + servidor Proxmox del centro (`1asixb`) | Un solo portátil con VirtualBox (el de clase o el de casa, con el mismo disco USB) |
| Router | MikroTik hEX **físico**; OPNsense y Debian en VM de VirtualBox | Los tres en VM (MikroTik como CHR) |
| Red exterior (WAN) | Red del aula 172.16.222.0/24 | NAT de VirtualBox, IP por DHCP |
| LAN | 192.168.30.0/24 con switches físicos | 192.168.XX.0/24, red interna de VirtualBox |
| NAS | TrueNAS en un PC físico (SSD + HDD de 2 TB) | TrueNAS en VM con discos virtuales |
| Clúster Proxmox | 3 nodos anidados en el servidor del centro | 2 nodos anidados en VirtualBox |
| Clientes | Portátiles físicos de los compañeros | 2 VM (Windows 11 y Ubuntu) |
| Qué simulo y qué no | — | Simulo el switch (red interna) y el router físico (CHR). No hay hardware de red real ni alta disponibilidad: con 2 nodos no hay quórum si uno cae |

## 5. Problemas encontrados

| Problema | Causa | Solución | Incidencia |
|---|---|---|---|
| VirtualBox no arranca VM en el portátil de casa | Secure Boot activado y clave de los módulos sin registrar en ese firmware | Registrar la clave (MOK) | [INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md) |
