Node1:
````
sudo apt update
sudo apt install mysql-server -y
````

````
cd /etc/mysql/mysql.conf.d
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
````

Este archivo debe tener:

````
[mysqld]

bind-address = 192.168.60.11
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
mysqlx-bind-address     = 127.0.0.1

key_buffer_size         = 16M

myisam-recover-options  = BACKUP

log_error = /var/log/mysql/error.log
max_binlog_size   = 100M
````

Luego, reiniciar el servicio de mysql:

````
sudo systemctl restart mysql
````

Conectar a mysql:

````
sudo mysql
````

Crear usuario para replica

````
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';

GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

FLUSH PRIVILEGES;
````

(Importante) Necesitamos cambiar una propiedad del usuario replicador para que tenga seguridad SSL y prevenir errores:

````
ALTER USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
````

Mostrar estado del maestro:

````
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |     1015 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
````

Memorizar o anotar el registro de las columnas "File" y "Position" para configurar los nodos esclavos

En los nodos esclavos:

````
sudo apt update
sudo apt install mysql-server -y
````
````
cd /etc/mysql/mysql.conf.d/
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
````

Node 2:

````
[mysqld]
bind-address = 192.168.60.12 
server-id = 2 
relay-log = /var/log/mysql/mysql-relay-bin.log
read_only = 1
super_read_only = 1
````

Node 3:

````
[mysqld]
bind-address = 192.168.60.13
server-id = 3
relay-log = /var/log/mysql/mysql-relay-bin.log
read_only = 1
super_read_only = 1
````

En cada nodo esclavo reiniciar el servicio de mysql

````
sudo mysql
````

(Importante) Cambiar MASTER_LOG_FILE y MASTER_LOG_POS; Con los valores que nos arrojó el nodo maestro antes

````
CHANGE MASTER TO
MASTER_HOST='192.168.60.11',
MASTER_USER='repl',
MASTER_PASSWORD='password',
MASTER_LOG_FILE='mysql-bin.xxxxxx',  
MASTER_LOG_POS=XXX;
````

Empezar la replica

````
START SLAVE;
SHOW SLAVE STATUS\G;
````

Si el valor de las variables "Slave_IO_Running" and "Slave_SQL_Running"  es igual a "Yes", mysql se implementó correctamente

Nosotros podemos probar el servicio:

Desde node1:

````
CREATE DATABASE test;
USE test;
CREATE TABLE prueba (id INT PRIMARY KEY, mensaje VARCHAR(50));
INSERT INTO prueba VALUES (1, 'Hola desde el maestro');
````

Desde node 2 and 3:

````
sudo mysql
````

Probar sincronizacion

````
mysql> SELECT * FROM test.prueba;
````

````
+----+-----------------------+
| id | mensaje               |
+----+-----------------------+
|  1 | Hola desde el maestro |
+----+-----------------------+
````

Ahora, nosotros instalaremos y configuraremos nginx en el nodo1:


Agregar este bloque de configuracion a /etc/nginx/nginx.conf

````
cd /etc/nginx/nginx.conf
sudo vim /etc/nginx/nginx.conf
````

Se debe agregar al archivo el siguiente bloque de configuracion:

````
stream {
    # Definir upstream para los esclavos (balanceo de carga)
    upstream mysql_slaves {
        least_conn;  # Distribuir conexiones según las menos activas
        server 192.168.60.12:3306;  # Node2 - MySQL Esclavo
        server 192.168.60.13:3306;  # Node3 - MySQL Esclavo
    }

    # Proxy para consultas de escritura (Maestro)
    server {
        listen 3307;
        proxy_pass 192.168.60.11:3306;  # Node1 - MySQL Maestro
        proxy_timeout 10s;
        proxy_connect_timeout 1s;
    }

    # Proxy para consultas de lectura (Esclavos)
    server {
        listen 3308;
        proxy_pass mysql_slaves;  # Balanceo de carga entre esclavos
        proxy_timeout 10s;
        proxy_connect_timeout 1s;
    }
}
````

Ahora reiniciamos nginx

````
sudo systemctl restart nginx
````

Para probarlo

El puerto 3308 es de los nodos esclavos (solo lectura)
El puerto 3307 es del nodo maestro (Escritura)

Realice estas pruebas en cualquier maquina virtual que tenga mysql instalado (node2, node3, sysbench)

Prueba de lectura:

Se redirige a un nodo esclavo mediante least_conn lo cual distribuye conexiones según los nodos esclavos menos activos

````
mysql -h 192.168.60.11 -P 3308 -u test -p -e "SELECT * FROM test.prueba;"
````

Prueba de escritura:

Prueba de que se puede hacer escritura al nodo 1:

`````
mysql -h 192.168.60.11 -P 3307 -u test -p -e "INSERT INTO test.prueba (id, mensaje) VALUES (7, 'PruebaEscritura2');"
`````

Prueba de que NO se puede hacer escritura a los nodos esclavos:

Node2:

````
mysql -h 192.168.60.11 -P 3308 -u test -p -e "INSERT INTO test.prueba (id, mensaje) VALUES (2, 'PruebaNodo2');"
````

Node3:

````
mysql -h 192.168.60.11 -P 3308 -u test -p -e "INSERT INTO test.prueba (id, mensaje) VALUES (3, 'PruebaNodo3');"
````

Pruebas de balanceo de carga con sysbench:

En la maquina virtual Sysbench instalar:

````
sudo apt update && sudo apt install -y sysbench
sysbench --version
````

En esta misma maquina realizar las pruebas:
Crear las base de datos de prueba "sbtest"

``````
    sysbench \
  --db-driver=mysql \
  --mysql-host=192.168.60.11 \
  --mysql-port=3306 \
  --mysql-user=test \
  --mysql-password=test \
  --mysql-db=sbtest \
  --tables=10 \
  --table-size=100000 \
  oltp_read_write prepare
``````

Prueba de escritura por el puerto asignado al nodo maestro (3307):

``````
sysbench \
  --db-driver=mysql \
  --mysql-host=192.168.60.11 \
  --mysql-port=3307 \
  --mysql-user=test \
  --mysql-password=test \
  --mysql-db=sbtest \
  --tables=10 \
  --table-size=100000 \
  --threads=10 \
  --time=60 \
  oltp_read_write run
``````

Prueba de lectura por el puerto asignado a los nodos esclavos (3308):

``````
sysbench \
  --db-driver=mysql \
  --mysql-host=192.168.60.11 \
  --mysql-port=3308 \
  --mysql-user=test \
  --mysql-password=test \
  --mysql-db=sbtest \
  --tables=10 \
  --table-size=100000 \
  --threads=10 \
  --time=60 \
  oltp_read_only run
``````

Eliminar bd de prueba:

``````
sysbench \
  --db-driver=mysql \
  --mysql-host=192.168.60.11 \
  --mysql-port=3306 \
  --mysql-user=test \
  --mysql-password=test \
  --mysql-db=sbtest \
  --tables=10 \
  --table-size=100000 \
  oltp_read_write cleanup
``````
