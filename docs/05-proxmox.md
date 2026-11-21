# T5 · Proxmox VE: virtualización y clúster

RA: ISO RA1 (instalación, red, clúster, migración), ISO RA3 (copias), ISO RA6 (medidas de la migración).

## Lo que pide la rúbrica

- [x] Instalar y comprobar versión y funcionamiento.
- [x] Usuario(s) de administración.
- [x] Parámetros de red de las interfaces.
- [x] Clúster con los nodos disponibles (si no hay suficientes, diseño o simulación autorizada).
- [x] Migrar un contenedor y una VM (Debian/Ubuntu Server **sin** entorno gráfico) entre nodos.
- [x] Medir tiempo y espacio de cada migración, interpretarlo y valorar mejoras.
- [x] Copiar configuraciones/imágenes al servidor de copias y justificar qué sí y qué no.
- [x] Explicar ventajas e inconvenientes de virtualizar servidores.
- [x] Explicar la diferencia entre contenedores y VM, y cuándo usar cada uno.

## 0. Qué es Proxmox

**Proxmox VE** es un sistema operativo (basado en Debian) que sirve **para crear y gestionar máquinas virtuales y contenedores**, igual que VirtualBox pero pensado para servidores: se maneja desde una **web** (`https://IP:8006`) y varios servidores se pueden juntar en un **clúster** para administrarlos como si fueran uno.

Aquí lo ejecuto **dentro de VirtualBox** (Proxmox *anidado*): son VM que a su vez crean VM. Va más lento que en un servidor de verdad, pero sirve para probar el clúster y la migración.

## 1. Nodos

En el clúster solo van las pruebas de Proxmox (un contenedor y una VM Debian sin escritorio). **Windows Server no va dentro del clúster**: es una VM aparte, `srv-dc` ([05-servidores](05-servidores.md)).

| Nodo | vCPU | RAM | Disco | IP | Virtualización anidada |
|---|---|---|---|---|---|
| `pve1` | 2 | 4 GB | 32 GB | 192.168.27.11 | Sí |
| `pve2` | 2 | 4 GB | 32 GB | 192.168.27.12 | Sí |
| `pve3` | 2 | 4 GB | 32 GB | 192.168.27.15 | Sí |

**Por qué 3 nodos y no 2:** por el quórum (explicado en el punto 4). Al principio el diseño tenía 2 para ahorrar RAM; el 18/11 añadí `pve3` ([cambio de diseño](00-entorno.md#cambio-de-diseño-de-2-a-3-nodos-de-proxmox-18112026)).

## 2. Instalación

- **ISO oficial** de Proxmox VE 9.2 con su huella comprobada ([00-entorno](00-entorno.md)). Instalados los tres nodos el 18/11. → [resumen](<../evidencias/t5-proxmox-01-resumen-instalacion-pve1-20261118.png>) · [pve1](<../evidencias/t5-proxmox-02-pve1-instalado-20261118.png>) · [pve2](<../evidencias/t5-proxmox-03-pve2-instalado-20261118.png>) · [pve3](<../evidencias/t5-proxmox-04-pve3-instalado-20261118.png>)
- **Disco:** el instalador crea **LVM** (gestor de volúmenes: reparte el disco en trozos que se pueden agrandar):

| Volumen | Tamaño | Para qué |
|---|---|---|
| `pve-root` (ext4) | 13,8 GB | El sistema. Ahí se ve como almacenamiento `local`: ISO, plantillas y copias |
| `pve-data` (*LVM-thin*) | 11,8 GB | Almacenamiento `local-lvm`: los discos de las VM y contenedores. *Thin* = solo ocupa lo que se usa de verdad |
| `pve-swap` | 3,8 GB | Memoria de intercambio |

- **Repositorios y actualización (21/11):** Proxmox viene con el repositorio **enterprise** (de pago, necesita suscripción) y sin él `apt` da error. Lo cambié por el **`pve-no-subscription`** (gratuito, el que se usa para pruebas) y desactivé el de Ceph enterprise (`Enabled: no`). Después `apt dist-upgrade`: **225 paquetes**, de la 9.2.2 a la **9.2.32**, y reinicio para usar el núcleo nuevo (`7.0.14-30-pve`).

```
pveversion
pve-manager/9.2.32/60a22e26e411402f (running kernel: 7.0.14-30-pve)
```

## 3. Usuarios y red

**Usuarios** → [captura](<../evidencias/t5-proxmox-05-version-usuario-red-pve1-20261121.png>)

| Usuario | Tipo (*realm*) | Para qué |
|---|---|---|
| `root@pam` | Usuario de Linux del nodo (`pam`) | Solo para emergencias y para tareas de sistema (actualizar, crear el clúster) |
| `alvaro@pve` | Usuario **propio de Proxmox** (`pve`), rol **Administrator** en `/` (todo) | El de todos los días en la web. Es **nominal**: si hace algo, se sabe quién fue |

```
pveum user add alvaro@pve --comment "Administrador (cuenta nominal)"
pveum passwd alvaro@pve
pveum acl modify / --users alvaro@pve --roles Administrator
```

Al juntar los nodos en el clúster, **los usuarios se comparten**: `alvaro@pve` sirve en los tres.

**Red** (fichero `/etc/network/interfaces` de cada nodo):

```
iface nic0 inet manual
iface vmbr0 inet static
	address 192.168.27.11/24
	gateway 192.168.27.1
	bridge-ports nic0
```

| Parte | Qué es |
|---|---|
| `nic0` | La tarjeta de red «física» del nodo. **No tiene IP**: solo está enchufada al puente |
| `vmbr0` | Un **puente** (*bridge*): un switch virtual dentro del nodo. Tiene la IP del nodo y a él se conectan las VM y contenedores, que así salen a la LAN como un equipo más |
| `address` / `gateway` | IP fija del nodo (los servidores llevan IP fija) y salida por el router |
| DNS | `192.168.27.1` (el router), en `/etc/resolv.conf` |

Además, en `/etc/hosts` de cada nodo están los **tres** nodos con su nombre: el clúster los busca por nombre.

**Problema:** el puente necesita que la tarjeta de VirtualBox acepte tráfico de otras MAC (las de las VM de dentro). Por eso las tarjetas de los nodos tienen el **modo promiscuo en «Permitir todo»** (`--nicpromisc1 allow-all`).

## 4. Clúster

**Quórum con 3 nodos:** cada nodo tiene 1 voto y hacen falta **más de la mitad** (2 de 3) para que el clúster funcione.

| Nodos encendidos | Votos | ¿Hay quórum? | Qué pasa |
|---|---|---|---|
| 3 de 3 | 3 | Sí | Todo normal |
| 2 de 3 | 2 | **Sí** (mayoría) | El clúster sigue funcionando sin el nodo caído |
| 1 de 3 | 1 | No | El nodo que queda se bloquea: no deja arrancar ni cambiar VM |

Con **2 nodos**, si cae uno queda 1 de 2, que no es mayoría: por eso el diseño inicial de 2 no permitía alta disponibilidad.

**Tres cosas distintas que se confunden:**
- **Gestión conjunta:** desde la web de cualquier nodo veo y administro los tres.
- **Migración:** **yo** muevo una VM o un contenedor de un nodo a otro.
- **Alta disponibilidad (HA):** si un nodo cae, **el clúster solo** arranca sus VM en otro nodo. Necesita quórum después de la caída, así que hacen falta al menos 3 nodos.

### Cómo se creó

```
# en pve1: crear el clúster
pvecm create clusteriso
# en pve2 y en pve3: unirse a pve1
pvecm add 192.168.27.11 --use_ssh
# comprobar
pvecm status
```

`--use_ssh` = el nodo nuevo se une usando SSH con clave de `root` (antes copié la clave de cada nodo a los otros dos).

**Problema:** la primera vez que `pve2` intentó unirse falló con `unable to copy ssh ID: Permission denied`. Fue porque lo lancé justo después de crear el clúster, mientras `pve1` reiniciaba su sistema de ficheros del clúster (`/etc/pve`), que durante esos segundos no deja escribir. Al repetirlo, entró sin problema.

Resultado: **3 nodos, 3 votos, quórum 2, `Quorate`** (= tiene quórum). → [captura](<../evidencias/t5-proxmox-06-cluster-3-nodos-quorum-20261121.png>)

### Prueba de quórum

→ [captura](<../evidencias/t5-proxmox-09-quorum-apagando-nodos-20261121.png>)

| Hora | Qué hice | Votos | Resultado |
|---|---|---|---|
| 21:53 | Apago `pve3` | 2 | `Quorate`: el clúster funciona y **deja escribir** su configuración |
| 21:54 | Apago también `pve2` | 1 | **`Activity blocked`**: `touch /etc/pve/prueba` → `Permission denied`. `pve1` se protege: no deja cambiar nada |
| 21:55 | Enciendo `pve2` y `pve3` | 3 | Vuelve a `Quorate` solo, sin hacer nada |

**Por qué se bloquea:** si `pve1` siguiera trabajando solo, podría pasar que los otros dos estuvieran vivos pero sin red entre ellos y `pve1`, y **los dos lados** cambiaran la misma VM a la vez. El quórum evita ese lío (se llama *split brain*, «cerebro partido»): solo trabaja el lado que tiene mayoría.

## 5. Contenedor y VM de prueba

→ [captura](<../evidencias/t5-proxmox-07-contenedor-y-vm-sin-escritorio-20261121.png>)

| | Contenedor `100 ct-debian` | VM `101 vm-debian` |
|---|---|---|
| Sistema | Debian 13 (plantilla oficial `debian-13-standard` de Proxmox) | Debian 13 **sin entorno gráfico** (imagen *cloud* oficial de Debian, huella SHA512 comprobada) |
| Recursos | 1 vCPU, 512 MB, disco 4 GB | 1 vCPU, 1 GB, disco 4 GB |
| IP | 192.168.27.20 (fija) | 192.168.27.21 (fija, puesta con *cloud-init*) |
| Acceso | SSH con clave (`root`) | SSH con clave (usuario `alvaro`) |
| Sin escritorio | | `dpkg -l | grep -cE "xserver|gnome|kde|xfce"` → **0** paquetes gráficos |

```
pct create 100 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst --hostname ct-debian \
  --cores 1 --memory 512 --rootfs local-lvm:4 --unprivileged 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.27.20/24,gw=192.168.27.1
qm create 101 --name vm-debian --cores 1 --memory 1024 --net0 virtio,bridge=vmbr0
qm set 101 --scsi0 local-lvm:0,import-from=/var/lib/vz/imagenes/debian-13-genericcloud-amd64.qcow2
qm set 101 --ide2 local-lvm:cloudinit --ciuser alvaro --ipconfig0 ip=192.168.27.21/24,gw=192.168.27.1
```

- **Contenedor sin privilegios** (`--unprivileged 1`): el `root` de dentro del contenedor **no** es el `root` del nodo. Si alguien entra en el contenedor, no controla el servidor.
- **`nesting=1`:** Proxmox avisó `Systemd 257 detected. You may need to enable nesting`. Debian 13 lo necesita para que algunos servicios funcionen bien dentro del contenedor; lo activé con `pct set 100 --features nesting=1`.
- **Imagen *cloud* + *cloud-init*:** en vez de instalar Debian paso a paso, se usa una imagen ya instalada y *cloud-init* le pone al arrancar el usuario, la clave SSH y la IP. Es como se crean los servidores en la nube.

## 6. Migración medida

Los dos se migraron de `pve1` a `pve2`. Mientras, desde `pve3` hacía `ping` cada medio segundo para ver **cuánto tiempo dejaban de responder**. → [captura](<../evidencias/t5-proxmox-08-migracion-ct-y-vm-tiempos-20261121.png>)

```
pct migrate 100 pve2 --restart                   # contenedor
qm migrate 101 pve2 --online --with-local-disks  # VM
```

| Qué se migra | Tipo de migración | Espacio copiado | Tiempo total | Sin servicio | Interpretación |
|---|---|---|---|---|---|
| **Contenedor** (512 MB RAM, disco 4 GB con 570 MB usados) | **Con reinicio** (`--restart`): se apaga, se copia el disco y se enciende en el otro nodo | **4 GB** de disco, a ~270 MB/s | **25 s** | **~22 s** (45 pings perdidos) | Un contenedor **no se puede mover encendido**: comparte el núcleo del nodo, no hay una «máquina» entera que congelar |
| **VM sin escritorio** (1 GB RAM, disco 4 GB) | **En caliente** (`--online`): sigue funcionando mientras se copia | **4 GB** de disco (en 22 s) + **738 MB de RAM** | **37 s** | **42 ms** de pausa (según Proxmox). El `ping` perdió 3 s mientras la red se enteraba de dónde estaba ahora la VM | El usuario casi no lo nota |

**Por qué se copian 4 GB si solo se usan 570-885 MB:** los discos están en el almacenamiento **local** de cada nodo (`local-lvm`). Para migrar, Proxmox tiene que **copiar el disco entero** por la red al otro nodo, y lo copia bloque a bloque, también la parte vacía.

**¿Se puede mejorar? ¿Cómo?**

| Mejora | Qué ganaría |
|---|---|
| **Almacenamiento compartido** (una carpeta NFS en el TrueNAS, o Ceph entre los nodos) | El disco ya está en un sitio que ven todos los nodos: **no se copia**. La VM solo mueve su RAM (unos segundos) y el contenedor solo se reinicia (~5 s) |
| **Discos más pequeños** | Menos que copiar (el tiempo crece con el tamaño del disco, no con lo que se usa) |
| **Red dedicada** para la migración (una segunda tarjeta, a 10 Gb/s en un servidor real) | La copia no compite con el tráfico normal |
| **Replicación** de ZFS entre nodos (`pvesr`) | El disco se copia por adelantado cada pocos minutos; al migrar solo falta lo último que ha cambiado |

## 7. Copias del clúster

| Se copia | Cómo | Dónde | Por qué |
|---|---|---|---|
| **Configuración de cada nodo**: `/etc/pve` (clúster, VM, usuarios, permisos), red (`interfaces`, `hosts`, `hostname`), Corosync y repositorios | `copia-config` (`tar.gz` de **24 KB**) con un **temporizador de systemd** diario a las **03:00**, igual que en `srv-linux` | NAS, usuario `proxmox` (`copias/proxmox`), las 7 más nuevas de cada nodo | Si un nodo se rompe, se reinstala Proxmox desde la ISO y se le devuelve su red y su sitio en el clúster |
| **Imagen del contenedor** | `vzdump 100 --mode snapshot --compress zstd`: **13 s**, **159 MB** (de 570 MB usados) | NAS, usuario `proxmox` | Una copia completa que se restaura con `pct restore`. Se hizo una vez para probarlo |

→ [captura](<../evidencias/t5-proxmox-10-copias-al-nas-20261121.png>)

| No se copia | Por qué |
|---|---|
| Proxmox (el sistema) | Se reinstala desde la ISO en 10 minutos |
| La VM de prueba | Se vuelve a crear desde la imagen *cloud* con 4 órdenes |
| Plantillas e imágenes descargadas | Se vuelven a descargar de los sitios oficiales |

**Problema encontrado:** al mirar los ficheros en el NAS salían con **hora de las 12:55** cuando eran las 21:55. TrueNAS tenía la zona horaria `America/Los_Angeles` (la que trae de serie). La cambié a `Europe/Madrid` (**System → General Settings → Localization**). Es importante: si las horas no coinciden, no se sabe qué copia es la buena.

## 8. Preguntas de teoría

### Ventajas e inconvenientes de virtualizar servidores

| Ventajas | Inconvenientes |
|---|---|
| **Menos máquinas físicas**: en un servidor caben muchos servidores virtuales → menos gasto en compra, luz y espacio | **Punto único de fallo**: si se rompe el servidor físico caen todas sus VM (por eso se hace un clúster) |
| **Rápido de crear**: una VM nueva en minutos, desde plantilla | **Rendimiento algo menor** que en físico, y las VM **compiten** por la misma RAM, CPU y disco (ver [T7](07-rendimiento.md)) |
| **Copias e instantáneas** de la máquina entera (`vzdump`, *snapshots*) | **Más complejo**: hay que saber del sistema de virtualización además de los sistemas de dentro |
| **Migración**: mover un servidor a otro equipo sin apagarlo (lo de arriba: 42 ms) para hacer mantenimiento | Si el servidor físico se queda corto de RAM, **todas** las VM van lentas |
| **Aislamiento**: si una VM tiene un virus o se cuelga, las demás siguen | Licencias: algunos programas cuentan los núcleos del servidor físico |
| **Probar** sin miedo: se hace una instantánea, se prueba, y si sale mal se vuelve atrás | |

### Contenedores frente a máquinas virtuales

| | Máquina virtual (KVM) | Contenedor (LXC) |
|---|---|---|
| Qué es | Un **ordenador entero** simulado, con su propio núcleo (*kernel*) | Un **grupo de programas aislados** que **usan el núcleo del nodo** |
| Sistemas | Cualquiera: Linux, Windows, BSD… | **Solo Linux** (el mismo núcleo que el nodo) |
| Arranque | Decenas de segundos | 1-2 segundos |
| Recursos | Más (en la prueba: 1 GB de RAM, 295 MB usados nada más arrancar) | Menos (512 MB, **23 MB usados**) |
| Aislamiento | **Muy fuerte**: un fallo del núcleo de la VM no afecta al nodo | Menor: comparten núcleo. Por eso, **sin privilegios** |
| Migración | **En caliente** (42 ms sin servicio) | **Con reinicio** (~22 s sin servicio) |

**Cuándo usar cada uno:**

- **Contenedor:** servicios Linux sencillos y ligeros: un servidor web, una base de datos pequeña, un DNS, un servidor de pruebas. Caben muchos en poco hardware.
- **Máquina virtual:** cuando hace falta **otro sistema operativo** (un Windows Server como `srv-dc`), **más seguridad** (servicios expuestos a Internet), un **núcleo propio** o módulos especiales, o que no se pare nunca al migrar.

## 9. Comparación con el grupo

| Aspecto | Grupo 3 | Mi entorno |
|---|---|---|
| Dónde corren los nodos | 3 VM en el servidor Proxmox del centro (Proxmox anidado) | 3 VM en mi VirtualBox (Proxmox anidado) |
| Nodos | 3 (`pve`, `pve2`, `pve3`), ~4 GiB cada uno | 3 (`pve1`, `pve2`, `pve3`), 4 GB cada uno |
| Almacenamiento | (ver su documentación) | Local en cada nodo (`local-lvm`): por eso la migración copia el disco entero |
