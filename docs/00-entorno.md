# T0 · Mi entorno de pruebas

RA: FDM RA1. Enunciado, Tasca 0 individual: preparar un entorno de pruebas en mi equipo que simule, más o menos, el del grupo, y justificar sus limitaciones.

## Lo que pide la rúbrica

- [ ] Preparar el entorno propio de pruebas.
- [ ] Identificar los recursos físicos y justificar sus limitaciones.
- [ ] Ampliación: documentarlo en Markdown y Git (esquema de VM, recursos y problemas resueltos).

## 1. Equipo anfitrión

Datos completos y comparación clase/casa en [02-equipos](02-equipos.md).

| Dato | Valor |
|---|---|
| Equipo donde vive el entorno | N/C (pendiente de decidir: clase o casa) |
| Disco donde se guardan las VM | N/C |
| Hipervisor | VirtualBox 7.2.18 + Extension Pack |
| Carpeta de las VM | N/C |

## 2. Máquinas virtuales

| VM | Función | SO y versión | vCPU | RAM | Disco | Red | Estado |
|---|---|---|---|---|---|---|---|
| | Router (MikroTik CHR) | | | | | WAN NAT + LAN | Pendiente |
| | Router (OPNsense) | | | | | WAN NAT + LAN | Pendiente |
| | Router (Debian + nftables) | | | | | WAN NAT + LAN | Pendiente |
| | NAS / copias (TrueNAS) | | | | | LAN | Pendiente |
| | Nodo Proxmox 1 | | | | | LAN | Pendiente |
| | Nodo Proxmox 2 | | | | | LAN | Pendiente |
| | Dominio (Windows Server 2025) | | | | | LAN | Pendiente |
| | Servidor GNU/Linux | | | | | LAN | Pendiente |
| | Cliente Windows 11 | | | | | LAN | Pendiente |
| | Cliente GNU/Linux | | | | | LAN | Pendiente |

## 3. Qué puedo arrancar a la vez y qué no (limitaciones)

> Explica con tus palabras, con la suma de RAM/CPU/disco de la tabla anterior frente a lo que tiene el equipo, qué combinaciones de VM caben a la vez y cuáles no. La medida real va en [07-rendimiento](07-rendimiento.md).

## 4. Comparación con el entorno del grupo

| Aspecto | Grupo 3 (aula) | Mi entorno individual |
|---|---|---|
| Dónde se ejecuta | Equipos físicos del aula + servidor Proxmox del centro (`1asixb`) | Un solo equipo con VirtualBox |
| Router | MikroTik hEX **físico**; OPNsense y Debian en VM de VirtualBox | Los tres en VM (MikroTik como CHR) |
| Red exterior (WAN) | Red del aula 172.16.222.0/24 (IP asignada por el profesor o puente) | NAT de VirtualBox, IP por DHCP |
| LAN | 192.168.30.0/24 con switches físicos | 192.168.XX.0/24, red interna de VirtualBox |
| NAS | TrueNAS en un PC físico (SSD + HDD de 2 TB) | TrueNAS en VM con discos virtuales |
| Clúster Proxmox | 3 nodos anidados en el servidor del centro | N/C nodos anidados en VirtualBox |
| Clientes | Portátiles físicos de los compañeros | VM |
| Qué simulo y qué no | — | N/C |

## 5. Problemas encontrados

| Problema | Causa | Solución | Incidencia |
|---|---|---|---|
