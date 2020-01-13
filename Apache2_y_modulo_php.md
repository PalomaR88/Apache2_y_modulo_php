# Ejecución de script PHP
### Apache2 y módulo PHP
Instalamos apache2 y el módulo que permite que los procesos de apache2 sean capaz de ejecutar el código PHP:
~~~
apt install apache2 php7.3 libapache2-mod-php7.3
~~~

Cuando hacemos la instalación se desactiva el MPM event y se activa el prefork:
~~~
...
Module mpm_event disabled.
Enabling module mpm_prefork.
apache2_switch_mpm Switch to prefork
...
~~~

Si queremos desactivar el módulo PHP de apache2:
~~~
apt remove libapache2-mod-php7.3
~~~

Y activamos el módulo event:
~~~
a2dismod mpm_prefork
a2enmod mpm_event
systemctl restart apache2
~~~

La configuración de php está dividida según desde donde se use:
**/etc/php/7.3/cli**: Configuración de php para php7.3-cli, cuando se utiliza php desde la línea de comandos.
**/etc/php/7.3/apache2**: Configuración de php para apache2 cuando utiliza el módulo.
**/etc/php/7.3/fpm**: Configuración de php para php-fpm
**/etc/php/7.3/mods-available**: Módulos disponibles de php que puedes estar configurados en cualquiera de los escenarios anteriores.


Si nos fijamos en la configuración de php para apache2:
**/etc/php/7.3/apache2/conf.d**: Módulos instalados en esta configuración de php (enlaces simbólicos a /etc/php/7.3/mods-available).
**/etc/php/7.3/apache2/php.ini**: Configuración de php para este escenario.


### PHP-FPM
**FPM** (FastCGI Process Manager) es una implementación alternativa al PHP FastCGI. FPM se encarga de interpretar código PHP. Aunque normalmente se utiliza junto a un servidor web (Apache2 o ngnix) vamos a hacer en primer lugar una instalación del proceso y vamos a estudiar algunos parámetros de configuración y estudiar su funcionamiento.

Esto mejora el rendimiento de php, ya que php junto a Apache tiene un rendimiento bajo. Php-fpm ejecuta el código php, auqnue los módulos de Apache saben hacerlo, si se hace a través de esto va mejor. 

Para instalarlo en Debian 10:
~~~
apt install php7.3-fpm php7.3
~~~

#### Configuración
Con esto hemos instalado php 7.3 y php-fpm. Veamos primeros algunos ficheros de configuración de php:

Si nos fijamos en la configuración de php para php-fpm:
**/etc/php/7.3/fpm/conf.d**: Módulos instalados en esta configuración de php (enlaces simbólicos a /etc/php/7.3/mods-available).
**/etc/php/7.3/fpm/php-fpm.conf**: Configuración general de php-fpm.
**/etc/php/7.3/fpm/php.ini**: Configuración de php para este escenario.
**/etc/php/7.3/fpm/pool.d**: Directorio con distintos pool de configuración. Cada aplicación puede tener una configuración distinta (procesos distintos) de php-fpm. Sepuede tener, de esta forma, distintos procesos fpm php que responda con diferentes cosas. Con los ficheros anteriores no se indican cuantos procesos se va a realizar, y todos los procesos se han a resolver con el fichero www.conf. Pero con pool.d se pueden indicar cuantos procesos se va a utilizar para cada aplicación. 

Por defecto tenemos un pool cuya configuración la encontramos en **/etc/php/7.3/fpm/pool.d/www.conf**, en este fichero podemos configurar muchos parámetros, los más importantes son:
**[www]**: Es el nombre del pool, si tenemos varios, cada uno tiene que tener un nombre. Por defecto www es por defecto.
**user y grorup**: Usuario y grupo con el que se va ejecutar los procesos.
**listen**: Indica donde va a estar escuchando php-fpm en un servidor web (cual va a ser la comunicación entre Apache y fpm-php). Esto se realiza a través de un fichero, /run/php/php7.3-fpm.sock, por defecto, donde Apache2 escribe y fpm-php lee. Se indica el socket unix o el socket TCP donde van a escuchar los procesos:
Por defecto, escucha por un socket unix: **listen = /run/php/php7.3-fpm.sock** Esto se utiliza por defecto porque parece más rápido que la otra posibilidad, pero la otra forma se utiliza cuando fpm-php esté en otra máquina.
Otra posibilidad es que en vez de a través de un fichero, escuche por un puerto TCP. Si queremos que escuche por un socket TCP: listen = 127.0.0.1:9000
En el caso en que queramos que escuche en cualquier dirección: listen = 9000

> IMPORTANTE: si el servidor web (Apache2) y php-fpm se separan, se alojan en diferentes máquinas, tiene que tener compartidos el documentRoot.

Directivas de procesamiento y multiprocesos, gestión de procesos:
**pm**: Por defecto igual a dynamic (el número de procesos se crean y destruyen de forma dinámica). Otros valores: static o ondemand.
Otras directivas: pm.max_children (numero de procesos máx que se pueden crear), pm.start_servers, pm.min_spare_servers,...
**pm.status_path = /status**: No es necesaria, pero vamos a activar la URL de status para comprobar el estado del proceso.

Por último reiniciamos el servicio:
~~~
systemctl restart php7.3-fpm
~~~

#### Pruebas de funcionamiento
Suponemos que tenemos configurado por defecto, por lo tanto los procesos están escuchando en un socket UNIX:
~~~
     listen = /run/php/php7.3-fpm.sock
~~~

    Para enviar ficheros php a los procesos para su interpretación vamos a utilizar el programa cgi-fcgi:
~~~
     apt-get install libfcgi0ldbl
~~~
   
    Y a continuación accedemos a la URL /status, para ello:
~~~
     SCRIPT_NAME=/status SCRIPT_FILENAME=/status REQUEST_METHOD=GET cgi-fcgi -bind -connect /run/php/php7.3-fpm.sock 
    	
     Expires: Thu, 01 Jan 1970 00:00:00 GMT
     Cache-Control: no-cache, no-store, must-revalidate, max-age=0
     Content-type: text/plain;charset=UTF-8		

     pool:                 www
     process manager:      dynamic
     start time:           13/Nov/2017:19:32:50 +0000
     start since:          38
     accepted conn:        6
     listen queue:         0
     max listen queue:     0
     listen queue len:     0
     idle processes:       1
     active processes:     1
     total processes:      2
     max active processes: 1
     max children reached: 0
     slow requests:        0
~~~

    Si queremos ejecutar un fichero php, vamos a crear un directorio /var/www y vamos a guardar un fichero holamundo.php con el siguiente contenido:
~~~
     <?php echo "Hola Mundo!!!";?>
~~~

    A continuación vamos a indicar el directorio de trabajo en el fichero /etc/php/7.3/fpm/pool.d/www.conf:
~~~
     chroot = /var/www
~~~

    Inicializamos el servicio:
~~~
     systemctl restart php7.3-fpm
~~~

    Y podríamos ejecutar el fichero de la siguiente manera:
~~~
     SCRIPT_NAME=/holamundo.php SCRIPT_FILENAME=/holamundo.php REQUEST_METHOD=GET cgi-fcgi -bind -connect /run/php/php7.3-fpm.sock 
 	
     Content-type: text/html; charset=UTF-8

     Hola Mundo!!!		
~~~

    Si suponemos que hemos configurado php-fpm para que escuche en un socket TCP:
~~~
     listen = 127.0.0.1:9000
~~~

    Para realizar las pruebas que hemos probado anteriormente:
~~~
     SCRIPT_NAME=/status SCRIPT_FILENAME=/status REQUEST_METHOD=GET cgi-fcgi -bind -connect 127.0.0.1:9000

     SCRIPT_NAME=/holamundo.php SCRIPT_FILENAME=/holamundo.php REQUEST_METHOD=GET cgi-fcgi -bind -connect 127.0.0.1:9000
~~~

### Configuración de Apache2 con php-fpm

Necesito activar los siguientes módulos, porque se necesita activar el proxy inverso (en nginx no es necesario, porque este servidor web es un proxy inverso):
~~~
a2enmod proxy_fcgi setenvif
~~~

#### Activarlo para cada virtualhost

Podemos hacerlo de dos maneras:

- Si php-fpm está escuchando en un socket TCP, dentro del cada virtualHost se configura de forma diferente:
~~~
      ProxyPassMatch ^/(.*\.php)$ fcgi://127.0.0.1:9000/var/www/html/$1
~~~
- Si php-fpm está escuchando en un socket UNIX:
~~~
      ProxyPassMatch ^/(.*\.php)$ unix:/run/php/php7.3-fpm.sock|fcgi://127.0.0.1/var/www/html
~~~

Otra forma de hacerlo es la siguiente, para todos la misma configuración:
- Si php-fpm está escuchando en un socket TCP:
~~~
      <FilesMatch "\.php$">
      	SetHandler "proxy:fcgi://127.0.0.1:9000"
      </FilesMatch>
~~~
- Si php-fpm está escuchando en un socket UNIX:
~~~
      <FilesMatch "\.php$">
     	    	SetHandler "proxy:unix:/run/php/php7.3-fpm.sock|fcgi://127.0.0.1/"
      </FilesMatch>
~~~

#### Activarlo para todos los virtualhost

Tenemos a nuestra disposición un fichero de configuración php7.3-fpm en el directorio **/etc/apache2/conf-available**. Por defecto funciona cuando php-fpm está escuchando en un socket UNIX, si escucha por un socket TCP, hay que cambiar la línea:
~~~
SetHandler "proxy:unix:/run/php/php7.3-fpm.sock|fcgi://localhost"
~~~

por esta:
~~~
SetHandler "proxy:fcgi://127.0.0.1:9000"
~~~

Por último activamos la configuración:
~~~
a2enconf php7.3-fpm
~~~

#### Configuración de Nginx con php-fpm

En el virtualhost descomentamos las siguientes líneas:
~~~
location ~ \.php$ {
    include snippets/fastcgi-php.conf;

    # With php-fpm (or other unix sockets):
    #fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
    # With php-cgi (or other tcp sockets):
    #fastcgi_pass 127.0.0.1:9000;
}
~~~

Descomentando la opción si php-fpm está en un socket Unix o en un socket TCP.