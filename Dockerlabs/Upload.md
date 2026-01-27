![1](Images/1.png)


Comenzamos escanenado con ``nmap`` los puertos abiertos de la máquina víctima.

``sudo nmap 172.17.0.2 -sS -p- --open --min-rate 5000 -n -Pn``

![2](Images/2.png)

Puerto 80. Muy probablemente sea servicio ``HTTP`` por ``well-known ports``, pero vamos a lanzar un segundo escaneo con ``nmap`` para ver qué tecnología y versión está corriendo.

``nmap 172.17.0.2 -sCV -p80``

![3](Images/3.png)


Tenemos un ``Apache 2.4.52``. Vamos a echarle un vistazo a nivel de navegador.

![4](Images/4.png)

Vale, tenemos una subida de ficheros, pero desconocemos si podremos acceder a posteriori al archivo. Vamos a indagar.

Como no tenemos información sobre el directorio al que se sube y no vemos nada en el código fuente, vamos a tirar fuerza bruta de directorios para intentar averiguarlo.

``gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 64 -x php,html,txt``

![5](Images/5.png)

Nos saca ``/uploads``. Muy interesante:

![6](Images/6.png)

Vale, dado que tenemos posibilidad de subida de archivos y suponemos dónde irán a parar, vamos a probar a subir un .php directamente, y si nos lo niega, intentaremos bypassearlo.

Vamos a nuestra revshell de confianza: pentestmonkey.

![7](Images/7.png)

Nos copiamos el contenido, generamos el archivo rev.php y lo subimos:

![8](Images/8.png)


Si accedemos a ``/uploads``:

![9](Images/9.png)


Vale, levantamos listener en el puerto que indicamos anteriormente (``nc -nvlp 4443``) y accedemos a nuestro archivo a través del navegador, a ver si sucede la magia:

Si revisamos listener:

![10](Images/10.png)


Estamos dentro de la máquina víctima como ``www-data``.

Hacemos tratamiento de la TTY.


# PRIVESC

Si probamos una de las escaladas típicas de linux: ``sudo -l``

![11](Images/11.png)

Muy buenas noticias. Podemos ejecutar ``/usr/bin/env`` como sudo sin necesidad de proporcionar contraseña.

https://gtfobins.github.io/gtfobins/env/#sudo

![12](Images/12.png)

``sudo /usr/bin/env /bin/bash``

![13](Images/13.png)

Hemos escalado correctamente a root.
