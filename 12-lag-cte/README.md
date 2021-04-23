Использование функций LAG и CTE  
- написать запрос суммы очков с группировкой и сортировкой по годам
- написать cte показывающее тоже самое
- используя функцию LAG вывести кол-во очков по всем игрокам за текущий код и за предыдущий

Создание и наполнение таблицы
```
CREATE TABLE statistic( player_name VARCHAR(100) NOT NULL, player_id INT NOT NULL, year_game SMALLINT NOT NULL CHECK (year_game > 0), points DECIMAL(12,2) CHECK (points >= 0), PRIMARY KEY (player_name,year_game) );

INSERT INTO statistic(player_name, player_id, year_game, points) VALUES ('Mike',1,2018,18), ('Jack',2,2018,14), ('Jackie',3,2018,30), ('Jet',4,2018,30), ('Luke',1,2019,16), ('Mike',2,2019,14), ('Jack',3,2019,15), ('Jackie',4,2019,28), ('Jet',5,2019,25), ('Luke',1,2020,19), ('Mike',2,2020,17), ('Jack',3,2020,18), ('Jackie',4,2020,29), ('Jet',5,2020,27);
```    
Запрос суммы очков с группировкой и сортировкой по годам
```
select
  year_game, sum(points) from statistic
group by
  year_game
order by 1;
```
|year_game|sum|
|---------|---|
|2018|92.00|
|2019|98.00|
|2020|110.00|

```
select
  year_game, player_name, sum(points) from statistic
group by
  (player_name, year_game)
order by 1, 2;
```
|year_game|player_name|sum|
|---------|-----------|---|
|2018|Jack|14.00|
|2018|Jackie|30.00|
|2018|Jet|30.00|
|2018|Mike|18.00|
|2019|Jack|15.00|
|2019|Jackie|28.00|
|2019|Jet|25.00|
|2019|Luke|16.00|
|2019|Mike|14.00|
|2020|Jack|18.00|
|2020|Jackie|29.00|
|2020|Jet|27.00|
|2020|Luke|19.00|
|2020|Mike|17.00|

CTE показывающее тоже самое
```
with year as (
	select distinct year_game from statistic order by 1
	)
select a.year_game cte_year, b.points from year a
join
(select distinct year_game, sum(points) over(partition by year_game) as points from statistic) b
on a.year_game = b.year_game;
```
|cte_year|points|
|--------|------|
|2018|92.00|
|2019|98.00|
|2020|110.00|

```
with t as (
	select year_game, player_name, sum(points) over(partition by year_game, player_name) from statistic order by year_game
	)
select * from t order by 1, 2;
```
|year_game|player_name|sum|
|---------|-----------|---|
|2018|Jack|14.00|
|2018|Jackie|30.00|
|2018|Jet|30.00|
|2018|Mike|18.00|
|2019|Jack|15.00|
|2019|Jackie|28.00|
|2019|Jet|25.00|
|2019|Luke|16.00|
|2019|Mike|14.00|
|2020|Jack|18.00|
|2020|Jackie|29.00|
|2020|Jet|27.00|
|2020|Luke|19.00|
|2020|Mike|17.00|

Кол-во очков по всем игрокам за текущий код и за предыдущий
```
with t as (
	select year_game, sum(points) points from statistic group by year_game order by year_game asc
	)
select
	year_game,
	points,
	lag(points, 1) over(order by year_game) prev_year_points from t;
```
|year_game|points|prev_year_points|
|---------|------|----------------|
|2018|92.00||
|2019|98.00|92.00|
|2020|110.00|98.00|

```
with t as (
	select year_game, player_name, sum(points) points from statistic group by (year_game, player_name) order by year_game, player_name asc
	)
select
	year_game,
	player_name,
	points,
	lag(points, 1) over(partition by player_name order by year_game) prev_year_points from t
order by 1, 2;
```
|year_game|player_name|points|prev_year_points|
|---------|-----------|------|----------------|
|2018|Jack|14.00||
|2018|Jackie|30.00||
|2018|Jet|30.00||
|2018|Mike|18.00||
|2019|Jack|15.00|14.00|
|2019|Jackie|28.00|30.00|
|2019|Jet|25.00|30.00|
|2019|Luke|16.00||
|2019|Mike|14.00|18.00|
|2020|Jack|18.00|15.00|
|2020|Jackie|29.00|28.00|
|2020|Jet|27.00|25.00|
|2020|Luke|19.00|16.00|
|2020|Mike|17.00|14.00|
