![1](Images/1.png)

Comenzamos escaneando los puertos abiertos de la máquina víctima.

``sudo nmap 172.17.0.2 -sS -p- --open --min-rate 5000 -n -Pn``

![2](Images/2.png)

Una vez conocemos los puertos abiertos, volvemos a lanzar otro escaneo sobre estos puertos abiertos para conocer los servicios y versiones que están corriendo sobre dichos puertos.

``nmap 172.17.0.2 -sCV -p22,80``

![3](Images/3.png)

Vale, tenemos SSH (``OpenSSH 9.6``) y HTTP (``Apache 2.4.58``). Por ahora no podemos hacer nada con SSH, por lo que vamos a echar un vistazo a nivel de navegador al servicio HTTP:

![4](Images/4.png)

Vemos cositas interesantes, como dos posibles usuarios: ``juan`` y ``carlota``. Los añadimos a ``users.txt``.

Si hacemos fuerza bruta de directorios o miramos el código fuente, no vemos nada interesante.

Como tenemos el servicio SSH abierto, vamos a intentar hacer fuerza bruta con estos dos usuarios utilizando la herramienta ``hydra`` y el diccionario ``rockyou``.

![5](Images/5.png)

``hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2``

![6](Images/6.png)

Nos encuentra credenciales válidas: ``carlota``:``babygirl``

Nos conectamos por SSH:

``ssh carlota@172.17.0.2``

![7](Images/7.png)

Estamos dentro de la máquina víctima como el usuario ``carlota``.


# PRIVESC

Si indagamos un poco por los directorios dentro del directorio personal de nuestro usuario, vemos:

![8](Images/8.png)

``/home/carlota/Desktop/fotos/vacaciones/imagen.jpg``

Vamos a compartirnos esta imagen con nuestra máquina atacante.

Hacemos la solicitud de recurso desde nuestra máquina atacante:

``scp carlota@172.17.0.2:/home/carlota/Desktop/fotos/vacaciones/imagen.jpg .``

![9](Images/9.png)


Si le echamos un vistazo:

![10](Images/10.png)


No parece tener nada inicialmente. Si miramos los metadatos tampoco hay nada interesante. Vamos a ver si tuviese algo de data oculta a través de esteganografía con ``steghide``:

``steghide info imagen.jpg``

![11](Images/11.png)

Nos pide una contraseña. Se intenta con la contraseña de carlota, ``babygirl``. Pero no funciona. 

Se repite el proceso sin proporcionar contraseña.


![12](Images/12.png)


Funciona. Nos ha extraído el fichero embebido "secret.txt". Si echamos un vistazo a su contenido:

![13](Images/13.png))


Parece que está en base64. Vamos a intentar decodearlo:

``cat secret.txt | base64 -d``

![14](Images/14.png)


eslacasadepinypon

Se intenta escalar a root con esta contraseña. No funciona.

Si echamos un vistazo al /etc/passwd, vemos que existen otros usuarios:

![15](Images/15.png)


Vamos a intentar pivotar a oscar con esta contraseña.

``su oscar``


![16](Images/16.png)


Hemos pivotado correctamente a oscar.

# privesc

``sudo -l``


![17](Images/17.png)


Vemos que podemos ejecutar /usr/bin/ruby como sudo sin proporcionar contraseña de root.

https://gtfobins.github.io/gtfobins/ruby/#sudo

![18](Images/18.png)


``sudo /usr/bin/ruby -e 'exec "/bin/sh"'``

![19](Images/19.png)

Hemos pivotado correctamente a root.
