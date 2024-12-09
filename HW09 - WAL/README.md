# Журналы и контрольные точки
- Создаём новую ВМ Ubuntu 22.04 LTS:
	```sh
	yc compute instance create --name pg-vm8 --hostname pg-vm8 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	yc compute ssh --name pg-vm8 --identity-file yc_key --login yc-user
	```
	
- Устанавливаем PostgreSQL 15:
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-15
	```
	
	```sh
	yc-user@pg-vm8:~$ pg_lsclusters
	Ver Cluster Port Status Owner    Data directory              Log file
	15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
	```
- Проверяем текущие настройки контрольных точек:
	```sql
	show checkpoint_timeout;
	```

	|checkpoint_timeout|
	|------------------|
	|5min|
	
	```sql
	show log_checkpoints;
	```
	
	|log_checkpoints|
	|---------------|
	|on|

- Настраиваем выполнение контрольной точки раз в 30 секунд:
	```sql
	alter system set checkpoint_timeout = 30;
	select pg_reload_conf();
	show checkpoint_timeout;
	```
	|checkpoint_timeout|
	|------------------|
	|30s|

- Создаём БД для тестов:
	```sh
	sudo su postgres
	pgbench -i postgres
	```
	
- Получаем текущий Log Sequence Number:
	```sql
	select pg_current_wal_insert_lsn();
	```
	|pg_current_wal_insert_lsn|
	|-------------------------|
	|0/19781A00|
	
-  Получаем имя файла WAL:
	```sql
	select pg_walfile_name('0/21CF1F8');
	```
	|pg_walfile_name|
	|---------------|
	|000000010000000000000020|
	
-  Общее количество контрольных точек, записанных по времени:
	```sql	
	select checkpoints_timed from pg_stat_bgwriter;
	```
	|checkpoints_timed|
	|-----------------|
	|280|

- Запускаем тест:
	```sh
	postgres@pg-vm8:/home/yc-user$ pgbench -c 4 -j 2 -T 600 postgres
	```	
	
- После выполнения теста получаем следующие значения:

	|pg_current_wal_insert_lsn|
	|-------------------------|
	|0/30AC9010|

	|pg_walfile_name|
	|---------------|
	|000000010000000000000030|
	
	|checkpoints_timed|
	|-----------------|
	|300|

- Размер журнальных файлов между ними: 
	```sql
	select '0/30AC9010'::pg_lsn - '0/19781A00'::pg_lsn as bytes;
	```
	
	|bytes|
	|-----|
	|389 314 064|

	Количество выполненных контрольных точек за 10 минут - 20.
	На одну контрольную точку приходится 389 314 064 \ 20 = 19 465 703,2 ~ 18,6 Мб

- Смотрим время выполнения контрольных точек в логах:
	```
	2024-12-09 14:05:05.128 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:05:32.110 UTC [10248] LOG:  checkpoint complete: wrote 1658 buffers (10.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.888 s, sync=0.025 s, total=26.983 s; sync files=7, longest=0.016 s, average=0.004 s; distance=17072 kB, estimate=18036 kB
	2024-12-09 14:05:35.114 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:06:02.199 UTC [10248] LOG:  checkpoint complete: wrote 1742 buffers (10.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.909 s, sync=0.064 s, total=27.086 s; sync files=15, longest=0.050 s, average=0.005 s; distance=18025 kB, estimate=18035 kB
	2024-12-09 14:06:05.202 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:06:32.107 UTC [10248] LOG:  checkpoint complete: wrote 1729 buffers (10.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.817 s, sync=0.014 s, total=26.906 s; sync files=6, longest=0.014 s, average=0.003 s; distance=17339 kB, estimate=17965 kB
	2024-12-09 14:06:35.110 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:07:02.149 UTC [10248] LOG:  checkpoint complete: wrote 1812 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.915 s, sync=0.020 s, total=27.039 s; sync files=14, longest=0.010 s, average=0.002 s; distance=18902 kB, estimate=18902 kB
	2024-12-09 14:07:05.151 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:07:32.053 UTC [10248] LOG:  checkpoint complete: wrote 1747 buffers (10.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.810 s, sync=0.028 s, total=26.902 s; sync files=8, longest=0.021 s, average=0.004 s; distance=18266 kB, estimate=18839 kB
	2024-12-09 14:07:35.056 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:08:02.137 UTC [10248] LOG:  checkpoint complete: wrote 1840 buffers (11.2%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.909 s, sync=0.048 s, total=27.082 s; sync files=15, longest=0.023 s, average=0.004 s; distance=18966 kB, estimate=18966 kB
	2024-12-09 14:08:05.140 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:08:32.070 UTC [10248] LOG:  checkpoint complete: wrote 1754 buffers (10.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.873 s, sync=0.014 s, total=26.930 s; sync files=10, longest=0.008 s, average=0.002 s; distance=19023 kB, estimate=19023 kB
	2024-12-09 14:08:35.073 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:09:02.101 UTC [10248] LOG:  checkpoint complete: wrote 1805 buffers (11.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.915 s, sync=0.041 s, total=27.028 s; sync files=14, longest=0.031 s, average=0.003 s; distance=18831 kB, estimate=19004 kB
	2024-12-09 14:09:05.104 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:09:32.075 UTC [10248] LOG:  checkpoint complete: wrote 1721 buffers (10.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.807 s, sync=0.007 s, total=26.972 s; sync files=10, longest=0.004 s, average=0.001 s; distance=18396 kB, estimate=18943 kB
	2024-12-09 14:09:35.079 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:10:02.103 UTC [10248] LOG:  checkpoint complete: wrote 1812 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.857 s, sync=0.053 s, total=27.025 s; sync files=13, longest=0.037 s, average=0.005 s; distance=19046 kB, estimate=19046 kB
	2024-12-09 14:10:05.106 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:10:32.094 UTC [10248] LOG:  checkpoint complete: wrote 1706 buffers (10.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.922 s, sync=0.010 s, total=26.989 s; sync files=9, longest=0.006 s, average=0.002 s; distance=18093 kB, estimate=18951 kB
	2024-12-09 14:10:35.097 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:11:02.182 UTC [10248] LOG:  checkpoint complete: wrote 1770 buffers (10.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.897 s, sync=0.074 s, total=27.086 s; sync files=13, longest=0.046 s, average=0.006 s; distance=18900 kB, estimate=18946 kB
	2024-12-09 14:11:05.186 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:11:32.077 UTC [10248] LOG:  checkpoint complete: wrote 1725 buffers (10.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.812 s, sync=0.032 s, total=26.892 s; sync files=10, longest=0.017 s, average=0.004 s; distance=18177 kB, estimate=18869 kB
	2024-12-09 14:11:35.080 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:12:02.176 UTC [10248] LOG:  checkpoint complete: wrote 1754 buffers (10.7%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.990 s, sync=0.023 s, total=27.096 s; sync files=9, longest=0.023 s, average=0.003 s; distance=18816 kB, estimate=18864 kB
	2024-12-09 14:12:05.179 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:12:32.073 UTC [10248] LOG:  checkpoint complete: wrote 1695 buffers (10.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.830 s, sync=0.017 s, total=26.894 s; sync files=9, longest=0.011 s, average=0.002 s; distance=18277 kB, estimate=18805 kB
	2024-12-09 14:12:35.074 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:13:02.083 UTC [10248] LOG:  checkpoint complete: wrote 1900 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.834 s, sync=0.019 s, total=27.009 s; sync files=14, longest=0.014 s, average=0.002 s; distance=18092 kB, estimate=18734 kB
	2024-12-09 14:13:05.086 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:13:32.052 UTC [10248] LOG:  checkpoint complete: wrote 1736 buffers (10.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.825 s, sync=0.016 s, total=26.966 s; sync files=10, longest=0.010 s, average=0.002 s; distance=17908 kB, estimate=18651 kB
	2024-12-09 14:13:35.055 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:14:02.113 UTC [10248] LOG:  checkpoint complete: wrote 1737 buffers (10.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.798 s, sync=0.031 s, total=27.059 s; sync files=11, longest=0.031 s, average=0.003 s; distance=18588 kB, estimate=18645 kB
	2024-12-09 14:14:05.116 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:14:32.081 UTC [10248] LOG:  checkpoint complete: wrote 1732 buffers (10.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.891 s, sync=0.015 s, total=26.966 s; sync files=8, longest=0.008 s, average=0.002 s; distance=18662 kB, estimate=18662 kB
	2024-12-09 14:15:35.145 UTC [10248] LOG:  checkpoint starting: time
	2024-12-09 14:16:02.075 UTC [10248] LOG:  checkpoint complete: wrote 1151 buffers (7.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.818 s, sync=0.015 s, total=26.931 s; sync files=15, longest=0.011 s, average=0.001 s; distance=15925 kB, estimate=18388 kB
	```
	
	Контрольные точки выполнялись примерно каждые 27 секунд. Так происходит из-за того, что значение параметра checkpoint_completion_target установлено в 0,9:
	```sql
	show checkpoint_completion_target;
	```
	|checkpoint_completion_target|
	|----------------------------|
	|0.9|
	
	Этот параметр определяет насколько быстро должная завершиться контрольная точке перед созданием предыдущей, чтобы более равномерно распределить процесс записи на диск. Так как checkpoint_timeout = 30 сек, то умножив его на 0,9 как раз и получаем искомые 27 сек.
	
# Синхронный и асинхронный режим
- Проверяем текущие настройки:
	```sql
	SHOW synchronous_commit;
	```
	|synchronous_commit|
	|------------------|
	|on|
	
	Кластер работает в синхронном режиме.
	
- Запускаем тест:
	```sh
	postgres@pg-vm8:/home/yc-user$ pgbench -c 4 -j 2 -T 600 postgres
	pgbench (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
	starting vacuum...end.
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 4
	number of threads: 2
	maximum number of tries: 1
	duration: 600 s
	number of transactions actually processed: 207871
	number of failed transactions: 0 (0.000%)
	latency average = 11.546 ms
	initial connection time = 10.333 ms
	tps = 346.447694 (without initial connection time)
	```	
	
- Настраиваем асинхронный режим, перезапускаем кластер и проверяем настройки:
	```sh
	sudo nano /etc/postgresql/15/main/postgresql.conf
	```
	
	```
	# Add settings for extensions here
	synchronous_commit = off
	```
	
	```sh
	sudo pg_ctlcluster 15 main restart
	```
	
	```sql
	SHOW synchronous_commit;
	```
	
	|synchronous_commit|
	|------------------|
	|off|
	
- Запускаем тест в асинхронном режиме
	```sh
	postgres@pg-vm8:/home/yc-user$ pgbench -c 4 -j 2 -T 600 postgres
	pgbench (15.10 (Ubuntu 15.10-1.pgdg24.04+1))
	starting vacuum...end.
	transaction type: <builtin: TPC-B (sort of)>
	scaling factor: 1
	query mode: simple
	number of clients: 4
	number of threads: 2
	maximum number of tries: 1
	duration: 600 s
	number of transactions actually processed: 1531318
	number of failed transactions: 0 (0.000%)
	latency average = 1.567 ms
	initial connection time = 10.237 ms
	tps = 2552.223682 (without initial connection time)
	```
	
	tps вырос в несколько раз, так как асинхронном режиме PostgreSQL не ожидает подтверждения записи на диск после завершения транзакции. Данные могут быть успешно записаны в журнал транзакций и кэшированы в памяти сервера, но фактическая запись на диск может произойти позже, что повышает производительность, но может привести к потере данных при сбое системы. 

# Контрольная сумма
- Останавливаем кластер
	```sh
	sudo pg_ctlcluster 15 main stop
	```
- Создаём новый с включённой контрольной суммой:
	```sh
	sudo pg_createcluster 15 alt -- --data-checksums
	sudo pg_ctlcluster 15 alt start
	pg_lsclusters
	```
	```
	Ver Cluster Port Status Owner    Data directory              Log file
	15  alt     5433 online postgres /var/lib/postgresql/15/alt  /var/log/postgresql/postgresql-15-alt.log
	15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
	```
- Создаём таблицу и заполняем данными:
	```sql
	CREATE TABLE broken_table (id int, value text);
	INSERT INTO broken_table VALUES (1, 'All work '), (2, 'and no play'), (3, 'makes Jack'), (4, 'a dull boy');
	```

- Местоположение файла с таблицей на диске
	```sql
	SELECT  pg_relation_filepath('broken_table');
	```
	|pg_relation_filepath|
	|--------------------|
	|base/5/16388|

- Останавливаем кластер:
	```sh
	sudo pg_ctlcluster 15 alt stop
	``` 
	
- Меняем несколько первых байт в файле:
	```sh
	sudo hexedit /var/lib/postgresql/15/alt/base/5/16388
	```
- Запускаем кластер:
	```sh
	sudo pg_ctlcluster 15 alt start
	```
- Пытаемся выполнить select и получаем ошибку:
	```sql
	SELECT  * FROM  broken_table;
	```
	
	```
	SQL Error [XX001]: ERROR: invalid page in block 0 of relation base/5/16388
	```
- Пробуем изменить параметр ignore_checksum_failure, позволяющий игнорировать ошибки контрольной суммы:
	```sql
	SET ignore_checksum_failure = on;
	SHOW ignore_checksum_failure;
	```
	|ignore_checksum_failure|
	|-----------------------|
	|on|
	
- Заново выполняем select:
	```sql
	SELECT  * FROM  broken_table;
	```
	|id|value|
	|--|-----|
	|1|All work |
	|2|and no play|
	|3|makes Jack|
	|4|a dull boy|
	
	Запрос успешно выполнился, так как видимо была повреждена некритичная часть страницы, а не сами данные. При этом выводится предупреждение о несовпадении контрольной суммы:
    ```
	page verification failed, calculated checksum 54856 but expected 28908
	```

	
	
