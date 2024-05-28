---
layout: post
title: Maquina BorazuwarahCTF
subtitle: DockerLabs
header-img: img/in-post/2020-10-07/borazuwarahctf-dl.png
header-style: text
catalog: true
tags:
  -  DockerLabs
  -  Esteganografía
  -  Steghide
  -  Escalado de privilegios
  -  Fuerza bruta
---

#### **Información**
- **Máquina:** BorazuwarahCTF
- **Plataforma:** [DockerLabs](https://dockerlabs.es/)
- **Creador:** El Pingüino de Mario
- **SO:** Linux
- **Dificultad:** `Muy Facil`{:.info}

#### **Configuración Entorno**
Primeramente, como siempre para configurar las máquinas de DockerLabs, exportamos los ficheros auto_deploy.sh y en este caso pinguinazo.tar.
Para ejecutarlo utilizamos el siguiente comando:
```bash
sudo bash auto_deploy.sh pinguinazo.tar
```
![Image 1](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img1.png)

**auto_deploy.sh:** este archivo se encarga de desplegar la máquina mediante docker y, una vez presionamos `ctrl+c`{:.info} pasa a un proceso de borrado, todo automático, sin necesidad de tener conocimientos de docker.

**borazuwarahctf.tar:** contiene todo el contenido de la máquina docker; es el corazón, donde está la máquina víctima.

#### **Reconocimiento**
Lo primero que realizamos es lanzar un ping contra la IP de la máquina para comprobar que tenemos conectividad.
```bash
ping -c 1 172.17.0.2
```
![Image 2](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img2.png)

Podemos observar que el ttl es de 64, por tanto, como norma general podemos afirmas que se trata de una maquina linux.
Una vez comprobada la conectividad, iniciaremos un escaneo con nmap, para detectar puertos abiertos y lo almacenaremos en un fichero llamado scan, con el siguiente comando:
```bash
nmap -p- --open -sV -sC -sS --min-rate=5000 -vvv -n -Pn 172.17.0.2 -oN scan
```
- **-p-** --> Búsqueda de puertos abiertos
- **--open** --> Enumera los puertos abiertos
- **-sS** --> Es un modo de escaneo rápido
- **-sC** --> Que use un conjunto de scripts de reconocimiento
- **-sV** --> Que encuentre la versión del servicio abierto
- **--min-rate=5000** --> Hace que el reconocimiento aun vaya más rápido mandando no menos de 5000 paquetes
- **-n** --> No hace resolución DNS
- **-Pn** --> No hace ping
- **-vvv** --> Muestra en pantalla a medida que encuentra puertos (Verbose)
![Image 3](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img3.png)


Vemos que tiene abiertos los puertos **22 (ssh)**, **80 (http)**.

#### **Explotación**
Lo primero que hacemos es acceder vía web por el puerto 80.
![Image 4](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img4.png)

Tenemos una web que solamente contiene una imagen, esto podría significar que mediante esteganografía hayan introducido algún dato o mensaje dentro de la imagen. Para ello, nos descargamos la imagen para analizarla.

Una vez descargada tratamos de extraer algún mensaje con la herramienta “steghide”. Nos genera un archivo secreto.txt, pero al visualizarlo nos indica que sigamos buscando dentro de la imagen…
![Image 5](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img5.png)

Como no es por aqui, utilizaremos la herramiento exiftool, esta herramienta permite leer, escribir y editar metadatos en imágenes y otros archivos multimedia.
![Image 6](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img6.png)

Encontramos el usuario **borazuwarah**, pero no encontramos ninguna contraseña. Como tenemos el puerto 22 de ssh abierto, probaremos a realizar fuerza bruta al usuario que hemos encontrado.
Para ello utilizaremos la herramienta de hydra y el diccionario de contraseñas comunes rockyou.txt
![Image 7](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img7.png)

Efectivamente encontramos el password del usuario **borazuwarah** para la conexión por SSH. Nos conectamos y revisamos que podemos ejecutar como root, vemos que podemos ejecutar /bin/bash, por tanto lo ejecutamos con sudo, y ya conseguiríamos ser root.
![Image 8](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img8.png)

¡¡¡¡Así que, hemos conseguido ser `root`{:.success}!!!! 

#### **Eliminación del entorno**

Para borrar la máquina solo debemos ir a la consola donde lo desplegamos y presionar `ctrl+c`{:.info} y eliminaría y borraría todo rastro de la máquina víctima en nuestro sistema LINUX.
![Image 9](https://aanton94.github.io/blog/img/posts/dl/borazuwarahctf/img9.png)

`¡¡¡Espero que os haya gustado y nos vemos en la próxima!!!`{:.success}
