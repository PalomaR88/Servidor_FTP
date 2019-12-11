# Servidor FTP
FTP (siglas en inglés de File Transfer Protocol, ‘Protocolo de Transferencia de Archivos’) en informática, es un protocolo de red para la transferencia de archivos entre sistemas conectados a una red TCP (Transmission Control Protocol), basado en la arquitectura cliente-servidor. Desde un equipo cliente se puede conectar a un servidor para descargar archivos desde él o para enviarle archivos, independientemente del sistema operativo utilizado en cada equipo.

Usa los puertos 20, 21.

## Cliente-servidor
- Terminal: ftp, lftp
- Gráficos: filezilla

## Acceso
#### Anónimo
- Annomious -> Directorio de solo lectura.
Cuando se entra como anónimo solo se entra en un directorio. El directorio en proftpd.conf:
~~~
<Anonymous ~ftp>
~~~
Pero esto puede modificarse e insertar la dirección que se quiera. Más abajo, en el fichero, hay unas líneas doblemente comentadas que sirven para que el directorio usaurio por defecto para los usuarios anónimos sea de lectura y escritura (algo raro que se utilice).

#### Autentificado
- Usuarios del sistema. Al crearse no se le añade shell porque no van a loguearse. El home de estos usuarios pueden ser usados de ruta documentRoot.
- Usuarios virtuales (ldap, mysql). Es la solución más sencilla.
El acceso puede ser entrando el usuario en su home o ejaulando al usuario.

## proFTPd
- La directiva que indicaa el usuario es defaultRoot. 
> Ejemplo, usando el porcentaje para indicar los usuarios que en la ruta se indiquen:
~~~
DefaultRoot     /home/%u/ftp
~~~

1. Primera configuración:
Hay dos usuarios (user1, user2) y se quiere que cada uno de ellos tenga una página en la la dirección iesgn.org. 

Se tiene un servidor web Apache2 con las rutas pag1.iesgn.org y pag2.iesgn.org (documentRoot /var/www/user1, /var/www/user2). Hay que indicar en el fichero de configuración de proFTPd la ruta de cada usuario: /var/www/%u.

2. Añadir un usuario:
- Se crea: user3.
- En apache se indica pag3.iesgn.org /var/www/user3.
- En FTP no hay que hace nada porque el defaultRoot ya está configurado.

3. Script que automatice todo:
- Que cree el usuario.
- Que añada un registro en el documentRoot.
- Que añada un registro en el DNS.


## Ejercicios prácticos
#### Ejercicio usuario anónimo
**1. Tenemos un sitio web y un sitio ftp, queremos que cuando accedamos a www.iesgn.org/documentos, aparecen una lista de documentos (como NAS). Cuando accedamos de forma anónima al FTP, accedemos al mismo directorio /documentos.**
Se va a utilizar una máquina con el [Servidor web Apache2](https://github.com/PalomaR88/Practica_Apache2/blob/master/Vagrantfile).

En primer lugar hay que descargar el servidor ftp:
~~~
vagrant@servidor:~$ sudo apt install ftp
vagrant@servidor:/var/www/html/documentos$ sudo apt install proftpd
~~~

Se crea el directorio documentos, dento del documentos root que se indique en el fichero de configuración de Apache.
Fichero de configuración de Apache2:
~~~
DocumentRoot /var/www/html
~~~

Creación del directorio y de los fichero que contendrá:
~~~
vagrant@servidor:/var/www/html$ sudo mkdir documentos
vagrant@servidor:/var/www/html$ sudo chown www-data:www-data documentos/
vagrant@servidor:/var/www/html/documentos$ sudo touch fichero1.txt
vagrant@servidor:/var/www/html/documentos$ sudo touch fichero2.txt
vagrant@servidor:/var/www/html/documentos$ sudo touch fichero3.txt
vagrant@servidor:/var/www/html/documentos$ sudo chown www-data:www-data fichero*
~~~

Configuración del documento /etc/proftpd/proftpd.conf:
~~~
 <Anonymous /var/www/html/documentos>
   User                         ftp
   Group                                nogroup
   # We want clients to be able to login with "anonymous" as well as "ftp"
   UserAlias                    anonymous ftp
   # Cosmetic changes, all files belongs to ftp user
   DirFakeUser  on ftp
   DirFakeGroup on ftp
 
   RequireValidShell            off

   # Limit the maximum number of anonymous logins
   MaxClients                   10

   # We want 'welcome.msg' displayed at login, and '.message' displayed
   # in each newly chdired directory.
   DisplayLogin                 welcome.msg
   DisplayChdir         .message

   # Limit WRITE everywhere in the anonymous chroot
   <Directory *>
     <Limit WRITE>
       DenyAll
     </Limit>
   </Directory>
~~~

Se reinicia el servicio:
~~~
vagrant@servidor:/var/www/html/documentos$ sudo systemctl restart proftpd.service 
~~~

Se prueba en local:
~~~
vagrant@servidor:/var/www/html/documentos$ ftp localhost
Connected to localhost.
220 ProFTPD Server (Debian) [::1]
Name (localhost:vagrant): ftp
331 Anonymous login ok, send your complete email address as your password
Password:
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 EPRT command successful
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 ftp      ftp             0 Dec 10 08:07 fichero1.txt
-rw-r--r--   1 ftp      ftp             0 Dec 10 08:07 fichero2.txt
-rw-r--r--   1 ftp      ftp             0 Dec 10 08:07 fichero3.txt
226 Transfer complete
ftp> 
~~~

Y después desde otra máquina:
~~~
paloma@coatlicue:~/DISCO2/CICLO II/SERVICIO DE RED E INTERNET/Servidor_FTP$ ftp www.iesgn.org
Connected to www.iesgn.org.
220 ProFTPD Server (Debian) [::ffff:172.22.9.45]
Name (www.iesgn.org:paloma): ftp
331 Anonymous login ok, send your complete email address as your password
Password:
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 ftp      ftp             0 Dec 10 08:07 fichero1.txt
-rw-r--r--   1 ftp      ftp             0 Dec 10 08:07 fichero2.txt
-rw-r--r--   1 ftp      ftp             0 Dec 10 08:07 fichero3.txt
226 Transfer complete
ftp> 
~~~

#### Usuarios
**2. Crear un virtualhost en apache y hacer que un usuario suba ficheros al virtualhost por ftp.**

Creación del virtualhost:
~~~
vagrant@servidor:/etc/apache2/sites-available$ sudo cp 000-default.conf user1.iesgn.org
vagrant@servidor:/etc/apache2/sites-available$ sudo cp 000-default.conf user2.iesgn.org
~~~

Configuración del virtualhost de user1:
~~~
ServerName user1.iesgn.org
DocumentRoot /var/www/html/user1
~~~

Configuración del virtualhost de user2:
~~~
ServerName user2.iesgn.org
DocumentRoot /var/www/html/user2
~~~

Se crean los usuarios:
~~~
vagrant@servidor:/var/www/html$ sudo adduser user1
Adding user `user1' ...
Adding new group `user1' (1001) ...
Adding new user `user1' (1001) with group `user1' ...
Creating home directory `/home/user1' ...
Copying files from `/etc/skel' ...
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for user1
Enter the new value, or press ENTER for the default
	Full Name []: 
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n] y
vagrant@servidor:/var/www/html$ sudo adduser user2
Adding user `user2' ...
Adding new group `user2' (1002) ...
Adding new user `user2' (1002) with group `user2' ...
Creating home directory `/home/user2' ...
Copying files from `/etc/skel' ...
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for user2
Enter the new value, or press ENTER for the default
	Full Name []: ç
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
chfn: name with non-ASCII characters: 'ç'
Is the information correct? [Y/n] y
~~~

Se crean los directorios de cada uno de los usuarios:
~~~
vagrant@servidor:/var/www/html$ sudo mkdir user1
vagrant@servidor:/var/www/html$ sudo touch user1/ficherouser1.txt
vagrant@servidor:/var/www/html$ sudo mkdir user2
vagrant@servidor:/var/www/html$ sudo touch user2/ficherouser2.txt
~~~

Y se añade estos directorios en el fichero de configuración /etc/proftpd/proftpd.conf:
~~~ 
DefaultRoot                     /var/www/html/%u
~~~

Comprobación en local del user1:
~~~
vagrant@servidor:/var/www/html/user2$ ftp localhost
Connected to localhost.
220 ProFTPD Server (Debian) [::1]
Name (localhost:vagrant): user1
331 Password required for user1
Password:
230 User user1 logged in
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 EPRT command successful
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 root     root            0 Dec 10 08:44 ficherouser1.txt
226 Transfer complete
ftp> 
~~~

Comprobación en local del user2:
~~~
vagrant@servidor:/var/www/html/user2$ ftp localhost
Connected to localhost.
220 ProFTPD Server (Debian) [::1]
Name (localhost:vagrant): user2
331 Password required for user2
Password:
230 User user2 logged in
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 EPRT command successful
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 root     root            0 Dec 10 08:51 ficherouser2.txt
226 Transfer complete
ftp> 
~~~

Comprobación en remoto del user1:
~~~
paloma@coatlicue:~/DISCO2/CICLO II/SERVICIO DE RED E INTERNET/Servidor_FTP$ ftp www.iesgn.org
Connected to www.iesgn.org.
220 ProFTPD Server (Debian) [::ffff:172.22.9.45]
Name (www.iesgn.org:paloma): user1
331 Password required for user1
Password:
230 User user1 logged in
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 root     root            0 Dec 10 08:44 ficherouser1.txt
226 Transfer complete
ftp> 
~~~

Comprobación en remoto del user2:
~~~
paloma@coatlicue:~/DISCO2/CICLO II/SERVICIO DE RED E INTERNET/Servidor_FTP$ ftp www.iesgn.org
Connected to www.iesgn.org.
220 ProFTPD Server (Debian) [::ffff:172.22.9.45]
Name (www.iesgn.org:paloma): user2
331 Password required for user2
Password:
230 User user2 logged in
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 root     root            0 Dec 10 08:51 ficherouser2.txt
226 Transfer complete
ftp> 
~~~

Se cambias los permisos de los directorios:
~~~
vagrant@servidor:/var/www/html$ sudo chown user1:user1 user1/
vagrant@servidor:/var/www/html$ sudo chown user2:user2 user2/
~~~

De esta forma se pueden subir cosas:
~~~
ftp> put Servidor_FTP.md 
local: Servidor_FTP.md remote: Servidor_FTP.md
200 PORT command successful
150 Opening BINARY mode data connection for Servidor_FTP.md
226 Transfer complete
ftp> ls
200 PORT command successful
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 root     root            0 Dec 10 08:44 ficherouser1.txt
-rw-r--r--   1 user1    user1           0 Dec 10 09:17 Servidor_FTP.md
226 Transfer complete
ftp> 
~~~


