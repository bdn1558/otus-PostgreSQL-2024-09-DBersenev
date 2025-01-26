- Создаём  ВМ
	```sh
	yc compute instance create --name pg-vm1 --hostname pg-vm1 --cores 2 --memory 4 --create-boot-disk size=10G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-24-04-lts --network-interface subnet-name=otus-subnet,nat-ip-version=ipv4 --ssh-key yc_key.pub
	```
	
- Устанавливаем и настраиваем PostgreSQL 17 на ВМ:
	```sh
	sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg && sudo apt update && sudo apt upgrade -y && sudo apt -y install postgresql-17
	```
	
	```sh
	sudo nano /etc/postgresql/17/main/postgresql.conf
	sudo nano /etc/postgresql/17/main/pg_hba.conf
	```
	
	```sh
	sudo -u postgres psql
	\password   ******
	\q
	```
	
	```
	sudo pg_ctlcluster 17 main restart
	```
- Создаём базу и схему. В качестве тестовой используем БД с https://github.com/emil/f1/:
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
	
- Прямое соединение двух и более таблиц
	```sql
	/* Итоговое положение Гран-при Австралии 2014 */
	SELECT r."name",
		   r."year",
		   rs."position_text" position,
		   d.forename || ' ' || d.surname driver_name,
		   c."name" constr_name
	  FROM f1.results rs
	  JOIN f1.races r
		ON r.race_id = rs.race_id
	  JOIN f1.drivers d
		ON d.driver_id = rs.driver_id
	  JOIN f1.constructors c
		ON c.constructor_id = rs.constructor_id
	  JOIN f1.status s
		ON s.status_id = rs.status_id 
	 WHERE r."year" = 2014
	 ORDER BY r.round, rs."position_order";
	```
		
	|name|year|position|driver_name|constr_name|
	|----|----|--------|-----------|-----------|
	|Australian Grand Prix|2014|1|Nico Rosberg|Mercedes|
	|Australian Grand Prix|2014|2|Kevin Magnussen|McLaren|
	|Australian Grand Prix|2014|3|Jenson Button|McLaren|
	|Australian Grand Prix|2014|4|Fernando Alonso|Ferrari|
	|Australian Grand Prix|2014|5|Valtteri Bottas|Williams|
	|Australian Grand Prix|2014|6|Nico Hülkenberg|Force India|
	|Australian Grand Prix|2014|7|Kimi Räikkönen|Ferrari|
	|Australian Grand Prix|2014|8|Jean-Éric Vergne|Toro Rosso|
	|Australian Grand Prix|2014|9|Daniil Kvyat|Toro Rosso|
	|Australian Grand Prix|2014|10|Sergio Pérez|Force India|
	|Australian Grand Prix|2014|11|Adrian Sutil|Sauber|
	|Australian Grand Prix|2014|12|Esteban Gutiérrez|Sauber|
	|Australian Grand Prix|2014|13|Max Chilton|Marussia|
	|Australian Grand Prix|2014|N|Jules Bianchi|Marussia|
	|Australian Grand Prix|2014|R|Romain Grosjean|Lotus F1|
	|Australian Grand Prix|2014|R|Pastor Maldonado|Lotus F1|
	|Australian Grand Prix|2014|R|Marcus Ericsson|Caterham|
	|Australian Grand Prix|2014|R|Sebastian Vettel|Red Bull|
	|Australian Grand Prix|2014|R|Lewis Hamilton|Mercedes|
	|Australian Grand Prix|2014|R|Felipe Massa|Williams|
	|Australian Grand Prix|2014|R|Kamui Kobayashi|Caterham|
	|Australian Grand Prix|2014|D|Daniel Ricciardo|Red Bull|

- Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
	```sql
	/* Сезоны, в которых проводились гонка на трассе Альберт-Парк, начиная с 1980 года */
	SELECT s."year", c."name"
	  FROM f1.seasons s
	  LEFT JOIN f1.races r
		ON r."year" = s."year"
	       AND r.circuit_id = 1
	  LEFT JOIN f1.circuits c
	   USING (circuit_id)
	 WHERE s."year" >= 1980
	 ORDER BY s."year"
	```	
	
	|year|name|
	|----|----|
	|1980||
	|1981||
	|1982||
	|1983||
	|1984||
	|1985||
	|1986||
	|1987||
	|1988||
	|1989||
	|1990||
	|1991||
	|1992||
	|1993||
	|1994||
	|1995||
	|1996|Albert Park Grand Prix Circuit|
	|1997|Albert Park Grand Prix Circuit|
	|1998|Albert Park Grand Prix Circuit|
	|1999|Albert Park Grand Prix Circuit|
	|2000|Albert Park Grand Prix Circuit|
	|2001|Albert Park Grand Prix Circuit|
	|2002|Albert Park Grand Prix Circuit|
	|2003|Albert Park Grand Prix Circuit|
	|2004|Albert Park Grand Prix Circuit|
	|2005|Albert Park Grand Prix Circuit|
	|2006|Albert Park Grand Prix Circuit|
	|2007|Albert Park Grand Prix Circuit|
	|2008|Albert Park Grand Prix Circuit|
	|2009|Albert Park Grand Prix Circuit|
	|2010|Albert Park Grand Prix Circuit|
	|2011|Albert Park Grand Prix Circuit|
	|2012|Albert Park Grand Prix Circuit|
	|2013|Albert Park Grand Prix Circuit|
	|2014|Albert Park Grand Prix Circuit|
	|2015|Albert Park Grand Prix Circuit|
	|2016|Albert Park Grand Prix Circuit|
	|2017|Albert Park Grand Prix Circuit|
	|2018|Albert Park Grand Prix Circuit|
	|2019|Albert Park Grand Prix Circuit|

- Реализовать кросс соединение двух или более таблиц:
	```sql
	/* Все возможные варианты японских гонщиков и команд */
	SELECT d.forename || ' ' || d.surname driver_name, c."name" constr_name
	FROM f1.drivers d
	CROSS JOIN 
	f1.constructors c
	WHERE d.nationality = 'Japanese'
	AND c.nationality = 'Japanese';
	```
	
	|driver_name|constr_name|
	|-----------|-----------|
	|Kazuki Nakajima|Toyota|
	|Kazuki Nakajima|Super Aguri|
	|Kazuki Nakajima|Honda|
	|Kazuki Nakajima|Kojima|
	|Kazuki Nakajima|Maki|
	|Takuma Sato|Toyota|
	|Takuma Sato|Super Aguri|
	|Takuma Sato|Honda|
	|Takuma Sato|Kojima|
	|Takuma Sato|Maki|
	|Sakon Yamamoto|Toyota|
	|Sakon Yamamoto|Super Aguri|
	|Sakon Yamamoto|Honda|
	|Sakon Yamamoto|Kojima|
	|Sakon Yamamoto|Maki|
	|Yuji Ide|Toyota|
	|Yuji Ide|Super Aguri|
	|Yuji Ide|Honda|
	|Yuji Ide|Kojima|
	|Yuji Ide|Maki|
	|...|...|


- Реализовать полное соединение двух или более таблиц:
	```sql
	/* Все трассы и сезоны, в которых проводились гонки*/
	SELECT r."year", c."name"  
	FROM f1.races r
	FULL JOIN f1.circuits c
	ON r.circuit_id = c.circuit_id
	ORDER BY r."year" DESC;
	```

	|year|name|
	|----|----|
	||Port Imperial Street Circuit|
	|2019|Autodromo Nazionale di Monza|
	|2019|Circuit de Spa-Francorchamps|
	|2019|Hungaroring|
	|2019|Hockenheimring|
	|2019|Silverstone Circuit|
	|2019|Yas Marina Circuit|
	|2019|Red Bull Ring|
	|2019|Circuit Paul Ricard|
	|2019|Circuit Gilles Villeneuve|
	|2019|Circuit de Monaco|
	|2019|Circuit de Barcelona-Catalunya|
	|2019|Baku City Circuit|
	|2019|Shanghai International Circuit|
	|2019|Bahrain International Circuit|
	|2019|Suzuka Circuit|
	|2019|Sochi Autodrom|
	|2019|Autódromo Hermanos Rodríguez|
	|2019|Circuit of the Americas|
	|2019|Autódromo José Carlos Pace|
	|2019|Marina Bay Street Circuit|
	|2019|Albert Park Grand Prix Circuit|
	|2018|Circuit de Barcelona-Catalunya|
	|...|...|

- Реализовать запрос, в котором будут использованы разные типы соединений:
	```sql
	/* Список трасс, на которых побеждал только один гонщик или не побеждал никто */
	WITH winners AS
	 (SELECT d.forename || ' ' || d.surname driver_name,
			 r2.circuit_id,
			 r1."position"
		FROM f1.results r1
		JOIN f1.races r2
	   USING (race_id)
		JOIN f1.drivers d
	   USING (driver_id)
	   WHERE r1."position" = 1)
	SELECT c."name", count(DISTINCT w.driver_name)
	  FROM f1.circuits c
	  LEFT JOIN winners w
	 USING (circuit_id)
	 GROUP BY c."name"
	 HAVING count(DISTINCT w.driver_name) <= 1
	 ORDER BY c."name"
	```
	
	|name|count|
	|----|-----|
	|Ain Diab|1|
	|AVUS|1|
	|Buddh International Circuit|1|
	|Donington Park|1|
	|Fair Park|1|
	|Le Mans|1|
	|Monsanto Park Circuit|1|
	|Nivelles-Baulers|1|
	|Okayama International Circuit|1|
	|Pescara Circuit|1|
	|Port Imperial Street Circuit|0|
	|Riverside International Raceway|1|
	|Sebring International Raceway|1|
	|Zeltweg|1|

- Задание со * не выполнялось