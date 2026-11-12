# T4 · Diseño de la red

RA: ISO RA1. Esquema de red de mi entorno, montaje en VirtualBox y direcciones.

## Lo que pide la rúbrica

- [ ] Dibujar el esquema de red individual, el montaje virtual y las direcciones previstas.
- [ ] Indicar la configuración de cada interfaz y la LAN 192.168.XX.0/24 según mi puesto en el aula; la WAN puede ir por DHCP (adaptador puente o, mejor, NAT).
- [ ] Ampliación: implementarlo y comprobar la conectividad con `ping`, `traceroute` y la tabla de rutas.
- [ ] Ampliación: evidencias de `ping`, `traceroute`, rutas y configuración de interfaces entre VM y hacia el exterior.

## 1. Esquema

> Exportación de [diagramas/](../diagramas/) cuando esté hecho.

## 2. Redes de VirtualBox

| Red de VirtualBox | Tipo | Para qué | Quién se conecta |
|---|---|---|---|
| N/C | NAT | WAN del router: salida al exterior | Solo el router activo |
| N/C | Red interna | LAN 192.168.XX.0/24 | Router (LAN) y todas las demás VM |

> Explica con tus palabras por qué la WAN va en NAT y la LAN en red interna, y qué pasaría con cada otro tipo (puente, solo-anfitrión).

## 3. Plan de direcciones

LAN: 192.168.XX.0/24 · máscara 255.255.255.0.

| Rango | Uso |
|---|---|
| N/C | Routers |
| N/C | Servidores (IP fija) |
| N/C | Clientes (DHCP) |

## 4. Configuración de cada interfaz

| VM | Interfaz | Red de VirtualBox | IP / máscara | Puerta de enlace | DNS | Método |
|---|---|---|---|---|---|---|

## 5. Ampliación: verificación de la conectividad

| Prueba | Desde | Hacia | Resultado | Evidencia |
|---|---|---|---|---|

## 6. Comparación con la red del grupo

| Aspecto | Grupo 3 | Mi entorno |
|---|---|---|
| LAN | 192.168.30.0/24 | 192.168.XX.0/24 |
| WAN | 172.16.222.0/24, IP asignada por el profesor (MikroTik .216) | NAT de VirtualBox por DHCP |
| Puerta de enlace de la LAN | MikroTik, 192.168.30.1 | N/C |
| Switch | Físicos no gestionables (TP-Link) | Red interna de VirtualBox (switch virtual) |
| Separación de redes | El router separa la LAN del grupo de la red del aula | N/C |
