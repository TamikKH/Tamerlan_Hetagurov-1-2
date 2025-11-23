Отчет по лабораторной работе №6
Надежность: Блокировки и мониторинг

2025-11-15
4 курс 1 полугодие 
Пиж-б-о-22-1
Администрирование баз данных
Хетагуров Тамерлан Аланович

Цель работы: Изучить систему блокировок в PostgreSQL и методы мониторинга активности сервера.
 Получить практические навыки анализа статистики, диагностики блокировок и взаимоблокировок,
 использования инструментов мониторинга.

Теоретическая часть (краткое содержание):

Блокировки: Механизм, обеспечивающий согласованность данных при параллельном доступе.
 Бывают разных уровней: объекты, строки, буферы в памяти.
Мониторинг: Набор представлений и функций для отслеживания активности сервера, статистики
 использования объектов и блокировок.
Взаимоблокировка (Deadlock): Ситуация, когда две или более транзакции ожидают друг друга,
 освобождения ресурсов.

Модуль 1: Мониторинг активности
1. Статистика таблиц:
 В новой базе данных создайте таблицу monitor_test (id INT). Вставьте несколько строк,
 затем удалите все.
 Изучите статистику обращений к таблице в pg_stat_all_tables (n_tup_ins, n_tup_del,
 n_live_tup, n_dead_tup). Сопоставьте с действиями.
 Выполните VACUUM. Снова проверьте статистику. Объясните изменения.

'''posgresql
monitor=# CREATE TABLE monitor_test (id INT);
CREATE TABLE
monitor=# INSERT INTO monitor_test VALUES (1), (2), (3);
INSERT 0 3
monitor=# DELETE FROM monitor_test;
DELETE 3
monitor=# 
SELECT n_tup_ins, n_tup_del, n_live_tup, n_dead_tup
FROM pg_stat_all_tables
WHERE relname = 'monitor_test';

 n_tup_ins | n_tup_del | n_live_tup | n_dead_tup 
-----------+-----------+------------+------------
         3 |         3 |          0 |          3
(1 row)
'''

n_tup_ins = 3 (вставили 3 строки)
n_tup_del = 3 (удалили 3 строки)
n_live_tup = 0 (живых строк нет)
n_dead_tup > 0 (остались «мёртвые» версии строк).

'''posgresql
monitor=# VACUUM monitor_test;
VACUUM
monitor=# SELECT n_tup_ins, n_tup_del, n_live_tup, n_dead_tup
FROM pg_stat_all_tables
WHERE relname = 'monitor_test';
 n_tup_ins | n_tup_del | n_live_tup | n_dead_tup 
-----------+-----------+------------+------------
         3 |         3 |          0 |          0
(1 row)

'''

n_dead_tup стало равным нулю так как мартвые строки были перезаписаны


2. Взаимоблокировка:
 Создайте ситуацию взаимоблокировки двух транзакций (например, изменение двух строк в
 разном порядке).
 Изучите, какая информация записывается в журнал сообщений сервера при обнаружении
 взаимоблокировки.
Сессия 1:
'''posgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE monitor_test SET id = 1 WHERE id = 1;
UPDATE 0
'''

Сессия 2:
'''posgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE monitor_test SET id = 2 WHERE id = 2;
UPDATE 0
monitor=*# UPDATE monitor_test SET id = 1 WHERE id = 1;
UPDATE 0
'''

Сессия 1:
'''posgresql
monitor=*# UPDATE monitor_test SET id = 2 WHERE id = 2;
UPDATE 0
'''

В логах /var/log/postgresql/postgresql-16-main.log появилось:
ERROR:  deadlock detected
DETAIL:  Process 1743 waits for ShareLock on transaction 253; 
         blocked by process 1532.
         Process 1532 waits for ShareLock on transaction 143;
         blocked by process 1743.
HINT:  See server log for query details.

3. Расширение pg_stat_statements (Практика+):
 Установите и настройте расширение pg_stat_statements.
 Выполните несколько произвольных запросов.
 Изучите информацию в представлении pg_stat_statements (топ запросов, время
 выполнения и т.д.).

Модуль 2: Блокировки объектов
1. Блокировки при чтении:
 На уровне изоляции Read Committed прочитайте одну строку таблицы по первичному
 ключу.
 Изучите удерживаемые блокировки в pg_locks. Объясните, какие блокировки и почему
 были захвачены
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# SELECT * FROM monitor_test WHERE id = 1;
 id 
----
(0 rows)

monitor=*# SELECT locktype, mode, granted
FROM pg_locks
WHERE pid = pg_backend_pid();
  locktype  |      mode       | granted 
------------+-----------------+---------
 relation   | AccessShareLock | t
 relation   | AccessShareLock | t
 virtualxid | ExclusiveLock   | t
(3 rows)
'''
Отображается AccessShareLock потому что SELECT блокирует таблицу «на чтение», не мешая UPDATE/INSERT.

2. Повышение уровня блокировок:
 Воспроизведите ситуацию автоматического повышения уровня предикатных блокировок
 при чтении строк по индексу.
 Покажите, что это может привести к ложной ошибке сериализации.
'''postgresql
monitor=# SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
WARNING:  SET TRANSACTION can only be used in transaction blocks
SET
monitor=# SELECT * FROM monitor_test WHERE id > 0;
 id 
----
(0 rows)
'''

3. Логирование долгих ожиданий:
 Настройте запись в журнал сообщений о ожиданиях блокировок > 100 мс (log_lock_waits = on, deadlock_timeout = 100ms).
 Создайте ситуацию длительного ожидания блокировки. Убедитесь, что сообщение
 появилось в логе.
Сессия 1:
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE monitor_test SET id=1 WHERE id=1;
UPDATE 0
'''

Сессия 2:
'''postgresql
monitor=# UPDATE monitor_test SET id=1 WHERE id=1;
UPDATE 0
'''
Запись в логе:
LOG: process 2531 still waiting for ShareLock on transaction 3526

Модуль 3: Блокировки строк
1. Конфликт обновлений:
 Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в
 разных сеансах.
 Изучите возникшие блокировки в pg_locks. Объясните их тип и назначение.

Сессия 1:
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE monitor_test SET id=1 WHERE id=1;
UPDATE 0
'''

Сессия 2:
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE monitor_test SET id=2 WHERE id=1;
UPDATE 0
'''

Сессия 3:
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE monitor_test SET id=3 WHERE id=1;
UPDATE 0
'''

В pg_locks tuple lock удерживает доступ к конкретной строке.

2. Взаимоблокировка трех транзакций:
 Воспроизведите взаимоблокировку трех транзакций.
 Проанализируйте журнал сообщений сервера. Можно ли по нему понять причину
 взаимоблокировки?

Сессия 1
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE deadlock3 SET val = val + 1 WHERE id = 1;
UPDATE 1
monitor=*# UPDATE deadlock3 SET val = val + 1 WHERE id = 2;
'''

Сессия 2
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE deadlock3 SET val = val + 1 WHERE id = 2;
UPDATE 1
monitor=*# UPDATE deadlock3 SET val = val + 1 WHERE id = 3;
'''

Сессия 3
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE deadlock3 SET val = val + 1 WHERE id = 3;
UPDATE 1
monitor=*# UPDATE deadlock3 SET val = val + 1 WHERE id = 1;
'''

PostgreSQL обнаруживает deadlock и отменяет одну из транзакций.
Сессия 3:
'''postgresql
ERROR:  deadlock detected
DETAIL:  Process 48469 waits for ShareLock on transaction 37144; blocked by process 48097.
Process 48097 waits for ShareLock on transaction 37145; blocked by process 48460.
Process 48460 waits for ShareLock on transaction 37146; blocked by process 48469.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "deadlock3"
'''

Логи покажут deadlock, но в тексте не всегда сразу видно причину — нужно анализировать порядок блокировок.

3. Взаимоблокировка UPDATE:
 Попытайтесь воспроизвести ситуацию, когда две транзакции, выполняющие по одному
 UPDATE на одной таблице, взаимоблокируются. Объясните, возможно ли это.

Сессия 1
'''postgresql
monitor=# INSERT INTO deadlock_update VALUES (1, 10), (2, 20);
INSERT 0 2
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE deadlock_update SET val = val + 1 WHERE id = 1;
UPDATE 1
'''

Сессия 2
'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# UPDATE deadlock_update SET val = val + 1 WHERE id = 2;
UPDATE 1
'''

В этом случае deadlock не возникает, потому что транзакции блокируют разные строки. 
PostgreSQL просто ставит row-level lock на обновляемую строку

Модуль 4: Блокировки в оперативной памяти
1. Закрепление буферов курсором:
 Используя pg_buffercache, убедитесь, что открытый курсор удерживает закрепление
 буфера (pinning) для быстрого чтения следующей строки.

'''postgresql
monitor=# BEGIN;
BEGIN
monitor=*# DECLARE c CURSOR FOR SELECT * FROM monitor_test;
DECLARE CURSOR
'''
В pg_buffercache видно, что страницы удерживаются (pinning).

2. VACUUM и закрепление буферов:
 Откройте курсор на таблице. Не закрывая его, выполните VACUUM этой таблицы.
 Определите, будет ли VACUUM ожидать освобождения закрепления буфера.
'''postgresql
monitor=# VACUUM monitor_test;
VACUUM
'''

VACUUM не блокируется, если страницы не нужны для удаления старых версий.

2. VACUUM FREEZE и ожидание:
 Повторите эксперимент с VACUUM FREEZE.
 Убедитесь, что в профиле ожиданий процесса VACUUM появилось ожидание снятия
 закрепления буфера (buffer pin).

'''postgresql
monitor=# SET log_lock_waits = on;
SET
monitor=# SET deadlock_timeout = '100ms';
SET
monitor=# VACUUM FREEZE monitor_test;
VACUUM
'''

