# Создание ВМ и тестовой БД
- Создаём новую ВМ Ubuntu 22.04 LTS:
	```sh
	yc compute instance create --name pg-vm20 --hostname pg-vm15 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	
	yc compute ssh --name pg-vm20 --identity-file yc_key --login yc-user
	```
	
- Устанавливаем PostgreSQL 17:
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-17
	```
	
	```sh
	pg_lsclusters
	
	Ver Cluster Port Status Owner    Data directory              Log file
	17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
	```
# Создание бэкапов через СOPY
- Создаём базу и схему:
	```sh
	sudo -u postgres psql
	```
	```sql
	CREATE DATABASE testdb1;
	```
	```sh
	\c testdb1
	```
	```sql
	CREATE SCHEMA schema1;
	SET SEARCH_PATH = schema1;
	```
- Создаём таблицу и заполняем
	```sql
	CREATE  TABLE test_table1
    AS
    SELECT 
        gs AS ID,
        md5(random()::text) AS test_data
    FROM generate_series(1, 100) gs;
	```

- Создаём каталог для бэкапа 
	```sh
	sudo -u postgres mkdir /var/lib/postgresql/backups
	```
- Делаем логический бэкап используя утилиту COPY
	```sh
	sudo -u postgres psql
	\c testdb1
	\COPY schema1.test_table TO '/var/lib/postgresql/backups/test_table.csv' CSV HEADER DELIMITER ';';
	COPY 100
	```
- Проверяем, что файл создан:
	```sh
	sudo nano /var/lib/postgresql/backups/test_table.csv
	```
	```
	id;test_data
	1;ecbb7d7e6e9831b356543e863aec0360
	2;8885ac4f9308d51b31b62993933ed401
	3;c45af9b00ca2c9851dbcbfdcdf20e786
	4;0dce8ad764528215589b65e2cf09c36d
	5;3966b46bfb4368fb07d0ee3be174dbf6
	6;da2d8e7a32091dc1c4ec7dfbbc1c49fb
	7;124f721589099ba89b59fc374ddb33bd
	8;4bdd4ec174fda4ab07dcdb6432576d95
	9;5e01a8b1b77367ac2bd812433e85f596
	10;508fd54d2616f73acadea082a5f2dc9f
	11;2be5685d2aa8d62de5a7e782df476fb5
	12;eac2ac055aa710b7a7d7c0b1e3f8a632
	13;271fd3da2251c26f8b357d7078089bec
	```
	
- Создаём новую таблицу на основе существующей:
	```sql
	CREATE TABLE test_table_backup (LIKE test_table)
	```
	
- Восстановим в 2 таблицу данные из бэкапа:
	```sh
	\COPY schema1.test_table_backup FROM '/var/lib/postgresql/backups/test_table.csv' CSV HEADER DELIMITER ';';
	COPY 100
	```
- Проверяем, что данные перенеслись:
	```sql
	SELECT * FROM  test_table_backup;
	```
	|id|test_data|
	|--|---------|
	|1|ecbb7d7e6e9831b356543e863aec0360|
	|2|8885ac4f9308d51b31b62993933ed401|
	|3|c45af9b00ca2c9851dbcbfdcdf20e786|
	|4|0dce8ad764528215589b65e2cf09c36d|
	|5|3966b46bfb4368fb07d0ee3be174dbf6|
	|6|da2d8e7a32091dc1c4ec7dfbbc1c49fb|
	|7|124f721589099ba89b59fc374ddb33bd|
	|8|4bdd4ec174fda4ab07dcdb6432576d95|
	|9|5e01a8b1b77367ac2bd812433e85f596|
	|10|508fd54d2616f73acadea082a5f2dc9f|
	|11|2be5685d2aa8d62de5a7e782df476fb5|
	|12|eac2ac055aa710b7a7d7c0b1e3f8a632|
	|13|271fd3da2251c26f8b357d7078089bec|
	
# Создание бэкапов через pg_dump
- Cоздаём бэкап в кастомном сжатом формате двух таблиц
	```sh
	sudo -u postgres pg_dump -U postgres -d testdb1 -Fc -t schema1.test_table -t schema1.test_table_backup -f /var/lib/postgresql/backups/tables_backup.dump	
	```
	
- Создаём вторую базу и схему:
	```sh
	sudo -u postgres psql
	```
	```sql
	CREATE DATABASE testdb2;
	```
	```sh
	\c testdb2
	```
	```sql
	CREATE SCHEMA schema1;
	```
	

	
- Восстанавливаем в новую БД только вторую таблицу:
	```sh
	sudo -u postgres pg_restore -U postgres -d testdb2 -t test_table_backup /var/lib/postgresql/backups/tables_backup.dump
	```
- Проверяем, что данные восстановились:	
	```sql
	SELECT * FROM  test_table_backup;
	```
	|id|test_data|
	|--|---------|
	|1|ecbb7d7e6e9831b356543e863aec0360|
	|2|8885ac4f9308d51b31b62993933ed401|
	|3|c45af9b00ca2c9851dbcbfdcdf20e786|
	|4|0dce8ad764528215589b65e2cf09c36d|
	|5|3966b46bfb4368fb07d0ee3be174dbf6|
	|6|da2d8e7a32091dc1c4ec7dfbbc1c49fb|
	|7|124f721589099ba89b59fc374ddb33bd|
	|8|4bdd4ec174fda4ab07dcdb6432576d95|
	|9|5e01a8b1b77367ac2bd812433e85f596|
	|10|508fd54d2616f73acadea082a5f2dc9f|
	|11|2be5685d2aa8d62de5a7e782df476fb5|
	|12|eac2ac055aa710b7a7d7c0b1e3f8a632|
	|13|271fd3da2251c26f8b357d7078089bec|
	
