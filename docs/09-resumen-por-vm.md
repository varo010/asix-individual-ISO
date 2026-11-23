# Resumen por VM: qué hay en cada una y cómo explicarlo

Una ficha por máquina, para repasar antes de la defensa (23-11) y de la prueba final (25-11). Cada ficha dice **qué es**, **qué tiene configurado y por qué**, **qué se probó** y **qué enseñar** si lo preguntan. El detalle completo está en el documento enlazado.

## Mapa rápido

```
               Internet
                  │
        NAT de VirtualBox (10.0.2.0/24)
                  │
   ┌──────────────┴──────────────┐
   │  Router (uno encendido)     │  r-mikrotik · r-opnsense · r-nftables
   │  LAN 192.168.27.1           │  DHCP .100-.199 · DNS · NAT · cortafuegos
   └──────────────┬──────────────┘
                  │  red interna lan-iso (192.168.27.0/24)
   ┌──────┬───────┼────────┬────────────┬───────────┬─────────┬──────────┐
  NAS   srv-dc  srv-linux  pve1-3     srv-debian1/2  cli-win11  cli-linux
  .10    .13      .14     .11/.12/.15   .20/.21       DHCP       DHCP
```

**Orden para arrancar:** primero el **router** (sin él nadie tiene red), después el **NAS** (destino de las copias) y luego lo que toque. Con 32 GB de RAM caben las 9 VM a la vez con un router (sin contar `srv-debian1` y `srv-debian2`, que se añadieron el 23/11 con 1 GB cada una) ([T7](07-rendimiento.md)).

## Routers (solo uno encendido a la vez)

Los tres hacen lo mismo con la misma IP (192.168.27.1), para poder compararlos. → [05-comunicaciones](05-comunicaciones.md)

| | `r-mikrotik` | `r-opnsense` | `r-nftables` |
|---|---|---|---|
| Qué es | Router comercial (RouterOS) | Cortafuegos de código abierto (FreeBSD) | Debian sin escritorio hecho router «a mano» |
| Se administra | Web (WebFig) y terminal | Web | SSH y ficheros de texto |
| Usuario | `alvaro` (grupo *full*); `admin` desactivado | `alvaro` (grupo *admins*) además de `root` | `alvaro` con `sudo`; `root` sin SSH |
| Copia | Sube solo su backup cifrado al NAS | XML cifrado subido desde `cli-linux` | Sube solo un `.tar.gz` de 4 ficheros |
| Recuperación probada | Sí | Sí (~5 min 30 s) | Sí (**50 s**) |

**Qué decir:** «Un router une dos redes: la WAN (la NAT de VirtualBox) y mi LAN. Da IP por **DHCP**, resuelve nombres (**DNS**), hace **NAT** (todas las VM salen con su IP) y filtra con un **cortafuegos**: dejé pasar la web y bloqueé el DNS de fuera, y el registro (*log*) muestra los bloqueos. Rompí la NAT a propósito y la recuperé desde la copia del NAS.»

**Qué enseñar en `r-nftables`:** `ip -br a` (las dos tarjetas), `ip r` (la ruta por defecto), `cat /proc/sys/net/ipv4/ip_forward` (= 1, reenvía), `sudo nft list ruleset` (las reglas).

## `nas-truenas` · servidor de copias

→ [05-truenas](05-truenas.md) · [05-copias](05-copias.md)

- **TrueNAS 25.10**, 8 GB de RAM (el mínimo oficial), IP fija **192.168.27.10**.
- Disco aparte de 20 GB con el pool **`copias`** y **una carpeta (dataset) y un usuario por equipo**: `mikrotik`, `opnsense`, `nftables`, `srv-dc`, `srv-linux`, `cli-win11`, `cli-linux`, `proxmox`. Cada usuario solo entra en su carpeta y solo por **SFTP**.
- Los equipos con copia automática entran con **clave SSH** (sin contraseña guardada en ningún sitio).
- 21/11: cambiada la zona horaria a `Europe/Madrid` (venía la de Los Ángeles y las horas de las copias salían mal).

**Qué decir:** «El NAS guarda las copias de todos. Un usuario por equipo para que, si roban la clave de uno, solo pueda tocar sus copias. Ojo: el pool no tiene redundancia (un solo disco) y está en el mismo portátil: en una empresa haría falta otra copia fuera (regla 3-2-1).»

## `srv-dc` · Windows Server 2025, controlador de dominio

→ [05-servidores](05-servidores.md#windows-server-2025)

- IP fija **192.168.27.13**, DNS a sí mismo (127.0.0.1) y reenvío al router.
- **Dominio `iso27.lan`** (nombre corto `ISO27`): Active Directory + DNS.
- **Unidades organizativas:** `Empresa` → `Administradores`, `Usuarios`, `Equipos`, `Grupos`.
- **Usuarios:** `Administrator` (el interno), `alvaro` (Domain Admins, el de todos los días), `empleado1` (grupo `Contabilidad`, debe cambiar la contraseña al entrar).
- **GPO** «Usuarios - sin Panel de control», enlazada a la OU `Usuarios`.
- **Copia automática** diaria a las 02:30 (tarea programada como `SYSTEM`): base de datos de AD (IFM), SYSVOL, GPO y lista de usuarios en un `.zip` de 4,8 MB, al NAS por SFTP con clave. Guarda las 7 últimas.
- **Problemas resueltos:** zona horaria del Pacífico (Kerberos falla con más de 5 min de diferencia); el reinicio después de crear el dominio no se hizo solo; la clave SSH con permisos de más (`UNPROTECTED PRIVATE KEY`).

**Qué decir:** «Un dominio centraliza usuarios y equipos: una sola cuenta para entrar en cualquier equipo y las normas (GPO) se aplican desde aquí a todos. Uso `.lan` porque es un nombre interno que no existe en Internet.»

**Qué enseñar:** **Herramientas → Usuarios y equipos de Active Directory** (la estructura de OU), **Administración de directivas de grupo** (la GPO), `Get-ScheduledTaskInfo "Copia diaria al NAS"` (`LastTaskResult : 0`).

**Ojo:** no responde al botón de apagado de VirtualBox. Para apagarlo: iniciar sesión y `shutdown /s /t 0`. Para iniciar sesión, **Ctrl derecho + Supr** (no el Ctrl+Alt+Supr del teclado, que lo coge el portátil).

## `srv-linux` · servidor Debian de infraestructura

→ [05-servidores](05-servidores.md#servidor-gnulinux-de-infraestructura) · [06-raid](06-raid.md)

- Debian 13 sin escritorio, 1 GB, IP fija **192.168.27.14** (`/etc/network/interfaces`), repositorios por HTTPS con el de seguridad.
- Usuario `alvaro` con `sudo`; `root` **bloqueado** (`passwd -S root` → `L`). SSH, `mdadm` y `rsync`.
- **Copia automática** diaria a las 02:00 con un **temporizador de systemd**: `/etc` + lista de paquetes. Recuperación probada borrando `sources.list`: **32 s**.
- **T6:** 7 discos de 1 GB con **RAID 0, 1 y 5** (`/srv/raid0`, `/srv/raid1`, `/srv/raid5`) + un disco más para el **espejo DRBD** con `cli-linux`.
- **T7:** tarjeta de red cambiada a `virtio` (casi el doble de velocidad).

**Qué enseñar:** `cat /proc/mdstat` (los tres RAID `[UU]`, `[UU]`, `[UUU]`), `systemctl list-timers copia-config.timer`, `sudo drbdadm status espejo`.

**Ojo:** el espejo DRBD no arranca solo. Después de reiniciar: `sudo drbdadm up espejo` en `srv-linux` y en `cli-linux`, luego `sudo drbdadm primary espejo` y `sudo mount /dev/drbd0 /srv/espejo` **solo en uno**.

## `pve1`, `pve2`, `pve3` · clúster Proxmox

→ [05-proxmox](05-proxmox.md)

- Proxmox VE **9.2.32** (actualizado), 4 GB cada nodo, IP fijas **.11, .12, .15**, puente `vmbr0`. Repositorio `pve-no-subscription`.
- Usuario **`alvaro@pve`** con rol `Administrator` (además de `root@pam`). Web en `https://192.168.27.11:8006`.
- **Clúster `clusteriso`** con 3 nodos. **Quórum probado:** con 2 de 3 sigue funcionando; con 1 de 3 se bloquea (`Activity blocked`).
- **Solo para demostrar el clúster** (23/11, a petición del profesorado): ya no tienen máquinas dentro. El contenedor y la VM de la prueba se borraron y ahora son `srv-debian1` y `srv-debian2`, VM normales de VirtualBox.
- **Migraciones medidas (21/11, antes de borrarlas):** contenedor 25 s (parado ~22 s, porque se reinicia); VM **en caliente** 37 s (parada solo **42 ms**). Copian el disco entero (4 GB) porque el almacenamiento es local.
- **Copias:** configuración de cada nodo al NAS (diaria, 03:00, 24 KB) + una imagen `vzdump` del contenedor (159 MB).

**Qué decir:** «Tres nodos porque el quórum necesita mayoría: con dos, si cae uno, el otro se bloquea. Un contenedor comparte el núcleo del nodo (ligero, solo Linux, no se puede migrar en caliente); una VM es un ordenador entero (cualquier sistema, más aislada, se migra sin pararla). Con almacenamiento compartido (NFS en el NAS o Ceph) la migración no tendría que copiar el disco.»

**Qué enseñar:** `pvecm status` (3 votos, `Quorate`), la web con los tres nodos en **Datacenter**. `pct list` y `qm list` salen vacíos: es lo esperado.

## `srv-debian1` y `srv-debian2` · dos Debian independientes

→ [05-proxmox, apartado 5 bis](05-proxmox.md#5-bis-las-dos-máquinas-independientes-srv-debian1-y-srv-debian2)

- Debian 13.8 sin escritorio, 1 vCPU, 1 GB, disco 10 GB, tarjeta `virtio`. IP fijas **.20** y **.21**.
- Usuario `alvaro` con `sudo`; `root` bloqueado. SSH.
- Instaladas **sin preguntas** con un fichero de respuestas (*preseed*).

**Qué decir:** «Antes eran un contenedor y una VM dentro de Proxmox. Los profesores pidieron que fueran máquinas independientes, porque meter una VM dentro de otra VM (virtualización anidada) va más lenta y no aporta nada aquí. Proxmox se queda para el clúster.»

**Qué enseñar:** `ip -br a`, `ip r`, `cat /etc/resolv.conf`, `sudo apt update`.

## `cli-win11` · cliente Windows 11

→ [05-clientes](05-clientes.md#windows-11)

- Windows 11 Pro con **UEFI, Secure Boot y TPM 2.0 virtual** (sin TPM no se instala). Cuenta **local** `alvaro` (administrador), sin cuenta de Microsoft (`start ms-cxh:localonly`).
- Nombre `cli-win11`, zona horaria de Madrid, red por **DHCP** (.188).
- **Copia automática** a las 13:00 de Documentos, Escritorio e Imágenes. Tarea **«Recuperar ultima copia»** para restaurar: probada en **4 s**.

**Qué decir:** «Un cliente no lleva IP fija: nadie tiene que buscarlo a él. Solo se copian los documentos de la persona; Windows y los programas se reinstalan.»

## `cli-linux` · cliente Ubuntu

→ [05-clientes](05-clientes.md#cliente-gnulinux)

- Ubuntu Desktop 26.04.1 LTS, 4 vCPU y 4 GB (con menos iba al 100 % con las webs de los routers). Usuario `alvaro` con `sudo`. Red por DHCP (.136). SSH instalado.
- Es el equipo desde el que se usan las **webs** de OPNsense, TrueNAS y Proxmox.
- **Copia automática** a las 13:00 de `/home`, `/etc` y la lista de paquetes. Recuperación probada (menos de 1 s).
- Segundo equipo del **espejo DRBD** de la T6.

## Si en la prueba me piden…

| Me piden | Voy a | Dato clave |
|---|---|---|
| Recuperar un fichero o un sistema | [05-copias](05-copias.md) | Cada equipo tiene su forma de restaurar en la tabla de inventario |
| Resolver un problema de arranque o instalación | Los apartados *Problema* de cada documento | Secure Boot en casa ([INC-01](../incidencias/INC-01-virtualbox-secure-boot-casa.md)), TPM de Windows 11, reinicio del DC |
| Diagnosticar rendimiento | [07-rendimiento](07-rendimiento.md) | `free -m` (RAM disponible), `vmstat` (si `si/so` no son 0, falta RAM), `iostat` (disco) |
| Explicar el RAID | [06-raid](06-raid.md) | 0 = velocidad sin seguridad; 1 = espejo; 5 = paridad. **Un RAID no es una copia** |
