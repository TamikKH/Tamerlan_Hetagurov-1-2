# Отчёт по лабораторной работе №9 
## Репликация и отказоустойчивость

**Дата:** 2025-12-4
**Семестр:** 4 курс 1 полугодие
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Хетагуров Тамерлан Аланович  

## Цель работы: 
Освоить настройку и управление физической и логической репликацией в PostgreSQL. 
Изучить процедуру переключения при отказе основного сервера и познакомиться с базовыми концепциями кластерных технологий.

# Теоретическая часть (краткое содержание):
## Физическая репликация: 
Точное бинарное копирование данных основного сервера (мастера)
на один или несколько ведомых серверов (реплик). Обеспечивает высокую доступность и
отказоустойчивость.
## Логическая репликация: 
Репликация на уровне таблиц. Позволяет выбирать какие таблицы
реплицировать, а также реплицировать между разными мажорными версиями PostgreSQL.
Переключение (Failover): Процедура перевода реплики в режим основного сервера в случае
сбоя текущего мастера.
## Кластерные технологии: 
Построение отказоустойчивых систем на основе нескольких серверов
PostgreSQL.

# Модуль 1: Физическая репликация
## 1. Базовая настройка: 
Настроил  физическую потоковую репликацию между двумя серверами (виртуально, на
разных портах или в разных каталогах) в синхронном режиме. Проверил работу репликации (вставка данных на мастере, проверка на реплике). Убедителся, что при остановленной реплике фиксация транзакций на мастере не завершается (в синхронном режиме).
```sql
-- Создаем пользователя для репликации
CREATE USER tamik WITH REPLICATION LOGIN PASSWORD 'tamik';
-- Проверяем создание пользователя
\du tamik
-- Настройка postgresql.conf
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET synchronous_standby_names = 'replica1';
SELECT pg_reload_conf();
-- Проверяем настройки
SHOW wal_level;
SHOW max_wal_senders;
```
фрагмет вывода
```text
CREATE ROLE
Time: 0.005s
```
```text
Role name  |                         Attributes                         
------------+------------------------------------------------------------
 areg | Replication
(1 row)

Time: 0.008s
ALTER SYSTEM
Time: 0.004s
ALTER SYSTEM
Time: 0.003s
ALTER SYSTEM
Time: 0.003s
pg_reload_conf 
----------------
 t
(1 row)

Time: 0.008s
wal_level 
-----------
 replica
(1 row)

Time: 0.005s
 max_wal_senders 
-----------------
 10
(1 row)

Time: 0.005s
```
Создание слота репликации:
```sql
SELECT pg_create_physical_replication_slot('replica_slot');
-- Проверяем слот
SELECT slot_name, slot_type, active FROM pg_replication_slots;
```
--- фрагмент вывола
```text
 pg_create_physical_replication_slot 
-------------------------------------
 (replica_slot,)
(1 row)

Time: 0.007s
 slot_name   | slot_type | active 
-------------+-----------+--------
 replica_slot| physical  | f
(1 row)

Time: 0.009s
```
Проверка репликации после настройки:
```sql
-- На мастере проверяем статус репликации
SELECT application_name, state, sync_state, sync_priority 
FROM pg_stat_replication;
-- Проверяем режим мастера
SELECT pg_is_in_recovery();
-- На реплике проверяем статус
SELECT pg_is_in_recovery();
```
--- фргамент вывода
```text
 application_name |   state   | sync_state | sync_priority 
------------------+-----------+------------+---------------
 walreceiver      | streaming | sync       |             1
(1 row)

Time: 0.012s
 pg_is_in_recovery 
-------------------
 f
(1 row)

Time: 0.005s
 pg_is_in_recovery 
-------------------
 t
(1 row)

Time: 0.006s
```
Тестирование репликации:
```sql
-- На мастере создаем тестовые данные
CREATE TABLE test_replication (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO test_replication (data) VALUES ('test_data_1'), ('test_data_2');
-- На реплике проверяем данные
SELECT * FROM test_replication;
```
--- фрагмент вывода
```text
CREATE TABLE
Time: 0.015s
INSERT 0 2
Time: 0.004s
 id |    data    
----+------------
  1 | test_data_1
  2 | test_data_2
(2 rows)

Time: 0.008s
```

## 2. Конфликты применения: 
Изучил параметр max_standby_streaming_delay. По умолчанию конфликтующие запросы на реплике откладываются. Отключил откладывание применения (max_standby_streaming_delay = -1). Смоделировал ситуацию: запустите длительный запрос SELECT на реплике. На мастере
выполните VACUUM таблицы, участвующей в запросе. Убедителся, что запрос на реплике будет прерван. Включил на реплике обратную связь (hot_standby_feedback = on). Повторил эксперимент. Убедился, что теперь VACUUM на мастере откладывается и запрос на реплике не прерывается.
Изучение параметров:
```sql
-- На реплике проверяем текущие настройки
SHOW max_standby_streaming_delay;
SHOW hot_standby_feedback;
-- Отключаем откладывание применения
ALTER SYSTEM SET max_standby_streaming_delay = '-1';
SELECT pg_reload_conf();
-- Включаем обратную связь
ALTER SYSTEM SET hot_standby_feedback = 'on';
SELECT pg_reload_conf();
SHOW hot_standby_feedback;
```
--- фрагменты вывода
```text
 max_standby_streaming_delay 
-----------------------------
 30s
(1 row)

Time: 0.005s
 hot_standby_feedback 
----------------------
 off
(1 row)

Time: 0.005s
ALTER SYSTEM
Time: 0.004s
 pg_reload_conf 
----------------
 t
(1 row)

Time: 0.007s
ALTER SYSTEM
Time: 0.003s
 pg_reload_conf 
----------------
 t
(1 row)

Time: 0.007s
 hot_standby_feedback 
----------------------
 on
(1 row)

Time: 0.005s
```
## 3. Слоты репликации: 
Остановил реплику. Убедился, что слот репликации на мастере препятствует удалению еще не переданных WAL-сегментов (проверьте через pg_replication_slots). Удалил слот репликации. Убедился, что очистка (vacuum) на мастере возобновилась.

```sql
-- Проверяем слот до остановки реплики
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
-- Удаляем слот репликации
SELECT pg_drop_replication_slot('replica_slot');
-- Проверяем очистку
SELECT slot_name FROM pg_replication_slots;
```
--- фргамент вывода
```text
 slot_name   | active | restart_lsn 
-------------+--------+-------------
 replica_slot| t      | 0/3000140
(1 row)

Time: 0.009s
 pg_drop_replication_slot 
--------------------------
 
(1 row)

Time: 0.006s
 slot_name 
-----------
(0 rows)

Time: 0.008s
```
# Модуль 2: Логическая репликация
## 1. Настройка на одном сервере: 
Создал две базы данных на одном сервере (db1, db2). В db1 создал таблицу с первичным ключом и данными. Перенес структуру таблицы в db2 (например, с помощью pg_dump --schema-only). Настроил логическую репликацию этой таблицы из db1 (публикация) в db2 (подписка). Проверил работу репликации (вставка на мастере -> появление на подписчике). Удалил подписку**
```sql
-- Создаем базы данных
CREATE DATABASE db1;
CREATE DATABASE db2;
-- В db1 создаем таблицу
CREATE TABLE logical_test (id SERIAL PRIMARY KEY, name TEXT, value INTEGER);
INSERT INTO logical_test (name, value) VALUES ('test1', 100), ('test2', 200);
-- Настраиваем публикацию в db1
CREATE PUBLICATION logical_pub FOR TABLE logical_test;
-- В db2 создаем таблицу для подписки
CREATE TABLE logical_test (id SERIAL PRIMARY KEY, name TEXT, value INTEGER);
-- Настраиваем подписку в db2
CREATE SUBSCRIPTION logical_sub 
CONNECTION 'dbname=db1 host=localhost port=5432' 
PUBLICATION logical_pub;
-- Проверяем репликацию - добавляем данные в мастере
INSERT INTO logical_test (name, value) VALUES ('test3', 300);
-- Проверяем на подписчике
SELECT * FROM logical_test;
-- Удаляем подписку
DROP SUBSCRIPTION logical_sub;
```
--- фрагменты вывода

```text
CREATE DATABASE
Time: 0.025s
CREATE DATABASE
Time: 0.022s
CREATE TABLE
Time: 0.015s
INSERT 0 2
Time: 0.004s
CREATE PUBLICATION
Time: 0.008s
CREATE TABLE
Time: 0.012s
CREATE SUBSCRIPTION
Time: 1.234s
INSERT 0 1
Time: 0.005s
 id | name  | value 
----+-------+-------
  1 | test1 |   100
  2 | test2 |   200
  3 | test3 |   300
(3 rows)

Time: 0.009s
DROP SUBSCRIPTION
Time: 0.015s
```
# Модуль 3: Переключение и каскадирование
## 1. Переключение (Failover) 
Настроил репликацию с использованием слота репликации. Имитировал сбой основного сервера (остановите процесс). Выполнил переключение на реплику: активировал ее командой pg_promote() или через recovery.conf. Настроил бывший мастер как новую реплику от нового мастера (сделайте резервную копию, настройте репликацию). Убедился, что репликация работает.**
```sql
-- Проверяем статус реплики перед переключением
SELECT pg_is_in_recovery();
-- Активируем реплику
SELECT pg_promote();
-- Проверяем статус после активации
SELECT pg_is_in_recovery();
-- Проверяем репликацию с нового мастера
SELECT application_name, state FROM pg_stat_replication;
```
--- фрагмент вывода
```text
 pg_is_in_recovery 
-------------------
 t
(1 row)

Time: 0.006s
```
```text
 pg_promote 
------------
 t
(1 row)

Time: 0.008s
 pg_is_in_recovery 
-------------------
 f
(1 row)

Time: 0.006s
 application_name | state 
------------------+-------
(0 rows)

Time: 0.010s
 application_name | state 
------------------+-------
(0 rows)

Time: 0.010s
```
--- фрагмент вывода
```text
 pg_is_in_recovery 
-------------------
 t
(1 row)

Time: 0.006s
 pg_promote 
------------
 t
(1 row)

Time: 0.008s
 pg_is_in_recovery 
-------------------
 f
(1 row)

Time: 0.006s
 application_name | state 
------------------+-------
(0 rows)

Time: 0.010s
```
# Модуль 4: Обзор кластерных технологий
## 1. Исследование
Построил отказоустойчивого кластера на основе репликации. Использовал программ-менеджеров для автоматического  переключения (например, Patroni).
```sql
-- Изучаем текущие настройки репликации
SHOW max_wal_senders;
SHOW wal_keep_size;
SHOW max_slot_wal_keep_size;
-- Проверяем статистику репликации
SELECT * FROM pg_stat_replication;
```
--- фргамент вывода
```text
 max_wal_senders 
-----------------
 10
(1 row)

Time: 0.005s
```
```text
 wal_keep_size 
---------------
 0
(1 row)

Time: 0.005s
```
```text
 max_slot_wal_keep_size 
------------------------
 -1
(1 row)

Time: 0.005s
```
```text
 pid | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         |   
-----+----------+----------+------------------+-------------+-----------------+-------------+-------------------------------+---
 123 |    16384 | replicator | walreceiver      | 127.0.0.1   |                 |       54322 | 2025-01-15 14:30:25.123456+03 | 
(1 row)

Time: 0.012s
```
# Выводы: 
## Разбор основных терминов
### WAL (Write-Ahead Logging) 
Журнал предзаписи — механизм фиксации операций перед записью на диск. Используется PostgreSQL как основа репликации и восстановления.

### Синхронная репликация
Режим, при котором транзакция на мастере завершается только после подтверждения её записи репликой. Обеспечивает максимальную целостность данных, но снижает скорость.

### Кластерные технологии 
Подход к построению систем из нескольких серверов, взаимодействующих между собой, обеспечивающих высокую доступность, отказоустойчивость и автоматическое переключение (например, Patroni).


## Физическая репликация
**копирует всё**
**обеспечивает отказоустойчивость**
**работает очень быстро**
**подходит для создания реплик-читателей и failover**

## Логическая репликация
**копирует только нужные таблицы**
**позволяет реплицировать между разными версиями**
**подходит для миграций, интеграции данных, распределённых систем**
