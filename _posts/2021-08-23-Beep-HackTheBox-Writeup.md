---
title: "Beep - HTB Writeup"
layout: single
excerpt: "Beep es una máquina de dificultad fácil de HackTheBox. En esta máquina mostramos 2 formas distintas de comprometer la máquina, una mediante una vulnerabilidad de tipo Shellshock y otra mediante un LFI."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/130439192-5c62dfdf-6672-417d-9530-3dfa6f441c23.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - LFI
  - Shellshock
  - Elastix
  - Linux
---

![titulomaquina](https://user-images.githubusercontent.com/67548295/130441416-5386c028-bbd3-43de-a562-8e3c55f9affe.png)

# Enumeración

Comenzamos como siempre, enviando una traza por ICMP para ver si la máquina se encuentra activa y para identificar el OS:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.7
PING 10.10.10.7 (10.10.10.7) 56(84) bytes of data.
64 bytes from 10.10.10.7: icmp_seq=1 ttl=63 time=69.8 ms

--- 10.10.10.7 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 69.779/69.779/69.779/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está activa y que observando el TTL, concluimos que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

## Nmap

Comenzamos la fase de enumeración de puertos con `nmap`:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -v -n 10.10.10.7 -sC -sV -oN targeted
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-23 12:46 WEST
Scanning 10.10.10.7 [65535 ports]
Discovered open port 143/tcp on 10.10.10.7
Discovered open port 993/tcp on 10.10.10.7
Discovered open port 22/tcp on 10.10.10.7
Discovered open port 110/tcp on 10.10.10.7
Discovered open port 111/tcp on 10.10.10.7
Discovered open port 443/tcp on 10.10.10.7
Discovered open port 25/tcp on 10.10.10.7
Discovered open port 3306/tcp on 10.10.10.7
Discovered open port 80/tcp on 10.10.10.7
Discovered open port 995/tcp on 10.10.10.7
Discovered open port 5038/tcp on 10.10.10.7
Discovered open port 4445/tcp on 10.10.10.7
Discovered open port 4559/tcp on 10.10.10.7
Discovered open port 10000/tcp on 10.10.10.7
Discovered open port 878/tcp on 10.10.10.7
Discovered open port 4190/tcp on 10.10.10.7
Nmap scan report for 10.10.10.7
Host is up (0.068s latency).
Not shown: 65519 closed ports
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 4.3 (protocol 2.0)
| ssh-hostkey: 
|   1024 ad:ee:5a:bb:69:37:fb:27:af:b8:30:72:a0:f9:6f:53 (DSA)
|_  2048 bc:c6:73:59:13:a1:8a:4b:55:07:50:f6:65:1d:6d:0d (RSA)
25/tcp    open  smtp       Postfix smtpd
|_smtp-commands: beep.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, 
80/tcp    open  http       Apache httpd 2.2.3
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.2.3 (CentOS)
|_http-title: Did not follow redirect to https://10.10.10.7/
110/tcp   open  pop3       Cyrus pop3d 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
|_pop3-capabilities: TOP UIDL APOP IMPLEMENTATION(Cyrus POP3 server v2) PIPELINING USER LOGIN-DELAY(0) AUTH-RESP-CODE EXPIRE(NEVER) RESP-CODES STLS
111/tcp   open  rpcbind    2 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1            875/udp   status
|_  100024  1            878/tcp   status
143/tcp   open  imap       Cyrus imapd 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
|_imap-capabilities: THREAD=ORDEREDSUBJECT NAMESPACE ATOMIC RIGHTS=kxte Completed SORT=MODSEQ CONDSTORE RENAME THREAD=REFERENCES MAILBOX-REFERRALS 
443/tcp   open  ssl/https?
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Issuer: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Public Key type: rsa
| Public Key bits: 1024
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2017-04-07T08:22:08
| Not valid after:  2018-04-07T08:22:08
| MD5:   621a 82b6 cf7e 1afa 5284 1c91 60c8 fbc8
|_SHA-1: 800a c6e7 065e 1198 0187 c452 0d9b 18ef e557 a09f
|_ssl-date: 2021-08-23T11:50:08+00:00; 0s from scanner time.
878/tcp   open  status     1 (RPC #100024)
993/tcp   open  ssl/imap   Cyrus imapd
|_imap-capabilities: CAPABILITY
995/tcp   open  pop3       Cyrus pop3d
3306/tcp  open  mysql      MySQL (unauthorized)
|_ssl-cert: ERROR: Script execution failed (use -d to debug)
|_ssl-date: ERROR: Script execution failed (use -d to debug)
|_sslv2: ERROR: Script execution failed (use -d to debug)
|_tls-alpn: ERROR: Script execution failed (use -d to debug)
|_tls-nextprotoneg: ERROR: Script execution failed (use -d to debug)
4190/tcp  open  sieve      Cyrus timsieved 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4 (included w/cyrus imap)
4445/tcp  open  upnotifyp?
4559/tcp  open  hylafax    HylaFAX 4.3.10
5038/tcp  open  asterisk   Asterisk Call Manager 1.1
10000/tcp open  http       MiniServ 1.570 (Webmin httpd)
|_http-favicon: Unknown favicon MD5: 74F7F6F633A027FA3EA36F05004C9341
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesnt have a title (text/html; Charset=iso-8859-1).
Service Info: Hosts:  beep.localdomain, 127.0.0.1, example.com, localhost; OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 413.86 seconds
           Raw packets sent: 65620 (2.887MB) | Rcvd: 65617 (2.625MB)
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


Vemos varios puertos abiertos, empezaré enumerando el servicio HTTP.

# User.txt

## Método 1

Hago uso de la herramienta `whatweb` para recolectar información del servidor web que hay en el puerto 80, así como tecnologías en uso y otra información:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $whatweb 10.10.10.7
http://10.10.10.7 [302 Found] Apache[2.2.3], Country[RESERVED][ZZ], HTTPServer[CentOS][Apache/2.2.3 (CentOS)], IP[10.10.10.7], RedirectLocation[https://10.10.10.7/], Title[302 Found]
ERROR Opening: https://10.10.10.7/ - SSL_connect returned=1 errno=0 state=error: dh key too small
```

Aquí estamos viendo que hay un redirect a HTTPS, pero whatweb no lo puede analizar por la clave DH que proporciona el servidor es poco robusta.

Esto se puede solucionar editando una configuración de seguridad el archivo /etc/ssl/openssl.cnf, más información de como hacerlo [aquí](https://imlc.me/dh-key-too-small)

Bueno, siguiendo con la enumeración me dispongo a visitar la web:

![navegador1](https://user-images.githubusercontent.com/67548295/130788818-b2652cb6-f45c-4f1c-990b-b7435aad5bcf.png)

Vemos un panel de login del servicio [Elastix](https://www.elastix.org/)

Si buscamos lo que es [PBX](https://www.cisco.com/c/es_mx/solutions/small-business/resource-center/collaboration/what-is-a-pbx.html), vemos que es el responsable de las molestas redirecciones de llamada de cuando contactamos con empresas de telefono o similares, no solo esto, tambien tiene más funciones que te dejo que investigues.

---

Siguiendo con la máquina, tras probar credenciales por defecto, veo que no funcionan.

Miro la captura de `nmap` y me doy cuenta de que hay un servicio HTTP corriendo el puerto 10000.

Lo visitamos y vemos un panel de login:

![navegador2](https://user-images.githubusercontent.com/67548295/130796925-10aa7169-643c-4ac2-a416-692c20b872c5.png)

Intento de nuevo credenciales por defecto, y encuentro una sorpresa:

![navegador3](https://user-images.githubusercontent.com/67548295/130797264-61f714d6-fbaa-4f99-87f5-db0da02bfbcd.png)

Hay un recurso con extension .cgi, tras ver esto se me ocurre que podriamos explotar una vulnerabilidad de tipo [shellshock](https://es.wikipedia.org/wiki/Shellshock_%28error_de_software%29)

Pruebo a ejecutar esta vulnerabilidad y consigo una shell directamente como root!!!

![terminal1](https://user-images.githubusercontent.com/67548295/130799861-cc141c1d-dc2e-4882-a79f-9dd70289b7c0.png)

Podriamos ver la flag en /root/root.txt

```bash
[root@beep /]# cat /root/root.txt 
687113XXXXXXXXXXXXXXXXXXXXXXXXXf4f5
```

## Método 2

Buscando exploits de Elastix, encuentro uno:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $searchsploit Elastix
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Elastix - 'page' Cross-Site Scripting                                                                                                                    | php/webapps/38078.py
Elastix - Multiple Cross-Site Scripting Vulnerabilities                                                                                                  | php/webapps/38544.txt
Elastix 2.0.2 - Multiple Cross-Site Scripting Vulnerabilities                                                                                            | php/webapps/34942.txt
Elastix 2.2.0 - 'graph.php' Local File Inclusion                                                                                                         | php/webapps/37637.pl
Elastix 2.x - Blind SQL Injection                                                                                                                        | php/webapps/36305.txt
Elastix < 2.5 - PHP Code Injection                                                                                                                       | php/webapps/38091.php
FreePBX 2.10.0 / Elastix 2.2.0 - Remote Code Execution                                                                                                   | php/webapps/18650.py
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Concretamente el exploit [Elastix 2.2.0 - 'graph.php' Local File Inclusion](https://www.exploit-db.com/exploits/37637)

Editamos el exploit y vemos como se consigue tener un LFI:

![terminal2](https://user-images.githubusercontent.com/67548295/130803684-a07a88ca-ceeb-46e1-ae04-abf03db9c690.png)

visito la web con esa ruta para ver si existe esta vulnerabilidad:

![navegador4](https://user-images.githubusercontent.com/67548295/130805535-ccb04949-7221-4d43-9c5b-d88100d6a73c.png)

Si funciona y obtenemos unas credenciales de un archivo de configuración!

Veo que el puerto 22 está abierto, asi que pruebo a acceder por SSH con las credenciales encontradas:

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~]
└──╼ $ssh admin@10.10.10.7
Unable to negotiate with 10.10.10.7 port 22: no matching key exchange method found. Their offer: diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
```
Obtenemos este error, pero es fácil de solucionar, solamente añadimos este parámetro y debería de funcionar:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ssh -oKexAlgorithms=+diffie-hellman-group-exchange-sha1 admin@10.10.10.7
admin@10.10.10.7 password: 
Permission denied, please try again.
admin@10.10.10.7's password: 
```

Pero no podemos acceder.

Pruebo a intentar acceder con usuario `root` en lugar de con `admin`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ssh -oKexAlgorithms=+diffie-hellman-group-exchange-sha1 root@10.10.10.7
root@10.10.10.7 password: 
Last login: Tue Jul 16 11:45:47 2019

Welcome to Elastix 
----------------------------------------------------

To access your Elastix System, using a separate workstation (PC/MAC/Linux)
Open the Internet Browser using the following URL:
http://10.10.10.7

[root@beep ~]# 
```

Y logramos acceder!

---

# Resumen y autopwn

En esta máquina hemos explotado dos formas distintas de acceder, una con un[shellshock](https://es.wikipedia.org/wiki/Shellshock_%28error_de_software%29) y
la otra con un [LFI](https://www.welivesecurity.com/la-es/2015/01/12/como-funciona-vulnerabilidad-local-file-inclusion/). Ambas formas accediendo directamente como root.
También he desarrollado un script autopwn que te proporciona una shell como root en la máquina tras ejecutarlo, está disponible en mi [GitHub](https://github.com/z3rObyte/HackTheBox-Autopwn)





