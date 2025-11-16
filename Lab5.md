```markdown
Отчет по лабораторной работе №5
Надежность: Журнал предзаписи (WAL)

2025-10-19
4 курс 1 полугодие 
Пиж-б-о-22-1
Администрирование баз данных
Хетагуров Тамерлан Аланович

Цель работы: Изучить работу буферного кеша и механизма журналирования предзаписи (WAL) в
 PostgreSQL. Получить практические навыки управления контрольными точками, анализа журнальных
 записей, настройки параметров WAL и исследования процессов восстановления после сбоев

Теоретическая часть (краткое содержание):

Буферный кеш: Область общей памяти для кэширования страниц данных, считываемых с диска.
 Измененные ("грязные") буферы периодически сбрасываются на диск.
Контрольная точка (Checkpoint): Процесс принудительной записи всех "грязных" буферов на
 диск. Ограничивает объем WAL, необходимый для восстановления.
Журнал предзаписи (WAL): Циклический журнал, в который записываются все изменения
 данных перед тем, как они попадут в основные файлы данных. Обеспечивает надежность и
 возможность восстановления после сбоя.
Восстановление: Процесс применения WAL-записей, созданных после последней контрольной
 точки, к данным на диске для приведения их в согласованное состояние

 Модуль 1: Процессы и режимы остановки
 1. Поиск процессов: Средствами ОС (например, ps aux | grep postgres) найдите процессы,
 отвечающие за работу буферного кеша (checkpointer, background writer) и журнала WAL
 (walwriter).

Ищем процессы:
checkpointer — отвечает за контрольные точки.
background writer — пишет грязные страницы из буферного кеша на диск.
walwriter — сбрасывает записи WAL на диск.

student:~$ ps aux | grep postgres
postgres   16347  0.0  3.1 225548 31444 ?        Ss   13:30   0:03 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres   16348  0.0  8.4 225732 83360 ?        Ss   13:30   0:02 postgres: 16/main: checkpointer 
postgres   16349  0.0  1.0 225608 10484 ?        Ss   13:30   0:00 postgres: 16/main: background writer 
postgres   16351  0.0  1.2 225584 11884 ?        Ss   13:30   0:01 postgres: 16/main: walwriter 
postgres   16353  0.0  0.9 227036  9244 ?        Ss   13:30   0:00 postgres: 16/main: logical replication launcher 

 2. Остановка Fast:
 Остановите PostgreSQL в режиме fast (sudo pg_ctlcluster 16 main stop).
 Запустите сервер. Просмотрите журнал сообщений сервера
 (/var/log/postgresql/postgresql-16-main.log). Найдите записи о контрольной точке,
 выполненной при завершении работы.

student:~$ sudo pg_ctlcluster 16 main stop -m fast
student:~$ sudo pg_ctlcluster 16 main start

В логах /var/log/postgresql/postgresql-16-main.log указана запись о том, что сделана контрольная точка перед завершением работы:
database system is shut down

 3. Остановка Immediate:
 Остановите PostgreSQL в режиме immediate (sudo pg_ctlcluster 16 main stop -m
 immediate).
 Запустите сервер. Просмотрите журнал сообщений. Найдите записи о восстановлении
 после сбоя (recovery). Сравните с предыдущим случаем.

 student:~$ sudo pg_ctlcluster 16 main stop -m immediate
 student:~$ sudo pg_ctlcluster 16 main start

 В логах появилось:
database system was interrupted; last known up at 16:26
database system was not properly shut down; automatic recovery in progress
redo starts at 16:35
redo done at 16:35

То есть сервер понял, что выключение было аварийным → запускается recovery из WAL.
Разница:
fast - контрольная точка, корректное завершение, восстановления не требуется.
immediate - без чекпоинта, после старта идёт откат/допроигрывание WAL.

 Модуль 2: Буферный кеш и контрольные точки
 1. Анализ размера:
 Создайте таблицу wal_test (id INT, data TEXT). Вставьте в нее достаточное количество
 строк.
 Определите, сколько страниц на диске занимает таблица (например, с помощью
 pg_relation_size(...) / current_setting('block_size')::int).
 Определите, сколько буферов занимает таблица в кеше (запрос к pg_buffercache).

student:~$ sudo -u postgres psql -d postgres
postgres=# CREATE TABLE wal_test (id INT, data TEXT);
CREATE TABLE
postgres=# INSERT INTO wal_test
postgres-# SELECT g, md5(random()::text) FROM generate_series(1,100000) g;
INSERT 0 100000
postgres=# SELECT pg_relation_size('wal_test') / current_setting('block_size')::int AS pages_on_disk;
 pages_on_disk 
---------------
           834
(1 row)

postgres=# CREATE EXTENSION pg_buffercache;
CREATE EXTENSION
postgres=# SELECT count(*) 
postgres-# FROM pg_buffercache 
postgres-# WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
 count 
-------
   838
(1 row)

postgres=# 

 2. Грязные буферы и контрольная точка:
 Узнайте общее количество "грязных" буферов в кеше (запрос к pg_stat_bgwriter или
 pg_buffercache).
 Выполните команду CHECKPOINT;.
 Снова проверьте количество "грязных" буферов. Объясните результат.

postgres=# SELECT count(*) FROM pg_buffercache WHERE isdirty;
 count 
-------
   263
(1 row)

postgres=# CHECKPOINT;
CHECKPOINT
postgres=# SELECT count(*) FROM pg_buffercache WHERE isdirty;
 count 
-------
     0
(1 row)

До чекпоинта существовали «грязные» буферы (изменены, но не на диске), после чекпоинта они были сброшены.

 3. Предварительное чтение (pg_prewarm):
 Подключите расширение pg_prewarm.
 Загрузите свою таблицу в кеш с помощью pg_prewarm(...).
 Перезапустите сервер.
 Проверьте, осталась ли таблица в кеше. Проанализируйте эффективность метода.

postgres=# CREATE EXTENSION pg_prewarm;
CREATE EXTENSION
postgres=# SELECT pg_prewarm('wal_test');
 pg_prewarm 
------------
        834
(1 row)

postgres=# SELECT count(*) FROM pg_buffercache WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
 count 
-------
   838
(1 row)

Результат: таблица в кеше уже не останется → кеш не сохраняется между рестартами.
pg_prewarm помогает только подогреть кеш сразу после запуска, а не сохранить его.

 Модуль 3: Журнал предзаписи (WAL)
 1. Размер WAL-записей:
 Запомните текущую позицию LSN (SELECT pg_current_wal_lsn();).
 Создайте таблицу с первичным ключом, вставьте несколько строк, зафиксируйте.
 Определите объем сгенерированных WAL-записей (SELECT pg_current_wal_lsn() 'LSN'::pg_lsn;).

postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/236AD678
(1 row)

postgres=# CREATE TABLE wal_size (id SERIAL PRIMARY KEY, data TEXT);
CREATE TABLE
postgres=# INSERT INTO wal_size (data) VALUES ('test1'), ('test2'), ('test3');
INSERT 0 3
postgres=# COMMIT;
WARNING:  there is no transaction in progress
COMMIT
postgres=# SELECT pg_current_wal_lsn()::pg_lsn;
 pg_current_wal_lsn 
--------------------
 0/236D0F18
(1 row)

postgres=# SELECT pg_current_wal_lsn() - '0/236D0F18'::pg_lsn;
 ?column? 
----------
        0
(1 row)

 2. Анализ WAL:
 Объясните относительно большой размер записей. При необходимости используйте
 pg_waldump для просмотра заголовков записей.

Большой размер объясняется тем, что WAL хранит:
 данные об изменённых строках,
 заголовки транзакций,
 метаданные (структура страниц).

 3. Восстановление после сбоя:
 Вставьте строку в таблицу и зафиксируйте.
 Начните новую транзакцию, обновите строки, НЕ фиксируйте.
 Сымитируйте сбой, принудительно завершив процесс postmaster (kill -9 или аварийная
 остановка).
 Запустите сервер. Убедитесь, что зафиксированное изменение сохранилось, а
 незафиксированное — откатилось.
 Найдите в журнале сообщений записи о процессе восстановления.

postgres=# BEGIN;
BEGIN
postgres=*# INSERT INTO wal_size (data) VALUES ('committed_row');
INSERT 0 1
postgres=*# COMMIT;
COMMIT
postgres=# BEGIN;
BEGIN
postgres=*# UPDATE wal_size SET data = 'uncommitted_update' WHERE data = 'committed_row';
UPDATE 1

student:~$ sudo kill -9 24659
student:~$ sudo systemctl start postgresql
student:~$ sudo -u postgres psql -d postgres
postgres=# SELECT * FROM wal_size WHERE data = 'committed_row';
 id |     data      
----+---------------
  4 | committed_row
(1 row)



 Модуль 4: Настройка WAL
 1. Влияние full_page_writes:
 Изучите влияние параметра full_page_writes (вкл./выкл.) на объем генерируемых WAL записей.
 Проведите простой тест: выполните контрольную точку, затем серию изменений в таблице
 (например, с помощью pgbench), измерьте объем WAL.
 Повторите тест с разным значением full_page_writes. Объясните разницу в результатах.

Тест 1:
postgres=# DROP TABLE IF EXISTS wal_test;
DROP TABLE
postgres=# CREATE TABLE wal_test (id SERIAL PRIMARY KEY, data TEXT);
CREATE TABLE
postgres=# CHECKPOINT;
CHECKPOINT
postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2B77E778
(1 row)

student:~$ sudo -u postgres sed -i "s/^#*full_page_writes.*/full_page_writes = on/" \
  /etc/postgresql/16/main/postgresql.conf
student:~$ sudo systemctl restart postgresql

postgres=# SHOW full_page_writes;
 full_page_writes 
------------------
 on
(1 row)

postgres=# CHECKPOINT;
CHECKPOINT
postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2B7806C8
(1 row)

student:~$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.08 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.31 s (drop tables 0.04 s, create tables 0.02 s, client-side generate 0.11 s, vacuum 0.07 s, primary keys 0.07 s).
student:~$ pgbench -T 10 postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg24.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 5782
number of failed transactions: 0 (0.000%)
latency average = 1.729 ms
initial connection time = 5.134 ms
tps = 578.457529 (without initial connection time)

postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2C6B3980
(1 row)

postgres=# SELECT '0/2C6B3980'::pg_lsn - '0/2B7806C8'::pg_lsn AS wal_bytes;
 wal_bytes 
-----------
  15938232
(1 row)

Тест 2:
student:~$ sudo -u postgres sed -i "s/^full_page_writes.*/full_page_writes = off/" \
  /etc/postgresql/16/main/postgresql.conf
student:~$ sudo systemctl restart postgresql
student:~$ sudo -u postgres psql -d postgres
postgres=# CHECKPOINT;
CHECKPOINT
postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2C6B3B90
(1 row)

pgbench -T 10 postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg24.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 5818
number of failed transactions: 0 (0.000%)
latency average = 1.719 ms
initial connection time = 2.174 ms
tps = 581.832292 (without initial connection time)

postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2C91E820
(1 row)

postgres=# SELECT '0/2C91E820'::pg_lsn - '0/2C6B3B90'::pg_lsn AS wal_bytes;
 wal_bytes 
-----------
   2534544
(1 row)

при full_page_writes on  WAL значительно больше
при full_page_writes off  меньше (нет записи полной страницы при первом изменении)

 2. Эффективность сжатия WAL:
 Изучите влияние параметра wal_compression (вкл./выкл.) на объем WAL.
 Проведите тест, аналогичный п.1, с включенным и выключенным сжатием.
 Определите, во сколько раз уменьшается размер WAL-записей при использовании сжатия.

Тест 1:
student:~$ sudo -u postgres sed -i "s/^#*wal_compression.*/wal_compression = off/" \
  /etc/postgresql/16/main/postgresql.conf
student:~$ sudo systemctl restart postgresql
student:~$ sudo -u postgres psql -d postgres
postgres=# CHECKPOINT;
CHECKPOINT
postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2C91EA30
(1 row)

student:~$ pgbench -T 10 postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg24.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 7252
number of failed transactions: 0 (0.000%)
latency average = 1.379 ms
initial connection time = 1.542 ms
tps = 725.261502 (without initial connection time)
student:~$ sudo -u postgres psql -d postgres
postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2CC0C648
(1 row)

postgres=# SELECT '0/2CC0C648'::pg_lsn - '0/2C91EA30'::pg_lsn AS wal_bytes;
 wal_bytes 
-----------
   3071000
(1 row)

Тест 2:

student:~$ sudo -u postgres sed -i "s/^wal_compression.*/wal_compression = on/" \
  /etc/postgresql/16/main/postgresql.conf
student:~$ sudo systemctl restart postgresql
student:~$ sudo -u postgres psql -d postgres
postgres=# CHECKPOINT;
CHECKPOINT
postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2CC0C7E0
(1 row)

student:~$ pgbench -T 10 postgres
pgbench (16.10 (Ubuntu 16.10-1.pgdg24.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 7206
number of failed transactions: 0 (0.000%)
latency average = 1.387 ms
initial connection time = 3.609 ms
tps = 720.842491 (without initial connection time)
student:~$ sudo -u postgres psql -d postgres

postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2CF01208
(1 row)

postgres=# SELECT '0/2CF01208'::pg_lsn - '0/2CC0C7E0'::pg_lsn AS wal_bytes;
 wal_bytes 
-----------
   3099176
(1 row)
