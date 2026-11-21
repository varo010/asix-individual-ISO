# T5 · Servidores de comunicaciones

RA: ISO RA1 (instalación, red, filtrado), ISO RA3 (copia y recuperación), ISO RA6 (registros). Las tres alternativas se prueban una detrás de otra en el mismo escenario.

## Comprobaciones generales (todas las VM de la T5)

- [x] Requisitos y origen de las imágenes (arquitectura, fuente fiable); integridad verificada cuando el fabricante publique el hash.
- [x] Actualizar y comprobar el funcionamiento antes de ponerlo «en producción».
- [x] Identificar problemas posibles, configuraciones críticas y tiempo de respuesta ante una incidencia de instalación o arranque.

**Resumen de las tres comprobaciones** (sirve para todas las VM de la T5):

| Comprobación | Dónde está |
|---|---|
| **Imágenes:** todas de la web oficial, 64 bits, y con la huella SHA256 comparada con la del fabricante (tabla de abajo y [00-entorno](00-entorno.md)). Única excepción: Windows Server, porque Microsoft no publica la huella | Tabla de abajo · [t5-isos-sha256](../evidencias/t5-isos-sha256-20261116.txt) |
| **Actualizar antes de usar:** OPNsense (96 paquetes → 26.7.9), Proxmox (225 paquetes → 9.2.32), `r-nftables` (20 paquetes) y `srv-linux` (31 paquetes), con `apt` y el repositorio de seguridad de Debian; Windows por Windows Update | Este documento · [05-proxmox](05-proxmox.md#2-instalación) · [05-servidores](05-servidores.md) |
| **Problemas, configuración crítica y tiempo de respuesta:** cada sistema tiene su apartado *Problema* y su prueba de **fallo provocado y recuperación** desde la copia del NAS, con el tiempo medido (MikroTik, OPNsense y nftables aquí; `srv-linux` 32 s; `cli-win11` 4 s; `cli-linux` menos de 1 s) | Apartados *Fallo y recuperación* · [05-copias](05-copias.md#1-inventario-de-copias) |

| Sistema | Versión | Origen (URL oficial) | Hash publicado | Hash comprobado | Fecha |
|---|---|---|---|---|---|
| MikroTik CHR | 7.24.6 (long-term), x86-64 | <https://mikrotik.com/download> (imagen de disco para VirtualBox `chr-7.24.6.vdi.zip`) | Sí (fichero `.sha256`) | `746a4975…5faae` ✅ coincide | 15/11/2026 |
| OPNsense | 26.7, amd64 | <https://pkg.opnsense.org/releases/26.7/> (`OPNsense-26.7-dvd-amd64.iso.bz2`) | Sí, del `.bz2` | `95cafedd…186bfb` ✅ coincide | 15/11/2026 |
| Debian | 13.7.0, amd64, netinst | <https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/> | Sí (`SHA256SUMS`) | `a7ef94ac…ce6167e355` ✅ coincide | 15/11/2026 |

Comprobación completa: [t5-isos-sha256-20261116.txt](../evidencias/t5-isos-sha256-20261116.txt). Cómo se hace: `sha256sum fichero` y comparar el resultado con el que publica el fabricante en su web.

## Checklist por router

| Tarea de la rúbrica | MikroTik | OPNsense | Debian + nftables |
|---|---|---|---|
| Instalar y comprobar versión y funcionamiento | [x] | [x] | [x] |
| Usuario(s) de administración | [x] | [x] | [x] |
| Red WAN y LAN | [x] | [x] | [x] |
| Acceso a Internet desde la LAN | [x] | [x] | [x] |
| Conexión permitida y bloqueada + registro interpretado | [x] | [x] | [x] |
| Copia de la configuración en el servidor de copias | [x] | [x] | [x] |
| Fallo provocado y recuperación | [x] | [x] | [x] |

## MikroTik (CHR)

**Qué es CHR:** *Cloud Hosted Router* es RouterOS, el mismo sistema que lleva el MikroTik hEX del grupo, preparado para funcionar en una máquina virtual. Lo uso porque en casa no tengo un MikroTik físico. Se configura igual que el hEX.

**Cómo lo administro:** con **WebFig**, la página web de configuración del router. El portátil no ve el router directamente: su WAN está detrás de la NAT de VirtualBox y su LAN es una red interna. Por eso he puesto en VirtualBox una **redirección de puerto**: lo que entra en el portátil por `127.0.0.1:8080` va al puerto 80 (la web) del router. Solo escucha en `127.0.0.1`, así que **solo se puede entrar desde mi propio portátil**, no desde la red del aula.

### Instalación

- **No hay instalador:** MikroTik publica CHR como un **disco virtual ya instalado** (`.vdi`). Lo descargué de la web oficial, comprobé su SHA256 (tabla de arriba) y lo usé como disco de la VM `r-mikrotik`.
- **VM:** 1 vCPU, 256 MB de RAM. Tarjeta 1 → NAT de VirtualBox (WAN). Tarjeta 2 → red interna `lan-iso` (LAN).
- Primer arranque: usuario `admin` sin contraseña; el sistema **obliga a poner una contraseña** nueva.
- **Comprobación de versión y funcionamiento** (`/system resource print`): RouterOS **7.24.6 (long-term)**, arquitectura x86_64, 256 MiB de RAM (56 MiB libres) y CPU al 0 %. → [captura](<../evidencias/t5-mikrotik-01-version-recursos-20261116.png>)
- Interfaces que ve el router (`/interface print` y `/ip address print`): `ether1` y `ether2`. Al arrancar, `ether1` ya tenía **10.0.2.15/24 por DHCP** (se lo da la NAT de VirtualBox) y `ether2` no tenía IP. → [captura](<../evidencias/t5-mikrotik-02-interfaces-ip-inicial-20261116.png>)

### Usuario de administración

En **System → Users** creé mi usuario `alvaro` en el grupo **full**, entré con él y después **desactivé** `admin`. → [captura](<../evidencias/t5-mikrotik-08-usuarios-20261116.png>)

- **Por qué un usuario propio:** todo el mundo sabe que un MikroTik trae el usuario `admin`. Quien intenta entrar solo tiene que adivinar la contraseña; con un usuario propio tiene que adivinar dos cosas. Además, el registro del router apunta **qué usuario hizo cada cambio**: si todos entran como `admin`, no se sabe quién fue.
- **Por qué en ese orden:** primero comprobé que podía entrar con `alvaro`. Si desactivo `admin` antes y mi usuario fallara, me quedaría sin acceso al router.
- **Por qué desactivar y no borrar:** si hiciera falta, `admin` se puede volver a activar desde la consola.
- **Grupos de RouterOS:** `full` (puede hacerlo todo), `write` (cambia la configuración pero no gestiona usuarios) y `read` (solo puede mirar). Un técnico que solo vigila el router tendría `read`.
- La contraseña **no está en este repositorio**.

### Red WAN y LAN

| Interfaz | Red | IP | Cómo |
|---|---|---|---|
| `ether1` (WAN) | NAT de VirtualBox | 10.0.2.15/24 | DHCP de VirtualBox (sale con una **D** de «dinámica») |
| `ether2` (LAN) | `lan-iso` | **192.168.27.1/24** | Fija, puesta a mano |

**IP de la LAN** (**IP → Addresses → New**: `192.168.27.1/24` en `ether2`). → [captura](<../evidencias/t5-mikrotik-03-ip-lan-20261116.png>)
Con esta IP el router «vive» dentro de la LAN y es la **puerta de enlace**: todos los equipos de 192.168.27.0/24 le mandan a él lo que va fuera de su red.
⚠️ Esto **no es un puente** (*bridge*). En MikroTik, un *bridge* une varios puertos para que funcionen como un switch; aquí solo le damos una IP a una interfaz.

**DNS** (**IP → DNS**: marcar *Allow Remote Requests*). → [captura](<../evidencias/t5-mikrotik-04-dns-20261116.png>)
El router **hace de servidor DNS** para la LAN. Los equipos le preguntan a él (192.168.27.1) y él reenvía la pregunta al DNS que recibió de VirtualBox (10.0.2.3, en *Dynamic Servers*). Sin marcar esa casilla, el router solo resuelve nombres para sí mismo.

**DHCP para los clientes** (**IP → DHCP Server → DHCP Setup**, en `ether2`):

| Qué entrega | Valor | Captura |
|---|---|---|
| IP | De la **192.168.27.100 a la .199** (100 direcciones). De la .2 a la .99 quedan para IP fijas (servidores), como se diseñó en la [T4](04-red.md) | [pool](<../evidencias/t5-mikrotik-05b-dhcp-pool-20261116.png>) |
| Máscara, puerta de enlace y DNS | 192.168.27.0/24 · 192.168.27.1 · 192.168.27.1 | [networks](<../evidencias/t5-mikrotik-05c-dhcp-networks-20261116.png>) |
| Duración del préstamo (*lease*) | 30 minutos | [servidor](<../evidencias/t5-mikrotik-05-dhcp-20261116.png>) |

Así un cliente que se conecta a la LAN no tiene que configurar nada a mano: recibe IP, puerta de enlace y DNS.

### Salida a Internet

**Regla de NAT** (**IP → Firewall → NAT → New**: *chain* `srcnat`, *Out. Interface* `ether1`, *action* `masquerade`). → [lista](<../evidencias/t5-mikrotik-06-nat-20261116.png>) · [regla](<../evidencias/t5-mikrotik-06b-nat-regla-20261116.png>)

En el *Firewall* de MikroTik hay dos cosas distintas, y conviene no confundirlas:
- **Filter:** decide qué tráfico **pasa o se bloquea**. Todavía no hay reglas, así que ahora pasa todo.
- **NAT:** **cambia las direcciones** de los paquetes. Es lo que se ha configurado aquí. Esta regla no «permite» nada.

**Qué hace *masquerade*** («disfraz»): las IP 192.168.x.x son **privadas**, y en Internet nadie sabría devolver una respuesta a 192.168.27.100. Por eso:

1. `cli-linux` (192.168.27.100) pide una web. El paquete sale con origen 192.168.27.100.
2. Al salir por `ether1`, el router **cambia el origen por su IP de la WAN** (10.0.2.15) y apunta en una tabla que esa conexión es del .100.
3. La respuesta vuelve a 10.0.2.15.
4. El router mira su tabla, **deshace el cambio** y entrega la respuesta al .100.

Es como el conserje de un edificio: los vecinos ponen como remite la dirección del edificio, y cuando llega la respuesta, el conserje mira su libreta y la sube al piso que toca. La tabla se puede ver en **IP → Firewall → Connections**.

**Por qué *masquerade* y no `src-nat`:** *masquerade* usa la IP que tenga `ether1` **en ese momento**. Como la WAN recibe su IP por DHCP y podría cambiar, la regla sigue funcionando sin tocarla. Con una IP fija se podría usar `src-nat` escribiendo la IP.

Después, la NAT de VirtualBox hace lo mismo otra vez (cambia 10.0.2.15 por la IP del portátil): por eso hay **doble NAT** ([T4](04-red.md)).

**Comprobación** (desde la terminal del router): → [captura](<../evidencias/t5-mikrotik-07-prueba-internet-20261116.png>)

| Prueba | Resultado | Qué demuestra |
|---|---|---|
| `/ping 10.0.2.2 count=3` | 3 de 3 respuestas, menos de 0,4 ms | El router llega a la puerta de enlace de la NAT de VirtualBox |
| `/tool fetch url="http://example.com"` | `status: finished`, código 200 | El router **sale a Internet** y el **DNS funciona** (ha traducido `example.com` a una IP) |

No uso `ping` a Internet porque la red del aula lo bloquea (se vio en la [T2](02-equipos.md)); la descarga de una web sí pasa.

**Comprobación desde un cliente de la LAN** (17/11/2026). Uso `cli-linux` arrancado desde la ISO de Ubuntu en modo **Probar Ubuntu**, sin instalar: basta para probar la red.

| Prueba | Resultado | Qué significa |
|---|---|---|
| `ip -4 a` | `192.168.27.199/24 … dynamic` en `enp0s3` | IP dentro del rango .100–.199; **dynamic** = recibida por DHCP. `enp0s3` es la tarjeta de la VM conectada a `lan-iso` |
| | `valid_lft 1407sec` | Quedan unos 23 min del préstamo de 30; el cliente lo renueva solo antes de que caduque |
| `ip route` | `default via 192.168.27.1 … proto dhcp` | La puerta de enlace es el router, también recibida por DHCP |
| `resolvectl` | `DNS Servers: 192.168.27.1` | El DNS es el router |
| Firefox → `www.wikipedia.org` | La página carga (HTTPS) | Funcionan **juntos** el DHCP, el DNS del router y el NAT |
| WebFig → **IP → DHCP Server → Leases** | 192.168.27.199, MAC `08:00:27:8B:D9:75`, *host* `ubuntu`, caduca en 28 min | El router apunta **a quién ha prestado cada IP**: es la misma máquina vista desde el otro lado |

Ha recibido la **.199** y no la .100: MikroTik reparte las direcciones del *pool* empezando por el final. Lo importante es que está dentro del rango.

Evidencias: [IP, rutas y DNS del cliente](<../evidencias/t5-mikrotik-09-cliente-dhcp-20261117.png>) · [web abierta](<../evidencias/t5-mikrotik-10-cliente-internet-20261117.png>) · [préstamos del DHCP](<../evidencias/t5-mikrotik-11-dhcp-leases-20261117.png>)

### Conexión permitida y bloqueada

**Qué he hecho:** dos reglas en **IP → Firewall → Filter Rules**, las dos con registro (*log*). → [reglas](<../evidencias/t5-mikrotik-12-reglas-filtro-20261117.png>)

| # | Qué hace | Chain | Condiciones | Action | Log prefix |
|---|---|---|---|---|---|
| 0 | **Permite** navegar por web desde la LAN | `forward` | Entra por `ether2`, TCP, puertos 80 y 443, solo conexiones nuevas | `accept` | `LAN-WEB-OK` |
| 1 | **Bloquea** usar un DNS de fuera desde la LAN | `forward` | Entra por `ether2`, UDP, puerto 53 | `drop` | `LAN-DNS-BLOQ` |

**Por qué estas dos:**
- La web es el uso normal de un equipo de la empresa: se permite y queda apuntado.
- El bloqueo obliga a que todos los equipos pregunten **al DNS de la empresa** (el router). Si cualquiera pudiera poner `8.8.8.8` a mano, se saltaría ese control.

**Conceptos de las reglas:**
- **Chain `forward`:** tráfico que **atraviesa** el router (de la LAN a Internet). El tráfico dirigido **al propio router** va por la chain `input`. Por eso la regla 1 **no rompe el DNS del router**: cuando el cliente pregunta a 192.168.27.1, el paquete va al router, no lo atraviesa.
- **Solo conexiones nuevas (`connection-state=new`):** se apunta el primer paquete de cada conexión y no todos, para no llenar el registro.
- **`drop` frente a `reject`:** `drop` tira el paquete sin contestar y el cliente espera hasta rendirse (*timed out*); `reject` le avisaría al momento de que está prohibido.
- **El tráfico que no coincide con ninguna regla se permite:** en MikroTik, si ninguna regla lo para, pasa. Por ejemplo, Firefox usa a veces HTTP/3, que va por **UDP** 443: no lo apunta la regla 0 (que es TCP), pero pasa igual.
- Las columnas **Bytes** y **Packets** cuentan lo que ha atrapado cada regla: 1 paquete en la 0 y 3 en la 1 (los tres intentos de `nslookup`).

**Qué hace cada regla y qué pasa si falta o se rompe.** Para entenderlo hay que saber una cosa: en MikroTik, el tráfico que **no coincide con ninguna regla, pasa**. Las reglas solo cambian algo cuando lo paran o cuando lo apuntan.

| Regla | Qué hace, en sencillo | Si no la creo o la borro | Si la hago mal |
|---|---|---|---|
| **NAT `masquerade`** (IP → Firewall → NAT) | Cambia la IP privada del cliente (192.168.27.x) por la del router al salir a Internet, y la deshace en la respuesta. Es el «conserje» | **La LAN se queda sin Internet.** Los paquetes salen, pero con una IP privada a la que nadie sabe contestar. Lo comprobé en el [fallo provocado](#fallo-y-recuperación): `curl` acaba en `Connection timed out` | Si pongo como salida `ether2` (la LAN) en vez de `ether1`, no cambia nada al salir a Internet y el resultado es el mismo: sin Internet |
| **0 · `LAN-WEB-OK`** (accept, TCP 80 y 443) | Deja pasar la web y **apunta en el registro** cada conexión nueva | **La web sigue funcionando igual**, porque lo que no coincide con ninguna regla pasa. Lo único que se pierde es el **registro**: ya no queda apuntado quién navega ni a dónde. Esta regla sirve para **dejar constancia**, no para abrir paso | Si quito `connection-state=new`, apunta **todos** los paquetes de cada conexión y el registro se llena en segundos. Si me equivoco de puertos (por ejemplo solo el 80), las webs HTTPS (443) siguen funcionando pero **no quedan apuntadas** |
| **1 · `LAN-DNS-BLOQ`** (drop, UDP 53) | Tira las preguntas DNS que intentan **atravesar** el router hacia un DNS de fuera (por ejemplo 8.8.8.8) | Cualquier equipo puede usar el DNS que quiera escribiéndolo a mano y **se salta el DNS de la empresa**: si el router filtrara webs por DNS, ese equipo lo esquivaría. Tampoco queda apuntado | Si la pongo en la chain **`input`** en vez de `forward`, bloquea las preguntas **al propio router**: los clientes se quedan **sin DNS**, no pueden traducir nombres y **no navega nada**, aunque Internet funcione |

**El orden también cuenta:** MikroTik lee las reglas de arriba abajo y aplica la primera que coincide. Si delante de la `LAN-DNS-BLOQ` hubiera una regla que aceptara todo lo de la LAN, el bloqueo **nunca llegaría a actuar**.

**Pruebas desde `cli-linux`:** → [captura](<../evidencias/t5-mikrotik-13-prueba-dns-bloqueado-20261117.png>)

| Prueba | Antes de las reglas | Después de las reglas |
|---|---|---|
| `nslookup example.com 8.8.8.8` (DNS de fuera) | Responde con la IP | **`timed out`**: bloqueado |
| `nslookup example.com` (DNS del router) | Responde | **Responde**: no le afecta |
| Navegar con Firefox | Funciona | Funciona |

`nslookup` muestra `Server: 127.0.0.53`: es un intermediario de DNS que Ubuntu lleva dentro del propio equipo (`systemd-resolved`). Las aplicaciones le preguntan a él y él pregunta al DNS que le dio el DHCP (192.168.27.1).

**El registro (Log):** → [lista](<../evidencias/t5-mikrotik-14-log-firewall-20261117.png>) · [línea bloqueada](<../evidencias/t5-mikrotik-15-log-dns-bloqueado-20261117.png>) · [línea permitida](<../evidencias/t5-mikrotik-16-log-web-permitida-20261117.png>)

Línea del bloqueo:

```
2026-11-17 07:35:50  firewall,info  LAN-DNS-BLOQ forward: in:ether2 out:ether1, connection-state:new
src-mac 08:00:27:8B:D9:75, proto UDP, 192.168.27.199:36702->8.8.8.8:53, len 57
```

| Parte | Qué significa |
|---|---|
| `2026-11-17 07:35:50` | Cuándo pasó |
| `firewall, info` | Lo ha escrito el cortafuegos, como información (no es un error del router) |
| `LAN-DNS-BLOQ` | El prefijo que puse: dice **qué regla** lo ha apuntado (la del bloqueo) |
| `forward` | El paquete intentaba **atravesar** el router |
| `in:ether2 out:ether1` | Entraba por la LAN e iba a salir por la WAN |
| `connection-state:new` | Era el primer paquete de una conexión nueva |
| `src-mac 08:00:27:8B:D9:75` | La tarjeta de red que lo envió: la de `cli-linux` (la misma que en los préstamos del DHCP) |
| `proto UDP` | Protocolo UDP, el que usa normalmente el DNS |
| `192.168.27.199:36702` | **Quién**: el cliente, desde un puerto que elige él al azar |
| `->8.8.8.8:53` | **A dónde**: el DNS de Google, puerto 53 (DNS) |
| `len 57` | Tamaño del paquete: 57 bytes, una pregunta DNS pequeña |

**En una frase:** a las 07:35:50, el equipo 192.168.27.199 (`cli-linux`) intentó preguntar al DNS 8.8.8.8 y el router lo tiró por la regla `LAN-DNS-BLOQ`.

Línea de la permitida:

```
2026-11-17 07:37:12  firewall,info  LAN-WEB-OK forward: in:ether2 out:ether1, connection-state:new
src-mac 08:00:27:8B:D9:75, proto TCP (SYN), 192.168.27.199:40300->185.125.190.99:80, len 60
```

Se lee igual. Lo nuevo: **`TCP (SYN)`** es el primer paso de una conexión TCP (el cliente «llama a la puerta»), y el destino es el **puerto 80** (web sin cifrar). La IP 185.125.190.99 es de Canonical (Ubuntu): seguramente es la comprobación automática de Ubuntu para saber si hay Internet, que se hace por web.

**En una frase:** a las 07:37:12, `cli-linux` abrió una conexión web al puerto 80 de 185.125.190.99 y el router la dejó pasar por la regla `LAN-WEB-OK`.

⚠️ El registro está en **memoria** (`memory`): se borra al reiniciar el router. Para guardarlo habría que enviarlo a un servidor de registros o a disco.

### Copia de la configuración

**Dónde se guarda:** en el NAS ([TrueNAS](05-truenas.md)), en la carpeta `/mnt/copias/mikrotik`, con un usuario `mikrotik` que solo sirve para esto. Una copia que se queda solo en el router no sirve: si el router se rompe, la copia se pierde con él.

**Dos tipos de copia** (17/11/2026). Hago las dos porque sirven para cosas distintas:

| | **Backup** (`.backup`) | **Export** (`.rsc`) |
|---|---|---|
| Qué es | Una «foto» completa del router en un fichero que no se puede leer | Un fichero de texto con los comandos de la configuración |
| Incluye | Todo, también usuarios y contraseñas | La configuración, sin contraseñas |
| Sirve para | Dejar **el mismo router** como estaba | Leerla, compararla o aplicarla a **otro router** |
| Cómo se hace | **Files → Backup**, nombre `r-mikrotik-20261117` y **contraseña** (el fichero queda cifrado) | En **Terminal**: `/export file=r-mikrotik-20261117` |
| Tamaño | 21,6 KiB | 976 bytes |

Le pongo contraseña al backup porque dentro van las contraseñas del router: aunque alguien se llevara el fichero del NAS, no podría leerlas.

→ [ficheros creados en el router](<../evidencias/t5-mikrotik-17-ficheros-copia-20261117.png>) · [`/file print`](<../evidencias/t5-mikrotik-18-ficheros-router-20261117.png>)

**Subir las copias al NAS.** El router las envía por **SFTP** (copiar ficheros por la red de forma cifrada, sobre SSH) con su herramienta **fetch**. En **Terminal**:

> **⬆️ Comando para SUBIR una copia del MikroTik al NAS**
>
> ```
> /tool fetch upload=yes mode=sftp address=192.168.27.10 user=mikrotik password=******** src-path=r-mikrotik-20261117.backup dst-path=/mnt/copias/mikrotik/r-mikrotik-20261117.backup
> ```
>
> Para el export es el mismo comando cambiando `.backup` por `.rsc` en los dos nombres.

| Parte | Qué significa |
|---|---|
| `upload=yes` | **Subir** el fichero (sin esto, fetch descarga) |
| `mode=sftp` | Enviarlo por SFTP, cifrado |
| `address=192.168.27.10` | La IP del NAS |
| `user=… password=…` | El usuario `mikrotik` del NAS (la contraseña no la pongo aquí) |
| `src-path` | **Origen**: el fichero en el router |
| `dst-path` | **Destino**: dónde se guarda en el NAS |

Al acabar tiene que decir `status: finished`.

**Comprobación en el NAS**, entrando por SFTP desde `cli-linux` (`sftp mikrotik@192.168.27.10` y `ls -l`): → [captura](<../evidencias/t5-mikrotik-19-copias-en-nas-20261117.png>)

```
-rw-r--r--  ? mikrotik mikrotik  22149 Nov 17 11:25 r-mikrotik-20261117.backup
-rw-r--r--  ? mikrotik mikrotik    976 Nov 17 11:28 r-mikrotik-20261117.rsc
```

Los tamaños son **los mismos que en el router** (22149 bytes = 21,6 KiB; 976 bytes). Comparar el tamaño en el origen y en el destino es una forma sencilla de ver que la copia no ha llegado cortada.

**Problemas que tuve:**
- Al hacer el export escribí `file=r-mikrotik-20261117.backup` y RouterOS añadió `.rsc` detrás: salió `r-mikrotik-20261117.backup.rsc`. Funcionaba, pero el nombre confundía.
- Luego intenté subir `r-mikrotik-20261117.rsc`, que no existía en el router. El error fue `cannot open file: No such file or directory`: el fallo estaba en el **origen** (el router), no en el NAS. Aun así, el NAS creó un fichero vacío de 0 bytes con ese nombre.
- Lo arreglé dentro del NAS por SFTP: borré el fichero vacío (`rm`) y cambié el nombre del otro (`rename … r-mikrotik-20261117.rsc`).

### Fallo y recuperación

**Qué simulo:** un desastre doble. Alguien borra la regla de NAT (la LAN se queda sin Internet) **y además** se pierden los ficheros del router. Así la recuperación tiene que salir del NAS, no del propio router.

**1. Antes del fallo.** Desde `cli-linux`, `curl -I https://www.wikipedia.org` devuelve `HTTP/2 200` (la web responde «OK»). → [captura](<../evidencias/t5-mikrotik-20-antes-fallo-web-ok-20261117.png>)

Uso `curl` y no `ping` porque la red del aula no deja pasar el ping hacia Internet (se vio en la [T2](02-equipos.md)). `curl -I` pide a la web solo la cabecera de la respuesta, sin descargar la página.

**2. Provoco el fallo** en WebFig:
- **IP → Firewall → NAT**: borro la regla *masquerade*. → [NAT vacío](<../evidencias/t5-mikrotik-23-fallo-nat-vacio-20261117.png>)
- **Files**: borro los dos ficheros de copia. Solo queda `skins`, una carpeta del sistema. → [captura](<../evidencias/t5-mikrotik-22-fallo-ficheros-borrados-20261117.png>)

**3. Compruebo que está roto:** → [captura](<../evidencias/t5-mikrotik-21-fallo-web-timeout-20261117.png>)

```
curl -I --max-time 10 https://www.wikipedia.org
curl: (28) Connection timed out after 10003 milliseconds
```

`--max-time 10` = esperar como mucho 10 segundos. La petición sale con la IP privada 192.168.27.199, nadie en Internet sabe devolverle la respuesta y se agota el tiempo (*timed out*).

**4. Recupero desde el NAS.** El router **sí llega al NAS aunque no haya NAT**: el NAT solo hace falta para salir a Internet, y el NAS está dentro de la LAN. Esta es una ventaja de tener las copias en la red local.

> **⬇️ Comando para BAJAR la copia del NAS al MikroTik**
>
> ```
> /tool fetch mode=sftp address=192.168.27.10 user=mikrotik password=******** src-path=/mnt/copias/mikrotik/r-mikrotik-20261117.backup dst-path=r-mikrotik-20261117.backup
> ```
>
> Es el comando de subir **al revés**: sin `upload=yes` fetch **descarga**, y ahora el **origen** (`src-path`) es el NAS y el **destino** (`dst-path`) el router.

Después compruebo que el `.backup` ha vuelto al router y mide 21,6 KiB, igual que el original. Solo bajo el `.backup` porque es el que sirve para restaurar; el `.rsc` sigue guardado en el NAS. → [captura](<../evidencias/t5-mikrotik-26-backup-bajado-del-nas-20261117.png>)

Y lo cargo:

> **🔁 Comando para RESTAURAR la copia en el MikroTik**
>
> ```
> /system backup load name=r-mikrotik-20261117.backup password=********
> ```
>
> Aquí la contraseña es la que puse **al crear el backup**, no la del usuario del NAS. Pregunta `Restore and reboot? [y/N]`: con `y` carga la copia y **reinicia el router**.

Durante el reinicio (unos 20-30 segundos) WebFig pierde la conexión. Es normal.

**5. Compruebo la recuperación:**

| Prueba | Resultado | Evidencia |
|---|---|---|
| **IP → Firewall → NAT** | La regla 0 `masquerade`, `srcnat`, salida `ether1`, ha vuelto y ya tiene tráfico (5 paquetes) | [captura](<../evidencias/t5-mikrotik-24-recuperado-nat-20261117.png>) |
| `curl -I --max-time 10 https://www.wikipedia.org` | Primero `timed out` (antes de restaurar) y después `HTTP/2 200` | [captura](<../evidencias/t5-mikrotik-25-recuperado-web-ok-20261117.png>) |

**Resultado:** con la copia guardada en el NAS, el router ha vuelto a estar exactamente como antes del fallo, en pocos minutos y sin volver a configurar nada a mano.

**Resumen de los comandos de copia:**

| Para qué | Comando (en la Terminal del MikroTik) |
|---|---|
| Crear el backup | **Files → Backup** (o `/system backup save name=… password=…`) |
| Crear el export | `/export file=r-mikrotik-20261117` |
| ⬆️ **Subir** al NAS | `/tool fetch upload=yes mode=sftp address=192.168.27.10 user=mikrotik password=… src-path=FICHERO dst-path=/mnt/copias/mikrotik/FICHERO` |
| ⬇️ **Bajar** del NAS | `/tool fetch mode=sftp address=192.168.27.10 user=mikrotik password=… src-path=/mnt/copias/mikrotik/FICHERO dst-path=FICHERO` |
| 🔁 Restaurar | `/system backup load name=FICHERO password=…` |

## OPNsense

**Qué es:** un sistema operativo hecho solo para ser **router y cortafuegos**, gratuito. Está construido sobre **FreeBSD** (otro sistema de la familia Unix, no es Linux) y se administra por una página web.

**Mismo escenario que el MikroTik:** OPNsense ocupa el mismo sitio: la WAN en la NAT de VirtualBox y la LAN en `lan-iso` con la IP **192.168.27.1**. Así el NAS y los clientes siguen funcionando sin tocarlos. Antes de encenderlo **apagué el MikroTik** (`/system shutdown`): con los dos encendidos habría dos equipos con la misma IP y dos servidores DHCP en la misma red, y los clientes recibirían datos de uno u otro al azar.

### Instalación

- **VM:** 2 vCPU, 2 GB de RAM, disco de 20 GB, arranque BIOS. Tarjeta 1 → NAT (WAN). Tarjeta 2 → `lan-iso` (LAN).
- **Orden de arranque:** primero el disco y después el DVD. Con el disco vacío arranca del DVD; al terminar la instalación arranca solo del disco. Así evité el problema que tuve con TrueNAS (volvía a salir el instalador).
- La ISO arranca en **modo live** (funciona sin instalar). El instalador se lanza entrando con el usuario `installer` y la contraseña `opnsense`, que son los que trae OPNsense de serie.
- Opciones: teclado español, **UFS** en vez de ZFS (UFS es más ligero y va mejor con 2 GB de RAM y un solo disco), disco de 20 GB y contraseña de `root`.
- Durante *Preparing target system* pareció quedarse parado unos minutos. Comprobé desde fuera que el disco ya tenía el sistema copiado (unos 2,4 GB) y esperé; terminó solo.

**Primer arranque:** → [consola](<../evidencias/t5-opnsense-01-consola-inicial-20261117.png>)

OPNsense había puesto las tarjetas **al revés** (`LAN (em0)`, `WAN (em1)`) y la LAN en 192.168.1.1, que es lo que trae de serie: la primera tarjeta como LAN. Lo arreglé en el menú de la consola (ver *Red WAN y LAN*).

**Actualización** (**System → Firmware → Status → Check for updates**):

| | Antes | Después |
|---|---|---|
| Versión | 26.7 | **26.7.9_3** |
| Actualizaciones | 96 paquetes, 298,8 MiB, pide reinicio | *There are no updates available* |

- 26.7 → 26.7.9 son **9 rondas de correcciones** de la misma versión, no una versión nueva. Incluía arreglos de seguridad en `openssl` (cifrado), `openssh` (SSH), `curl` y las VPN.
- La hice **antes de configurar nada**: si algo salía mal, se perdía muy poco. En un router en uso se haría fuera de horario, porque durante el reinicio la LAN se queda sin Internet.

→ [lista de actualizaciones](<../evidencias/t5-opnsense-05-actualizaciones-lista-20261117.png>) · [resumen](<../evidencias/t5-opnsense-06-actualizaciones-resumen-20261117.png>) · [sin actualizaciones](<../evidencias/t5-opnsense-07-sin-actualizaciones-20261117.png>) · [versión](<../evidencias/t5-opnsense-08-version-actualizada-20261117.png>)

**Asistente inicial** (**System → Configuration → Wizard**): → [terminado](<../evidencias/t5-opnsense-09-asistente-terminado-20261117.png>)

| Pantalla | Lo que puse | Por qué |
|---|---|---|
| General | Nombre `r-opnsense`, dominio `local`, zona horaria **Europe/Madrid** | Con la hora en UTC (la de serie), las horas del registro irían una hora por detrás del reloj |
| | *Override DNS* ✅, *Enable Resolver* ✅, DNSSEC no | Usar el DNS que recibe la WAN y que el router conteste las preguntas DNS de la LAN |
| WAN | DHCP. **Block RFC1918 Private Networks: desmarcado** | Esa casilla bloquea lo que llega a la WAN desde redes privadas. Nuestra WAN **está** en una red privada (la NAT de VirtualBox, 10.0.2.x), igual que un router detrás del router de casa |
| | *Block bogon networks* ✅ | Las direcciones *bogon* no son de nadie en Internet: si llega algo desde ellas, es falso |
| LAN | 192.168.27.1/24, DHCP ✅ | |
| Tipo de instalación | *Multiwan* ✗, *DHCP/DNS registration* ✅, *IPsec* ✗ | Solo hay una WAN y no hay VPN. El registro DHCP/DNS hace que los equipos se puedan encontrar por su nombre |
| Contraseña | La de `root` | |

### Usuario de administración

**System → Access → Users → +**: usuario **`alvaro`**, grupo **`admins`** (*unrestricted*, puede hacer todo). Cerré la sesión de `root` y entré con `alvaro`: arriba sale `alvaro@r-opnsense.local` y se ven todos los menús. → [usuarios](<../evidencias/t5-opnsense-15-usuarios-20261117.png>) · [sesión](<../evidencias/t5-opnsense-16-sesion-alvaro-20261117.png>)

**Diferencia con el MikroTik:** allí desactivé `admin`; aquí **dejo `root` activo como cuenta de emergencia**. `root` es quien entra en el **menú de la consola** (reasignar tarjetas, cambiar la IP de la LAN, restaurar una copia…). Si un día la web no funciona, la consola es la única forma de arreglar el router. Reparto: `alvaro` para el día a día por la web, `root` solo para emergencias, con una contraseña fuerte guardada aparte.

### Red WAN y LAN

Desde el menú de la consola, entrando con `root`:

- **Opción 1 (*Assign interfaces*):** LAGG no, VLAN no, **WAN = `em0`** (tarjeta 1, NAT), **LAN = `em1`** (tarjeta 2, `lan-iso`). `em0` y `em1` son los nombres que da FreeBSD a las tarjetas, en el mismo orden que en VirtualBox.
- **Opción 2 (*Set interface IP address*), LAN:** sin DHCP, IP 192.168.27.1/24, sin puerta de enlace, sin IPv6, servidor DHCP de .100 a .199, mantener la web en HTTPS.

→ [interfaces asignadas](<../evidencias/t5-opnsense-02-interfaces-asignadas-20261117.png>)

| Interfaz | Tarjeta | IP | Cómo |
|---|---|---|---|
| WAN | `em0` (NAT) | 10.0.2.15/24 (y una IPv6 privada `fd17:…`) | DHCP de la NAT de VirtualBox |
| LAN | `em1` (`lan-iso`) | 192.168.27.1/24 | Fija |

**DHCP** (**Services → Dnsmasq DNS & DHCP → DHCP ranges**). En esta versión el DHCP lo da **Dnsmasq**, un programa pequeño que hace DHCP y DNS.
- **Problema:** el asistente **había cambiado el rango** que puse en la consola: lo dejó en **.41 – .245**. → [captura](<../evidencias/t5-opnsense-13-dhcp-rango-asistente-20261117.png>)
- La .10 del NAS quedaba fuera, pero lo devolví a **.100 – .199** para tener el mismo escenario que con el MikroTik y dejar las IP bajas para equipos con IP fija. → [corregido](<../evidencias/t5-opnsense-14-dhcp-rango-corregido-20261117.png>)
- **Lección:** revisar siempre lo que deja hecho un asistente automático.

**Cliente:** `cli-linux` volvió a pedir IP (`sudo nmcli device disconnect enp0s3` y `connect`) y recibió la **.199** con `valid_lft 86381sec`: **24 horas** de préstamo, lo que trae OPNsense de serie (el MikroTik daba 30 min). Por eso se sabe que la IP la ha dado OPNsense y no el MikroTik. → [captura](<../evidencias/t5-opnsense-03-cliente-dhcp-20261117.png>)

### Salida a Internet

**Diferencia con el MikroTik:** OPNsense **trae hecho de serie** el NAT de salida (*Outbound NAT* automático: hace el *masquerade* solo) y una regla que deja salir todo desde la LAN. En el MikroTik los tuve que crear a mano. → [dashboard](<../evidencias/t5-opnsense-04-dashboard-20261117.png>)

**Problema con el DNS:** la web no cargaba. → [captura](<../evidencias/t5-opnsense-10-problema-dns-20261117.png>)

```
curl -I --max-time 10 https://www.wikipedia.org
curl: (28) Resolving timed out after 10000 milliseconds
```

- **Pista:** dice ***Resolving*** timed out, no *Connection* timed out. Falla la **traducción del nombre** (DNS), no la conexión.
- **Causa:** el DNS de OPNsense (**Unbound**) viene como **resolvedor**: no pregunta a otro DNS, va él mismo a los servidores raíz de Internet y baja paso a paso. La red del aula no lo deja salir así, igual que bloquea el ping. El MikroTik funcionaba porque **reenviaba** las preguntas al DNS de la NAT de VirtualBox (10.0.2.3).
- **Solución:** **Services → Unbound DNS → Query Forwarding → Use System Nameservers** ✅. Ahora Unbound reenvía a 10.0.2.3, como el MikroTik. → [captura](<../evidencias/t5-opnsense-11-dns-reenvio-20261117.png>)
- La fila `local → 127.0.0.1:53053` la pone OPNsense solo: las preguntas por nombres de la LAN (`*.local`) se las pasa a Dnsmasq, que conoce los nombres porque es quien reparte las IP.

**Comprobación:** → [captura](<../evidencias/t5-opnsense-12-cliente-dns-web-ok-20261117.png>)

| Prueba | Resultado |
|---|---|
| `resolvectl status enp0s3` | DNS: 192.168.27.1 (el router) |
| `resolvectl query www.wikipedia.org` | 185.15.58.224 en 24,3 ms |
| `curl -I --max-time 10 https://www.wikipedia.org` | `HTTP/2 200` |

### Conexión permitida y bloqueada

Las mismas dos reglas que en el MikroTik, para poder comparar.

**Alias `puertos_web`** (**Firewall → Aliases**): tipo *Port(s)*, contenido **80 y 443**. Un **alias** es un nombre que agrupa varios puertos o direcciones: así hago **una** regla para los dos puertos en vez de dos reglas. → [captura](<../evidencias/t5-opnsense-17-alias-puertos-web-20261118.png>)

**Reglas** (**Firewall → Rules → LAN**): → [captura 1](<../evidencias/t5-opnsense-18-reglas-lan-a-20261118.png>) · [captura 2](<../evidencias/t5-opnsense-19-reglas-lan-b-20261118.png>)

| Descripción | Acción | Protocolo | Origen | Destino | Puerto | Log |
|---|---|---|---|---|---|---|
| `LAN-WEB-OK` | **Pass** (dejar pasar) | TCP | LAN network | cualquiera | `puertos_web` | ✅ |
| `LAN-DNS-BLOQ` | **Block** (tirar) | UDP | LAN network | **! This Firewall** | DOMAIN (53) | ✅ |

**Conceptos del formulario:**
- **Quick ✅:** «si coincide, decide ya y no sigas mirando». Hace que **gane la primera regla que coincide**, como en el MikroTik.
- **Block = `drop`** del MikroTik: tira el paquete sin avisar. *Reject* avisaría al cliente.
- **Keep state:** el cortafuegos **recuerda la conexión**; si dejó salir la petición, deja volver la respuesta sin otra regla.
- **`! This Firewall`** (*Invert Destination* + *This Firewall*): «a cualquier sitio **menos al router**». En el MikroTik la regla iba en `forward` (solo el tráfico que atraviesa el router); en OPNsense las reglas de la LAN miran **todo** lo que entra por la LAN, también lo que va al propio router. Sin el `!`, la regla bloquearía también el DNS del router.
- **DOMAIN (53):** es el puerto 53. OPNsense usa los nombres oficiales de los puertos, y el 53 se llama *domain* (de *Domain Name System*).
- **Description:** es lo que aparece en el registro para saber qué regla actuó, como el *log prefix* del MikroTik.

**Qué hace cada regla y qué pasa si falta o se rompe.** OPNsense trae de serie en la LAN la regla **«Default allow LAN to any»**, que deja pasar todo lo que sale de la LAN. Por eso pasa lo mismo que en el MikroTik: lo que no coincide con mis reglas, pasa.

| Regla | Qué hace, en sencillo | Si no la creo o la borro | Si la hago mal |
|---|---|---|---|
| **NAT de salida automático** (de serie) | El *masquerade*: cambia la IP privada del cliente por la de la WAN | La LAN se queda **sin Internet**, igual que al borrar el `masquerade` del MikroTik. Pasaría si cambio *Firewall → NAT → Outbound* a modo manual sin crear reglas | — |
| **Default allow LAN to any** (de serie) | Deja salir todo desde la LAN | Si la quito, **solo pasa lo que permitan mis reglas**. `LAN-WEB-OK` deja pasar la web, pero **nada permite el DNS al router**: los clientes no podrían traducir nombres y **tampoco navegarían**. Habría que añadir otra regla que permita UDP 53 hacia el router. Es la forma más segura de trabajar (*todo prohibido salvo lo permitido*), pero obliga a pensar todo lo que hace falta | — |
| **`LAN-WEB-OK`** (Pass, TCP 80/443) | Deja pasar la web y **la apunta en el registro** | Con la regla de serie activa, **la web sigue funcionando**: solo se pierde el registro. Sin la regla de serie, sería **imprescindible**: sin ella no cargaría ninguna web | Si el alias solo tuviera el 80, las webs HTTPS no quedarían apuntadas (y sin la regla de serie, **no cargarían**) |
| **`LAN-DNS-BLOQ`** (Block, UDP 53 a `! This Firewall`) | Tira las preguntas DNS que van a un DNS de fuera | Cualquier equipo puede saltarse el DNS de la empresa poniendo 8.8.8.8 a mano, y no queda apuntado | **Sin el `!` (Invert):** bloquea también el DNS **del router**, los clientes no traducen nombres y **no navega nada** (`Resolving timed out`). **Si queda por debajo de la regla de serie:** nunca actúa, porque la de serie deja pasar el DNS antes |
| **Alias `puertos_web`** | Junta 80 y 443 bajo un nombre | Habría que hacer una regla por puerto. OPNsense **no deja borrar** un alias que está usando una regla | Si quito un puerto del alias, la regla deja de cubrirlo |

**Problema: la regla de bloqueo no bloqueaba.** Después de crear las reglas, `nslookup example.com 8.8.8.8` seguía contestando. Cómo lo encontré:

1. **Comprobé que estaban aplicadas** (botón *Apply*): lo estaban, y seguía sin bloquear.
2. **Miré el registro** (**Firewall → Log Files → Live View**, buscando `8.8.8.8`): solo salían líneas de la **WAN** con la regla *let out anything from firewall host itself* (el paquete ya pasado por el NAT, con origen 10.0.2.15). **No había ninguna línea de la LAN.** Si `LAN-DNS-BLOQ` hubiera actuado, saldría en rojo, porque tiene el registro activado. Conclusión: otra regla **sin registro** dejaba pasar el paquete antes. → [captura](<../evidencias/t5-opnsense-20-liveview-solo-wan-20261118.png>)
3. **Miré la tabla de estados** (**Firewall → Diagnostics → States**): es la lista de conexiones abiertas, como *Connections* en el MikroTik. La columna **Rule** dice qué regla aceptó cada conexión: todas las de la LAN ponían **«Default allow…»**. → [captura](<../evidencias/t5-opnsense-22-estados-default-allow-20261118.png>)
4. **Volví a la lista de reglas** viéndola entera: las dos reglas de serie *Default allow LAN to any* (IPv4 e IPv6) estaban **por encima** de las mías. → [captura](<../evidencias/t5-opnsense-21-reglas-orden-mal-20261118.png>)

**Causa:** se gana con la primera regla que coincide (*Quick*). El DNS a 8.8.8.8 ya coincidía con *Default allow* (deja pasar todo) y nunca llegaba a `LAN-DNS-BLOQ`.

**Solución:** marqué mis dos reglas y, en la fila de *Default allow LAN to any rule*, pulsé la flecha **←** (*mover las seleccionadas delante de esta*). Después, **Apply**. → [orden correcto](<../evidencias/t5-opnsense-23-reglas-orden-bien-20261118.png>)

| Orden | Regla | Registro |
|---|---|---|
| 1 | `LAN-WEB-OK` | ✅ |
| 2 | `LAN-DNS-BLOQ` | ✅ |
| 3 | Default allow LAN to any (IPv4) | ✗ (campana tachada) |
| 4 | Default allow LAN IPv6 to any | ✗ |

**Lección:** una regla bien hecha **en el sitio equivocado no hace nada**. Y el registro sirve para diagnosticar: que una regla con log **no deje ninguna línea** ya es una pista.

**Pruebas desde `cli-linux`** (ya con el orden correcto): → [captura](<../evidencias/t5-opnsense-24-prueba-dns-bloqueado-20261118.png>)

| Prueba | Resultado | Qué significa |
|---|---|---|
| `nslookup example.com 8.8.8.8` | `communications error … timed out` tres veces y `no servers could be reached` | **Bloqueado.** `nslookup` lo intenta 3 veces y se rinde |
| `nslookup example.com` | Responde con las IP | El DNS del router funciona: el `!` lo deja fuera del bloqueo |
| `curl -I https://www.wikipedia.org` | `HTTP/2 200` | La web funciona |

**El registro (Live View):** → [bloqueo](<../evidencias/t5-opnsense-25-log-bloqueo-20261118.png>) · [web](<../evidencias/t5-opnsense-26-log-web-20261118.png>)

| Interface | Dir | Hora | Protocolo | Origen → Destino | Acción | Regla |
|---|---|---|---|---|---|---|
| LAN | In | 09:24:18 | UDP | 192.168.27.199 → 8.8.8.8:53 | **block** (rojo) | `LAN-DNS-BLOQ` |
| LAN | In | 09:24:23 | UDP | 192.168.27.199 → 8.8.8.8:53 | **block** | `LAN-DNS-BLOQ` |
| LAN | In | 09:24:28 | UDP | 192.168.27.199 → 8.8.8.8:53 | **block** | `LAN-DNS-BLOQ` |
| LAN | In | 09:25:17 | TCP | 192.168.27.199 → 185.15.5x.x | **pass** (verde) | `LAN-WEB-OK` |
| WAN | Out | 09:25:17 | TCP | 10.0.2.15 → 185.15.5x.x | pass | *let out anything from firewall host itself* |

Cómo se lee:
- **Las tres líneas rojas, separadas 5 segundos:** son los **tres intentos** de `nslookup`. Espera 5 segundos la respuesta, no llega (el paquete se ha tirado) y vuelve a probar. Coincide con los tres `timed out` de la terminal.
- **`LAN In`:** el paquete **entraba** al router por la LAN. Ahí es donde actúan mis reglas.
- **La web sale dos veces a la misma hora (09:25:17):** es **la misma conexión** vista en dos puntos. Primero entra por la LAN desde 192.168.27.199 (`LAN-WEB-OK`); después sale por la WAN con origen **10.0.2.15**, porque el NAT ya ha cambiado la IP del cliente por la del router. La IP 185.15.5x.x es de Wikipedia.
- **`anti-lockout rule`:** es mi Firefox entrando a la web de administración del router (192.168.27.1). Es la regla de serie que nunca deja bloquear ese acceso desde la LAN.
- **La hora:** el Live View la escribe en **UTC** (la `T` y el formato `2026-11-18T09:24:18` son el estándar internacional). En España, en invierno, es una hora más: 10:24:18.

**En una frase:** a las 10:24 (hora de España), `cli-linux` intentó tres veces preguntar al DNS 8.8.8.8 y el router lo tiró las tres veces por la regla `LAN-DNS-BLOQ`; a las 10:25 abrió una web de Wikipedia y el router la dejó pasar y la apuntó con `LAN-WEB-OK`.

### Copia de la configuración

**Qué permite OPNsense** (**System → Configuration → Backups**): → [descarga](<../evidencias/t5-opnsense-27-backups-descarga-20261118.png>) · [restaurar](<../evidencias/t5-opnsense-28-backups-restaurar-20261118.png>)

| Opción | Qué hace |
|---|---|
| **Backup Count** | Cuántas versiones antiguas de la configuración guarda el router **dentro de sí mismo**. No es una copia de seguridad: si el router muere, se pierden con él |
| **Download configuration** | Descarga toda la configuración en un fichero **XML** (texto con etiquetas, como el HTML) |
| **Do not backup RRD data** ✅ | Deja fuera los datos de las **gráficas** (tráfico, CPU…): no son configuración y harían el fichero mucho más grande |
| **Encrypt this configuration file** ✅ | Cifra la copia con una contraseña. Hay que marcarla: el fichero lleva las contraseñas de los usuarios y las claves de los certificados |

**Diferencia con el MikroTik:** esta versión **solo deja descargar** la copia al navegador; no tiene un comando para enviarla a otro servidor como `/tool fetch`. Existen complementos (*plugins*) para mandarla a la nube, pero no a un servidor SFTP propio. Por eso el camino es:

**OPNsense → (descargar con el navegador) → `cli-linux` → (SFTP) → TrueNAS**

Es lo que haría un administrador desde su equipo. La desventaja es que la copia **la tiene que hacer una persona**; el MikroTik la sube solo.

**En el NAS:** dataset **`copias/opnsense`** y usuario **`opnsense`** (solo SSH, con su carpeta `/mnt/copias/opnsense`), igual que para el MikroTik: un usuario y una carpeta por equipo. → [usuario](<../evidencias/t5-truenas-14-usuario-opnsense-20261118.png>) · [dataset](<../evidencias/t5-truenas-15-dataset-opnsense-20261118.png>)

**Pasos:**
1. En OPNsense: marcar *Encrypt*, poner contraseña y **Download configuration**. Se guarda en `cli-linux` como `config-r-opnsense.local-20261118093237.xml` (nombre del router, fecha y hora).
2. Subirla al NAS desde la terminal de `cli-linux`:

> **⬆️ Subir la copia de OPNsense al NAS** (desde `cli-linux`)
>
> ```
> cd ~/Downloads
> sftp opnsense@192.168.27.10
> sftp> put config-r-opnsense.local-20261118093237.xml
> sftp> ls -l
> sftp> bye
> ```

Resultado: `100 %, 59 KB` y en el NAS `-rw-r--r-- opnsense opnsense 60067 … config-r-opnsense.local-20261118093237.xml`. → [captura](<../evidencias/t5-opnsense-29-copia-subida-nas-20261118.png>)

La huella SSH del NAS que pidió confirmar (`SHA256:js+p3aiajK0n…`) es **la misma** que la del día anterior: es el mismo NAS. La volvió a preguntar porque `cli-linux` es un sistema live y se olvida de todo al reiniciar.

**Comprobar que va cifrada:** `head -c 200 config-r-opnsense*.xml` → [captura](<../evidencias/t5-opnsense-30-copia-cifrada-20261118.png>)

```
---- BEGIN config.xml ----
Version: OPNsense 26.7.9_3
Cipher: AES-256-CBC
PBKDF2: 100000
Hash: SHA512

U2FsdGVkX19xIAH3FNzH4AKCu67ee+RoSTF/zn1XoK+…
```

| Línea | Qué significa |
|---|---|
| `---- BEGIN config.xml ----` | La marca de OPNsense de que va cifrado |
| `Version` | La versión de la que se sacó. Para restaurar, mejor la misma |
| `Cipher: AES-256-CBC` | El **algoritmo de cifrado**: AES con clave de 256 bits |
| `PBKDF2: 100000` | La contraseña se pasa **100 000 veces** por una función antes de usarla. Para mí es un instante; a quien intente adivinarla probando millones de contraseñas le hace ir 100 000 veces más lento |
| `Hash: SHA512` | La función de esas vueltas (familia del SHA256 de las ISO) |
| `U2FsdGVkX19…` | La configuración ya cifrada: sin la contraseña no se puede leer |

### Fallo y recuperación

**Qué simulo:** el mismo desastre doble que en el MikroTik: se rompe la salida a Internet, se borran las reglas **y** se pierde la copia del equipo del administrador. La recuperación tiene que salir del NAS.

**1. Antes del fallo** (08:44:46 UTC): `curl -I https://www.wikipedia.org` → `HTTP/2 200`. → [captura](<../evidencias/t5-opnsense-31-antes-fallo-web-ok-20261118.png>)

**2. Provoco el fallo:**
- **Firewall → NAT → Source NAT** → modo **Disable Source NAT rule generation** → *Apply*. Es como borrar el `masquerade` del MikroTik. → [captura](<../evidencias/t5-opnsense-32-fallo-nat-desactivado-20261118.png>)
  - En esta versión, lo que antes se llamaba *Outbound NAT* se llama **Source NAT** (NAT de **origen**: cambia la IP de origen al salir, como la chain `srcnat` del MikroTik). **Destination NAT** es lo contrario: cambia el destino al entrar; sirve para abrir puertos.
- **Firewall → Rules**: borro `LAN-WEB-OK` y `LAN-DNS-BLOQ` → *Apply*. Quedan solo las *Default allow*. → [captura](<../evidencias/t5-opnsense-33-fallo-reglas-borradas-20261118.png>)
- En `cli-linux`: `rm config-r-opnsense…xml` y `ls` vacío.

**3. Compruebo que está roto** (08:49:20 UTC): → [captura](<../evidencias/t5-opnsense-34-fallo-sin-copia-sin-web-20261118.png>)

```
curl: (28) Connection timed out after 10003 milliseconds
```

Esta vez es ***Connection*** timed out y no *Resolving*: el DNS funciona (lo pregunta el propio router, que no necesita NAT), pero la conexión del cliente sale con IP privada y no vuelve.

**4. Recupero desde el NAS** (inicio: **09:06:00 UTC**):

> **⬇️ Bajar la copia de OPNsense del NAS** (desde `cli-linux`)
>
> ```
> cd ~/Downloads
> sftp opnsense@192.168.27.10
> sftp> get config-r-opnsense.local-20261118093237.xml
> sftp> bye
> ```
>
> `get` es lo contrario de `put`: **trae** el fichero del servidor a mi equipo.

→ [captura](<../evidencias/t5-opnsense-36-copia-bajada-del-nas-20261118.png>)

> **🔁 Restaurar en OPNsense**
>
> **System → Configuration → Backups → Restore**: *Restore areas* **All**, **Browse…** → el fichero, marcar **Configuration file is encrypted** y poner la contraseña de la copia, dejar **Reboot after a successful restore** → **Restore configuration**.

Mensaje: *The configuration has been restored. The system is rebooting now.* → [captura](<../evidencias/t5-opnsense-35-restaurado-20261118.png>)

**5. Compruebo la recuperación:**

| Prueba | Resultado | Evidencia |
|---|---|---|
| **Source NAT** | Vuelve a **Automatic Source NAT rule generation** | [captura](<../evidencias/t5-opnsense-38-recuperado-nat-20261118.png>) |
| **Rules (LAN)** | `LAN-WEB-OK` y `LAN-DNS-BLOQ` han vuelto, **en su sitio** (por encima de las *Default allow*) | [captura](<../evidencias/t5-opnsense-37-recuperado-reglas-20261118.png>) |
| Firefox → Google «hora exacta» | Carga: **10:11:30 CET** (09:11:30 UTC) | [captura](<../evidencias/t5-opnsense-39-recuperado-internet-20261118.png>) |

**Tiempo de recuperación: unos 5 minutos y medio** (de 09:06:00 a 09:11:30 UTC), contando la descarga del NAS, la restauración, el reinicio del router y un error mío (ver abajo).

**Problemas que tuve durante la recuperación:**
- **`cli-linux` iba muy lento** (Firefox *Not Responding*, el medidor de CPU de VirtualBox en rojo): sus 2 procesadores estaban al 100 %. El portátil estaba bien (18 núcleos y 14 GB libres). Le subí la VM de 3 a **4 GB de RAM** y de 2 a **4 procesadores**. Al ser un sistema live, guarda todo en la RAM, y las webs de OPNsense y TrueNAS pesan mucho por sus gráficas en directo.
- **No arrancaba** (*No bootable option or device was found*): al apagarse, Ubuntu live **expulsa el DVD**, y el disco de `cli-linux` está vacío porque todavía no está instalado. Volví a meter la ISO en el lector virtual. Mientras no esté instalado, hay que hacerlo cada vez que se apague.
- **Error al restaurar** (*The uploaded file could not be decrypted*): en *Browse* había elegido **otro fichero** (el acceso directo del instalador de Ubuntu del escritorio), no la copia. OPNsense intentó descifrarlo y no pudo. Con el fichero correcto funcionó a la primera.

**Resultado:** con la copia del NAS, el router ha vuelto exactamente como estaba: NAT automático y las dos reglas **en el mismo orden**, que es justo lo que costó dejar bien.

## GNU/Linux (Debian) con nftables

**Qué es:** un Debian normal, sin escritorio, al que **convierto en router a mano** con programas sueltos:

| Pieza | Qué hace | En el MikroTik / OPNsense lo hacía… |
|---|---|---|
| `ip_forward` (ajuste del núcleo de Linux) | Deja que el equipo **pase paquetes** de una tarjeta a otra | Lo traen hecho de serie |
| **nftables** (`nft`) | El **cortafuegos** de Linux: reglas de filtrado y NAT | IP → Firewall / Firewall → Rules |
| **dnsmasq** | Servidor **DHCP y DNS** pequeño para la LAN | IP → DHCP Server / Dnsmasq |

**Mismo escenario:** WAN en la NAT de VirtualBox y LAN en `lan-iso` con la IP **192.168.27.1**. Antes de encenderlo apagué los otros dos routers (misma IP y mismo DHCP en la misma red).

**Cómo lo administro:** por **terminal**, conectándome por **SSH** desde el portátil. Igual que con WebFig, puse en VirtualBox una **redirección de puerto**: `127.0.0.1:2222` del portátil → puerto 22 (SSH) del router. Así puedo **copiar y pegar** comandos, cosa que en la ventana de VirtualBox no se puede. Entro con `ssh -p 2222 alvaro@127.0.0.1`. → [captura](<../evidencias/t5-nftables-10-ssh-desde-portatil-20261119.png>)

### Instalación

- **VM:** 1 vCPU, **1 GB de RAM** (con 512 MB el instalador no arrancaba), disco de 8 GB, BIOS. Tarjeta 1 → NAT (WAN), tarjeta 2 → `lan-iso` (LAN).
- Debian 13.7 *netinst* (ISO comprobada con su hash, ver arriba).
- **Programas:** solo **servidor SSH** y **utilidades estándar**, sin escritorio: un router no necesita entorno gráfico, y cuanto menos instalado, menos que actualizar y menos fallos de seguridad. → [selección](<../evidencias/t5-nftables-01-seleccion-programas-20261118.png>) · [primer arranque](<../evidencias/t5-nftables-02-primer-arranque-20261118.png>)

### Usuario de administración

- Administro con el usuario **`alvaro`**, que tiene permiso de **`sudo`** (ejecutar órdenes como administrador poniendo `sudo` delante). Por SSH no se entra como `root`.
- **`root` está bloqueado:** en la instalación dejé su contraseña vacía y Debian lo bloquea solo. Se comprueba con `sudo passwd -S root`: la `L` (*locked*) de la segunda columna significa bloqueado; nadie puede entrar como `root` con contraseña. → [captura](<../evidencias/t5-nftables-26-root-bloqueado-20261119.png>)
- **Ventaja:** cada orden de administrador queda apuntada en el registro con el nombre de quien la hizo, y para hacer algo peligroso hay que escribir `sudo` a propósito.

### Red WAN y LAN

**Ver las tarjetas** (`ip -br a`): → [captura](<../evidencias/t5-nftables-03-tarjetas-antes-20261119.png>)

| Tarjeta | Red | IP |
|---|---|---|
| `enp0s3` | NAT (WAN) | 10.0.2.15, por DHCP (el instalador ya la dejó configurada) |
| `enp0s8` | `lan-iso` (LAN) | Ninguna todavía |

**IP fija de la LAN.** Añadí al final de `/etc/network/interfaces`: → [fichero](<../evidencias/t5-nftables-04-interfaces-20261119.png>) · [IP puesta](<../evidencias/t5-nftables-05-lan-ip-fija-20261119.png>)

```
# LAN (red lan-iso)
auto enp0s8
iface enp0s8 inet static
    address 192.168.27.1/24
```

Sin `gateway`: la puerta de enlace (salida a Internet) la da la WAN.

**Dejar pasar paquetes (`ip_forward`).** Linux, de serie, **no reenvía** paquetes entre tarjetas: los que no son para él los tira. Lo activo con un fichero propio, para que no se pierda al reiniciar: → [captura](<../evidencias/t5-nftables-06-ip-forward-20261119.png>)

```
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-router.conf
sudo sysctl --system
```

Si falta: los clientes reciben IP y llegan al router, pero **nada pasa a Internet**.

**DHCP y DNS para la LAN** con dnsmasq, en `/etc/dnsmasq.d/lan.conf`: → [funcionando](<../evidencias/t5-nftables-11-dnsmasq-funcionando-20261119.png>)

```
interface=enp0s8
bind-interfaces
dhcp-range=192.168.27.100,192.168.27.199,255.255.255.0,12h
dhcp-option=option:router,192.168.27.1
dhcp-option=option:dns-server,192.168.27.1
```

| Línea | Qué hace | Qué pasa si falta |
|---|---|---|
| `interface=enp0s8` | Solo atiende por la LAN | Daría DHCP **hacia la WAN** también |
| `bind-interfaces` | Escucha solo en esa tarjeta | Puede chocar con otros servicios que usan el puerto 53 |
| `dhcp-range` | Reparte de la .100 a la .199, préstamos de 12 h | No da IP a nadie |
| `option:router` | Dice a los clientes que su puerta de enlace es .1 | Los clientes no saben por dónde salir |
| `option:dns-server` | Dice a los clientes que su DNS es .1 | No traducen nombres de web |

dnsmasq **reenvía** las preguntas DNS al DNS que tiene el propio router (el de la NAT de VirtualBox), como hacía el MikroTik. Así no tuve el problema del DNS de OPNsense.

**Cliente:** `cli-linux` recibió la **192.168.27.136** con puerta de enlace .1. → [captura](<../evidencias/t5-nftables-14-cliente-ip-dhcp-20261119.png>)

### Salida a Internet

El NAT (*masquerade*) va en el fichero de nftables, junto con las reglas (siguiente apartado).

**Problema: `apt` no podía instalar nada.** → [error](<../evidencias/t5-nftables-07-apt-error-http-bloqueado-20261119.png>)

```
No se pudo conectar a deb.debian.org:http
```

- **Causa:** la red donde estaba **bloquea el puerto 80 (HTTP)**; el 443 (HTTPS) sí pasa. Los repositorios de Debian venían con `http://`. → [repositorios](<../evidencias/t5-nftables-08-repositorios-http-20261119.png>)
- **Solución:** cambiar `http://` por `https://` en todo el fichero:

```
sudo sed -i 's|http://|https://|g' /etc/apt/sources.list
```

`sed -i` = editar el fichero **en el sitio**; `s|A|B|g` = sustituir A por B, todas las veces. Después `apt install dnsmasq` funcionó. → [captura](<../evidencias/t5-nftables-09-apt-install-https-20261119.png>)

En `cli-linux` (Ubuntu) pasó lo mismo con `apt install curl`, y en el registro del router se veían los intentos al puerto 80 pasando bien por él: el bloqueo **no es de mi router**, es de la red de fuera. → [captura](<../evidencias/t5-nftables-15-cliente-apt-http-bloqueado-20261119.png>). Por eso en las pruebas uso **`wget`**, que Ubuntu ya trae.

### Conexión permitida y bloqueada

Las mismas dos reglas que en los otros routers. Todo va en **un fichero de texto**, `/etc/nftables.conf`, que se carga al arrancar: → [fichero](<../evidencias/t5-nftables-12-fichero-reglas-20261119.png>) · [reglas cargadas](<../evidencias/t5-nftables-13-reglas-cargadas-20261119.png>)

```
flush ruleset
table inet filtro {
    chain forward {
        type filter hook forward priority filter; policy accept;
        iifname "enp0s8" oifname "enp0s3" tcp dport { 80, 443 } ct state new log prefix "LAN-WEB-OK " accept
        iifname "enp0s8" udp dport 53 log prefix "LAN-DNS-BLOQ " drop
    }
}
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "enp0s3" masquerade
    }
}
```

| Parte | Qué significa |
|---|---|
| `flush ruleset` | Borra todas las reglas antes de cargar las del fichero (si no, se sumarían a las que ya hay) |
| `table` / `chain` | Las reglas van en **cadenas**, y las cadenas en **tablas** |
| `hook forward` | La cadena mira los paquetes que **atraviesan** el router (de la LAN a Internet). Lo que va **al propio router** pasa por otra cadena, `input` |
| `policy accept` | Lo que no coincida con ninguna regla, pasa |
| `iifname` / `oifname` | Tarjeta por la que **entra** / por la que **sale** |
| `tcp dport { 80, 443 }` | Puerto de destino 80 o 443 (web). Las llaves agrupan, como el alias de OPNsense |
| `ct state new` | Solo el **primer paquete** de cada conexión: así el registro apunta una línea por conexión y no una por paquete |
| `log prefix "…"` | Apunta el paquete en el registro con esa etiqueta |
| `accept` / `drop` | Dejar pasar / tirar sin avisar |
| `masquerade` | El NAT: cambia la IP privada del cliente por la de la WAN al salir |

**Por qué la regla de DNS no tapa el DNS de la LAN:** cuando un cliente pregunta al **router** (.1), el paquete va por `input`, no por `forward`. La regla solo bloquea a quien intenta usar **otro** DNS de Internet (como 8.8.8.8) saltándose el del router.

Para cargar el fichero: `sudo systemctl enable --now nftables` (cargarlo ahora y en cada arranque) y `sudo nft list ruleset` para ver lo que hay cargado.

**Pruebas desde `cli-linux`:**

| Prueba | Resultado | Evidencia |
|---|---|---|
| `nslookup example.com` (DNS del router) | Responde | [captura](<../evidencias/t5-nftables-16-prueba-dns-20261119.png>) |
| `nslookup example.com 8.8.8.8` (DNS de fuera) | *timed out*: bloqueado | [captura](<../evidencias/t5-nftables-16-prueba-dns-20261119.png>) |
| Firefox → Google «hora» | Carga (11:11:04) | [captura](<../evidencias/t5-nftables-17-cliente-web-ok-20261119.png>) |

**Registro** (`sudo journalctl -k | grep LAN-`; `-k` = mensajes del **núcleo**, que es donde escribe nftables): → [web](<../evidencias/t5-nftables-18-log-web-20261119.png>) · [DNS bloqueado](<../evidencias/t5-nftables-19-log-dns-bloqueado-20261119.png>)

| Trozo de la línea | Qué significa |
|---|---|
| `LAN-DNS-BLOQ` / `LAN-WEB-OK` | La etiqueta de la regla que lo apuntó |
| `IN=enp0s8 OUT=enp0s3` | Entró por la LAN e iba a salir por la WAN |
| `SRC=192.168.27.136` | Quién: `cli-linux` |
| `DST=8.8.8.8` | A dónde iba |
| `PROTO=UDP DPT=53` | Protocolo y puerto: DNS |

Salen **3 líneas** de `LAN-DNS-BLOQ` separadas 5 segundos (11:10:22, :27 y :32): `nslookup` **reintenta** al no recibir respuesta.

### Copia de la configuración

**Qué copio:** a diferencia de los otros routers, aquí **no hay un botón de copia**: la configuración son **4 ficheros de texto** que he escrito yo. Los meto en un paquete `.tar.gz` (un fichero que junta varios y los comprime, como un `.zip`):

```
cd ~
sudo tar czf copia-r-nftables-$(date +%Y%m%d-%H%M).tar.gz /etc/nftables.conf /etc/dnsmasq.d/lan.conf /etc/network/interfaces /etc/sysctl.d/99-router.conf
tar tzf copia-r-nftables-*.tar.gz
```

| Parte | Qué significa |
|---|---|
| `c` / `t` / `x` | **C**rear el paquete / lis**t**ar lo que hay dentro / e**x**traer |
| `z` | Comprimir (gzip) |
| `f` | El nombre del paquete va justo detrás |
| `$(date +%Y%m%d-%H%M)` | Pone la fecha y hora en el nombre: `copia-r-nftables-20261119-1121.tar.gz` |
| `sudo` | Hace falta para leer algunos ficheros de `/etc` |

Resultado: **1048 bytes**. OPNsense ocupaba 59 KB porque guarda todo en un XML enorme; aquí solo va lo que he cambiado yo.

**En el NAS:** dataset **`copias/nftables`** y usuario **`nftables`** (solo SSH), como para los otros routers. → [usuario](<../evidencias/t5-truenas-16-usuario-nftables-20261119.png>) · [dataset](<../evidencias/t5-truenas-17-dataset-nftables-20261119.png>)

> **⬆️ Subir la copia al NAS** (desde `r-nftables`)
>
> ```
> sftp nftables@192.168.27.10
> put copia-r-nftables-20261119-1121.tar.gz
> ls -l
> bye
> ```
>
> Los comandos `put`, `ls` y `bye` se escriben **dentro** de sftp (cuando sale `sftp>`), no en la terminal normal.

Resultado: en el NAS `/mnt/copias/nftables/copia-r-nftables-20261119-1121.tar.gz`, 1048 bytes, **el mismo tamaño** que en el router. La huella del NAS (`SHA256:js+p3aiajK0n…`) es la misma de siempre; la preguntó porque era la primera vez que **este** equipo se conectaba. → [captura](<../evidencias/t5-nftables-20-copia-subida-nas-20261119.png>)

**Igual que el MikroTik, la copia la sube el propio router**, sin pasar por otro equipo. No va cifrada, pero tampoco lleva contraseñas: solo IP y reglas.

### Fallo y recuperación

**Qué simulo:** el mismo desastre doble que en los otros routers.
1. Alguien **machaca el fichero de reglas** y las aplica: desaparecen el NAT y el cortafuegos. La LAN se queda **sin Internet**.
2. Además se **borra la copia** que había en el router.

Así la única forma de arreglarlo es **la copia del NAS**. Apunto la hora con `date` en cada paso para medir cuánto se tarda.

**Equipos que intervienen:**

| Equipo | Papel en la prueba |
|---|---|
| `r-nftables` (192.168.27.1) | El router que se rompe y se recupera |
| `cli-linux` (192.168.27.136) | Cliente de la LAN: sirve para ver si hay Internet o no |
| TrueNAS (192.168.27.10) | Donde está guardada la copia |

#### 1. Antes del fallo (11:25:17)

En `cli-linux`:

```
date; wget -S --spider https://www.wikipedia.org 2>&1 | grep "HTTP/"
HTTP/1.1 200 OK
```

| Parte | Qué hace |
|---|---|
| `date;` | Escribe la hora y luego (`;`) sigue con la siguiente orden |
| `wget --spider` | Pregunta a la web si existe **sin descargar** la página |
| `-S` | Enseña la respuesta del servidor |
| `2>&1 \| grep "HTTP/"` | wget escribe esa respuesta por la salida de errores; `2>&1` la junta con la normal para que `grep` pueda filtrar solo la línea del código |
| `200 OK` | La web ha respondido: **hay Internet** |

→ [captura](<../evidencias/t5-nftables-21-antes-del-fallo-web-ok-20261119.png>)

#### 2. Provoco el fallo

En el router (por SSH):

```
echo 'flush ruleset' | sudo tee /etc/nftables.conf
sudo systemctl restart nftables
sudo nft list ruleset
rm ~/copia-r-nftables-*.tar.gz
ls ~
```

| Orden | Qué hace |
|---|---|
| `echo 'flush ruleset' \| sudo tee /etc/nftables.conf` | **Sobrescribe** el fichero de reglas: ahora solo contiene `flush ruleset` («borra todo»). `tee` escribe en el fichero lo que le llega y además lo enseña en pantalla |
| `sudo systemctl restart nftables` | Vuelve a cargar el fichero, es decir, **borra todas las reglas** |
| `sudo nft list ruleset` | Enseña las reglas cargadas: **no sale nada** |
| `rm ~/copia-r-nftables-*.tar.gz` | Borra la copia local. Preguntó `¿borrar … protegido contra escritura? (s/n)`: el fichero lo creó `root` (con `sudo tar`), y `rm` avisa antes de borrar un fichero que no es tuyo. Respondí `s` |
| `ls ~` | Comprueba que la copia ya no está |

→ [captura](<../evidencias/t5-nftables-22-romper-reglas-y-borrar-copia-20261119.png>)

**Qué se ha roto exactamente:**
- **Sin `masquerade`:** los paquetes de los clientes salen a Internet con su IP privada (192.168.27.x). Nadie en Internet sabe devolver la respuesta a esa IP: la conexión nunca recibe contestación.
- **Sin el cortafuegos:** tampoco quedan las reglas `LAN-WEB-OK` ni `LAN-DNS-BLOQ`, ni su registro.
- **Qué sigue funcionando:** el `ip_forward` y dnsmasq están en **otros ficheros** que no he tocado. Los clientes siguen recibiendo IP y DNS; solo falla la salida a Internet.

#### 3. Compruebo que está roto (11:28:44)

En `cli-linux`, la misma prueba, esperando como mucho 10 segundos y un solo intento:

```
date; wget -S --spider --timeout=10 --tries=1 https://www.wikipedia.org 2>&1 | grep "HTTP/"
```

**No sale ninguna línea `HTTP/`**: la web no ha contestado nada en 10 segundos. Antes del fallo salía `200 OK`. → [captura](<../evidencias/t5-nftables-23-cliente-sin-internet-20261119.png>)

| Opción | Qué hace |
|---|---|
| `--timeout=10` | Esperar la respuesta como mucho 10 segundos |
| `--tries=1` | Un solo intento (de serie wget lo intenta 20 veces y tardaría minutos) |

#### 4. Recupero desde el NAS (inicio: 11:29:16)

El router **sí llega al NAS aunque esté roto**: el NAS está en la **LAN**, así que la conexión no necesita NAT ni pasa por la cadena `forward`. Lo que se ha roto es la salida **de los clientes** a Internet.

En el router:

> **⬇️ Bajar la copia del NAS** (desde `r-nftables`)
>
> ```
> date
> sftp nftables@192.168.27.10
> get copia-r-nftables-20261119-1121.tar.gz
> bye
> ```
>
> `get` es lo contrario de `put`: **trae** el fichero del NAS al equipo donde estoy. Igual que al subir, `get` y `bye` se escriben **dentro** de sftp.

Resultado: `100% 1048`, el mismo tamaño que tenía la copia original: ha llegado entera.

> **🔁 Restaurar las reglas** (en `r-nftables`)
>
> ```
> sudo tar xzf copia-r-nftables-20261119-1121.tar.gz -C / etc/nftables.conf
> sudo systemctl restart nftables
> sudo nft list ruleset
> date
> ```

| Parte | Qué hace |
|---|---|
| `tar xzf copia….tar.gz` | E**x**trae del paquete comprimido |
| `-C /` | Lo extrae **a partir de la raíz** (`/`). Dentro del paquete el fichero se llama `etc/nftables.conf` (sin la `/` del principio), así que acaba en `/etc/nftables.conf`, **su sitio original**, sobrescribiendo el roto |
| `etc/nftables.conf` al final | Saca **solo ese fichero**. Los otros tres que hay en la copia no se han roto y no hace falta tocarlos |
| `systemctl restart nftables` | Carga el fichero recuperado |
| `nft list ruleset` | Comprueba que han vuelto la tabla `filtro` con las dos reglas y la tabla `nat` con el `masquerade` |

→ [captura de la bajada y la restauración](<../evidencias/t5-nftables-24-recuperacion-desde-nas-20261119.png>)

**Hora final: 11:30:06.**

#### 5. Compruebo la recuperación (11:30:07)

En `cli-linux`, la misma orden que en el paso 3:

```
date; wget -S --spider --timeout=10 --tries=1 https://www.wikipedia.org 2>&1 | grep "HTTP/"
HTTP/1.1 200 OK
```

Vuelve el **`200 OK`**: la LAN tiene Internet otra vez. → [captura](<../evidencias/t5-nftables-25-cliente-web-ok-otra-vez-20261119.png>)

#### Resumen de la prueba

| Hora | Dónde | Qué pasó |
|---|---|---|
| 11:25:17 | `cli-linux` | Hay Internet (`200 OK`) |
| ≈11:27 | `r-nftables` | Fichero de reglas machacado, reglas borradas y copia local borrada |
| 11:28:44 | `cli-linux` | Sin Internet (sin respuesta en 10 s) |
| 11:29:16 | `r-nftables` | Empiezo la recuperación: bajo la copia del NAS |
| 11:30:06 | `r-nftables` | Reglas restauradas y comprobadas |
| 11:30:07 | `cli-linux` | Hay Internet otra vez (`200 OK`) |

**Tiempo de recuperación: 50 segundos** (de 11:29:16 a 11:30:06). Es el más rápido de los tres routers: no hay que reiniciar el equipo, solo recargar las reglas, y el fichero es pequeño.

**Problema que tuve:** al hacer `ls ~` aparecieron dos ficheros raros, `bye` y `get`. En un paso anterior pegué en la **terminal normal** líneas que empezaban por `sftp>` (copiadas tal cual de las instrucciones). Para la terminal, `>` significa «**guarda la salida en un fichero**», así que `sftp> get …` creó un fichero llamado `get`, y `sftp> bye` uno llamado `bye`. Son ficheros vacíos sin importancia; se borran con `rm ~/bye ~/get`. **Lección:** `sftp>` es lo que escribe sftp en pantalla para pedir una orden; no hay que copiarlo.

**Resultado:** con la copia del NAS, el router ha quedado exactamente como antes, sin volver a escribir las reglas a mano.

**Resumen de los comandos de copia:**

| Para qué | Dónde | Comando |
|---|---|---|
| Crear la copia | `r-nftables` | `sudo tar czf copia-r-nftables-$(date +%Y%m%d-%H%M).tar.gz /etc/nftables.conf /etc/dnsmasq.d/lan.conf /etc/network/interfaces /etc/sysctl.d/99-router.conf` |
| Ver qué hay dentro | `r-nftables` | `tar tzf FICHERO` |
| ⬆️ **Subir** al NAS | `r-nftables` | `sftp nftables@192.168.27.10` y dentro `put FICHERO` |
| ⬇️ **Bajar** del NAS | `r-nftables` | `sftp nftables@192.168.27.10` y dentro `get FICHERO` |
| 🔁 Restaurar las reglas | `r-nftables` | `sudo tar xzf FICHERO -C / etc/nftables.conf` y `sudo systemctl restart nftables` |
| 🔁 Restaurar todo | `r-nftables` | `sudo tar xzf FICHERO -C /` y reiniciar el equipo |

## Comparación de las tres soluciones

| Criterio | MikroTik | OPNsense | Debian + nftables |
|---|---|---|---|
| Recursos que consume (VM) | 1 vCPU, 256 MB | 2 vCPU, 2 GB | 1 vCPU, 1 GB |
| Cómo se administra | Web (WebFig) y terminal | Web; consola solo para lo básico (tarjetas e IP) | Solo terminal (SSH desde el portátil); todo son ficheros de texto |
| Cómo se hace la copia | Backup cifrado + export; **el router la sube solo** al NAS por SFTP (`/tool fetch`) | XML cifrado; **se descarga con el navegador** y se sube al NAS desde otro equipo | `.tar.gz` con 4 ficheros de texto (1 KB); **el router la sube solo** por SFTP |
| Tiempo de recuperación medido | Unos minutos (no lo cronometré) | Unos 5 min 30 s, contando un error mío | **50 s** (sin reiniciar el router) |

## Comparación con el grupo

| Aspecto | Grupo 3 | Mi entorno |
|---|---|---|
| MikroTik | hEX físico (RB750Gr3), RouterOS 7.20.8 | CHR 7.24.6 en VM, administrado por WebFig |
| OPNsense | VM en VirtualBox del PC-01 | VM en mi VirtualBox, OPNsense 26.7.9 |
| Debian + nftables | VM en VirtualBox del PC-01 | VM en mi VirtualBox |
| WAN de los routers virtuales | Adaptador puente a la red del aula | NAT |
