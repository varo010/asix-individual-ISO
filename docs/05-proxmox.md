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

En el clúster solo van las pruebas de Proxmox (un contenedor y una VM Debian sin escritorio). **Windows Server no va dentro del clúster**: es una VM aparte, `srv-dc` ([05-servidores](05-servidores.md)).

| Nodo | vCPU | RAM | Disco | IP | Virtualización anidada |
|---|---|---|---|---|---|
| `pve1` | 2 | 4 GB | 32 GB | 192.168.27.11 | Sí |
| `pve2` | 2 | 4 GB | 32 GB | 192.168.27.12 | Sí |
| `pve3` | 2 | 4 GB | 32 GB | 192.168.27.15 | Sí |

**Por qué 3 nodos y no 2:** por el quórum (explicado en el punto 4). Al principio el diseño tenía 2 para ahorrar RAM; el 18/11 añadí `pve3` ([cambio de diseño](00-entorno.md#cambio-de-diseño-de-2-a-3-nodos-de-proxmox-18112026)).

## 2. Instalación
## 3. Usuarios y red
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

> Pendiente: montar el clúster y comprobar el quórum apagando un nodo.

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
| Dónde corren los nodos | 3 VM en el servidor Proxmox del centro (Proxmox anidado) | 3 VM en mi VirtualBox (Proxmox anidado) |
| Nodos | 3 (`pve`, `pve2`, `pve3`), ~4 GiB cada uno | 3 (`pve1`, `pve2`, `pve3`), 4 GB cada uno |
