---
title: "Blocky - HTB Writeup"
layout: single
excerpt: "Knife es una máquina de dificultad fácil de HackTheBox. En esta máquina explotamos un backdoor de una versión de PHP para acceder. Para escalar privilegios abusamos de un binario que podemos correr como root"
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/132039212-af13a643-9ae0-4980-b117-8795f1bd9415.png"
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
 
![titulo](https://user-images.githubusercontent.com/67548295/132039885-e9de5e03-fe17-48ef-8fdf-9c4e9c975a8e.png)

# Enumeración

Empezamos enviando una traza [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet) con la herramienta `ping`, con esto veremos el estado de la máquina y su sistema operativo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.37
PING 10.10.10.37 (10.10.10.37) 56(84) bytes of data.
64 bytes from 10.10.10.37: icmp_seq=1 ttl=63 time=74.1 ms

--- 10.10.10.37 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 74.113/74.113/74.113/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está activa y que observando el TTL, concluimos que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

## Nmap

Comenzamos la fase de enumeración de puertos, en mi caso voy a hacer uso de la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -v -n 10.10.10.37 -sC -sV -oN targeted
Starting Nmap 7.91 ( https://nmap.org ) at 2021-09-03 17:47 WEST
NSE: Loaded 153 scripts for scanning.
Completed Ping Scan at 17:47, 0.12s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 17:47
Scanning 10.10.10.37 [65535 ports]
Discovered open port 80/tcp on 10.10.10.37
Discovered open port 21/tcp on 10.10.10.37
Discovered open port 22/tcp on 10.10.10.37
Discovered open port 25565/tcp on 10.10.10.37
Completed SYN Stealth Scan at 17:47, 32.92s elapsed (65535 total ports)
Nmap scan report for 10.10.10.37
Host is up (0.074s latency).
Not shown: 65530 filtered ports, 1 closed port
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE   VERSION
21/tcp    open  ftp       ProFTPD 1.3.5a
22/tcp    open  ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp    open  http      Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.8
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: BlockyCraft &#8211; Under Construction!
25565/tcp open  minecraft Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 45.18 seconds
           Raw packets sent: 131093 (5.768MB) | Rcvd: 30 (1.300KB)
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

Vemos 3 puertos

| Puerto | Servicio |
|:------:|:--------:|
| 21 | [FTP](https://es.digitaltrends.com/computadoras/que-es-ftp-y-para-que-sirve/)
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh) |
| 80 | [HTTP](https://developer.mozilla.org/es/docs/Web/HTTP){:target="\_blank"} |
| 25565 | Servidor dedicado de Minecraft |

# User.txt


### FTP
Empiezo probando si el servicio FTP acepta _Anonymous login_:

```bash
└──╼ $ftp 10.10.10.37
Connected to 10.10.10.37.
220 ProFTPD 1.3.5a Server (Debian) [::ffff:10.10.10.37]
Name (10.10.10.37:z3r0byte): anonymous
331 Password required for anonymous
Password:
530 Login incorrect.
Login failed.
```

### HTTP

Pero como vemos, esta deshabilitado.

Lo siguiente que hago es empezar a enumerar el servidor web que se aloja en el puerto 80.

Hago uso de la herramienta `whatweb` para listar las tecnologías que emplea el servidor web:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $whatweb 10.10.10.37
http://10.10.10.37 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.37], JQuery[1.12.4], MetaGenerator[WordPress 4.8], PoweredBy[WordPress,WordPress,], Script[text/javascript], Title[BlockyCraft &#8211; Under Construction!], UncommonHeaders[link], WordPress[4.8]
```

Nos reporta que hay un wordpress 4.8 y varias tecnologías más.

Navego hasta la web para ver que hay:

![navegador1](https://user-images.githubusercontent.com/67548295/132043442-53c66ab0-5f0a-4053-8332-232941a4bcf7.png)

Se ve que es un blog relacionado con Minecraft.

Lo primero que me llama la atención es el post que hay públicado

![navegador2](https://user-images.githubusercontent.com/67548295/132043559-8d8c4510-97ee-4dc0-bf43-d75129f99a3c.png)

El administrador dice estar desarrollando una wiki y un plugin.

Esto me llama la atención asi que hago uso de la herramienta `gobuster` para buscar directorios en el servidor web:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $gobuster dir -u "http://10.10.10.37/" -w "/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt" -t 50
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.37/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/09/03 18:29:30 Starting gobuster in directory enumeration mode
===============================================================
/wiki                 (Status: 301) [Size: 309] [--> http://10.10.10.37/wiki/]
/wp-content           (Status: 301) [Size: 315] [--> http://10.10.10.37/wp-content/]
/plugins              (Status: 301) [Size: 312] [--> http://10.10.10.37/plugins/]   
/wp-includes          (Status: 301) [Size: 316] [--> http://10.10.10.37/wp-includes/]
```

Vaya, hemos encontrado un directorio wiki y otro plugins (los demas son internos de wordpress)

Accedo al recurso `wiki` para ver que contiene:

![navegador3](https://user-images.githubusercontent.com/67548295/132045025-74429b63-5c07-4264-b0f9-4d4a7758c7e9.png)



