# Создание тестовой среды
- Создаём схему в БД PostgreSQL 15 из предыдущего ДЗ:
	```sql
	DROP SCHEMA IF EXISTS pract_functions CASCADE;
	CREATE SCHEMA pract_functions;

	SET search_path = pract_functions;
	 ```
- Создаём таблицы и заполняем данным:
	```sql
	-- Товары
	CREATE TABLE goods
	(
		goods_id    integer PRIMARY KEY,
		good_name   varchar(63) NOT NULL,
		good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
	);
	
	INSERT INTO goods (goods_id, good_name, good_price)
	VALUES  (1, 'Спички хозайственные', .50),
			(2, 'Автомобиль Ferrari FXX K', 185000000.01);

	-- Продажи
	CREATE TABLE sales
	(
		sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
		good_id     integer REFERENCES goods (goods_id),
		sales_time  timestamp with time zone DEFAULT now(),
		sales_qty   integer CHECK (sales_qty > 0)
	);

	INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
	```
- Создаём таблицу для витрины:
	```sql
	CREATE TABLE good_sum_mart
	(
		good_name   varchar(63) NOT NULL,
		sum_sale    numeric(16, 2)NOT NULL
	);
	```
- Заполняем витрину существующими данными:
	```sql
	INSERT INTO good_sum_mart 
		SELECT g.good_name, sum(g.good_price * s.sales_qty) 
		FROM sales s, goods g
		WHERE  s.good_id = g.goods_id 
		GROUP BY g.good_name;
	```
# Создание триггера для заполнения витрины:
- Создаём триггерную функцию:
	```sql
	CREATE OR REPLACE FUNCTION update_good_sum_mart()
	RETURNS TRIGGER AS $$
	BEGIN
		/*
		При обновлении\удалении вычитаем старую сумму
		*/
		IF TG_OP IN ('UPDATE', 'DELETE') THEN
			UPDATE good_sum_mart
			SET sum_sale = sum_sale - (SELECT g.good_price * OLD.sales_qty
										  FROM goods g
										  WHERE g.goods_id = OLD.good_id)
			WHERE good_name = (SELECT g.good_name FROM goods g WHERE g.goods_id = OLD.good_id);    
		END IF;
		/*
		При добавлении\обновлении вставляем новую сумму, если товара нет
		или обновляем сумму в существующем
		*/
		IF TG_OP IN ('UPDATE', 'INSERT') THEN
			MERGE INTO good_sum_mart m
			USING (SELECT good_name, good_price
					FROM goods
					WHERE goods_id = NEW.good_id) g
			ON m.good_name = g.good_name
			WHEN MATCHED THEN
			  UPDATE SET sum_sale = sum_sale + g.good_price * NEW.sales_qty
			WHEN NOT MATCHED THEN
				INSERT VALUES (g.good_name, g.good_price * NEW.sales_qty);
		END IF;
		/* 
		Удаляем строки с нулевыми суммами, которые могут появиться при изменении или удалении заказа
		*/
		DELETE FROM good_sum_mart WHERE sum_sale <= 0;
		RETURN null;
	END;
	$$ LANGUAGE plpgsql;
	``` 
- Создаём триггер уровня строки, вызывающий нашу триггерную функцию после INSERT, UPDATE И DELETE:
	```sql
	CREATE TRIGGER update_good_sum_mart_trigger
	AFTER INSERT OR UPDATE OR DELETE ON sales
	FOR EACH ROW
	EXECUTE FUNCTION update_good_sum_mart();
	```
- Проверяем работу триггера. Текущее состояние витрины:
	```sql
	SELECT  * FROM good_sum_mart;
	```

	|good_name|sum_sale|
	|---------|--------|
	|Автомобиль Ferrari FXX K|185000000.01|
	|Спички хозайственные|65.50|
	
	
- Создаём продажу 10 единиц спичек по цене 0,5 и проверяем витрину:
	```sql
	INSERT INTO sales (good_id, sales_qty) VALUES (1, 10);
	```

	|good_name|sum_sale|
	|---------|--------|
	|Автомобиль Ferrari FXX K|185000000.01|
	|Спички хозайственные|70.50|
	
	Сумма увеличилась на 5, как и должно было быть.
	
- Создаём новый товар и его продажу:
	```sql
	INSERT INTO goods (goods_id, good_name, good_price)
	VALUES  (3, 'Галоши резиновые', 1000);
	```
	```sql
	INSERT INTO sales (good_id, sales_qty) VALUES (3, 5);
	```
	|good_name|sum_sale|
	|---------|--------|
	|Автомобиль Ferrari FXX K|185000000.01|
	|Спички хозайственные|70.50|
	|Галоши резиновые|5000.00|

- Обновляем предыдущую продажу, изменив количество на 10 штук:
	```sql
	UPDATE sales SET sales_qty = 10 WHERE sales_id = 42;
	```
	|good_name|sum_sale|
	|---------|--------|
	|Автомобиль Ferrari FXX K|185000000.01|
	|Спички хозайственные|70.50|
	|Галоши резиновые|10000.00|
	
- Удаляем продажу Автомобиль Ferrari FXX K:
	```sql
	DELETE  FROM sales WHERE sales_id = 40;
	```	
	
	|good_name|sum_sale|
	|---------|--------|
	|Спички хозайственные|70.50|
	|Галоши резиновые|10000.00|

# Задание со *
- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)? Подсказка: В реальной жизни возможны изменения цен:

	Витрина может быть настроена так, чтобы автоматически учитывать изменение цены на товары. Это означает, что любые изменения будут немедленно отражены в витрине, что позволяет избежать ситуации, когда отчет основан на устаревших данных. Триггеры в данном случае обеспечивают синхронизацию между таблицами, поддерживая актуальность данных в витрине без необходимости ручного вмешательства.
	
	Текущая реализация витрины имеет проблемы с целостностью данных, так как при изменении цены товара в таблице goods и последующего изменения заказа с этим товаром в sales (например, количества проданного товара) мы обновляем витрину исходя из новой цены, хотя товар был продан по старой.
	
	Чтобы этого избежать, требуется хранить исторические данные по цене. Например, можно реализовать таблицу, в которую с помощью триггера на goods записывать старую цену и даты начала и окончания её действия. С учётом того, что в sales указывается дата и время продажи, то в триггере, обновляющем витрину, мы можем учесть цену на конкретный момент времени.
	
	Так же можно хранить цену непосредственно в sales, зафиксировав её на момент продажи.