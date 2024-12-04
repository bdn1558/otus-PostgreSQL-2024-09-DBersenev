#Настройка autovacuum с учетом особенностей производительности
- Создаём новую ВМ Ubuntu 22.04 LTS с параметрами:
	- vCPU - 2
	- RAM - 4 ГБ
	- Объём дискового пространства (SDD) - 10 ГБ 
	```sh
	yc compute instance create --name pg-vm1 --hostname pg-vm1 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	yc compute ssh --name pg-vm1 --identity-file yc_key --login yc-user
	```
- Устанавливаем PostgreSQL 15 
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-15
	```
	
	```sh
	yc-user@pg-vm1:~$ pg_lsclusters
	Ver Cluster Port Status Owner    Data directory              Log file
	15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
	```
	
- Создаём БД для тестов:
	```sh
	sudo su postgres
	```
	```sh
	pgbench -i postgres
	```
- Запускаем тест с д:
	```sh
	postgres@pg-vm1:/home/yc-user$ pgbench -c8 -P 6 -T 60 -U postgres postgres
	```
	
	```
	pgbench (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
	starting vacuum...end.
	progress: 6.0 s, 287.5 tps, lat 27.542 ms stddev 29.561, 0 failed
	progress: 12.0 s, 391.7 tps, lat 20.478 ms stddev 23.012, 0 failed
	progress: 18.0 s, 405.2 tps, lat 19.734 ms stddev 22.238, 0 failed
	progress: 24.0 s, 393.7 tps, lat 20.312 ms stddev 21.201, 0 failed
	progress: 30.0 s, 456.0 tps, lat 17.546 ms stddev 17.113, 0 failed
	progress: 36.0 s, 281.2 tps, lat 28.432 ms stddev 31.226, 0 failed
	progress: 42.0 s, 315.2 tps, lat 25.404 ms stddev 30.292, 0 failed
	progress: 48.0 s, 279.7 tps, lat 28.609 ms stddev 29.860, 0 failed
	progress: 54.0 s, 434.0 tps, lat 18.434 ms stddev 19.088, 0 failed
	progress: 60.0 s, 496.2 tps, lat 16.112 ms stddev 17.143, 0 failed
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 8
	number of threads: 1
	maximum number of tries: 1
	duration: 60 s
	number of transactions actually processed: 22449
	number of failed transactions: 0 (0.000%)
	latency average = 21.392 ms
	latency stddev = 24.042 ms
	initial connection time = 24.366 ms
	tps = 373.801312 (without initial connection time)
	```
	
- Применяем параметры настройки PostgreSQL из домашнего задания, добавив их в конец postgresql.conf:
	```
	max_connections = 40
	shared_buffers = 1GB
	effective_cache_size = 3GB
	maintenance_work_mem = 512MB
	checkpoint_completion_target = 0.9
	wal_buffers = 16MB
	default_statistics_target = 500
	random_page_cost = 4
	effective_io_concurrency = 2
	work_mem = 65536kB
	min_wal_size = 4GB
	max_wal_size = 16GB
	```
	
	```sh
	sudo nano /etc/postgresql/15/main/postgresql.conf
	sudo pg_ctlcluster 15 main restart
	```
- Тестируем заново
	```sh
	postgres@pg-vm1:/home/yc-user$ pgbench -c8 -P 6 -T 60 -U postgres postgres
	```
	```
	pgbench (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
	starting vacuum...end.
	progress: 6.0 s, 449.3 tps, lat 17.708 ms stddev 20.022, 0 failed
	progress: 12.0 s, 330.5 tps, lat 23.978 ms stddev 26.756, 0 failed
	progress: 18.0 s, 251.2 tps, lat 32.068 ms stddev 36.389, 0 failed
	progress: 24.0 s, 181.8 tps, lat 43.930 ms stddev 37.073, 0 failed
	progress: 30.0 s, 312.2 tps, lat 25.646 ms stddev 24.179, 0 failed
	progress: 36.0 s, 358.5 tps, lat 22.348 ms stddev 27.187, 0 failed
	progress: 42.0 s, 371.3 tps, lat 21.543 ms stddev 23.756, 0 failed
	progress: 48.0 s, 359.0 tps, lat 22.240 ms stddev 27.201, 0 failed
	progress: 54.0 s, 221.2 tps, lat 36.148 ms stddev 37.243, 0 failed
	progress: 60.0 s, 391.2 tps, lat 20.501 ms stddev 24.014, 0 failed
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 8
	number of threads: 1
	maximum number of tries: 1
	duration: 60 s
	number of transactions actually processed: 19365
	number of failed transactions: 0 (0.000%)
	latency average = 24.777 ms
	latency stddev = 28.390 ms
	initial connection time = 25.967 ms
	tps = 322.761646 (without initial connection time)
	```
	Уменьшился tps. Возможно из-за того, что настройки не соответствуют рекомендуемым для сервера с данными характеристиками. Например, max_wal_size превышает доступный объём дискового пространства.

- Проверяем включён ли AUTOVACUUM:
	```sql
	SELECT name, setting FROM pg_settings WHERE name='autovacuum';
	```
	|name|setting|
	|----|-------|
	|autovacuum|on|
	
- Создаём таблицу с текстовым полем и заполняем случайными данным в размере 1млн строк:
	```sql
	CREATE  TABLE test_table
	AS
	SELECT md5(random()::text) as test_data
	FROM generate_series(1, 1000000);
	```
- Размер таблицы:
	```sql
	SELECT pg_size_pretty(pg_total_relation_size('test_table'));
	```
	|pg_size_pretty|
	|--------------|
	|65 MB|
	
- Для удобства сделаем задание со звёздочкой и напишем анонимную процедуру, которая в цикле обновляет все строчки в таблице заданное количество раз:
	```sql
	DO $$
	DECLARE
	 vCnt int := 10;
	BEGIN
		FOR i IN 1..vCnt LOOP
			UPDATE test_table SET test_data = test_data || chr((floor(random() * 94) + 32)::int);
			RAISE NOTICE 'Шаг цикла: %', i;
		END LOOP;
	END $$;
	```
- 5 раз обновляем все строки и проверяем количество мертвых строк в таблице и когда последний раз приходил AUTOVACUUM:
	```sql
	SELECT relname, last_autovacuum, n_dead_tup 
	FROM pg_stat_user_tables
	where relname = 'test_table';
	```
	|relname|last_autovacuum|n_dead_tup|
	|-------|---------------|----------|
	|test_table|2024-11-30 20:42:18.431 +0400|5000000|
	
- Ждём, когда отработает AUTOVACUUM:

	|relname|last_autovacuum|n_dead_tup|
	|-------|---------------|----------|
	|test_table|2024-11-30 21:48:21.481 +0400|0|

- 5 раз обновляем все строки и смотрим размер файла с таблицей:
	```sql
	SELECT pg_size_pretty(pg_total_relation_size('test_table'));
	```
	|pg_size_pretty|
	|--------------|
	|415 MB|
- Отключаем AUTOVACUUM на тестовой таблице
	```sql
	ALTER TABLE test_table SET (autovacuum_enabled = false);
	```
- 10 раз обновляем все строки и смотрим размер файла с таблицей:
	```sql
	SELECT pg_size_pretty(pg_total_relation_size('test_table'));
	```
	|pg_size_pretty|
	|--------------|
	|841 MB|
- Включаем AUTOVACUUM обратно:
	```sql
	ALTER TABLE test_table SET (autovacuum_enabled = true);
	SELECT reloptions FROM pg_class WHERE relname='test_table';
	```
	
	|reloptions|
	|----------|
	|{autovacuum_enabled=true}|
	
- После отработки AUTOVACUUM мёртвых строк нет, но размер таблицы не изменился:
	|relname|last_autovacuum|n_dead_tup|
	|-------|---------------|----------|
	|test_table|2024-11-30 22:09:39.907 +0400|0|
	
	|pg_size_pretty|
	|--------------|
	|841 MB|
	
	
- PostgreSQL при выполнение UPDATE не перезаписывает старую строку, а создаёт новую версию с обновлёнными значениями, так как старая версия может использоваться другими транзакциями. То есть каждая операция UPDATE добавляет новые записи, не удаляя старые, а старые при этом помечаются "мертвыми", что приводит к увеличению физического размера таблицы. AUTOVACUUM очищает "мертвые" строки, освобождая страницы, в которых они хранились, для переиспользования, поэтому размер таблицы на диске не меняется. Для освобождения места можно использовать VACUUM FULL, которая фактически пересоздаёт таблицу, поэтому блокирует доступ к ней во время выполнения:
	```sql
	VACUUM FULL test_table
	```
	|pg_size_pretty|
	|--------------|
	|81 MB|
	




