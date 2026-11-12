# T5 · Servidores de comunicaciones

RA: ISO RA1 (instalación, red, filtrado), ISO RA3 (copia y recuperación), ISO RA6 (registros). Las tres alternativas se prueban una detrás de otra en el mismo escenario.

## Comprobaciones generales (todas las VM de la T5)

- [ ] Requisitos y origen de las imágenes (arquitectura, fuente fiable); integridad verificada cuando el fabricante publique el hash.
- [ ] Actualizar y comprobar el funcionamiento antes de ponerlo «en producción».
- [ ] Identificar problemas posibles, configuraciones críticas y tiempo de respuesta ante una incidencia de instalación o arranque.

| Sistema | Versión | Origen (URL oficial) | Hash publicado | Hash comprobado | Fecha |
|---|---|---|---|---|---|
| MikroTik CHR | | | | | |
| OPNsense | | | | | |
| Debian | | | | | |

## Checklist por router

| Tarea de la rúbrica | MikroTik | OPNsense | Debian + nftables |
|---|---|---|---|
| Instalar y comprobar versión y funcionamiento | [ ] | [ ] | [ ] |
| Usuario(s) de administración | [ ] | [ ] | [ ] |
| Red WAN y LAN | [ ] | [ ] | [ ] |
| Acceso a Internet desde la LAN | [ ] | [ ] | [ ] |
| Conexión permitida y bloqueada + registro interpretado | [ ] | [ ] | [ ] |
| Copia de la configuración en el servidor de copias | [ ] | [ ] | [ ] |
| Fallo provocado y recuperación | [ ] | [ ] | [ ] |

## MikroTik (CHR)

### Instalación
### Usuario de administración
### Red WAN y LAN
### Salida a Internet
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
| MikroTik | hEX físico (RB750Gr3), RouterOS 7.20.8 | CHR en VM |
| OPNsense | VM en VirtualBox del PC-01 | VM en mi VirtualBox |
| Debian + nftables | VM en VirtualBox del PC-01 | VM en mi VirtualBox |
| WAN de los routers virtuales | Adaptador puente a la red del aula | NAT |
