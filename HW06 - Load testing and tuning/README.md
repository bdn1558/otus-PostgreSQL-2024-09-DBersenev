#  Нагрузочное тестирование и тюнинг PostgreSQL
- Создаём новую ВМ Ubuntu 22.04 LTS с параметрами:
	- vCPU - 4
	- RAM - 12 ГБ
	- Объём дискового пространства (HDD) - 15 ГБ 
	```sh
	yc compute instance create --name pg-vm4 --hostname pg-vm4 --cores 4 --memory 12 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2204-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	yc compute ssh --name pg-vm4 --identity-file yc_key --login yc-user
	```
- Устанавливаем  PostreSQL 17
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-17
	```
- Инициализируем pgbench в БД postgres:
	```sh
	sudo su postgres
	```
	```sh
	pgbench -i postgres
	```

- Запускаем тест с дефолтными настройками:
	```sh
	postgres@pg-vm4:/home/yc-user$ pgbench -c 50 -j 2 -P 10 -T 60 postgres
	```
	```
	pgbench (17.0 (Ubuntu 17.0-1.pgdg22.04+1))
	starting vacuum...end.
	progress: 10.0 s, 345.6 tps, lat 141.770 ms stddev 150.930, 0 failed
	progress: 20.0 s, 387.6 tps, lat 129.324 ms stddev 158.081, 0 failed
	progress: 30.0 s, 368.9 tps, lat 134.262 ms stddev 156.375, 0 failed
	progress: 40.0 s, 428.9 tps, lat 116.926 ms stddev 142.314, 0 failed
	progress: 50.0 s, 336.5 tps, lat 148.107 ms stddev 161.734, 0 failed
	progress: 60.0 s, 364.1 tps, lat 137.743 ms stddev 160.827, 0 failed
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 50
	number of threads: 2
	maximum number of tries: 1
	duration: 60 s
	number of transactions actually processed: 22366
	number of failed transactions: 0 (0.000%)
	latency average = 134.141 ms
	latency stddev = 155.365 ms
	initial connection time = 71.063 ms
	tps = 371.892577 (without initial connection time)
	```

- Генерируем рекомендуемые настройки с помощью PGCоnfig (https://www.pgconfig.org):
	```
	# Memory Configuration
	shared_buffers = 3GB
	effective_cache_size = 9GB
	work_mem = 31MB
	maintenance_work_mem = 614MB

	# Checkpoint Related Configuration
	min_wal_size = 2GB
	max_wal_size = 3GB
	checkpoint_completion_target = 0.9
	wal_buffers = 16MB

	# Network Related Configuration
	max_connections = 100

	# Storage Configuration
	random_page_cost = 4.0
	effective_io_concurrency = 2

	# Worker Processes Configuration
	max_worker_processes = 8
	max_parallel_workers_per_gather = 2
	max_parallel_workers = 2
	```
- Вставляем их в конец postgresql.conf и перезагружаем кластер:
	```sh
	sudo nano /etc/postgresql/17/main/postgresql.conf
	sudo pg_ctlcluster 17 main restart
	```
	
- Запускаем тест с новыми настройками PostreSQL:
	```sh
	postgres@pg-vm4:/home/yc-user$ pgbench -c 50 -j 2 -P 10 -T 60 postgres
	```
	```
	pgbench (17.0 (Ubuntu 17.0-1.pgdg22.04+1))
	starting vacuum...end.
	progress: 10.0 s, 401.7 tps, lat 122.088 ms stddev 139.049, 0 failed
	progress: 20.0 s, 458.1 tps, lat 109.270 ms stddev 130.680, 0 failed
	progress: 30.0 s, 379.6 tps, lat 131.777 ms stddev 151.246, 0 failed
	progress: 40.0 s, 317.6 tps, lat 157.340 ms stddev 181.850, 0 failed
	progress: 50.0 s, 285.9 tps, lat 174.075 ms stddev 204.201, 0 failed
	progress: 60.0 s, 443.1 tps, lat 113.819 ms stddev 127.536, 0 failed
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 50
	number of threads: 2
	maximum number of tries: 1
	duration: 60 s
	number of transactions actually processed: 22910
	number of failed transactions: 0 (0.000%)
	latency average = 130.934 ms
	latency stddev = 155.035 ms
	initial connection time = 70.706 ms
	tps = 381.560225 (without initial connection time)
	```
	tps вырос, но незначительно.
	
- Увеличить tps получилось через совокупное изменение следующих параметров сверх рекомендуемых:
	```
	shared_buffers = 9GB
	effective_cache_size = 12GB
	work_mem = 256MB
	```

	```sh
	postgres@pg-vm4:/home/yc-user$ pgbench -c 50 -j 2 -P 10 -T 60 postgres
	```
	```
	pgbench (17.0 (Ubuntu 17.0-1.pgdg22.04+1))
	starting vacuum...end.
	progress: 10.0 s, 464.4 tps, lat 105.108 ms stddev 124.531, 0 failed
	progress: 20.0 s, 429.0 tps, lat 117.494 ms stddev 141.606, 0 failed
	progress: 30.0 s, 375.7 tps, lat 131.915 ms stddev 154.856, 0 failed
	progress: 40.0 s, 476.8 tps, lat 105.764 ms stddev 121.111, 0 failed
	progress: 50.0 s, 520.4 tps, lat 96.200 ms stddev 109.437, 0 failed
	progress: 60.0 s, 430.7 tps, lat 115.896 ms stddev 136.264, 0 failed
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 50
	number of threads: 2
	maximum number of tries: 1
	duration: 60 s
	number of transactions actually processed: 27020
	number of failed transactions: 0 (0.000%)
	latency average = 110.973 ms
	latency stddev = 131.053 ms
	initial connection time = 71.172 ms
	tps = 450.223966 (without initial connection time)
	```
	
	- shared_buffers - количество памяти, выделенной PostgreSQL для кеширования данных. Увеличение улучшает производительность, так как уменьшает количество обращений к диску, но может привести к перерасходу оперативной памяти. Например, при значении равном размеру ОП (12GB), кластер перестаёт запускаться.
	- effective_cache_size -  оценочный объем памяти, которая доступна для кэширования данных. Чем больше значение, тем больше вероятность, что оптимизатор будет использовать индексы.
	- work_mem - количество памяти, которое БД использует для сортировки данных или выполнения операций хеш-соединения. Так как указанный объём выделяется на каждую операцию, то его чрезмерное увеличение может привести к увеличению потребления оперативной памяти и превышения физически доступной.
	
- Наибольший прирост даёт изменение параметра synchronous_commit:
	```
	synchronous_commit = off
	```
	```sh
	postgres@pg-vm4:/home/yc-user$ pgbench -c 50 -j 2 -P 10 -T 60 postgres
	```
	```
	pgbench (17.0 (Ubuntu 17.0-1.pgdg22.04+1))
	starting vacuum...end.
	progress: 10.0 s, 3141.5 tps, lat 15.773 ms stddev 13.917, 0 failed
	progress: 20.0 s, 3165.6 tps, lat 15.792 ms stddev 14.514, 0 failed
	progress: 30.0 s, 3101.2 tps, lat 16.122 ms stddev 14.414, 0 failed
	progress: 40.0 s, 3168.5 tps, lat 15.768 ms stddev 13.853, 0 failed
	progress: 50.0 s, 3124.5 tps, lat 16.000 ms stddev 14.544, 0 failed
	progress: 60.0 s, 3079.4 tps, lat 16.240 ms stddev 14.160, 0 failed
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 50
	number of threads: 2
	maximum number of tries: 1
	duration: 60 s
	number of transactions actually processed: 187857
	number of failed transactions: 0 (0.000%)
	latency average = 15.958 ms
	latency stddev = 14.258 ms
	initial connection time = 76.932 ms
	tps = 3130.522266 (without initial connection time)
	```
	В этом режиме PostgreSQL не ожидает подтверждения записи на диск после завершения транзакции. Данные могут быть успешно записаны в журнал транзакций и кэшированы в памяти сервера, но фактическая запись на диск может произойти позже, что повышает производительность, но может привести к потере данных при сбое системы. Особенно это заметно по резко выросшему количеству обработанных транзакции.
