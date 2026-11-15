# Preguntas para la defensa

Preguntas que me pueden hacer y las ideas clave de la respuesta. **Hay que saber decirlas con mis palabras**, no leerlas.

## T0 · Entorno

| Pregunta | Ideas clave |
|---|---|
| ¿Dónde está tu entorno y por qué ahí? | En el SSD USB de clase: el mismo entorno funciona en el portátil de clase y en el de casa sin copiar nada |
| ¿Puedes arrancar todas las VM a la vez? | No: suman unos 31 GB y los portátiles tienen 30–31 GiB, de los que el anfitrión necesita unos 4. Se trabaja por escenarios |
| ¿Qué recurso limita cuántas VM arrancas? | La RAM. La CPU va sobrada; el disco limita la velocidad, no el número |
| ¿Qué es la virtualización anidada y para qué la necesitas? | Un hipervisor dentro de una VM. Proxmox va dentro de VirtualBox; para que sus VM arranquen, la VM necesita las instrucciones VT-x |
| ¿Qué es un disco dinámico? | Un disco virtual que crece según se escribe hasta su tamaño máximo; ahorra espacio pero puede llenar el disco real si no se vigila |

## T2 · Equipos

| Pregunta | Ideas clave |
|---|---|
| ¿Qué diferencia hay entre tus dos equipos? | Mismo disco y sistema; cambian CPU (18 frente a 32 hilos), red (cable frente a Wi-Fi) y Secure Boot (desactivado frente a activado) |
| ¿Cuál es el cuello de botella? | El disco USB: todas las VM lo comparten y va por un puente USB de 5 Gb/s (~400 MB/s). Falta medir el acceso aleatorio con `fio` |
| ¿Por qué el test de CPU dio dos resultados distintos? | Con batería el portátil recorta potencia en cuanto la carga dura: 30 → 18 GB/s. Las pruebas se repiten con cargador |
| ¿Por qué no funcionaba VirtualBox en casa? | Secure Boot solo carga módulos firmados con claves que conoce el firmware. La clave de VirtualBox estaba registrada en el portátil de clase, no en el de casa: el registro está en la placa, no en el disco |
| ¿Por qué no desactivaste Secure Boot? | Porque en casa hay Windows con BitLocker, que pediría la clave de recuperación. Registrar la clave (MOK) no le afecta |
| ¿Qué es un error CRC en SMART? ¿Tu disco está mal? | Un bloque que llega dañado por el cable SATA y se reenvía. El SSD está sano (0 sectores reasignados); el fallo está en el enlace con la caja USB. Para saber si es actual: leer el contador, hacer una prueba larga de disco y ver si sube |
| ¿Qué diferencia hay entre `fio` secuencial y aleatorio? | Secuencial = ficheros grandes seguidos (MB/s). Aleatorio 4K = muchos trozos pequeños en sitios distintos (IOPS), que es lo que hacen varias VM a la vez |
| ¿Por qué el ping no mide la velocidad de la red? | Mide cuánto tarda un paquete (latencia), no cuántos datos pasan por segundo; para eso, una descarga o `iperf3` |

## T4 · Red

| Pregunta | Ideas clave |
|---|---|
| ¿Por qué la WAN del router va en NAT y no en puente? | Funciona igual en clase y en casa (el puente sobre Wi-Fi falla), y mi entorno no toca la red del aula (nada de DHCP de pruebas en la red compartida) |
| ¿Por qué tu LAN es la 192.168.27.0/24? | En el aula mi portátil coge IP por DHCP (172.16.222.27 el 12/11): uso ese 27 como número de puesto. No choca con las redes de los grupos (10–40) |
| ¿Qué es una red interna de VirtualBox? | Un switch virtual que solo une a las VM con el mismo nombre; ni el anfitrión ni la red real la ven |
| ¿Por qué tienes doble NAT? ¿Es un problema? | Mi router traduce la LAN a 10.0.2.15 y VirtualBox la traduce a la IP del portátil. Solo es un problema para entrar desde fuera, que no hace falta |
| ¿Por qué los tres routers tienen la misma IP? | Solo hay uno encendido; al cambiar de router los demás equipos siguen con la misma puerta de enlace |
| ¿Por qué los servidores llevan IP fija y los clientes DHCP? | A los servidores se les busca siempre en la misma dirección; los clientes no necesitan una IP concreta |
| ¿Para qué el modo promiscuo en Proxmox? | Las VM de dentro del nodo salen con su propia MAC; sin modo promiscuo VirtualBox tira esas tramas |
| Describe el camino de un paquete del cliente a Internet | Cliente → router (NAT 1) → NAT de VirtualBox (NAT 2) → red real → Internet, y la respuesta deshaciendo los dos NAT |

## T5 · Montaje de las VM e imágenes

| Pregunta | Idea clave |
|---|---|
| ¿Cómo sabes que la ISO no está corrupta ni modificada? | Calculo su SHA256 (`sha256sum`) y lo comparo con el que publica el fabricante en su web oficial. Si cambia un solo bit, el hash es totalmente distinto |
| ¿Y si el fabricante no publica el hash (Windows Server)? | Me fío del origen: servidor oficial de Microsoft por HTTPS (el certificado garantiza quién lo envía) y la descarga completa. Apunto su hash para demostrar que instalé esa misma |
| OPNsense solo da el hash del `.bz2`, ¿cómo compruebas la ISO? | Compruebo el `.bz2`; al descomprimir, `bunzip2` revisa el CRC de cada bloque, así que una ISO descomprimida sin error es la original |
| ¿Por qué no usaste la `w11.iso` que ya tenías? | No sé de dónde salió y es una versión de 2023: no puedo garantizar ni el origen ni que esté al día |
| ¿Por qué la VM de Proxmox tiene «VT-x anidado»? | Proxmox es un hipervisor: necesita las instrucciones de virtualización del procesador dentro de la VM para arrancar sus propias VM |
| ¿Por qué Windows 11 lleva TPM y Secure Boot? | Son requisitos de instalación de Windows 11; VirtualBox los emula (TPM 2.0 y claves de Microsoft en la UEFI) |
| ¿Por qué discos dinámicos? | Solo ocupan lo que se escribe: caben las 10 VM en el disco USB aunque la suma de tamaños máximos sea mayor |
