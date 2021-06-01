---
title: "VulnNet: Internal | TryHackMe WriteUp"
layout: single
excerpt:
show_date: true
classes: wide
header:
  teaser: "https://github.com/z3rObyte/assets/images/machines/THM/PickleRick/data/pickleRick.png?raw=true"
  teaser_home_page: true
  icon: "assets/images/icons/TryHackMe-icon.png"
categories:
  - Writeup
  - TryHackMe
tags:
  - Redis
  - Samba
---

# VulnNet: Internal Writeup
---

VulnNet: Internal es una máquina Linux de la plataforma de **TryHackMe** donde se deberán explotar diferentes servicios como _Samba, Redis, Rsync_, etc.

# Enumeración
---

Empezamos utilizando la utilidad ```ping``` para enviar una traza ICMP a la maquina para ver si está activa

```bash
$~  ping -c 1 10.10.104.158       
PING 10.10.104.158 (10.10.104.158) 56(84) bytes of data.
64 bytes from 10.10.104.158: icmp_seq=1 ttl=63 time=112 ms

--- 10.10.104.158 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 112.036/112.036/112.036/0.000 ms

```

Como se puede ver, la máquina nos responde.  

Mediante el TTL se puede saber el sistema operativo de la máquina, que en este caso es Linux. 

[Aquí](https://subinsb.com/default-device-ttl-values/) puedes ver como identificar el OS por el TTL. O también puedes utilizar mi herramienta [OSidentifier](https://github.com/z3rObyte/OSidentifier) 


### Nmap

Empecemos con el escaneo de puertos con ```nmap```:

```bash

$~  nmap -p- --open -n -v -T5 10.10.104.158 -sC -sV                                                                            


Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-01 16:35 WEST
Not shown: 34582 filtered ports, 30944 closed ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 5e:27:8f:48:ae:2f:f8:89:bb:89:13:e3:9a:fd:63:40 (RSA)
|   256 f4:fe:0b:e2:5c:88:b5:63:13:85:50:dd:d5:86:ab:bd (ECDSA)
|_  256 82:ea:48:85:f0:2a:23:7e:0e:a9:d9:14:0a:60:2f:ad (ED25519)
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      39085/tcp   mountd
|   100005  1,2,3      39189/tcp6  mountd
|   100005  1,2,3      56555/udp   mountd
|   100005  1,2,3      57963/udp6  mountd
|   100021  1,3,4      33957/udp6  nlockmgr
|   100021  1,3,4      34127/tcp   nlockmgr
|   100021  1,3,4      44773/tcp6  nlockmgr
|   100021  1,3,4      46176/udp   nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
873/tcp   open  rsync       (protocol version 31)
34127/tcp open  nlockmgr    1-4 (RPC #100021)
38403/tcp open  java-rmi    Java RMI
39085/tcp open  mountd      1-3 (RPC #100005)
57689/tcp open  mountd      1-3 (RPC #100005)
Service Info: Host: VULNNET-INTERNAL; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 121.30 seconds
```

Vemos que tenemos varios servicios disponibles donde empezar a enumerar, en mi caso empezaré por el **Samba**

### Samba

Haré uso de la herramienta ```smbclient``` para ver si tenemos la capacidad de hacer un Null Sesion (Ingresar sin credenciales):

```bash
$~ smbclient -L 10.10.104.158 -N       

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	shares          Disk      VulnNet Business Shares
	IPC$            IPC       IPC Service (vulnnet-internal server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

Se puede ver que hay un recurso compartido con el nombre **shares**, veamos lo que hay dentro 

```bash
smbclient //10.10.104.158/shares -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Tue Feb  2 09:20:09 2021
  ..                                  D        0  Tue Feb  2 09:28:11 2021
  temp                                D        0  Sat Feb  6 11:45:10 2021
  data                                D        0  Tue Feb  2 09:27:33 2021

		11309648 blocks of size 1024. 3275916 blocks available
smb: \> ls data/*
  .                                   D        0  Tue Feb  2 09:27:33 2021
  ..                                  D        0  Tue Feb  2 09:20:09 2021
  data.txt                            N       48  Tue Feb  2 09:21:18 2021
  business-req.txt                    N      190  Tue Feb  2 09:27:33 2021

    11309648 blocks of size 1024. 3275916 blocks available
smb: \> ls temp/*
  .                                   D        0  Sat Feb  6 11:45:10 2021
  ..                                  D        0  Tue Feb  2 09:20:09 2021
  services.txt                        N       38  Sat Feb  6 11:45:09 2021

		11309648 blocks of size 1024. 3275916 blocks available
smb: \>
```
Tenemos 2 directorios: **temp** y **data**

* Directorio temp --> flag services.txt
* Directorio data --> data.txt y bussines-req.txt

Despues de descargar estos archivos y analizarlos, me doy cuenta de que no contienen nada valioso.

Asi que prosigo a enumerar el NFS que me ha facilitado el escaneo de ```nmap```

### NFS

Utilizo ```showmount``` que es una herramienta que nos da informacion sobre monturas NFS:

```bash
$~  showmount -e 10.10.104.158         
Export list for 10.10.104.158:
/opt/conf *
```
Y vemos que hay una montura de /opt/conf. Monto ese recurso en mi equipo y veo que hay varios directorios:

```bash
$~ sudo mount -t nfs 10.10.104.158:/opt/conf ./mount ; ls ./mount 
 
 hp   init   opt   profile.d   redis   vim   wildmidi
 
```
Tenemos varias carpetas donde buscar archivos críticos, podriamos ir una por una, pero yo en este caso voy a utilizar ```grep```









