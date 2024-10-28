## Cоздание ВМ c Ubuntu 20.04 на Yandex Cloud
1) Устанавливаем OpenSSH для Windows из дистрибутива https://opensshwindows.com
2) Создаём открытый и закрытый ключ:
    ```sh
    ssh-keygen -t ed25519
    ```
3) Устанавливаем Интерфейс командной строки Yandex Cloud (CLI)  для PowerShell
    ```sh
    iex (New-Object System.Net.WebClient).DownloadString('https://storage.yandexcloud.net/yandexcloud-yc/install.ps1')
    ```
4) Создаём сетевую инфраструктуру через CLI
    ```sh
    yc vpc network create --name otus-net --description "otus-net"
    yc vpc subnet create --name otus-subnet --range 192.168.0.0/24 --network-name otus-net --description "otus-subnet"
    ```
5) Создаём ВМ
    ```sh
    yc compute instance create --name otus-vm --hostname otus-vm --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
    ```
6) Проверяем, что созданная ВМ есть в списке в статусе Running:
    ```sh
    yc compute instance list
    ```
    ```
    +----------------------+---------+---------------+---------+--------------+--------------+
    |          ID          |  NAME   |    ZONE ID    | STATUS  | EXTERNAL IP  | INTERNAL IP  |
    +----------------------+---------+---------------+---------+--------------+--------------+
    | xxxxxxxxxxxxxххххххх | otus-vm | ru-central1-a | RUNNING | 89.169.130.x | 192.168.0.32 |
    +----------------------+---------+---------------+---------+--------------+--------------+
    ```
7) Подключаемся к созданной ВМ через CLI с использованием ключа:
    ```sh 
    yc compute ssh --id xxxxxxxxxxxxxххххххх --identity-file yc_key --login yc-user
    ```
## Установка PostgreSQL 17
1) 	Обновляем список доступных пакетов и обновляем установленные:
	```sh 
	sudo apt update && sudo apt upgrade -y
	```
2) Добавляем репозиторий PostgreSQL 17:
    ```sh 
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    ```
3) Добавляем ключ репозитория:
    ``` 
	curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
	```
4) Обновляем список доступных пакетов
	```sh 
	sudo apt update && sudo apt upgrade -y
	```
5) Устанавливаем PostgreSQL 17
    ```sh
    sudo apt -y install postgresql-17
    ``` 
6) Проверяем статус кластера:
	```sh 
	pg_lsclusters
		Ver Cluster Port Status Owner    Data directory              Log file
		17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
	```
7) Устанавливаем пароль для postgres:
    ```sh 
    sudo -u postgres psql
    \password   ******
    \q
    ```
8) Настраиваем сетевую доступность:
    ```sh 
    cd /etc/postgresql/7/main/
    sudo nano /etc/postgresql/17/main/postgresql.conf
    listen_addresses = '*'
    ```
    
    ```sh 
    sudo nano /etc/postgresql/17/main/pg_hba.conf
    host    all             all             0.0.0.0/0               scram-sha-256 
    ```
9) Рестарт кластера
    ```sh
    sudo pg_ctlcluster 17 main restart
    ```
## Работа с уровнями изоляции
1) Переключаем dBeaver в режим Режим транзакций (ручной коммит)
2) Создаём первую сессию (новый SQL-скрипт)
3) Создаём вторую сессию (новый SQL-скрипт)
4) В первой сессии создаём таблицу и наполняем её данными:
     ```sql
    create table persons(id serial, first_name text, second_name text); 
    insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
    insert into persons(first_name, second_name) values('petr', 'petrov'); 
    commit;
     ```
5) Проверяем текущий уровень изоляции:
     ```sql
    show transaction isolation level
     ```
    |transaction_isolation|
    |---------------------|
    |read committed|
6) Начинаем в обоих сессиях новую транзакцию:
     ```sql
    begin;
     ```
7) В первой сессии добавляем новую запись:
     ```sql
    insert into persons(first_name, second_name) values('sergey', 'sergeev');
    ```
8) Во второй сессии делаем:
    ```sql
    select * from persons
    ```
    |id|first_name|second_name|
    |--|----------|-----------|
    |1|ivan|ivanov|
    |2|petr|petrov|
	
    Новая запись не видна, так как изменения в первой сессии не были зафиксированы через commit, а уровень изоляции Read Committed не допускает грязное чтение. 
9) Завершаем транзакцию в первой сессии:
     ```sql
    commit;
     ```
10) Во второй сессии делаем:
    ```sql
    select * from persons
    ```
    |id|first_name|second_name|
    |--|----------|-----------|
    |1|ivan|ivanov|
    |2|petr|petrov|
    |3|sergey|sergeev|
	
    Новая запись теперь видна, так как транзакция в первой сессии была зафиксирована, а уровень изоляции Read Committed допускает неповторяющееся чтение
11) Завершаем транзакцию во второй сессии:
     ```sql
    commit;
     ```
12) Устанавливаем в обоих сессиях уровень изоляции Repeatable Read и начинаем новые транзакции:
     ```sql
    set transaction isolation level repeatable read;
    begin;
     ```
13) В первой сессии добавляем новую запись:
     ```sql
    insert into persons(first_name, second_name) values('sveta', 'svetova');
    ```
14) Во второй сессии делаем:
    ```sql
    select * from persons
    ```
    |id|first_name|second_name|
    |--|----------|-----------|
    |1|ivan|ivanov|
    |2|petr|petrov|
    |3|sergey|sergeev|
	
    Новая запись не видна, так как изменения в первой сессии не были зафиксированы.
15) Завершаем транзакцию в первой сессии:
     ```sql
    commit;
     ``` 
14) Во второй сессии снова делаем:
    ```sql
    select * from persons
    ```
    |id|first_name|second_name|
    |--|----------|-----------|
    |1|ivan|ivanov|
    |2|petr|petrov|
    |3|sergey|sergeev|
	
    Новая запись всё равно не видна, так как уровень изоляции Repeatable Read не допускает неповторяющееся чтение и вторая сессия видит только те данные, которые были зафиксированы до начала транзакции.   
15) Завершаем транзакцию во второй сессии:
     ```sql
    commit;
     ```
14) Во второй сессии снова делаем:
    ```sql
    select * from persons
    ```
    |id|first_name|second_name|
    |--|----------|-----------|
    |1|ivan|ivanov|
    |2|petr|petrov|
    |3|sergey|sergeev|
    |4|sveta|svetova|
	
    Новая запись теперь видна, так как транзакции в обоих сессиях были завершены.