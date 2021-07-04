---
title: "OpenAdmin - HTB Writeup"
layout: single
excerpt:
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/124387478-6624e600-dcce-11eb-9877-5701b0edc04d.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - ShellShock
  - Perl
  - sudo
---

![titulo](https://user-images.githubusercontent.com/67548295/124387610-cae04080-dcce-11eb-932a-6940d389c173.jpeg)


# Enumeración

Comenzamos enviando una traza ICMP a la máquina para verificar que está activa.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.171
PING 10.10.10.171 (10.10.10.171) 56(84) bytes of data.
64 bytes from 10.10.10.171: icmp_seq=1 ttl=63 time=59.7 ms

--- 10.10.10.171 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 59.701/59.701/59.701/0.000 ms
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $
```
La máquina nos responde, y mediante el TTL veo que es una máquina Linux.

Más información sobre la deteccion de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

# Nmap

Hacemos uso de la herramienta ```nmap``` para escanear los puertos y servicios de la máquina:

```bash
nmap -p- --open -T5 -n 10.10.10.171 -sC -sV 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-04 15:14 WEST
Nmap scan report for 10.10.10.171
Host is up (0.059s latency).
Not shown: 57738 closed ports, 7795 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.52 seconds
```
Solo hay un servicio SSH y HTTP, enumeremos el servicio HTTP

# User

Antes de visitar el servidor web, vamos a fuzzearlo con gobuster:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/OpenAdmin/nmap]
└──╼ $gobuster dir -u http://10.10.10.171/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.171/
[+] Threads:        50
[+] Wordlist:       /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/07/04 19:22:52 Starting gobuster
===============================================================
/music (Status: 301)
/artwork (Status: 301)
/sierra (Status: 301)
/server-status (Status: 403)
===============================================================
2021/07/04 19:27:48 Finished
===============================================================
```

Como se puede ver, tenemos 3 rutas que podemos probar.

Investiguemos la primera:

![navegador1](https://user-images.githubusercontent.com/67548295/124395986-a730f080-dcf6-11eb-879f-0bd5de884c5a.png)

Vemos como si fuera una pagina web de música.

Se puede observar que hay una opcion de login asi que voy ahí para ver si puedo iniciar sesion con credenciales comunes,

Y me encuentro con una sorpresa:

![navegador2](https://user-images.githubusercontent.com/67548295/124396088-21617500-dcf7-11eb-93c0-35a07dd74bd1.png)

El botón de login nos redirige a un servicio con nombre _OpenNetAdmin_




