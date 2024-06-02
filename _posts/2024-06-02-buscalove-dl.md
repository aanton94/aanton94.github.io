---
layout: post
title: Maquina BuscaLove
subtitle: DockerLabs
header-img: img/in-post/2020-10-07/database-dl.png
header-style: text
catalog: true
tags:
  -  DockerLabs
  -  LFI
  -  Fuerza bruta
  -  Escalado de privilegios
  -  Fuzzing Web
---

#### **Información**
- **Máquina:** BuscaLove
- **Plataforma:** [DockerLabs](https://dockerlabs.es/)
- **Creador:** El Pingüino de Mario
- **SO:** Linux
- **Dificultad:** `Facil`{:.success}

#### **Configuración Entorno**
Primeramente, como siempre para configurar las máquinas de DockerLabs, exportamos los ficheros auto_deploy.sh y en este caso pinguinazo.tar.
Para ejecutarlo utilizamos el siguiente comando:
```bash
sudo bash auto_deploy.sh buscalove.tar
```
![Image 1](https://aanton94.github.io/blog/img/posts/dl/buscalove/img1.png)

**auto_deploy.sh:** este archivo se encarga de desplegar la máquina mediante docker y, una vez presionamos `ctrl+c`{:.info} pasa a un proceso de borrado, todo automático, sin necesidad de tener conocimientos de docker.

**buscalove.tar:** contiene todo el contenido de la máquina docker; es el corazón, donde está la máquina víctima.

#### **Reconocimiento**
Lo primero que realizamos es lanzar un ping contra la IP de la máquina para comprobar que tenemos conectividad.
```bash
ping -c 1 172.18.0.2
```
![Image 2](https://aanton94.github.io/blog/img/posts/dl/buscalove/img2.png)

Podemos observar que el ttl es de 64, por tanto, como norma general podemos afirmas que se trata de una maquina linux.
Una vez comprobada la conectividad, iniciaremos un escaneo con nmap, para detectar puertos abiertos y lo almacenaremos en un fichero llamado scan, con el siguiente comando:
```bash
nmap -p- --open -sV -sC -sS --min-rate=5000 -vvv -n -Pn 172.18.0.2 -oN scan
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
![Image 3](https://aanton94.github.io/blog/img/posts/dl/buscalove/img3.png)


Vemos que tiene abiertos los puertos **22 (ssh)** y **80 (http)**.

#### **Explotación**
Al acceder por el puerto 80 y vemos la página por defecto de Apache.

Realizamos fuzzing web con gobuster para tratar de localizar directorios. Encontramos el directorio /wordpress.
![Image 4](https://aanton94.github.io/blog/img/posts/dl/buscalove/img4.png)
```bash
gobuster dir -u http://172.18.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -q -x php,html,txt  
```
Revisamos la web y no encontramos nada relevante, lo único un comentario en el código fuente, que dice lo siguiente:
```html
<!-- El desarollo de esta web esta en fase verde muy verde te dejo aqui la ventana abierta con mucho love para los curiosos que gustan de leer -->
```

Realizamos fuzzing del /wordpress y localizamos el archivo index.php.
```bash
gobuster dir -u http://172.18.0.2/wordpress/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -q -x php,html,txt 
```
![Image 5](https://aanton94.github.io/blog/img/posts/dl/buscalove/img5.png)

Vemos que es la misma página que anteriormente, por lo tanto, trataremos de hacer fuzzing com wfuzz para localizar si tiene una vulnerabilidad de tipo LFI (Local File Inclusion).
```bash
wfuzz -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 172.18.0.2/wordpress/index.php'?'FUZZ=../../../../../../../etc/passwd --hc 404 --hl 40
```
![Image 6](https://aanton94.github.io/blog/img/posts/dl/buscalove/img6.png)
Encontramos que el parámetro es “love”.

![Image 7](https://aanton94.github.io/blog/img/posts/dl/buscalove/img7.png)

Localizamos los usuarios Pedro y Rosa.
Ahora mediante fuerza bruta con hydra y el diccionario rockyou.txt trataremos de averiguar la contraseña de las conexiones SSH.

Para Pedro no localizamos contraseña, pero para Rosa si!!!
![Image 8](https://aanton94.github.io/blog/img/posts/dl/buscalove/img8.png)

Accedemos vía SSH con el usuario de Rosa, una vez dentro utilizamos sudo -l para ver que podemos ejecutar en modo root.
Vemos que podemos ejecutar tanto el comando ls como el comando cat.
Después de buscar en diferentes directorios, encontramos en el directorio root el archivo secret.txt.

Convertimos el código hexadecimal a ASCII y tenemos:
**NZXWCY3FOJ2GC4TBONXXG2IK**
Este parece ser un texto codificado, posiblemente una cadena en Base32. Para verificarlo y decodificarlo, intentemos convertirlo de Base32 a texto ASCII.
La cadena decodificada de Base32 resulta en:
**noacertarasosi**
Probamos si es la contraseña de root y no, asique intentamos con el usuario Pedro, y logramos acceder.
![Image 10](https://aanton94.github.io/blog/img/posts/dl/buscalove/img10.png)

Revisamos que podemos ejecutar como root y tenemos que podemos ejecutar /usr/bin/env, por tanto, podemos escalar privilegios de la siguiente manera.
![Image 11](https://aanton94.github.io/blog/img/posts/dl/buscalove/img11.png)

¡¡¡¡Así que, hemos conseguido ser `root`{:.success}!!!! 

#### **Eliminación del entorno**

Para borrar la máquina solo debemos ir a la consola donde lo desplegamos y presionar `ctrl+c`{:.info} y eliminaría y borraría todo rastro de la máquina víctima en nuestro sistema LINUX.

![Image 12](https://aanton94.github.io/blog/img/posts/dl/buscalove/img12.png)

`¡¡¡Espero que os haya gustado y nos vemos en la próxima!!!`{:.success}
