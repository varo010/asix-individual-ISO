# T5 · TrueNAS: servidor de datos y copias

RA: ISO RA1 (instalación y red), ISO RA3 (espacio de copias).

## Lo que pide la rúbrica

- [x] Instalar el sistema y comprobar versión y funcionamiento.
- [x] Usuario(s) de administración.
- [x] Parámetros de red de las interfaces.
- [x] Disco adicional (o partición) para las copias de los equipos críticos.
- [x] Preparar el espacio de copias y justificar qué se copia, cómo y dónde. *(Hecho para el MikroTik; faltan los demás equipos cuando estén instalados.)*

## 1. Requisitos y VM

**TrueNAS** es un sistema operativo para montar un **NAS** (*Network Attached Storage*): un equipo cuyo trabajo es guardar ficheros y compartirlos por la red. Uso la versión gratuita, **Community Edition 25.10.8** («Goldeye»). Guarda los datos con **ZFS**, un sistema de ficheros que comprueba que los datos no se estropeen y permite hacer instantáneas.

| Dato | Requisito oficial | Asignado en la VM `nas-truenas` |
|---|---|---|
| Procesador | 64 bits, 2 núcleos | 2 vCPU |
| RAM | 8 GB como mínimo | 8 GB |
| Disco de arranque | 16 GB como mínimo | 20 GB (`sda`) |
| Discos de datos | Al menos uno, **distinto** del de arranque | 1 de 20 GB (`sdb`) |
| Arranque | UEFI o BIOS | UEFI |

TrueNAS no deja guardar datos en el disco donde está instalado: por eso la VM lleva **dos discos**. Pide tanta RAM porque ZFS usa la memoria libre como caché de los discos (se ve en el Dashboard, más abajo).

## 2. Instalación

1. Arranqué la VM con la ISO oficial (hash comprobado en [T0](00-entorno.md)) y elegí **Install/Upgrade**.
2. Disco de destino: **`sda`** (20 GB). El `sdb` lo dejé libre para los datos.
3. Usuario de administración: `truenas_admin`, con contraseña (ver punto 3).
4. Al acabar, reinicié.

**Problema:** después de reiniciar **volvió a salir el instalador** en vez del sistema instalado.
- **Causa:** la ISO seguía metida en el lector de la VM y, con arranque UEFI, VirtualBox arrancaba antes desde el DVD que desde el disco.
- **Solución:** sacar la ISO del lector virtual con la VM apagada (`VBoxManage storageattach … --medium emptydrive --forceunmount`) y volver a arrancar. Desde entonces arranca desde el disco.
- **Lección:** después de instalar cualquier sistema en una VM, **sacar la ISO**.

**Versión y funcionamiento** (Dashboard de la web): → [versión](<../evidencias/t5-truenas-03-dashboard-version-20261117.png>) · [CPU](<../evidencias/t5-truenas-04-dashboard-cpu-20261117.png>) · [memoria](<../evidencias/t5-truenas-05-dashboard-memoria-20261117.png>) · [red](<../evidencias/t5-truenas-06-dashboard-red-20261117.png>)

| Gráfica | Valor | Qué significa |
|---|---|---|
| Versión | 25.10.8 Community | La instalada |
| CPU | 1 % | Sin trabajo, como debe estar un NAS en reposo |
| Memoria | 7,7 GiB: 3,0 libres, 3,5 de **caché ZFS**, 1,2 de servicios | ZFS usa la RAM sobrante para leer los discos más rápido. No está «gastada»: la devuelve si otro programa la necesita |
| Red | `enp0s3`, 192.168.27.10, *Link State Up* | La tarjeta está conectada con su IP fija |

**Actualizaciones** (**System → Update**): `Current version: 25.10.8` y **`System is up to date!`**, así que no hay nada que instalar. El canal de actualizaciones es **General**, el más conservador: solo trae versiones ya probadas. Los canales *Early Adopter* y *Developer* traen novedades antes pero con más fallos, y no tiene sentido usarlos en un equipo que guarda copias. → [captura](<../evidencias/t5-truenas-07-actualizaciones-20261117.png>)

## 3. Usuarios

| Usuario | Para qué | Accesos | Captura |
|---|---|---|---|
| `root` | Usuario interno del sistema, viene de serie | *Full Admin*; no lo uso | |
| `truenas_admin` | Administrarlo yo por la web | *Full Admin* | |
| `mikrotik` | Que el router deje sus copias | **Solo SSH/SFTP**, sin web, sin SMB y sin `sudo` | [lista de usuarios](<../evidencias/t5-truenas-10-usuario-mikrotik-20261117.png>) |

**Por qué un usuario solo para el router:** por la misma razón que en el MikroTik desactivé `admin`: cada uno entra con su usuario y **solo con los permisos que necesita**. Si alguien robara la contraseña de `mikrotik`, solo podría tocar la carpeta de las copias del router, no administrar el NAS.

Cómo lo creé (**Credentials → Users → Add**):
- **Username** `mikrotik`, **Full Name** `Copias del router`.
- **Allow Access**: solo **SSH Access**, con **Allow SSH Login with Password**. *Shell Access* se marca solo al elegir SSH y no se puede quitar.
- **Home Directory**: `/mnt/copias/mikrotik`, la carpeta del dataset (punto 5).
- **Groups**: TrueNAS crea un grupo `mikrotik` para él, como hace Linux normalmente.

**Mejora posible:** TrueNAS avisa de que entrar por SSH con contraseña es *not recommended*. En una empresa se usaría una **clave SSH** en lugar de contraseña. En el laboratorio uso contraseña para que sea más sencillo.

## 4. Red

| Dato | Valor | Por qué |
|---|---|---|
| Interfaz | `enp0s3`, en la red interna `lan-iso` | Es la LAN del router |
| IP | **192.168.27.10/24**, fija | Un servidor necesita una IP que no cambie: el router y los equipos tienen que saber siempre dónde dejar las copias |
| Puerta de enlace | 192.168.27.1 | El router |
| DNS | 192.168.27.1 | El router (que reenvía las preguntas) |
| Nombre | `nas-truenas`, dominio `local` | |

**Cómo lo hice:**
1. En el primer arranque, la consola mostró la IP que le dio el DHCP del router: **192.168.27.198**. → [captura](<../evidencias/t5-truenas-01-consola-ip-dhcp-20261117.png>)
2. Desde Firefox en `cli-linux` entré a `http://192.168.27.198` con `truenas_admin`.
3. **System → Network → Global Configuration**: nombre, dominio, puerta de enlace y DNS.
4. **Interfaces → enp0s3**: quité el DHCP y puse la IP fija 192.168.27.10/24. → [captura](<../evidencias/t5-truenas-02-red-ip-fija-20261117.png>)
5. **Test Changes**: TrueNAS aplica el cambio **solo durante 60 segundos**. Si en ese tiempo no entras por la IP nueva y pulsas **Save Changes**, vuelve solo a la configuración anterior. Así no te quedas sin acceso si te equivocas de IP. Entré por `http://192.168.27.10` y guardé.

La .10 está **fuera del rango del DHCP** (.100–.199): así el router nunca se la podrá dar a otro equipo.

## 5. Disco y espacio de copias

**Pool `copias`** (**Storage → Create Pool**). Un **pool** es el «almacén» de ZFS: junta uno o varios discos y los datos se guardan ahí. → [captura](<../evidencias/t5-truenas-08-pool-copias-20261117.png>)

| Opción | Valor | Por qué |
|---|---|---|
| Nombre | `copias` | |
| Disco | `sdb` (20 GB) | El de datos, no el de arranque |
| Distribución (*Layout*) | **Stripe** | Con un solo disco no hay otra opción |
| Cifrado | No | Laboratorio; en una empresa sí lo pondría |

| Lo que se ve | Qué significa |
|---|---|
| `1 x DISK, 1 wide, 20 GiB` con un aviso naranja | El aviso no es un error: dice que el pool **no tiene redundancia** |
| 18,89 GiB útiles de 20 | El disco se vende en GB (1000) y TrueNAS cuenta en GiB (1024), que son más grandes; además ZFS se reserva un poco para funcionar |
| *Online, no errors* | El pool está sano |
| *Scheduled Scrub*: domingos a las 00:00 | Cada semana ZFS lee todos los datos y comprueba que no se han estropeado. Viene programado de serie |
| Sin temperatura del disco | Normal en una VM: el disco virtual no tiene sensor |

⚠️ **Riesgo del Stripe:** si el disco `sdb` falla, **se pierden todas las copias**. Con dos discos se podría hacer un **espejo** (*mirror*, como un RAID 1): cada dato se guarda en los dos discos. Lo veremos en la [T6 (RAID)](06-raid.md). El grupo tiene el mismo riesgo con su HDD de 2 TB.

**Dataset `copias/mikrotik`** (**Datasets → copias → Add Dataset**, *preset* **Generic**). Un **dataset** es como una carpeta dentro del pool, pero con sus propios permisos y opciones. Hago uno por cada equipo para no mezclar las copias. No reserva espacio fijo: va cogiendo del pool lo que necesita. → [captura](<../evidencias/t5-truenas-09-dataset-mikrotik-20261117.png>)

**Servicio SSH** (**System → Services**): **Running** y **Start Automatically**, para que arranque solo al encender el NAS. El resto de servicios (SMB, NFS, FTP…) siguen parados: en un servidor solo se encienden los servicios que se usan, porque cada servicio abierto es una puerta más para entrar. → [captura](<../evidencias/t5-truenas-11-servicio-ssh-20261117.png>)

**Prueba desde `cli-linux`** antes de usarlo con el router, para saber que el NAS funciona por sí solo:

```
sftp mikrotik@192.168.27.10
```

1. Primera conexión: SSH enseña la **huella** del NAS (`ED25519 key fingerprint is SHA256:js+p3…`) y pregunta si me fío. Contesto `yes` y el equipo la apunta. Si un día otro equipo se hiciera pasar por 192.168.27.10, la huella no coincidiría y SSH avisaría. → [captura](<../evidencias/t5-truenas-12-sftp-conexion-20261117.png>)
2. Dentro, compruebo que puedo escribir: → [captura](<../evidencias/t5-truenas-13-sftp-prueba-escritura-20261117.png>)

| Comando | Resultado | Qué demuestra |
|---|---|---|
| `pwd` | `/mnt/copias/mikrotik` | El usuario entra directamente en su carpeta |
| `put /etc/hostname prueba.txt` | 100 %, 7 bytes | Puede **subir** ficheros |
| `ls -l` | `-rw-r--r-- mikrotik mikrotik … prueba.txt` | El fichero es suyo; no hay errores de permisos |

Luego borré `prueba.txt`.

**Problema durante la prueba:** dentro de la VM no podía escribir la `@`. La **tecla Host** de VirtualBox (la que usa VirtualBox para sus atajos, por ejemplo soltar el ratón) estaba en **AltGr**, y VirtualBox no la pasaba a la VM. La ventana de Preferencias no dejaba cambiarla, así que la cambié a **Bloq Mayús** con `VBoxManage setextradata global GUI/Input/HostKeyCombination 65509`. Hay que cerrar y volver a abrir la ventana de la VM para que lo note.

## 6. Qué se copia, cómo y dónde

| Equipo crítico | Qué se copia | Cómo llega al NAS | Dónde | Por qué |
|---|---|---|---|---|
| Router MikroTik | Backup completo y cifrado (`.backup`) y configuración en texto (`.rsc`) | Lo sube el propio router por SFTP con `/tool fetch` (detalle en [T5 comunicaciones](05-comunicaciones.md#copia-de-la-configuración)) | `copias/mikrotik` | Sin router no hay Internet ni DHCP en la LAN: es lo primero que hay que poder recuperar |
| Router OPNsense | Configuración completa en XML **cifrado** | Se descarga con el navegador en `cli-linux` y se sube por SFTP con el usuario `opnsense` ([detalle](05-comunicaciones.md#copia-de-la-configuración-1)) | `copias/opnsense` | Igual que el MikroTik: es la salida a Internet de toda la LAN |
| Router Debian + nftables | Paquete `.tar.gz` con sus 4 ficheros de configuración (reglas, DHCP, red e `ip_forward`) | Lo sube el propio router por SFTP con el usuario `nftables` ([detalle](05-comunicaciones.md#copia-de-la-configuración-2)) | `copias/nftables` | Igual que los otros routers: es la salida a Internet de toda la LAN |
| Windows Server | Pendiente | | | |

**Prueba de que la copia sirve:** borré la NAT del MikroTik y sus ficheros, bajé el backup desde este NAS y lo restauré. La LAN volvió a tener Internet ([fallo y recuperación](05-comunicaciones.md#fallo-y-recuperación)).

## 7. Comparación con el grupo

| Aspecto | Grupo 3 | Mi entorno |
|---|---|---|
| Equipo | PC físico Ticnova E70 (8 GB RAM) | VM con 2 vCPU y 8 GB |
| Disco de datos | HDD de 2 TB en stripe (sin redundancia) | Disco virtual de 20 GB en stripe (sin redundancia) |
| Versión | TrueNAS 25.10.7 | TrueNAS 25.10.8 |
| Riesgo | Si el HDD falla, se pierden las copias | Igual, y además todo está en el mismo portátil que los routers: si el portátil falla, se pierde todo. En una empresa, las copias tienen que estar en **otro equipo** y mejor también **fuera del edificio** |
