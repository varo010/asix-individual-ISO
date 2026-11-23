# T5 · Copias de seguridad del entorno

RA: ISO RA3. Resumen de todas las copias y, como ampliación, su automatización.

## Lo que pide la rúbrica

- [x] Preparar el espacio de copias en TrueNAS y justificar qué es copia, cómo y dónde.
- [x] Ampliación: automatizar las copias de configuraciones o imágenes de los servidores críticos.

## 0. La idea en una frase

**Cada equipo deja su copia en el NAS, en su propia carpeta y con su propio usuario.** El NAS ([TrueNAS](05-truenas.md)) tiene un pool `copias` en un disco aparte de 20 GB, y dentro un *dataset* (carpeta con sus propios permisos) por equipo. Cada usuario del NAS **solo puede entrar en su carpeta**, y solo por **SFTP** (copia de ficheros cifrada sobre SSH).

¿Por qué un usuario por equipo? Si alguien roba la clave del cliente Windows, solo puede tocar las copias del cliente Windows, no las del dominio ni las de los routers.

## 1. Inventario de copias

| Equipo | Qué se copia | Formato y tamaño | Destino en el NAS | Frecuencia | Cómo se restaura | Probada |
|---|---|---|---|---|---|---|
| Router MikroTik | Backup completo cifrado + configuración en texto | `.backup` (22 KB) + `.rsc` (1 KB) | `copias/mikrotik` | A mano | `/tool fetch` y `/system backup load` | ✅ [T5 comunicaciones](05-comunicaciones.md#fallo-y-recuperación) |
| Router OPNsense | Configuración completa cifrada | `.xml` (59 KB) | `copias/opnsense` | A mano | **System → Configuration → Backups → Restore** | ✅ [T5 comunicaciones](05-comunicaciones.md#fallo-y-recuperación-1) |
| Router nftables | 4 ficheros de configuración | `.tar.gz` (1 KB) | `copias/nftables` | A mano | `sftp get` + `tar xzf … -C /` | ✅ 50 s · [T5 comunicaciones](05-comunicaciones.md#fallo-y-recuperación-2) |
| `srv-dc` (dominio) | Base de datos de AD (IFM) + SYSVOL + GPO + lista de usuarios | `.zip` (4,8 MB) | `copias/srv-dc` | **Diaria 02:30, automática** | Crear el DC desde el medio de la copia / `Restore-GPO` | Copia comprobada en el NAS |
| `srv-linux` | `/etc` + lista de paquetes | `.tar.gz` (425 KB) | `copias/srv-linux` | **Diaria 02:00, automática** | `sftp get` + `tar xzf … -C /` | ✅ 32 s · [05-servidores](05-servidores.md#copias-qué-sí-y-qué-no-1) |
| `cli-win11` | Documentos, Escritorio, Imágenes | `.zip` | `copias/cli-win11` | **Diaria 13:00, automática** | Tarea `Recuperar ultima copia` | ✅ 4 s · [05-clientes](05-clientes.md#copias) |
| `cli-linux` | `/home` + `/etc` + lista de paquetes | `.tar.gz` (1,4 MB) | `copias/cli-linux` | **Diaria 13:00, automática** | `sftp get` + `tar xzf … -C /` | ✅ < 1 s · [05-clientes](05-clientes.md#copias-1) |
| Nodos Proxmox (`pve1-3`) | Configuración del clúster (`/etc/pve`), red, Corosync y repositorios de cada nodo | `.tar.gz` (24 KB por nodo) | `copias/proxmox` | **Diaria 03:00, automática** | Reinstalar Proxmox desde la ISO y devolver los ficheros | Copia comprobada en el NAS · [05-proxmox](05-proxmox.md#7-copias-del-clúster) |
| Contenedor `ct-debian` (borrado el 23/11) | Imagen completa (`vzdump`) | `.tar.zst` (159 MB) | `copias/proxmox` | Una vez, a mano | `pct restore` | Copia comprobada en el NAS |

Estado del NAS leído por su API (datasets, usuarios y ficheros): → [t5-truenas-18-estado-copias-api-20261121.txt](<../evidencias/t5-truenas-18-estado-copias-api-20261121.txt>)

## 2. Qué no se copia y por qué

| No se copia | Por qué |
|---|---|
| **Sistemas operativos enteros** (imágenes de disco) | Se reinstalan desde la ISO comprobada en 10-20 minutos. Una imagen de Windows ocupa más de 15 GB: el disco de copias (20 GB) se llenaría con una sola |
| **Programas** | Se reinstalan. En Linux, con la lista de paquetes guardada (`dpkg --set-selections`) |
| Cachés, descargas, temporales | Se vuelven a generar solos |
| Datos de prueba del RAID | Son ficticios. En una empresa **sí** se copiarían: el RAID no es una copia |
| El propio NAS | Es el destino. En una empresa haría falta **una segunda copia fuera** (regla 3-2-1: 3 copias, en 2 soportes distintos, 1 fuera de la oficina) |

**Criterio general:** se copia **lo que es único y no se puede volver a crear igual** (configuraciones hechas a mano, la base de datos del dominio, los documentos de las personas). Lo que se puede reinstalar no se copia.

## 3. Ampliación: automatización

Los **servidores críticos** (`srv-dc`, `srv-linux`, los nodos Proxmox) y los clientes hacen la copia **solos**, cada día, sin que nadie tenga que acordarse.

| Equipo | Herramienta | Cuándo | Qué pasa si el equipo estaba apagado |
|---|---|---|---|
| `srv-dc`, `cli-win11` | **Programador de tareas** de Windows, como `SYSTEM` | 02:30 / 13:00 | `cli-win11`: «ejecutar lo antes posible» al encender |
| `srv-linux`, `cli-linux`, `pve1-3` | **Temporizador de systemd** (`copia-config.timer` → `copia-config.service`) | 02:00 / 13:00 / 03:00 | `Persistent=true`: se hace al arrancar |

**Todas siguen los mismos pasos:**

1. Juntan lo que hay que copiar en **un solo fichero comprimido** con la fecha y la hora en el nombre (`equipo-AAAAMMDD-HHMM`).
2. Lo suben al NAS por **SFTP con una clave SSH**, no con contraseña: así no hay ninguna contraseña escrita en un fichero. La clave del equipo está registrada en el usuario del NAS (**Credentials → Users → SSH public key**).
3. Borran en el NAS las copias viejas y dejan **las 7 más nuevas** (una semana). Así el disco de copias no se llena nunca.
4. Borran el fichero temporal del equipo.

**Cómo se comprueba que funciona:**

| Equipo | Comprobación |
|---|---|
| Linux | `systemctl list-timers copia-config.timer` (próxima ejecución) y `journalctl -u copia-config` (línea «Copia subida: … (tamaño)») |
| Windows | `Get-ScheduledTaskInfo "Copia diaria al NAS"` → `LastTaskResult : 0` (0 = sin errores) y el registro `C:\Copias-registro\ultima-copia.log` |
| NAS | Ver los ficheros con fecha de hoy en cada carpeta (fichero de la API de arriba) |

**Lo que no está automatizado:** las copias de los **routers** se hicieron a mano (solo cambian cuando alguien toca la configuración) y la **recuperación** (la rúbrica la deja desactivada; además, restaurar sin que una persona lo decida podría pisar cambios buenos).
