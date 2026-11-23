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
| T0 · Entorno de pruebas | Montar mi entorno y justificar sus límites. Ampliación: documentarlo en Git | [00-entorno](docs/00-entorno.md) | FDM RA1 | Hecho: diseñado, 13 VM creadas con sus ISO comprobadas (`pve3` añadida el 18/11; `dades` y `serveis` el 23/11), repo en GitHub. Incidencia [INC-01](incidencias/INC-01-virtualbox-secure-boot-casa.md) cerrada |
| T2 · Equipos | Inventario y comparativa del equipo de clase y el de casa; cuellos de botella. Ampliación: tests de memoria, disco y red | [02-equipos](docs/02-equipos.md) | FDM RA1 | Hecho, con ampliación (tests en los dos equipos) |
| T4 · Red | Esquema de red y montaje en VirtualBox. Ampliación: implementarlo y verificarlo | [04-red](docs/04-red.md) | ISO RA1 | Hecho, con ampliación (red implementada y comprobada con `ping`, `traceroute` y rutas) |
| T5 · Comunicaciones | MikroTik, OPNsense y Debian con nftables | [05-comunicaciones](docs/05-comunicaciones.md) | ISO RA1, RA3, RA6 | Terminada: los tres routers (red, DHCP, DNS, NAT, cortafuegos con registro, copia en el NAS y recuperación) y comparación de las tres soluciones |
| T5 · Datos | TrueNAS como servidor de copias | [05-truenas](docs/05-truenas.md) | ISO RA1, RA3 | Hecho: instalado, IP fija, pool `copias` con una carpeta y un usuario por equipo (routers, servidores, clientes y Proxmox) |
| T5 · Virtualización | Clúster Proxmox y migración medida | [05-proxmox](docs/05-proxmox.md) | ISO RA1, RA6 | Hecho: clúster de 3 nodos con quórum probado, contenedor y VM migrados y medidos (borrados el 23/11: ahora son las VM independientes `dades` y `serveis`), copias al NAS |
| T5 · Servidores | Windows Server 2025 y servidor GNU/Linux | [05-servidores](docs/05-servidores.md) | ISO RA1, RA3 | Hecho: dominio `iso27.lan` y Debian, con copias automáticas y recuperación probada |
| T5 · Clientes | Windows 11 y GNU/Linux | [05-clientes](docs/05-clientes.md) | ISO RA1, RA3 | Hecho: Windows 11 y Ubuntu, con copias automáticas y recuperación probada |
| T5 · Copias | Qué se copia, cómo y dónde. Ampliación: automatizarlo | [05-copias](docs/05-copias.md) | ISO RA3 | Hecho, con ampliación (copias automáticas diarias) |
| T6 · RAID | 3 tipos de RAID. Ampliación: espejo entre dos equipos | [06-raid](docs/06-raid.md) | ISO RA3 | Hecho, con ampliación (espejo DRBD entre `srv-linux` y `cli-linux`, probado apagando un equipo) |
| T7 · Rendimiento | Qué satura cada VM y cuántas puedo usar a la vez. Ampliación: 2 optimizaciones | [07-rendimiento](docs/07-rendimiento.md) | FDM RA1, ISO RA6 | Hecho, con ampliación (2 optimizaciones medidas) |
| T8 · Final | Caso práctico el 25-11 sobre este entorno | [08-final](docs/08-final.md) | Todos | Pendiente |

## Estructura

- [docs/](docs/) — un documento por tarea
- [diagramas/](diagramas/) — esquemas editables (`.drawio`) y su imagen
- [evidencias/](evidencias/) — salidas de comandos y capturas, sin contraseñas
- [incidencias/](incidencias/) — problemas encontrados y cómo se resolvieron
- [docs/preguntas.md](docs/preguntas.md) — preguntas de la defensa
- [docs/09-resumen-por-vm.md](docs/09-resumen-por-vm.md) — qué hay en cada VM y cómo explicarlo (repaso para la defensa)

## Normas

- Nunca se suben contraseñas, ISOs ni discos virtuales (ver [.gitignore](.gitignore)).
- Las evidencias se nombran `tX-descripcion-AAAAMMDD` y se explican en el documento que las enlaza.
- Las comparaciones con el grupo citan el [repositorio del grupo](https://github.com/varo010/asix-aula-grup-03-ISO); el trabajo individual no reutiliza su configuración.
