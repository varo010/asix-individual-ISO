# INC-01 · VirtualBox no arranca VM en el portátil de casa

| Campo | Contenido |
|---|---|
| Fecha y equipo | 15/11/2026 · portátil de casa (Lenovo Legion Pro 5) con el disco USB de clase |
| Qué vi (síntoma) | `VBoxManage` avisa: *The vboxdrv kernel module is not loaded … You will not be able to start VMs* (el controlador de VirtualBox no está cargado y no se podrán arrancar VM). En el portátil de clase, con el mismo disco, sí funciona |
| Qué pensé que era (hipótesis) | 1) Faltan los controladores de VirtualBox en este sistema, o 2) están, pero Secure Boot no deja cargarlos |
| Cómo lo comprobé | Los controladores de VirtualBox **están** en el disco (`ls /lib/modules/$(uname -r)/misc/`), así que no es la 1. `mokutil --sb-state`: Secure Boot está activado en casa y desactivado en clase. `modinfo -F signer …/vboxdrv.ko`: los controladores están firmados con nuestra clave (la de `/var/lib/shim-signed/mok/MOK.der`). `mokutil --list-enrolled`: el portátil de casa **solo conoce la clave de Ubuntu (Canonical)**, no la nuestra. Es la 2 |
| Causa | Con Secure Boot, el sistema solo carga controladores firmados con una clave de confianza. Nuestra clave (se llama MOK, «clave del propietario de la máquina») se creó al instalar VirtualBox, pero **la lista de claves de confianza se guarda en la placa base de cada equipo**, no en el disco. El portátil de casa no la conoce |
| Qué hacer (solución) | Registrar nuestra clave en el portátil de casa, sin desactivar Secure Boot (ver pasos) |
| Por qué no desactivar Secure Boot | También funcionaría, pero el NVMe de casa tiene Windows con **BitLocker**: al cambiar Secure Boot, Windows pedirá la clave de recuperación de BitLocker. Registrar la clave no afecta a Windows |
| Cómo comprobaré que quedó bien | `lsmod | grep vbox` muestra `vboxdrv`, `vboxnetflt` y `vboxnetadp`, y `VBoxManage --version` ya no da el aviso |
| Cómo volvería atrás | `sudo mokutil --delete /var/lib/shim-signed/mok/MOK.der`, reiniciar y confirmar el borrado en la pantalla azul |
| Qué aprendí | Un disco con Linux que se mueve entre equipos lleva consigo los controladores firmados, pero no la confianza en la clave: eso lo guarda la placa base de cada equipo |

## Estado

- 15/11/2026: paso 1 hecho (`mokutil --list-new` muestra nuestra clave esperando a que se confirme). **Pendiente: reiniciar y confirmar en la pantalla azul (pasos 2–4).**
- 16/11/2026, en el portátil de clase (Secure Boot desactivado): los controladores de VirtualBox cargan y las VM `r-mikrotik`, `pve1` (con VT-x anidado) y `srv-dc` arrancan. Confirma que el problema es solo de la placa del portátil de casa. **En casa sigue pendiente la pantalla azul.**

## Pasos

1. Pedir el registro de la clave. Pide una contraseña de un solo uso: elige una sencilla, solo se usa en el paso 3.

   ```bash
   sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
   ```

2. Reiniciar el portátil **con el disco USB conectado**.
3. Sale una pantalla azul (*MOK management*). Con el teclado: **Enroll MOK** → **Continue** → **Yes** → escribir la contraseña del paso 1 → **Reboot**. Si no se hace nada en unos segundos, la pantalla desaparece y hay que repetir desde el paso 1.
4. Ya en Linux, comprobar:

   ```bash
   mokutil --list-enrolled | grep Subject   # ahora sale también localhost.localdomain
   lsmod | grep vbox                        # vboxdrv, vboxnetflt, vboxnetadp
   VBoxManage --version                     # sin el aviso
   ```

   Si los controladores no se cargan solos, se cargan con `sudo modprobe vboxdrv vboxnetflt vboxnetadp`.

**Ojo con las actualizaciones del núcleo (kernel) de Linux:** después hay que volver a generar los controladores de VirtualBox (`sudo /sbin/vboxconfig`), que se firman otra vez con la misma clave. Mientras la clave no cambie, no hay que repetir el registro en ninguno de los dos portátiles.
