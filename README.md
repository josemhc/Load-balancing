Node1:
``

sudo apt update
sudo apt install mysql-server -y

``

``

cd /etc/mysql/mysql.conf.d/mysqld.cnf

``
This file has:

[mysqld]

bind-address = 192.168.60.11
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
mysqlx-bind-address     = 127.0.0.1

key_buffer_size         = 16M

myisam-recover-options  = BACKUP

log_error = /var/log/mysql/error.log
max_binlog_size   = 100M

Then, restart the service mysql:
``

sudo systemctl restart mysql

``
connect to mysql:
``

sudo mysql

``
``

CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
``

We need to change something to have secure SSL and not have errors:
``

ALTER USER 'repl'@'192.168.60.%' IDENTIFIED WITH mysql_native_password BY 'password';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
``

``

mysql> SHOW MASTER STATUS;
``

+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |     1015 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+

Save file and position to other nodes

On the others nodes:
``

sudo apt update
sudo apt install mysql-server -y

``
``

cd /etc/mysql/mysql.conf.d/mysqld.cnf

``
Node 2:

bind-address = 192.168.60.12 
server-id = 2 
relay-log = /var/log/mysql/mysql-relay-bin.log

Node 3:

bind-address = 192.168.60.13
server-id = 3
relay-log = /var/log/mysql/mysql-relay-bin.log
``
sudo mysql

``

``

CHANGE MASTER TO
MASTER_HOST='192.168.60.11',
MASTER_USER='repl',
MASTER_PASSWORD='password',
MASTER_LOG_FILE='mysql-bin.xxxxxx',  
MASTER_LOG_POS=XXX;

``

Remember to change MASTER_LOG_FILE and MASTER_LOG_POS=XXX; to value's master (Node1)

``

START SLAVE;
SHOW SLAVE STATUS\G;

``

If there is not error and the value of the variables "Slave_IO_Running" and "Slave_SQL_Running" 
are "Yes", mysql was implemented successfully

We can test the service with:

From node1:

``

CREATE DATABASE test;
USE test;
CREATE TABLE prueba (id INT PRIMARY KEY, mensaje VARCHAR(50));
INSERT INTO prueba VALUES (1, 'Hola desde el maestro');

``

From node 2 and 3:

``

sudo mysql

``
``

mysql> SELECT * FROM test.prueba;

``
+----+-----------------------+
| id | mensaje               |
+----+-----------------------+
|  1 | Hola desde el maestro |
+----+-----------------------+

Then, we will install and configure nginx on node1:


Create new config file to proxy on /etc/nginx/sites-available/default

``

cd /etc/nginx/sites-available/
sudo nano /etc/nginx/sites-available/mysql_load_balancer

``

this file must have:

upstream mysql_slaves {
    least_conn;
    server 192.168.60.12:3306;  # Node2 - MySQL esclavo
    server 192.168.60.13:3306;  # Node3 - MySQL esclavo
}

server {
    listen 3307;

    location /write {
        proxy_pass http://192.168.60.11:3306;  # Node1 - MySQL master
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }

    location /read {
        proxy_pass http://mysql_slaves;  # Balanceo de carga a esclavos
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
    }
}


Now, we will create a symbolic link and verify that everything is ok 

``

sudo ln -s /etc/nginx/sites-available/mysql_load_balancer /etc/nginx/sites-enabled/
sudo nginx -t

``

Restart nginx

``

sudo systemctl restart nginx

``

