# T5 · Servidores: Windows Server 2025 y GNU/Linux de infraestructura

RA: ISO RA1 (instalación y red), ISO RA3 (copias).

## Checklist

| Tarea de la rúbrica | Windows Server 2025 | Servidor GNU/Linux |
|---|---|---|
| Instalar y comprobar versión y funcionamiento | [ ] | [ ] |
| Usuario(s) de administración | [ ] | [ ] |
| Parámetros de red | [ ] | [ ] |
| Copiar configuraciones/imágenes al servidor de copias y justificar qué sí y qué no | [ ] | [ ] |
| Explicar qué es un dominio y qué aporta la gestión centralizada | [ ] | — |
| Software para su función (de momento ninguno adicional) | — | [ ] |

## Windows Server 2025

> **Va en una VM aparte, fuera del clúster Proxmox** (indicación del profesorado). `srv-dc` es una VM de VirtualBox con sus propios recursos (2 vCPU, 4 GB, 50 GB) y no depende de `pve1` ni `pve2`.
> **Por qué:** el clúster corre con virtualización anidada (una VM dentro de otra), que es más lenta y más delicada. Si un nodo falla o va justo de RAM, el dominio caería con él, y todos los equipos que dependen del dominio se quedarían sin poder iniciar sesión. Separado, el servidor más importante no depende de la parte más frágil del entorno.

### Instalación (edición, licencia de evaluación, requisitos)
### Usuario de administración
### Red
### Copias: qué sí y qué no
### Qué es un dominio y qué aporta a la empresa

## Servidor GNU/Linux de infraestructura

### Instalación
### Usuario de administración
### Red
### Copias: qué sí y qué no
