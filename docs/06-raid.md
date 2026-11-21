# T6 · RAID

RA: ISO RA3. Con discos virtuales y datos ficticios.

## Lo que pide la rúbrica

- [x] Crear y verificar al menos tres tipos de RAID.
- [x] Explicar qué aporta cada tipo.
- [x] Ampliación: investigar y justificar un espejo entre discos de dos equipos distintos.
- [x] Ampliación: implementarlo con la solución de replicación o almacenamiento distribuido elegida.
- [x] Ampliación: verificar el funcionamiento y explicar un caso de uso.

## 0. Qué es un RAID

Un **RAID** junta varios discos para que el sistema los vea como **uno solo**. Según cómo reparta los datos, se gana **velocidad**, **seguridad ante la avería de un disco**, o las dos cosas. Lo hago por **software** con `mdadm`, la herramienta de Linux: no hace falta una tarjeta RAID especial.

> **Un RAID no es una copia de seguridad.** Protege de que se **rompa un disco**, pero si alguien **borra un fichero** o entra un virus, el cambio se hace en todos los discos a la vez. Las copias van aparte, al NAS ([05-copias](05-copias.md)).

## 1. Entorno

| VM | SO | Discos virtuales añadidos | Herramienta |
|---|---|---|---|
| `srv-linux` | Debian 13.7 sin escritorio | **7 discos de 1 GB** (`sdb` … `sdh`): 4 creados el 15/11 y 3 más el 21/11. Uno más para el espejo del apartado 4 | `mdadm` 4.4 |

Hacen falta 7 discos para tener los tres RAID **a la vez**: 2 (RAID 0) + 2 (RAID 1) + 3 (RAID 5). Los 3 discos nuevos se añadieron con la VM apagada:

```
VBoxManage storagectl srv-linux --name SATA --portcount 8
VBoxManage createmedium disk --filename srv-linux-raid5.vdi --size 1024
VBoxManage storageattach srv-linux --storagectl SATA --port 5 --device 0 --type hdd --medium srv-linux-raid5.vdi
```

(lo mismo para los discos 6 y 7). → [discos antes de crear el RAID](<../evidencias/t6-raid-01-discos-antes-20261121.png>)

## 2. RAID creados

```
sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
sudo mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sdd /dev/sde
sudo mdadm --create /dev/md5 --level=5 --raid-devices=3 /dev/sdf /dev/sdg /dev/sdh
cat /proc/mdstat
```

| Parte | Qué significa |
|---|---|
| `/dev/md0` | El disco nuevo que crea el RAID (`md` = *multiple devices*) |
| `--level=0/1/5` | El tipo de RAID |
| `--raid-devices=N` | Cuántos discos lo forman |
| `/proc/mdstat` | Fichero del núcleo que enseña el estado de todos los RAID |

En `mdstat`, `[UU]` quiere decir que los dos discos están bien (*Up*); `[_U]`, que falta el primero. Al crear el RAID 1 y el RAID 5 sale `resync` / `recovery`: **sincroniza** los discos (copia o calcula los datos de seguridad) antes de estar completos. → [captura](<../evidencias/t6-raid-02-crear-raid0-raid1-raid5-20261121.png>)

**Formatear, montar y que sobrevivan a un reinicio:** → [captura](<../evidencias/t6-raid-03-formatear-montar-persistir-20261121.png>)

```
sudo mkfs.ext4 -L raid0 /dev/md0                       # sistema de ficheros
sudo mkdir -p /srv/raid0                                 # carpeta donde se verá
echo "/dev/md0 /srv/raid0 ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

| Paso | Qué pasa si falta |
|---|---|
| `mkfs.ext4` | El RAID existe pero no se pueden guardar ficheros |
| Línea en `/etc/fstab` | Al reiniciar el RAID no se monta solo. `nofail` = si el RAID falla, el servidor arranca igual |
| `mdadm.conf` + `update-initramfs` | Al reiniciar el RAID podría aparecer con otro nombre (`md127`) y la línea de `fstab` no lo encontraría |

**Datos ficticios:** en cada RAID, tres ficheros de 20 MB con datos aleatorios (`clientes.dat`, `facturas.dat`, `nominas.dat`) y su **huella SHA256** guardada en `sumas.sha256`. Con `sha256sum -c` se comprueba después que no ha cambiado ni un bit. → [captura](<../evidencias/t6-raid-04-datos-ficticios-y-velocidad-20261121.png>)

### Resultados

| Tipo | Discos | Capacidad útil | Tolera la caída de | Velocidad de escritura medida | Prueba de fallo | Resultado | Evidencia |
|---|---|---|---|---|---|---|---|
| **RAID 0** (reparto) | 2 × 1 GB | **2 GB** (100 %) | **Ningún disco** | **1,5 GB/s** | Borro un disco entero (`dd if=/dev/zero of=/dev/sdc`) | El RAID **no se puede montar**: `no RAID superblock on /dev/sdc - assembly aborted`. **Se pierde todo** | [fallo](<../evidencias/t6-raid-07-raid0-un-disco-roto-todo-perdido-20261121.png>) |
| **RAID 1** (espejo) | 2 × 1 GB | **1 GB** (50 %) | 1 disco | 543 MB/s | `mdadm /dev/md1 --fail /dev/sdd --remove /dev/sdd` | Sigue funcionando con un disco (`[_U]`) y los datos están **intactos** («La suma coincide») | [fallo](<../evidencias/t6-raid-05-fallo-disco-datos-intactos-20261121.png>) |
| **RAID 5** (paridad) | 3 × 1 GB | **2 GB** (66 %) | 1 disco | 734 MB/s | `mdadm /dev/md5 --fail /dev/sdg --remove /dev/sdg` | Sigue funcionando con dos discos (`[U_U]`) y los datos están **intactos** | [fallo](<../evidencias/t6-raid-05-fallo-disco-datos-intactos-20261121.png>) |

La velocidad se midió escribiendo 200 MB sin caché (`dd … oflag=direct`). Son discos virtuales en el mismo disco USB, así que sirven para **comparar entre ellos**, no como velocidad real.

**Cambio del disco averiado** (RAID 1 y RAID 5): se pone un disco «nuevo» (`wipefs` lo deja como recién comprado) y se añade:

```
sudo mdadm /dev/md1 --add /dev/sdd
sudo mdadm /dev/md5 --add /dev/sdg
```

El RAID **se reconstruye solo** (`recovery`) y en menos de 25 segundos vuelve a `[UU]` y `[UUU]`, `State : clean`, `Failed Devices : 0`. → [captura](<../evidencias/t6-raid-06-disco-nuevo-reconstruido-20261121.png>)

**RAID 0 después del desastre:** solo se puede **crear de nuevo vacío** y volver a meter los datos **desde una copia** (aquí, desde el RAID 1). Las huellas coinciden: los datos vuelven gracias a la copia, no al RAID. → [captura](<../evidencias/t6-raid-08-raid0-rehecho-datos-desde-copia-20261121.png>)

**Después de reiniciar** el servidor los tres RAID se montan solos, con sus datos. Detalle curioso: los discos cambiaron de letra (`sdh` aparece en el RAID 0). No pasa nada porque `mdadm` reconoce los discos por su **UUID** (un identificador escrito dentro de cada disco), no por la letra. → [captura](<../evidencias/t6-raid-09-persisten-tras-reiniciar-20261121.png>)

## 3. Qué aporta cada tipo

| Tipo | Cómo guarda los datos | Aporta | Inconveniente | Uso típico |
|---|---|---|---|---|
| **RAID 0** | Parte cada fichero en trozos y escribe un trozo en cada disco a la vez | **Velocidad** (en la prueba, el triple que el RAID 1) y toda la capacidad | **Ninguna seguridad**: si cae un disco se pierde **todo** (con 2 discos el riesgo es el doble que con uno) | Datos temporales que se pueden volver a generar: edición de vídeo, cachés |
| **RAID 1** | Escribe **lo mismo en los dos discos** | **Seguridad**: aguanta la caída de un disco; lectura rápida | Se pierde la **mitad** de la capacidad | Disco del sistema de un servidor, datos pequeños e importantes |
| **RAID 5** | Reparte los datos y añade **paridad** (un cálculo que permite reconstruir lo que falte) repartida entre todos los discos | Seguridad ante la caída de un disco **perdiendo solo un disco** de capacidad (con 3 discos, se aprovecha el 66 %) | Escribe más lento que el RAID 0 (hay que calcular la paridad); mientras se reconstruye, si cae otro disco se pierde todo | Servidores de ficheros y NAS con 3 o más discos |

**Qué elegiría en la empresa:** RAID 1 para el disco del sistema de los servidores y RAID 5 (o RAID 6 / RAIDZ2 si hay muchos discos) para los datos. **Nunca RAID 0 para datos importantes.** Y siempre, además, copias en el NAS.

## 4. Ampliación: espejo entre dos equipos

**La idea:** un RAID 1 normal protege de que se rompa **un disco**, pero si se rompe **el servidor entero** (placa, fuente, o se apaga), los dos discos se quedan dentro y no hay datos. Un **espejo entre dos equipos** guarda una copia exacta en **otro ordenador** a la vez, por la red.

### Opciones investigadas

| Opción | Cómo funciona | A favor | En contra | ¿La elijo? |
|---|---|---|---|---|
| **DRBD** | Un «RAID 1 por la red»: cada escritura se hace en el disco local **y** en el del otro equipo antes de darla por buena | Viene en el núcleo de Linux; es lo que usa Proxmox y muchas empresas para alta disponibilidad con 2 servidores | Solo un equipo puede usar los datos a la vez (el *primario*) | **Sí** |
| `mdadm` RAID 1 + disco remoto por iSCSI/NBD | Se presta un disco del otro equipo por la red y se hace un RAID 1 normal con él | Usa lo mismo que el apartado 2 | Si cae la red, `mdadm` marca el disco como roto y hay que reconstruirlo **entero** | No |
| **GlusterFS / Ceph** | Almacenamiento distribuido entre 3 o más equipos | Varios equipos usan los datos a la vez; crece añadiendo equipos | Necesita 3 equipos o más y mucha RAM. Demasiado para el portátil ([T7](07-rendimiento.md)) | No |
| **`rsync` / replicación ZFS programada** | Copia cada cierto tiempo | Muy sencillo | **No es un espejo**: se pierde lo escrito desde la última copia | No |

**Por qué DRBD:** es un espejo de verdad (no se pierde nada escrito), funciona con **solo 2 equipos** y solo vuelve a copiar **lo que ha cambiado** cuando un equipo vuelve.

### Implementación

| | `srv-linux` | `cli-linux` |
|---|---|---|
| Disco nuevo | 1 GB (añadido con la VM apagada) | 1 GB (añadido con la VM apagada) |
| Programa | `drbd-utils` (DRBD 8.4.11 del núcleo) | `drbd-utils` (DRBD 8.4.11 del núcleo) |
| IP y puerto | 192.168.27.14:7789 | 192.168.27.136:7789 |
| Papel al empezar | **Primario** (usa los datos) | **Secundario** (recibe la copia) |

Fichero `/etc/drbd.d/espejo.res`, **igual en los dos equipos**:

```
resource espejo {
  protocol C;
  disk { c-plan-ahead 0; resync-rate 100M; }
  device    /dev/drbd0;
  meta-disk internal;
  on srv-linux {
    disk    /dev/disk/by-id/ata-VBOX_HARDDISK_VB7a805efa-e21ed0c0;
    address 192.168.27.14:7789;
  }
  on cli-linux {
    disk    /dev/disk/by-id/ata-VBOX_HARDDISK_VB57eb120f-6f57ee3b;
    address 192.168.27.136:7789;
  }
}
```

| Línea | Qué significa |
|---|---|
| `protocol C` | Una escritura **no se da por terminada** hasta que está en los **dos** discos. Es el modo seguro: si cae un equipo, el otro tiene todo |
| `device /dev/drbd0` | El disco «espejo» que se usa (se formatea y se monta como cualquier disco) |
| `disk /dev/disk/by-id/…` | El disco real de cada equipo, por su **número de serie** y no por la letra (`sdh`), que cambia al reiniciar (lo vimos en el apartado 2) |
| `meta-disk internal` | DRBD guarda sus datos de control al final del mismo disco |
| `resync-rate 100M` | **Problema:** la primera sincronización iba a **364 KB/s (43 minutos)** por el límite que trae de serie. Subiendo el límite tardó **menos de 2 minutos** |

**Órdenes** (en los dos equipos, salvo lo marcado):

```
sudo apt install drbd-utils
sudo drbdadm create-md espejo        # prepara el disco
sudo drbdadm up espejo               # lo activa y se conecta con el otro equipo
# solo en srv-linux, la primera vez:
sudo drbdadm primary --force espejo  # «mis datos son los buenos»: copia al otro
sudo mkfs.ext4 -L espejo /dev/drbd0
sudo mount /dev/drbd0 /srv/espejo
```

`cat /proc/drbd` o `drbdadm status` enseñan el estado: `Connected` (los dos se ven), `Primary/Secondary` (quién usa los datos) y `UpToDate/UpToDate` (los dos discos iguales). → [captura](<../evidencias/t6-espejo-01-drbd-sincronizado-datos-en-srvlinux-20261121.png>)

Datos de prueba: los mismos tres ficheros de 20 MB que en el apartado 2, más `leeme.txt`, con sus huellas SHA256.

### Verificación: se cae el servidor entero

| Hora | Qué hice | Resultado | Evidencia |
|---|---|---|---|
| 22:39 | Datos escritos en `srv-linux` (primario) | Los dos discos `UpToDate` | [captura](<../evidencias/t6-espejo-01-drbd-sincronizado-datos-en-srvlinux-20261121.png>) |
| 22:40:10 | **Apago `srv-linux` de golpe** (`VBoxManage controlvm srv-linux poweroff`, como quitarle el cable) | En `cli-linux`: `WFConnection` (espera al otro), pero su disco sigue `UpToDate` | |
| 22:40:28 | En `cli-linux`: `drbdadm primary espejo` y `mount` | **Los datos están todos** y las 4 huellas dan `OK`. El cambio de equipo tarda **menos de 1 segundo** (los 18 s anteriores son lo que tardé en escribir las órdenes) | [captura](<../evidencias/t6-espejo-02-srvlinux-caido-datos-en-clilinux-20261121.png>) |
| 22:40 | Mientras `srv-linux` está caído, se escribe `pedido-nuevo.txt` en `cli-linux`. Lo desmonto y lo dejo como secundario | | |
| 22:41 | Enciendo `srv-linux` y `drbdadm up espejo` | DRBD sabe qué ha cambiado y **solo copia eso** (116 MB, no el disco entero) | |
| 22:45 | `srv-linux` vuelve a ser primario | Tiene el `pedido-nuevo.txt` escrito mientras estaba apagado, y las huellas coinciden | [captura](<../evidencias/t6-espejo-03-srvlinux-vuelve-y-recibe-cambios-20261121.png>) |

**No se monta solo al arrancar** (no está en `/etc/fstab` y el servicio `drbd` está desactivado) **a propósito**: si los dos equipos se pusieran como primario a la vez, cada uno escribiría cosas distintas (*split brain*, el mismo problema que evita el quórum de [Proxmox](05-proxmox.md#prueba-de-quórum)). Decidir quién es el primario lo hace una persona o un programa de alta disponibilidad (Pacemaker).

### Caso de uso

**Servidor de ficheros de una oficina con dos servidores.** Los documentos de la empresa están en un espejo DRBD entre el servidor principal y uno de reserva, en **otra sala**. Si el principal se estropea (placa, fuente, incendio en esa sala), el de reserva se pone como primario y en un minuto la oficina vuelve a tener sus documentos, **sin perder nada** de lo escrito hasta el último segundo.

| | RAID 1 (apartado 2) | Espejo DRBD | Copia en el NAS ([05-copias](05-copias.md)) |
|---|---|---|---|
| Se rompe un disco | ✅ | ✅ | ✅ (pierde lo del último día) |
| Se rompe el servidor entero | ❌ | ✅ | ✅ (pierde lo del último día) |
| Borro un fichero sin querer / virus | ❌ (se borra en los dos) | ❌ (se borra en los dos) | ✅ |

**Por eso el espejo tampoco sustituye a las copias:** protege de averías, no de errores ni de virus.
