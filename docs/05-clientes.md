# T5 · Clientes: Windows 11 y GNU/Linux

RA: ISO RA1 (instalación y red), ISO RA3 (copias). Unirlos al dominio no es obligatorio en la rúbrica individual.

## Checklist

| Tarea de la rúbrica | Windows 11 | Cliente GNU/Linux |
|---|---|---|
| Instalar y comprobar versión y funcionamiento | [x] | [x] |
| Usuario(s) de administración | [x] | [x] |
| Parámetros de red | [x] | [x] |
| Copiar configuraciones/imágenes al servidor de copias y justificar qué sí y qué no | [x] | [x] |

Los dos clientes reciben la red **por DHCP** del router: un cliente no necesita IP fija, porque nadie tiene que encontrarlo a él. Es lo contrario que los servidores.

## Windows 11

### Requisitos en VM (TPM 2.0, UEFI, Secure Boot)

Windows 11 **no se instala** si el equipo no tiene:

| Requisito | Qué es | Cómo se lo di a la VM |
|---|---|---|
| **UEFI** | El «BIOS moderno» que arranca el equipo | `VBoxManage modifyvm cli-win11 --firmware efi` |
| **Secure Boot** | Solo deja arrancar sistemas firmados (evita virus de arranque) | Activado en la configuración UEFI de la VM |
| **TPM 2.0** | Un chip que guarda claves de cifrado (lo usa BitLocker, por ejemplo) | `VBoxManage modifyvm cli-win11 --tpm-type 2.0` (TPM virtual) |
| 4 GB de RAM y 64 GB de disco | Los mínimos de Microsoft | VM de 2 vCPU, 4 GB y 64 GB |

**Problema:** la VM se creó sin TPM y el instalador decía que el equipo no cumplía los requisitos. Se arregló añadiendo el TPM virtual.

### Instalación

- **Edición:** Windows 11 Pro, ISO oficial en español (compilación 26300), con su huella SHA256 comprobada ([00-entorno](00-entorno.md)).
- **Cuenta local, sin cuenta de Microsoft.** El instalador obliga a usar una cuenta de Microsoft y conexión a Internet. Para crear una cuenta **local**: **Shift+F10** (abre una consola en el instalador) → `start ms-cxh:localonly`. Una empresa no quiere que cada equipo dependa de cuentas personales en la nube.
- **Guest Additions** de VirtualBox, igual que en `srv-dc`: controladores para copiar y pegar, ratón y pantalla. → [captura](<../evidencias/t5-win11-04-guest-additions-20261121.png>)

→ [listo para instalar](<../evidencias/t5-win11-01-listo-para-instalar-20261118.png>) · [instalado](<../evidencias/t5-win11-02-instalado-escritorio-20261118.png>)

### Usuario de administración

| Cuenta | Para qué |
|---|---|
| `alvaro` | Cuenta **local** de administrador (grupo `Administradores`). Para tareas de administrador Windows pide confirmación (**UAC**, el «¿Quieres permitir que esta aplicación haga cambios?») |
| `Administrador` | La cuenta interna de Windows, **desactivada** de serie. Se deja así |

**Cambios hechos** (PowerShell como administrador):

| Orden | Para qué |
|---|---|
| `Rename-Computer -NewName cli-win11 -Restart` | El instalador le puso `DESKTOP-FDUL3OP`. Un nombre reconocible ayuda a saber qué equipo es en el router, en el NAS y en los registros |
| `Set-TimeZone -Id "Romance Standard Time"` | Tenía la hora UTC (1 hora menos que en España) |

### Red

| Parámetro | Valor | Cómo |
|---|---|---|
| IP | 192.168.27.188/24 | DHCP del router (`Dhcp: Enabled`) |
| Puerta de enlace | 192.168.27.1 | DHCP |
| DNS | 192.168.27.1 | DHCP |
| Ping al NAS | 0 % perdidos | |
| Web | `Invoke-WebRequest https://www.wikipedia.org` → **200** | |

→ [nombre, usuario, red e Internet](<../evidencias/t5-win11-05-nombre-usuario-red-internet-20261121.png>)

### Copias

| Se copia | Por qué |
|---|---|
| **Documentos, Escritorio e Imágenes** del usuario | Es lo único que no se puede recuperar reinstalando: el trabajo de la persona |

| No se copia | Por qué |
|---|---|
| Windows y los programas | Se reinstalan desde la ISO (o con una imagen preparada si hubiera muchos equipos iguales). Una imagen completa ocupa más de 20 GB por equipo |
| Configuración del sistema | En un cliente es estándar: no hay nada único que guardar |

**Automatizado** (ampliación): tarea programada **`Copia diaria al NAS`** a las **13:00** (a mediodía, porque los clientes están apagados de noche), como `SYSTEM`, con **«ejecutar lo antes posible si se ha pasado la hora»** (por si el equipo estaba apagado):

1. `Compress-Archive` comprime Documentos, Escritorio e Imágenes en `cli-win11-AAAAMMDD-HHMM.zip`.
2. `scp` lo sube al NAS (usuario `cli-win11`) con una **clave SSH** que solo puede leer `SYSTEM`.
3. En el NAS se guardan las **7 más nuevas**.

Para recuperar hay una segunda tarea, **`Recuperar ultima copia`**, que se lanza a mano: baja la copia más nueva y la descomprime en `C:\Users\Public\Recuperado`.

**Prueba de recuperación:** → [copia](<../evidencias/t5-win11-06-copia-automatica-al-nas-20261121.png>) · [borrado y recuperación](<../evidencias/t5-win11-07-borrado-y-recuperacion-20261121.png>)

| Hora | Qué pasó |
|---|---|
| 21:28:52 | Copia con el documento `presupuesto.txt` subida al NAS |
| 21:30:02 | **Fallo:** borro `presupuesto.txt` (`Test-Path` → `False`) |
| 21:30:06 | `Recuperar ultima copia` → el documento vuelve con su contenido |

**Tiempo de recuperación: 4 segundos.**

**Problema:** la primera vez la copia falló con `UNPROTECTED PRIVATE KEY FILE`: la clave privada la podía leer también `alvaro`, y el SSH de Windows se niega a usarla. Se arregló dejándola **solo** para `SYSTEM`.

## Cliente GNU/Linux

### Instalación

- **Sistema:** Ubuntu Desktop 26.04.1 LTS (LTS = soporte de 5 años), ISO oficial con su huella comprobada.
- **VM:** 4 vCPU y 4 GB de RAM. Con 2 vCPU y 3 GB iba al 100 % de CPU cuando abría las webs de OPNsense y TrueNAS (ver [T5 comunicaciones](05-comunicaciones.md)).
- Al principio se usó **sin instalar** (modo *live*) para las pruebas de los routers. El 18/11 se instaló en el disco de 25 GB.
- **Problema:** el botón «Siguiente» del instalador no respondía al teclado. Se reinició el instalador y se usó el ratón.

→ [resumen](<../evidencias/t5-clilinux-01-resumen-instalacion-20261118.png>) · [terminada](<../evidencias/t5-clilinux-02-instalacion-terminada-20261118.png>) · [arranca desde el disco](<../evidencias/t5-clilinux-03-arranca-desde-disco-20261118.png>)

### Usuario de administración

- **`alvaro`**, en el grupo **`sudo`** (puede hacer tareas de administrador poniendo `sudo` delante) y `adm` (puede leer los registros del sistema).
- `root` sin contraseña (bloqueado), como hace Ubuntu de serie.
- Instalé **`openssh-server`** para administrarlo por SSH desde el portátil.

### Red

| Parámetro | Valor | Cómo |
|---|---|---|
| IP | 192.168.27.136/24 | DHCP (`proto dhcp` en `ip r`) |
| Puerta de enlace | 192.168.27.1 | DHCP |
| DNS | 192.168.27.1 | `resolvectl status` |
| Ping al NAS | 0 % de pérdida | |
| Web | `wget --spider https://www.wikipedia.org` → OK | |

→ [nombre, usuario, red e Internet](<../evidencias/t5-clilinux-04-red-usuario-internet-20261121.png>)

### Copias

| Se copia | Por qué |
|---|---|
| `/home` (documentos de los usuarios), sin cachés ni descargas | El trabajo de las personas |
| `/etc` y la lista de paquetes | Por si se ha personalizado algo (impresoras, red, programas) |

| No se copia | Por qué |
|---|---|
| El sistema y los programas | Se reinstalan desde la ISO y la lista de paquetes |
| `.cache`, `snap`, `Descargas` | Se pueden volver a generar o descargar; solo ocuparían espacio |

**Automatizado** (ampliación): igual que `srv-linux`, con un **temporizador de systemd** (`copia-config.timer`), pero a las **13:00** y con `Persistent=true` (si el equipo estaba apagado, la copia se hace al encenderlo). Sube `copia-cli-linux-AAAAMMDD-HHMM.tar.gz` (**1,4 MB**) al NAS con la clave SSH de `root` y guarda las 7 más nuevas.

**Prueba de recuperación:** → [captura](<../evidencias/t5-clilinux-05-copia-borrado-y-recuperacion-20261121.png>)

| Hora | Qué pasó |
|---|---|
| 21:13:28 | Copia subida con el documento `~/Documentos/presupuesto.txt` |
| 21:13:48 | **Fallo:** `rm ~/Documentos/presupuesto.txt` |
| 21:13:48 | Bajo la copia (`sftp … get`) y saco solo ese fichero: `tar xzf … -C / home/alvaro/Documentos/presupuesto.txt` |
| 21:13:48 | `cat` → el documento ha vuelto con su contenido |

**Tiempo de recuperación: menos de 1 segundo** (la copia es pequeña y el NAS está en la misma red).
