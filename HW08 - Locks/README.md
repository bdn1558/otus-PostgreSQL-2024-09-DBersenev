# Настройка сервера 
- Настраиваем сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд
	```sh
	sudo nano /etc/postgresql/15/main/postgresql.conf
	```
	```
	# - What to Log -
	log_lock_waits = on                     # log lock waits >= deadlock_timeout

	#------------------------------------------------------------------------------
	# LOCK MANAGEMENT
	#------------------------------------------------------------------------------
	deadlock_timeout = 200ms
	```
	```sh
	sudo pg_ctlcluster 15 main restart
	```
- Создаём тестовую таблицу и заполняем данными:
	```sql
	 CREATE TABLE test_table3
	(
		id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
		value int
	);

	INSERT INTO test_table3 (value)
	VALUES (generate_series(1, 10));
	```
- Переключаем в DBeaver в режим транзакций и воспроизводим блокировку:
	Первая сессия (pid=1662):
	```sql
	UPDATE test_table3 SET value = 11 WHERE id = 1;
	```
	Вторая сессия (pid=1661):
	```sql
	UPDATE test_table3 SET value = 12 WHERE id = 1;
	```
- В журнале (/var/log/postgresql/postgresql-15-main.log) видим сообщения о блокировках
	```
	2024-12-08 11:12:58.148 UTC [1661] postgres@postgres LOG:  process 1661 still waiting for ShareLock on transaction 158606 after 200.186 ms
	2024-12-08 11:12:58.148 UTC [1661] postgres@postgres DETAIL:  Process holding the lock: 1662. Wait queue: 1661.
	2024-12-08 11:12:58.148 UTC [1661] postgres@postgres CONTEXT:  while updating tuple (0,15) in relation "test_table3"
	2024-12-08 11:12:58.148 UTC [1661] postgres@postgres STATEMENT:  UPDATE test_table3 SET value = 12 WHERE id = 1
	2024-12-08 11:13:16.936 UTC [1661] postgres@postgres LOG:  process 1661 acquired ShareLock on transaction 158606 after 18988.548 ms
	2024-12-08 11:13:16.936 UTC [1661] postgres@postgres CONTEXT:  while updating tuple (0,15) in relation "test_table3"
	2024-12-08 11:13:16.936 UTC [1661] postgres@postgres STATEMENT:  UPDATE test_table3 SET value = 12 WHERE id = 1
	```
	Вторая сессия (1661) ожидает ожидает получения блокировки типа ShareLock (запрет изменения любых полей строки), которая на текущий момент установлена процессом из первой сессии (1662). После выполнения commit в первой сессии блокировка снимается и вторая сессия успешно обновляет запись.
	
# Обновление одной и той же строки тремя командами UPDATE в разных сеансах
- Моделируем ситуацию:

	- Первая сессия:
	```sql
	SELECT txid_current(), pg_backend_pid();	
	```
	
	|txid_current|pg_backend_pid|
	|------------|--------------|
	|158617|2732|
	
	```sql
	SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
	FROM pg_locks WHERE pid = 2732;
	```
	
	|locktype|relation|virtxid|xid|mode|granted|
	|--------|--------|-------|---|----|-------|
	|relation|test_table3_pkey|||RowExclusiveLock|true|
	|relation|test_table3|||RowExclusiveLock|true|
	|virtualxid||5/25||ExclusiveLock|true|
	|transactionid|||158617|ExclusiveLock|true|


	Транзакция 158617 удерживает блокировку таблицы и индекса (relation) с уровнем RowExclusiveLock, так как изменяет данные, а так же эксклюзивную блокировку своего реального и виртуального номера.

	- Вторая сессия:
	```sql
	SELECT txid_current(), pg_backend_pid();	
	```
	
	|txid_current|pg_backend_pid|
	|------------|--------------|
	|158620|2733|
	
	```sql
	select pg_blocking_pids(2733);
	```
		
	|pg_blocking_pids|
	|----------------|
	|{2732}|

	
	```sql
	SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
	FROM pg_locks WHERE pid = 2733;
	```

	|locktype|relation|virtxid|xid|mode|granted|
	|--------|--------|-------|---|----|-------|
	|relation|test_table3_pkey|||RowExclusiveLock|true|
	|relation|test_table3|||RowExclusiveLock|true|
	|virtualxid||6/26||ExclusiveLock|true|
	|tuple|test_table3|||ExclusiveLock|true|
	|transactionid|||158617|ShareLock|false|
	|transactionid|||158620|ExclusiveLock|true|

	Вторая транзакция аналогично удерживает блокировку таблицы, индекса и своих номеров. Кроме этого она ожидает завершения транзакции из первой сессии (158617), запрашивая блокировку её номера и удерживает блокировку версии строки (tuple), чтобы встать в очередь.
	
	- Третья сессия:
	
	```sql
	SELECT txid_current(), pg_backend_pid();	
	```
	
	|txid_current|pg_backend_pid|
	|------------|--------------|
	|158628|2734|
	
	```sql
	select pg_blocking_pids(2734);
	```
		
	|pg_blocking_pids|
	|----------------|
	|{2733}|
	
	```sql
	SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
	FROM pg_locks WHERE pid = 2734;
	```
	
	|locktype|relation|virtxid|xid|mode|granted|
	|--------|--------|-------|---|----|-------|
	|relation|test_table3_pkey|||RowExclusiveLock|true|
	|relation|test_table3|||RowExclusiveLock|true|
	|virtualxid||7/42||ExclusiveLock|true|
	|transactionid|||158628|ExclusiveLock|true|
	|tuple|test_table3|||ExclusiveLock|false|
	
	Третья транзакция (158628) встаёт в очередь, запрашивая блокировку версии строки (tuple)
	```sql
	SELECT pid, wait_event_type, wait_event, pg_blocking_pids(pid)
	FROM pg_stat_activity
	WHERE backend_type = 'client backend' ORDER BY pid;
	```
	
	|pid|wait_event_type|wait_event|pg_blocking_pids|
	|---|---------------|----------|----------------|
	|2732|Client|ClientRead|{}|
	|2733|Lock|transactionid|{2732}|
	|2734|Lock|tuple|{2733}|
	
# Взаимоблокировка
- Воспроизводим взаимоблокировку трех транзакций. Последовательно выполняем в трёх сессиях:
	1) Первая сессия (3623):
	```sql
	UPDATE test_table3 SET value = value + 1 WHERE id = 1;
	```
	2) Вторая сессия (3255):
	```sql
	UPDATE test_table3 SET value = value + 2 WHERE id = 2;
	```
	3) Третья сессия (3254):
	```sql
	UPDATE test_table3 SET value = value + 3 WHERE id = 3;
	```
	4) Первая сессия (3623):
	```sql
	UPDATE test_table3 SET value = value + 2 WHERE id = 2;
	```
	5) Вторая сессия (3255):
	```sql
	UPDATE test_table3 SET value = value + 3 WHERE id = 3;
	```
	6) Третья сессия (3254):
	```sql
	UPDATE test_table3 SET value = value + 1 WHERE id = 1;
	```
	
	На последнем UPDATE получаем ошибку: 
	```
	SQL Error [40P01]: ERROR: deadlock detected
	  Подробности: Process 3254 waits for ShareLock on transaction 158637; blocked by process 3623.
	Process 3623 waits for ShareLock on transaction 158638; blocked by process 3255.
	Process 3255 waits for ShareLock on transaction 158639; blocked by process 3254.
	  Подсказка: See server log for query details.
	  Где: while updating tuple (0,19) in relation "test_table3"	
	```
	
	В журнале есть вся информация, то есть постфактум разобраться в причинах возникновения взаимоблокировки можно:
	```
	2024-12-08 16:24:52.578 UTC [3623] postgres@postgres LOG:  process 3623 still waiting for ShareLock on transaction 158638 after 200.118 ms
	2024-12-08 16:24:52.578 UTC [3623] postgres@postgres DETAIL:  Process holding the lock: 3255. Wait queue: 3623.
	2024-12-08 16:24:52.578 UTC [3623] postgres@postgres CONTEXT:  while updating tuple (0,36) in relation "test_table3"
	2024-12-08 16:24:52.578 UTC [3623] postgres@postgres STATEMENT:  UPDATE test_table3 SET value = value + 2 WHERE id = 2
	2024-12-08 16:25:13.455 UTC [3255] postgres@postgres LOG:  process 3255 still waiting for ShareLock on transaction 158639 after 200.218 ms
	2024-12-08 16:25:13.455 UTC [3255] postgres@postgres DETAIL:  Process holding the lock: 3254. Wait queue: 3255.
	2024-12-08 16:25:13.455 UTC [3255] postgres@postgres CONTEXT:  while updating tuple (0,18) in relation "test_table3"
	2024-12-08 16:25:13.455 UTC [3255] postgres@postgres STATEMENT:  UPDATE test_table3 SET value = value + 3 WHERE id = 3
		
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres LOG:  process 3254 detected deadlock while waiting for ShareLock on transaction 158637 after 200.180 ms
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres DETAIL:  Process holding the lock: 3623. Wait queue: .
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres CONTEXT:  while updating tuple (0,19) in relation "test_table3"
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres STATEMENT:  UPDATE test_table3 SET value = value + 1 WHERE id = 1
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres ERROR:  deadlock detected
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres DETAIL:  Process 3254 waits for ShareLock on transaction 158637; blocked by process 3623.
		Process 3623 waits for ShareLock on transaction 158638; blocked by process 3255.
		Process 3255 waits for ShareLock on transaction 158639; blocked by process 3254.
		Process 3254: UPDATE test_table3 SET value = value + 1 WHERE id = 1
		Process 3623: UPDATE test_table3 SET value = value + 2 WHERE id = 2
		Process 3255: UPDATE test_table3 SET value = value + 3 WHERE id = 3
		
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres HINT:  See server log for query details.
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres CONTEXT:  while updating tuple (0,19) in relation "test_table3"
	2024-12-08 16:25:34.197 UTC [3254] postgres@postgres STATEMENT:  UPDATE test_table3 SET value = value + 1 WHERE id = 1
	2024-12-08 16:25:34.197 UTC [3255] postgres@postgres LOG:  process 3255 acquired ShareLock on transaction 158639 after 20942.631 ms
	2024-12-08 16:25:34.197 UTC [3255] postgres@postgres CONTEXT:  while updating tuple (0,18) in relation "test_table3"
	2024-12-08 16:25:34.197 UTC [3255] postgres@postgres STATEMENT:  UPDATE test_table3 SET value = value + 3 WHERE id = 3
	```
# Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?	
Теоретически могут, так как обновление происходит не мгновенно и строки в таблице обновляются транзакциями в разном порядке. В явном виде воспроизвести не получилось, поэтому пришлось имитировать через FOR UPDATE в цикле с вызовом pg_sleep:
- В первой сессии запускаем анонимный блок, извлекающий в цикле строку из курсора со всеми записями таблицы, отсортированными по возрастанию:
	```sql
	DO $$
	DECLARE
		cur CURSOR FOR SELECT * FROM test_table3 ORDER BY id FOR UPDATE;
		r record;
	BEGIN
		OPEN cur;
		LOOP 
			FETCH cur INTO r;
			IF NOT FOUND then exit; end IF;
			RAISE NOTICE 'ID: %', r.id;
			PERFORM pg_sleep(5);
		END LOOP;
	END $$;
	```
- В второй сессии запускаем аналогичный блок, но с сортировкой по убыванию:
	```sql
	DO $$
	DECLARE
		cur CURSOR FOR SELECT * FROM test_table3 ORDER BY id DESC FOR UPDATE;
		r record;
	BEGIN
		OPEN cur;
		LOOP 
			FETCH cur INTO r;
			IF NOT FOUND then exit; end IF;
			RAISE NOTICE 'ID: %', r.id;
			PERFORM pg_sleep(5);
		END LOOP;
	END $$;
	```
- При попытке извлечь строку с id=6 во второй сессии получаем взаимоблокировку:
	```sql
	SQL Error [40P01]: ERROR: deadlock detected
	  Подробности: Process 7661 waits for ShareLock on transaction 158667; blocked by process 7662.
	Process 7662 waits for ShareLock on transaction 158668; blocked by process 7661.
	  Подсказка: See server log for query details.
	  Где: while locking tuple (0,5) in relation "test_table3"
	PL/pgSQL function inline_code_block line 8 at FETCH
	```



