# Отчёт по лабораторной работе №7 
## Управление доступом, расширениями и локализацией

**Дата:** 2025-11-15  
**Курс:** 4 курс, 1 полугодие  
**Группа:** Пиж-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Хетагуров Тамерлан Аланович  

## Цель работы
Освоить управление правами доступа пользователей, работу с расширениями
 PostgreSQL и настройку параметров локализации. Получить практические навыки настройки
 аутентификации, управления привилегиями, установки расширений и миграции данных между разными
 кодировками.

# Теоретическая часть (краткое содержание):
## Управление доступом:
Система ролей и привилегий в PostgreSQL. Настройка аутентификации
 через файл pg_hba.conf.
## Расширения: 
Способ упаковки и установки дополнительного функционала (типы данных,
 функции, операторы) в PostgreSQL.
## Локализация:
Настройки кодировки, правил сортировки (collation) и формата даты/времени.

# Модуль 1: Управление доступом
## 1. Базовые привилегии:
 Создайте новую базу данных access_db и двух пользователей: writer (с правом создания
 объектов) и reader (только чтение).
 Отзовите у роли public все привилегии на схему public в новой БД. Выдайте роли writer
 права CREATE и USAGE на схему public, а роли reader — только USAGE.
 Настройте привилегии по умолчанию так, чтобы роль reader автоматически получала
 право SELECT на новые таблицы в схеме public, принадлежащие writer.
 Создайте пользователей w1 (входит в роль writer) и r1 (входит в роль reader).
 Подключитесь под writer, создайте таблицу test_table. Убедитесь, что r1 может только
 читать ее, а w1 — имеет полный доступ (включая DELETE).

```posgresql
monitor=# CREATE DATABASE access_db;
CREATE DATABASE
monitor=# CREATE ROLE writer;
CREATE ROLE
monitor=# CREATE ROLE reader;
CREATE ROLE
monitor=# CREATE USER w1 PASSWORD 'w1pass';
CREATE ROLE
monitor=# CREATE USER r1 PASSWORD 'r1pass';
CREATE ROLE
monitor=# GRANT writer TO w1;
GRANT ROLE
monitor=# GRANT reader TO r1;
GRANT ROLE
monitor=# \c access_db;
You are now connected to database "access_db" as user "postgres".
access_db=# REVOKE ALL ON SCHEMA public FROM PUBLIC;
REVOKE
access_db=# GRANT CREATE, USAGE ON SCHEMA public TO writer;
GRANT
access_db=# GRANT USAGE ON SCHEMA public TO reader;
GRANT
access_db=# ALTER DEFAULT PRIVILEGES FOR ROLE writer GRANT SELECT ON TABLES TO reader;
ALTER DEFAULT PRIVILEGES
access_db=# \c access_db w1
You are now connected to database "access_db" as user "w1".
access_db=> ;
access_db=> CREATE TABLE test_table (id INT, val TEXT);
CREATE TABLE
access_db=> INSERT INTO test_table VALUES (1,'aaa'),(2,'bbb');
INSERT 0 2
access_db=> \c access_db r1
You are now connected to database "access_db" as user "r1".
access_db=> SELECT * FROM test_table;
 id | val 
----+-----
  1 | aaa
  2 | bbb
(2 rows)
access_db=> DELETE FROM test_table WHERE id=1;
ERROR:  permission denied for table test_table
```


## 2. Аутентификация (Практика+):
 Создайте пользовательские роли alice и bob.
 Отредактируйте pg_hba.conf, разрешив беспарольный вход (trust) только для postgres и
 student. Для всех остальных методов установите reject или md5. Перезагрузите
 конфигурацию. Убедитесь, что вход для alice и bob запрещен.
 
```posgresql
access_db=# CREATE ROLE alice LOGIN;
CREATE ROLE
access_db=# CREATE ROLE bob LOGIN;
CREATE ROLE
student:~$ sudo pg_ctlcluster 16 main reload;
access_db=# \c access_db alice
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  pg_hba.conf rejects connection for host "[local]", user "alice", database "access_db", no encryption
Previous connection kept
```

Для alice и bob настройте аутентификацию по peer (сопоставление с пользователем ОС).
 Убедитесь, что войти нельзя без создания пользователя в ОС.
 Создайте в ОС пользователя alice. Настройте вход в PostgreSQL для роли alice с методом
 peer. Проверьте вход.
 Проверьте, можно ли использовать одно отображение peer для нескольких ролей.
```bash
student:~$ sudo adduser alice;
info: Adding user `alice' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `alice' (1001) ...
info: Adding new user `alice' (1001) with group `alice (1001)' ...
info: Creating home directory `/home/alice' ...
info: Copying files from `/etc/skel' ...
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for alice
Enter the new value, or press ENTER for the default
	Full Name []: alice
	Room Number []: 12345
	Work Phone []: 12345
	Home Phone []: 12345
	Other []: 12345
Is the information correct? [Y/n] y
info: Adding new user `alice' to supplemental / extra groups `users' ...
info: Adding user `alice' to group `users' ...
student:~$
```



# Модуль 2: Управление расширениями
## 1. Установка расширения:
 Установите расширение units_of_measure (или uom, или аналогичное, доступное в вашей
 сборке). Убедитесь, что оно появилось в pg_available_extensions.
```posgresql
access_db=# SELECT * FROM pg_available_extensions;
        name        | default_version | installed_version |                                comment                                 
--------------------+-----------------+-------------------+------------------------------------------------------------------------
 file_fdw           | 1.0             |                   | foreign-data wrapper for flat file access
 pg_walinspect      | 1.1             |                   | functions to inspect contents of PostgreSQL Write-Ahead Log
 plpgsql            | 1.0             | 1.0               | PL/pgSQL procedural language
 pg_buffercache     | 1.4             |                   | examine the shared buffer cache
 pg_visibility      | 1.2             |                   | examine the visibility map (VM) and page-level visibility info
 btree_gin          | 1.3             |                   | support for indexing common datatypes in GIN
 btree_gist         | 1.7             |                   | support for indexing common datatypes in GiST
 pg_freespacemap    | 1.2             |                   | examine the free space map (FSM)
 pgcrypto           | 1.3             |                   | cryptographic functions
 xml2               | 1.1             |                   | XPath querying and XSLT
 autoinc            | 1.0             |                   | functions for autoincrementing fields
 isn                | 1.2             |                   | data types for international product numbering standards

access_db=# CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION
```


## 2. Создание и исследование:
 Создайте расширение в своей БД без указания версии. Определите, какая версия
 установилась и какие скрипты выполнились (можно посмотреть в системном каталоге).
```posgresql
access_db=# SELECT * FROM pg_extension WHERE extname='uuid-ossp';

  oid  |  extname  | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition 
-------+-----------+----------+--------------+----------------+------------+-----------+--------------
 24996 | uuid-ossp |       10 |         2200 | t              | 1.1        |           | 
(1 row)
```


## 3. Добавление данных:
 Добавьте в справочник расширения новые единицы измерения (например, футы и дюймы).
```posgresql
access_db=# CREATE TABLE uom (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name TEXT NOT NULL,
    abbreviation TEXT NOT NULL UNIQUE
);
CREATE TABLE
access_db=# INSERT INTO uom (name, abbreviation) VALUES ('foot','ft'), ('inch','in');
INSERT 0 2
```

## 4. Управление доступом:
 Измените права доступа к таблицам расширения: отзовите SELECT у public, выдайте его
 новой специальной роли.
```posgresql
access_db=# REVOKE SELECT ON uom FROM PUBLIC;
REVOKE
access_db=# CREATE ROLE meas_reader;
CREATE ROLE
access_db=# GRANT SELECT ON uom TO meas_reader;
GRANT
```
 
## 5. Резервное копирование:
 Используя pg_dump, выгрузите объекты расширения. Изучите дамп. Убедитесь, что
 выгружаются метаданные (таблицы, типы, функции) и данные.

```bash
student:~$ pg_dump -Fc access_db > access_db.dump
student:~$ pg_restore -l access_db.dump
;
; Archive created at 2025-12-07 19:09:38 MSK
;     dbname: access_db
;     TOC Entries: 16
;     Compression: gzip
;     Dump Version: 1.15-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;     Dumped by pg_dump version: 16.10 (Ubuntu 16.10-1.pgdg24.04+1)
;
;
; Selected TOC Entries:
;
3458; 0 0 ACL - SCHEMA public pg_database_owner
2; 3079 24996 EXTENSION - uuid-ossp 
3459; 0 0 COMMENT - EXTENSION "uuid-ossp" 
216; 1259 24989 TABLE public test_table w1
3460; 0 0 ACL public TABLE test_table w1
217; 1259 25007 TABLE public uom postgres
3461; 0 0 ACL public TABLE uom postgres
3450; 0 24989 TABLE DATA public test_table w1
3451; 0 25007 TABLE DATA public uom postgres
3304; 2606 25016 CONSTRAINT public uom uom_abbreviation_key postgres
3306; 2606 25014 CONSTRAINT public uom uom_pkey postgres
2052; 826 24988 DEFAULT ACL - DEFAULT PRIVILEGES FOR TABLES writer
```
 
# Модуль 3: Локализация
## 1. Миграция между кодировками:
 Создайте базу данных с кодировкой KOI8R. В ней создайте таблицу и вставьте строки с
 кириллическими символами.
 Сделайте логический дамп этой базы с помощью pg_dump.
 Создайте базу данных с кодировкой UTF8. Восстановите в нее дамп.
 Проверьте целостность данных. Объясните возможные проблемы и их решения.
## 2. Локализация дат:
 Получите номер сегодняшнего дня недели (например, с помощью EXTRACT(DOW FROM
 CURRENT_DATE)).
 Измените параметры локализации сеанса (например, lc_time). Повлияло ли это на номер
 дня недели? Объясните, почему


# Результаты выполнения 
## Сводная таблица результатов 
```posgresql
| Модуль | Задача | Статус     | Ключевые наблюдения                                                                                                  |
|    1   |    1   |Выполнено   | После INSERT и DELETE статистика фиксирует вставленные (n_tup_ins=3),                                                |
|        |        |            | удалённые (n_tup_del=3) и мёртвые строки (n_dead_tup=3).                                                             |
|        |        |            | После VACUUM n_dead_tup=0 — «мёртвые» версии очищены.                                                                | 
|    1   |    2   | Выполнено  | В логах зафиксирован deadlock — PostgreSQL обнаружил циклическое ожидание блокировок и отменил одну транзакцию.      | 
|    1   |    3   | Невыполнено|----------------------------------------------------------------------------------------------------------------------| 
|    2   |    1   | Выполнено  | SELECT создаёт AccessShareLock на таблицу и virtualxid — чтение не блокирует UPDATE/DELETE.                          | 
|    2   |    2   | Выполнено  | В SERIALIZABLE PostgreSQL повышает уровень предикатных блокировок, что может вызвать ложный конфликт сериализации.   | 
|    2   |    3   | Выполнено  | При log_lock_waits=on зафиксировано сообщение о долгом ожидании блокировки (дольше deadlock_timeout).                | 
|    3   |    1   | Выполнено  | При обновлении одной строки тремя транзакциями возникает очередь row-level locks; фиксированы tuple-level блокировки.| 
|    3   |    2   | Выполнено  | PostgreSQL обнаружил циклическую зависимость и завершил одну транзакцию;                                             |
|        |        |            | в логах видно цепочку процессов, но не всегда сразу причину.                                                         | 
|    3   |    3   | Выполнено  | При обновлении разных строк deadlock невозможен — транзакции блокируют независимые tuple, ожиданий нет.              | 
|    4   |    1   | Выполнено  | Открытый курсор удерживает pin страниц, что видно через pg_buffercache.                                              | 
|    4   |    2   | Выполнено  | VACUUM не блокируется, если pinned-страницы не требуют очистки dead tuples.                                          | 
|    4   |    3   | Выполнено  | VACUUM FREEZE ожидает снятия pin, что отражено в профиле ожиданий (buffer pin wait).                                 | 
```

## Анализ и выводы 
1. PostgreSQL ведёт подробную статистику по вставкам, удалению и «мёртвым» строкам.
2. После удаления строки физически остаются в таблице, пока VACUUM не очистит их.
3. VACUUM уменьшает n_dead_tup, но не меняет n_tup_ins и n_tup_del, так как это счётчики операций.
4. SELECT ставит лёгкие блокировки (AccessShareLock) и не мешает изменениям.
5. В режиме SERIALIZABLE может произойти ложная ошибка сериализации из-за предикатных блокировок.
6. PostgreSQL логирует долгие ожидания и deadlock-ситуации, что облегчает диагностику.
7. UPDATE блокирует только нужные строки (tuple-level lock), что позволяет высокой параллельности.
8. Deadlock возникает только при циклическом ожидании — даже три транзакции могут попасть в петлю.
9. Если строки разные — deadlock невозможен.
10. Открытый курсор удерживает pin страниц, чтобы обеспечить согласованное последовательное чтение.
11. VACUUM может ждать снятия буферных pin’ов, особенно в режиме FREEZE.
12. Обычный VACUUM часто не блокируется, так как не трогает pinned-страницы.

## Сравнительный анализ
```posgresql
Механизм          | Преимущества                                      | Ограничения                             |
tuple-level locks | Высокая параллельность, блокируется только строка | Возможность deadlock при цикле          | 
AccessShareLock   | Не мешает изменениям                              |защищает только структуру таблицы        |
VACUUM           	| Убирает dead tuples    	                          | Не освобождает место ОС без VACUUM FULL |
Cursor buffer pin | Быстрое последовательное чтение                   | Может задерживать VACUUM FREEZE         |
```
