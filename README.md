# Entorno de pruebas individual · ISO / FDM · «Naix un sistema»

Parte individual de la actividad ASIX ThinkLab (1.º ASIX). Aquí está el montaje de **mi propio entorno de pruebas** en VirtualBox: el mismo tipo de sistema que monta el grupo 3 en el aula, pero hecho por mí desde cero en mi equipo.

| Dato | Valor |
|---|---|
| Alumno | Álvaro · grupo 3 |
| Puesto en el aula (XX) | 27 (IP por DHCP en el aula; ver [04-red](docs/04-red.md)) |
| Red LAN del entorno | 192.168.27.0/24 |
| Salida al exterior | NAT de VirtualBox (IP por DHCP) |
| Hipervisor | VirtualBox 7.2 |
| Periodo | 12-11-2026 → 22-11-2026 (final individual: 25-11-2026) |

## Estado por tarea

Estados: Pendiente · En curso · Hecho · Simulado.

| Tarea | Qué pide la parte individual | Documento | RA | Estado |
|---|---|---|---|---|
| T0 · Entorno de pruebas | Montar mi entorno y justificar sus límites. Ampliación: documentarlo en Git | [00-entorno](docs/00-entorno.md) | FDM RA1 | Hecho: diseñado, 11 VM creadas con sus ISO comprobadas (`pve3` añadida el 18/11), repo en GitHub. En casa falta cerrar [INC-01](incidencias/INC-01-virtualbox-secure-boot-casa.md) |
| T2 · Equipos | Inventario y comparativa del equipo de clase y el de casa; cuellos de botella. Ampliación: tests de memoria, disco y red | [02-equipos](docs/02-equipos.md) | FDM RA1 | Hecho, con ampliación (tests en los dos equipos) |
| T4 · Red | Esquema de red y montaje en VirtualBox. Ampliación: implementarlo y verificarlo | [04-red](docs/04-red.md) | ISO RA1 | Obligatorio hecho (diseño y VM creadas con su red); ampliación al instalar el router |
| T5 · Comunicaciones | MikroTik, OPNsense y Debian con nftables | [05-comunicaciones](docs/05-comunicaciones.md) | ISO RA1, RA3, RA6 | Terminada: los tres routers (red, DHCP, DNS, NAT, cortafuegos con registro, copia en el NAS y recuperación). Falta la comparación final |
| T5 · Datos | TrueNAS como servidor de copias | [05-truenas](docs/05-truenas.md) | ISO RA1, RA3 | Hecho: instalado, IP fija, pool `copias` y copias de los tres routers por SFTP |
| T5 · Virtualización | Clúster Proxmox y migración medida | [05-proxmox](docs/05-proxmox.md) | ISO RA1, RA6 | Pendiente |
| T5 · Servidores | Windows Server 2025 y servidor GNU/Linux | [05-servidores](docs/05-servidores.md) | ISO RA1, RA3 | Pendiente |
| T5 · Clientes | Windows 11 y GNU/Linux | [05-clientes](docs/05-clientes.md) | ISO RA1, RA3 | Pendiente |
| T5 · Copias | Qué se copia, cómo y dónde. Ampliación: automatizarlo | [05-copias](docs/05-copias.md) | ISO RA3 | Pendiente |
| T6 · RAID | 3 tipos de RAID. Ampliación: espejo entre dos equipos | [06-raid](docs/06-raid.md) | ISO RA3 | Pendiente |
| T7 · Rendimiento | Qué satura cada VM y cuántas puedo usar a la vez. Ampliación: 2 optimizaciones | [07-rendimiento](docs/07-rendimiento.md) | FDM RA1, ISO RA6 | Pendiente |
| T8 · Final | Caso práctico el 25-11 sobre este entorno | [08-final](docs/08-final.md) | Todos | Pendiente |

## Estructura

- [docs/](docs/) — un documento por tarea
- [diagramas/](diagramas/) — esquemas editables (`.drawio`) y su imagen
- [evidencias/](evidencias/) — salidas de comandos y capturas, sin contraseñas
- [incidencias/](incidencias/) — problemas encontrados y cómo se resolvieron
- [docs/preguntas.md](docs/preguntas.md) — preguntas de la defensa

## Normas

- Nunca se suben contraseñas, ISOs ni discos virtuales (ver [.gitignore](.gitignore)).
- Las evidencias se nombran `tX-descripcion-AAAAMMDD` y se explican en el documento que las enlaza.
- Las comparaciones con el grupo citan el [repositorio del grupo](https://github.com/varo010/asix-aula-grup-03-ISO); el trabajo individual no reutiliza su configuración.
