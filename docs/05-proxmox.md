# T5 · Proxmox VE: virtualización y clúster

RA: ISO RA1 (instalación, red, clúster, migración), ISO RA3 (copias), ISO RA6 (medidas de la migración).

## Lo que pide la rúbrica

- [ ] Instalar y comprobar versión y funcionamiento.
- [ ] Usuario(s) de administración.
- [ ] Parámetros de red de las interfaces.
- [ ] Clúster con los nodos disponibles (si no hay suficientes, diseño o simulación autorizada).
- [ ] Migrar un contenedor y una VM (Debian/Ubuntu Server **sin** entorno gráfico) entre nodos.
- [ ] Medir tiempo y espacio de cada migración, interpretarlo y valorar mejoras.
- [ ] Copiar configuraciones/imágenes al servidor de copias y justificar qué sí y qué no.
- [ ] Explicar ventajas e inconvenientes de virtualizar servidores.
- [ ] Explicar la diferencia entre contenedores y VM, y cuándo usar cada uno.

## 1. Nodos

| Nodo | vCPU | RAM | Disco | IP | Virtualización anidada |
|---|---|---|---|---|---|

## 2. Instalación
## 3. Usuarios y red
## 4. Clúster

> Explica: cuántos nodos, cómo funciona el quórum con ese número, y la diferencia entre gestión conjunta, migración y alta disponibilidad.

## 5. Migración medida

| Qué se migra | Tipo de migración | Tamaño en disco | Tiempo | Interpretación | Evidencia |
|---|---|---|---|---|---|
| Contenedor | | | | | |
| VM sin GUI | | | | | |

> ¿Se puede mejorar? ¿Cómo?

## 6. Copias del clúster
## 7. Preguntas de teoría

### Ventajas e inconvenientes de virtualizar servidores
### Contenedores frente a máquinas virtuales

## 8. Comparación con el grupo

| Aspecto | Grupo 3 | Mi entorno |
|---|---|---|
| Dónde corren los nodos | 3 VM en el servidor Proxmox del centro (Proxmox anidado) | N/C VM en VirtualBox (Proxmox anidado) |
| Nodos | 3 (`pve`, `pve2`, `pve3`), ~4 GiB cada uno | N/C |
