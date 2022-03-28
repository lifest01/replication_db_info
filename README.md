# Настройка репликации
## Шаг 1. Настройка Мастер сервера
#### Настройки my.cnf:
```sh
[client]
port=3306
socket=/tmp/mysql.sock

[mysqld]
port=3306
socket=/var/lib/mysql/mysql.sock
server-id = 11
log_bin = /var/lib/mysql/mysql-bin.log
binlog_do_db = scoring
bind-address = 0.0.0.0
```
Перезагружаем MySQL:
```sh
/etc/init.d/mysql restart
# или
systemctl restart mysqld.service
```
## Шаг 2. Права на репликацию
Создаем пользователя и даем ему права
-`REPLICATION SLAVE` - привилегия позволяющая подключиться к серверу т запросить обновлённые на мастере данные;
-`REPLICATION CLIENT` - привилегия, позволяющая использовать статистику:
1. SHOW MASTER STATUS
2. SHOW SLAVE STATUS
3. SHOW BINARY LOGS
```sh
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave_user'@'<указываем ip адресс slave servera>' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```
Далее блокируем все таблицы в нашей базе данных:
```sh
USE scoring;
FLUSH TABLES WITH READ LOCK;
```
Проверяем статус Мастер-сервера:
```sh
SHOW MASTER STATUS;
```
Мы увидим что-то похожее на:
```sh
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000001 |      107 | scoring      |                  |
+------------------+----------+--------------+------------------+
1 row in set (0.00 sec)
```
## Шаг 3. Дамп базы
Теперь необходимо сделать дамп базы данных:
    `--master-data` - включить в дамп информацию о бинарном логе мастер хоста;
    `-R` - включить в дамп процедуры и функции.
```sh
mysqldump --master-data -R -u root -p scoring > scoring_dumb.sql
```
Разблокируем таблицы в консоли mysql:
```sh
USE scoring;
UNLOCK TABLES;
```
## Шаг 4. Создание базы на слейве
```sh 
CREATE DATABASE scoring;
```
После этого загружаем дамп (из bash):
```sh 
mysql -u root -p scoring < scoring_dumb.sql
```
## Шаг 5. Настройка Слейва
В настройках my.cnf на Слейве указываем следующие параметры:

`server-id` - идентификатор сервера, должен быть уникален. Лучше не использовать 1. Это единственный обязательный параметр;
`log_bin` - путь к бинарному логу. Оптимально указывать по аналогии с мастером;
`log_slave_updates` - включает запись реляционных событий в собственный журнал на подчинённом сервере
`binlog_do_db` - позволяет перечислить отдельные базы, для которых будет использоваться реплика.Если не инициализирована, то реплицируются все.
```sh
[mysqld]
server-id = 22
log_bin = /var/lib/mysql/mysql-bin.log
binlog_do_db = scoring
bind-address    = 0.0.0.0
#port = 3306
socket=/var/lib/mysql/mysql.sock
log_bin = /var/log/mysql/mysql-bin.log
relay_log = mysql-relay-bin
```
## Шаг 6. Запуск Слейва
Для запуска slave-сервера необходимо:
1. Указать параметры соединения (master-data).
2. Запустить репликацию.

```sh
CHANGE MASTER TO MASTER_HOST='<ip адрес мастер сервера>', MASTER_USER='slave_user', MASTER_PASSWORD='password',
MASTER_LOG_FILE = '', MASTER_LOG_POS = ;
```
Значения `MASTER_LOG_FILE` и `MASTER_LOG_POS` есть в файле дампа.

Запуск репликации выполняется следующей командой:
```sh
START SLAVE;
```
### Статус репликации
Проверить работу репликации на Слейве можно запросом:
```sh
SHOW SLAVE STATUS\G
```

### Траблшутинг

#### Last_IO_Error: error connecting to master...
Если **SHOW SLAVE STATUS** выводим примерно следующую ошибку:
```sh
*************************** 1. row ***************************
               Slave_IO_State: Connecting to master
                  Master_Host: 10.1.0.11
                  Master_User: slave_user
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 419
               Relay_Log_File: mysql-relay-bin.000005
                Relay_Log_Pos: 529
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Connecting
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 419
              Relay_Log_Space: 1281
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 2003
                Last_IO_Error: error connecting to master 'slave_user@10.1.0.11:3306' - retry-time: 60  retries: 86400  message: Can't connect to MySQL server on '10.1.0.11' (113)
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 11

```
то у slave-сервера отсутствует возможность соединения с master-сервером. Причины:
  - некорректные авторизационные данные пользователя репликации;
  - закрыт порт MySQL для исходящих соединений на slave-сервере;
  - закрыт порт MySQL для входящих соединений на master-сервере.

Добавим правило на slave-сервере
```sh
iptables -I OUTPUT -p tcp -m tcp --dport 3306 -j ACCEPT
```

Добавим правило на master-сервере:

```sh
iptables -I INPUT -p tcp -m tcp --dport 3306 -j ACCEPT
```
