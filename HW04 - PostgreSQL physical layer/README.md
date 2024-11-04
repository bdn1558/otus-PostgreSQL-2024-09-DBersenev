# Физический уровень PostgreSQL
## Создание ВМ
1) Создаём ВМ c Ubuntu 22.04 c использованием существующей настроек сети и ключа
    ```sh
    yc compute instance create --name pg-vm1 --hostname pg-vm1 --cores 2 --memory 4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
    
	yc compute instance list
	+----------------------+--------+---------------+---------+---------------+--------------+
	|          ID          |  NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP  |
	+----------------------+--------+---------------+---------+---------------+--------------+
	| fhmpb0udos6d******** | pg-vm1 | ru-central1-a | RUNNING | 89.169.***.** | 192.168.0.10 |
	+----------------------+--------+---------------+---------+---------------+--------------+
	```
2) Подключаемся к созданной ВМ через CLI с использованием ключа:
    ```sh 
    yc compute ssh --name pg-vm1 --identity-file yc_key --login yc-user
    ```
## Установка PostgreSQL 17
1) Добавляем репозиторий PostgreSQL 17:
    ```sh 
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    ```
2) Добавляем ключ репозитория:
    ``` 
	curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
	```
3) Обновляем список доступных пакетов
	```sh 
	sudo apt update && sudo apt upgrade -y
	```
4) Устанавливаем PostgreSQL 17
    ```sh
    sudo apt -y install postgresql-17
    ``` 
5) Проверяем статус кластера:
	```sh 
	pg_lsclusters
		Ver Cluster Port Status Owner    Data directory              Log file
		17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
	```
6) Устанавливаем пароль для postgres:
    ```sh 
    sudo -u postgres psql
    \password
    \q
    ```
7) Запускаем psql под пользователем postgres
	```sh
	sudo -u postgres psql
	```
8) Создаём таблицу и наполняем данными:
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
9) Останавливаем кластер:
	```sh
	sudo -u postgres pg_ctlcluster 17 main stop
	```
## Создание и монтирование диска	
1) Cоздаём новый диск размером 10GB:
	```sh
	yc compute disk create --name new-disk --size 10 --description "new disk"
	```  
	```
	yc compute disk list
	+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------+
	|          ID          |   NAME   |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP | DESCRIPTION |
	+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------+
	| fhmob3r3a0uf******** | new-disk | 10737418240 | ru-central1-a | READY  |                      |                 | new disk    |
	| fhmth7p1isa8******** |          | 16106127360 | ru-central1-a | READY  | fhmpb0udos6d******** |                 |             |
	+----------------------+----------+-------------+---------------+--------+----------------------+-----------------+-------------+
	```
2) Подключshаем созданный диск к ВМ и проверяем, что он подключён:
	```sh
	yc compute instance attach-disk pg-vm1 --disk-name new-disk --mode rw
	```
6) Проверяем доступные диски, чтобы узнать имя подключённого диска:
	```sh
	sudo fdisk -l
	Disk /dev/vda: 15 GiB, 16106127360 bytes, 31457280 sectors
	Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
	```
7) Разбиваем диск /dev/vdb на разделы через cfdisk:
	```sh
	sudo cfdisk /dev/vdb
	``` 
8) Выбираем тип разметки GPT 
	```
	┌ Select label type ───┐
	│ gpt                  │
	│ dos                  │
	│ sgi                  │
	│ sun                  │
	└──────────────────────┘
	```
9) Выбираем создание нового раздела во Free space c Partition size: 10G:
	```	
		Device               Start           End       Sectors      Size Type
	>>  Free space            2048      20971486      20969439       10G
	```
10) Проверяем и сохраняем изменения через Write => yes:
	```
		Device             Start         End     Sectors    Size Type
	>>  /dev/vdb1           2048    20971486    20969439     10G Linux filesystem
	```
11) Проверяем, что в системе появился новый раздел vdb1 на диске /dev/vdb:
	```sh
	lsblk
	vda    252:0    0    15G  0 disk
	├─vda1 252:1    0     1M  0 part
	└─vda2 252:2    0    15G  0 part /
	vdb    252:16   0    10G  0 disk
	└─vdb1 252:17   0    10G  0 part
	```
12) Форматируем раздел в ext4
	```sh
	sudo mkfs.ext4 /dev/vdb1
	```
13) Создаём папку, в которую будет смонтирован диск:
	```sh
	sudo mkdir -p /mnt/data
	```
14) Настраиваем автоматическое монтирование:
	```sh
	sudo nano /etc/fstab	
	/dev/vdb1     mnt/data     ext4     defaults     0 0
	```	
15) Перегружаем ВМ и проверяем, что диск смонтирован:
	```sh
	df -h -x tmpfs
	Filesystem      Size  Used Avail Use% Mounted on                                                                                                  
	/dev/vda2        15G  4.3G  9.8G  31% /
	/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
	```
16) Делаем postgres владельцем /mnt/data
	```sh
	sudo chown -R postgres:postgres /mnt/data/
	```
17) Переносим содержимое /var/lib/postgres/17 в /mnt/data
	```sh
	mv /var/lib/postgresql/17 /mnt/data
	```
18) Запускаем кластер и получаем ошибку, что каталог с данными недоступен, так как он был перемещён на новый раздел, а значение data_directory в postgresql.conf ссылается на старое местоположение:
	```sh
	sudo -u postgres pg_ctlcluster 17 main start
	Error: /var/lib/postgresql/17/main is not accessible or does not exist

	sudo nano /etc/postgresql/17/main/postgresql.conf
	data_directory = '/var/lib/postgresql/17/main'
	```
20) Изменяем значение на новое:
	```
	data_directory = '/mnt/data/17/main'
	```
21) Запускаем кластер:
	```sh
	sudo -u postgres pg_ctlcluster 17 main start
	```
22) Подключаемся под postgres и проверяем наличие данных в таблице:
	```sh
	sudo -u postgres psql
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
## Задание со звёздочкой
1) Создаём вторую ВМ по аналогии с первой:
	```sh
	yc compute instance list
	+----------------------+--------+---------------+---------+---------------+--------------+
	|          ID          |  NAME  |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP  |
	+----------------------+--------+---------------+---------+---------------+--------------+
	| fhm82pad7edi******** | pg-vm2 | ru-central1-a | RUNNING | 89.169.***.** | 192.168.0.29 |
	| fhmpb0udos6d******** | pg-vm1 | ru-central1-a | RUNNING | 89.169.***.** | 192.168.0.10 |
	+----------------------+--------+---------------+---------+---------------+--------------+
	```
2) Устанавливаем и настраиваем PostgreSQL 17:
	```sh
	pg_lsclusters
	Ver Cluster Port Status Owner    Data directory              Log file
	17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
	```
3) Останавливаем кластер:
	```sh
	sudo -u postgres pg_ctlcluster 17 main stop
	```
4) Удаляем каталог с данными PostgreSQL
	```sh
	sudo rm -R /var/lib/postgresql/17
	```
5) Кластер не запускается:
	```sh
	sudo -u postgres pg_ctlcluster 17 main start
	Error: /var/lib/postgresql/17/main is not accessible or does not exist
	```
6) Останавливаем обе ВМ:
	```sh
	yc compute instance stop pg-vm1
	yc compute instance stop pg-vm2
	```
7) Отключаем диск от первой ВМ и подключаем ко второй:
	```sh
	yc compute instance detach-disk pg-vm1 --disk-name new-disk
	yc compute instance attach-disk pg-vm2 --disk-name new-disk --mode rw
	```
8)  Запускаем вторую ВМ и подключаемся к ней:
	```sh
	yc compute instance start pg-vm2
	yc compute ssh --name pg-vm2 --identity-file yc_key --login yc-user
	```
9) Создаём папку для монтирования диска:
	```sh
	sudo mkdir -p /mnt/data
	```
10) Настраиваем автоматическое монтирование:
	```sh
	sudo nano /etc/fstab	
	/dev/vdb1     mnt/data     ext4     defaults     0 0
	```	
11) Изменяем значение data_directory в postgresql.conf:
	```sh
	sudo nano /etc/postgresql/17/main/postgresql.conf
	data_directory = '/mnt/data/17/main'
	```
12) Делаем postgres владельцем /mnt/data
	```sh
	sudo chown -R postgres:postgres /mnt/data/
	```
13) Рестартуем ВМ:
	```sh
	yc compute instance restart pg-vm2
	```
14) Подключаемся под postgres и проверяем наличие данных в таблице:
	```sh
	sudo -u postgres psql
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