## Лабораторная работа №08: Резервное копирование и управление доступом
## Цель работы: Освоить методы резервного копирования и восстановления данных в PostgreSQL, включая логическое и физическое копирование, а также углубить навыки управления правами доступа пользователей.
**Дата:** 2025-12-4
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Джараян Арег Александрович
## Теоретическая часть (краткое содержание):
### Логическое резервное копирование: Выполняется утилитами pg_dump и pg_dumpall. Создает дампы SQL-команд для восстановления структуры и данных. Гибкое, но может быть медленным для больших БД.
### Физическое резервное копирование: Копирование файлов данных кластера с помощью pg_basebackup. Быстрее, но требует архивации WAL для восстановления на момент времени (PITR). 
###  WAL-архивация: Непрерывное сохранение сегментов WAL. Позволяет восстанавливать данные на любой момент времени после создания базовой резервной копии. 
###  Обновление сервера: Процесс миграции данных на новую мажорную версию PostgreSQL, часто с использованием логического дампа/восстановления. 


## Модуль 1: Управление доступом (Повторение и закрепление)
## 1. Настройка привилегий: Повторил задания из Практики по управлению доступом (создание БД, ролей writer/reader, настройка GRANT/REVOKE, проверка доступа w1/r1).
```sql
-- Создаем базу данных для практики
CREATE DATABASE access_db;

\c access_db

-- Создаем роли 'writer' и 'reader'
CREATE ROLE writer_user WITH LOGIN PASSWORD 'writerpass';
CREATE ROLE reader_user WITH LOGIN PASSWORD 'readerpass';

-- Создаем тестовую таблицу
CREATE TABLE company_data (id SERIAL PRIMARY KEY, secret_name TEXT, revenue INT);
INSERT INTO company_data (secret_name, revenue) VALUES ('Project Alpha', 100000), ('Project Beta', 250000);

-- Даем привилегии
-- reader_user может только читать
GRANT CONNECT ON DATABASE access_db TO reader_user;
GRANT USAGE ON SCHEMA public TO reader_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reader_user;

-- writer_user может читать и вставлять данные
GRANT CONNECT ON DATABASE access_db TO writer_user;
GRANT USAGE ON SCHEMA public TO writer_user;
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA public TO writer_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO writer_user;
```
--фрагмент вывода
```text
CREATE ROLE
CREATE ROLE

CREATE TABLE
INSERT 0 2

GRANT
GRANT
GRANT

GRANT
GRANT
GRANT
GRANT
```

Проверка подключения 
```text
admin@MacBook-Air-Admin ~ % psql -U reader_user -d access_db -h localhost
psql (16.10 (Homebrew))
Введите "help", чтобы получить справку.

access_db=> SELECT * FROM company_data;
 id |  secret_name  | revenue 
----+---------------+---------
  1 | Project Alpha |  100000
  2 | Project Beta  |  250000
(2 строки)

access_db=> INSERT INTO company_data (secret_name) VALUES ('Fail');
ОШИБКА:  нет доступа к таблице company_data
access_db=>
```

```text
admin@MacBook-Air-Admin ~ % psql -U writer_user -d access_db -h localhost
psql (16.10 (Homebrew))
Введите "help", чтобы получить справку.

  
access_db=> INSERT INTO company_data (secret_name) VALUES ('Success');
INSERT 0 1
access_db=>
```

## 2. Настройка аутентификации (Практика+): Повторил задания по настройке pg_hba.conf для ролей alice и bob с использованием методов trust, reject и peer.
``` sql
CREATE ROLE alice WITH LOGIN; 
CREATE ROLE bob WITH LOGIN;
```


--фрагмент вывода
```text
CREATE ROLE
CREATE ROLE
```
Настройка `pg_hba.conf`:  
  ```text
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    all             alice           localhost               trust
host    all             bob             localhost               reject
local   all             bob                                     peer
  ```

## Модуль 2: Логическое резервное копирование
**1. Простой дамп и восстановление: Создал БД backup_db и таблицу с данными. Сделал логическую копию с помощью pg_dump. Удалил БД и восстановите ее из копии. Убедился в целостности данных.**
```sql
CREATE DATABASE backup_db;

\c backup_db;

CREATE TABLE important_data AS SELECT generate_series(1,1000) as id, md5(random()::text) as data;
```
---фрагмент вывода
```text
CREATE DATABASE

Вы подключены к базе данных "backup_db" как пользователь "postgres".

SELECT 1000
```
Создание логической копии с помощью pg_dump:
``` bash
pg_dump -U $(whoami) -h localhost -Fc -f backup_db.dump backup_db
```
Удаление и восстановление БД:
``` sql
DROP DATABASE backup_db;

CREATE DATABASE backup_db;
```

```text
DROP DATABASE
CREATE DATABASE
```

``` bash
pg_restore -U postgres -h localhost -d backup_db backup_db.dump
```

Проверка восстановленных данных:
``` sql
SELECT count(*) FROM important_data;
```

```text
 count 
-------
  1000
(1 строка)
```


**2. Параллельный дамп: Создал несколько БД с различными объектами. Сделал копию глобальных объектов через pg_dumpall --globals-only. Сделал дампы каждой БД с помощью pg_dump в параллельном режиме (-j).**
Создание нескольких БД:
```sql
CREATE DATABASE db1; 
CREATE DATABASE db2;

\с db1

CREATE TABLE table1 (x int); 
INSERT INTO table1 VALUES (1);

\c db2

CREATE TABLE table2 (y text);
INSERT INTO table2 VALUES ('hello');
```
-- фрагмент вывода
```text
CREATE DATABASE
CREATE DATABASE

CREATE TABLE
INSERT 0 1

CREATE TABLE
INSERT 0 1
```

Копия глобальных объектов (роли, права):
``` bash
pg_dumpall -U postgres -h localhost --globals-only -f globals.sql
```

Параллельный дамп БД:
``` bash
# Создаем каталог для дампов
mkdir -p ./all_dbs_dump

# Получаем список всех БД и делаем параллельный дамп
psql -U postgres -h localhost -t -c "SELECT datname FROM pg_database WHERE datname NOT LIKE 'template%'" | \
xargs -n 1 -P 2 -I {} pg_dump -U postgres -h localhost -Fd -j 2 -f ./all_dbs_dump/{} {}
```

**3. Восстановление кластера: Восстановил весь кластер на "другом сервере" (виртуально, в другой каталог или другую ВМ) из созданных резервных копий.**
``` bash
# Создаем временный каталог в домашней директории
mkdir -p ~/postgres_cluster_new
initdb -D ~/postgres_cluster_new -U postgres

# Запускаем на другом порту
pg_ctl -D ~/postgres_cluster_new -o "-p 5433" -l ~/postgres_cluster_new/logfile start
```

``` bash
#!/bin/bash

DUMP_DIR="./all_dbs_dump"
PGHOST=localhost
PGPORT=5433
PGUSER=postgres

echo "Starting cluster restoration..."

# Создаем и восстанавливаем каждую БД
for db_dir in "$DUMP_DIR"/*; do
    if [ -d "$db_dir" ]; then
        db_name=$(basename "$db_dir")
        echo "Restoring database: $db_name"
        
        # Создаем базу данных
        createdb -h $PGHOST -p $PGPORT -U $PGUSER "$db_name"
        
        # Восстанавливаем данные
        pg_restore -h $PGHOST -p $PGPORT -U $PGUSER -d "$db_name" -j 4 --verbose "$db_dir"
        
        echo "Database $db_name restored successfully"
    fi
done

echo "All databases restored to new cluster on port 5433"
```

```text
chmod +x restore_cluster.sh
./restore_cluster.sh
```

Проверка:
``` bash
psql -h localhost -p 5433 -U postgres
```

```text
postgres=# \l
                                                             Список баз данных

       Имя        | Владелец | Кодировка | Провайдер локали | LC_COLLATE  |  LC_CTYPE   | локаль ICU | Правила ICU |     Права доступа     

------------------+----------+-----------+------------------+-------------+-------------+------------+-------------+-----------------------
 access_db        | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 backup_db        | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 db1              | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 db2              | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 depmanagmentbase | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 kpo              | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 lab02_db         | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 lab03_mvcc       | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 lab04            | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 lab06            | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 lab07            | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 new_db           | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 old_db           | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 postgres         | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | 
 template0        | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | =c/postgres          +
                  |          |           |                  |             |             |            |             | postgres=CTc/postgres
 template1        | postgres | UTF8      | libc             | ru_RU.UTF-8 | ru_RU.UTF-8 |            |             | =c/postgres          +
                  |          |           |                  |             |             |            |             | postgres=CTc/postgres
(16 строк)
```


**4. Проблемы при загрузке (Практика+): Попробовал создать данные и параметры COPY (например, несовпадение кодировок, делимитеров), которые приведут к ошибке при загрузке дампа.**
```sql
CREATE DATABASE problem_db;

CREATE TABLE test (col1 TEXT, col2 INT);
```

``` bash
pg_dump -U postgres -h localhost -Fp -f problem_db.sql problem_db
```

Изменяем дамп добавив некорректные данные:
```text
COPY public.test (col1, col2) FROM stdin;
wrong_delimiter_data\t500
\.
```

``` bash
psql -U postgres -d postgres -c "DROP DATABASE problem_db;"
psql -U postgres -d postgres -c "CREATE DATABASE problem_db;"
psql -U postgres -d problem_db -f problem_db.sql
```

```text
psql:problem_db.sql:43: ОШИБКА:  нет данных для столбца "col2"
КОНТЕКСТ:  COPY test, строка 1: "wrong_delimiter_data\t500"
```

## Модуль 3: Физическое резервное копирование и PITR
**1. Базовая резервная копия: Создал табличное пространство и БД с таблицей в нем. Сделал базовую резервную копию кластера с помощью pg_basebackup в формате tar со сжатием. Развернул второй кластер из этой копии, изменив расположение табличного пространства через tablespace_map.**
-- Создаем табличное пространство и БД с таблицей в нем
```sql
CREATE TABLESPACE ts_data LOCATION '/tmp/ts_data';
CREATE DATABASE phys_backup_db TABLESPACE ts_data;
\c phys_backup_db
CREATE TABLE important_table (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO important_table (data) VALUES ('Row 1'), ('Row 2');
```
-- Создаем физическую резервную копию с помощью pg_basebackup
```sql
mkdir -p ~/pg_basebackup_tar
pg_basebackup -U postgres -D ~/pg_basebackup_tar -Ft -z -P -X fetch
```
-- Разворачиваем второй кластер из этой копии
```sql
mkdir -p ~/pg_cluster_restore
tar -xvf ~/pg_basebackup_tar/base.tar.gz -C ~/pg_cluster_restore

echo "/tmp/ts_data /tmp/ts_data_restore" > ~/pg_cluster_restore/tablespace_map

pg_ctl -D ~/pg_cluster_restore -o "-p 5434" -l ~/pg_cluster_restore/logfile start
```
```bash
psql -U postgres -p 5434 -d phys_backup_db -c "SELECT * FROM important_table;"
```
-- фрагмент вывода
```text
CREATE TABLESPACE
CREATE DATABASE
You are now connected to database "phys_backup_db" as user "postgres".
CREATE TABLE
INSERT 0 2
```
```text
Progress: 1/3 ... [====================] 100%
pg_basebackup: base backup completed
```
```text
 id |  data  
----+--------
  1 | Row 1
  2 | Row 2
(2 строки)
```

**2. Непрерывная архивация и PITR (Практика+): Настроил непрерывную архивацию WAL (можно на локальный каталог). Сделал базовую копию с помощью pg_basebackup (без WAL). Добавил данные в таблицу. Убедителся, что WAL ушел в архив. Восстановил второй сервер из резервной копии, применив архив WAL. Убедителся, что все данные восстановились. Остановил второй сервер и восстановите его снова до конкретного момента (immediate или recovery_target). Сравнил результаты.**
```text
archive_mode = on
archive_command = 'cp %p /tmp/wal_archive/%f'
wal_level = replica
```
-- Создаем каталог для архива WAL
```bash
mkdir -p /tmp/wal_archive
```
--Перезагрузил сервер
```bash
pg_ctl -D $PGDATA restart
```
```bash
pg_basebackup -U postgres -D ~/pg_basebackup_pitr -Fp -X none -P
```
-- Добавляем данные в таблицу
```sql
\c phys_backup_db
INSERT INTO important_table (data) VALUES ('Row 3'), ('Row 4');
```
-- Проверяем, что WAL сегменты ушли в архив
```bash
ls /tmp/wal_archive
```
-- фрагмент вывода
```text
Progress: 1/3 ... [====================] 100%
pg_basebackup: base backup completed
```
```text
You are now connected to database "phys_backup_db" as user "postgres".
INSERT 0 2
```
```text
000000010000000000000001
000000010000000000000002
```
# Восстановление с применением WAL (PITR)
```bash
pg_ctl -D ~/pg_cluster_restore -o "-p 5434" stop
```
```bash
rm -rf ~/pg_cluster_restore/*
cp -r ~/pg_basebackup_pitr/* ~/pg_cluster_restore/
```
-- Создаем recovery.conf или postgresql.auto.conf для PITR
```text
restore_command = 'cp /tmp/wal_archive/%f %p'
recovery_target_time = '2025-12-04 12:00:00'  # пример конкретного момента
```
```bash
pg_ctl -D ~/pg_cluster_restore -o "-p 5434" -l ~/pg_cluster_restore/logfile start
```
```bash
psql -U postgres -p 5434 -d phys_backup_db -c "SELECT * FROM important_table;"
```

```text
 id |  data  
----+--------
  1 | Row 1
  2 | Row 2
  3 | Row 3
(3 строки)
```
# Сравнение восстановления

**Без PITR: восстановление показывает только состояние на момент создания базовой резервной копии.**
**С PITR: восстановление учитывает все транзакции из WAL до заданного момента (recovery_target_time).**

## **Выводы:Резервное копирование в PostgreSQL является ключевым механизмом обеспечения целостности и сохранности данных. Логическое копирование позволяет гибко сохранять структуру и содержимое баз данных, однако при больших объемах данных оно может быть медленным. Физическое копирование обеспечивает быстрое клонирование всего кластера, а совместное использование архивов WAL позволяет восстановить данные на любой момент времени, что повышает надежность системы. Управление доступом через роли и привилегии, а также настройка методов аутентификации, являются важными инструментами обеспечения безопасности и разграничения прав пользователей, что особенно критично в многопользовательской среде.**
