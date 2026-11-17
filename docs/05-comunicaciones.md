# T5 · Servidores de comunicaciones

RA: ISO RA1 (instalación, red, filtrado), ISO RA3 (copia y recuperación), ISO RA6 (registros). Las tres alternativas se prueban una detrás de otra en el mismo escenario.

## Comprobaciones generales (todas las VM de la T5)

- [ ] Requisitos y origen de las imágenes (arquitectura, fuente fiable); integridad verificada cuando el fabricante publique el hash.
- [ ] Actualizar y comprobar el funcionamiento antes de ponerlo «en producción».
- [ ] Identificar problemas posibles, configuraciones críticas y tiempo de respuesta ante una incidencia de instalación o arranque.

| Sistema | Versión | Origen (URL oficial) | Hash publicado | Hash comprobado | Fecha |
|---|---|---|---|---|---|
| MikroTik CHR | 7.24.6 (long-term), x86-64 | <https://mikrotik.com/download> (imagen de disco para VirtualBox `chr-7.24.6.vdi.zip`) | Sí (fichero `.sha256`) | `746a4975…5faae` ✅ coincide | 15/11/2026 |
| OPNsense | 26.7, amd64 | <https://pkg.opnsense.org/releases/26.7/> (`OPNsense-26.7-dvd-amd64.iso.bz2`) | Sí, del `.bz2` | `95cafedd…186bfb` ✅ coincide | 15/11/2026 |
| Debian | 13.7.0, amd64, netinst | <https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/> | Sí (`SHA256SUMS`) | `a7ef94ac…ce6167e355` ✅ coincide | 15/11/2026 |

Comprobación completa: [t5-isos-sha256-20261116.txt](../evidencias/t5-isos-sha256-20261116.txt). Cómo se hace: `sha256sum fichero` y comparar el resultado con el que publica el fabricante en su web.

## Checklist por router

| Tarea de la rúbrica | MikroTik | OPNsense | Debian + nftables |
|---|---|---|---|
| Instalar y comprobar versión y funcionamiento | [x] | [ ] | [ ] |
| Usuario(s) de administración | [x] | [ ] | [ ] |
| Red WAN y LAN | [x] | [ ] | [ ] |
| Acceso a Internet desde la LAN | [x] | [ ] | [ ] |
| Conexión permitida y bloqueada + registro interpretado | [x] | [ ] | [ ] |
| Copia de la configuración en el servidor de copias | [x] | [ ] | [ ] |
| Fallo provocado y recuperación | [x] | [ ] | [ ] |

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

Después compruebo con `/file print` que el `.backup` vuelve a medir 21,6 KiB, y lo cargo:

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

### Instalación
### Usuario de administración
### Red WAN y LAN
### Salida a Internet
### Conexión permitida y bloqueada
### Copia de la configuración
### Fallo y recuperación

## GNU/Linux (Debian) con nftables

### Instalación
### Usuario de administración
### Red WAN y LAN
### Salida a Internet
### Conexión permitida y bloqueada
### Copia de la configuración
### Fallo y recuperación

## Comparación de las tres soluciones

| Criterio | MikroTik | OPNsense | Debian + nftables |
|---|---|---|---|
| Recursos que consume (medidos) | | | |
| Cómo se administra | | | |
| Cómo se hace la copia | | | |
| Tiempo de recuperación medido | | | |

## Comparación con el grupo

| Aspecto | Grupo 3 | Mi entorno |
|---|---|---|
| MikroTik | hEX físico (RB750Gr3), RouterOS 7.20.8 | CHR 7.24.6 en VM, administrado por WebFig |
| OPNsense | VM en VirtualBox del PC-01 | VM en mi VirtualBox |
| Debian + nftables | VM en VirtualBox del PC-01 | VM en mi VirtualBox |
| WAN de los routers virtuales | Adaptador puente a la red del aula | NAT |
