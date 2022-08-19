# PostgreSQL. Backup + Репликация

## Задание

настроить hot_standby репликацию с использованием слотов
настроить правильное резервное копирование
Для сдачи работы присылаем ссылку на репозиторий, в котором должны обязательно быть
Vagranfile (2 машины)
плейбук Ansible
конфигурационные файлы postgresql.conf, pg_hba.conf и recovery.conf,
конфиг barman, либо скрипт резервного копирования.
Команда "vagrant up" должна поднимать машины с настроенной репликацией и резервным копированием.
Рекомендуется в README.md файл вложить результаты (текст или скриншоты) проверки работы репликации и резервного копирования.

## Решение

Стенд разворачивается командой ```vagrant up```

Проверяю статус репликации на мастере и создаю тестовую базу.

```
postgres=# select * from pg_stat_replication;
 pid  | usesysid | usename  | application_name |  client_addr   | client_hostname | client_port |        backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | writ
e_lag | flush_lag | replay_lag | sync_priority | sync_state |          reply_time
------+----------+----------+------------------+----------------+-----------------+-------------+------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----
------+-----------+------------+---------------+------------+-------------------------------
 5092 |    16384 | repluser | walreceiver      | 192.168.11.151 |                 |       57564 | 2022-08-18 11:08:52.78769+00 |              | streaming | 0/3000AD0 | 0/3000AD0 | 0/3000AD0 | 0/3000AD0  |
      |           |            |             0 | async      | 2022-08-18 11:59:05.451727+00
(1 row)
postgres=# CREATE DATABASE repltest ENCODING='UTF8';
CREATE DATABASE
```

Проверяю статус WAL на реплике и наличие новой базы.

```
postgres=# select * from pg_stat_wal_receiver;
 pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time        | slot_name  |  sender_host   | sender_port |
                                   conninfo

------+-----------+-------------------+-------------------+-------------+-------------+--------------+-------------------------------+-------------------------------+----------------+-------------------------------+------------+----------------+-------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 4524 | streaming | 0/3000000         |                 1 | 0/3000AD0   | 0/3000AD0   |            1 | 2022-08-18 11:59:42.848464+00 | 2022-08-18 11:59:42.846747+00 | 0/3000AD0      | 2022-08-18 11:28:10.753517+00 | pgstandby1 | 192.168.11.150 |        5432 | user=repluser password=******** channel_binding=prefer dbname=replication host=192.168.11.150 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
(1 row)
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 repltest  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)
```

Проверяю наличие бэкапа и пробую восстановиться.

```
t=# drop database postgres;
DROP DATABASE
t=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 t         | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)


[root@master data]# systemctl restart postgresql-14.service
[root@master data]# su postgres
bash-4.2$ psql
psql (14.5)
Type "help" for help.
[root@backup ~]# barman recover pg 20220819T084502 /var/lib/pgsql/14/data/ --remote-ssh-comman "ssh postgres@192.168.11.150"
postgres@192.168.11.150's password:
postgres@192.168.11.150's password:
Starting remote restore for server pg using backup 20220819T084502
Destination directory: /var/lib/pgsql/14/data/
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
```
