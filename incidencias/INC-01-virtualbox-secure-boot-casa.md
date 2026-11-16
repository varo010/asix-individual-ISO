# INC-01 · VirtualBox no arranca VM en el portátil de casa

| Campo | Contenido |
|---|---|
| Fecha y equipo | 15/11/2026 · portátil de casa (Lenovo Legion Pro 5) con el disco USB de clase |
| Qué vi (síntoma) | `VBoxManage` avisa: *The vboxdrv kernel module is not loaded … You will not be able to start VMs*. En el portátil de clase, con el mismo disco, sí funciona |
| Qué pensé que era (hipótesis) | 1) Falta el módulo para este kernel, o 2) el módulo existe pero el firmware no deja cargarlo por Secure Boot |
| Cómo lo comprobé | `ls /lib/modules/$(uname -r)/misc/` → los módulos `vboxdrv.ko`, `vboxnetflt.ko`, `vboxnetadp.ko` **existen** (descarta la 1). `mokutil --sb-state` → *SecureBoot enabled* en casa y *disabled* en clase. `modinfo -F signer …/vboxdrv.ko` → firmado con la clave *localhost.localdomain Secure Boot Module Signature key*, la misma que está en `/var/lib/shim-signed/mok/MOK.der`. `mokutil --list-enrolled` → en el firmware de casa **solo está la clave de Canonical**, no la nuestra (confirma la 2) |
| Causa | Con Secure Boot, el kernel solo carga módulos firmados con una clave en la que confía el firmware. Nuestra clave (MOK) se creó al instalar VirtualBox, pero **la lista de claves se guarda en la placa base de cada equipo**, no en el disco. El portátil de casa no la conoce |
| Qué hacer (solución) | Registrar nuestra clave en el firmware de casa, sin desactivar Secure Boot (ver pasos) |
| Por qué no desactivar Secure Boot | También funcionaría, pero el NVMe de casa tiene Windows con **BitLocker**: al cambiar Secure Boot, Windows pedirá la clave de recuperación de BitLocker. Registrar la clave no afecta a Windows |
| Cómo comprobaré que quedó bien | `lsmod | grep vbox` muestra `vboxdrv`, `vboxnetflt` y `vboxnetadp`, y `VBoxManage --version` ya no da el aviso |
| Cómo volvería atrás | `sudo mokutil --delete /var/lib/shim-signed/mok/MOK.der`, reiniciar y confirmar el borrado en la pantalla azul |
| Qué aprendí | Un disco con Linux que se mueve entre equipos lleva consigo los módulos firmados, pero no la confianza en la clave: eso depende del firmware de cada máquina |

## Estado

- 15/11/2026: paso 1 hecho (`mokutil --list-new` muestra la clave con huella `bf:0b:f4:2f:…:bc:b0`, la misma que firma los módulos). **Pendiente: reiniciar y confirmar en la pantalla azul (pasos 2–4).**
- 16/11/2026, en el portátil de clase (Secure Boot desactivado): los módulos `vboxdrv`, `vboxnetflt` y `vboxnetadp` cargan y las VM `r-mikrotik`, `pve1` (con VT-x anidado) y `srv-dc` arrancan. Confirma que el problema es solo del firmware de casa. **En casa sigue pendiente la pantalla azul.**

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

   Si los módulos no aparecen solos, se cargan con `sudo modprobe vboxdrv vboxnetflt vboxnetadp`.

**Ojo con las actualizaciones del kernel:** al actualizar el kernel hay que recompilar los módulos (`sudo /sbin/vboxconfig`), que se vuelven a firmar con la misma clave de `/var/lib/shim-signed/mok/`. Mientras la clave no cambie, no hay que repetir el registro en ninguno de los dos portátiles.
