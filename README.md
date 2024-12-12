# Репликация MySQL

## Задание

В материалах приложена ссылка на дамп базы `bet.sql`. Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы: `bookmaker`, `competition`, `market`, `odds`, `outcome`. Настроить **GTID** репликацию.

## Реализация

Задание сделано на **rockylinux/9** версии **v9.5-20241118.0**. Для автоматизации процесса написан **Ansible Playbook** [playbook.yml](playbook.yml) который последовательно запускает следующие роли:

- **mysql** - разбита на несколько файлов задач:
  - [install.yml](roles/mysql/tasks/install.yml) - устанавливает **Percona Server 8.4**;
  - [firewalld.yml](roles/mysql/tasks/firewalld.yml) - разрешает сервис **mysql** в настройках **firewalld**;
  - [config.yml](roles/mysql/tasks/config.yml.yml) - обновляет конфигурационные файлы **Percona Server**, шаблоны файлов лежат в директории [templates](roles/mysql/templates/);
  - [init.yml](roles/mysql/tasks/init.yml) - меняет пароль **root** на значение указанное в переменной **mysql_root_password** (по умолчанию берётся значение из файл `passwords/mysql_root_password.txt`, который при необходимости создаётся автоматически) и запускает сервер;
  - [users.yml](roles/mysql/tasks/users.yml) - создаёт дополнительных пользователей, указанных в переменной **mysql_users** (в частности **repl** на **master**) и даёт им необходимые права (**GRANT**).
- **mysql_import** - импортирует базу данных из файлов, указанных в переменной **mysql_import_databases**, в частности база данных **bet** импортируется из файла [bet.sql](roles/mysql_import/files/bet.sql) на **master** и [bet-slave.sql](roles/mysql_import/files/bet-slave.sql) на **slave**.
- **mysql_export** - экспортирует базы данных в файлы, указанные в переменной **mysql_export_databases**, в частности база данных **bet** на **master** экспортируется в файл [bet-slave.sql](roles/mysql_import/files/bet-slave.sql);
- **mysql_repl** - настраивает и запускает репликацию, берёт необходимые параметры из переменных **mysql_repl_primary_host**, **mysql_repl_primary_user**, **mysql_repl_primary_password**.

Указанные роли используют переменные, которые определены для каждого из узлов в файлах [host_vars/master.yml](host_vars/master.yml) и [host_vars/slave.yml](host_vars/slave.yml). Также используются **defaults** переменные ролей. Пароли генерятся автоматически и сохраняются в директории `passwords`:

- `mysql_root_password.txt` - пароль **root@localhost** для обоих серверов;
- `mysql_repl_password.txt` - пароль пользователя **repl** на **master**;
- `mysql_repl_salt.txt` - соль для хеширования пароля пользователя **repl**.

## Запуск

Необходимо скачать **VagrantBox** для **rockylinux/9** версии **v9.5-20241118.0** и добавить его в **Vagrant** под именем **rockylinux/9/v9.5-20241118.0**. Сделать это можно командами:

```shell
curl -OL https://dl.rockylinux.org/pub/rocky/9.5/images/x86_64/Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box
vagrant box add Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box --name "rockylinux/9/v9.5-20241118.0"
rm Rocky-9-Vagrant-Vbox-9.5-20241118.0.x86_64.box
```

Для того, чтобы **vagrant 2.3.7** работал с **VirtualBox 7.1.0** необходимо добавить эту версию в **driver_map** в файле **/usr/share/vagrant/gems/gems/vagrant-2.3.7/plugins/providers/virtualbox/driver/meta.rb**:

```ruby
          driver_map   = {
            "4.0" => Version_4_0,
            "4.1" => Version_4_1,
            "4.2" => Version_4_2,
            "4.3" => Version_4_3,
            "5.0" => Version_5_0,
            "5.1" => Version_5_1,
            "5.2" => Version_5_2,
            "6.0" => Version_6_0,
            "6.1" => Version_6_1,
            "7.0" => Version_7_0,
            "7.1" => Version_7_0,
          }
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.1.4_SUSE r165100**
- **Ansible 2.18.1**
- **Python 3.11.11**
- **Jinja2 3.1.4**

## Проверка

Проверим идентификаторы серверов и то, что **GTID** включён на обоих серверах:

```text
❯ vagrant ssh master -c "mysql -u root -p -e \"SHOW VARIABLES WHERE Variable_name in ('server_id', 'gtid_mode');\""
Enter password:
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
| server_id     | 1     |
+---------------+-------+

❯ vagrant ssh slave -c "mysql -u root -p -e \"SHOW VARIABLES WHERE Variable_name in ('server_id', 'gtid_mode');\""
Enter password:
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
| server_id     | 2     |
+---------------+-------+
```

Проверим, что база данных **bet** есть на обоих серверах и на реплике нет лишних таблиц:

```text
❯ vagrant ssh master -c 'mysql -u root -p -e "USE bet; SHOW TABLES;"'
Enter password:
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+

❯ vagrant ssh slave -c 'mysql -u root -p -e "USE bet; SHOW TABLES;"'
Enter password:
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
```

Проверим, что репликация настроена на **slave**:

```text
❯ vagrant ssh slave -c 'mysql -u root -p -e "SHOW REPLICA STATUS\G"'
Enter password:
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 192.168.11.150
                  Source_User: repl
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000003
          Read_Source_Log_Pos: 93156
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 422
        Relay_Source_Log_File: mysql-bin.000003
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Source_Log_Pos: 93156
              Relay_Log_Space: 633
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Source_SSL_Allowed: Yes
           Source_SSL_CA_File:
           Source_SSL_CA_Path:
              Source_SSL_Cert:
            Source_SSL_Cipher:
               Source_SSL_Key:
        Seconds_Behind_Source: 0
Source_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Source_Server_Id: 1
                  Source_UUID: 573b580b-b887-11ef-8d8f-00163e0e40a1
             Source_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Source_Retry_Count: 10
                  Source_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Source_SSL_Crl:
           Source_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set: 57381fef-b887-11ef-931d-00163e0e40a1:1-2,
573b580b-b887-11ef-8d8f-00163e0e40a1:1-40
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Source_TLS_Version:
       Source_public_key_path:
        Get_Source_public_key: 0
            Network_Namespace:
```

Проверим содержимое таблицы **bet.bookmaker** на обоих узлах:

```text
❯ vagrant ssh master -c 'mysql -u root -p -e "SELECT * FROM bet.bookmaker;"'
Enter password:
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+

❯ vagrant ssh slave -c 'mysql -u root -p -e "SELECT * FROM bet.bookmaker;"'
Enter password:
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
```

Обновим таблицу **bookmaker** на **master** и проверим работу репликации:

```text
❯ vagrant ssh master -c "mysql -u root -p -e \"INSERT INTO bet.bookmaker (id,bookmaker_name) VALUES(1,'1xbet');\""
Enter password:

❯ vagrant ssh master -c 'mysql -u root -p -e "SELECT * FROM bet.bookmaker;"'
Enter password:
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+

❯ vagrant ssh slave -c 'mysql -u root -p -e "SELECT * FROM bet.bookmaker;"'
Enter password:
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
```

Как видно, репликация работает. Однако дополнительно проверим **binlog** на **slave**:

```text
❯ vagrant ssh slave -c 'sudo mysqlbinlog -v /var/lib/mysql/mysql-bin.000003'
# The proper term is pseudo_replica_mode, but we use this compatibility alias
# to make the statement usable on server versions 8.0.24 and older.
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#241212 12:48:19 server id 2  end_log_pos 127 CRC32 0xde28f554  Start: binlog v 4, server v 8.4.2-2 created 241212 12:48:19 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
k9taZw8CAAAAewAAAH8AAAABAAQAOC40LjItMgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAACT21pnEwANAAgAAAAABAAEAAAAYwAEGggAAAAAAAACAAAACgoKKioAEjQA
CigAAAFU9Sje
'/*!*/;
# at 127
#241212 12:48:19 server id 2  end_log_pos 198 CRC32 0x71700150  Previous-GTIDs
# 57381fef-b887-11ef-931d-00163e0e40a1:1
# at 198
#241212 12:48:24 server id 2  end_log_pos 275 CRC32 0x4e40b223  GTID    last_committed=0        sequence_number=1       rbr_only=no     original_committed_timestamp=1734007704113351        immediate_commit_timestamp=1734007704113351     transaction_length=184
# original_commit_timestamp=1734007704113351 (2024-12-12 12:48:24.113351 UTC)
# immediate_commit_timestamp=1734007704113351 (2024-12-12 12:48:24.113351 UTC)
/*!80001 SET @@session.original_commit_timestamp=1734007704113351*//*!*/;
/*!80014 SET @@session.original_server_version=80402*//*!*/;
/*!80014 SET @@session.immediate_server_version=80402*//*!*/;
SET @@SESSION.GTID_NEXT= '57381fef-b887-11ef-931d-00163e0e40a1:2'/*!*/;
# at 275
#241212 12:48:24 server id 2  end_log_pos 382 CRC32 0x1d6869d8  Query   thread_id=10    exec_time=0     error_code=0    Xid = 12
SET TIMESTAMP=1734007704/*!*/;
SET @@session.pseudo_thread_id=10/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1168113696/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=45,@@session.collation_connection=45,@@session.collation_server=255/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80011 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
/*!80016 SET @@session.default_table_encryption=0*//*!*/;
CREATE DATABASE `bet`
/*!*/;
# at 382
#241212 13:42:39 server id 1  end_log_pos 468 CRC32 0x334e7c08  GTID    last_committed=1        sequence_number=2       rbr_only=yes    original_committed_timestamp=1734010959063815        immediate_commit_timestamp=1734010959076637     transaction_length=293
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1734010959063815 (2024-12-12 13:42:39.063815 UTC)
# immediate_commit_timestamp=1734010959076637 (2024-12-12 13:42:39.076637 UTC)
/*!80001 SET @@session.original_commit_timestamp=1734010959063815*//*!*/;
/*!80014 SET @@session.original_server_version=80402*//*!*/;
/*!80014 SET @@session.immediate_server_version=80402*//*!*/;
SET @@SESSION.GTID_NEXT= '573b580b-b887-11ef-8d8f-00163e0e40a1:41'/*!*/;
# at 468
#241212 13:42:39 server id 1  end_log_pos 537 CRC32 0xabd5df3e  Query   thread_id=16    exec_time=0     error_code=0
SET TIMESTAMP=1734010959/*!*/;
SET @@session.sql_mode=1168637984/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=255,@@session.collation_connection=255,@@session.collation_server=255/*!*/;
BEGIN
/*!*/;
# at 537
#241212 13:42:39 server id 1  end_log_pos 597 CRC32 0x0cd9bc2a  Table_map: `bet`.`bookmaker` mapped to number 215
# has_generated_invisible_primary_key=0
# at 597
#241212 13:42:39 server id 1  end_log_pos 644 CRC32 0x5ffe0d24  Write_rows: table id 215 flags: STMT_END_F

BINLOG '
T+haZxMBAAAAPAAAAFUCAAAAANcAAAAAAAEAA2JldAAJYm9va21ha2VyAAIDDwL9AgIBAQACASEq
vNkM
T+haZx4BAAAALwAAAIQCAAAAANcAAAAAAAEAAgAC/wABAAAABQAxeGJldCQN/l8=
'/*!*/;
### INSERT INTO `bet`.`bookmaker`
### SET
###   @1=1
###   @2='1xbet'
# at 644
#241212 13:42:39 server id 1  end_log_pos 675 CRC32 0x96fdad54  Xid = 179
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

Видно, что вставленная строка реплицировалась на **slave**:

```text
### INSERT INTO `bet`.`bookmaker`
### SET
###   @1=1
###   @2='1xbet'
```
