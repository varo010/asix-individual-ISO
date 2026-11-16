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

Comprobación completa: [t5-isos-sha256-20261115.txt](../evidencias/t5-isos-sha256-20261115.txt). Cómo se hace: `sha256sum fichero` y comparar el resultado con el que publica el fabricante en su web.

## Checklist por router

| Tarea de la rúbrica | MikroTik | OPNsense | Debian + nftables |
|---|---|---|---|
| Instalar y comprobar versión y funcionamiento | [x] | [ ] | [ ] |
| Usuario(s) de administración | [x] | [ ] | [ ] |
| Red WAN y LAN | [x] | [ ] | [ ] |
| Acceso a Internet desde la LAN | [ ] el router sale; falta probarlo desde un cliente | [ ] | [ ] |
| Conexión permitida y bloqueada + registro interpretado | [ ] | [ ] | [ ] |
| Copia de la configuración en el servidor de copias | [ ] | [ ] | [ ] |
| Fallo provocado y recuperación | [ ] | [ ] | [ ] |

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

No uso `ping` a Internet porque la red del aula lo bloquea (se vio en la [T2](02-equipos.md)); la descarga de una web sí pasa. Falta comprobar que **un cliente de la LAN** recibe IP por DHCP y sale a Internet.

### Conexión permitida y bloqueada
### Copia de la configuración
### Fallo y recuperación

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
