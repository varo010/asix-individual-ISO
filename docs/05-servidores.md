# T5 · Servidores: Windows Server 2025 y GNU/Linux de infraestructura

RA: ISO RA1 (instalación y red), ISO RA3 (copias).

## Checklist

| Tarea de la rúbrica | Windows Server 2025 | Servidor GNU/Linux |
|---|---|---|
| Instalar y comprobar versión y funcionamiento | [x] | [x] |
| Usuario(s) de administración | [x] | [x] |
| Parámetros de red | [x] | [x] |
| Copiar configuraciones/imágenes al servidor de copias y justificar qué sí y qué no | [x] | [x] |
| Explicar qué es un dominio y qué aporta la gestión centralizada | [x] | — |
| Software para su función (de momento ninguno adicional) | — | [x] (SSH, `mdadm` para el RAID de la T6 y `rsync`) |

## Windows Server 2025

> **Va en una VM aparte, fuera del clúster Proxmox** (indicación del profesorado). `srv-dc` es una VM de VirtualBox con sus propios recursos (2 vCPU, 4 GB, 50 GB) y no depende de `pve1` ni `pve2`.
> **Por qué:** el clúster corre con virtualización anidada (una VM dentro de otra), que es más lenta y más delicada. Si un nodo falla o va justo de RAM, el dominio caería con él, y todos los equipos que dependen del dominio se quedarían sin poder iniciar sesión. Separado, el servidor más importante no depende de la parte más frágil del entorno.

### Instalación (edición, licencia de evaluación, requisitos)

| Dato | Valor | Por qué |
|---|---|---|
| Edición | **Windows Server 2025 Standard Evaluation**, con escritorio (*Desktop Experience*) | La de evaluación es gratuita durante **180 días**: suficiente para el proyecto. Con escritorio porque es más fácil de enseñar y comprobar |
| Versión | 10.0.26100 (compilación 26100), 64 bits | Comprobado con `Get-ComputerInfo` |
| VM | 2 vCPU, 4 GB de RAM, disco de 50 GB, red `lan-iso` | Microsoft pide como mínimo 2 GB con escritorio; con 4 GB va con soltura |
| Imagen | ISO oficial de Microsoft (inglés) | Microsoft no publica su huella SHA256 de la evaluación (ver [00-entorno](00-entorno.md)) |

→ [edición elegida](<../evidencias/t5-ws-01-edicion-20261118.png>) · [disco](<../evidencias/t5-ws-02-disco-20261118.png>) · [Server Manager](<../evidencias/t5-ws-03-server-manager-20261118.png>) · [versión](<../evidencias/t5-ws-05-version-zona-horaria-antes-20261121.png>)

**Guest Additions de VirtualBox.** Son los controladores de VirtualBox para Windows: sin ellos no funciona copiar y pegar entre el portátil y la VM, ni el ratón integrado, ni el ajuste de pantalla. Las instalé en modo silencioso (`VBoxWindowsAdditions.exe /S`) después de añadir los certificados de Oracle como «de confianza» (`VBoxCertUtil.exe add-trusted-publisher`), para que Windows no preguntara por cada controlador.

**Problemas encontrados:**
- **Hora equivocada:** el instalador en inglés deja la zona horaria del **Pacífico (UTC-8)**. Las horas salían 9 horas mal. Lo arreglé con `Set-TimeZone -Id "Romance Standard Time"` (la zona de Madrid). Es importante: Active Directory **no deja iniciar sesión** si el reloj de un equipo se separa más de **5 minutos** del servidor (lo exige Kerberos, el sistema de autenticación del dominio).
- **Ctrl+Alt+Supr:** en el portátil esa combinación la recoge Linux (cierra la sesión y con ella VirtualBox, apagando todas las VM de golpe). En la VM se envía con **Ctrl derecho + Supr** o con el menú **Entrada → Teclado → Insertar Ctrl-Alt-Supr**.

### Usuario de administración

| Cuenta | Para qué |
|---|---|
| `Administrator` | La cuenta que crea Windows. Al crear el dominio pasa a ser el **administrador del dominio**. Queda como cuenta de emergencia |
| `alvaro` (dominio `ISO27`) | **Mi cuenta nominal** de administrador: está en el grupo **Domain Admins**. Así cada cambio queda a nombre de una persona y no de una cuenta genérica |
| `empleado1` | Usuario normal de prueba, en el grupo `Contabilidad`. Tiene que cambiar la contraseña al primer inicio de sesión |

→ [usuarios, grupos y unidades organizativas](<../evidencias/t5-ws-08-unidades-usuarios-grupos-gpo-20261121.png>)

### Red

| Parámetro | Valor | Por qué |
|---|---|---|
| Nombre | `srv-dc` (antes `WIN-0QDJBUCRNR0`) | Un nombre fijo y reconocible; el dominio se apoya en él |
| IP | **192.168.27.13/24, fija** | Los equipos del dominio lo buscan siempre en la misma IP. Por DHCP podría cambiar |
| Puerta de enlace | 192.168.27.1 (el router) | Para salir a Internet |
| DNS del propio servidor | **127.0.0.1** (él mismo) | Un controlador de dominio es su propio DNS: ahí están los registros que dicen dónde está el dominio |
| Reenviador DNS | 192.168.27.1 | Lo que no sabe (por ejemplo `www.wikipedia.org`) se lo pregunta al router |

Órdenes usadas (PowerShell como administrador):

```
Set-TimeZone -Id "Romance Standard Time"
Set-NetIPInterface -InterfaceAlias Ethernet -Dhcp Disabled
New-NetIPAddress -InterfaceAlias Ethernet -IPAddress 192.168.27.13 -PrefixLength 24 -DefaultGateway 192.168.27.1
Set-DnsClientServerAddress -InterfaceAlias Ethernet -ServerAddresses 192.168.27.1
Rename-Computer -NewName srv-dc -Restart
```

| Orden | Qué hace | Qué pasa si falta |
|---|---|---|
| `Set-NetIPInterface … -Dhcp Disabled` | Deja de pedir IP al router | La IP podría cambiar y nadie encontraría el servidor |
| `New-NetIPAddress …` | IP fija .13 con máscara /24 (255.255.255.0) y puerta de enlace .1 | — |
| `Set-DnsClientServerAddress …` | DNS = router (antes de crear el dominio). Al crear el dominio pasa a ser él mismo | Sin DNS no traduce nombres |
| `Rename-Computer … -Restart` | Cambia el nombre y reinicia (hace falta reiniciar) | Se quedaría con un nombre aleatorio |

→ [IP fija y zona horaria](<../evidencias/t5-ws-06-zona-horaria-ip-fija-20261121.png>)

### Dominio `iso27.lan`

**Cómo lo creé** (PowerShell como administrador):

```
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName iso27.lan -DomainNetbiosName ISO27 -InstallDns -SafeModeAdministratorPassword (contraseña)
```

| Parte | Qué significa |
|---|---|
| `AD-Domain-Services` | El **rol** (función) de Active Directory: la base de datos del dominio |
| `Install-ADDSForest` | Crea un **bosque** nuevo (el primer dominio de la empresa) y convierte el equipo en **controlador de dominio** (DC) |
| `iso27.lan` | Nombre del dominio. `27` es mi número de red (192.168.**27**.0). Uso `.lan` y no `.local` porque `.local` lo usan los equipos Linux y Mac para otra cosa (mDNS) y daría problemas |
| `ISO27` | Nombre corto (NetBIOS), el que sale en `ISO27\alvaro` |
| `-InstallDns` | Instala también el servidor DNS, imprescindible para el dominio |
| `-SafeModeAdministratorPassword` | Contraseña del **modo de restauración** (DSRM): se usa para arrancar el DC en modo reparación si la base de datos se estropea |

Después hay que **reiniciar**. El primer arranque como DC tarda varios minutos («Applying computer settings»).

**Comprobación:** → [captura](<../evidencias/t5-ws-07-dominio-iso27-funcionando-20261121.png>)

| Prueba | Resultado |
|---|---|
| `Get-ADDomain` | `iso27.lan`, NetBIOS `ISO27`, nivel `Windows2025Domain`, PDC `srv-dc.iso27.lan` |
| `Get-ADDomainController` | `srv-dc.iso27.lan`, 192.168.27.13, catálogo global: sí |
| Servicios | `NTDS` (la base de datos), `DNS`, `Netlogon` (inicios de sesión) y `KDC` (Kerberos) en **Running** |
| DNS | Cliente DNS = 127.0.0.1; reenviador = 192.168.27.1; resuelve `www.wikipedia.org` |

**Problema que tuve:** después de crear el dominio, el primer reinicio **no se llegó a hacer** (la orden se lanzó desde fuera y falló). Los servicios `NTDS`, `KDC` y `Netlogon` salían **Stopped** y el equipo era solo «servidor miembro» (`DomainRole = 3`). Al reiniciar de verdad pasó a `DomainRole = 5` (controlador de dominio principal) y todo arrancó. **Lección:** si los servicios del dominio están parados justo después de crearlo, lo primero es comprobar si falta el reinicio.

**Organización del dominio** (para enseñar la gestión centralizada):

```
iso27.lan
└── Empresa (unidad organizativa)
    ├── Administradores  → alvaro (Domain Admins)
    ├── Usuarios         → empleado1   ← directiva «Usuarios - sin Panel de control»
    ├── Equipos
    └── Grupos           → Contabilidad (empleado1)
```

- Una **unidad organizativa (OU)** es una carpeta del dominio para ordenar usuarios y equipos, y sobre todo para **aplicarles directivas** a todos a la vez.
- La **directiva de grupo (GPO)** «Usuarios - sin Panel de control» está enlazada a la OU `Usuarios`: cualquier usuario que se cree ahí **no podrá abrir el Panel de control** en ningún equipo del dominio, sin tocar los equipos uno a uno. Es un ejemplo pequeño de lo que aporta un dominio.

### Copias: qué sí y qué no

**Dónde:** NAS, dataset `copias/srv-dc`, usuario `srv-dc` (solo SFTP), igual que el resto de equipos.

| Se copia | Por qué |
|---|---|
| **Base de datos de Active Directory** (con `ntdsutil … ifm`: *Install From Media*) | Es lo único que no se puede volver a crear igual: usuarios, grupos, contraseñas, OU. `ifm` hace una copia coherente de la base de datos y de `SYSVOL` (donde están las directivas) aunque el servidor esté funcionando |
| **Directivas de grupo** (`Backup-GPO -All`) | Permiten restaurar una sola directiva si alguien la estropea, sin tocar todo el dominio |
| **Lista de usuarios** en CSV | Para consultarla rápido sin restaurar nada |

| No se copia | Por qué |
|---|---|
| Windows entero (imagen del disco, 15-20 GB) | Se reinstala desde la ISO en unos 20 minutos. Copiarlo entero cada día llenaría el NAS (20 GB) en un solo día |
| Programas | No hay programas propios instalados |

**Cómo se hace: automatizado** (ampliación). Una **tarea programada** de Windows, `Copia diaria al NAS`, se ejecuta **cada día a las 02:30** como `SYSTEM` (la cuenta del propio sistema, que no depende de ninguna persona):

1. Crea una carpeta con fecha y hora (`srv-dc-20261121-2109`).
2. `ntdsutil "activate instance ntds" "ifm" "create sysvol full …"` → copia de AD y SYSVOL.
3. `Backup-GPO -All` → copia de las directivas. `Get-ADUser … | Export-Csv` → lista de usuarios.
4. `Compress-Archive` → lo comprime en un `.zip` (**4,8 MB**).
5. `scp` → lo sube al NAS con una **clave SSH** (sin contraseña escrita en ningún sitio).
6. En el NAS deja solo las **7 copias más nuevas** (una semana) y borra las viejas.
7. Borra la carpeta temporal del servidor.

→ [tarea programada y resultado](<../evidencias/t5-ws-09-copia-automatica-ad-al-nas-20261121.png>) · [ficheros en el NAS](<../evidencias/t5-truenas-18-estado-copias-api-20261121.txt>)

**Problemas al automatizarla** (y cómo se arreglaron):

| Error | Causa | Solución |
|---|---|---|
| `Host key verification failed` | El cliente SSH de Windows no conocía la huella del NAS | Opción `StrictHostKeyChecking=accept-new`: la acepta la primera vez y después la comprueba |
| `Access is denied` al borrar la carpeta temporal | La copia de `SYSVOL` mantiene sus permisos especiales | `takeown` + `icacls /reset` antes de borrar |
| `UNPROTECTED PRIVATE KEY FILE` | El SSH de Windows se niega a usar una clave privada que puedan leer otros usuarios | La clave solo es de `SYSTEM` (`icacls … /setowner SYSTEM` y quitar el resto de permisos) |

**Cómo se restauraría:** reinstalar Windows Server, ponerle la misma IP y nombre, y crear el DC con `Install-ADDSForest` **desde el medio** de la copia (`-InstallationMediaPath`) o, para un usuario o una GPO concreta, sacarlos de la copia (`Restore-GPO`). Para borrados de usuarios del día a día, lo más rápido es la **papelera de reciclaje de AD**.

### Qué es un dominio y qué aporta a la empresa

**Qué es:** un **dominio** es un grupo de equipos y usuarios que se administran **desde un servidor central**, el **controlador de dominio**. En Windows se hace con **Active Directory**: una base de datos con todas las cuentas, grupos y equipos de la empresa.

**Sin dominio** (grupo de trabajo), cada equipo tiene sus propios usuarios: con 20 equipos y 20 empleados habría que crear 400 cuentas y cambiar las contraseñas en cada equipo.

**Qué aporta la gestión centralizada:**

| Ventaja | Ejemplo en mi entorno |
|---|---|
| **Una cuenta por persona** para todos los equipos | `empleado1` podría iniciar sesión en cualquier equipo del dominio con la misma contraseña |
| **Altas y bajas en un solo sitio** | Si un empleado se va, se desactiva su cuenta en `srv-dc` y deja de entrar en todos los equipos a la vez |
| **Directivas de grupo (GPO)** | La GPO «sin Panel de control» se aplica a todos los usuarios de la OU `Usuarios` sin tocar los equipos |
| **Permisos por grupos** | Las carpetas de contabilidad se dan al grupo `Contabilidad`, no persona a persona |
| **Seguridad** | Contraseñas con normas comunes (longitud, caducidad), autenticación Kerberos y registro de quién inicia sesión |
| **DNS integrado** | Los equipos se encuentran por nombre (`srv-dc.iso27.lan`) |

**Inconvenientes:** el controlador de dominio se vuelve **crítico** (si cae, nadie inicia sesión nuevo: por eso va fuera del clúster Proxmox y tiene copia diaria) y exige mantener la hora sincronizada y el DNS bien configurado.

## Servidor GNU/Linux de infraestructura

### Instalación

| Dato | Valor | Por qué |
|---|---|---|
| Sistema | **Debian 13.7** (*trixie*), sin escritorio | Un servidor no necesita entorno gráfico: menos memoria, menos actualizaciones y menos fallos de seguridad |
| VM | 1 vCPU, 1 GB de RAM, disco de 10 GB + **7 discos de 1 GB** (para el RAID de la [T6](06-raid.md)) | — |
| Particiones | Automáticas en el disco de 10 GB (todo en una partición + intercambio) | → [particiones](<../evidencias/t5-srvlinux-01-particiones-20261118.png>) |
| Programas | Solo «utilidades estándar» | → [selección](<../evidencias/t5-srvlinux-02-seleccion-programas-20261118.png>) · [instalado](<../evidencias/t5-srvlinux-03-instalado-20261118.png>) |

Se instaló sin red (sin servidor *mirror*), así que al terminar no tenía red configurada ni repositorios en Internet (solo el DVD). → [estado inicial](<../evidencias/t5-srvlinux-04-red-y-repositorios-antes-20261121.png>)

### Usuario de administración

- **`alvaro`** con permiso de `sudo` (grupo `sudo`): administra el servidor.
- **`root` bloqueado** (`passwd -S root` → `L`): en la instalación dejé su contraseña vacía y Debian lo bloquea. Nadie puede entrar como `root` con contraseña; cada orden de administrador queda apuntada con el nombre de quien la hizo.

→ [captura](<../evidencias/t5-srvlinux-08-copia-en-nas-root-bloqueado-20261121.png>)

### Red

Añadido al final de `/etc/network/interfaces`:

```
# LAN (red lan-iso): IP fija del servidor
auto enp0s3
iface enp0s3 inet static
    address 192.168.27.14/24
    gateway 192.168.27.1
```

Y el DNS en `/etc/resolv.conf`: `nameserver 192.168.27.1`. Se activa con `sudo ifup enp0s3`.

| Parámetro | Valor | Por qué fija |
|---|---|---|
| IP | 192.168.27.14/24 | Es un servidor: los demás equipos (copias, SSH) lo buscan siempre en la misma IP |
| Puerta de enlace | 192.168.27.1 | El router |
| DNS | 192.168.27.1 | El router reenvía las preguntas a Internet |

Comprobación: `ip -br a`, `ip r` y `ping` al router y al NAS (0 % de pérdida). → [captura](<../evidencias/t5-srvlinux-05-ip-fija-y-ping-20261121.png>)

**Repositorios:** cambié el del DVD por los de Internet **por HTTPS** (la red de clase bloquea el puerto 80, como pasó con el router nftables):

```
deb https://deb.debian.org/debian trixie main non-free-firmware
deb https://deb.debian.org/debian trixie-updates main non-free-firmware
deb https://security.debian.org/debian-security trixie-security main non-free-firmware
```

La tercera línea son las **actualizaciones de seguridad**, imprescindibles en un servidor.

**Programas instalados** (`apt-get install openssh-server mdadm rsync`): → [captura](<../evidencias/t5-srvlinux-06-instalar-ssh-mdadm-20261121.png>)

| Programa | Para qué |
|---|---|
| `openssh-server` | Administrarlo por SSH desde otro equipo (copiar y pegar órdenes, sin depender de la ventana de VirtualBox) |
| `mdadm` | Crear y gestionar el RAID por software de la [T6](06-raid.md) |
| `rsync` | Copiar carpetas de forma eficiente (solo lo que cambia) |

### Copias: qué sí y qué no

| Se copia | Por qué |
|---|---|
| `/etc` entero | Toda la configuración del sistema (red, usuarios, servicios, RAID en `mdadm.conf`, `fstab`) |
| Lista de paquetes instalados (`dpkg --get-selections`) | Para volver a instalar exactamente los mismos programas en un Debian nuevo |

| No se copia | Por qué |
|---|---|
| El sistema operativo | Se reinstala desde la ISO en 10 minutos; los paquetes se recuperan con la lista |
| Los datos del RAID (`/srv/raid*`) | Son datos ficticios de prueba. En una empresa sí se copiarían: **un RAID no es una copia de seguridad** (si se borra un fichero, se borra en todos los discos a la vez) |

**Automatizado** (ampliación): un **temporizador de systemd** (`copia-config.timer`) lanza cada día a las **02:00** el servicio `copia-config.service`, que ejecuta `/usr/local/sbin/copia-config`:

1. Guarda la lista de paquetes.
2. Empaqueta `/etc` y la lista en `copia-srv-linux-AAAAMMDD-HHMM.tar.gz` (**425 KB**).
3. Lo sube al NAS por SFTP con la **clave SSH de root** (sin contraseña).
4. En el NAS deja solo las **7 más nuevas**.

`Persistent=true` hace que, si el servidor estaba apagado a las 02:00, la copia se haga nada más arrancar.

→ [programa y temporizador](<../evidencias/t5-srvlinux-07-copia-automatica-programa-y-timer-20261121.png>) · [copia en el NAS](<../evidencias/t5-srvlinux-08-copia-en-nas-root-bloqueado-20261121.png>)

**Prueba de recuperación** (fallo provocado):

| Hora | Qué pasó |
|---|---|
| 20:42:11 | **Fallo:** borro `/etc/apt/sources.list`. `apt-get update` ya no sabe de dónde descargar nada → [captura](<../evidencias/t5-srvlinux-09-fallo-sources-borrado-20261121.png>) |
| 20:42:26 | Bajo la copia del NAS: `echo "get copia-srv-linux-20261121-2041.tar.gz" \| sudo sftp -b - srv-linux@192.168.27.10` |
| — | Saco **solo** ese fichero: `sudo tar xzf copia-….tar.gz -C / etc/apt/sources.list` |
| 20:42:58 | `apt-get update` vuelve a funcionar (17,2 MB descargados) → [captura](<../evidencias/t5-srvlinux-10-recuperado-desde-nas-20261121.png>) |

**Tiempo de recuperación: 32 segundos** (casi todo es la descarga de `apt-get update`).
