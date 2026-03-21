# Unidad 3:  Hardening de servidor web. Implementación de HTTPS, CPS, HSTS, mitigación de configuración insegura e instalar WAF de ModSecurity con Reglas CRS de OWASP

## Introducción

En esta práctica se ha realizado el aseguramiento (hardening) de un servidor web Apache desplegado en un entorno multicontenedor LAMP mediante Docker.

El objetivo principal ha sido identificar configuraciones inseguras y aplicar medidas de seguridad para mitigar diferentes riesgos habituales en servidores web.

Entre las medidas implementadas se encuentran:

- Configuración de servidores virtuales
- Activación de HTTPS
- Implementación de Content Security Policy (CSP)
- Ocultación de información sensible del servidor
- Bloqueo del listado de directorios
- Restricción de métodos HTTP inseguros
- Implementación de cabeceras de seguridad
- Instalación de un WAF con ModSecurity y OWASP CRS

---

## 1. Inicio del entorno de pruebas

Comienzo igual que en el resto de actividades del tema, situandome en la carpeta del entorno de pruebas del servidor LAMP en mi caso lo hago usando el comando:
```bash
cd ~/Unidad3/CreacionEntornoPruebas
```
Una vez en esta ubicación veo el estado del escenario multicontenedor con el comando
```bash
docker ps
```
Esto me permite verificar que el entorno esta activo y funcionando correctamente poara poder comenzar la práctica.
![vista docker ps](/images/img1.png)

---

## 2. Configuración del servidor virtual pps.edu
Edito el fichero de configuración del virtualhost: 
```bash
nano ~/Unidad3/CreacionEntornoPruebas/docker-compose-lamp/config/vhosts/default.conf
```
Con el contenido 
```bash
<VirtualHost *:80>

        ServerName www.pps.edu
	ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```
![archivo default.conf](/images/img2.png)


Recargo apache con el comando 
```bash
docker exec lamp-php84 /bin/bash -c "service apache2 reload"
```
![reinicio del servidor apache](/images/img3.png)

---

## 3. Resolución local de nombres: dns o fichero /etc/hosts
A continuación edito en la máquina kali el fichero 
```bash
sudo nano /etc/hots 
```

y añado 
```bash
127.0.0.1 pps.edu www.pps.edu
```
Como se ve en la siguiente imágen
![fichero hosts](/images/img4.png)

En el siguiente paso accedo mediante el buscador firefox a la web <http://www.pps.edu> y sale la web de lamp
![lamp](/images/img5.png)

---

## 4. Creación de un servidor virtual Hacker
Para crear el servidor virtual lo primero que hago es entrar al contenedor con el comando 
```bash
docker exec -it lamp-php84 /bin/bash
```
![accedo al contenedor](/images/img6.png)

A continuación creo una carpeta para el sitio y me muevo a ella con el uso de los comandos
```bash
mkdir /var/www/html/hacker 
cd /hacker
```
![carpeta del sitio](/images/img7.png)

En esta carpera creo un archivo llamado index.html con nano y pongo un contenido simple en mi caso  
```html
<h1>Servidor Hacker de aguado <3 </h1>
```
![contenido index.html](/images/img8.png)


Establezco permisos y propietarios con los comandos
```bash
chown -R www-data:www-data /var/www/html/hacker
chmod -R 755 /var/www/html/hacker
``` 
![permisos a hacker](/images/img9.png)

Creo el archivo de configuración con el comando 
> nano /etc/apache2/sites-available/hacker.conf 
y el contenido 
```bash
<VirtualHost *:80>

    ServerName www.hacker.edu

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/hacker

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```
![fichero hacker.conf](/images/img10.png)

En la siguiente captura lo que hago es activar el sitio, recargar el servicio y añadirlo para la resolución de nombres en el fichero </etc/hosts>
```bash
a2ensite hacker.conf
service apache2 reload
```
![activacion de hacker](/images/img11.png)
![reinicio de apache2](/images/img12.png)

Lo que ahgo ahora es entrar en el archivo 
```bash 
nano etc/hosts
```
Y añado "hacker" y "www.hacker.edu" delante de pps.edu y www.pps.edu
![etc/hosts hacker](/images/img13.png)

Desde firefox pruebo que en el buscador puedo abrir la web http://www.hacker.edu
![www.hacker.edu firefox](/images/img14.png)

---

## 5. Habilitar HTTPS con SSL/TLS en Servidor Apache
Para proteger nuestro servidor es crucial habilitar HTTPS en el servidor local. Veamos cómo podemos habilitarlo en Apache con dos métodos diferentes.
Lo que he hecho ha sido seguir el método 1 del ejercicio guiado el profesor, el cual es:
**Habilitar HTTPS en Apache con OpenSSL con certificado autofirmado**

### Paso 1: Crear la clave privada y el certificado SSL
Como estoy trabajando bajo docker, las operaciones las sigo haciendo en el contenedor, para acceder a el ejecuto
```bash
docker exec -it lamp-php84 /bin/bash
```
![contenedor lamp docker](/images/img15.png)

Me muevo a la carpeta SSL
```bash
cd etc/apacher2/ssl/
```
![carpeta ssl](/images/img16.png)

En el siguiente paso creo el certificado ssl con el comando 
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout localhost.key -out localhost.crt
``` 
Explicación de los parámetros del comando:

+ req: inicia la generación de una solicitud de certificado.
+ -x509: crea un certificado autofirmado en lugar de una CSR.
+ -nodes: omite el cifrado de la clave privada, evitando el uso de contraseña.
+ -newkey rsa:2048: genera una nueva clave RSA de 2048 bits.
+ -keyout server.key: nombre del archivo que contendrá la clave privada: Observa que estamos una clave privada con nombre localhost. Si volvemos a generar se reescribirá. Si queremos tener varias claves, lo único que tenemos que hacer es crear archivos con diferentes nombres.
+ -out server.crt: nombre del archivo de salida para el certificado.
+ -days 365: el certificado será válido por 365 días.
y cuando me pregunte Common Name pongo 
>pps.edu
![crear certificado ssl](/images/img17.png)

### Paso 2: Configurar Apache para usar HTTPS
Una vez he creado el certificado y la clave privada debo configurar Apache para poder usarlos. Para ello cambio el fichero de configuración:
> ./config/vhosts/default.conf
Y el siguiente contenido:
```
<VirtualHost *:80>

    ServerName www.pps.edu

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>

<VirtualHost *:443>
    ServerName www.pps.edu

   #activar uso del motor de protocolo SSL
    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/localhost.crt
    SSLCertificateKeyFile /etc/apache2/ssl/localhost.key

    DocumentRoot /var/www/html
</VirtualHost>
```
![fichero default.conf](/images/img18.png)

### Paso 3: Habilitar SSL y el sitio
A continuación vuelvo al contenedor y activo el módulo ssl con el comando 
```bash
a2enmod ssl
``` 
y reinicio apache con el comando 
```bash
service apache2 restart
```
![activacion ssl](/images/img19.png)

### Paso 4: pongo dirección en /etc/hosts o habilito el puerto 443
Añado al dominio en el archivo /etc/hosts de la máquina anfitriona para que resulva bien los dns.
Ahora el servidor soportaría HTTPS. Accedemos al servidor en la siguiente dirección: https://www.pps.edu
***He activado SSL pero no lo he firmado con ninguan entidad de certificación, por lo que nos deja trabajar con SSL pero nos va a dar un aviso de servidor no confiable, por lo que tengo que darle a Avanzado y Acceder a www.pps.edu (sitio no seguro)***
![pps.edu con ssl](/images/img20.png)
![pps.edu con ssl otra](/images/img21.png)

---

## 6. Forzar HTTPS en Apache2 (default.conf y .htaccess)
Mediante esta medida de seguridad fuerzo a que siempre se utilice HTTPS en el servidor aunque el navegador de algún cliente diga que no lo soporta. Tengo dos opciones para forzar HTTPS:
+ Mediante la configuración de apache en la configuración del sitiol.
+ Mediante la inclusión en las carpetas de contenidos del archivo htaccess.
En este caso voy a explicar como hacer la primera opción.
### Paso 1: Configuración en default.conf (archivo de configuración de Apache)
en esta captura lo que hago es editar
> ./config/vhosts/default.conf
Y añado 
```
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```
![archivo default.conf](/images/img22.png)

activo el módulo con el comando 
```bash
docker exec lamp-php84 /bin/bash -c "a2enmod rewrite
```
y reinicio apache2 desde la máquina kali con:
```
docker exec lamp-php84 /bin/bash -c "service apache2 restart
```
![mod_rewrite](/images/img23.png)

---

## 7. Implementación y Evaluación de Content Security Policy (CSP)
Para reforzar más HTTPS podemos implementar la política de seguridad de contenidos:

CSP (Content Security Policy) es un mecanismo de seguridad que limita los orígenes de scripts, estilos e imágenes en una aplicación web para evitar ataques como XSS. Para habilitarla añadimos en el fichero de configuración del sitio y lo que hago es añadir al fichero 
> ./config/vhosts/default.conf" 
La siguiente configuración
```
<IfModule mod_headers.c>
    Header always set Content-Security-Policy "default-src 'self'; script-src 'self'  object-src 'none'; base-uri 'self'; frame-ancestors 'none'"
</IfModule>
```
Por ejemplo, de esta forma solo permitimos la carga de contenidos de nuestro sitio, ningúno de servidores externos.
![CSP default.conf](/images/img24.png)

---

## 8. Identificación y Corrección de Security Misconfiguration
En este apartado veremos la configuración segura en servidores y aplicaciones web

Para comprobar si hay exposición de infomración sensible en nuestro servidor ejecuto 
```bash
curl -I http://pps.edu
``` 
Y como la respuesta contiene 
> "Server: Apache/2.4.66 (Debian) y/o X-Powered-By: PHP/8.4.18 el sistema nos está ofreciendo información sobre las versiones de Apache y PHP." 

Por lo tanto Los atacantes pueden aprovechar vulnerabilidades conocidas en versiones específicas de software.
![1er curl](/images/img25.png)

Corrijo la configuración del servidor de apache entrando n el contenedor con el comando
```bash
docker exec -it lamp-php84 /bin/bash
```
y pongo la configuración mínima por defecto en el fichero
> /etc/apache2/conf-available/security.conf
> 
En el que pondré 
> ServerSignature Off
>  
> ServerTokens Prod
![security.conf SS off](/images/img26.png)

Vuelvo a ejecutar
```bash
curl -I http://pps.edu
``` 
En la captura aún se ve como nos muestra informacion de la version de apache usada.
![2o curl](/images/img27.png)

Las directivas pueden estar en distintos archivos según la distribución y la configuración de Apache. Intentar encontrarlas desde el terminal en nuestra máquina Apache con:
```bash
grep -Ri "ServerSignature\|ServerTokens" /etc/apache2/
```

Como vemos en la imagen, nosotros las tenemos únicamente en el fichero 
> /etc/apache2/conf-available/security.conf 
 
![grep Ri](/images/img28.png)

Como vemos en la imagen, nosotros las tenemos únicamente en el fichero /etc/apache2/conf-available/security.conf (Recordamos que las configuraciones presentes en el directorio sites-enabled son enlaces a las presentes en el directorio sites-available).
En los sistemas que usan Debian/Ubuntu como base, las directivas ServerSignature y ServerTokens se configuran en el archivo /etc/apache2/conf-available/security.conf.

Reinicio apache desde la máquina anfitriona con "docker exec lamp-php84 /bin/bash -c
```bash
docker exec lamp-php84 /bin/bash -c "service apache2 reload"
```
![nuevo reinicio](/images/img29.png)

Vuelvo a ejecutar
```bash
curl -I http://pps.edu
``` 
![3er curl](/images/img30.png)

**Ocultar la versión de PHP**
Para ocultar la version de php Editar el archivo de configuración de PHP correspondiente. En nuestro caso: desde la máquina anfitriona " 
```bash
nano ./config/php/php.ini
``` 
Y pongo la siguiente configuracion o contenido 
```memory_limit = 256M
post_max_size = 100M
upload_max_filesize = 100M

# Xdebug 2
#xdebug.remote_enable=1
#xdebug.remote_autostart=1
#xdebug.remote_connect_back=1
#xdebug.remote_host=host.docker.internal
#xdebug.remote_port=9000

# Xdebug 3
#xdebug.mode=debug
xdebug.start_with_request=yes
date.timezone = "Europe/Madrid"
#xdebug.client_host=host.docker.internal
#xdebug.client_port=9003
#xdebug.idekey=VSCODE
#disable_functions = shell_exec, system, exec, passthru, popen, proc_open
disable_functions =
open_basedir =
# Nos muestra información de PHP
expose_php = On
#Deshabilita que se muestre la información de PHP
#expose_php= Off
```
![archivo php.ini](/images/img31.png)

Vuelvo a recargar el servidor de apache con
```bash
docker exec lamp-php84 /bin/bash -c "service apache2 reload"
```
![otro reinicio de apache2](/images/img32.png)

Para que no se nos muestre la información de PHP, cambiamos expose_php = On por expose_php = Off en el php.ini
![archivo php.ini de nuevo](/images/img33.png)

Cambiamos también para que no se nos muestre la información de apache, ponemos el archivo de configuración de seguridad de apache  
> /etc/apache2/conf-available/security.conf. 

Pongo 
> ServerTokens Full
![security.conf](/images/img34.png)

Vuelvo a ejecutar
```bash
curl -I http://pps.edu
``` 
para obtener información del servidor y ya no debería mostrar la versión de PHP
![curl sin info](/images/img35.png)

**Otras mitigaciones para Configuración Insegura y Mejores Prácticas**
Eliminamos el archivo .htaccess que hemos colocado anteriormente.
![adios al .htaccess](/images/img36.png)

Vuelvo al archivo de configuración 
> /etc/apache2/sites-enabled/default.conf

Y pongo la siguiente configuración 
```
<VirtualHost *:80>

    ServerName www.pps.edu

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    <Directory /var/www/html>
        Options +Indexes 
        AllowOverride None 
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>

<VirtualHost *:443>
    ServerName www.pps.edu

   #activar uso del motor de protocolo SSL 
    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/localhost.crt
    SSLCertificateKeyFile /etc/apache2/ssl/localhost.key

    DocumentRoot /var/www/html
</VirtualHost>
```
![cambio archivo default.conf](/images/img37.png)

Para la prueba, creo una carpeta de ejemplo e introduzco en ella dos archivos vacíos: esto lo hago usando los comandos 
```bash
# desde el contenedor
mkdir /var/www/html/hacker/ejemplo
touch /var/www/html/hacker/ejemplo/ejemplo1.txt
touch /var/www/html/hacker/ejemplo/ejemplo2.txt
chmod -R www-data:www-data /var/www/htmlhacker//ejemplo
# si quisieramos hacerlo desde la máquina anfitriona tendríamos que cambiar la ruta a
# ./www/hacker/ejemplo
```
![archivos ejemplo](/images/img38.png)

Edito la configuracion del sitio hacker y pongo:
```
<Directory /var/www/html/hacker>
    Options -Indexes +FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```
![archivo hacker.conf](/images/img39.png)

Como siempre recargo apache con el comando
```bash
service apache2 reload
```
![otro reinicio mas como no de apache](/images/img40.png)

Pruebo en el navegador http://hacker.edu/ejemplo y da error como se ve en la captura.
![firefox edu/ejemplo](/images/img41.png)

Cambio los permisos del archivo con el comando 
´´´bash
chmod 640 /etc/apache2/apache2.conf
´´´
![cambio de permisos apache](/images/img42.png)

**Desactivar métodos HTTP inseguros**
En el fichero 
> nano /etc/apache2/sites-enabled/default.conf

añado
```
<Directory />
    <LimitExcept GET POST>
        Deny from all
    </LimitExcept>
</Directory>
```
Esto es para desactivar métodos HTTP inseguros como PUT, DELETE, TRACE u OPTIONS
![de nuevo fichero default.conf](/images/img43.png)

Vuelvo a recargar apache con el comando 
```bash
service apache2 reload
```
![reinicio de apache desde dentro](/images/img44.png)

**Configurar cabeceras de seguridad en Apache**
Aplicamos diferentes mejoras que nos proporciona el módulo headers.
Para habilitar el módulo:
```bash
a2enmod headers
```
![a2enmod](/images/img45.png)


Incluimos en /etc/apache2/defaul.confo en nuestro archivo de sitio virtual /etc/apache2/sites-available/XXXXX.conf: 
```
Header always unset X-Powered-By
Header always set X-Frame-Options "DENY"
Header always set X-XSS-Protection "1; mode=block"
Header always set X-Content-Type-Options "nosniff"
```
![cabeceras apache](/images/img46.png)

Vuelvo a recargar apache con el comando 
```bash
service apache2 reload
```
![reinicio de apache](/images/img47.png)

## 9. Configuración de mod_security con reglas OWASP CRS en Apache
**Instalar mod_security**
Para instalar la libreria de Apache ModSecurity ejecuto en línea de comandos 
```bash
apt update
apt install libapache2-mod-security2
```
![instalacion mod_security](/images/img48.png)

copio el archivo de configuración redomendado con el comando
```bash
cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
nano /etc/modsecurity/modsecurity.conf
```
![copia del fichero](/images/img49.png)

En el fichero de configuracion 
```bash
nano /etc/modsecurity/modsecurity.conf
```

Me aseguro de que esté 
> SecRuleEngine DetectionOnly

como se ve en la imágen a continuación.
![copia del fichero](/images/img50.png)

Vuelvo a guardar y reiniciar el servidor de Apache con el comando 
```bash
service apache2 reload
```
![vuelta reinicio de apache](/images/img51.png)

Verifico que mod_security esté cargado con el comando 
```bash
apachectl -M | grep security
``` 
y me aseguro de que pone 
> security2_module (shared)
 
tal cual se ve en la siguiente captura
![security2_module](/images/img52.png)

---

## 10. Descargar OWASP ModSecurity Core Rule Set (CRS)
Para incorporar las reglas CRS de OWASP a mod_security clonamos el repositorio y copiamos el archivo de configuración. 
```bash
cd /etc/modsecurity
apt install git
git clone https://github.com/coreruleset/coreruleset.git
cd coreruleset
cp crs-setup.conf.example crs-setup.conf
```
![clonacion](/images/img53.png)

 Para comprobar si están añadidas las reglas de modsecurity-crs, ejecuto 
 ```bash
 apache2ctl -t -D DUMP_INCLUDES|grep modsecurity
 ``` 
![Comprobacion security](/images/img54.png)

Creo el archivo 
```bash
nano /etc/apache2/conf-available/security-crs.conf
``` 
y añado: 
```
# Activar CRS
IncludeOptional /etc/modsecurity/coreruleset/crs-setup.conf
IncludeOptional /etc/modsecurity/coreruleset/rules/*.conf
```
![archivo security-crs.conf](/images/img55.png)


Luego, habilito el archivo de configuración y reinicio el servicio con los comandos 
```bash
a2enconf security-crs
service apache2 reload
```
![habilitada conf y reinicio apache como no](/images/img56.png)

**Probar el WAF**
Pruebo reglas usando cadenas típicas de ataques en la URL: https://pps.edu/lfi.php?file=../../../../etc/passwd El acceso da bloqueado como se ve en la imágen
![Pruebo reglas](/images/img57.png)

**Ver logs de ModSecurity**
Observo los logs de modSecurity usando
```bash
cat /var/log/apache2/modsec_audit.log
```
![Logs](/images/img58.png)

**Volver a dejarlo todo Niquelao**
Empiezo a dejarlo todo niquelao con 
```bash
apt remove --purge  libapache2-mod-security2
rm -rf /etc/modsecurity
```
![niquelao](/images/img59.png)

Volvemos a colocar los archivos por defecto 
```bash
apache2ctl -t -D DUMP_INCLUDES|grep modsecurity
```
![archivos por defecto](/images/img60.png)


Vuelvo a colocar los archivos por defecto "/etc/apache2/apache2.conf" y "/usr/local/etc/php/php.ini"
![archivos por defecto2](/images/img61.png)

## 11. Mejoras de seguridad implementadas

Durante la práctica se han aplicado diferentes mecanismos de seguridad que mejoran la protección del servidor web. A continuación se explica brevemente qué aporta cada uno de ellos.

### HTTPS (SSL/TLS)

HTTPS permite cifrar la comunicación entre el cliente y el servidor mediante el protocolo SSL/TLS.  
Esto evita que un atacante pueda interceptar o modificar la información que se transmite por la red (por ejemplo contraseñas o cookies de sesión). Además, garantiza la autenticidad del servidor y protege contra ataques de tipo **Man-in-the-Middle**.

### Forzar HTTPS

Forzar HTTPS mediante reglas de redirección asegura que todos los usuarios accedan siempre al sitio utilizando conexiones cifradas, incluso si intentan entrar mediante HTTP.  
De esta forma se evita que el tráfico pueda viajar sin protección.

### Content Security Policy (CSP)

CSP es una política de seguridad que permite definir qué recursos puede cargar una página web (scripts, imágenes, estilos, etc.).  
Con esta política se limita la ejecución de código procedente de fuentes externas y se reduce el riesgo de ataques como **Cross-Site Scripting (XSS)**.

### HSTS (HTTP Strict Transport Security)

HSTS es una cabecera de seguridad que indica al navegador que el sitio web solo debe ser accedido mediante HTTPS.  
Esto evita que un atacante pueda forzar conexiones inseguras mediante ataques de **downgrade** o manipulación del tráfico.

### Ocultación de información del servidor

Ocultar información como la versión de Apache o PHP evita que posibles atacantes obtengan datos sobre el software que utiliza el servidor.  
Esta medida reduce la posibilidad de que se aprovechen vulnerabilidades conocidas asociadas a versiones concretas del software.

### Desactivar listado de directorios

Cuando el listado de directorios está activado, un usuario puede ver todos los archivos de una carpeta si no existe un `index.html`.  
Desactivarlo evita que se exponga información sensible o archivos internos del servidor.

### Restricción de métodos HTTP inseguros

Limitar los métodos HTTP a los estrictamente necesarios (por ejemplo GET y POST) evita que se utilicen otros métodos como PUT, DELETE o TRACE que podrían ser aprovechados por atacantes para manipular recursos del servidor.

### Cabeceras de seguridad HTTP

Las cabeceras de seguridad añaden protección adicional al navegador del usuario. Algunas de las más importantes son:

- **X-Frame-Options**: evita ataques de clickjacking.
- **X-XSS-Protection**: ayuda a mitigar ataques XSS.
- **X-Content-Type-Options**: evita ataques basados en MIME sniffing.

### ModSecurity con reglas OWASP CRS

ModSecurity actúa como un **Web Application Firewall (WAF)** que analiza las peticiones HTTP antes de que lleguen a la aplicación web.  
Las reglas **OWASP CRS (Core Rule Set)** detectan patrones de ataques comunes como:

- SQL Injection
- Cross-Site Scripting (XSS)
- Path Traversal
- Local File Inclusion (LFI)

Gracias a esto el servidor puede **detectar o bloquear ataques automáticamente**, añadiendo una capa adicional de protección a la aplicación web.
