# Otros servicios que puede prestar el servidor {#otros_servicios_que_puede_prestar_el_servidor}

## Cuotas

Pueden emplearse cuotas para limitar el espacio en disco y la cantidad
de archivos que un usuario o un grupo puede tener.

Para usarlo debe: (1) especificar sistemas de archivos en los que se
usará cuota (2) especificar cuota por usuario o grupo y (3) activar
chequeo de cuota durante el arranque.

Indique los sistemas de archivos en los que usará cuota, en `/etc/fstab`
agregando como opción del sistema de archivos: `userquota` y/o
`groupquota`. Por ejemplo:

        /dev/wd0d /home ffs rw,nodev,nosuid,userquota 1 2

Después active el sistema de cuotas con:

        doas quotaon -a

Para especificar la cuota por usuario o por grupo emplee `edquota`, por
ejemplo:

        doas edquota pabram

que lo dejará en un editor en el que podrá cambiar las especificaciones
de la cuota en cada sistema de archivos con cuotas:

        Quotas for user pabram:
        /home: blocks in use: 1292980, limits (soft = 1000000, hard = 2000000)
            inodes in use: 142318, limits (soft = 0, hard = 0)

el límite blando podrá extenderse para cada usuario por un periodo de
tiempo (en este ejemplo), tras el cual se convertirá en un límite duro.
Límites en 0 indican que no hay restricción.

Puede replicar la configuración de cuotas en otros usuarios con:

        doas edquota -p pabram pedgar juamar margo

Después de especificar las cuotas de los usuarios puede verificar la
política de cuotas con:

        repquota -a

o chequear que se cumplan todas las cuotas con:

        quotacheck -a

Para activar sistema de cuotas y que las cuotas sean verificadas cada
vez que el sistema inicia agregue la siguiente línea al archivo
`/etc/rc.conf.local`:

        check_quotas=YES

### Referencias y lecturas recomendadas {#referencias-quota}

Las siguientes páginas del manual de OpenBSD: quota 1, edquota 8,
quotaon 8, quotaoff 8, quotacheck 8 y repquota 8. En el FAQ de OpenBSD
hay una sección sobre quotas:
<http://www.openbsd.org/faq/faq10.html#Quotas>


## Motor de bases de datos PostgreSQL {#postgresql}

PostgreSQL es un motor de bases de datos relacionales (RDBMS) que
verifica integridad referencial con gran funcionalidad como base de
datos, aunque un poco más lenta que otros motores. Su licencia es tipo
BSD. En esta sección describimos brevemente la instalación y uso en un
sistema adJ.

### Primera instalación del servidor {#primera-instalacion}

Este motor de bases de datos se instala con el archivo de ordenes
`/inst-adJ.sh` que en instalaciones tipicas de adJ basta ejecutar y
volver a ejecutar para actualizar o para volver a inicializar PostgreSQL
u otro paquete de esta distribución. En caso de actualizar este archivo
sacará respaldo de la información de la base de 2 formas (copiando
directorios de PostgreSQL y sacando un volcado de toda la base).

A continuación se dan detalles del proceso de instalación y uso de
PostgreSQL en caso de que requiera instalar por su cuenta o aprender más
sobre este motor de bases de datos.

Para emplearlo por primera vez instale el paquete `postgresql-server` (también 
es recomendable `postgresql-docs`).

Este paquete deja instrucciones específicas para inicializar la base de
datos, permitir conexiones de red e inicializar la base de datos cada
vez que arranque el sistema en:
`/usr/local/share/doc/postgresql/README.OpenBSD`. Los pasos que este
escrito describe son:

-   Opcional. El paquete de PostgreSQL disponible crea el usuario del
    sistema `_postgresql`, sin embargo si está actualizando o lo
    requiere puede crearlo con:

            doas useradd -c "Administrador de PostgreSQL" -g =uid -m -d /var/postgresql \
            -s /bin/sh -u 503 _postgresql 
            doas passwd _postgresql

-   A diferencia de versiones anteriores, este paquete ya no inicializa
    la base. Inicialicela con:

            doas mkdir -p /var/postgresql/data
            doas chown -R _postgresql:_postgresql /var/postgresql
            doas su - _postgresql
            initdb --auth=md5 -D /var/postgresql/data -U postgres --encoding=UTF8

    En adJ se emplea por defecto autenticación md5, que requiere
    suministrar clave cada vez que se carga la interfaz `psql` o abre
    conexiones. Si no desea autenticación md5, al inicializar la base
    con `initdb` emplee la opción `--auth=trust`. Una vez inicializada
    puede cambiar de un método a otro en el archivo
    `/var/postgresql/data/pg_hba.conf`

-   Altamente Recomendado. Agregue a `/etc/sysctl.conf`:

            kern.seminfo.semmni=256
            kern.seminfo.semmns=2048
            kern.shminfo.shmmax=50331648

-   Configurar servicio para que inicie en el arranque y se detenga al
    apagar. En `/etc/rc.conf.local` agregue:

            pg_ctl_flags="start \
                -D /var/postgresql/data -l /var/postgresql/logfile \
                -o '-D /var/postgresql/data'"

    y en el mismo archivo en la línea donde se define `pkg_scripts`
    agregue postgresql, así un computador donde estén configuradas las
    particiones cifradas, respaldo cifrado y los servicios httpd, cupsd
    dirá:

            pkg_scripts="cron montaencres montaencpos postgresql httpd cupsd"

Inicialmente el servidor queda configurado con un zocalo (socket) Unix 
(solo desde la misma máquina). Puede comprobar que está corriendo el servidor
(postmaster) con:

        pgrep post

y revisar el zócalo examinando la presencia del archivo
`/var/www/var/run/postgresql/.s.PGSQL.5432` (otra ubicación más
común pero fuera de la jaula chroot para servidores web es
`/tmp/.s.PGSQL.5432`).

Para emplear el protocolo TCP/IP para conexiones desde algunas máquinas
de su elección, edite `/var/postgresql/data/pg_hba.conf` y agregue por
ejemplo máquinas y usuarios que puedan hacer conexiones. También edite
`/var/postgresql/data/postgresql.conf` para que incluya líneas de
configuración como:

        max_connections = 100
        port = 5432

Para mejorar desempeño especialmente en sitios que atiendan bastantes
conexiones simultáneamente, consulte primero
`/usr/local/share/doc/postgresql/README.OpenBSD`.

En adJ por seguridad (e.g cuando ejecuta Apache con `chroot` en
`/var/www`) no se permiten conexiones TCP/IP y se emplea una ruta para
los zócalos diferente a la ruta por defecto (i.e `/tmp`), se trata de
`/var/www/var/run/postgresql`, que se define en el archivo de configuración de
PostgreSQL con:

        unix_socket_directories = '/var/www/var/run/postgresql'

Antes de reiniciar PostgreSQL asegúrese de crear el directorio:

        doas mkdir  /var/www/var/run/postgresql
        doas chmod a+w /var/www/var/run/postgresql
        doas chmod +t /var/www/var/run/postgresql

También tenga en cuenta que las diversas herramientas reciben como
parámetro adicional `-h ruta`. Por ejemplo si ejecuta Apache con
`chroot` en `/var/www/` puede tener configurado su directorio para
zócalos en `/var/www/var/run/postgresql`, en ese caso puede iniciar 
`psql` con la base `prueba` usando:

        psql -h /var/www/var/run/postgresql prueba

En paquetes anteriores al de adJ 4.1 el superusuario de la base
coincidía con el usuario del sistema `_postgresql`, desde 4.1 el
superusuario de la base es `postgres`, así que para realizar operaciones
debe agregarse la opción `-U postgres`. El instalador de adJ
seleccionará una clave con el programa `apg` y la dejará en el archivo
`/var/postgresql/.pgpass` en una línea de la forma:

        *:*:*:postgres:clave

De esta manera desde la cuenta `_postgresql` no necesitará dar la clave
del usuario `postgres`.

### Habilitando autenticación {#autenticacion}

Por defecto la forma de inicializar PostgreSQL no establecerá una clave
de administrador ni exigirá autenticación para cuentas que se conecten
desde la misma máquina. Sin embargo esto debe mejorarse si tiene varias
cuentas en el mismo servidor.

Una manera es en el momento de la inicialización de la base de datos con
las opciones `--auth` y `--pwfile` o `-W` de `initdb`, por ejemplo:

        su - _postgresql
        echo "MiClave" > clave.txt
        initdb -Upostgres --auth=md5 --pwfile=clave.txt -D/var/postgresql/data

que inicializará PostgreSQL con autenticación md5 y clave de
administrador `MiClave`

Otra posibilidad es cambiar la configuración después de haber
inicializado sin autenticación. Para esto cambie la clave del
administrador con[^aut.1]:

        psql -h /var/www/var/run/postgresql -U postgres template1 
        template1=# alter user postgres with password 'MiClave';

Después edite `/var/postgresql/data/pg_hba.conf` y cambie en las lineas
de acceso la palabra `trusted` por `md5`, por ejemplo:

        local   all         all                               md5
        host    all         all         127.0.0.1/32          md5
        host    all         all         ::1/128               md5

Detenga el servidor y vuelvalo a iniciar, notará que todo intento de
ingreso exige la clave.

El listado de bases de datos puede consultarse con:

        SELECT * FROM pg_database ;

y el listado de los usuarios con:

        SELECT username FROM pg_users;

Las claves de los diversos usuarios pueden cambiarse de forma análoga a
la presentada para `postgres`:

        ALTER ROLE &EUSUARIO; with UNENCRYPTED PASSWORD 'clave-plana';

Desde PostgreSQL 8.1 se emplea un esquema de roles que unifica los
conceptos de usuario y grupo. Además de exigir clave para cada ingreso,
cada rol de PostgreSQL que crea objetos puede modificar permisos para
restringir o dar acceso a otros roles. Por ejemplo para restringir el
acceso a una tabla `cuenta`:

        REVOKE ALL ON cuenta FROM PUBLIC;

Cuando un rol crea una base de datos, queda como dueño de todas las
tablas y en principio es el único que puede acceder a estas. Otro rol en
principio no podrá ni siquiera examinar los datos de las tablas:

        SELECT * FROM solicitud;
        ERROR:  permission denied for relation solicitud

Para dar permiso a otro rol puede usarse:

        GRANT ALL on solicitud TO &EUSUARIO2;;

### Creación de una base de datos {#creacion-base}

Para crear la base de datos `prueba` puede usar el superusuario con la
opción `-U postgres` o desde una cuenta que tenga permiso para crear
bases de datos:

        createdb -h /var/www/var/run/postgresql -U postgres prueba
        psql -h /var/www/var/run/postgresql -U postgres prueba

Desde la interfaz `psql`, pueden darse ordenes SQL y otros específicos
de PostgreSQL (ver [Uso de una base de datos](#uso-base)). En particular
el usuario `postgres` y desde cuentas con permiso para crear usuarios,
puede crear otros usuarios (globales para todas las bases de datos
manejadas por el servidor). Por ejemplo para crear un usuario normal sin
clave, desde `psql` ingresar:

        CREATE USER usejemplo

La orden `CREATE USER` presentado puede ir seguido de `CREATEUSER`
para crear un superusuario (sin restricción alguna), o `CREATEDB` para
crear un usuario que pueda crear bases de datos o `PASSWORD 'clave'`
para crear un usuario con una clave (emplea autenticación configurada).
Desde la línea de ordenes puede crearse un usuario con:

        createuser -h /var/www/var/run/postgresql -U postgres  usejemplo

Para eliminar un usuario desde `psql` se usa:

        DROP USER usejemplo;

y para eliminarlo desde línea de ordenes:

        dropuser -h /var/www/var/run/postgresql -U postgres usejemplo

Puede ejecutarse un script SQL (`crea.sql`) desde la línea de ordenes a
un base de datos con

        psql -h /var/www/var/run/postgresql -d test -U ejusuario --password -f crea.sql

### Uso de una base de datos {#uso-base}

Puede emplear `psql`, la interfaz texto que acepta ordenes SQL y que se
distribuye con PostgreSQL. Para esto, entre a una base (digamos `b1908`)
como un usuario (digamos `u1908`) con:

        psql -h /var/www/var/run/postgresql  -U u1908 -d b1908

En esta interfaz puede dar ordenes SQL y algunas ordenes internos que
puede listar con `\h`. Algunos ejemplos de operaciones útiles son:

`\dt`

:   para ver tablas disponibles.

`\d usuarios`

:   Describe la estructura de la tabla `usuarios`

`SELECT victim_nombre,victim_apellido FROM victimas WHERE victim_edad<=12;`

:   Que muestre los nombres de niños de 12 años o menos listados en la
    tabla `victimas`

`\h update`

:   Da ayuda sobre la orden `update` (que permite actualizar registros
    de una tabla.)

Es recomendable que los usuarios del sistema que también son usuarios de
PostgreSQL creen el archivo `~/.pgpass` donde puede almacenarse la clave
que usa en PostgreSQL ---de forma que las diversas herramientas no la
solicitaran--- con una línea de la forma:

        *:*:*:usuario:clave

### Cotejación en PostgreSQL {#postgresql-cotejacion}

Desde adJ 5.2 se incluye un porte de PostgreSQL (la versión incluida en
adJ &VER-ADJ; es &p-postgresql-server;) que soporta cotejaciones de acuerdo
al locale, y por defecto se crean cotejaciones para todos los países de
habla hispana, con nombres de la forma `es_CO_UTF_8`. Donde `CO`
representa Colombia, pero se pueden emplear otras abreviaturas de países
como se usan en Internet (estándar ISO-3166-1): ES CO PE VE EC GT CU BO
HN PY SV CR PA GQ MX AR CH DO NI UY PR.

Puede examinar las cotejaciones disponibles ingresando al interprete
`psql` y ejecutando:

        SELECT * FROM pg_collation;

Puede crear una cotejación (digamos `esp`) para un locale soportado por
el sistema operativo y con la misma codificación de su base de datos
(digamos `es_CO.ISO8859-1`) con:

        CREATE COLLATION esp (LOCALE='es_CO.ISO8859-1');

Una vez creada puede realizar operaciones empleándola por ejemplo:

        SELECT 'Á' < 'B' COLLATE "esp";
        ...
        SELECT nombre FROM clase ORDER BY nombre COLLATE "esp";

la primera sentencia dará:

        ?column? 
        ----------
        t
        (1 row)

Puede crear columnas de tablas especificando el tipo de cotejación por
defecto para operaciones con esa columna.

        CREATE TABLE ejc (
        nombre VARCHAR(100) COLLATE "esp"
        );

o cambiar las existentes:

        ALTER TABLE clase ALTER nombre TYPE VARCHAR(500) COLLATE "esp";

También podrá renombrar cotejaciones que haya creado con
`ALTER COLLATION esp RENAME TO es_CO_ISO8859_1;`, así como borrarlas con
`DROP COLLATION esp;`. Puede consultar más en
<http://www.postgresql.org/docs/9.1/static/collation.html>.

### Copias de respaldo {#respaldo-postgresql}

Para sacar una copia de respaldo de todas las base de datos manejadas
con PostgreSQL (y suponiendo que el zócalo está en 
`/var/www/var/run/postgresql`):
ingrese a la cuenta del administrador:

        doas su - _postgresql
        pg_dumpall -U postgres -h /var/www/var/run/postgresql/ \
            --inserts --attribute-inserts > /respaldos/pgdump.sql

Puede restablecer una copia con

        psql -U postgres -h /var/www/var/run/postgresql/ \
            -f /respaldos/pgdump.sql template1


### Base PostgreSQL remota {#base-postgresql-remota}

PostgreSQL permite conexiones remotas y cifradas, así que la aplicación 
puede estar en un servidor y la base de datos en otra.

Para la operación cifrada se requiere un certificado para el servidor y un 
certificado para cada usuario de la base de datos que se emplee en
conexiones remotas.  Los certificados para los clientes deben tener el CN
con el nombre del usuario que hará la conexión.   
Por lo mismo en lugar de comprar certificados para esto es más práctico 
tener una autoridad certificadora que pueda firmarlos.

#### Autoridad certificadora SSL

Las operaciones con SSL depende en cliente y en servidor de la librería 
LibreSSL (en otros sistemas OpenSSL). Esta incluye el programa
```openssl``` para hacer varias operaciones, incluyendo operaciones
de una autoridad certificadora. 

Un certificado SSL siempre va con una llave privada (el certificado es la 
llave pública).   

El proceso para crear un certificado es:

1. Crear la llave privada para el certificado (extensión .key)
2. Generar el certificado (llave pública) pero sin firma (extensión .csr)
3. Firmar el certificado con una autoridad certificadora y generar el certificado 
4. Usar el certificado firmado junto con la llave privada para realizar conexiones (el certificado firmado se compartirá, mientras que la llave privada no)

Los archivos intermedios pueden examinarse así:

* Solicitudes: ```openssl req -noout -text -in client.csr```
* Llaves: ```openssl rsa -check -in client.key ```
* Certificados: ```openssl x509 -noout -text -in client.crt```

La autoridad certificadora no es más que un certificado autofirmado que 
se configura y usa consistentemente como autoridad certificadora.

#### Configuración de servidor

En el servidor deben quedar certificados del servidor en ```/var/postgresql/data```:

* root.crt Autoridad certificadora (igual a server.crt)
* root.crl Lista de revocación
* server.crt Certificado del servidor
* server.key Llave privada del servidor

Por cada cliente que se va a conectar debe configurarse en 
```/var/postgresql/data/pg_hba.conf``` el/los usuarios que
se conectarán.  Contrario a lo especificado en la documentación de 
PostgreSQL en casos de SSL en ese archivo sólo nos han funcionado 
líneas de la forma: 

        hostssl all usuario 192.168.100.11/32 cert clientcert=1

Es decir conexión SSL exigiendo certificado al cliente y que la autenticación 
sea por certificado. Lo cual también exige que el certificado del cliente
tenga el CN igual al usuario.

#### configuración de cada cliente

Para cada usuario debe hacerse un certificado que se ubica en 
cada comptuador cliente en ```~/.postgresql/{usuario.crt, usuario.key}```  
donde usuario debe correponder al usuario en la base de datos y 
al CN del certificado.

Desde el servidor puede generar y firmar certificado para cliente por
10 años (cambie ```usuario``` por el usuario PostgreSQL dueño de la 
bae de datos y que usara desde los clientes para conectarse, si
prefiere un lapsode tiemo diferente especifiquelo en días después
de la opción ```-days```):

        doas su - 
        cd /var/postgresql/data
        openssl genrsa -des3 -out usuario.key 1024
        openssl rsa -in usuario.key -out usuario.key
        openssl req -new -key usuario.key -out usuario.csr -subj '/C=CO/ST=Cundinamarca/L=Bogota/O=Pasos de Jesus/CN=usuario'
        openssl x509 -req -days 3650 -in usuario.csr -CA root.crt -CAkey server.key -out usuario.crt -CAcreateserial

A continuación copie el certificado generado (```usuario.crt```) y la 
llave privada (```usuario.key```) al computador cliente donde se usará:

        scp usuario.key usuario.crt mius@192.168.100.11:~/.postgresql/

En el servidor edite el archivo ```/var/postgresql/data/pg_hba.conf``` 
y asegurese de agregar una línea para el usuario y el computador cliente:

        hostssl all usuario 192.168.100.11/32 cert clientcert=1

Reinicie PostgreSQL.

        doas sh /etc/rc.d/postgresql -d restart

Desde el cliente ejecute:

        doas chmod 0600 /home/usis/.postgresql/usuario.key

y pruebe la conexión asegurando que se usa el certificado 
del usuario respectivo:

        PGSSLCERT=/home/usis/.postgresql/usuario.crt \
        PGSSLKEY=/home/usis/.postgresql/usuario.key  \
        psql -h192.168.100.21 -Uusuario usuario

Configure la aplicación para que en cada arranque o uso establezca:

        PGSSLCERT=/home/usis/.postgresql/usuario.crt 
        PGSSLKEY=/home/usis/.postgresql/usuario.key

##### Clientes en PHP

Copie las llaves dentro de la jaula chroot, haga que el dueño sea
www:www e incluya en alguna fuente
usada antes de las conexiones a base de datos (por ejemplo intente en
index.php):

        putenv('PGSSLCERT=/ojs/certs/ojs.crt');
        putenv('PGSSLKEY=/ojs/certs/ojs.key');

##### Clientes en Ruby on Rails

En ```config/database.yml``` debe verse algo como:

        username: usuario
        host: 192.168.100.21
        sslmode: "require"

y al hacer operaciones que usen base de datos (rails dbconsole, iniciar unicorn, etc) asegurese de ejecutarlas en un ambiente donde se definan bien las variables PGSSLCERT y PGSSLKEY, por ejemplo:

        PGSSLCERT=/home/usis/.postgresql/usuario.crt \
        PGSSLKEY=/home/usis/.postgresql/usuario.key \
        rails dbconsole

### Referencias y lecturas recomendadas {#referencias-postgresql}

-   Documentación del paquete postgresql (README.OpenBSD, INSTALL).

-   Documentación disponible en el paquete `postgresql-doc` (ver
    postgresql-doc) y en <http://www.postgresql.org/docs>.

-   Páginas del manual de Unix: psql 1

-   SSL Certificates For PostgreSQL :
    <https://www.howtoforge.com/postgresql-ssl-certificates>

[^aut.1]: Note que de esta forma puede cambiar la clave de otros 
    usuarios de PostgreSQL.


## MariaDB {#mariadb}

A partir de OpenBSD/adJ 5.7 MariaDB remplaza a MySQL. Según
<https://es.wikipedia.org/wiki/MariaDB> MariaDB fue iniciada por el
fundador de MySQL después de que Oracle compró Sun y MySQL, pues
consideraba que Oracle había hecho la compra para reducir competencia de
sus bases de datos.

Debe instalar los paquetes &p-mariadb-client; y &p-mariadb-server;. Aunque
el nombre de los paquetes cambia las ordenes para operarla siguen
siendo los mismos. Tras instalar el servidor debe ejecutar
`mysql_install_db`.

Inicialice el directorio donde estarán las bases de datos con

        doas /usr/local/bin/mysql_install_db

Para aumentar el límite de archivos que el usuario `_mysql` de clase
`mysql` puede abrir agregue a `/etc/login.conf`:

        mysql:\
            :openfiles-cur=2048:\
            :openfiles-max=4096:\
            :tc=servicio:

tenga en cuenta no dejar espacios al final de cada línea y que desde la
segunda línea cada una comiencen con el caracter tabulador. A
continuación regenere el archivo binario `/etc/login.conf.db` con

        cd /etc
        doas cap_mkdb /etc/login.conf 

Después agregue `mysqld` a `pkg_scripts` en `/etc/rc.conf.local` por ejemplo con:
	doas rcctl enable mysqld

A continuación lance el servidor con:

        doas sh /etc/rc.d/mysqld start

Los errores quedarán en `/var/mysql/host.err`.

Después puede establecer una clave para el usuario `root` de MariaDB
cuando ingresa desde `localhost` con:

        /usr/local/bin/mysqladmin -u root  password 'nueva-clave' 
        /usr/local/bin/mysqladmin -u root -pnueva-clave -h &ESERV; password 'nueva-clave'
        

Después puede iniciar una sesión, crear bases de datos, crear usuarios y
otorgarles privilegios.

Para apagar el servidor mysql:

        mysqladmin -u root -p shutdown
        

Si desea usar mysql con php, instale además de los paquetes básicos de
php (`php-core-v` y `php-mysql-v`)

### Uso básico {#uso-mariadb}

        mysql -u root -p 

puede crear la base de datos `datos`, y un usuario `erfurt` que la pueda
administrar (i.e con todos los privilegios excepto GRANT) y con clave
`vsewf` usando:

        CREATE DATABASE datos;
        GRANT ALL PRIVILEGES ON datos.* TO erfurt@localhost IDENTIFIED BY 'vsewf';

Algunas operaciones usuales del administrador son:

        SHOW DATABASES;

que muestra todas las bases disponibles.

        USE base1;

que permite usar la base base1.

        SHOW TABLES;

que muestra todas las tablas de la base activa.

        DESCRIBE tabla;
        SHOW CREATE TABLE tabla;

que presentan estructura de la tabla.

### Cambio de la clave de administrador {#clave-mysql}

Si olvida la clave de root después de haberla establecido puede
cambiarla entrando a la cuenta de administrador:

-   Detenga el servidor.

-   Inicie el servidor con
    `/usr/local/libexec/mysqld --user=root --skip-grant-tables`

-   Ejecute:

            # mysql
            mysql> USE mysql
            mysql> UPDATE USER SET PASSWORD=password('miclave') WHERE user='root';
            mysql> FLUSH PRIVILEGES;
            mysql> EXIT

-   Vuelva a apagar el servidor y reinicielo con:
    `/usr/local/bin/mysqld_safe &`

### Recuperación y backups {#recuperacion-backups}

MariaDB mantiene bases de datos en directorios y las tablas en archivos.
No es recomendable que modifique tales archivos, al menos no, mientras
el servidor esté activo.

Para sacar una copia de respaldo de todas las bases de datos con:

        mysqldump --force -p --all-databases > /respaldomysql/dump-1nov2007.sql

y posteriormente restaurarla con:

        mysql < /respaldomysql/dump-1nov2007.sql

### MariaDB y servidor web con chroot {#chroot-mysql}

Puede emplear aplicaciones para nginx o Apache en modo `chroot` que usen 
bases de datos MariaDB de tres formas: (1) Conectando la aplicación web a la 
base de datos mediante un puerto TCP/IP donde responda MariaDB, 
(2) poniendo el zócalo de MariaDB en un directorio dentro de la 
jaula del servidor web o (3) Corriendo MariaDB dentro de la 
jaula `chroot` (ver
<http://structio.sourceforge.net/guias/servidor_OpenBSD/mysql.html#mysql-chroot>).

A continuación documentamos como ubicar el zócalo de MariaDB dentro de la 
jaula del servidor web (/var/www/) que nos parece un método seguro y 
fácil e implementar.
 
Una vez instale `mariadb-server` cree el directorio en el cual ubicará el
zócalo, digamos:

```sh
        doas mkdir -p /var/www/var/run/mysql/
        doas chown _mysql:_mysql /var/www/var/run/mysql/
        doas chmod a+w /var/www/var/run/mysql/
        doas chmod +t /var/www/var/run/mysql/
```
y después inicie MariaDB indicando la ruta del zócalo con la opción
`--socket`, por ejemplo para que el cambio se efectúe en cada inicio,
edite `/etc/rc.conf.local` para agregar:

```sh
        mysqld_flags="--socket=/var/www/var/run/mysql/mysql.sock"
```

y reinicie el servicio con:

```sh
	doas rcctl -d restart mysqld
```

Puede verificar que el zócalo queda bien ubicado con:
```sh
	$ ls -l /var/www/var/run/mysql/
```
que debe responde con algo como
```sh
	srwxrwxrwx  1 _mysql  _mysql  0 Jul 18 21:41 mysql.sock     
```

Así una aplicación PHP que corran en el mismo servidor podrían realizar
una conexión con:

```php
        $dbhost  = "localhost";
        $dbuname = "miusuario";
        $dbpass  = "miclave";
        mysql_connect($dbhost, $dbuname, $dbpass);
```

Tenga en cuenta también que otros binarios de MariaDB también requerirán
la opción `--socket=/var/www/var/run/mysql/mysql.sock` al ejecutarse por
ejemplo:

        mysqldump --socket=/var/www/var/run/mysql/mysql.sock  \
		-p --all-databases
        

### Lecturas recomendadas {#lecutras-mysql}

Referencias:

-   Una explicación de algo de la instalación y el uso de MySQL en
    OpenBSD:
    <http://www.sancho2k.net/filemgmt_data/files/mysql_notes.html>

-   La documentación de MariaDB :
    <https://mariadb.com/kb/en/mariadb/documentation/>

-   Ayuda para cambiar clave de root en sistemas Linux:
    <http://www.netadmintools.com/art90.html>


## Servidor ldapd {#ldapd}

LDAP (Lightweight Directory Access Protocol) es un protocolo para
mantener e intercambiar información almacenada en directorios (i.e bases
de datos especiales), su versión 3 se define en los RFC 2251, 2256,
2829, 2830 y 3377.

Un uso típico de LDAP es mantener en un servidor información de los
usuarios de una organización para permitir su autenticación en otros
servicios (e.g nombres, apellidos, dirección, teléfono, login, clave).

OpenBSD incluye (desde OpenBSD 4.8) un servidor para LDAP versión 3,
`ldapd`. No incluye cliente para LDAP pero desde la línea de ordenes
puede emplearse el paquete `openldap-client` o como interfaz web
`phpldapadmin`[^lda.1].

### Instalación de ldapd {#instalacion-ldapd}

No necesita instalar paquetes para la operación como servidor.

La configuración que se presenta emplea LDAPS para conexiones en la red
local, empleando un certificados cuyas llaves pública y privada debe
copiarse a `/etc/ldap/certs` y ejecutar:

        cd /etc/ldap/certs
        chown _ldapd:_ldapd *
        chmod 0640 /etc/ldap/certs/*key
        chmod 0644 /etc/ldap/certs/*crt 

Para configurar el servidor, verifique que exista el usuario `_ldapd` y
el grupo `_ldapd` y edite `/etc/ldapd.conf`:

        schema "/etc/ldap/core.schema"                                                  
        schema "/etc/ldap/inetorgperson.schema"                                         
        schema "/etc/ldap/nis.schema"                                                   
         
        lan_if = "re1"
        
        listen on $lan_if ldaps certificate www.pasosdeJesus.org
        listen on lo0 secure
        listen on "/var/run/ldapi"
        
        namespace "dc=www,dc=pasosdeJesus,dc=org" {
                rootdn          "cn=root,dc=www,dc=pasosdeJesus,dc=org"
                rootpw          "secret"
                index           sn
                index           givenName
                index           cn
                index           mail
                index           objectClass
                index           sn
                fsync           on
        }                                                                               

Recuerde que la clave del directorio debe ser mejor que la presentada
(i.e remplace `secret` por una buena clave). En lugar de poner la clave
plana también es posible poner la cadena generada con:

        doas slappasswd -v -u -h {CRYPT} -s secret

que en el caso de la clave '`secret`' es '`{CRYPT}uPUCy906TIu/k`'

La configuración por defecto emplea `/var/db/ldap` como directorio para
mantener las bases de datos y mantiene una por cada espacio de nombres
(namespace). Las conexiones no cifradas por defecto operan en el puerto
389 y deben autenticarse con SASL (a menos que tengan la opción `secure`
para permitir autenticación plana) y las que empleen certificado iran
cifradas en el puerto 636.

Cada vez que modifique el archivo de configuración del servidor, puede
verificarlo con:

        doas ldapd -n

Para iniciar el servidor LDAP en modo de depuración para ver posibles
errores:

        doas ldadpd -dv

Tras verificar el funcionamiento, para que en cada arranque se inicie el
servidor puede agregar a `/etc/rc.conf.local`:

        ldapd_flags=""
        pkg_scripts = "ldapd"

E iniciar el servicio con `/etc/rc.d/ldapd start` y detenerlo con
`/etc/rc.d/ldadp stop`

Es muy recomendable que agregue el esquema LDAP de Courier, de esta
forma tomada de {3}:

-   Descarguelo y renombrelo:

            doas ftp -o /etc/ldap/courier.schema \
            http://courier.cvs.sourceforge.net/viewvc/courier/libs/authlib/authldap.schema

-   Edite /etc/ldap/courier.schema y quite comentario a las líneas:

            attributetype ( 1.3.6.1.4.1.10018.1.1.14 NAME 'mailhost'                        
                    DESC 'Host to which incoming POP/IMAP connections should be proxied'
                    EQUALITY caseIgnoreIA5Match                                        
                    SYNTAX 1.3.6.1.4.1.1466.115.121.1.26{256} )  

-   Reinicie ldapd

### Pruebas Iniciales con openldap-client {#pruebas-openldap}

Instale el paquete con:

        doas pkg_add openldap-client

Verifique localmente que el servidor no cifrado corre con:

        ldapsearch -x -b 'dc=www,dc=pasosdeJesus,dc=org' '(objectclass=*)'

Respecto al servidor cifrado puede analizar la conexión SSL con:

        openssl s_client -connect 192.168.2.1:465

Puede verificar sus certificados contra la entidad que los expide
siguiendo instrucciones de {4}.

Puede deshabilitar la verificación de certificados de ldapsearch
poniendo en `/etc/openldap/ldap.conf`:

        TLS_REQCERT never

Para hacer pruebas desde otro computador, tenga en cuenta que en OpenBSD
ldapsearch utiliza openssl mientras que por ejemplo en Ubuntu emplea
GNUTLS, por esto eventualmente puede requerir descargar certificado de
autoridad certificadora (digamos gd.pem) y ejecutar:

        doas cat /etc/ssl/certs/gd.pem >> /etc/ssl/certs/ca-certificates.crt 

Además de modificar `/etc/ldap/ldap.conf` para agregar

        TLS_CACERT      /etc/ssl/certs/ca-certificates.crt

### Adición de datos iniciales {#datos-iniciales-ldapd}

Una vez esté corriendo `ldapd` deberá iniciar un directorio para su
organización y los usuarios que se autenticarán. Puede agregar estos
datos con el programa `ldapadd` que hace parte de openldap-client.
Programa que recibe datos en formato ldif, por ejemplo leidos de un
archivo. Un primer archivo con datos de la organización puede ser
`org.ldif` y contener:

        dn:     dc=www,dc=pasosdeJesus,dc=org
        objectClass:    dcObject
        objectClass:    organization
        o:      Pasos de Jesús
        dc:     correo

        dn: cn=admin,dc=correo,dc=pasosdeJesus,dc=org
        objectClass: organizationalRole
        cn: admin

        dn:ou=gente, dc=correo,dc=pasosdeJesus,dc=org
        objectClass:    top
        objectClass:    organizationalUnit
        ou:     gente

        dn:ou=grupos,dc=correo,dc=pasosdeJesus,dc=org
        objectClass:    top
        objectClass:    organizationalUnit
        ou:     grupos

        dn:ou=sendmail,dc=www,dc=pasosdeJesus,dc=org
        ou: sendmail
        objectClass: top
        objectClass: organizationalUnit
        userPassword: sendmail

Nota: Al agregar información verifique no dejar espacios en blanco al
final de cada línea. Se pueden agregar `org.ldif` con:

        ldapadd -x -D "cn=admin,dc=www,dc=pasosdeJesus,dc=org" -W \
        -h www.pasosdeJesus.org -f org.ldif

Además de poder revisar los mensajes que `slapd` genere al ejecutarse en
modo de depuración, podrá consultar los datos ingresados al directorio
con:

        ldapsearch -x -b 'dc=www,dc=pasosdeJesus,dc=org' '(objectclass=*)'

### Instalación y configuración de `phpldapadmin` {#phpldapadmin}

Instale el paquete y siga las instrucciones que de (por ejemplo activar
`php-ldap`):

        doas pkg_add phpldapadmin

También debe asegurar que pueden emplearse los dispositivos de
generación de números aleatorios en la jaula chroot de Apache (esto lo
hace por defecto el instalador de adJ 5.5). Para esto verifique que en
`/etc/fstab` al montar la partición `/var` este permitiendo dispositivos
(que no este la opción `nodev`) y ejecute:

        cd /var/www
        doas mkdir -p dev
        cd dev
        /dev/MAKEDEV arandom

### Referencias y lecturas recomendadas {#referencias-ldapd}

-   Las siguientes páginas man: ldapd 8. ldapctl 8. ldapd.conf 5.

-   <http://dhobsd.pasosdejesus.org/index.php?id=ldapd>.

-   <http://www.cyberciti.biz/faq/test-ssl-certificates-diagnosis-ssl-certificate/>.

-   <http://www.tumfatig.net/20120817/monitoring-openbsds-ldap-daemon/>.

-   <http://openbsd.7691.n7.nabble.com/dev-random-as-chrooted-named-s-entropy-source-current-td64344.html>.

[^lda.1]: Si emplea un adJ 5.2 y planea conectarse desde clientes digamos en
    Ubuntu reciente requerirá el parche descrito en
    <http://openbsd.7691.n7.nabble.com/ldapd-and-quot-The-Diffie-Hellman-prime-sent-by-the-server-is-not-acceptable-quot-td59635.html>


## Autoridad certificadora interna {#autoridad_certificadora}

Los servicios en red (en particular ldapd, ver [xref](#ldapd)  y 
postgresql remoto, ver [xref](#postgresql)) 
requieren cada vez más comunicaciones cifradas y suelen emplear 
SSL o TLS que requieren certificados públicos firmados por 
autoridades certificadoras.  

Cada vez los programas, librerías y lenguajes están verificando con más 
insistencia que los certificados sean efectivamente firmados por 
autoridades certificadoras.  

En genral las autoridades certificadoras cobran por emitir firmas para 
certificados.  Sin embargo <http://letsencrypt.org> es una autoridad 
certificadora que expide certificados gratuitos para sitios públicos 
pero no para sitios en redes internas por cuanto el proceso de 
expedición de certificados 
requiere resolver por DNS desde sus servidores el dominio para el cual 
se está creando el certificado.  Además sus certificados son de 
3 meses por cuanto deben renovarse cada 3 meses.  

Esto hace necesario que cada organización que requiere servicios 
cifrados con SSL o TLS en su red interna (como PostgreSQL remoto o 
LDAP) cuente con su propia autoridad certificadora interna

###  Conceptos

Las operaciones con SSL dependen en cliente y en servidor de la 
librería LibreSSL (en otros sistemas OpenSSL). Esta incluye el 
programa openssl para hacer varias operaciones, incluyendo operaciones 
de una autoridad certificadora.

Un certificado SSL siempre se asocia a una llave privada (el 
certificado es la llave pública).

El proceso para crear un certificado es:

1. Crear la llave privada para el certificado (extensión .key)
2. Generar el certificado (llave pública) pero sin firma (extensión .csr)
3. Firmar el certificado con una autoridad certificadora y generar el certificado
4. Usar el certificado firmado junto con la llave privada para realizar conexiones (el certificado firmado se compartirá, mientras que la llave privada no)

Los archivos intermedios pueden examinarse así:

- Solicitudes: `openssl req -noout -text -in client.csr`
- Llaves: `openssl rsa -check -in client.key`
- Certificados: `openssl x509 -noout -text -in client.crt`

La autoridad certificadora no es más que un certificado autofirmado 
que se configura y se usa consistentemente como autoridad certificadora.

### Configuración de servidor

Supongamos que ubicamos en `/var/postgresql/data` los 
archivos de la autoridad certificadora:

- `root.crt` Autoridad certificadora (igual a server.crt)
- `root.crl` Lista de revocación
- `server.crt` Certificado del servidor
- `server.key` Llave privada del servidor

Se pueden generar así (como se explica en 
<https://www.howtoforge.com/postgresql-ssl-certificates>):


        openssl genrsa -des3 -out server.key 1024
        openssl rsa -in server.key -out server.key
        chmod 400 server.key
        chown postgres.postgres server.key
        openssl req -new -key server.key -days 3650 -out server.crt -x509 -subj '/C=CO/ST=Bogota/L=MiOng/O=MiOng/CN=miong.org.co/emailAddress=info@miong.org'
 
### Generación de un par de certificados

LDAP requiere que el CN del certificado corresponda al nombre del 
computador en la red interna.

Los certificados para clientes de PostgreSQL requieren que el CN del 
Certificado corresponda al usuario en la base de datos.

En el computador que hará la conexión (en este ejemplo 
`apbd2.miong.org.co`) ejecute:

        mkdir ~/ssl
        cd ~/ssl
        openssl genrsa -des3 -out apbd2.miong.org.co.key 1024
        openssl rsa -in apbd2.miong.org.co.key -out apbd2.miong.org.co.key

De una clave temporal y borrela con

        openssl rsa -in apbd2.miong.org.co.key -out apbd2.miong.org.co.key

Cree la solicitud de certificado con:

        openssl req -new -key apbd2.miong.org.co.key -out apbd2.miong.org.co.csr -subj '/C=CO/ST=Cundinamarca/L=Bogota/O=CINEP/CN=apbd2.miong.org.co'

Copie la solicitud `apbd2.miong.org.co.csr` al servidor 
apbd1.miong.org.co y dejela en el directorio `/var/postgresql/data`

Y allí ejecute:

        doas su - 
        cd /var/postgresql/data
        openssl x509 -req -days 3650 -in apbd2.miong.org.co.csr -CA root.crt -CAkey server.key -out apbd2.miong.org.co.crt -CAcreateserial

A continuación copie el certificado generado 
(`apbd2.miong.org.co.crt`)  al computador cliente donde se usará:

        scp apbd2.miong.org.co.crt apbd2.miong.org.co:~/ssl/

### Uso de los certificados

#### Caso PostgreSQL

En el servidor edite el archivo `/var/postgresql/data/pg_hba.conf` y 
asegurese de agregar una línea para el usuario y el computador 
cliente:
        hostssl all usuario 192.168.100.11/32 cert clientcert=1

Reinicie PostgreSQL.

        doas sh /etc/rc.d/postgresql -d restart

Desde el cliente ejecute:

        doas chmod 0600 /home/usis/.postgresql/usuario.key

y pruebe la conexión asegurando que se usa el certificado del usuario 
respectivo:

        PGSSLCERT=/home/usis/.postgresql/usuario.crt \
        PGSSLKEY=/home/usis/.postgresql/usuario.key \
        psql -h192.168.100.21 -Uusuario usuario

Configure la aplicación para que en cada arranque o uso establezca:

        PGSSLCERT=/home/usis/.postgresql/usuario.crt 
        PGSSLKEY=/home/usis/.postgresql/usuario.key

#### Caso LDAPD

Ubique el ceritifcado y llave en `/etc/ldad/certs/` del servidor 
donde corre ldapd:

        doas cp apbd2.miong.org.co.{key,crt} /etc/ldap/certs/

Configure `/etc/ldapd.conf`

        listen on $if1 tls certificate apbd2.miong.org.co

En los computadores que realicen conexiones al LDAP asegurese de 
agregar la llave de la entidad certificadora 
`/var/postgresql/data/root.crt`, es decir en el servidor apbd1 ejecute:

        cd /var/postgresql/data
        openssl x509 -noout -text -in root.crt > root-paracerts

Copie el archivo `root-paracerts` en el cliente y agregue ese archivo 
al final de `/etc/ssl/cert.pem`

### Referencias

- <https://www.howtoforge.com/postgresql-ssl-certificates>

