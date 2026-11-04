-- Query steps:
-- Step 1: info
-- Step 2: tables, cols
-- Step 3: common field

-- -- -- -- SELECT -- -- -- --

-- SELECT DISTINCT

-- SELECT DISTINCT customer_id AS "CID"
-- FROM payment
-- WHERE amount >= 7.99
-- ORDER BY customer_id DESC;

-- SELECT 
-- 	title AS "Title",
-- 	rental_rate AS "Rental Cost",
-- 	replacement_cost AS "Replacement Cost"
-- FROM film
-- WHERE
-- 	rental_rate > 2.99
-- OR
-- 	replacement_cost > 19.99;

-- SELECT 
-- 	count(title)
-- FROM film
-- WHERE
-- 	rental_rate > 2.99
-- OR
-- 	replacement_cost > 19.99;

-- -- -- -- ORDER BY -- -- -- --

-- ORDER BY (ASC is default)
-- SELECT rental_rate FROM film ORDER BY rental_rate;
-- SELECT rental_rate FROM film ORDER BY rental_rate ASC;
-- SELECT rental_rate FROM film ORDER BY rental_rate DESC;
-- SELECT rental_rate, title, replacement_cost FROM film ORDER BY rental_rate ASC, replacement_cost DESC;

-- -- -- -- LIMIT -- -- -- --

-- LIMIT
-- SELECT DISTINCT replacement_cost FROM film LIMIT 10;
-- SELECT title, replacement_cost FROM film ORDER BY replacement_cost DESC LIMIT 5;
-- SELECT amount, payment_date, customer_id FROM payment ORDER BY payment_date DESC LIMIT 10;
-- SELECT DISTINCT payment_date AS "Date" FROM payment LIMIT 10;

-- -- -- -- BETWEEN -- -- -- --

-- BETWEEN
-- SELECT * FROM payment WHERE amount BETWEEN 1 AND 2;
-- SELECT * FROM payment WHERE amount NOT BETWEEN 1 AND 2;
-- SELECT DISTINCT amount FROM payment WHERE amount BETWEEN 1 AND 2;

-- -- -- -- IN -- -- -- --

-- IN
-- SELECT * FROM payment WHERE amount IN (1, 2); -- no results
-- SELECT * FROM payment WHERE amount IN (1.99, 2.99);
-- SELECT * FROM address WHERE district IN ('California', 'Alberta') ORDER BY district DESC;
-- SELECT * FROM address WHERE district NOT IN ('California', 'Alberta') ORDER BY district DESC;

-- -- -- -- LIKE -- -- -- --

-- LIKE
-- % any sequence of 0 or more chars
-- _ to match any single character
-- ILIKE - insensitive case
-- SELECT district FROM address WHERE district LIKE '%c%';
-- SELECT district FROM address WHERE district LIKE '%C%';
-- SELECT district FROM address WHERE district LIKE '_a_%'
-- SELECT district FROM address WHERE district LIKE '_alifornia'
-- SELECT address FROM address WHERE address LIKE '%Avenue%'
-- SELECT district FROM address WHERE district LIKE '%and%'
-- SELECT district FROM address WHERE district LIKE '%and_'
-- SELECT district FROM address WHERE district LIKE 'And%'
-- SELECT district FROM address WHERE district ILIKE '%AnD%'

-- -- -- -- WHERE -- -- -- --

-- show -> film title, replacement_cost, rental_duration
-- condition -> replacement_cost, rental_duration, highest to lowest AND rented out 4-6 weeks, top 100
-- SELECT title, replacement_cost, rental_duration 
-- FROM film 
-- WHERE rental_duration BETWEEN 4 AND 6 
-- ORDER BY replacement_cost DESC LIMIT 100;

-- rating G or PG, more than 120 min and `Action` in desc
-- SELECT title, rating, description
-- FROM film
-- WHERE rating IN ('G', 'PG') 
-- 	AND length > 120 
-- 	AND description LIKE '%Action%';

-- -- -- -- AGGREGATE FUNCTIONS -- -- -- --

-- Aggregate functions - can enhance and use with GROUP BY to select smaller groups
-- AVG
-- SELECT AVG(replacement_cost) FROM film;
-- SELECT AVG(rental_rate) FROM film;

-- MIN
-- SELECT MIN(replacement_cost) FROM film;
-- SELECT MIN(rental_rate) FROM film;
-- MAX
-- SELECT MAX(replacement_cost) FROM film;
-- SELECT MAX(rental_rate) FROM film;

-- SUM
-- SELECT SUM(replacement_cost) FROM film;
-- SELECT SUM(rental_rate) FROM film;

-- COUNT
-- SELECT COUNT(*) FROM film;
-- SELECT COUNT(*) FROM film WHERE release_year != 20062;
-- SELECT COUNT(DISTINCT special_features) FROM film;
-- SELECT COUNT(replacement_cost) FROM film;
-- SELECT COUNT(*) FROM film WHERE rating = 'PG-13';
-- combinations
-- SELECT ROUND(AVG(rental_rate), 2) AS "Average rental cost", MAX(replacement_cost) AS "Maximum replacement cost" FROM film;

-- -- -- -- GROUP BY -- -- -- --

-- GROUP BY
-- see rental rate and replacement cost per year
-- SELECT release_year, SUM(rental_rate), SUM(replacement_cost) FROM film GROUP BY release_year;
-- see totals by rating of replacement cost, and total of each rating group
-- SELECT rating, SUM(replacement_cost), COUNT(rating) FROM film GROUP BY rating;
-- see revenue each staff member brought in
-- SELECT staff_id, SUM(amount) AS "Money earned" FROM payment GROUP BY staff_id;
-- same but top sales first
-- SELECT staff_id, SUM(amount) FROM payment GROUP BY staff_id ORDER BY SUM(amount) DESC;
-- for certain dates
-- SELECT staff_id, SUM(amount) FROM payment WHERE payment_date BETWEEN '2007-02-01' and '2007-02-28' GROUP BY staff_id ORDER BY SUM(amount) DESC;

-- -- -- -- GROUP BY + HAVING -- -- -- --

-- HAVING - on aggregated data, used with GROUP BY, allows selecting based on agg functions
-- SELECT staff_id, SUM(amount) FROM payment
-- GROUP BY staff_id
-- HAVING SUM(amount) > 31000;

-- SELECT staff_id, AVG(amount) FROM payment
-- GROUP BY staff_id
-- HAVING AVG(amount) > 4.2;
-- SELECT staff_id AS "Staff ID", SUM(amount) AS "Total Sales" FROM payment
-- WHERE payment_date BETWEEN '2007-02-15' AND '2007-02-20'
-- GROUP BY staff_id
-- ORDER BY SUM(amount) DESC;

-- -- -- -- SUB-QUERIES -- -- -- --

-- Sub-query, nested query - SQL first runs subquery so think about that first
-- average payment
-- SELECT AVG(amount) FROM payment;
-- staff_id who has higher average than the average
-- SELECT staff_id, AVG(amount) AS "Average Amount" FROM payment 
-- GROUP BY staff_id
-- HAVING AVG(amount) > (SELECT AVG(amount) FROM payment);


-- SELECT title, rental_rate FROM film
-- LIMIT 10;
-- only get movies whos rental rate is greater than the total average
-- start with sub query - average
-- SELECT AVG(rental_rate) FROM film;
-- SELECT title, rental_rate FROM film
-- WHERE rental_rate > (SELECT AVG(rental_rate) FROM film);

-- Which are my top 5 highest renting customers? How much have they spent?
-- payemnts table
-- how much spent is their SUM
-- find limit of top highest DESC renting customers, so spent most
-- SELECT customer_id FROM payment GROUP BY customer_id; -- 599 total customers
-- SELECT customer_id FROM payment GROUP BY customer_id LIMIT 5;
-- solution
-- SELECT DISTINCT customer_id AS "Customer ID", SUM(amount) AS "Spend" FROM payment GROUP BY customer_id ORDER BY SUM(amount) DESC LIMIT 5;

-- The actor table has names of all actors- which actor first names are most common and how many records have it
-- use the COUNT function
-- SELECT DISTINCT first_name FROM actor ORDER BY first_name ASC;
-- Solution
--  FAILS SELECT DISTINCT first_name, COUNT(first_name) FROM actor LIMIT 25;
--  FAILS SELECT first_name, COUNT(first_name) FROM actor LIMIT 25;
-- solution
-- SELECT DISTINCT first_name, COUNT(*) FROM actor
-- GROUP BY first_name
-- ORDER BY count DESC, first_name ASC;

-- -- -- -- SQL ORDER -- -- -- --

-- SQL ORDER
-- -- FROM
-- -- WHERE
-- -- GROUP BY
-- -- HAVING
-- -- ORDER BY

-- -- -- -- JOIN -- -- -- -- 

-- JOINS
-- INNER JOIN
-- LEFT JOIN
-- RIGHT JOIN
-- FULL JOIN
-- UNION

-- JOIN checklist
-- 1. What is the end info you want?
-- 2. What tables do we need to get that, and their structure
-- 3. What is the common fiend we will join?

-- INNER JOIN ** NOTE - Joins looks for presence of the rows, all have the film_id as all films have categories
-- SELECT * FROM film INNER JOIN film_category ON film.film_id = film_category.film_id;
-- SELECT first_name, last_name FROM customer INNER JOIN payment ON customer.customer_id = payment.customer_id;
-- Requiremnt: send out coupons to end users
-- 1. send out coupons - first_name, last_name, complete address - address, address2, district, postal_code, phone
-- 2. 
	-- customer: customer_id, address_id, first_name, last_name
	-- address: address_id, address, address2, district, postal_code, phone
-- 3. address_id

-- SELECT CONCAT(customer.first_name, ' ', customer.last_name) AS "Name", address.address, address.address2, address.district, address.postal_code, address.phone
-- FROM customer
-- INNER JOIN address
-- ON customer.address_id = address.address_id;

-- LEFT OUTER JOIN
-- ALL records from 1st, only matching from 2nd
-- what ISNT available from 2nd, will be NULL
-- if exists in 2nd, it will not be shown in results

-- film, inventory
-- Requirement: film title, inventory id, inventory id that is nULL
-- Step 1: info
	-- film.title
	-- inventory inventory.id
	-- inventory.id which is null
-- Step 2: tables, cols
	-- film: 
	-- inventory: 
-- Step 3: common field

-- Question: Need to find movies that don't have inventory at any store
-- SELECT * FROM inventory;
-- SELECT film.film_id, film.title, inventory.store_id, inventory.inventory_id FROM film 
-- LEFT JOIN inventory
-- ON film.film_id = inventory.film_id
-- WHERE inventory.inventory_id IS NULL;
-- ORDER BY inventory.inventory_id DESC; // this one gets ALL the movies regardles of inventory_id

-- RIGHT OUTER JOIN
-- All records from right table
-- if a match on left table, include it, else add NULL
-- SELECT & FROM table1 RIGHT JOIN table2 ON table1.col_name = table2.col_name;

-- Staff and Address
-- SELECT * FROM staff;
-- SELECT * FROM address;
-- shared - address_id;

-- SELECT 
-- 	CONCAT(staff.first_name, ' ', staff.last_name) AS "Staff Fullname", 
-- 	address.address, address.postal_code, address.district
-- FROM staff
-- RIGHT JOIN address
-- ON staff.address_id = address.address_id;

-- FULL OUTER JOIN
-- Returns all rows in both tables, if match include it else NULL
-- SELECT * FROM table1 FULL JOIN table2 ON table1.col_name = table2.col_name;

-- SELECT * FROM customer FULL JOIN address ON customer.address_id = address.address_id
-- WHERE customer.address_id IS NULL;

-- UNION - stacked results
-- combines results from two or more SELECT statements into a single results
-- Syntax: SELECT cons FROM table1 UNION SELECT cols FROM table2;
-- Every select must have the same number of cols
-- cols must have same data type
-- cols must be in same order
-- results will be distinct, note duplicates
-- BASICALLY it gets a total of all results in 2 tables, if may be missing in either or

-- -- -- -- UNION -- -- -- --

-- SELECT name FROM category
-- UNION
-- SELECT name FROM language;

-- -- -- -- EX1 -- -- -- --
-- Pattern between film language and rental rate. Need language of each movie, title, and rental rate
-- hint - join film table title rental rate adn language table
-- Step 1: info
-- -- 
-- Step 2: tables, cols
-- -- 
-- -- 
-- Step 3: common field
-- -- language_id
-- Quirks
-- -- 5 films have no language_id

-- ** ALIASING TABLES ** --
-- SELECT f.title, f.rental_rate, l.name
-- FROM film f
-- INNER JOIN language l
-- ON f.language_id = l.language_id 
-- WHERE f.language_id IS NOT NULL
-- ORDER BY f.title ASC;


-- -- -- -- EX2 -- -- -- --
-- *** JOIN 3 TABLES!!
-- List of actors, count of movies they've been in
-- sort so actor who has done the most is at the top
-- Step 1: info
-- -- actor first_name, actor last_name
-- -- count of movies - COUNT(film_id)
-- Step 2: tables, cols
-- -- film: film_id
-- -- actor: first_name, last_name, actor_id
-- -- film_actor: film_id, actor_id
-- Step 3: common field
-- -- actor_id on film_actor

-- Attempt - dont know the count how to get that 
-- SELECT 
-- 	DISTINCT CONCAT(a.first_name, ' ', a.last_name) AS "Actor Fullname"
-- FROM film f
-- JOIN film_actor fa
-- 	ON f.film_id = fa.film_id
-- JOIN actor a 
-- 	ON fa.actor_id = a.actor_id;

-- Solution
-- SELECT
-- 	a.actor_id,
-- 	CONCAT(a.first_name, ' ', a.last_name) AS "Actor Name",
-- 	COUNT(f.film_id) AS "Film Count"
-- FROM actor a
-- JOIN film_actor fa
-- 	ON a.actor_id = fa.actor_id
-- JOIN film f
-- 	ON f.film_id = fa.film_id
-- GROUP BY a.actor_id
-- ORDER BY "Film Count" DESC;


-- -- -- -- EX3 -- -- -- --
-- different ratings of ALL movies, and count of movies that have been rented for each rating
-- film_id and ratings in film
-- rental table has films that is rented, but no film_id - need 3rd table
-- Step 1: info
-- -- all movies rented per rating
-- -- group by rating
-- Step 2: tables, cols
-- -- film: film_id, title, rating
-- -- inventory: film_id, inventory_id
-- -- rental: inventory_id
-- Step 3: common field
-- -- 

-- -- Attempt 1: 958 results
-- SELECT 
-- 	f.title,
-- 	f.rating,
-- 	COUNT(f.film_id) AS "Rental Count"
-- FROM film f
-- JOIN inventory i
-- 	ON f.film_id = i.film_id
-- JOIN rental r
-- 	ON i.inventory_id = r.inventory_id
-- GROUP BY f.title, f.rating
-- ORDER BY f.rating, "Rental Count" DESC;

-- ATTEMPT 2
-- SELECT
-- 	f.title AS "Film Title",
-- 	f.rating AS "Film Rating",
-- 	COUNT(r.rental_id) AS "Rentals"
-- FROM rental r
-- JOIN inventory i
-- 	ON r.inventory_id = i.inventory_id
-- RIGHT JOIN film f
-- 	ON f.film_id = i.film_id
-- GROUP BY "Film Title", "Film Rating"
-- ORDER BY "Rentals" DESC

-- Solution
-- Rental -> Inventory -> film
-- scenario
-- movie	-	rating
-- A		-	PG13
-- B		-	R
-- C		-	G

-- movie	-	rental count
-- A		-	1
-- B		-	5
-- C		-	0
-- need to include movie C, so want all data from film
-- other tables only matches

-- SELECT 
-- 	f.rating,
-- 	COUNT(i.inventory_id) AS "Count"
-- FROM rental r
-- JOIN inventory i
-- 	ON r.inventory_id = i.inventory_id
-- RIGHT JOIN film f
-- 	ON i.film_id = f.film_id
-- GROUP BY f.rating
-- ORDER BY "Count" DESC;

-- ADVANCED - same thing with left join
-- SELECT 
-- 	f.rating,
-- 	COUNT(i.inventory_id) AS "Count"
-- FROM film f
-- LEFT JOIN inventory i 
-- 	ON f.film_id = i.film_id
-- LEFT JOIN rental r 
-- 	ON i.inventory_id = r.inventory_id
-- GROUP BY f.rating
-- ORDER BY "Count" DESC;








-- -- -- -- DATA TYPES & OPERATIONS -- -- -- --
-- COMMON DATA TYPES: postgresql.org/docs/current/datatype.html
-- -- Numeric
-- -- -- smallint
-- -- -- integer
-- -- -- bigint
-- -- -- float
-- -- -- numeric
-- -- -- real
-- -- -- double precision
-- -- -- smallserial
-- -- -- serial
-- -- -- bigserial
-- -- Character
-- -- -- varchar - variable length
-- -- -- char - fixed length
-- -- -- text - vairable unlimited
-- -- Date/Time
-- -- -- timestamp without time zone TIMESTAMPTZ
-- -- -- timestamp with time zone
-- -- -- date
-- -- -- time without time zone
-- -- -- time with time zone
-- -- -- interval
-- -- Boolean - true, false
-- -- XML
-- -- JSON
-- -- Array

-- Undertand when picking data types
-- Dates -timestamp
-- TIPS
-- -- Amalyze the data, needed analytics
-- -- Accommodate teh future growth
-- -- Use variable types like varchar instead of char
-- -- Always use date and time, not date
-- -- Watch out for special cases- numbers with hyphens, plus, leading zeroes
-- -- use TIMESTAMPTZ and set db to UTC

-- SET TIME ZONE 'UTC';
-- ALTER DATABASE your_database_name SET timezone TO 'UTC';
-- ALTER ROLE your_username SET timezone TO 'UTC';

-- DATE/TIME FUNCTIONS
-- -- CURRENT_DATE - today's date 2026-11-02
-- -- CURRENT_TIME - current time 23:80:46.239831-05
-- -- CURRENT_TIMESTAMP - 2022-11-29 23:08:46.239831-05
-- -- now() - 2022-11-04 06:18:16.271765+07
-- -- age(timestamp, timestamp) - subtracts 2nd timestamp and returns interval
-- -- to_date(text, format) - string to date value TO_DATE('20231029','YYYYMMDD') > '2023-10-29'

-- SELECT CURRENT_DATE;
-- SELECT CURRENT_TIME;
-- SELECT CURRENT_TIMESTAMP;
-- SELECT NOW();
-- SELECT AGE('2024-11-29', '2023-05-01') -- 1 year 6 mons 28 days
-- SELECT AGE(TIMESTAMP'2023-05-01'); -- 3 years 6 mons 1 day
-- SELECT TO_DATE('20230501','YYYYMMDD') -- 2023-05-01

-- EXTRACT
-- parse unit from sql date time string
-- SELECT EXTRACT(EPOCH FROM TIMESTAMP '2023-09-14 17:10:29') -- 1694711429.000000
-- SELECT EXTRACT(MILLENNIUM FROM TIMESTAMP '2023-09-14 17:10:29') -- 3
-- SELECT EXTRACT(JULIAN FROM TIMESTAMP '2023-09-14 17:10:29') -- 2460202.71561342592592592593
-- SELECT EXTRACT(CENTURY FROM TIMESTAMP '2023-09-14 17:10:29') -- 21
-- SELECT EXTRACT(DECADE FROM TIMESTAMP '2023-09-14 17:10:29') -- 202
-- SELECT EXTRACT(YEAR FROM TIMESTAMP '2023-09-14 17:10:29') -- 2023
-- SELECT EXTRACT(QUARTER FROM TIMESTAMP '2023-09-14 17:10:29') -- 3
-- SELECT EXTRACT(MONTH FROM TIMESTAMP '2023-09-14 17:10:29') -- 9
-- SELECT EXTRACT(WEEK FROM TIMESTAMP '2023-09-14 17:10:29') -- 37
-- SELECT EXTRACT(DAY FROM TIMESTAMP '2023-09-14 17:10:29') -- 14
-- SELECT EXTRACT(DOW FROM TIMESTAMP '2023-09-14 17:10:29') -- 4
-- SELECT EXTRACT(ISODOW FROM TIMESTAMP '2023-09-14 17:10:29') -- 4
-- SELECT EXTRACT(HOUR FROM TIMESTAMP '2023-09-14 17:10:29') --  17
-- SELECT EXTRACT(SECOND FROM TIMESTAMP '2023-09-14 17:10:29') -- 29.000000
-- SELECT EXTRACT(MILLISECOND FROM TIMESTAMP '2023-09-14 17:10:29') -- 29000.00
-- SELECT EXTRACT(MICROSECOND FROM TIMESTAMP '2023-09-14 17:10:29') -- 2900000
-- SELECT EXTRACT(NANOSECOND FROM TIMESTAMP '2023-09-14 17:10:29') -- FAILS

-- SELECT DISTINCT EXTRACT(YEAR FROM rental.rental_date) FROM rental;
-- SELECT 
-- 	DISTINCT EXTRACT(MONTH FROM rental.rental_date) AS "Data"
-- FROM rental
-- ORDER BY "Data" DESC;


-- MATH FUNCTIONS
-- -- + - * /
-- -- abs(number)
-- -- mod(x,y)
-- -- power(x,y)
-- -- round(x,y) - round(8.43932,2) > 8.44
-- -- truncate(x,y) - trunc(8.43932,2) > 8.43
-- -- bitwise AND &
-- -- bitwise OR |
-- -- bitwise XOR #
-- -- bitwise NOT ~
-- -- bitwise shift left <<
-- -- bitwise shift right >>

-- SELECT amount + 2 AS "New AMount" FROM payment
-- LIMIT 25;
-- SELECT ABS(-7.99) -- 7.99


-- STRING FUNCTIONS 
-- -- concat - 
-- -- length(text) - charlen
-- -- left(text, n) - first n chars from string - left('peter', 2) > pe
-- -- right(text, n) - last n chars from string - left('peter', 2) > er
-- -- position(substring IN string) - position('sql' in 'postgresql') > 8
-- -- substr(text, start, [count]) - substr('postgresql', 3)> `stgresql`;  substr('postgresql', 3, 2)> `st`
-- -- *** 1-indexed, not 0-indexed

-- SELECT CONCAT(first_name, ' ', last_name) FROM staff;
-- SELECT substr(last_name, 0, 3) FROM staff WHERE staff_id = 2; > 'St'



-- -- -- -- EX1 -- -- -- --
-- rental table rental_date, return_date
-- need to find customers wwith duration 7 days or more, send them email
-- rental date, return date, rental duration diff using AGE() fn
-- Customer first_name, last_name, email
-- sort by rent duration high to low to see longest renters first
-- Step 1: info
-- -- 
-- Step 2: tables, cols
-- -- rental: rental_date, returnDate, rental_id, customer_id
-- -- customer: first_name, last_name, email
-- Step 3: common field
-- -- customer_id

-- SELECT 
-- 	CONCAT(c.first_name, ' ', c.last_name) AS "Customer Name", 
-- 	c.email, 
-- 	r.rental_date,
-- 	r.return_date,
-- 	AGE(r.return_date, r.rental_date) AS "Rent Duration"
-- FROM rental r
-- JOIN customer c
-- 	ON r.customer_id = c.customer_id
-- WHERE r.return_date IS NOT NULL
-- 	AND AGE(r.return_date, r.rental_date) >= interval '7 days'
-- ORDER BY "Rent Duration" ASC;

-- -- -- -- EX2 -- -- -- --
-- film table - title, some titles are long, need to use substr
-- return string after 9th character
-- SELECT title, SUBSTR(title, 10) FROM film;
-- SELECT title, SUBSTR(title, 10,2) FROM film;
-- return string after 15th char, need to check length of LENGTH() 
-- SELECT title, LENGTH(title), SUBSTR(title, 16) FROM film WHERE LENGTH(title) >= 15;

-- SELECT customer_id, amount*30 AS "Monthly Payment"
-- FROM payment

-- -- -- -- CONDITIONAL EXPRESSIONS & OPERATORS -- -- -- --

-- CAST
-- -- converting data types
-- -- CAST(expression AS target_type)
-- -- SELECT CAST('50' AS INTEGER), SELECT CAST('2025-12-03' AS DATE), SELECT CAST('50A' AS INTEGER) > ERROR

-- SELECT city_id, CAST(city_id AS varchar) from address;

-- CASE
-- -- IF ELSE > WHEN THEN ELSE
-- -- CASE
-- -- -- WHEN condition_1 THEN result_1
-- -- -- WHEN condition_2 THEN result_2
-- -- -- [WHEN...]
-- -- -- [ELSE...]
-- -- END
-- -- Example

-- SELECT emp_id, emp_name, emp_salary
-- CASE
-- 	WHEN emp_salary <= 30000 THEN 'Low'
-- 	WHEN emp_salary > 30000 AND emp_salary <= 50000 THEN 'Average'
-- 	WHEN emp_salary > 50000 THEN 'High'
-- END AS "Salary Rating"
-- FROM emp_details;

-- COALESCE
-- -- returns the first arg that is not null
-- -- if all are null, returns NULL
-- -- Use case - substitute a default value for null values
-- -- COALESE(arg_1, arg_2, ...)

-- SELECT date, tickets, COALESCE(tickets, 0) FROM ticket_sales;

-- SELECT 
-- 	address_id, address, address2, COALESCE(address2, 'N/A')
-- FROM address
-- 	WHERE address2 IS NULL
-- LIMIT 10;

-- NULLIF - basically an equality check that returns null
-- -- returns a null if the given arguments match, else it returns the arg1
-- -- NULLIF(arg1, arg2)
-- -- NULLIF(20, 20) > NULL -- used for equality, so if equal then null
-- -- NULLIF(20, 0) > 20
-- -- NULLIF('John', 'Peter') > 'John'
-- -- USE-CASE - math and cannot divide by zero, dividing by null is ok


-- // shows null
-- SELECT amount AS "USD", 10/NULLIF(amount,0) AS "INR" from payment ORDER BY "INR" DESC;
-- // force nulls back to 0 with COALESCE
-- SELECT amount AS "USD", COALESCE(10/NULLIF(amount,0), 0) AS "INR" from payment ORDER BY "INR" DESC;

-- VIEWS
-- -- a virtual table based on a result set of a sql statement
-- -- can be created from or more tables using join
-- -- helpful to view commplex query results quickly, create reports, no need to run same query again and again
-- CREATE VIEW view_name AS query;

-- -- CREATE -- -- 
-- CREATE OR REPLACE VIEW test AS
-- SELECT f.film_id, f.title, f.language_id
-- FROM film f
-- JOIN language l ON f.language_id = l.language_id
-- LIMIT 10;

-- -- READ -- -- 
-- -- basic read
-- SELECT * FROM test;
-- -- some other pg command
-- SELECT definition 
-- FROM pg_views 
-- WHERE viewname = 'test';

-- -- UPDATE -- -- 
-- CREATE OR REPLACE VIEW test AS
-- SELECT f.film_id, f.title, f.language_id, f.rental_rate
-- FROM film f
-- JOIN language l ON f.language_id = l.language_id
-- LIMIT 20;
-- SELECT * FROM test;

-- -- DELETE -- -- 
-- DROP VIEW test;


-- EXISTS
-- "Show me the title of every film that has at least one physical copy in the inventory."
-- Correlated Subquery: The subquery checks the inventory table (i.film_id = f.film_id) for every single row in the main film table (f).
-- Short-circuiting: EXISTS doesn't return data; it just returns a boolean (TRUE or FALSE). 
-- As soon as it finds at least one matching inventory row for a film, 
-- it stops searching that branch and includes the film in the results.
-- SELECT 
--     f.title
-- FROM film f
-- WHERE EXISTS (
--     SELECT 1 
--     FROM inventory i 
--     WHERE i.film_id = f.film_id
-- );

-- -- -- -- EX1 -- -- -- --

-- Step 1: info
-- -- customer anme, email, total rental purchaes, customer category case type
-- Stepaymentp 2: tables, cols
-- -- payment: customer_id, amount
-- -- customer: customer_id, first_name, last_name, email
-- Step 3: common field
-- -- payment: customer_id
-- -- 
-- attempt 1 --
-- SELECT CONCAT(c.first_name, ' ', c.last_name) AS "Customer Name", SUM(p.amount)
-- FROM customer c
-- JOIN payment p 
-- 	ON c.customer_id = p.customer_id
-- GROUP BY "Customer Name"
-- ORDER BY "Customer Name";

-- solution
-- SELECT 
-- 	CONCAT(c.first_name, ' ', c.last_name) AS "Customer Name",
-- 	c.email AS "Email",
-- 	SUM(p.amount) AS "Total Rentals",
-- 	CASE
-- 		WHEN SUM(amount) >= 200 THEN 'Elite'
-- 		WHEN SUM(amount) >= 150 THEN 'Platinum'
-- 		WHEN SUM(amount) >= 100 THEN 'Gold'
-- 		WHEN SUM(amount) >= 0 THEN 'Silver'
-- 	END AS "Member Type"
-- FROM payment p
-- JOIN customer c 
-- 	ON p.customer_id = c.customer_id
-- GROUP BY "Customer Name", "Email" -- NOTE we must group by all of these, SQL will let us know
-- ORDER BY 
-- 	CASE
-- 		WHEN SUM(p.amount) >= 200 THEN 1
-- 		WHEN SUM(p.amount) >= 150 THEN 2
-- 		WHEN SUM(p.amount) >= 100 THEN 3
-- 		WHEN SUM(p.amount) >= 0 THEN 4
-- 	END,
-- 	"Customer Name";






-- -- -- -- EX2 -- -- -- --
-- -- a view for the previous exercise
-- CREATE OR REPLACE VIEW customer_segments AS
-- SELECT 
-- 	CONCAT(c.first_name, ' ', c.last_name) AS "Customer Name",
-- 	c.email AS "Email",
-- 	SUM(p.amount) AS "Total Rentals",
-- 	CASE
-- 		WHEN SUM(amount) >= 200 THEN 'Elite'
-- 		WHEN SUM(amount) >= 150 THEN 'Platinum'
-- 		WHEN SUM(amount) >= 100 THEN 'Gold'
-- 		WHEN SUM(amount) >= 0 THEN 'Silver'
-- 	END AS "Member Type"
-- FROM payment p
-- JOIN customer c 
-- 	ON p.customer_id = c.customer_id
-- GROUP BY "Customer Name", "Email" -- NOTE we must group by all of these, SQL will let us know
-- ORDER BY 
-- 	CASE
-- 		WHEN SUM(p.amount) >= 200 THEN 1
-- 		WHEN SUM(p.amount) >= 150 THEN 2
-- 		WHEN SUM(p.amount) >= 100 THEN 3
-- 		WHEN SUM(p.amount) >= 0 THEN 4
-- 	END,
-- 	"Customer Name";
-- SELECT * FROM customer_segments;



-- -- -- -- CREATE DATABASE AND TABLES -- -- -- --

-- Key concepts
-- PRIMARY KEY
-- -- column or columns, always add a PK
-- -- a table can only have one PK
-- -- PK should be unique and not NULL

-- FOREIGN KEY
-- -- related another table, refers to PK in another table
-- -- child that contains the FK is the child table, table with PK reference to it is the parent table
-- -- can have multiple FKs in a table
-- -- use when joining tables

-- CONTRAINTS
-- -- SQL has 2 levels of constraints - conditions, restrictions
-- -- -- column level
-- -- == table level
-- -- Types of contraints
-- -- -- NOT NULL
-- -- -- UNIQUE - no dupes
-- -- -- PRIMARY KEY - not null, unique
-- -- -- FOREIGN KEY - refrence to PK of another talbe
-- -- -- CHECK - enforce a certain condition
-- -- -- DEFAULT - default value if nothing given


-- DATABASE / CREATE, DROP
-- -- 

-- CREATE TABLE
-- -- 

-- INSERT
-- -- 

-- UPDATE
-- -- 

-- ALTER
-- -- 

-- DELETE
-- -- 

-- DROP, TRUNCATE
-- -- 


-- -- -- -- EX1 -- -- -- --
-- Step 1: info
-- -- 
-- Step 2: tables, cols
-- -- 
-- Step 3: common field
-- -- 
-- -- -- -- EX2 -- -- -- --
-- Step 1: info
-- -- 
-- Step 2: tables, cols
-- -- 
-- Step 3: common field
-- -- 

-- -- -- -- FINAL TIPS -- -- -- --















-- CREATE TABLE employee (
-- 	id SERIAL PRIMARY KEY,
-- 	emp_name VARCHAR(50) NOT NULL,
-- 	emp_location VARCHAR(100) NOT NULL,
-- 	emp_salary INT
-- )

-- SELECT * FROM employee;
-- INSERT INTO employee (emp_name, emp_location, emp_salary) VALUES ('John', 'Chicago', 0);
-- INSERT MULTIPLE VALUES
-- INSERT INTO employee (emp_name, emp_location, emp_salary) VALUES ('Emily', 'Atlanta', 500000), ('Pete', 'Tampa', 100000) ;


-- UPDATE
-- modify existing rows with some conditoin
-- UPDATE table_name SET col1=val1, col2=val2 -- !! updates ALL
-- UPDATE table_name SET col1=val1, col2=val2 WHERE condition
-- UPDATE employee SET emp_name='Barry' WHERE id=2;
-- update multiple
-- UPDATE employee SET emp_location='Mumbai', emp_salary=30000
-- WHERE id=2;


-- ALTER
-- add, remove, modify and existing table columns
-- Add column - ALTER TABLE table_name ADD COLUMN col_name datatype;
-- Rename column - ALTER TABLE table_name RENAME COLUMN col_name TO new_col_name;
-- Drop column - ALTER TABLE table_name DROP COLUMN col_name;
-- Change data type - ALTER TABLE table_name ALTER COLUMN col_name TYPE datatype;
-- Constraints - ALTER TABLE table_name ALTER COLUMN col_name SET NOT NULL;


-- ALTER TABLE employee ADD COLUMN emp_designation VARCHAR(50);
-- ALTER TABLE employee RENAME COLUMN emp_designation TO emp_type;
-- ALTER TABLE employee DROP COLUMN emp_type;
-- ALTER TABLE employee ALTER COLUMN emp_salary SET NOT NULL;

-- DELETE
-- DELETE FROM table_name;
-- Deletes rows from a table
-- NEVER use without a WHERE clause
-- Use SELECT * first conirm
-- DELETE FROM table_name WHERE condition

-- SELECT * FROM employee WHERE emp_salary=0;
-- DELETE FROM employee WHERE emp_salary=0;
-- SELECT * FROM employee;

-- DROP
-- DROP TABLE IF EXISTS table_name;
-- if used in any view, function, stored proc, it will not be removed
-- use CASCADE option DROP TABLE table_name CASCADE;
-- SELECT * FROM employee;

-- TRUNCATE
-- used to delete ALL data from a table
-- more efficient than delete
-- TRUNCATE TABLE table_name;
-- Drop vs Truncate
-- -- Drop remvoes the data, index, structure
-- -- Truncate removes all rowse but retains the table structure


-- CREATE TABLE order_details (
--     id SERIAL PRIMARY KEY,
--     customer_id INT NOT NULL,
--     customer_name VARCHAR(50) NOT NULL,
--     product_name VARCHAR(50) NOT NULL,
--     ordered_from VARCHAR(50) NOT NULL,
--     order_amount INT NOT NULL,
--     order_date TIMESTAMPTZ NOT NULL,
--     delivery_date TIMESTAMPTZ,
--      delivery_timezone VARCHAR(50) DEFAULT 'UTC'
-- );

-- SELECT * FROM order_details;


-- INSERT INTO order_details 
--     (customer_id, customer_name, product_name, ordered_from, order_amount, order_date, delivery_date, delivery_timezone)
-- VALUES
--     (1001,'Linda','Pen','Store',9.82,'2025-01-02T00:00:00Z','2025-01-06T00:00:00Z','America/New_York'),
--     (1002,'Stephanie','Pencil','App',12.79,'2025-01-04T00:00:00Z','2025-01-08T00:00:00Z','Europe/London'),
--     (1003,'Deborah','Scissors','Website',13.68,'2025-01-07T00:00:00Z','2025-01-12T00:00:00Z','Asia/Tokyo'),
--     (1004,'Andrew','Pen','Website',7.62,'2025-01-07T00:00:00Z','2025-01-15T00:00:00Z','Australia/Sydney'),
--     (1005,'Steven','Chair','Website',9.76,'2025-01-07T00:00:00Z','2025-01-11T00:00:00Z','America/Los_Angeles'),
--     (1006,'Susan','Pencil','Website',10.8,'2025-01-10T00:00:00Z','2025-01-13T00:00:00Z',NULL), -- Tests UTC Default
--     (1007,'Robert','Pen','App',5.1,'2025-01-13T00:00:00Z','2025-01-23T00:00:00Z','Europe/Paris'),
--     (1008,'Melissa','Eraser','Website',9.13,'2025-01-13T00:00:00Z','2025-01-17T00:00:00Z','Asia/Dubai'),
--     (1009,'David','Pencil','Website',6.26,'2025-01-13T00:00:00Z','2025-01-15T00:00:00Z','America/Chicago'),
--     (1010,'Jennifer','Marker','Store',5.67,'2025-01-16T00:00:00Z','2025-01-21T00:00:00Z',NULL), -- Tests UTC Default
--     (1011,'Thomas','Pen','Store',9.04,'2025-01-16T00:00:00Z','2025-01-22T00:00:00Z','Asia/Singapore'),
--     (1012,'Lisa','Binder','App',7.13,'2025-01-17T00:00:00Z','2025-01-26T00:00:00Z','Europe/Berlin'),
--     (1013,'Richard','Desk','Website',15.2,'2025-01-17T00:00:00Z','2025-01-18T00:00:00Z','America/Denver'),
--     (1014,'Matthew','Binder','Website',9.24,'2025-01-17T00:00:00Z','2025-01-18T00:00:00Z','Pacific/Auckland'),
--     (1015,'Charles','Envelope','Website',6.58,'2025-01-17T00:00:00Z','2025-01-18T00:00:00Z','Asia/Seoul'),
--     (1016,'Michelle','Envelope','Store',9.17,'2025-01-18T00:00:00Z','2025-01-21T00:00:00Z',NULL), -- Tests UTC Default
--     (1017,'Sandra','Envelope','App',6.86,'2025-01-19T00:00:00Z','2025-01-23T00:00:00Z','America/Phoenix'),
--     (1018,'Mary','Pencil','App',14.31,'2025-01-22T00:00:00Z','2025-01-29T00:00:00Z','Europe/Rome'),
--     (1019,'Michael','Pen Set','Website',7.95,'2025-01-25T00:00:00Z','2025-01-29T00:00:00Z','Asia/Hong_Kong'),
--     (1020,'Donald','Eraser','App',11.13,'2025-01-26T00:00:00Z','2025-01-28T00:00:00Z','America/Toronto'),
--     (1021,'Amanda','Pen Set','Website',12.54,'2025-01-27T00:00:00Z','2025-01-29T00:00:00Z','Europe/Madrid'),
--     (1022,'Kenneth','Pen Set','App',6.82,'2025-01-31T00:00:00Z','2025-02-03T00:00:00Z', NULL);

-- SELECT * FROM order_details;

-- ALTER TABLE order_details ALTER COLUMN order_amount TYPE NUMERIC(7,2);
-- TRUNCATE TABLE order_details;

-- ALTER TABLE order_details RENAME COLUMN "customer_name" TO "customer_first_name";
-- ALTER TABLE order_details ADD COLUMN "cancel_date" TIMESTAMPTZ;

-- SELECT * FROM order_details;

-- SELECT 
-- 	product_name,
-- 	COUNT(product_name) AS "Orders",
-- 	SUM(order_amount) AS "TOTAL SALES"
-- FROM order_details
-- GROUP BY product_name;

















