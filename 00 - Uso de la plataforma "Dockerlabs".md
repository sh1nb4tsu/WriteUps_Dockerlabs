Lo primero de todo, es dar las gracias a Mario (Maalfer en Github), creador de esta plataforma tan maravillosa llamada Dockerlabs.

Yendo directos al grano, para que las maquinas de DockerLabs puedan funcionar, Docker debe estar instalado en tu sistema. Para ello, usaremos el comando:
```
sudo apt install docker.io
```
##### COMO EJECUTAR LAS MAQUINAS
Una vez hayamos descargado una maquina y la hayamos descomprimido, veremos que aparecen 2 archivos:
1. auto_deploy.sh
2. NOMBREMAQUINA.tar

Lo que haremos sera "desplegar" las maquinas mediante el comando:
```
sudo bash auto_deploy.sh NOMBREMAQUINA.tar
```
##### COMO ELIMINAR LAS MAQUINAS
Para eliminarlas, es sencillo. Una vez hayamos terminado con el laboratorio, simplemente tendriamos que pulsar "CONTROL + C" y todo el laboratorio se eliminara del sistema.
##### SOLUCIÓN DE ERRORES
Es posible que en algunos casos puntuales se pueda experimentar algun error. No obstante, hay una serie de comandos que puede que solucionen la mayoría de dichos errores:
```
	1- sudo systemctl restart docker
	2- sudo docker stop $(docker ps -q)
	3- sudo docker container prune force
```
