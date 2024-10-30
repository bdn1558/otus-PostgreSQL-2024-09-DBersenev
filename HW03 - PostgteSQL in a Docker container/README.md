## Установка Docker на Ubuntu
1) Запускаем ранее созданную ВМ
	```sh
	yc compute instance start otus-vm
	```
2) Подключаемся к созданной ВМ через CLI с использованием ключа:
	```sh
	yc compute ssh --id  xxxxxxxxxxxxxххххххх --identity-file yc_key --login yc-user
	Enter passphrase for key 'yc_key':
	Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-198-generic x86_64)
	```
3) Обновляем репозиторий пакетов:
	```sh 
	sudo apt update && sudo apt upgrade -y
	```
4) Установка необходимых зависимостей:
	```sh 
	sudo apt install apt-transport-https ca-certificates curl software-properties-common
	```
5) Устанавливаем GPG ключ для репозитория Docker:
	```sh
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
	```
6) Добавляем репозиторий Docker:
	```sh
	sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
	```
7) Устанавливаем Docker:
	```sh
	sudo apt update
	sudo apt install docker-ce	
	```
8) Проверяем статус:
	```sh
	sudo systemctl status docker
	● docker.service - Docker Application Container Engine
		 Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset>
		 Active: active (running) since Tue 2024-10-29 17:42:19 UTC; 2min 32s ago
	```
	```sh
	sudo docker run hello-world
	Hello from Docker!
	This message shows that your installation appears to be working correctly.

	```
## Установка PostgreSQL в Docker контейнере
1) Создаём каталог:
	```sh
	sudo mkdir /var/lib/postgres
	```
2) Создаем docker-сеть:
	```sh
	sudo docker network create pg-net
	```
3) Создаём контейнер с сервером Postgres:
	```sh
	sudo docker run -d --name postgres_server --network pg-net -e POSTGRES_PASSWORD=135246 -v /var/lib/postgres:/var/lib/postgresql/data -p 5432:5432 postgres
	```
4) Проверяем статус контейнера:
	```sh
	sudo docker ps
	```
	|CONTAINER ID|IMAGE|COMMAND|CREATED|STATUS|PORTS|NAMES|
	|--------------|--------------|--------------|--------------|--------------|--------------|--------------|
	|de46a2dde1f1|postgres|"docker-entrypoint.s…" |2 minutes ago|Up 2 minutes|0.0.0.0:5432->5432/tcp, :::5432->5432/tcp|postgres_server|
5) Запускаем отдельный контейнер с клиентом:
	```sh
	sudo docker run -it --rm --name postgres_client --network pg-net --link postgres_server:postgres postgres psql -h postgres -U postgres
	```
6) Создаём таблицу и наполняем данными:
	```sql
	CREATE TABLE phones (id SERIAL PRIMARY KEY, number varchar(10));
	INSERT INTO phones (number) VALUES ('9324567854'), ('9567854525'), ('9297894523'), ('9834565599'), ('9474523987');
	SELECT * FROM phones;

	id |   number
	----+------------
	  1 | 9324567854
	  2 | 9567854525
	  3 | 9297894523
	  4 | 9834565599
	  5 | 9474523987
	(5 rows)
	```
7) Устанавливаем nano, так как текстовые редакторы в контейнере отсутствуют:
	```sh	
	sudo docker exec -it postgres_server bash
	apt-get update
	apt-get install nano
	```
8) Редактируем конфиги Postgres, чтобы открыть сетевой доступ:
	```sh
	nano /var/lib/postgresql/data/postgresql.conf
	listen_addresses = '*'
	```
	```sh
	nano /var/lib/postgresql/data/pg_hba.conf
	# IPv4 local connections:
	host    all             all             0.0.0.0/0            trust
	```
9) Делаем рестарт контейнера:
	```sh
	sudo docker restart postgres_server
	```
10) С локальной машины создаём новое соединение с Postgres в dBeaver на IP ВМ, порт 5432 и с данными УЗ, указанными при создании контейнера. Проверяем наличие данных в таблице:
	```sql
	SELECT * FROM phones;
	```
	|id|number|
	|--|------|
	|1|9324567854|
	|2|9567854525|
	|3|9297894523|
	|4|9834565599|
	|5|9474523987|
11) Останавливаем и удаляем контейнер 
	```sh
	sudo docker stop postgres_server 
	sudo docker rm postgres_server
	```
12) Создаем контейнер заново:
	```sh
	sudo docker run -d --name postgres_server --network pg-net -e POSTGRES_PASSWORD=pg135246 -v /var/lib/postgres:/var/lib/postgresql/data -p 5432:5432 postgres
	```
13) Подключаемся к контейнеру с клиентом и проверяем, что данные в таблице остались 
	```sh
	sudo docker run -it --rm --name postgres_client --network pg-net --link postgres_server:postgres postgres psql -h postgres -U postgres
	```

	```sql
	SELECT * FROM phones;

	id |   number
	----+------------
	  1 | 9324567854
	  2 | 9567854525
	  3 | 9297894523
	  4 | 9834565599
	  5 | 9474523987
	(5 rows)
	```