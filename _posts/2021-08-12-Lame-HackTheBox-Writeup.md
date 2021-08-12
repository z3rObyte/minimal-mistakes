---
title: "Lame - HTB Writeup"
layout: single
excerpt: "Lame es una máquina de nivel fácil de HackTheBox. En esta máquina explotamos únicamente una antigua versión vulnerable de samba para acceder directamente como usuario root."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/129220317-5439270a-e4bb-47e0-b981-ef0a3aab911e.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Samba
  - RCE
---

![titulo](https://user-images.githubusercontent.com/67548295/129220778-695a7fa5-39f9-4bbe-b0bd-87bec3ec2d3c.png)

# Enumeración

Comenzamos enviando una traza ICMP a la máquina, con esto lograremos identificar el OS y saber si la máquina esta activa.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.3
PING 10.10.10.3 (10.10.10.3) 56(84) bytes of data.
64 bytes from 10.10.10.3: icmp_seq=1 ttl=63 time=203 ms

--- 10.10.10.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 203.041/203.041/203.041/0.000 ms
```
| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Podemos observar que la máquina está activa y que observando el TTL, concluimos que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

## Nmap

Comenzamos con la fase de escaneo de puertos:

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -T5 -v -n 10.10.10.3 -sV -sC -oN targeted
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-12 16:20 WEST
Initiating Ping Scan at 16:20
Scanning 10.10.10.3 [4 ports]
Completed Ping Scan at 16:20, 0.13s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 16:20
Scanning 10.10.10.3 [65535 ports]
Discovered open port 22/tcp on 10.10.10.3
Discovered open port 21/tcp on 10.10.10.3
Discovered open port 139/tcp on 10.10.10.3
Discovered open port 445/tcp on 10.10.10.3
Stats: 0:00:24 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
Discovered open port 3632/tcp on 10.10.10.3
Completed SYN Stealth Scan at 16:24, 234.90s elapsed (65535 total ports)
Initiating Service scan at 16:24
Scanning 5 services on 10.10.10.3
Completed Service scan at 16:24, 11.29s elapsed (5 services on 1 host)
NSE: Script scanning 10.10.10.3.
Initiating NSE at 16:24
NSE: [ftp-bounce] PORT response: 500 Illegal PORT command.
Nmap scan report for 10.10.10.3
Host is up (0.10s latency).
Not shown: 65530 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.12
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 2h00m22s, deviation: 2h49m43s, median: 21s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2021-08-12T11:24:39-04:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)

NSE: Script Post-scanning.
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 288.06 seconds
           Raw packets sent: 131238 (5.774MB) | Rcvd: 181 (7.948KB)
```
| Parámetro | Acción |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535 |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', que es más rápido y sigiloso que el tipo de escaneo por defecto |
| `--min-rate [valor]` | envia paquetes tan o más rápido que la tasa dada |
| `-v` | Especifica que queremos más 'verbose', es decir, que nos muestre mas información de la convencional |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido |
| `-sC` | Utiliza un escaneo con una serie de scripts por defecto de nmap |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo |

Podemos ver varios puertos:

| Puerto | servicio |
|:------:|:--------:|
| 21 | FTP: un protocolo pernsado para manejar archivos. |
| 22 | SSH: protocolo que sirve para conectarse remotamente a un equipo. |
| 139 | NetBIOS session service |
| 445| samba: la versión de Linux de SMB |
| 3632 | distccd: un compilador gratuito de C y C++  |

# User.txt y Root.txt

Pruebo un _Anonymous login_ en el FTP, pero no hay ningún recurso.

Sigo enumerando y veo que la versión de samba es vulnerable:

![terminal1](https://user-images.githubusercontent.com/67548295/129226546-ff20bb53-a20d-40a5-9a2e-64c376efd3e3.png)

Como se puede ver, hay un exploit para esta versión:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $searchsploit samba 3.0.20
-------------------------------------------------------------------------------------------------------------------------------------------------------
Exploit Title                                                                                                             |  Path
-------------------------------------------------------------------------------------------------------------------------------------------------------
Samba 3.0.10 < 3.3.5 - Format String / Security Bypass                                                                     | multiple/remote/10095.txt
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)                                           | unix/remote/16320.rb
Samba < 3.0.20 - Remote Heap Overflow                                                                                      | linux/remote/7701.txt
Samba < 3.0.20 - Remote Heap Overflow                                                                                      | linux/remote/7701.txt
Samba < 3.6.2 (x86) - Denial of Service (PoC)                                                                              | linux_x86/dos/36741.py
--------------------------------------------------------------------------------------------------------------------------------------------------------
```
Pero es con metasploit, y aqui no utilizamos eso.

Asi que lo que he hecho a sido buscar un exploit en github, y he encontrado [este](https://github.com/amriunix/CVE-2007-2447/blob/master/usermap_script.py)

Lo descargamos, nos ponemos en escucha con nc, y ejecutamos el exploit con los datos correspondientes:

![terminal2](https://user-images.githubusercontent.com/67548295/129231114-ce7f0bc2-3666-490d-92e2-efd54fa02212.png)

¡Y recibimos una shell como root!

Pues a partir de aquí ya podriamos las dos flags!

La flag de User está en /home/makis/user.txt:

```bash
root@lame:/# cat /home/makis/user.txt 
52fXXXXXXXXXXXXXXXXXXXXXXXXXXXX825
```
Y la de root está donde siempre, en /root/root.txt:

```bash
root@lame:/# cat /root/root.txt 
773XXXXXXXXXXXXXXXXXXXXXXXXXXd02
```
# Resumen

En esta máquina lo único que explotamos es una antigua versión vulnerable del servicio samba para acceder directamente como usuario root.


