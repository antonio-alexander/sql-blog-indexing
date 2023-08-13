# sql-blog-indexing (github.com/antonio-alexander/sql-blog-indexing)

The goal of this repository is to research how queries can become less performant with scale, how indexes can be used to reduce query times, how queries can be optimized and through this, how to structure queries better. This repository seeks to do the following:

- How to use the explain command to understand the query plan
- How to determine what indexes to create

This repository is meant to be a deep[er] dive into the "Performance Considerations" section in the [github.com/antonio-alexander/sql-blog-paging](github.com/antonio-alexander/sql-blog-paging) repository.

## Bibliography

These links were used while putting together this article:

- [https://dev.mysql.com/doc/refman/8.0/en/using-explain.html](https://dev.mysql.com/doc/refman/8.0/en/using-explain.html)
- [https://dev.mysql.com/doc/refman/8.0/en/explain.html](https://dev.mysql.com/doc/refman/8.0/en/explain.html)
- [https://github.com/datacharmer/test_db/tree/d01cb62fcfa4671773f167480df174d8ce4316c1](https://github.com/datacharmer/test_db/tree/d01cb62fcfa4671773f167480df174d8ce4316c1)
- [https://dev.mysql.com/doc/refman/8.0/en/table-scan-avoidance.html](https://dev.mysql.com/doc/refman/8.0/en/table-scan-avoidance.html)
- [https://www.sitepoint.com/using-explain-to-write-better-mysql-queries/](https://www.sitepoint.com/using-explain-to-write-better-mysql-queries/)
- [https://dev.mysql.com/doc/refman/8.0/en/create-index.html](https://dev.mysql.com/doc/refman/8.0/en/create-index.html)
- [https://stackoverflow.com/q/10524651](https://stackoverflow.com/q/10524651)
- [https://www.w3schools.com/mysql/mysql_join_inner.asp](https://www.w3schools.com/mysql/mysql_join_inner.asp)
- [https://dba.stackexchange.com/questions/74261/optimize-the-query-with-big-and-low-cardinality](https://dba.stackexchange.com/questions/74261/optimize-the-query-with-big-and-low-cardinality)

## Getting-Started

I think the flow of this repo will be different than other repos I've done because there isn't a core "concept" that needs to be _really_ broken down. The general goal of this repo is to attempt some scenarios and attempt to optimize them using explain. For each scenario I'll generate a schema a data set and a number of variations on the scenario to be able to compare and contrast between different kinds of optimizations.

These are the scenarios we're going to attempt to work through:

1. How does querying against a table become affected by the presence or absence of indexes? 10 employees? 100 employees? 1000 employees? 10000 employees?
2. How does the deleted column being added to the employees table affect queries that include a  constraint for deleted? 10 employees? 100 employees? 1000 employees? 10000 employees?
3. For soft-deletes, what's the comparison on using 'WHERE deleted=true' when we're using a solution with one table versus a solution with two tables
4. Comparison for migration of delete+recreate vs update in place in terms of time

Things I'm curious about:

- Is there any room to optimize inserts?
- How do triggers affect mass inserts/mass updates?
- Is there an overall difference in on-disk size if using varchar() vs char()?

There's a lot of ways to get the test database into the image, I want to give you the opportunity to figure out how to do it.

## Explain and Indexes

Before we dive too deep into each scenario, I want to briefly describe both EXPLAIN and INDEX. EXPLAIN is a command you can use with any query to determine how the SQL server will go about determining the most efficient way to complete your query amongst a host of variables. An INDEX works hand-in-hand with query plans, indexes (sort of) pre-calculate some of the effort of a given query and if done correctly can significantly impact the query plan.

> I wanted to give some examples, but EXPLAIN and INDEX are a bit impossible out of context, so we'll get into the scenarios now-ish

## Scenario 1: Adding Indexes to Optimize Queries with Constraints

This scenario uses the [https://github.com/datacharmer/test_db](https://github.com/datacharmer/test_db) test database as a starting point. We're going to do a benchmark of the database as-is and see what we can do to optimize the employees table for specific queries, especially at scale.

A hard question is what query we can attempt to optimize; I have a little foreknowledge about how indexes work (from sql-blog-pagination) so I'll do some min/max work on some of the fields given the data set:

- emp_no: 1001 - 499999 (300024)
- birth_date: 1952-02-01 - 1965-02-01 (300024)
- hire_date: 1984-01-01 - 2000-01-28 (300024)
- gender: M(179973), F(120051)

Although first_name and last_name can be searched (and indexed), there's not much useful context (at scale) for searching using either, especially at scale. I'm going to make a strong assumption that there's the MOST variation in hire_date as opposed to emp_no or birth_date; I think that more variation equals more to optimize.

So let's attempt to query for employees that have a birthdate AFTER the median hire_date (1992-01-01):

```sql
SELECT COUNT(*) FROM employees WHERE hire_date >= '1992-01-01';
```

This gives us 87049 different employees which should cover the scale we want to work with. So to start, let's do some basic benchmarking:

```sql
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 10;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 100;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 1000;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 10000;
```

These are the results of the earlier queries:

- 10 employees: 0.010s
- 100 employees: 0.010s
- 1000 employees: 0.051s
- 10000 employees: 0.071s

Looking at the above, it's clear that it's not until we try to get more than 100 employees that the plan gets weird and it takes longer, maybe we can optimize those two middle queries and see if we can get those query times down.

<!-- THOSE ARE ROOKIE NUMBERS (gif) -->

Also, I'll show the indexes that exist on the table:

```sql
SHOW INDEXES FROM employees;
```

```log
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Ignored |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
| employees |          0 | PRIMARY  |            1 | emp_no      | A         |      299335 |     NULL | NULL   |      | BTREE      |         |               | NO      |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+
```

This indicates that there's only a single index on emp_no (the primary key); with that done, let's take a look at the EXPLAINs for those two middle queries at 100 and 1000 employees.

```sql
EXPLAIN SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 100;
```

```log
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 299335 | Using where |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
```

Decoding this EXPLAIN, we can determine the following:

- it wasn't able to use any keys (note that possible_keys, key, key_len and ref are null)
- to read 100 rows, we had to read 299335 (out of 300024)
- the type is "ALL" which generally communicates a table scan meaning that its fastest to just scan the entire table (than doing ANYTHING else)

```sql
EXPLAIN SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 1000;
```

```log
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 299335 | Using where |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
```

This explain is pretty much the same as the above (except it's 10x the data). So, I think we can optimize queries against hire_date by adding an index:

```sql
CREATE INDEX idx_hire_date ON employees (hire_date);
```

> Keep in mind that we're creating an index to optimize queries rather than a constraint (e.g. a unique index for a primary key)

Creating an index takes time (in this case it took 1.936s) and increases the on-disk size of the table; hopefully an OK price to optimize the queries. Let's try to benchmark again and see what the results are:

```sql
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 10;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 100;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 1000;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 10000;
```

These are the results of the earlier queries:

- 10 employees: 0.004s (vs 0.010s)
- 100 employees: 0.007s (vs 0.010s)
- 1000 employees: 0.030s (vs 0.051s)
- 10000 employees: 0.115s (vs 0.071s)

In comparison to our earlier benchmark, we see a significant decrease in query time, we're talking orders of magnitude difference such that in the time we previously could only query 10 employees, now we can query 10000 employees. Now the fun part, let's re-examine the explain to see if there are any noticeable differences:

```sql
EXPLAIN SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' LIMIT 1000;
```

```log
+------+-------------+-----------+-------+---------------+---------------+---------+------+--------+-----------------------+
| id   | select_type | table     | type  | possible_keys | key           | key_len | ref  | rows   | Extra                 |
+------+-------------+-----------+-------+---------------+---------------+---------+------+--------+-----------------------+
|    1 | SIMPLE      | employees | range | idx_hire_date | idx_hire_date | 3       | NULL | 149667 | Using index condition |
+------+-------------+-----------+-------+---------------+---------------+---------+------+--------+-----------------------+
```

Now this is significantly more interesting, we can see that it's no longer using 'ALL' (so no table scan) and that it has an index it can use. We also see that the number of rows it had to scan through was halved (149667 vs 299335). To put this in context, if we put together this database and found that a lot of the queries that were run against the database involved hire_date, we could add this index and significantly reduce the query time.

In short, for this specific query, adding an index would significantly reduce the query time __for this specific query__ OTHER queries may not be able to take advantage of the index or may have some ability to take advantage of the index, SQL could also find that a table scan is the fastest way as well.

## Scenario 2: How Do Logical Deletes Affect Queries with Constraints

<!-- How does the deleted column being added to the employees table affect queries that include a  constraint for deleted? 10 employees? 100 employees? 1000 employees? 10000 employees? -->

When I implemented [sql-blog-soft-deletes](github.com/antonio-alexander/sql-blog-soft-deletes); there was a question I had that didn't make sense to ask (or answer) within that effort: I wanted to know how performance was affected when adding the deleted column as a query and if using a separate table was more performant. I'm pretty sure using the table is more performant, but I wanted to quantify it (and at scale).

Because I need the data set to prove either at scale, we'll be modifying the database post [scenario 1](#scenario-1-adding-indexes-to-optimize-queries-with-constraints). We'll do the following to attempt to prove out this scenario:

1. Modify the existing table to add the deleted column to gather some benchmarks
2. Attempt to optimize attempts to query those deleted (or un-deleted) employees
3. Create a new table to copy those "deleted" employees
4. Attempt to optimize attempts to query those deleted (or un-deleted) employees using the different table

> Keep in mind that there are strong architectural implications with regards to data consistency when performing a logic delete using an alternate table. Things such as enforcing foreign key constraints, handling auditing and triggers. It's always easier if you leave data consistency up to SQL

To add the deleted column to the existing employees database, we'll need to enter the following sql:

```sql
ALTER TABLE employees ADD COLUMN deleted bool DEFAULT false;
```

We're also going to simulate deleting "half" of the employees, but we want to do this in a way that does an unbiased distribution, so we don't have to worry about anything weird when we attempt to add indexes/optimize. We'll do this by deleting employees that have an even employee number. I found this on [https://tableplus.com/blog/2019/09/select-rows-odd-even-value.html](https://tableplus.com/blog/2019/09/select-rows-odd-even-value.html):

```sql
UPDATE employees SET deleted=true WHERE mod(emp_no,2) = 0;
```

This statement should take a while to complete (a couple of seconds); but once it's done, we should have a relative even distribution of "deleted" employees. We can confirm the following after we set that command, that out of 300,024 employees, we've deleted 150,012 of them. So here are the queries we want to attempt to optimize:

```sql
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE deleted=true LIMIT 10;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE deleted=true LIMIT 100;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE deleted=true LIMIT 1000;
SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE deleted=true LIMIT 10000;
```

These are the results of those queries:

- 10 employees: .09s - .020s
- 100 employees: .007s - .052s
- 1000 employees: .016s - .070s
- 10000 employees: .054s - .092s

> Keep in mind that although i'm doing benchmarks, there are __tons__ of variables that affect how long a query takes that have little to do with sql but more to do with available RAM, disk speed (e.g. if its SSD or spinny) etc...

In comparison to the benchmarks we did earlier, there was significantly more variation in the queries, so I've given a range instead of a single value. Before we start modifying the table, let's do an EXPLAIN on the query with 1000 employees:

```sql
EXPLAIN SELECT emp_no, birth_date, first_name, last_name, gender, hire_date FROM employees WHERE hire_date >= '1992-01-01' AND deleted=true LIMIT 1000;
```

```log
+------+-------------+-----------+-------+---------------+---------------+---------+------+--------+------------------------------------+
| id   | select_type | table     | type  | possible_keys | key           | key_len | ref  | rows   | Extra                              |
+------+-------------+-----------+-------+---------------+---------------+---------+------+--------+------------------------------------+
|    1 | SIMPLE      | employees | range | idx_hire_date | idx_hire_date | 3       | NULL | 149667 | Using index condition; Using where |
+------+-------------+-----------+-------+---------------+---------------+---------+------+--------+------------------------------------+
```

So...let's start with the obvious first step (which I don't think will make a difference). Let's add an index to the deleted column, and see if it changes any of the results:

```sql
CREATE INDEX idx_deleted ON employees(deleted);
```

Let's attempt to run the same queries we ran earlier:

- 10 employees: .008s - .017s (.09s - .020s)
- 100 employees: .011s - .019s (.007s - .052s)
- 1000 employees: .037s - .082s (.016s - .070s)
- 10000 employees: .094s - .231s (.054s - .092s)

In comparison to the previous benchmarks, there is what seems like improvement, but the range is still all over the place. There's not a clear and consistent change in the query time and at a glance, I'll concede that adding a index to the deleted column did not improve our query.

There's a [stack overflow discussion](https://stackoverflow.com/q/10524651) that I think is super useful, but the gist is that boolean columns (in general) have _very low_ cardinality; this means that in a given data set there's not a huge variation of boolean values so adding an index DOES have benefits, but is an exercise in diminishing returns. The exercise I did early in [scenario 1](#scenario-1-adding-indexes-to-optimize-queries-with-constraints) was to try to show the min/max of the columns; this (kind of) communicated the level of cardinality available within each of those columns.

> You can also think that indexes use binary trees (at least in sql) and that with a boolean column, the binary tree is __INCREDIBLY__ shallow...we can call it a binary shrub: valid values for the column are true/false/null

But we have an alternate solution (suggested by the caveman DBA): we can create another table that mirrors employees, but instead of holding "employees", it'll hold "deleted "employees" which don't exist in the other table. This will be more of a "proof of concept" because we're not going to attempt to solve the problem with foreign key constraints.

To create this other table, we can execute the following:

```sql
CREATE TABLE employees_deleted (
    emp_no INT NOT NULL,
    FOREIGN KEY (emp_no) REFERENCES employees(emp_no)
);
```

We can populate this table with deleted employees (from earlier) by entering the following:

```sql
INSERT INTO employees_deleted
    SELECT emp_no FROM employees WHERE deleted=true;
```

Alternatively, if we wanted to do this from scratch (without adding a deleted column), we could do the following to _delete_ every other user:

```sql
INSERT INTO employees_deleted
    SELECT emp_no from employees WHERE mod(emp_no,2) = 0;
```

We can semi-confirm that this has been successful with the following queries:

```sql
SELECT count(*) FROM employees WHERE deleted=true;
SELECT count(*) FROM employees_deleted;
```

Both queries should have a count of 150,012 rows. Now that we have a table of deleted employees and a table of employees, we can query the _deleted_ employees by doing a join on the two tables:

```sql
SELECT employees.emp_no, employees.birth_date, employees.first_name, employees.last_name, employees.gender, employees.hire_date FROM employees INNER JOIN employees_deleted ON employees.emp_no = employees_deleted.emp_no LIMIT 10;
```

Inner joins aren't special, but we may be able to optimize this because there's an __exploitable__ cardinality that we can get from emp_no that we couldn't get from a boolean column.

```sql
SELECT employees.emp_no, employees.birth_date, employees.first_name, employees.last_name, employees.gender, employees.hire_date FROM employees INNER JOIN employees_deleted ON employees.emp_no = employees_deleted.emp_no LIMIT 10;
SELECT employees.emp_no, employees.birth_date, employees.first_name, employees.last_name, employees.gender, employees.hire_date FROM employees INNER JOIN employees_deleted ON employees.emp_no = employees_deleted.emp_no LIMIT 100;
SELECT employees.emp_no, employees.birth_date, employees.first_name, employees.last_name, employees.gender, employees.hire_date FROM employees INNER JOIN employees_deleted ON employees.emp_no = employees_deleted.emp_no LIMIT 1000;
SELECT employees.emp_no, employees.birth_date, employees.first_name, employees.last_name, employees.gender, employees.hire_date FROM employees INNER JOIN employees_deleted ON employees.emp_no = employees_deleted.emp_no LIMIT 10000;
```

These are the results of those queries:

- 10 employees: .008s - .016s (.008s - .017s)
- 100 employees: .011s - .017s (.011s - .019s)
- 1000 employees: .030s - .046s (.037s - .082s)
- 10000 employees: .143s - .165s (.094s - .231s)

In my opinion, there's not a huge difference from the previous queries in terms of overall timing, but I do think there was more precision in the queries than before. An EXPLAIN gives us the following:

```log
+------+-------------+-------------------+--------+---------------+---------+---------+------------------------------------+--------+-------------+
| id   | select_type | table             | type   | possible_keys | key     | key_len | ref                                | rows   | Extra       |
+------+-------------+-------------------+--------+---------------+---------+---------+------------------------------------+--------+-------------+
|    1 | SIMPLE      | employees_deleted | index  | emp_no        | emp_no  | 4       | NULL                               | 150306 | Using index |
|    1 | SIMPLE      | employees         | eq_ref | PRIMARY       | PRIMARY | 4       | employees.employees_deleted.emp_no | 1      |             |
+------+-------------+-------------------+--------+---------------+---------+---------+------------------------------------+--------+-------------+
```

It feels like there's more room for optimization or there's something _else_ I don't know about, but I think this is good enough to communicate that there's a certain difficulty associated with optimizing queries involving columns with very low cardinality.

## Scenario 3: How Can You Optimize a Table for Pagination?

This scenario is _almost_ a copy+paste from [github.com/antonio-alexander/sql-blog-paging](github.com/antonio-alexander/sql-blog-paging). To briefly summarize: for large data sets, it's not always practical to query the entire data set, so instead you query a subset of the data or more functionally, a page. A page, for all intents and purposes, is a consistent query you can make for the _same_ data set. You can think of a bank statement that shows you all of your transactions for a certain criteria and if its over a certain number of transactions, instead of giving you every transaction it'll give you the first page of transactions.

The purpose of this scenario is to try to optimize those queries; we're going to start from a state of zero indexes. Similar to other scenarios, we have a handful of columns with reasonable [cardinality](https://en.wikipedia.org/wiki/Cardinality_(SQL_statements)) such that there's room to _actually_ optimize.

The queries for pagination come in a handful of forms, so we'll start with the simple (...well bad) queries to give us a starting point to what's happening:

```sql
EXPLAIN SELECT * FROM employees LIMIT 10;
EXPLAIN SELECT * FROM employees;
```

```log
MariaDB [employees]> EXPLAIN SELECT * FROM employees LIMIT 10;
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 299246 |       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
1 row in set (0.001 sec)

MariaDB [employees]> EXPLAIN SELECT * FROM employees;
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 299246 |       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------+
1 row in set (0.001 sec)
```

Unsurprisingly, the explain indicates the queries are totally un-optimized: there were no possible keys to use and it had to read almost every row (299246 of 300024) to perform each query. Alterantively, this is an un-optimized query because the type is ALL indicating that a table scan was the fastest way to query the data. An ideal result (ignoring the time to complete the query) would be that the query with the limit ONLY had to read 10 rows and would use a type other than ALL.

The below query attempts to approximate offset/limit based pagination (we're just querying employee number to nudge our explain in the right direction)

```sql
EXPLAIN SELECT emp_no FROM employees LIMIT 10;
EXPLAIN SELECT emp_no FROM employees LIMIT 10 OFFSET 10;
```

```log
MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees LIMIT 10;
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | index | NULL          | PRIMARY | 4       | NULL | 299246 | Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
1 row in set (0.001 sec)

MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees LIMIT 10 OFFSET 10;
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | index | NULL          | PRIMARY | 4       | NULL | 299246 | Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+-------------+
1 row in set (0.001 sec)
```

These queries are a bit more specific, the "nudge" we gave is by using emp_no (the primary key) as criteria which allows use of index associated with the primary key; still we read almost all of the rows, but the type is not ALL but index. This is a better query and should generally be more performant, especially at scale. The queries we did above are based on offset/limit based criteria while the next will be more cursor based pagination:

```sql
EXPLAIN SELECT emp_no FROM employees WHERE emp_no > 10010 LIMIT 10;
EXPLAIN SELECT emp_no FROM employees WHERE emp_no < 10010 LIMIT 10;
```

```log
MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees WHERE emp_no > 10010 LIMIT 10;
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows   | Extra                    |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+--------------------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL | 149623 | Using where; Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+--------+--------------------------+
1 row in set (0.001 sec)

MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees WHERE emp_no < 10010 LIMIT 10;
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows | Extra                    |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL | 9    | Using where; Using index |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+--------------------------+
1 row in set (0.001 sec)
```

These queries still use LIMIT, but don't use OFFSET and there's a distinct difference to the EXPLAINs: you'll see that the type is different (range) and more interesting, the number of rows is now significantly reduced (to both 149623 and 9 respectively).

Although there's a big difference in performance, I'll ruin the success in noting that calculating the cursors (knowing the start emp_no) requires an additional query which could kill your performance in aggregate. Let's say that our data set is 10 pages of 100 employees each sorted by employee number, we can pre-calculate the cursors (10 pages of 100 employees each) for each page with the following query:

```sql
SET @a = 0;
SELECT emp_no FROM employees WHERE emp_no > 10100 AND ((@a := @a + 1) % 100 = 0 OR @a %100 = 99) LIMIT 10;
```

```log
MariaDB [employees]> SELECT emp_no FROM employees WHERE emp_no > 10100 AND ((@a := @a + 1) % 100 = 0 OR @a %100 = 99) LIMIT 10;
+--------+
| emp_no |
+--------+
|  10199 |
|  10200 |
|  10299 |
|  10300 |
|  10399 |
|  10400 |
|  10499 |
|  10500 |
|  10599 |
|  10600 |
+--------+
10 rows in set (0.001 sec)
```

From this query we can determine that the first page is from emp_no 10100 to 10199 so forth and so on.

> This query is very safe to copy+paste because it is a complete solution for situations where the index of the page (e.g., emp_no) isn't consecutive; this handles the situation where a row has been deleted

Once we have the cursors, we can then query a specific page (this would be page 9):

```sql
EXPLAIN SELECT emp_no, first_name, last_name, gender FROM employees WHERE emp_no BETWEEN 10500 AND 10599 ORDER BY emp_no ASC;
```

```log
MariaDB [employees]> EXPLAIN SELECT emp_no, first_name, last_name, gender FROM employees WHERE emp_no BETWEEN 10500 AND 10599 ORDER BY emp_no ASC;
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
| id   | select_type | table     | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
|    1 | SIMPLE      | employees | range | PRIMARY       | PRIMARY | 4       | NULL | 100  | Using where |
+------+-------------+-----------+-------+---------------+---------+---------+------+------+-------------+
1 row in set (0.001 sec)
```

As you can see above, this seems relatively optimized, the number of rows read is equal to the number of rows we're interested in and the type is not _ALL_. This technique can work for different kinds of pages ordered differently as long as you index by a unique column (using a non-unique column is possible, but you most likely won't get a consistent number of items).

## Scenario 4: How to Optimize Columns With Low Cardinality

This scenario will be very extreme, but it's an attempt to try to solve the problem when you have a column with significantly low cardinality. Data types with low cardinality are things like enums or booleans which have very few unique values. With our sample data set, the column with the least amount of cardinality is the gender column (M or F).

Let's try to do an EXPLAIN on the following queries:

```sql
EXPLAIN SELECT emp_no FROM employees WHERE gender ='M';
EXPLAIN SELECT emp_no FROM employees WHERE gender ='F';
```

```log
MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees WHERE gender ='M';
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 299246 | Using where |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
1 row in set (0.001 sec)

MariaDB [employees]> EXPLAIN SELECT emp_no FROM employees WHERE gender ='F';
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
| id   | select_type | table     | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
|    1 | SIMPLE      | employees | ALL  | NULL          | NULL | NULL    | NULL | 299246 | Using where |
+------+-------------+-----------+------+---------------+------+---------+------+--------+-------------+
```

From the above EXPLAINs, we see that a table scan (ALL) was the fastest way to complete this query. If we run the queries themselves, these are the times we get (on average):

```sql
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender = 'F';
SELECT emp_no, first_name, last_name, gender FROM employees WHERE gender = 'M';
```

- Male: 0.301s - 0.319s
- Female: 0.278s - 0.355s

This isn't terrible (in general), it's probably really great given the circumstances, but maybe there's a way we can optimize it using JOINs. What if we created two new tables, employee_male and employee_female (with a single column):

```sql
CREATE TABLE employees_male (
    emp_no INT NOT NULL,
    FOREIGN KEY (emp_no) REFERENCES employees(emp_no)
);

CREATE TABLE employees_female (
    emp_no INT NOT NULL,
    FOREIGN KEY (emp_no) REFERENCES employees(emp_no)
);

INSERT INTO employees_male SELECT emp_no FROM employees WHERE gender = 'M';
INSERT INTO employees_female SELECT emp_no FROM employees WHERE gender = 'F';
```

With these new tables, we should be able to query for our male (or female) employees by doing a RIGHT JOIN:

```sql
EXPLAIN SELECT e.emp_no, first_name, last_name, gender FROM employees AS e RIGHT JOIN employees_male AS em ON e.emp_no = em.emp_no;
```

```log
MariaDB [employees]> EXPLAIN SELECT e.emp_no, first_name, last_name, gender FROM employees AS e RIGHT JOIN employees_male AS em ON e.emp_no = em.emp_no;
+------+-------------+-------+--------+---------------+---------+---------+---------------------+--------+-------------+
| id   | select_type | table | type   | possible_keys | key     | key_len | ref                 | rows   | Extra       |
+------+-------------+-------+--------+---------------+---------+---------+---------------------+--------+-------------+
|    1 | SIMPLE      | em    | index  | NULL          | emp_no  | 4       | NULL                | 180154 | Using index |
|    1 | SIMPLE      | e     | eq_ref | PRIMARY       | PRIMARY | 4       | employees.em.emp_no | 1      |             |
+------+-------------+-------+--------+---------------+---------+---------+---------------------+--------+-------------+
```

This explain, while not great, is definitely a bit more promising, let's skip to how long it takes to do the queries we did earlier:

```sql
SELECT e.emp_no, first_name, last_name, gender FROM employees AS e RIGHT JOIN employees_male AS em ON e.emp_no = em.emp_no;
SELECT e.emp_no, first_name, last_name, gender FROM employees AS e RIGHT JOIN employees_female AS ef ON e.emp_no = ef.emp_no;
```

These are the results of those queries:

- Male: 1.090s - 1.102s (vs 0.301s - 0.319s)
- Female: 0.743s - 0.759s (vs 0.278s - 0.355s)

This is a bit unexpected (for me) and there may still be some situations where this kind of setup makes sense, but in general, what it means is that given the query, using a WHERE constraint is the fastest way.
