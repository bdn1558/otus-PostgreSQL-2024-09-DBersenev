# Создание ВМ и тестовой БД
- Создаём новую ВМ Ubuntu 22.04 LTS:
	```sh
	yc compute instance create --name pg-vm9 --hostname pg-vm9 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	yc compute ssh --name pg-vm9 --identity-file yc_key --login yc-user
	```
	
- Устанавливаем PostgreSQL 17:
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-17
	```
	
	```sh
	yc-user@pg-vm9:~$ pg_lsclusters
	Ver Cluster Port Status Owner    Data directory              Log file
	17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-15-main.log
	```
- В качестве тестовой БД используем https://github.com/emil/f1/

- Создаём базу и схему:
	```sh
	sudo -u postgres psql
	```
	```sql
	CREATE DATABASE f1db;
	```
	```sh
	\c f1db
	```
	```sql
	CREATE SCHEMA f1;
	SET SEARCH_PATH = f1;
	```
	```
	\i f1db_postgres.sql
	```
	
# Создание индекса
- Выполняем запрос, отбирающий гонки из таблицы races за последние 10 лет:
	```sql
	SELECT * FROM f1.races r
	WHERE r.date >= CURRENT_DATE - INTERVAL '10 years'; 
	```
	В плане видим, что выполняется последовательное сканирование Seq Scan, так как индекса по полю date нет:
	
	|QUERY PLAN|
	|----------|
	|Seq Scan on races  (cost=0.00..34.81 rows=102 width=100)|
	|  Filter: (date >= (CURRENT_DATE - '10 years'::interval))|

- Создаём индекс по полю date и смотрим план:
	```sql
	CREATE INDEX  races_date_idx on f1.races  (date);
	```

	|QUERY PLAN|
	|----------|
	|Bitmap Heap Scan on races r  (cost=5.07..23.86 rows=102 width=100)|
	|  Recheck Cond: (date >= (CURRENT_DATE - '10 years'::interval))|
	|  ->  Bitmap Index Scan on races_date_idx  (cost=0.00..5.04 rows=102 width=0)|
	|        Index Cond: (date >= (CURRENT_DATE - '10 years'::interval))|

	- Сначала производится поиск по индексу и создание в памяти битовой карты (Bitmap Index Scan) для всех записей с date >= (CURRENT_DATE - '10 years'::interval)
	- Затем выполняется Bitmap Heap Scan, чтобы извлечь данные из таблицы на основе битовой карты

# Индекс для полнотекстового поиска	
- Выполняем полнотекстовый поиск по полю name в таблице circuits
	```sql
	SELECT * FROM f1.circuits c WHERE to_tsvector('english', c."name") @@ to_tsquery('speedway | raceway');
	```
	Снова видим, что идёт последовательное сканирование:
	
	|QUERY PLAN|
	|----------|
	|Seq Scan on circuits c  (cost=0.00..39.41 rows=1 width=118)|
	|  Filter: (to_tsvector('english'::regconfig, (name)::text) @@ to_tsquery('speedway &#124; raceway'::text))|

	
- Создаём индекс для полнотекстового поиска по полю name:
	```sql
	CREATE INDEX circuits_name_idx ON f1.circuits USING gin(to_tsvector('english', name));
	```
	
	Аналогично предыдущему примеру, сначала создаётся битовая карта по индексу, а потом на её основе извлекаются данные:
	
	|QUERY PLAN|
	|----------|
	|Bitmap Heap Scan on circuits c  (cost=13.21..17.73 rows=1 width=118)|
	|  Recheck Cond: (to_tsvector('english'::regconfig, (name)::text) @@ to_tsquery('speedway &#124; raceway'::text))|
	|  ->  Bitmap Index Scan on circuits_name_idx  (cost=0.00..13.21 rows=1 width=0)|
	|        Index Cond: (to_tsvector('english'::regconfig, (name)::text) @@ to_tsquery('speedway &#124; raceway'::text))|

# Частичный индекс
- Создаём индекс на часть таблицы pit_stops, указав предикат lap > 40:
	```sql
	CREATE INDEX  pit_stops_lap_idx on f1.pit_stops (lap) WHERE lap > 40;
	```
	
	```sql
	SELECT * FROM  f1.pit_stops ps
	WHERE ps.lap > 40;
	```
	
	При указании в условии отбора lap > 40 индекс используется:
	
	|QUERY PLAN|
	|----------|
	|Bitmap Heap Scan on pit_stops ps  (cost=13.47..83.15 rows=1014 width=35)|
	|  Recheck Cond: (lap > 40)|
	|  ->  Bitmap Index Scan on pit_stops_lap_idx  (cost=0.00..13.22 rows=1014 width=0)|
	
	
	При указании в условии отбора lap > 20 идёт последовательное сканирование:
	
	|QUERY PLAN|
	|----------|
	|Seq Scan on pit_stops ps  (cost=0.00..141.84 rows=3906 width=35)|
	|  Filter: (lap > 20)|

# Индекс на поле с функцией
- Запрос по полному имени пилота без индекса:
	```sql
	SELECT * FROM f1.drivers d
	WHERE lower(d.forename || ' ' || d.surname) = 'takuma sato' 
	```

	|QUERY PLAN|
	|----------|
	|Seq Scan on drivers d  (cost=0.00..30.88 rows=4 width=90)|
	|  Filter: (lower((((forename)::text &#124;&#124; ' '::text) &#124;&#124; (surname)::text)) = 'takuma sato'::text)|
	
- Создаём индекс на функцию:
	```sql
	CREATE INDEX  drivers_fullname_idx on f1.drivers(lower(forename || ' ' || surname));
	```
- Отбор идёт по индексу:

	|QUERY PLAN|
	|----------|
	|Bitmap Heap Scan on drivers d  (cost=4.31..13.97 rows=4 width=90)|
	|  Recheck Cond: (lower((((forename)::text &#124;&#124; ' '::text) &#124;&#124; (surname)::text)) = 'takuma sato'::text)|
	|  ->  Bitmap Index Scan on drivers_fullname_idx  (cost=0.00..4.31 rows=4 width=0)|
	|        Index Cond: (lower((((forename)::text &#124;&#124; ' '::text) &#124;&#124; (surname)::text)) = 'takuma sato'::text)|

# Индекс на несколько полей
- Отбираем результаты гонок по гонке и пилоту без индекса:
	```sql
	SELECT * FROM f1.results r
	WHERE r.race_id  = 18 AND r.driver_id = 10;
	```
	
	|QUERY PLAN|
	|----------|
	|Seq Scan on results r  (cost=0.00..657.95 rows=1 width=86)|
	|  Filter: ((race_id = 18) AND (driver_id = 10))|
	
- Создаём составной индекс:
	```sql
	CREATE INDEX results_race_driver_idx ON f1.results (race_id, driver_id);
	```
- Смотрим план и видим, что используется непосредственно доступ по индексу (Index Scan):

	|QUERY PLAN|
	|----------|
	|Index Scan using results_race_driver_idx on results r  (cost=0.29..8.31 rows=1 width=86)|
	|  Index Cond: ((race_id = 18) AND (driver_id = 10))|
	
- Если отбирать только поля, входящие в индекс, то можно добиться, чтобы использовался Index Only Scan, который получает данные непосредственно из индекса без обращения к основной таблице:
	```sql
	EXPLAIN 
	SELECT r.race_id, r.driver_id FROM f1.results r
	WHERE r.race_id  = 18 AND r.driver_id in (10, 11)
	```
	
	|QUERY PLAN|
	|----------|
	|Index Only Scan using results_race_driver_idx on results r  (cost=0.29..8.60 rows=1 width=8)|
	|  Index Cond: ((race_id = 18) AND (driver_id = ANY ('{10,11}'::integer[])))|

	
# Создание комментариев к индексам	
- Создание
	```sql
	COMMENT ON INDEX f1.results_race_driver_idx  IS 'Индекс по гонке и пилоту';
	COMMENT ON INDEX f1.drivers_fullname_idx  IS 'Индекс по полному имени пилота в нижнем регистре';
	COMMENT ON INDEX f1.pit_stops_lap_idx  IS 'Частичный индекс по кругу, на котором был сделан питстоп';
	COMMENT ON INDEX f1.circuits_name_idx  IS 'Индекс для полнотекстового поиска по названию гоночного трека';
	COMMENT ON INDEX f1.races_date_idx  IS 'Индекс по дате гонки';
	```	
- Просмотр
	```sql
	SELECT obj_description('f1.results_race_driver_idx'::regclass, 'pg_class');
	```
	|obj_description|
	|---------------|
	|Индекс по гонке и пилоту|







