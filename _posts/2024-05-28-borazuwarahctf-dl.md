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
![Image 1](https://aanton94.github.io/blog/img/posts/dl/collection/img1.png)

**auto_deploy.sh:** este archivo se encarga de desplegar la máquina mediante docker y, una vez presionamos `ctrl+c`{:.info} pasa a un proceso de borrado, todo automático, sin necesidad de tener conocimientos de docker.

**borazuwarahctf.tar:** contiene todo el contenido de la máquina docker; es el corazón, donde está la máquina víctima.

#### **Reconocimiento**
Lo primero que realizamos es lanzar un ping contra la IP de la máquina para comprobar que tenemos conectividad.
```bash
ping -c 1 172.17.0.2
```
![Image 2](https://aanton94.github.io/blog/img/posts/dl/collection/img2.png)

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
![Image 3](https://aanton94.github.io/blog/img/posts/dl/collection/img3.png)


Vemos que tiene abiertos los puertos **22 (ssh)**, **80 (http)** y **27017 (mongodb)**.

#### **Explotación**
Lo primero que hacemos es acceder vía web por el puerto 80.
![Image 4](https://aanton94.github.io/blog/img/posts/dl/collection/img4.png)

Tenemos la página de inicio de Apache, pero no detectamos nada destacable.
Por lo tanto, utilizamos gobuster para hacer fuzzing web.
![Image 5](https://aanton94.github.io/blog/img/posts/dl/collection/img5.png)

Nos encontramos que tenemos una ruta de wordpress, accedemos a ella y nos encontramos con una web creada en wordpress.
![Image 6](https://aanton94.github.io/blog/img/posts/dl/collection/img6.png)

Dentro de la web no vemos nada destaccable, inspeccionando el codigo fuente de la web encontramos un nombre de dominio **collections.dl**.
![Image 7](https://aanton94.github.io/blog/img/posts/dl/collection/img7.png)

Lo añadimos el el fichero /etc/hosts y refrescamos la pagina.
![Image 8](https://aanton94.github.io/blog/img/posts/dl/collection/img8.png)

Vemos que la pagina carga mejor pero no detectamos nada destacable, por lo tanto mediante la herramienta WpScan, lanzaremos un escaneo para detectar usuarios y posibles pluguins vulnerables en wordpress.
```bash
wpscan --url http://collections.dl/wordpress -e u,vp 
```
- **u** --> Busqueda de usuarios
- **vp** --> Enumera los plugins vulnerables

![Image 9](https://aanton94.github.io/blog/img/posts/dl/collection/img9.png)

Nos encuentra el usuario **chocolate** :chocolate_bar:
Probamos a realizar fuerza bruta al usuario utilizando el siguiente comando con WpScan y el diccionario Rockyou.

```bash
wpscan --url http://collections.dl/wordpress --passwords /usr/share/wordlists/rockyou.txt 
```
![Image 10](https://aanton94.github.io/blog/img/posts/dl/collection/img10.png)

Y nos encuentra que el password es el mismo que el usuario :angry:

Como ya tenemos usuario y password, accedemos al panel de administracion de Wordpress.
![Image 11](https://aanton94.github.io/blog/img/posts/dl/collection/img11.png)
![Image 12](https://aanton94.github.io/blog/img/posts/dl/collection/img12.png)

En el menú, accedemos a los plugins instalados y vemos que tenemos el plugin Site Editor Version 1.1 instalado.
Buscamos información sobre este plugin y vemos que es vulnerable a LFI (Local File Inclusion), y encontramos un script que permite explotar esta vulnerabilidad, por tanto lo descargamos desde [https://github.com/jessisec/CVE-2018-7422](https://github.com/jessisec/CVE-2018-7422).
![Image 13](https://aanton94.github.io/blog/img/posts/dl/collection/img13.png)

Lo ejecutamos siguiendo las instrucciones, y conseguimos leer el fichero /etc/passwd.
![Image 14](https://aanton94.github.io/blog/img/posts/dl/collection/img14.png)

Encontramos los usuarios de sistema **chocolate** y **dbadmin**.

Tratamos de acceder vía SSH con el usuario chocolate y la contraseña que encontramos anteriormente, pero no logramos acceder, por lo tanto, aplicamos fuerza bruta con hydra a la conexión SSH.

```bash
hydra 172.17.0.2 ssh -s 22 -l chocolate -P /usr/share/wordlists/rockyou.txt -f -I -t 64 
```
![Image 15](https://aanton94.github.io/blog/img/posts/dl/collection/img15.png)

Encontramos la contraseña del usuario **chocolate**, así que, accedemos vía SSH y logramos la conexión.
![Image 16](https://aanton94.github.io/blog/img/posts/dl/collection/img16.png)

Encontramos el archivo **mongodb-27017.sock** en el directorio /tmp. Tratamos de ejecutar mongodb desde la maquina víctima, pero no nos permite, como tenía expuesto mongodb por el puerto **27017**, lanzaremos la conexión desde la maquina atacante (tuve que instalar mongodb).
![Image 17](https://aanton94.github.io/blog/img/posts/dl/collection/img17.png)

Una vez conectados, revisamos que bases de datos tenemos, nos llama la atención la base de datos **accesos**, dentro de ella encontramos la colección **usuarios** y al leer los datos descubrimos el usuario dbadmin y su password.
![Image 18](https://aanton94.github.io/blog/img/posts/dl/collection/img18.png)

Accedemos con este usuario por SSH para comprobar si podemos realizar la escalada de privilegios, pero no detectamos nada potencialmente vulnerable.
![Image 19](https://aanton94.github.io/blog/img/posts/dl/collection/img19.png)

Como no detectamos nada probamos a acceder como root con las diferentes contraseñas que hemos encontrado y logramos acceder con la contraseña del usuario **dbadmin**.
![Image 20](https://aanton94.github.io/blog/img/posts/dl/collection/img20.png)

¡¡¡¡Así que, hemos conseguido ser `root`{:.success}!!!! 

#### **Eliminación del entorno**

Para borrar la máquina solo debemos ir a la consola donde lo desplegamos y presionar `ctrl+c`{:.info} y eliminaría y borraría todo rastro de la máquina víctima en nuestro sistema LINUX.

![Image 10](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img10.png)

`¡¡¡Espero que os haya gustado y nos vemos en la próxima!!!`{:.success}
