# Логическая репликация

## Создание ВМ
- Создаём 3 ВМ
	```sh
	yc compute instance create --name pg-vm1 --hostname pg-vm1 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	```
	```sh
	yc compute instance create --name pg-vm2 --hostname pg-vm2 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	```
	```sh
	yc compute instance create --name pg-vm3 --hostname pg-vm3 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	```

- Устанавливаем и настраиваем PostgreSQL 17 на каждой из ВМ:
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-17
	```
	
	```sh
	sudo nano /etc/postgresql/17/main/postgresql.conf
	sudo nano /etc/postgresql/17/main/pg_hba.conf
	```
	
	```sh
	sudo -u postgres psql
	\password   ******
	\q
	```
	
	```
	sudo pg_ctlcluster 17 main restart
	```
## Настройка ВМ1	
- Создаём базу и таблицы test и test2
	```sh
	sudo -u postgres psql
	```
	```sql
	CREATE DATABASE testdb;
	```
	```sh
	\c testdb
	```
	```sql
	CREATE SCHEMA testschema;
	SET SEARCH_PATH = testschema;
	CREATE TABLE test (ID SERIAL PRIMARY KEY, value text);
	CREATE TABLE test2 (ID SERIAL PRIMARY KEY, value text);
	```
- Устанавливаем на первой ВМ уровень уровня ведения журнала на логический:
	```sql
	ALTER SYSTEM SET wal_level = logical;
	```
- Перезагружаем кластер и проверяем
	```sql
	show wal_level;
	```
	|wal_level|
	|---------|
	|logical|

- Создаем публикацию таблицы test
	```sh
	\c testdb
	```
	```sql
	CREATE PUBLICATION test_pub FOR TABLE testschema.test;
	```
## Настройка ВМ2
- Создаём базу и таблицы test и test2
	```sh
	sudo -u postgres psql
	```
	```sql
	CREATE DATABASE testdb;
	```
	```sh
	\c testdb
	```
	```sql
	CREATE SCHEMA testschema;
	SET SEARCH_PATH = testschema;
	CREATE TABLE test (ID SERIAL PRIMARY KEY, value text);
	CREATE TABLE test2 (ID SERIAL PRIMARY KEY, value text);
	```
- Устанавливаем уровень уровня ведения журнала на логический:
	```sql
	ALTER SYSTEM SET wal_level = logical;
	```
- Перезагружаем кластер и проверяем
	```sql
	show wal_level;
	```
	|wal_level|
	|---------|
	|logical|

- Создаем публикацию таблицы test2
	```sh
	\c testdb
	```
	```sql
	CREATE PUBLICATION test2_pub FOR TABLE testschema.test2;
	```	
	
## Создание подписок
- На ВМ2 подписываемся на публикацию таблицы test с ВМ1	
	```sql
	CREATE SUBSCRIPTION test_sub CONNECTION 'host=192.168.0.17 port=5432 user=postgres password=****** dbname=testdb' PUBLICATION test_pub WITH (copy_data = false);
	```
- На ВМ1 подписываемся на публикацию таблицы test с ВМ2

	```sql
	CREATE SUBSCRIPTION test2_sub CONNECTION 'host=192.168.0.14 port=5432 user=postgres password=****** dbname=testdb' PUBLICATION test2_pub WITH (copy_data = false);
	```

- Добавляем записи в таблицу test на ВМ1
	```sql
	INSERT INTO test(id, value ) VALUES (1, 'Easy come'), (2, 'easy go');
	```

- Проверяем, что данные реплицировались в test на ВМ2:
	```sql
	SELECT * FROM  test;
	```
	|id|value|
	|--|-----|
	|1|Easy come|
	|2|easy go|
	

- Обновляем запись в таблице test на ВМ1 и проверяем на ВМ2:
	```sql
	UPDATE test SET value = value || '###' WHERE id = 2;
	```

	|id|value|
	|--|-----|
	|1|Easy come|
	|2|easy go###|
	
- Проверяем удаление:
	```sql
	DELETE  FROM  test WHERE id = 2;
	```
	|id|value|
	|--|-----|
	|1|Easy come|
	
- Добавляем записи в таблицу test2 на ВМ2 и проверяем репликацию на ВМ1:
	```sql
	INSERT INTO test2(id, value ) VALUES (1, 'Curiosity'), (2, 'killed'), (3, 'the cat');
	```
	
	```sql
	SELECT * FROM test2;
	```
	
	|id|value|
	|--|-----|
	|1|Curiosity|
	|2|killed|
	|3|the cat|

- Обновляем запись в таблице test2 на ВМ2 и проверяем репликацию на ВМ1:
	```sql
	UPDATE test2 SET value = 'the dog' WHERE id = 3;
	```
	
	|id|value|
	|--|-----|
	|1|Curiosity|
	|2|killed|
	|3|the dog|

## Настройка ВМ3, как реплики для чтения и бэкапов
- Создаём базу и таблицы test и test2
	```sh
	sudo -u postgres psql
	```
	```sql
	CREATE DATABASE testdb;
	```
	```sh
	\c testdb
	```
	```sql
	CREATE SCHEMA testschema;
	SET SEARCH_PATH = testschema;
	CREATE TABLE test (ID SERIAL PRIMARY KEY, value text);
	CREATE TABLE test2 (ID SERIAL PRIMARY KEY, value text);
	```	
- Создание подписок
	```sql
	CREATE SUBSCRIPTION test3_sub CONNECTION 'host=192.168.0.17 port=5432 user=postgres password=****** dbname=testdb' PUBLICATION test_pub WITH (copy_data = false);

	CREATE SUBSCRIPTION test4_sub CONNECTION 'host=192.168.0.14 port=5432 user=postgres password=****** dbname=testdb' PUBLICATION test2_pub WITH (copy_data = false);
	```
	
- Вставка на ВМ1:
	```sql
	INSERT INTO testschema.test(id, value ) VALUES (11, 'Better'), (12, 'late'), (13, 'than'), (14, 'never');
	```
	
- Вставка на ВМ2:
	```sql
	INSERT INTO testschema.test2(id, value ) VALUES (11, 'Too many cooks'), (12, 'spoil the broth');
	```
	
- Проверка репликации на ВМ3:
	```sql
	SELECT * FROM test;
	```
	|id|value|
	|--|-----|
	|11|Better|
	|12|late|
	|13|than|
	|14|never|
	
	```sql
	SELECT * FROM test2;
	```
	|id|value|
	|--|-----|
	|11|Too many cooks|
	|12|spoil the broth|

# Физическая репликация (задание со звёздочкой)

## Создание ВМ для реплики

- Создаём ещё одну ВМ
	```sh
	yc compute instance create --name pg-vm4 --hostname pg-vm4 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	```
	
- Устанавливаем PostgreSQL 17:
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-17
	```
	
## Подготовка мастера
- Создаём на ВМ3 пользователя для репликации
	```sh
	sudo -u postgres psql
	```
	```sql
	CREATE ROLE testrepl WITH REPLICATION PASSWORD '******' LOGIN;
	```
- Разрешаем подключение для репликации c ВМ4:
	```sh
	sudo nano /etc/postgresql/17/main/pg_hba.conf
	```
	```
	host    replication     testrepl     192.168.0.40/32          md5
	```
	
- Проверяем настройки:
	```sql
	SHOW wal_level;
	```
	|wal_level|
	|---------|
	|replica|
	
	
	```sql
	SHOW max_wal_senders;
	```
	|max_wal_senders|
	|---------------|
	|10|
	
	```sql
	SHOW hot_standby;
	```	
	|hot_standby|
	|-----------|
	|on|
	
- Рестартуем кластер
	```
	sudo pg_ctlcluster 17 main restart
	```
## Подготовка реплики (Standby)
- Останавливаем кластер:
	```sh
	sudo pg_ctlcluster 17 main stop
	```

- Удаляем файлы c данными:
	```
	sudo -u postgres rm -r /var/lib/postgresql/12/main/
	```
- Создаём резервную копию с основного сервера:
	```sh
	sudo -u postgres pg_basebackup -h 192.168.0.11 -p 5432 -U testrepl -D /var/lib/postgresql/17/main/ -Fp -Xs -R
	```
- Запускаем кластер:
	```sh
	sudo pg_ctlcluster 17 main start
	```
	
## Проверка репликации:
- На основном сервере (ВМ3), проверяем информацию о подключенных репликах:
	```sql
	SELECT * FROM pg_stat_replication;
	```
	|pid|usesysid|usename|application_name|client_addr|client_hostname|client_port|backend_start|backend_xmin|state|sent_lsn|write_lsn|flush_lsn|replay_lsn|write_lag|flush_lag|replay_lag|sync_priority|sync_state|reply_time|
	|---|--------|-------|----------------|-----------|---------------|-----------|-------------|------------|-----|--------|---------|---------|----------|---------|---------|----------|-------------|----------|----------|
	|2453|16411|testrepl|17/main|192.168.0.40||49112|2024-12-27 13:38:46.935 +0400||streaming|0/3000168|0/3000168|0/3000168|0/3000168||||0|async|2024-12-27 13:44:39.688 +0400|

- На реплике (ВМ4) проверяем состояние получения WAL с основного сервера:
	```sql
	SELECT * FROM pg_stat_wal_receiver;
	```
	|pid|status|receive_start_lsn|receive_start_tli|written_lsn|flushed_lsn|received_tli|last_msg_send_time|last_msg_receipt_time|latest_end_lsn|latest_end_time|slot_name|sender_host|sender_port|conninfo|
	|---|------|-----------------|-----------------|-----------|-----------|------------|------------------|---------------------|--------------|---------------|---------|-----------|-----------|--------|
	|7104|streaming|0/3000000|1|0/3000168|0/3000168|1|2024-12-27 13:56:09.886 +0400|2024-12-27 13:56:09.933 +0400|0/3000168|2024-12-27 13:41:39.592 +0400||192.168.0.11|5432|user=testrepl password=******** channel_binding=prefer dbname=replication host=192.168.0.11 port=5432 fallback_application_name=17/main sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable|

- Проверяем, что логическая и физическая репликации работают:
	1) Добавляем строку в test на ВМ1
	```sql
	INSERT INTO testschema.test(id, value ) VALUES (100, 'Тест репликации');
	```
	2) Добавляем строку в test2 на ВМ2
	```sql
	INSERT INTO testschema.test(id, value ) VALUES (100, 'Тест репликации');
	```
	3) Проверяем test и test2 на ВМ4:
	```sql
	SELECT * FROM test;
	```
	|id|value|
	|--|-----|
	|11|Better|
	|12|late|
	|13|than|
	|14|never|
	|100|Тест репликации|


	```sql
	SELECT * FROM test2;	
	```
	|id|value|
	|--|-----|
	|11|Too many cooks|
	|12|spoil the broth|
	|100|Тест репликации|
