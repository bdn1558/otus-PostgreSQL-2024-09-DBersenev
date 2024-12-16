# Создание ВМ и тестовой БД
- Создаём новую ВМ Ubuntu 22.04 LTS:
	```sh
	yc compute instance create --name pg-vm15 --hostname pg-vm15 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	yc compute ssh --name pg-vm15 --identity-file yc_key --login yc-user
	```
	
- Устанавливаем PostgreSQL 17:
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-17
	```
	
	```sh
	yc-user@pg-vm15:~$ pg_lsclusters
	Ver Cluster Port Status Owner    Data directory              Log file
	17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-15-main.log
	```
	
- Разворачиваем БД из файла	
	```sh
	psql -U postgres -d postgres -f /tmp/demo_medium.sql
	```
# Секционирование таблицы	
- Секционируем таблицу ticket_flights по хешу. Создаём новую таблицу с полями, как в существующей, которая будет использовать секционирование по хешу по ticket_no:
	```sql
	CREATE TABLE bookings.ticket_flights_part
	(LIKE bookings.ticket_flights)
	PARTITION BY HASH (ticket_no);
	```	
- Создаём 4 партиции
	```sql
	CREATE TABLE bookings.ticket_flights_part_0 PARTITION OF bookings.ticket_flights_part FOR VALUES WITH (modulus 4, remainder 0);
	CREATE TABLE bookings.ticket_flights_part_1 PARTITION OF bookings.ticket_flights_part FOR VALUES WITH (modulus 4, remainder 1);
	CREATE TABLE bookings.ticket_flights_part_2 PARTITION OF bookings.ticket_flights_part FOR VALUES WITH (modulus 4, remainder 2);
	CREATE TABLE bookings.ticket_flights_part_3 PARTITION OF bookings.ticket_flights_part FOR VALUES WITH (modulus 4, remainder 3);
	```
	
- Создаём индексы в секциях, как в исходной таблице:
	```sql
	CREATE UNIQUE INDEX ticket_flights_part_0_pkey ON bookings.ticket_flights_part_0 USING btree (ticket_no, flight_id);
	CREATE UNIQUE INDEX ticket_flights_part_1_pkey ON bookings.ticket_flights_part_1 USING btree (ticket_no, flight_id);
	CREATE UNIQUE INDEX ticket_flights_part_2_pkey ON bookings.ticket_flights_part_2 USING btree (ticket_no, flight_id);
	CREATE UNIQUE INDEX ticket_flights_part_3_pkey ON bookings.ticket_flights_part_3 USING btree (ticket_no, flight_id);
	```
- Переносим данные из исходной таблицы в новую секционированную таблицу: 
	```sql
	INSERT INTO bookings.ticket_flights_part (ticket_no, flight_id, fare_conditions, amount)
	SELECT ticket_no, flight_id, fare_conditions, amount FROM bookings.ticket_flights;
	```
- Проверяем список секций и их размер:
	```sql
	SELECT 
		child.relname AS partition_name,
		pg_size_pretty(pg_total_relation_size(child.oid)) AS partition_size
	FROM pg_inherits
	JOIN pg_class child 
		ON child.oid = pg_inherits.inhrelid
	JOIN pg_class parent 
		ON parent.oid = pg_inherits.inhparent
	WHERE 
		parent.relname = 'ticket_flights_part';
	```

	|partition_name|partition_size|
	|--------------|--------------|
	|ticket_flights_part_0|61 MB|
	|ticket_flights_part_1|61 MB|
	|ticket_flights_part_2|62 MB|
	|ticket_flights_part_3|61 MB|

- Создаём новое бронирование:
	```sql
	INSERT INTO bookings (book_ref, book_date, total_amount)
	VALUES ('QWERTY', bookings.now(), 0);

	INSERT INTO tickets (ticket_no, book_ref, passenger_id, passenger_name)
	VALUES ('9000000000009', 'QWERTY', '8456 145698', 'DICK LAURENT');

	INSERT INTO ticket_flights_part (ticket_no, flight_id, fare_conditions, amount)
	VALUES ('9000000000009', 9720, 'Business', 0),
	 ('9000000000009', 6662, 'Business', 0);
	```
- Новое значение попало в секцию ticket_flights_part_1
	```sql
	EXPLAIN ANALYZE
	SELECT *
	FROM bookings.ticket_flights_part
	WHERE ticket_no = '9000000000009';
	```
	
	|QUERY PLAN|
	|----------|
	|Bitmap Heap Scan on ticket_flights_part_1 ticket_flights_part  (cost=4.45..16.26 rows=3 width=32) (actual time=0.029..0.030 rows=2 loops=1)|
	|  Recheck Cond: (ticket_no = '9000000000009'::bpchar)|
	|  Heap Blocks: exact=1|
	|  ->  Bitmap Index Scan on ticket_flights_part_1_pkey  (cost=0.00..4.45 rows=3 width=0) (actual time=0.021..0.021 rows=2 loops=1)|
	|        Index Cond: (ticket_no = '9000000000009'::bpchar)|
	|Planning Time: 0.085 ms|
	|Execution Time: 0.048 ms|
