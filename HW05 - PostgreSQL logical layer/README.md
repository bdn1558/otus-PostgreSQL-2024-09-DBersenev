# Логический уровень PostgreSQL
- По аналогии с предыдущими домашними заданиями создаём ВМ c Ubuntu 22.04 в Yandex Cloud и устанавливаем PostreSQL 17:
	```sh 
	pg_lsclusters
		Ver Cluster Port Status Owner    Data directory              Log file
		17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
	```
- Запускаем psql под пользователем postgres:
	```sh
	sudo -u postgres psql
	```
- Создаём новую базу данных testdb:
	```sql
	CREATE DATABASE testdb;
	```
- Заходим в созданную базу данных под пользователем postgres:
	```sql
	\c testdb
	You are now connected to database "testdb" as user "postgres".
	```
- Создаём новую схему testnm:
	```sql
	CREATE SCHEMA testnm;
	```
- Создаём новую таблицу t1 с одной колонкой c1 типа integer:
	```sql
	CREATE TABLE testnm.t1(c1 integer);
	```

- Вставляем строку со значением c1=1:
	```sql
	INSERT INTO testnm.t1 values(1);
	INSERT 0 1
	```
- Создаём новую роль readonly и выдаём ей права:
	```sql
	CREATE ROLE readonly;
	GRANT CONNECT ON DATABASE testdb TO readonly;
	GRANT USAGE ON SCHEMA testnm TO readonly;
	GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
	```
- Создаём пользователя testread с паролем test123 и выдаём ему роль readonly:
	```sql
	CREATE USER testread WITH PASSWORD 'test123';
	GRANT readonly TO testread;
	```
- Заходим под пользователем testread в базу данных testdb и получаем ошибку:
	```sh
	\c testdb testread;
	connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
	```
-  Меняем метод аутентификации для локальных подключений, чтобы можно было использовать вход с паролем	
    ```sh 
    sudo nano /etc/postgresql/17/main/pg_hba.conf
	# TYPE  DATABASE        USER            ADDRESS                 METHOD
	local   all             all                                     md5
    ```
- Пробуем подключиться после рестарта кластера:
	```sh
	psql -U testread -d testdb
	Password for user testread:
	psql (17.0 (Ubuntu 17.0-1.pgdg22.04+1))	
	```
- Выполняем запрос и получаем ошибку: 
	```sql
	select * from t1;
	ERROR:  relation "t1" does not exist
	LINE 1: select * from t1;
	```
- Ошибка появляется из-за того, что таблица была создана с указанием схемы testnm, которая не входит в search_path, поэтому система не может её найти без явного указания в запросе. В данном случае таблица ищется сначала в схеме с именем текущего пользователя, которая не создавалась, а потом в схеме public:
	```sql
	SHOW search_path;
	   search_path
	-----------------
	 "$user", public
	(1 row)
	```
	
- С указанием схемы в запросе он выполняется без ошибки:
	```sql
	select * from testnm.t1;
	 c1
	----
	  1
	(1 row)
	```
- Можно добавить схему в search_path и тогда к таблице можно будет обращаться без указания схемы:
	```sql
	SET search_path TO testnm, public;
	```
	
	```sql
	select * from t1;
	 c1
	----
	  1
	(1 row)
	```
- Удаляем таблицу t1 и создаём заново в схеме testnm с одной колонкой c1 типа integer:
	```sh
	sudo -u postgres psql testdb
	```
	```sql
	DROP TABLE testnm.t1;
	CREATE TABLE testnm.t1(c1 integer);
	INSERT INTO testnm.t1 values(1);
	```
- Снова пытаемся выполнять запрос под пользователем testread. Получаем ошибку, так как GRANT SELECT ON ALL TABLES выдаёт права на все существующие на тот момент таблицы, а t1 была пересоздана:
	```sql
	\c testdb testread;
	select * from testnm.t1;
	ERROR:  permission denied for table t1
	```
-  Чтобы этого избежать нужно добавить привилегии по умолчанию, которые автоматически будут добавляться к роли readonly при создании таблиц в схеме testnm:
	```sh
	sudo -u postgres psql testdb
	```
	```sql
	ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
	```
- Однако после этого снова получаем ошибку, потому что привилегии по умолчанию действуют только на новые объекты, а таблица t1 уже была создана к моменту их изменения. 
	```sql
	\c testdb testread;
	select * from testnm.t1;
	ERROR:  permission denied for table t1
	```
- Чтобы доступ к t1 появился надо снова выполнить GRANT SELECT ON ALL TABLES или пересоздать её:
	```sql
	select * from testnm.t1;
	 c1
	----
	  1
	(1 row)
	```
- Пробуем создать таблицу t2 без указания схемы. Получаем ошибку, так как начиная с 15 версии у пользователей по умолчанию отозваны права на CREATE в схеме public:
	>PostgreSQL 15 also revokes the CREATE permission from all users except a database owner from the public (or default) schema.
	
	```sql
	create table t2(c1 integer);
	ERROR:  permission denied for schema public
	LINE 1: create table t2(c1 integer);
	```
- Чтобы вернуть права на создание таблиц в схеме public можно (но не нужно) выполнить:
	```sh
	sudo -u postgres psql testdb
	```
	```sql
	GRANT CREATE on SCHEMA public TO public;
	```
	```sh
	psql -U testread -d testdb
	```
	```sql
	create table t2(c1 integer);
	```
	```sh
	\dt
	 Schema | Name | Type  |  Owner
	--------+------+-------+----------
	 public | t2   | table | testread
	```
	
		