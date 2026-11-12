# T2 · Mis equipos: clase y casa

RA: FDM RA1. Inventario de los dos equipos, comparativa y cuellos de botella.

## Lo que pide la rúbrica

- [ ] Inventariar el equipo de clase y el de casa.
- [ ] Compararlos e identificar cuellos de botella.
- [ ] Proponer soluciones ajustadas al hardware disponible.
- [ ] Ampliación: tests de memoria, disco y red en los dos equipos.
- [ ] Ampliación: registrar herramienta, condiciones y resultado.
- [ ] Ampliación: confirmar o descartar con datos los cuellos de botella previstos.

## 1. Inventario y comparativa

N/C = no comprobado todavía.

| Dato | Equipo de clase | Equipo de casa | Cómo se comprueba |
|---|---|---|---|
| Modelo | Portátil ASUS ExpertBook B3605CCA | N/C | `/sys/class/dmi/id/` o `sudo dmidecode -t system` |
| Firmware | UEFI, BIOS B3605CCA.303 (06/11/2025), Secure Boot desactivado | N/C | `mokutil --sb-state`, `/sys/firmware/efi` |
| CPU | Intel Core Ultra 5 125H: 14 núcleos, 18 hilos, hasta 4,5 GHz, 18 MiB L3 | N/C | `lscpu` |
| Virtualización | VT-x, `/dev/kvm` presente | N/C | `lscpu`, `ls /dev/kvm` |
| RAM | 30 GiB visibles | N/C | `free -h`; módulos con `sudo dmidecode -t memory` |
| Ampliación de RAM | N/C | N/C | `sudo dmidecode -t memory` |
| Disco del sistema | SSD SATA Kingston SA400 480 GB **conectado por USB** (UAS, 5 Gb/s) | N/C | `lsblk -o NAME,TRAN,MODEL`, `lsusb -t` |
| Otros discos | NVMe Samsung 1 TB interno (otros sistemas) | N/C | `lsblk` |
| Espacio libre para VM | ~227 GB en `/home` | N/C | `df -h` |
| Red cableada | Intel Ethernet, 1000 Mb/s | N/C | `ip -br a`, `/sys/class/net/*/speed` |
| Wi-Fi | Realtek RTL8852BE (Wi-Fi 6) | N/C | `lspci` |
| Gráficos | Intel Arc integrada (Meteor Lake) | N/C | `lspci` |
| Sistema operativo | Linux Mint 22.3, kernel 6.8 | N/C | `/etc/os-release`, `uname -r` |
| Hipervisor | VirtualBox 7.2.18 + Extension Pack | N/C | `VBoxManage --version` |
| Alimentación | Batería + cargador del portátil | N/C | Inspección |

Evidencias: [t2-inventario-clase-20261112.txt](../evidencias/t2-inventario-clase-20261112.txt).

## 2. Cuellos de botella previstos

> Escribe aquí, para cada equipo, qué pieza crees que limitará cuando arranques varias VM y por qué. Son predicciones: se confirman o descartan en el apartado 3.

| Equipo | Recurso | Por qué podría limitar | Cómo lo compruebo | Solución posible |
|---|---|---|---|---|

## 3. Ampliación: tests de diagnóstico

| Equipo | Recurso | Herramienta | Condiciones | Resultado | Interpretación | Evidencia |
|---|---|---|---|---|---|---|

## 4. Conclusión

> ¿Se confirman los cuellos de botella? ¿En qué equipo conviene montar el entorno y por qué?

## 5. Comparación con el inventario del grupo

| Aspecto | Equipos del grupo (aula) | Mis equipos |
|---|---|---|
| Cuello de botella que encontró el grupo | Discos mecánicos (HDD) con accesos aleatorios; red Gigabit con ficheros grandes | N/C |
| Herramientas usadas | `lscpu`, `free`, `smartctl`, `fio`, `openssl speed`, `ethtool`, `iperf3`, `top` | N/C |
