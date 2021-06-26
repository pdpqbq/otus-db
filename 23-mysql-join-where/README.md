SQL выборка

Цель:  
Научиться джойнить таблицы и использовать условия в SQL выборке

Написать запрос с inner join  
Написать запрос с left join  
Написать 5 запросов с WHERE с использованием разных операторов, описать для чего нужна такая выборка данных

Используется демо-БД employee https://github.com/datacharmer/test_db с сайта https://dev.mysql.com/doc/index-other.html

Vagrantfile создает ВМ и устанавливает демо-БД

INNER JOIN  

Количество действующих сотрудников в каждом отделе
```
select count(*), departments.dept_name
from dept_emp
inner join departments on dept_emp.dept_no = departments.dept_no
where dept_emp.to_date like '9999%' group by dept_emp.dept_no
;
```
LEFT JOIN

Список менеджеров - имя, фамилия, дата увольнения, если еще работает - то в поле даты NULL
```
with t1 as (
select e.emp_no, e.first_name, e.last_name, t.title, t.to_date from employees e
join titles t
on e.emp_no = t.emp_no
where t.title = 'Manager'
),
t2 as (
select e.emp_no, e.first_name, e.last_name, t.title, t.to_date from employees e
join titles t
on e.emp_no = t.emp_no
where t.title = 'Manager' and t.to_date != '9999-01-01'
)
select t1.first_name, t1.last_name, t2.to_date from t1
left join t2
on t1.emp_no = t2.emp_no;
```
WHERE

Количество сотрудников, принятых в 1999 году
```
select count(emp_no) from employees where hire_date like '1999%';
```
Сотрудники, принятые в 1990-1991 годах
```
select * from employees where hire_date between '1990-01-01' and '1991-12-31';
```
Сотрудники старше 65 лет
```
select * from employees where birth_date < DATE_ADD(SYSDATE(), INTERVAL -65 YEAR);
```
Количество действующих сотрудников-женщин
```
select count(*) from employees e
join dept_emp de
on e.emp_no = de.emp_no
where e.gender = 'F' and de.to_date = '9999-01-01';
```
Действующие сотрудники с зарплатой меньше 39000
```
select e.emp_no, e.first_name, e.last_name, s.salary, t.title
from employees e
join salaries s
on e.emp_no = s.emp_no
join titles t
on e.emp_no = t.emp_no
where s.to_date = '9999-01-01' and s.salary < 39000;
```
