<!-- TOC --><a name="sql-postgresql-notes"></a>
# SQL & PostgreSQL notes

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [SQL & PostgreSQL notes](#sql-postgresql-notes)
   * [Basic definitions](#basic-definitions)
   * [Database structure](#database-structure)
   * [Creating a table with CREATE TABLE](#creating-a-table-with-create-table)
      + [Keywords](#keywords)
      + [Identifiers](#identifiers)
      + [Data types](#data-types)
   * [Deleting a table using DROP TABLE](#deleting-a-table-using-drop-table)
   * [Inserting records into a table with INSERT INTO](#inserting-records-into-a-table-with-insert-into)
      + [Inserting multiple records](#inserting-multiple-records)
   * [Updating records](#updating-records)
   * [Retrieving data from tables using SELECT FROM](#retrieving-data-from-tables-using-select-from)
      + [Retrieving specific columns](#retrieving-specific-columns)
      + [Performing calculations on retrieved columns](#performing-calculations-on-retrieved-columns)
      + [Renaming retrieved columns ](#renaming-retrieved-columns)
      + [Operators on retrieved strings ](#operators-on-retrieved-strings)
   * [Filtering records with WHERE](#filtering-records-with-where)
      + [Comparison operators within a WHERE clause](#comparison-operators-within-a-where-clause)
      + [Compound WHERE clauses](#compound-where-clauses)
      + [Calculations in WHERE clauses](#calculations-in-where-clauses)
   * [Updating rows using UPDATE](#updating-rows-using-update)
   * [Deleting rows using DELETE FROM](#deleting-rows-using-delete-from)
   * [Very basics of designing a database ](#very-basics-of-designing-a-database)
      + [Types of relationships](#types-of-relationships)
         - [One to Many](#one-to-many)
         - [Many to One](#many-to-one)
         - [One to One](#one-to-one)
         - [Many to Many](#many-to-many)
   * [Primary and foreign keys](#primary-and-foreign-keys)
      + [Creating primary keys](#creating-primary-keys)
      + [Creating foreign keys](#creating-foreign-keys)
      + [Queries on associated data](#queries-on-associated-data)
      + [Constraints on foreign keys during INSERTIONS](#constraints-on-foreign-keys-during-insertions)
      + [Constraints on foreign keys during DELETIONS](#constraints-on-foreign-keys-during-deletions)
   * [Joins](#joins)
      + [Four types of joins](#four-types-of-joins)
      + [Joins with WHERE clauses](#joins-with-where-clauses)
      + [Three way joins](#three-way-joins)
   * [Grouping and aggregations](#grouping-and-aggregations)
      + [Combining GROUP BY and aggregates](#combining-group-by-and-aggregates)
      + [A gotcha! with COUNT](#a-gotcha-with-count)
      + [GROUP BY with JOINs](#group-by-with-joins)
      + [Filter out groups by HAVING](#filter-out-groups-by-having)
   * [Sorting rows using ORDER BY](#sorting-rows-using-order-by)
      + [Ordering by by multiple columns](#ordering-by-by-multiple-columns)
   * [Offset and limits](#offset-and-limits)
   * [Selecting distinct values with DISTINCT](#selecting-distinct-values-with-distinct)
   * [Utility operations and keywords](#utility-operations-and-keywords)
   * [Writing subqueries](#writing-subqueries)
      + [Understanding the shape of query results to simplify subqueries](#understanding-the-shape-of-query-results-to-simplify-subqueries)
      + [SELECT can only use subqueries returning a single value](#select-can-only-use-subqueries-returning-a-single-value)
      + [FROM can only use subqueries compatible with outer constructs](#from-can-only-use-subqueries-compatible-with-outer-constructs)
      + [JOIN can only use subqueries compatible with ON criteria](#join-can-only-use-subqueries-compatible-with-on-criteria)
      + [Subqueries that can be used in WHERE depend on the operator](#subqueries-that-can-be-used-in-where-depend-on-the-operator)
         - [ALL and SOME operators inside a WHERE clause](#all-and-some-operators-inside-a-where-clause)
      + [Correlated subqueries](#correlated-subqueries)
         - [A SELECT without a FROM](#a-select-without-a-from)
   * [Using sets in SQL queries](#using-sets-in-sql-queries)
   * [Order of execution of a SELECT statement](#order-of-execution-of-a-select-statement)
   * [PostgreSQL local installation](#postgresql-local-installation)
      + [pgAdmin on macOS](#pgadmin-on-macos)
   * [Structure of a Postgres server](#structure-of-a-postgres-server)
   * [Fast rules for storing numbers](#fast-rules-for-storing-numbers)
   * [Storing text ](#storing-text)
   * [Storing booleans](#storing-booleans)
   * [Storing date and time values](#storing-date-and-time-values)
   * [Row level validations or constraints](#row-level-validations-or-constraints)
      + [Is the value defined?](#is-the-value-defined)
      + [Do we have a default value?](#do-we-have-a-default-value)
      + [Is the value unique?](#is-the-value-unique)
         - [Adding multicolumn uniqueness](#adding-multicolumn-uniqueness)
      + [Dropping a constraint](#dropping-a-constraint)
      + [Is value passing conditional checks?](#is-value-passing-conditional-checks)
         - [Multicolumn checks](#multicolumn-checks)
   * [Typecasting values](#typecasting-values)
   * [App vs database validations](#app-vs-database-validations)
   * [Database design ](#database-design)
      + [Database schema design tools](#database-schema-design-tools)

<!-- TOC end -->


<!-- TOC --><a name="basic-definitions"></a>
## Basic definitions

**Database**: A tool to store information. We take some amount of data from some source, put it inside of a database where it gets persisted either in memory or on disk. Postgres is one type of database.

**Client**: We use a client to connect to a database. A client can be any piece of software, so it might even be some API server you put together. There can be different types of clients.

**SQL**: Structured Query Language or SQL is how we interact with a database when we connect to it. SQL is kind of like a programming language. It tells our database some information that we want to put inside of it that we want to retrieve, update or delete (CRUD operations).

**SQL != PostgreSQL**
* SQL is a language used to interact with Databases while Postgres is a database system as a whole
* SQL is used with other databases too with slight differences in the way it is defined. However, knowing basic SQL helps you work with most database systems such as Postgres, SQL server, MySQL, and so on

<!-- TOC --><a name="database-structure"></a>
## Database structure

**Table**: A table is something that sits inside of our database that is going to store a collection of records (Ex: Cities). A table may contain many records. Each row in a table corresponds to a single record.

**Columns**: Inside a table, we may have many different columns and each column is going to store some information about a very specific property of some record. Ex: Store the name of the country, its population and so on. ***Column data type***: Finally, we say that each of those different columns are going to store a very different specific kind or type of information. Ex: Name is a string, population a number, etc.

| name (VARCHAR) | population (INT) | area (FLOAT) |
| -------------- | ---------------- | ------------ |
| New York       | 8419600          | 783.8        |
| Tokyo          | 13929286         | 2194.1       |
| Paris          | 2140526          | 105.4        |
| Mumbai         | 12442373         | 603.4        |
| Sydney         | 5312163          | 12367.7      |


**Start a PostgreSQL server after installation**: 

Use the `psql` command on CLI to get into the database system

**Create database**: 

```SQL
CREATE DATABASE database_name;
```

**Delete a database**: 

```SQL
DROP DATABASE database_name;
```

It is important to note that you cannot drop a database if there are active connections to it. 
You must ensure no users or applications are connected to the database before attempting to drop it. You can also add IF EXISTS to prevent errors if the database does not exist:

```SQL
DROP DATABASE IF EXISTS database_name;
```

**List the databases**: 

`\l` or `\list ` command inside the database system i.e `psql` env

<!-- TOC --><a name="creating-a-table-with-create-table"></a>
## Creating a table with CREATE TABLE

Use the `CREATE TABLE` syntax with name and arguments containing a comma (`,`) separated list of column names and data types for it.

```sql
CREATE TABLE cities (
  name VARCHAR(50),
  country VARCHAR(50),
  population INTEGER,
  area INTEGER
);
```

**Note**: The semicolon (`;`) at the end is very important in SQL. Do not forget to append it to your statement!

<!-- TOC --><a name="keywords"></a>
### Keywords

SQL has keywords that are reserved for instructions. Ex: `CREATE`, `TABLE`, `SELECT`, etc. These are always written in UPPERCASE as per convention.

<!-- TOC --><a name="identifiers"></a>
### Identifiers

SQL has identifiers for what we want the database to "act on". These can be names for tables, columns, etc. These are always written in lowercase as per convention.

<!-- TOC --><a name="data-types"></a>
### Data types

String data types in PostgreSQL are generally listed as `VARCHAR` followed by the *maximum* length of characters in parentheses (`()`). It stands for "variable character".

For an `INTEGER` type, the range is ~-2billion to ~+2billion.

Table explaining the various data types:
| Data Type        | Example Value           | Description                             |
| ---------------- | ----------------------- | --------------------------------------- |
| INTEGER          | 1234                    | Whole numbers for basic counting        |
| BIGINT           | 9876543210              | Large whole numbers, often used for IDs |
| SERIAL           | auto-increment (1,2)    | Auto-incrementing integer for keys      |
| VARCHAR(n)       | 'New York'              | Variable-length character string        |
| TEXT             | 'Description text'      | Large arbitrary-length text             |
| BOOLEAN          | TRUE                    | True/false logical values               |
| DATE             | '2025-11-06'            | Stores calendar dates                   |
| TIMESTAMP        | '2025-11-06 19:44'      | Date and time (no timezone info)        |
| NUMERIC(p,s)     | 123456.78               | High-precision numeric values           |
| REAL             | 3.14159                 | Single precision floating-point numbers |
| DOUBLE PRECISION | 2.718281828459          | Double precision floating-point numbers |
| BYTEA            | \\xDEADBEEF             | Binary data (blobs, files)              |
| UUID             | 'a0eebc99-9c0b-4ef8...' | Universally unique identifiers          |
| JSON             | '{"a":1,"b":2}'         | Structured data in JSON format          |

<!-- TOC --><a name="deleting-a-table-using-drop-table"></a>
## Deleting a table using DROP TABLE

Use the `DROP` command to delete a table and its entire contents

Ex:
```SQL
DROP TABLE photos;

-- DROP successful!
-- DROP complete
```

<!-- TOC --><a name="inserting-records-into-a-table-with-insert-into"></a>
## Inserting records into a table with INSERT INTO

Use the `INSERT INTO` command syntax. It needs a table name followed by arguments for column names. This in turn is followed by `VALUE` and an argument list containing values for the listed columns.

```SQL
INSERT INTO cities (name, country, population, area)
VALUES ('Tokyo', 'Japan', 38505000, 8223);
```

**Note**: The order of values inserted must match the the column order. Else, it could result in an error or worse, a mess where the column values do not correspond to the column name (Ex: Japan for name and Tokyo for country)

<!-- TOC --><a name="inserting-multiple-records"></a>
### Inserting multiple records

Use the same  `INSERT INTO` command but the arguments to  `VALUES` can contain multiple records enclosed in parentheses (`()`) and separated by a comma (`,`) like shown below:
```SQL
INSERT INTO cities (name, country, population, area)
VALUES 
	('Delhi', 'India', 28125000, 8223),
	('Shanghai', 'China', 22125000, 4015),
	('Sao Paulo', 'Brazil', 20935000, 3043);
```

<!-- TOC --><a name="updating-records"></a>
## Updating records

Use the `UPDATE` command.
Syntax: `UPDATE <table> SET <column>=<value> WHERE <condition>`

We will learn about `WHERE` later. For now, it is just a condition applied on the rows which results in only those rows that pass/satisfy it.

```SQL
-- Update all products that have no price defined to a default value
UPDATE products
SET price = 99999
WHERE price IS NULL; -- Make sure to use `IS` instead of `=` in WHERE of an UPDATE

-- In a WHERE of a SELECT, we can use `=` (See next section)
```

<!-- TOC --><a name="retrieving-data-from-tables-using-select-from"></a>
## Retrieving data from tables using SELECT FROM

Use the `SELECT` statements to query from a table.

```SQL
SELECT * FROM cities;

-- name	country	population	area
-- Tokyo	Japan	38505000	8223
-- Delhi	India	28125000	8223
-- Shanghai	China	22125000	4015
-- Sao Paulo	Brazil	20935000	3043
```

**Note**:
1. `*` is a wildcard for selecting all the columns in a table
2. `--` is a comment in SQL. We are using that to display examples results of the `SELECT` operation

<!-- TOC --><a name="retrieving-specific-columns"></a>
### Retrieving specific columns

We can replace `*` with a comma (`,`) separated list of columns. 

The order of the columns do not matter and the results will be listed in the order specified.

```SQL
SELECT name, population FROM cities;

-- name	population
-- Tokyo	38505000
-- Delhi	28125000
-- Shanghai	22125000
-- Sao Paulo	20935000
```

**Note**: We can also repeat the same column name multiple times (Ex: `name, name, name`) but there are very rare cases where that would be useful.

<!-- TOC --><a name="performing-calculations-on-retrieved-columns"></a>
### Performing calculations on retrieved columns

It is possible to use certain operators on or between columns within a `SELECT` command in order to process the data further.

Example of calculating the population density by dividing the population column by the area column:
```SQL
SELECT name, population / area FROM cities;

-- name	?column?
-- Tokyo	4682
-- Delhi	3420
-- Shanghai	5510
-- Sao Paulo	6879
```

**Note**: Common arithmetic operators include `+`, `-`, `*`, `/`, `^` (exponent), `|/` (square root), `@` (absolute value), and `%` (remainder).

<!-- TOC --><a name="renaming-retrieved-columns"></a>
### Renaming retrieved columns 

We can rename columns in a `SELECT` statement using the `AS` keyword. It is especially useful on calculated / processed columns.

Examples:
```SQL
SELECT name, population AS city_population 
FROM cities;

-- name	city_population
-- Tokyo	38505000
-- Delhi	28125000
-- Shanghai	22125000
-- Sao Paulo	20935000
```

```SQL
SELECT name, population / area AS population_density 
FROM cities;

-- name	population_density
-- Tokyo	4682
-- Delhi	3420
-- Shanghai	5510
-- Sao Paulo	6879
```

<!-- TOC --><a name="operators-on-retrieved-strings"></a>
### Operators on retrieved strings 

Apart from numerical i.e arithmetic operators, we have operators & functions for string values.
The common string operators are:
1. `||` : Join two strings. **Note**: we might need to add a space (`' '`) literal in between for formatting
2. `CONCAT()`: Join two strings
3. `LOWER()`: Gives a lowercase string
4. `LENGTH()`: Gives the number of characters in a string
5. `UPPER()`: Gives an uppercase string

**Note**: It is better to give a name the column generated from these operations using the `AS` keyword

```SQL
SELECT name || country AS location
FROM cities;

-- location
-- TokyoJapan
-- DelhiIndia
-- ShanghaiChina
-- Sao PauloBrazil
```

```SQL
SELECT name || ', ' || country AS location
FROM cities;

-- location
-- Tokyo, Japan
-- Delhi, India
-- Shanghai, China
-- Sao Paulo, Brazil
```

Functions can be ***nested***! 
```SQL
SELECT CONCAT(UPPER(name), ', ', UPPER(country)) AS location
FROM cities;

-- location
-- TOKYO, JAPAN
-- DELHI, INDIA
-- SHANGHAI, CHINA
-- SAO PAULO, BRAZIL
```

<!-- TOC --><a name="filtering-records-with-where"></a>
## Filtering records with WHERE

Use the `WHERE` clause and its criteria/condition inside a `SELECT` statement.

:warning: **It is important to know the order of execution of a WHERE clause.**
10. First, the `FROM` clause considers the whole table
11. Next, the `WHERE` clause is applied on the above result 
12. Lastly, the `SELECT` operation selects the columns on the above result

Hence, `FROM` to `WHERE` to `SELECT` is how we can conceptualise the order of execution! 

Example:
```SQL
SELECT name, area
FROM cities WHERE area > 4000;

-- name	area
-- Tokyo	8223
-- Delhi	8223
-- Shanghai	4015
```

Here is the flow of execution:
```SQL
-- 1. FROM is applied
-- name	country	population	area
-- Tokyo	Japan	38505000	8223
-- Delhi	India	28125000	8223
-- Shanghai	China	22125000	4015
-- Sao Paulo	Brazil	20935000	3043

-- 2. WHERE is applied
-- name	country	population	area
-- Tokyo	Japan	38505000	8223
-- Delhi	India	28125000	8223
-- Shanghai	China	22125000	4015

-- 3. SELECT is applied
-- name	area
-- Tokyo	8223
-- Delhi	8223
-- Shanghai	4015
```

<!-- TOC --><a name="comparison-operators-within-a-where-clause"></a>
### Comparison operators within a WHERE clause

We can add conditions on columns in a WHERE clause. The list of comparison operators within a `WHERE` clause are:
1. `=` 
2. `>`
3. `<`
4. `>=`
5. `<=`
6. `<>` or `!=` to check for inequality i.e are values not equal?
7. `IN` to check if a value is in a given list (Also use parentheses `()` for the list)
8. `NOT IN` to check if a value is not in a given list (Also use parentheses `()` for the list)
9. `BETWEEN` to check if a value falls in between two values (Also use the `AND` keyword)

Examples:
```SQL
SELECT name, area
FROM cities
WHERE area = 4015

-- name	area
-- Shanghai	4015
``` 
```sql
SELECT name, area
FROM cities
WHERE area BETWEEN 3000 AND 4000;

-- name	area
-- Sao Paulo	3043
```
```SQL
SELECT name, area
FROM cities
WHERE name IN ('Delhi', 'Tokyo');

-- name	area
-- Tokyo	8223
-- Delhi	8223
```
```SQL
SELECT name, area
FROM cities
WHERE AREA NOT IN (8223, 3043);

-- name	area
-- Shanghai	4015
```

<!-- TOC --><a name="compound-where-clauses"></a>
### Compound WHERE clauses

We can use the `AND` or `OR` operators to combine conditionals to form complex criteria for the `SELECT` clause.

Examples:
```SQL
SELECT
  name,
  area
FROM
  cities
WHERE
  area NOT IN (8223, 3043)
  OR name = 'Delhi';

-- name	area
-- Delhi	8223
-- Shanghai	4015
```
```SQL
SELECT
  name,
  area
FROM
  cities
WHERE
  area > 3000
  AND country = 'Japan';

-- name	area
-- Tokyo	8223
```

<!-- TOC --><a name="calculations-in-where-clauses"></a>
### Calculations in WHERE clauses

`WHERE` clause conditions need not only work on columns. It can also work on calculations on columns. Ex: It can work on `population / area`.

**Note**: When we use calculations in our WHERE clause, the mathematical operations are performed first (Ex: `/`, `+`, `*`) and only then is the comparison operator applied! :warning:

Example:
```SQL
SELECT
  name,
  population,
  area
FROM
  cities
WHERE
  population / area > 5000

-- name	population	area
-- Shanghai	22125000	4015
-- Sao Paulo	20935000	3043
```

<!-- TOC --><a name="updating-rows-using-update"></a>
## Updating rows using UPDATE

Use the `UPDATE` statement. The syntax is usually `UPDATE...SET...WHERE`.
You update a table, set its column value, only if that row's column matches a where clause.

```SQL
UPDATE cities
SET population = 39505000
WHERE name = 'Tokyo';

-- 1 row updated
```
```SQL
SELECT * FROM cities;
-- name	country	population	area
-- Delhi	India	28125000	8223
-- Shanghai	China	22125000	4015
-- Sao Paulo	Brazil	20935000	3043
-- Tokyo	Japan	39505000	8223
```

**Note**: If you intend to update only one row then make sure your `WHERE` clause does an *exact* match.

<!-- TOC --><a name="deleting-rows-using-delete-from"></a>
## Deleting rows using DELETE FROM

Use the `DELETE FROM` statement. The syntax is usually `DELETE FROM...WHERE`. 
You choose a table to delete from and specify the where clause which picks the rows to delete accordingly.

```SQL
DELETE FROM cities
WHERE name = 'Tokyo';

-- 1 row deleted
```
```SQL
SELECT * FROM cities;

-- name	country	population	area
-- Delhi	India	28125000	8223
-- Shanghai	China	22125000	4015
-- Sao Paulo	Brazil	20935000	3043
```

**Note**: If you intend to delete only one row then make sure your `WHERE` clause does an *exact* match.

<!-- TOC --><a name="very-basics-of-designing-a-database"></a>
## Very basics of designing a database 

Starting approach:
1. Identify the ***features*** of the application you want to build
	-  Google for ideas for the system you are building. It is likely that it has been built before
2. Create a ***separate table*** for each of these features. Treat them as ***resources***
3.  Identify the ***relations*** or ***ownership*** between the resources

Example:
- Instagram (photo sharing app) has users' profiles, photos, comments & likes on photos. 
- Users, Photos, Comments, & Likes can be the separate tables
- Users can like & comment on photos while following and being followed by other users (relationships).

<!-- TOC --><a name="types-of-relationships"></a>
### Types of relationships

There are 4 types of relationships between records of different tables (i.e resources)

<!-- TOC --><a name="one-to-many"></a>
#### One to Many

One record in a table has relationships with multiple records of another table.

Examples of this kind of a relationship: 
- *"A user has many photos"* (1 user mapped to N photos)
- *"A photo can have many comments"* (1 photo mapped to N comments)
- *"A company has many employees"*

<!-- TOC --><a name="many-to-one"></a>
#### Many to One

This is *actually the same as the One to Many relationship* but the ***perspective is shifted to the other resource***.

Examples of this kind of a relationship: 
- *"A photo has one user"* (N photos are mapped to 1 user)
- *"A comment has one photo"* (N comments are mapped to 1 photo)
- *"An employee works for one company"*

<!-- TOC --><a name="one-to-one"></a>
#### One to One

In this type of relationship, a record in a table has a relationship with *exactly one record* in another table

Examples of this kind of a relationship: 
- "A ship has only one captain and vice versa"
- "A company has only one CEO and vice versa"
- "A person has only one driver's license and vice versa"

<!-- TOC --><a name="many-to-many"></a>
#### Many to Many

In this type of relationship, many records in a table may have relationships with many records of another table

Examples of this kind of a relationship: 
- "Students attend many classes and a class has many students"
- "Engineers work on many tasks and a task may have many engineers working on it"
- "A movie casts many actors and each actor can act in many movies"

<!-- TOC --><a name="primary-and-foreign-keys"></a>
## Primary and foreign keys

**Primary key:** 
- The goal of this key is to *uniquely identify a row in a table*
- If we set a column (or a set of columns) as the primary key (**PK**) then that column *has to be unique for each row* in the entire table. Ex: First name cannot be a primary key of a user table
- Usually named as `id` and is generally an ordered set of *numbers* or *UUIDs* (Random UUIDs are also possible but are bad for performance if they don't have an order)

**Foreign key:**
- The goal of this key is to *relate the current record (row) with another record placed in another table* (or sometimes, even within the same table)
- We generally set up an extra column for the foreign key (**FK**) (Ex: A foreign key column in Photos table that references the user id (primary key) of the Users table)
- Usually, it is named as `xyz_id` where `xyz` references the name of another table's records
- It is exactly equal to the primary key of a record of the table it references
- **Note**: Whenever we talk about 1-1, 1-Many, Many-1, or Many-Many relationships, we always realize it using *foreign keys*!
- The foreign key can change if the relationship changes

**Important**
In One to Many / Many to One relationships, *the "many" side receives the foreign key* (The side the "many" belong to do not have a foreign key). Examples:
- A user has many photos: The Photos table will have a column (FK) referencing the user it belongs to
- A photo has many comments: The Comments table will have a column (FK) referencing the photo it belongs to

Example:
```
Comments:
id (PK) | text | user_id (FK) | photo_id (FK)

Photos:
id (PK) | url | user_id (FK)

Users:
id (PK) | username | email
```
```
Users
| id (INT, PK) | username | email            |
| ------------ | -------- | ---------------- |
| 1            | alice    | alice@mail.com   |
| 2            | bob      | bob@mail.com     |
| 3            | charlie  | charlie@mail.com |

| id (INT, PK) | url      | user_id (INT, FK→Users.id) |
| ------------ | -------- | -------------------------- |
| 1            | img1.jpg | 1                          |
| 2            | img2.jpg | 2                          |
| 3            | img3.jpg | 1                          |

| id (INT, PK) | text            | user_id (INT, FK→Users.id) | photo_id (INT, FK→Photos.id) |
| ------------ | --------------- | -------------------------- | ---------------------------- |
| 1            | Nice shot!      | 2                          | 1                            |
| 2            | Love this cafe! | 3                          | 3                            |
| 3            | Inspiring view  | 1                          | 2                            |
| 4            | Great place!    | 2                          | 3                            |
```
 
 **Auto-generated IDs**: 
- Instead of manually entering `id`s for primary keys, databases allow us to generate an ordered set of values for it automatically every time we insert a row
- In PostgreSQL, we have to use the keyword `SERIAL` during table creation to make a column *auto-incremented*
- ***We can exclude this column during insertion of records!*** as the value will be automatically populated
- **Note**: We should *not* use `SERIAL` for foreign keys since that value may vary on the relationship (Or, it is generated in another table and we simply reference it!)

<!-- TOC --><a name="creating-primary-keys"></a>
### Creating primary keys

**Primary key in PostgreSQL**: Use the **`PRIMARY KEY`** keyword for a column during table creation

Example:
```SQL
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50)
);

INSERT INTO users (username)
VALUES
	('monahan93'),
	('pferrer'),
	('si93onis'),
	('99stroman');
```

**Note**: In the above example, we excluded the SERIAL `id` primary key value during the insertion operations since it is auto-incremented

```SQL
SELECT * FROM users;

-- id	username
-- 1	monahan93
-- 2	pferrer
-- 3	si93onis
-- 4	99stroman
```

<!-- TOC --><a name="creating-foreign-keys"></a>
### Creating foreign keys

Foreign keys are created using the **`REFERENCES`** keyword during table creation. Its argument is the table name with the column it references within parenthese (`()`). Example: `... REFERENCES <table>(<column>)`

Example:
```SQL
CREATE TABLE photos (
  id SERIAL PRIMARY KEY,
  url VARCHAR(200),
  user_id INTEGER REFERENCES users(id)
);

INSERT INTO photos (url, user_id)
VALUES
	('http://one.jpg', 4);
```
```SQL
SELECT * FROM photos;

-- id	url	         user_id
-- 1	http://one.jpg	4
```

<!-- TOC --><a name="queries-on-associated-data"></a>
### Queries on associated data

We can write queries (`SELECT`) statements using foreign keys to fetch associated data.
1. We can use the `WHERE` clause
2. We can also perform `JOIN`s (Explained later)

```SQL
SELECT * FROM photos
WHERE user_id = 4;

-- id	url	       user_id
-- 1	http://one.jpg	4
-- 7	http://256.jpg	4
```
```SQL
SELECT * FROM photos
JOIN users ON photos.user_id = users.id;

-- id	url	user_id	id	username
-- 1	http://36.jpg	1	1	monahan93
-- 1	http://25.jpg	1	1	monahan93
-- 1	http://two.jpg	1	1	monahan93
-- 2	http://754.jpg	2	2	pferrer
-- 3	http://35.jpg	3	3	si93onis
-- 4	http://256.jpg	4	4	99stroman
-- 4	http://one.jpg	4	4	99stroman
```

<!-- TOC --><a name="constraints-on-foreign-keys-during-insertions"></a>
### Constraints on foreign keys during INSERTIONS

1. Trying to insert a foreign key for a value that does not exist (Ex: photo for a user with id 1913 who does not exist in the user's table) will throw an error
2. Trying to insert `NULL` as a foreign key when you don't know the associated value is okay! Works fine in PostgreSQL

```SQL
INSERT INTO photos (url, user_id)
VALUES ('http://99.jpg', 1913);

-- insert or update on table "photos" violates foreign key constraint "photos_user_id_fkey"
```
```SQL
INSERT INTO photos (url, user_id)
VALUES ('http://99.jpg', NULL);

-- INSERT successful!
-- 1 row(s) inserted
```
```SQL
SELECT * FROM photos;

-- id	url	user_id
-- 1	http://one.jpg	4
-- 2	http://two.jpg	1
-- 3	http://25.jpg	1
-- 4	http://36.jpg	1
-- 5	http://754.jpg	2
-- 6	http://35.jpg	3
-- 7	http://256.jpg	4
-- 9	http://99.jpg	null
```

<!-- TOC --><a name="constraints-on-foreign-keys-during-deletions"></a>
### Constraints on foreign keys during DELETIONS

What should happen to the rows containing foreign keys when the referenced row in another table is deleted? Ex: If a user with id 1 is deleted from Users, what should happen to the rows in Photos that have FK user_id = 1? Should they be deleted or should the pointer be a dangling one? Policies are set in PostgreSQL that define the expected behaviour.

**Policies**: (Set during `CREATE TABLE` as an option to the Foreign Key (FK) column)
1. `ON DELETE RESTRICT`: This is the default policy for Foreign Keys (FKs). The delete operation will throw an error
2. `ON DELETE NO ACTION`: Similar to above. The delete operation will throw an error
3. `ON DELETE CASCADE`: Deletes all the associated rows in tables with the matching foreign key when the referenced row is deleted (Useful if you want to remove all associated data on delete). Ex: Deleting the discussion forum data i.e replies for a deleted post
4. `ON DELETE SET NULL`: Sets the Foreign Key field to `NULL` when the referenced row is deleted (Useful if you want to continue maintaining a resource created a user who is now removed). Ex: Continue displaying a photo or article of a user who deleted their account
5. `ON DELETE SET DEFAULT`: Sets a pre-defined default value in the Foreign Key field when the referenced row is deleted 
	- **Note**: We need to specify `DEFAULT <value>` inline but the  `REFERENCES` syntax is written separately as a `CONSTRAINT`)
	- **Note**: The default value must be a valid referenced field value i.e it must exist on the referenced table (Ex: If row from user with `user_id = 1` is deleted but the default specified for foreign key referencing it in photos is `2` then PostgreSQL attempts to set `2` as the default but will throw an error if `2` is not an existing, i.e valid, reference)

```SQL
CREATE TABLE photos (
id SERIAL PRIMARY KEY,
url VARCHAR(200),
user_id INTEGER REFERENCES users(id)
);
 
INSERT INTO photos (url, user_id)
VALUES
('http:/one.jpg', 4),
('http:/two.jpg', 1),
('http:/25.jpg', 1),
('http:/36.jpg', 1),
('http:/754.jpg', 2),
('http:/35.jpg', 3),
('http:/256.jpg', 4);

DELETE FROM users
WHERE ID = 1;

-- update or delete on table "users" violates foreign key constraint "photos_user_id_fkey" on table "photos"
```
```SQL
CREATE TABLE photos (
id SERIAL PRIMARY KEY,
url VARCHAR(200),
user_id INTEGER REFERENCES users(id) ON DELETE NO ACTION
);
 
INSERT INTO photos (url, user_id)
VALUES
('http:/one.jpg', 4),
('http:/two.jpg', 1),
('http:/25.jpg', 1),
('http:/36.jpg', 1),
('http:/754.jpg', 2),
('http:/35.jpg', 3),
('http:/256.jpg', 4);

DELETE FROM users
WHERE ID = 1;

-- update or delete on table "users" violates foreign key constraint "photos_user_id_fkey" on table "photos"
```
```SQL
CREATE TABLE photos (
id SERIAL PRIMARY KEY,
url VARCHAR(200),
user_id INTEGER REFERENCES users(id) ON DELETE CASCADE
);
 
INSERT INTO photos (url, user_id)
VALUES
('http:/one.jpg', 4),
('http:/two.jpg', 1),
('http:/25.jpg', 1),
('http:/36.jpg', 1),
('http:/754.jpg', 2),
('http:/35.jpg', 3),
('http:/256.jpg', 4);

DELETE FROM users
WHERE ID = 1;

SELECT * FROM photos;

-- id	url	user_id
-- 1	http:/one.jpg	4
-- 5	http:/754.jpg	2
-- 6	http:/35.jpg	3
-- 7	http:/256.jpg	4
```
```SQL
CREATE TABLE photos (
    id SERIAL PRIMARY KEY,
    url VARCHAR(200),
    user_id INTEGER DEFAULT 2,
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET DEFAULT
);

INSERT INTO photos (url, user_id)
VALUES
('http:/one.jpg', 4),
('http:/754.jpg', 2),
('http:/35.jpg', 3),
('http:/256.jpg', 4);

DELETE FROM users
WHERE ID = 4;

SELECT * FROM photos;

-- id	url	user_id
-- 2	http:/754.jpg	2
-- 3	http:/35.jpg	3
-- 1	http:/one.jpg	2
-- 4	http:/256.jpg	2
```

<!-- TOC --><a name="joins"></a>
## Joins

**Joins**:
- Produces values by merging together rows from different related tables
- Use a join *most times* that you are asked to find data that involves multiple values
- Use the `JOIN <reference-table> ON <criteria>` syntax. Use the dot (`.`) notation inside the criteria to refer to columns from different tables (Ex: `users.id`)

How join conceptually works at a basic level:
1. Fetches all the contents, i.e rows, of the `FROM` table (left/source table)
2. Reviews the criteria of the `JOIN ON` table (right/target table)
3. Adds the rows of the right/target table to the left/source table where the criteria matches (1 row from left/source can be mapped to multiple rows of the right/target table if multiple rows match the criteria)
4. Selects the columns requested and discard the others. Presents the result

**Note**: Use the following data for the join examples that follow later in this section. Data examples:
```SQL
SELECT * FROM users;

-- id	username
-- 1	Reyna.Marvin
-- 2	Micah.Cremin
-- 3	Alfredo66
-- 4	Gerard_Mitchell42
-- 5	Frederique_Donnelly
```
```SQL
SELECT * FROM photos;

-- id	url					user_id
-- 1	https://santina.net		3
-- 2	https://alayna.net		5
-- 3	https://kailyn.name		3
-- 4	http://marjolaine.name	1
-- 5	http://chet.net			5
-- 6	http://jerrold.org		2
-- 7	https://meredith.net	4
-- 8	http://isaias.net		4
-- 9	http://dayne.com		4
-- 10	http://colten.net		2
-- 11	https://adelbert.biz	5
-- 12	http://kolby.org		1
-- 13	https://deon.biz		2
-- 14	https://marina.com		5
-- 15	http://johnson.info		1
-- 16	https://linda.info		2
-- 17	https://tyrique.info	4
-- 18	http://buddy.info		5
-- 19	https://elinore.name	2
-- 20	http://sasha.com		3
```

```SQL
SELECT * FROM comments;

-- id	contents	user_id	photo_id
-- 1	Quo velit iusto ducimus quos a incidunt nesciunt facilis.	2	4
-- 2	Non est totam.	5	5
-- 3	Fuga et iste beatae.	3	3
-- 4	Molestias tempore est.	1	5
-- 5	Est voluptatum voluptatem voluptatem est ullam quod quia in.	1	5
-- 6	Aut et similique porro ullam.	1	3
-- 7	Fugiat cupiditate consequatur sit magni at non ad omnis.	1	2
-- 8	Accusantium illo maiores et sed maiores quod natus.	2	5
-- 9	Perferendis cumque eligendi.	1	2
-- 10	Nihil quo voluptatem placeat.	5	5
-- 11	Rerum dolor sunt sint.	5	2
-- 12	Id corrupti tenetur similique reprehenderit qui sint qui nulla tenetur.	2	1
-- 13	Maiores quo quia.	1	5
-- 14	Culpa perferendis qui perferendis eligendi officia neque ex.	1	4
-- 15	Reprehenderit voluptates rerum qui veritatis ut.	1	1
-- 16	Aut ipsum porro deserunt maiores sit.	5	3
-- 17	Aut qui eum eos soluta pariatur.	1	1
-- 18	Praesentium tempora rerum necessitatibus aut.	4	3
-- 19	Magni error voluptas veniam ipsum enim.	4	2
-- 20	Et maiores libero quod aliquam sit voluptas.	2	3
-- 21	Eius ab occaecati quae eos aut enim rem.	5	4
-- 22	Et sit occaecati.	4	3
-- 23	Illum omnis et excepturi totam eum omnis.	1	5
-- 24	Nemo nihil rerum alias vel.	5	1
-- 25	Voluptas ab eius.	5	1
-- 26	Dolor soluta quisquam voluptatibus delectus.	3	5
-- 27	Consequatur neque beatae.	4	5
-- 28	Aliquid vel voluptatem.	4	5
-- 29	Maiores nulla ea non autem.	4	5
-- 30	Enim doloremque delectus.	1	4
-- 31	Facere vel assumenda.	2	5
-- 32	Fugiat dignissimos dolorum iusto fugit voluptas et.	2	1
-- 33	Sed cumque in et.	1	3
-- 34	Doloribus temporibus hic eveniet temporibus corrupti et voluptatem et sint.	5	4
-- 35	Quia dolorem officia explicabo quae.	3	1
-- 36	Ullam ad laborum totam veniam.	1	2
-- 37	Et rerum voluptas et corporis rem in hic.	2	3
-- 38	Tempora quas facere.	3	1
-- 39	Rem autem corporis earum necessitatibus dolores explicabo iste quo.	5	5
-- 40	Animi aperiam repellendus in aut eum consequatur quos.	1	2
-- 41	Enim esse magni.	4	3
-- 42	Saepe cumque qui pariatur.	4	4
-- 43	Sit dolorem ipsam nisi.	4	1
-- 44	Dolorem veniam nisi quidem.	2	5
-- 45	Porro illum perferendis nemo libero voluptatibus vel.	3	3
-- 46	Dicta enim rerum culpa a quo molestiae nam repudiandae at.	2	4
-- 47	Consequatur magnam autem voluptas deserunt.	5	1
-- 48	Incidunt cum delectus sunt tenetur et.	4	3
-- 49	Non vel eveniet sed molestiae tempora.	2	1
-- 50	Ad placeat repellat et veniam ea asperiores.	5	1
-- 51	Eum aut magni sint.	3	1
-- 52	Aperiam voluptates quis velit explicabo ipsam vero eum.	1	3
-- 53	Error nesciunt blanditiis quae quis et tempora velit repellat sint.	2	4
-- 54	Blanditiis saepe dolorem enim eos sed ea.	1	2
-- 55	Ab veritatis est.	2	2
-- 56	Vitae voluptatem voluptates vel nam.	3	1
-- 57	Neque aspernatur est non ad vitae nisi ut nobis enim.	4	3
-- 58	Debitis ut amet.	4	2
-- 59	Pariatur beatae nihil cum molestiae provident vel.	4	4
-- 60	Aperiam sunt aliquam illum impedit.	1	4
-- 61	Aut laudantium necessitatibus harum eaque.	5	3
-- 62	Debitis voluptatum nesciunt quisquam voluptatibus fugiat nostrum sed dolore quasi.	3	2
-- 63	Praesentium velit voluptatem distinctio ut voluptatum at aut.	2	2
-- 64	Voluptates nihil voluptatum quia maiores dolorum molestias occaecati.	1	4
-- 65	Quisquam modi labore.	3	2
-- 66	Fugit quia perferendis magni doloremque dicta officia dignissimos ut necessitatibus.	1	4
-- 67	Tempora ipsam aut placeat ducimus ut exercitationem quis provident.	5	3
-- 68	Expedita ducimus cum quibusdam.	5	1
-- 69	In voluptates doloribus aut ut libero possimus adipisci iste.	3	2
-- 70	Sit qui est sed accusantium quidem id voluptatum id.	1	5
-- 71	Libero eius quo consequatur laudantium reiciendis reiciendis aliquid nemo.	1	2
-- 72	Officia qui reprehenderit ut accusamus qui voluptatum at.	2	2
-- 73	Ad similique quo.	4	1
-- 74	Commodi culpa aut nobis qui illum deserunt reiciendis.	2	3
-- 75	Tenetur quam aut rerum doloribus est ipsa autem.	4	2
-- 76	Est accusamus aut nisi sit aut id non natus assumenda.	2	4
-- 77	Et sit et vel quos recusandae quo qui.	1	3
-- 78	Velit nihil voluptatem et sed.	4	4
-- 79	Sunt vitae expedita fugiat occaecati.	1	3
-- 80	Consequatur quod et ipsam in dolorem.	4	2
-- 81	Magnam voluptatum molestias vitae voluptatibus beatae nostrum sunt.	3	5
-- 82	Alias praesentium ut voluptatem alias praesentium tempora voluptas debitis.	2	5
-- 83	Ipsam cumque aut consectetur mollitia vel quod voluptates provident suscipit.	3	5
-- 84	Ad dignissimos quia aut commodi vel ut nisi.	3	3
-- 85	Fugit ut architecto doloremque neque quis.	4	5
-- 86	Repudiandae et voluptas aut in excepturi.	5	3
-- 87	Aperiam voluptatem animi.	5	1
-- 88	Et mollitia vel soluta fugiat.	4	1
-- 89	Ut nemo voluptas voluptatem voluptas.	5	2
-- 90	At aut quidem voluptatibus rem.	5	1
-- 91	Temporibus voluptates iure fuga alias minus eius.	2	3
-- 92	Non autem laboriosam consectetur officiis aut excepturi nobis commodi.	4	3
-- 93	Esse voluptatem sed deserunt ipsum eaque maxime rerum qui.	5	5
-- 94	Debitis ipsam ut pariatur molestiae ut qui aut reiciendis.	4	4
-- 95	Illo atque nihil et quod consequatur neque pariatur delectus.	3	3
-- 96	Qui et hic accusantium odio quis necessitatibus et magni.	4	2
-- 97	Debitis repellendus inventore omnis est facere aliquam.	3	3
-- 98	Occaecati eos possimus deleniti itaque aliquam accusamus.	3	4
-- 99	Molestiae officia architecto eius nesciunt.	5	4
-- 100	Minima dolorem reiciendis excepturi culpa sapiente eos deserunt ut.	3	3
```

**Join example**:
```SQL
SELECT contents, username
FROM comments
JOIN users ON comments.user_id = users.id;

-- contents					username
-- Quo...					Micah.Cremin
-- Non...					Frederique_Donnelly
-- Fuga...					Alfredo66
-- Mole...					Reyna.Marvin
... LIST GOES ON
```
How the join populates data is explained in the section on the types of joins below.

**Note**:
1. The ***order of the tables*** in JOIN matters. The results can vary (or stay the same) depending on the *type of join*
```SQL
-- THIS:
SELECT contents, username
FROM comments
JOIN users ON comments.user_id = users.id;

-- !! NEED NOT BE THE SAME AS THIS !!:
SELECT contents, username
FROM users
JOIN comments ON comments.user_id = users.id;
```
2. We must ***give context if column names collide***. Ex: If both joined users & photos tables have a column named `id` that we want to use, we need to reference them as `users.id` and `photos.id` respectively
```SQL
SELECT id			-- Should be `users.id` or `photos.id` 
					-- (Ex: `SELECT users.id AS userid)
FROM comments
JOIN users ON comments.user_id = users.id;

-- column reference "id" is ambiguous
```
3. ***Tables can be renamed using the `AS` keyword***
```SQL
SELECT p.id, username
FROM photos AS p
JOIN users ON p.user_id = users.id;
```

<!-- TOC --><a name="four-types-of-joins"></a>
### Four types of joins

There are 4 types of joins:
1. `INNER JOIN`: Returns only matching rows from both tables
	- *Real application example: When you want to fetch all photos posted by an existing user only*
2. `LEFT JOIN`: Returns all rows from A (left table), and matching rows from B (right table)
	- *Real application example: When you want to fetch all photos even if poster details are unavailable but with attributes of the user listed if one has indeed posted them*
3. `RIGHT JOIN`: Returns all rows from B (right table), and matching rows from A (left table)
	- *Real application example: When you want to fetch all photos posted by an existing user only and also even the details of users for those who have not posted any photo at all*
4. `FULL OUTER JOIN`: Returns all rows from both A (left table) and B (right table)
	- *Real application example: When you want to fetch all photos even if poster details are unavailable and also all the users even if they have not posted any photo* 

**Note**: 
- Right table (target) is the table on which you `JOIN` ... `ON`. Left (source) is the one specified in the `FROM`. That is why ORDER MATTERS in a join! (left vs right table)
- The **default** join is `INNER JOIN`. We can either write `JOIN` or `INNER JOIN` in the syntax for it!

```
SQL JOINS — Text Venn Diagram Representation
--------------------------------------------
Table A ●        Table B ○

1️⃣ INNER JOIN
   Returns only matching rows from both tables i.e overlapping rows only.
  unused A       (A ∩ B)       unused B
     ●●            ●●●○○          ○○
     ●●           ●●●●●○○         ○○
     ●●            ●●●○○          ○○
          (Only overlap of A & B)
       
2️⃣ LEFT JOIN
   Returns all rows from A, and matching rows from B.
  unused A       (A ∩ B)       unused B
                 ●●●●●○○          ○○
                ●●●●●●●○○         ○○
                 ●●●●●○○          ○○
          (All of A + overlap)

3️⃣ RIGHT JOIN
   Returns all rows from B, and matching rows from A.
  unused A         (A ∩ B)       unused B
     ●●            ●●●○○○○          
     ●●           ●●●●●○○○○         
     ●●            ●●●○○○○          
           (All of B + overlap)

4️⃣ FULL OUTER JOIN
   Returns all rows from both A and B.
  unused A       (A ∩ B)       unused B
                 ●●●●●○○○○          
                ●●●●●●●○○○○         
                 ●●●●●○○○○          
           (Everything from A and B)

LEGEND:
● = Rows from Table A
○ = Rows from Table B
Overlap (●○) = Matching rows between A and B
```

Logical steps explained: 
The order of execution is similar to a `SELECT` statement i.e `FROM` followed by a type of `JOIN` and then finally `SELECT`.
1. `INNER JOIN`:
	- First, take the whole left/source table in `FROM` (Similar to logic of a regular `SELECT`)
	- Find matching rows of right/target table (`JOIN ON`) and create combined rows in the result (*Note*: If there are N matches in target for one row in source then there are N rows in the result for that one source row)
	- Discard any row in left/source table with no matching rows in right/target table
2. `LEFT JOIN`
	- First, take the whole left/source table in `FROM` (Similar to logic of a regular `SELECT`)
	- Find matching rows of right/target table (`JOIN ON`) and create combined rows in the result (*Note*: If there are N matches in target for one row in source then there are N rows in the result for that one source row)
	- ***For any row in left/source table without a matching row in right/target table, add the row in result with `NULL` values for the right table columns***
3. `RIGHT JOIN`
	- First, take the whole left/source table in `FROM` (Similar to logic of a regular `SELECT`)
	- Find matching rows of right/target table (`JOIN ON`) and create combined rows in the result (*Note*: If there are N matches in target for one row in source then there are N rows in the result for that one source row)
	- ***For any row in right/target table without a matching row in left/source table, add the row in result with `NULL` values for the left table columns***
4. `FULL OUTER JOIN`
	- First, take the whole left/source table in `FROM` (Similar to logic of a regular `SELECT`)
	- Find matching rows of right/target table (`JOIN ON`) and create combined rows in the result (*Note*: If there are N matches in target for one row in source then there are N rows in the result for that one source row)
	- ***For any row in left/source table without a matching row in right/target table, add the row in result with `NULL` values for the right table columns***
	- ***For any row in right/target table without a matching row in left/source table, add the row in result with `NULL` values for the left table columns***

**Examples**: (Refer to the `users` and `photos` data above)
```SQL
SELECT url, username
FROM photos
JOIN users ON photos.user_id = users.id;
-- Can also use `INNER JOIN` (Same as `JOIN`)

-- url						username
-- http://johnson.info		Reyna.Marvin
-- http://kolby.org			Reyna.Marvin
-- http://marjolaine.name	Reyna.Marvin
-- https://elinore.name		Micah.Cremin
-- https://linda.info		Micah.Cremin
-- https://deon.biz			Micah.Cremin
-- http://colten.net		Micah.Cremin
-- http://jerrold.org		Micah.Cremin
-- http://sasha.com			Alfredo66
-- https://kailyn.name		Alfredo66
-- https://santina.net		Alfredo66
-- https://tyrique.info		Gerard_Mitchell42
-- http://dayne.com			Gerard_Mitchell42
-- http://isaias.net		Gerard_Mitchell42
-- https://meredith.net		Gerard_Mitchell42
-- http://buddy.info		Frederique_Donnelly
-- https://marina.com		Frederique_Donnelly
-- https://adelbert.biz		Frederique_Donnelly
-- http://chet.net			Frederique_Donnelly
-- https://alayna.net		Frederique_Donnelly
```

```SQL
INSERT INTO photos (url, user_id)
VALUES ('https://monroe.jpg', NULL);

SELECT url, username
FROM photos
LEFT JOIN users ON photos.user_id = users.id;

-- url						username
-- http://johnson.info		Reyna.Marvin
-- http://kolby.org			Reyna.Marvin
-- http://marjolaine.name	Reyna.Marvin
-- https://elinore.name		Micah.Cremin
-- https://linda.info		Micah.Cremin
-- https://deon.biz			Micah.Cremin
-- http://colten.net		Micah.Cremin
-- http://jerrold.org		Micah.Cremin
-- http://sasha.com			Alfredo66
-- https://kailyn.name		Alfredo66
-- https://santina.net		Alfredo66
-- https://tyrique.info		Gerard_Mitchell42
-- http://dayne.com			Gerard_Mitchell42
-- http://isaias.net		Gerard_Mitchell42
-- https://meredith.net		Gerard_Mitchell42
-- http://buddy.info		Frederique_Donnelly
-- https://marina.com		Frederique_Donnelly
-- https://adelbert.biz		Frederique_Donnelly
-- http://chet.net			Frederique_Donnelly
-- https://alayna.net		Frederique_Donnelly
-- https://monroe.jpg		null (Note: Null added for unmatched right table column(s))
```

```SQL
INSERT INTO users (username)
VALUES ('David');

SELECT url, username
FROM photos
RIGHT JOIN users ON photos.user_id = users.id;

-- url						username
-- http://johnson.info		Reyna.Marvin
-- http://kolby.org			Reyna.Marvin
-- http://marjolaine.name	Reyna.Marvin
-- https://elinore.name		Micah.Cremin
-- https://linda.info		Micah.Cremin
-- https://deon.biz			Micah.Cremin
-- http://colten.net		Micah.Cremin
-- http://jerrold.org		Micah.Cremin
-- http://sasha.com			Alfredo66
-- https://kailyn.name		Alfredo66
-- https://santina.net		Alfredo66
-- https://tyrique.info		Gerard_Mitchell42
-- http://dayne.com			Gerard_Mitchell42
-- http://isaias.net		Gerard_Mitchell42
-- https://meredith.net		Gerard_Mitchell42
-- http://buddy.info		Frederique_Donnelly
-- https://marina.com		Frederique_Donnelly
-- https://adelbert.biz		Frederique_Donnelly
-- http://chet.net			Frederique_Donnelly
-- https://alayna.net		Frederique_Donnelly
-- null						David (Note: Null added for unmatched left table column(s))
```

```SQL
-- INSERT INTO photos (url, user_id)
-- VALUES ('https://monroe.jpg', NULL);

-- INSERT INTO users (username)
-- VALUES ('David');

SELECT url, username
FROM photos
FULL OUTER JOIN users ON photos.user_id = users.id;

-- url						username
-- http://johnson.info		Reyna.Marvin
-- http://kolby.org			Reyna.Marvin
-- http://marjolaine.name	Reyna.Marvin
-- https://elinore.name		Micah.Cremin
-- https://linda.info		Micah.Cremin
-- https://deon.biz			Micah.Cremin
-- http://colten.net		Micah.Cremin
-- http://jerrold.org		Micah.Cremin
-- http://sasha.com			Alfredo66
-- https://kailyn.name		Alfredo66
-- https://santina.net		Alfredo66
-- https://tyrique.info		Gerard_Mitchell42
-- http://dayne.com			Gerard_Mitchell42
-- http://isaias.net		Gerard_Mitchell42
-- https://meredith.net		Gerard_Mitchell42
-- http://buddy.info		Frederique_Donnelly
-- https://marina.com		Frederique_Donnelly
-- https://adelbert.biz		Frederique_Donnelly
-- http://chet.net			Frederique_Donnelly
-- https://alayna.net		Frederique_Donnelly
-- null						David (Note: Null added for unmatched left table column(s))
-- https://monroe.jpg		null (Note: Null added for unmatched right table column(s))
```

<!-- TOC --><a name="joins-with-where-clauses"></a>
### Joins with WHERE clauses

`WHERE` clauses in `JOIN`s:
1. `WHERE` clause is always listed **AFTER** the `JOIN` statement
2. *Order of execution* is similar to that of a regular SELECT statement: `FROM` followed by a `JOIN` followed by a `WHERE` and finally `SELECT` the columns

Example: Find comments of users who commented on their own photo
```SQL
SELECT comments.user_id, photos.user_id, url, contents 
FROM comments
JOIN photos on comments.photo_id = photos.id
WHERE comments.user_id = photos.user_id;

-- user_id	user_id	url	contents
-- 5	5	http://chet.net	Non est totam.
-- 3	3	https://kailyn.name	Fuga et iste beatae.
-- 5	5	http://chet.net	Nihil quo voluptatem placeat.
-- 5	5	https://alayna.net	Rerum dolor sunt sint.
-- 1	1	http://marjolaine.name	Culpa perferendis qui perferendis eligendi officia neque ex.
-- 1	1	http://marjolaine.name	Enim doloremque delectus.
-- 3	3	https://santina.net	Quia dolorem officia explicabo quae.
-- 3	3	https://santina.net	Tempora quas facere.
-- 5	5	http://chet.net	Rem autem corporis earum necessitatibus dolores explicabo iste quo.
-- 3	3	https://kailyn.name	Porro illum perferendis nemo libero voluptatibus vel.
-- 3	3	https://santina.net	Eum aut magni sint.
-- 3	3	https://santina.net	Vitae voluptatem voluptates vel nam.
-- 1	1	http://marjolaine.name	Aperiam sunt aliquam illum impedit.
-- 1	1	http://marjolaine.name	Voluptates nihil voluptatum quia maiores dolorum molestias occaecati.
-- 1	1	http://marjolaine.name	Fugit quia perferendis magni doloremque dicta officia dignissimos ut necessitatibus.
-- 3	3	https://kailyn.name	Ad dignissimos quia aut commodi vel ut nisi.
-- 5	5	https://alayna.net	Ut nemo voluptas voluptatem voluptas.
-- 5	5	http://chet.net	Esse voluptatem sed deserunt ipsum eaque maxime rerum qui.
-- 3	3	https://kailyn.name	Illo atque nihil et quod consequatur neque pariatur delectus.
-- 3	3	https://kailyn.name	Debitis repellendus inventore omnis est facere aliquam.
-- 3	3	https://kailyn.name	Minima dolorem reiciendis excepturi culpa sapiente eos deserunt ut.
```

<!-- TOC --><a name="three-way-joins"></a>
### Three way joins

To join 3 or more tables, simply add an extra join for each new table you wish to join.
Please note the conceptual order of the execution of the statement:
1. `FROM` (left/source) table is taken as a whole
2. The first `JOIN` type is applied i.e right/target table
3. For the subsequent `JOIN` types, the left/source table is the resultant table compiled so far!
4. Apply `WHERE` clauses (if any) and finally execute the column `SELECT`.

Example: Fetching usernames, comments, and URLs where the user has commented on his/her own photo
```SQL
SELECT username, url, contents 
FROM comments
JOIN photos on comments.photo_id = photos.id
JOIN users on users.id = comments.user_id AND users.id = photos.user_id; -- Condition needs careful thought!

-- username				url				contents
-- Frederique_Donnelly	http://chet.net	Non est totam.
-- Alfredo66			https://kailyn.name	Fuga et iste beatae.
-- Frederique_Donnelly	http://chet.net	Nihil quo voluptatem placeat.
-- Frederique_Donnelly	https://alayna.net	Rerum dolor sunt sint.
-- Reyna.Marvin			http://marjolaine.name	Culpa perferendis qui perferendis eligendi officia neque ex.
-- Reyna.Marvin			http://marjolaine.name	Enim doloremque delectus.
-- Alfredo66			https://santina.net	Quia dolorem officia explicabo quae.
-- Alfredo66			https://santina.net	Tempora quas facere.
-- Frederique_Donnelly	http://chet.net	Rem autem corporis earum necessitatibus dolores explicabo iste quo.
-- Alfredo66			https://kailyn.name	Porro illum perferendis nemo libero voluptatibus vel.
-- Alfredo66			https://santina.net	Eum aut magni sint.
-- Alfredo66			https://santina.net	Vitae voluptatem voluptates vel nam.
-- Reyna.Marvin			http://marjolaine.name	Aperiam sunt aliquam illum impedit.
-- Reyna.Marvin			http://marjolaine.name	Voluptates nihil voluptatum quia maiores dolorum molestias occaecati.
-- Reyna.Marvin			http://marjolaine.name	Fugit quia perferendis magni doloremque dicta officia dignissimos ut necessitatibus.
-- Alfredo66			https://kailyn.name	Ad dignissimos quia aut commodi vel ut nisi.
-- Frederique_Donnelly	https://alayna.net	Ut nemo voluptas voluptatem voluptas.
-- Frederique_Donnelly	http://chet.net	Esse voluptatem sed deserunt ipsum eaque maxime rerum qui.
-- Alfredo66			https://kailyn.name	Illo atque nihil et quod consequatur neque pariatur delectus.
-- Alfredo66			https://kailyn.name	Debitis repellendus inventore omnis est facere aliquam.
-- Alfredo66			https://kailyn.name	Minima dolorem reiciendis excepturi culpa sapiente eos deserunt ut.
```
<!-- TOC --><a name="grouping-and-aggregations"></a>
## Grouping and aggregations

**Grouping**
- Reduces many rows down to *fewer* rows 
- Done by using the `GROUP BY` keyword
- Tip: Visualising the result is the key to using it!

**Aggregations**:
- Looks at many rows and calculates a *single value* i.e reduces many values down to *one*
- Think of words like *average*, *most/max*, *least/min*, *count*, etc. These are aggregations
- Done by *"aggregate functions"*

Imagining how `GROUP BY` works:
- *Groups are created* based on the column(s) values specified in the `GROUP BY`
- All the rows in the query are placed within each of these groups i.e the rows are categorised
- **Important!** You may query i.e `SELECT` on the resultant table **ONLY ON** either 
	- **(1) Any of the GROUP BY columns,** or 
	- **(2) Any column BUT only if you use an *aggregate function* on it**
	- *Why?* It is because there is no single value for a column to display unless it is grouped by that column or if an aggregation on it reduces it to a single value

Examples:
```SQL
SELECT user_id
FROM comments
GROUP BY user_id;

-- user_id
-- 1
-- 3
-- 5
-- 4
-- 2
```

```SQL
SELECT contents
FROM comments
GROUP BY user_id;

-- column "comments.contents" must appear in the GROUP BY clause or be used in an aggregate function

-- Why the ERROR? We are selecting a column not in group by 
-- nor having an aggregate function applied
```

Common aggregate functions list:
1. `COUNT()`: Finds the *number of values* in a group of values
2. `SUM()`: Finds the *sum* of a group of values
3. `AVG()`: Finds the *average* of a group of values 
4. `MIN()`: Finds the *minimum* of a group of values 
5. `MAX()`: Finds the *maximum* of a group of values 

**Note**: ***The ENTIRE table is ONE GROUP if there is NO grouping*** i.e No `GROUP BY`

Example: Find the max comment id value in the entire comments table
```SQL
SELECT MAX(id)
FROM comments;

-- max
-- 100
```

<!-- TOC --><a name="combining-group-by-and-aggregates"></a>
### Combining GROUP BY and aggregates

Order of execution is similar to a regular `SELECT` statement:
1. `FROM`
2. `GROUP BY` (Create that imaginary table with groups)
3. Finally, `SELECT`

Example: Find the number of comments posted by a user 
```SQL
SELECT user_id, COUNT(id) AS number_of_comments
FROM comments
GROUP BY user_id;

-- user_id	number_of_comments
-- 1			23
-- 3			17
-- 5			20
-- 4			22
-- 2			18
```

<!-- TOC --><a name="a-gotcha-with-count"></a>
### A gotcha! with COUNT

When we execute `COUNT` on a column, it ***IGNORES*** `NULL` values. Hence, if we want to include rows with `NULL` values in the count, we should use the wildcard `*` to select the entire row instead

Examples: Note that the photos table has 21 rows but one of them has user_id `NULL`
```SQL
SELECT COUNT(user_id)
FROM photos;

-- count
-- 20
```
```SQL
SELECT COUNT(*)
FROM photos;

-- count
-- 21
```

This gotcha! applies to *groups* as well: 
* While a group is created for a `NULL` column that appears in the `GROUP BY`, the `COUNT` will not include it for that group!

```SQL
SELECT user_id, COUNT(user_id)
FROM photos
GROUP BY user_id;

-- user_id	count
-- null		0
-- 3		3
-- 5		5
-- 4		4
-- 2		5
-- 1		3
```
```SQL
SELECT user_id, COUNT(*)
FROM photos
GROUP BY user_id;

-- user_id  count
-- null 	1
-- 3    	3
-- 5    	5
-- 4    	4
-- 2    	5
-- 1    	3
```

**Note**: It is recommend to use **`COUNT(*)`** usually instead of referencing any particular column

<!-- TOC --><a name="group-by-with-joins"></a>
### GROUP BY with JOINs

Similar to `WHERE`, `GROUP BY` can be used with `JOIN`s:
* `GROUP BY` is placed after the type of `JOIN`
* If a `WHERE` clause is present after the `JOIN`s, the `GROUP BY` is written after it 
* Order of execution: `FROM` to `JOIN` to `WHERE` (if any) to `GROUP BY` and finally, `SELECT`

Example: Find the number of photos posted by each user
```SQL
SELECT username, COUNT(photos.id)
FROM photos
JOIN users ON photos.user_id = users.id
GROUP BY username;

-- username	   				count
-- Reyna.Marvin	   			 3
-- Frederique_Donnelly 	    5
-- Alfredo66	   			 3
-- Micah.Cremin	    		5
-- Gerard_Mitchell42	    4
```

<!-- TOC --><a name="filter-out-groups-by-having"></a>
### Filter out groups by HAVING

`HAVING` filters out groups:
- You can ***never*** have `HAVING` ***without*** a `GROUP BY` 
- Works for both (1) *Filtering groups by simple conditions*, or (2) *Filtering by aggregate conditions*
- Order of execution: `FROM` to `JOIN` (if any) to `WHERE` (if any) to `GROUP BY` to `HAVING`  and finally, `SELECT`

**Note**: `HAVING` usually has an *"aggregate function"* inside of it i.e inside its criteria. However, this need not always be true!

Example: Selecting comments with photo id less than 3, grouping them by photo ids having a count of more than 2, and fetching the number of photos for each photo id group 
```SQL
SELECT photo_id, COUNT(*)
FROM comments
WHERE photo_id < 3
GROUP BY photo_id
HAVING COUNT(*) > 2;

-- photo_id	count
-- 1		19
-- 2		19
```

<!-- TOC --><a name="sorting-rows-using-order-by"></a>
## Sorting rows using ORDER BY

Use `ORDER BY` to sort rows:
- Default order is ascending (i.e `ASC`). You may also explicitly define it
- To order in descending order, use `DESC`
- We can order on number values as well as text (lexicographic)

```SQL
SELECT photo_id, contents
FROM comments
ORDER BY photo_id;

-- photo_id	contents
-- 1	Vitae voluptatem voluptates vel nam.
-- 1	Aperiam voluptatem animi.
-- 1	Sit dolorem ipsam nisi.
-- 1	Eum aut magni sint.
-- 1	Consequatur magnam autem voluptas deserunt.
-- 1	Id corrupti tenetur similique reprehenderit qui sint qui nulla tenetur.
-- 1	Non vel eveniet sed molestiae tempora.
-- 1	Ad placeat repellat et veniam ea asperiores.
-- 1	Reprehenderit voluptates rerum qui veritatis ut.
-- 1	At aut quidem voluptatibus rem.
-- 1	Aut qui eum eos soluta pariatur.
-- 1	Ad similique quo.
-- 1	Nemo nihil rerum alias vel.
-- 1	Voluptas ab eius.
-- 1	Expedita ducimus cum quibusdam.
-- 1	Fugiat dignissimos dolorum iusto fugit voluptas et.
-- 1	Et mollitia vel soluta fugiat.
-- 1	Quia dolorem officia explicabo quae.
-- 1	Tempora quas facere.
-- 2	Qui et hic accusantium odio quis necessitatibus et magni.
-- 2	Animi aperiam repellendus in aut eum consequatur quos.
-- 2	Debitis voluptatum nesciunt quisquam voluptatibus fugiat nostrum sed dolore quasi.
-- 2	Ab veritatis est.
-- 2	Fugiat cupiditate consequatur sit magni at non ad omnis.
-- 2	Blanditiis saepe dolorem enim eos sed ea.
-- 2	Rerum dolor sunt sint.
-- 2	Officia qui reprehenderit ut accusamus qui voluptatum at.
-- 2	Perferendis cumque eligendi.
-- 2	Ullam ad laborum totam veniam.
-- 2	Libero eius quo consequatur laudantium reiciendis reiciendis aliquid nemo.
-- 2	In voluptates doloribus aut ut libero possimus adipisci iste.
-- 2	Consequatur quod et ipsam in dolorem.
-- 2	Tenetur quam aut rerum doloribus est ipsa autem.
-- 2	Debitis ut amet.
-- 2	Quisquam modi labore.
-- 2	Praesentium velit voluptatem distinctio ut voluptatum at aut.
-- 2	Ut nemo voluptas voluptatem voluptas.
-- 2	Magni error voluptas veniam ipsum enim.
-- 3	Minima dolorem reiciendis excepturi culpa sapiente eos deserunt ut.
-- 3	Fuga et iste beatae.
-- 3	Aut et similique porro ullam.
-- 3	Aut ipsum porro deserunt maiores sit.
-- 3	Praesentium tempora rerum necessitatibus aut.
-- 3	Et maiores libero quod aliquam sit voluptas.
-- 3	Et sit occaecati.
-- 3	Sed cumque in et.
-- 3	Et rerum voluptas et corporis rem in hic.
-- 3	Enim esse magni.
-- 3	Porro illum perferendis nemo libero voluptatibus vel.
-- 3	Incidunt cum delectus sunt tenetur et.
-- 3	Aperiam voluptates quis velit explicabo ipsam vero eum.
-- 3	Neque aspernatur est non ad vitae nisi ut nobis enim.
-- 3	Aut laudantium necessitatibus harum eaque.
-- 3	Tempora ipsam aut placeat ducimus ut exercitationem quis provident.
-- 3	Commodi culpa aut nobis qui illum deserunt reiciendis.
-- 3	Et sit et vel quos recusandae quo qui.
-- 3	Sunt vitae expedita fugiat occaecati.
-- 3	Ad dignissimos quia aut commodi vel ut nisi.
-- 3	Repudiandae et voluptas aut in excepturi.
-- 3	Temporibus voluptates iure fuga alias minus eius.
-- 3	Non autem laboriosam consectetur officiis aut excepturi nobis commodi.
-- 3	Illo atque nihil et quod consequatur neque pariatur delectus.
-- 3	Debitis repellendus inventore omnis est facere aliquam.
-- 4	Voluptates nihil voluptatum quia maiores dolorum molestias occaecati.
-- 4	Fugit quia perferendis magni doloremque dicta officia dignissimos ut necessitatibus.
-- 4	Quo velit iusto ducimus quos a incidunt nesciunt facilis.
-- 4	Eius ab occaecati quae eos aut enim rem.
-- 4	Occaecati eos possimus deleniti itaque aliquam accusamus.
-- 4	Culpa perferendis qui perferendis eligendi officia neque ex.
-- 4	Est accusamus aut nisi sit aut id non natus assumenda.
-- 4	Velit nihil voluptatem et sed.
-- 4	Debitis ipsam ut pariatur molestiae ut qui aut reiciendis.
-- 4	Molestiae officia architecto eius nesciunt.
-- 4	Dicta enim rerum culpa a quo molestiae nam repudiandae at.
-- 4	Error nesciunt blanditiis quae quis et tempora velit repellat sint.
-- 4	Saepe cumque qui pariatur.
-- 4	Doloribus temporibus hic eveniet temporibus corrupti et voluptatem et sint.
-- 4	Pariatur beatae nihil cum molestiae provident vel.
-- 4	Aperiam sunt aliquam illum impedit.
-- 4	Enim doloremque delectus.
-- 5	Magnam voluptatum molestias vitae voluptatibus beatae nostrum sunt.
-- 5	Alias praesentium ut voluptatem alias praesentium tempora voluptas debitis.
-- 5	Ipsam cumque aut consectetur mollitia vel quod voluptates provident suscipit.
-- 5	Rem autem corporis earum necessitatibus dolores explicabo iste quo.
-- 5	Maiores nulla ea non autem.
-- 5	Fugit ut architecto doloremque neque quis.
-- 5	Non est totam.
-- 5	Aliquid vel voluptatem.
-- 5	Consequatur neque beatae.
-- 5	Sit qui est sed accusantium quidem id voluptatum id.
-- 5	Dolor soluta quisquam voluptatibus delectus.
-- 5	Illum omnis et excepturi totam eum omnis.
-- 5	Nihil quo voluptatem placeat.
-- 5	Est voluptatum voluptatem voluptatem est ullam quod quia in.
-- 5	Accusantium illo maiores et sed maiores quod natus.
-- 5	Facere vel assumenda.
-- 5	Esse voluptatem sed deserunt ipsum eaque maxime rerum qui.
-- 5	Dolorem veniam nisi quidem.
-- 5	Molestias tempore est.
-- 5	Maiores quo quia.
```

<!-- TOC --><a name="ordering-by-by-multiple-columns"></a>
### Ordering by by multiple columns

Provide comma (`,`) separated column names to sort them based on priority.

Syntax: `ORDER BY column1, column2 <ASC|DESC?>, column3 <ASC|DESC?>, ...`

How it works:
* `column1` is sorted first (1st weight)
* On the above sorted set, we sort again using `column2` for each `column1` value subset (2nd weight)
* On the above sorted set, we sort again using `column3` for each `column2` value subset (3rd weight)
* ... and so on.

Example: Ordering comments by photo ids from lowest to highest and sorting those again based on user id from highest to lowest:
```SQL
SELECT photo_id, user_id, contents
FROM comments
ORDER BY photo_id, user_id DESC;

-- photo_id	user_id	contents
-- 1	5	Expedita ducimus cum quibusdam.
-- 1	5	Nemo nihil rerum alias vel.
-- 1	5	Consequatur magnam autem voluptas deserunt.
-- 1	5	Voluptas ab eius.
-- 1	5	At aut quidem voluptatibus rem.
-- 1	5	Ad placeat repellat et veniam ea asperiores.
-- 1	5	Aperiam voluptatem animi.
-- 1	4	Et mollitia vel soluta fugiat.
-- 1	4	Ad similique quo.
-- 1	4	Sit dolorem ipsam nisi.
-- 1	3	Vitae voluptatem voluptates vel nam.
-- 1	3	Quia dolorem officia explicabo quae.
-- 1	3	Tempora quas facere.
-- 1	3	Eum aut magni sint.
-- 1	2	Fugiat dignissimos dolorum iusto fugit voluptas et.
-- 1	2	Id corrupti tenetur similique reprehenderit qui sint qui nulla tenetur.
-- 1	2	Non vel eveniet sed molestiae tempora.
-- 1	1	Aut qui eum eos soluta pariatur.
-- 1	1	Reprehenderit voluptates rerum qui veritatis ut.
-- 2	5	Ut nemo voluptas voluptatem voluptas.
-- 2	5	Rerum dolor sunt sint.
-- 2	4	Debitis ut amet.
-- 2	4	Magni error voluptas veniam ipsum enim.
-- 2	4	Consequatur quod et ipsam in dolorem.
-- 2	4	Qui et hic accusantium odio quis necessitatibus et magni.
-- 2	4	Tenetur quam aut rerum doloribus est ipsa autem.
-- 2	3	In voluptates doloribus aut ut libero possimus adipisci iste.
-- 2	3	Debitis voluptatum nesciunt quisquam voluptatibus fugiat nostrum sed dolore quasi.
-- 2	3	Quisquam modi labore.
-- 2	2	Officia qui reprehenderit ut accusamus qui voluptatum at.
-- 2	2	Ab veritatis est.
-- 2	2	Praesentium velit voluptatem distinctio ut voluptatum at aut.
-- 2	1	Ullam ad laborum totam veniam.
-- 2	1	Fugiat cupiditate consequatur sit magni at non ad omnis.
-- 2	1	Perferendis cumque eligendi.
-- 2	1	Libero eius quo consequatur laudantium reiciendis reiciendis aliquid nemo.
-- 2	1	Animi aperiam repellendus in aut eum consequatur quos.
-- 2	1	Blanditiis saepe dolorem enim eos sed ea.
-- 3	5	Tempora ipsam aut placeat ducimus ut exercitationem quis provident.
-- 3	5	Aut ipsum porro deserunt maiores sit.
-- 3	5	Repudiandae et voluptas aut in excepturi.
-- 3	5	Aut laudantium necessitatibus harum eaque.
-- 3	4	Praesentium tempora rerum necessitatibus aut.
-- 3	4	Incidunt cum delectus sunt tenetur et.
-- 3	4	Neque aspernatur est non ad vitae nisi ut nobis enim.
-- 3	4	Non autem laboriosam consectetur officiis aut excepturi nobis commodi.
-- 3	4	Enim esse magni.
-- 3	4	Et sit occaecati.
-- 3	3	Minima dolorem reiciendis excepturi culpa sapiente eos deserunt ut.
-- 3	3	Fuga et iste beatae.
-- 3	3	Porro illum perferendis nemo libero voluptatibus vel.
-- 3	3	Ad dignissimos quia aut commodi vel ut nisi.
-- 3	3	Illo atque nihil et quod consequatur neque pariatur delectus.
-- 3	3	Debitis repellendus inventore omnis est facere aliquam.
-- 3	2	Temporibus voluptates iure fuga alias minus eius.
-- 3	2	Et rerum voluptas et corporis rem in hic.
-- 3	2	Et maiores libero quod aliquam sit voluptas.
-- 3	2	Commodi culpa aut nobis qui illum deserunt reiciendis.
-- 3	1	Sed cumque in et.
-- 3	1	Et sit et vel quos recusandae quo qui.
-- 3	1	Aut et similique porro ullam.
-- 3	1	Sunt vitae expedita fugiat occaecati.
-- 3	1	Aperiam voluptates quis velit explicabo ipsam vero eum.
-- 4	5	Eius ab occaecati quae eos aut enim rem.
-- 4	5	Molestiae officia architecto eius nesciunt.
-- 4	5	Doloribus temporibus hic eveniet temporibus corrupti et voluptatem et sint.
-- 4	4	Velit nihil voluptatem et sed.
-- 4	4	Debitis ipsam ut pariatur molestiae ut qui aut reiciendis.
-- 4	4	Pariatur beatae nihil cum molestiae provident vel.
-- 4	4	Saepe cumque qui pariatur.
-- 4	3	Occaecati eos possimus deleniti itaque aliquam accusamus.
-- 4	2	Dicta enim rerum culpa a quo molestiae nam repudiandae at.
-- 4	2	Est accusamus aut nisi sit aut id non natus assumenda.
-- 4	2	Quo velit iusto ducimus quos a incidunt nesciunt facilis.
-- 4	2	Error nesciunt blanditiis quae quis et tempora velit repellat sint.
-- 4	1	Voluptates nihil voluptatum quia maiores dolorum molestias occaecati.
-- 4	1	Culpa perferendis qui perferendis eligendi officia neque ex.
-- 4	1	Fugit quia perferendis magni doloremque dicta officia dignissimos ut necessitatibus.
-- 4	1	Enim doloremque delectus.
-- 4	1	Aperiam sunt aliquam illum impedit.
-- 5	5	Rem autem corporis earum necessitatibus dolores explicabo iste quo.
-- 5	5	Non est totam.
-- 5	5	Nihil quo voluptatem placeat.
-- 5	5	Esse voluptatem sed deserunt ipsum eaque maxime rerum qui.
-- 5	4	Fugit ut architecto doloremque neque quis.
-- 5	4	Maiores nulla ea non autem.
-- 5	4	Aliquid vel voluptatem.
-- 5	4	Consequatur neque beatae.
-- 5	3	Dolor soluta quisquam voluptatibus delectus.
-- 5	3	Ipsam cumque aut consectetur mollitia vel quod voluptates provident suscipit.
-- 5	3	Magnam voluptatum molestias vitae voluptatibus beatae nostrum sunt.
-- 5	2	Accusantium illo maiores et sed maiores quod natus.
-- 5	2	Alias praesentium ut voluptatem alias praesentium tempora voluptas debitis.
-- 5	2	Facere vel assumenda.
-- 5	2	Dolorem veniam nisi quidem.
-- 5	1	Est voluptatum voluptatem voluptatem est ullam quod quia in.
-- 5	1	Sit qui est sed accusantium quidem id voluptatum id.
-- 5	1	Illum omnis et excepturi totam eum omnis.
-- 5	1	Maiores quo quia.
-- 5	1	Molestias tempore est.
```

<!-- TOC --><a name="offset-and-limits"></a>
## Offset and limits

**Offset**: 
- Skips a specific set of rows of the result set
- Use `OFFSET`

**Limit**:
- Limits the result set to the specified number of rows
- Use `LIMIT`

Offset and limit are usually combined together and are useful in the ***pagination of results*** sent to the client.

We write `OFFSET` and `LIMIT` ***last*** in a query i.e after other instructions. The order between the two does not matter. However, we **specify `LIMIT` before `OFFSET` by convention**

Examples:
```SQL
-- First 3 results
SELECT *
FROM users
LIMIT 3;

-- id	username
-- 1	Reyna.Marvin
-- 2	Micah.Cremin
-- 3	Alfredo66
```

```SQL
-- Select results after the first 3 rows
SELECT *
FROM users
OFFSET 3;

-- id	username
-- 4	Gerard_Mitchell42
-- 5	Frederique_Donnelly
-- 6	David
```

```SQL
-- Order results by user_id and return 5 results after skipping the first 10
SELECT *
FROM photos
ORDER BY user_id
OFFSET 10
LIMIT 5;

-- id	url	user_id
-- 1	https://santina.net	3
-- 9	http://dayne.com	4
-- 17	https://tyrique.info	4
-- 7	https://meredith.net	4
-- 8	http://isaias.net	4
```

<!-- TOC --><a name="selecting-distinct-values-with-distinct"></a>
## Selecting distinct values with DISTINCT

`DISTINCT` is an expression that provides us with distinct values for a column in a `SELECT` statement

Example: Without `DISTINCT`
```SQL
SELECT department
FROM products;

-- department
-- Toys
-- Outdoors
-- Movies
-- Industrial
-- Movies
-- Tools
-- Toys
-- Grocery
-- Books
-- Toys
...
...
```
```SQL
SELECT DISTINCT department
FROM products;

-- department
-- Tools
-- Books
-- Outdoors
-- Toys
-- Industrial
-- Grocery
-- Movies
```

We can combine it with aggregated functions too!
```SQL
-- Get a count of distinct departments
SELECT COUNT(DISTINCT department)
FROM products;

-- count
-- 21
```

**Note**:
- `DISTINCT` is similar to but not the same as `GROUP BY`
- **You can `GROUP BY` as an alternate to `DISTINCT`**
- However, **you cannot use `DISTINCT` to replace `GROUP BY`**. **Why?** Because only group by can make use of aggregate functions to take a look at values inside those different groups

<!-- TOC --><a name="utility-operations-and-keywords"></a>
## Utility operations and keywords

1. Get the greatest value in a list: Use **`GREATEST()`** 
```SQL
SELECT GREATEST(1, 200, 10, 190);

-- greatest
-- 200
```
```SQL
SELECT name, weight, GREATEST(30, 2 * weight)
FROM products
```

2. Get the least value in a list: Use **`LEAST()`** 
```SQL
SELECT LEAST(1, 200, 10, 190);

-- least
-- 200
```
```SQL
SELECT name, weight, LEAST(30, 2 * weight)
FROM products
```

3. Use **`CASE...END`** to display column values conditionally. You will also need to use `WHEN`, `THEN` and `ELSE` constructs (Like an if-else in programming)

**Note**: `CASE...END` is not used very often since our application code generally has the logic to format the text displayed to the user based on the database values

```SQL
SELECT 
	id,
  price,
  CASE
  	WHEN price > 400 THEN 'high'
    WHEN price > 200 THEN 'medium'
    ELSE 'cheap'
  END
FROM products;

-- id	price	case
-- 1	876		high
-- 2	412		high
-- 3	10		cheap
-- 4	796		high
-- 5	10		cheap
-- 6	328		medium
-- 7	989		high
-- 8	801		high
-- 9	926		high
-- 10	298		medium
-- 11	887		high
```

<!-- TOC --><a name="writing-subqueries"></a>
## Writing subqueries

Subqueries are a powerful tool in SQL
* It is essentially a query within another query
* In many places in a `SELECT` statement, we would like to work with a single or derived set of values. However, those values are not immediately available. For deriving these before the main query executes, we can write a subquery which acts as a placeholder for the single/derived set of values.
* Use parentheses `()` to write subqueries
* Subqueries can be written in many places inside a `SELECT` including `SELECT`, `FROM`, `JOIN`, `WHERE`, etc. This is what makes them flexible but also complex to understand. 
Complex example (Do not need to understand this!):
```SQL
SELECT p1.name, (SELECT COUNT(name) FROM products)
FROM (SELECT * FROM products) as p1
JOIN (SELECT * FROM products) as p2 on p1.id = p2.id
WHERE p1.id IN (SELECT id FROM products);
```

**A practical example of writing a subquery:**

**Problem**: Find all product names and prices in a product table which have values greater than the maximum value of a product in the `Toys` department

**Naive solution**: All the data I need resides inside 'products' so I can do the following...
```SQL
SELECT name, price
FROM products
WHERE price > 800; -- Assuming that this is the max value of a product in 'Toys'
```
- This is neither a correct solution since the value `800` might be the current maximum price of a 'Toys' product but this could change. 
- This is not a scalable solution either because we have perused through the table to identify `800` the maximum in 'Toys' but in a large database we cannot always visually scan it to fetch the max (impractical, not feasible)
- What is true though is that we always need the max product value of 'Toys' and we need it in an automated way

**Subquery solution**: Replace the static value with a computed value using a subquery that runs before the main query does
```SQL
SELECT name, price
FROM products
WHERE price > (SELECT MAX(price) FROM products WHERE department = 'Toys');

-- name						price
-- Incredible Granite Mouse	989
-- Practical Rubber Mouse	948
-- Handmade Rubber Chicken	959
-- Awesome Fresh Keyboard	982
-- Incredible Granite Bacon	982
-- Fantastic Fresh Chips	966
-- Small Fresh Gloves		991
```

<!-- TOC --><a name="understanding-the-shape-of-query-results-to-simplify-subqueries"></a>
### Understanding the shape of query results to simplify subqueries

Subqueries can be difficult to understand since they can appear in many places but if we understand the shape of query results, it makes it easier to work with subqueries.

**Shape 1**: Many rows, Many columns
```SQL
SELECT *
FROM products;

-- id	name	department	price	weight
-- 1	Practical Fresh Shirt	Toys	876	3
-- 2	Gorgeous Steel Towels	Outdoors	412	16
-- ... ... ...
```
```SQL
SELECT name, price
FROM products;

-- name	price
-- Practical Fresh Shirt	876
-- Gorgeous Steel Towels	412
-- ... ... ...
```

**Shape 2**: Many rows, One column
```SQL
SELECT price
FROM products;

-- price
-- 876
-- 412
-- 10
-- 796
-- ...
```

**Shape 3**: One row, One column (***Single value*** or *S**calar query***)
```SQL
SELECT price
FROM products
WHERE id = 3;

-- price
-- 10
```

<!-- TOC --><a name="select-can-only-use-subqueries-returning-a-single-value"></a>
### SELECT can only use subqueries returning a single value

Example: Listing the price of products but also comparing it with the max price in an adjacent column
```SQL
SELECT 
	name,
	price,
	(SELECT MAX(price) FROM products) AS max_price -- Returns a single value
FROM products;

-- name                   price			max_price
-- Practical Fresh Shirt	876			991
-- Gorgeous Steel Towels	412			991
```

<!-- TOC --><a name="from-can-only-use-subqueries-compatible-with-outer-constructs"></a>
### FROM can only use subqueries compatible with outer constructs

Rules
1. Subquery in FROM must be ***compatible*** with outer `SELECT`, `WHERE`, etc depending on the construct. Ex:
	- `SELECT`: The `FROM` subquery must return columns that an outer `SELECT` will use
	- `WHERE`: The `FROM` subquery must return columns that an outer `WHERE` will criteria on
2. We must always use an ***alias*** (`AS` keyword) to give the table a name

Examples:
```SQL
-- We really don't need a subquery here but this is just for explanation's sake:
SELECT price_to_weight_ratio, name
FROM (SELECT name, price / weight as price_to_weight_ratio FROM products) AS p
WHERE price_to_weight_ratio > 5;

-- price_to_weight_ratio	name
-- 292	Practical Fresh Shirt
-- 25	Gorgeous Steel Towels
-- 34	Tasty Wooden Ball
-- ... ... ...
```
```SQL
-- The FROM subquery can also return a single value as long as it is compatible with outer constructs
SELECT * 
FROM (SELECT MAX(price) FROM products) AS p;

-- max
-- 991
```

<!-- TOC --><a name="join-can-only-use-subqueries-compatible-with-on-criteria"></a>
### JOIN can only use subqueries compatible with ON criteria

Rules:
1. Use subqueries in JOIN if the columns used in the `ON` conditional are available in the subquery result
2. We must always use an ***alias*** (`AS` keyword) to give the table a name

Example:
```SQL
-- Not a useful example since we can do it with a regular join 
-- Used for explaining the mechanics of subqueries in a JOIN:
SELECT first_name
FROM users
JOIN (
  SELECT user_id
  FROM orders
  WHERE product_id = 3
) AS o 
ON o.user_id = users.id;

-- first_name
-- Luigi
-- Kaylah
-- Clara
-- Violette
-- Kathryn
```

<!-- TOC --><a name="subqueries-that-can-be-used-in-where-depend-on-the-operator"></a>
### Subqueries that can be used in WHERE depend on the operator

Operator:
1. **`IN`** or **`NOT IN`**: Can only use subqueries that return ***single-column values*** i.e many rows, one column (Ex: Only the `id` column)
2. **`>`**, **`<`**, **`>=`**, **`<=`**, **`=`**, **`<>`**, or **`!=`**: Can only use subqueries that return ***a single value*** i.e scalar query
3. **`> ALL/SOME/MANY`**, **`< ALL/SOME/MANY`**, **`>= ALL/SOME/MANY`**, **`<= ALL/SOME/MANY`**, **`= ALL/SOME/MANY`**, or **`<> ALL/SOME/MANY`**: Can only use subqueries that return ***single-column values*** i.e many rows, one column (Ex: Only the `id` column)

**Note**: *Subqueries in WHERE can also usually be written as JOIN statements*. 
- Performance-wise, they are both equivalent, but...
- Perhaps the WHERE clause is easier to comprehend

Examples:
```SQL
-- Single value subquery for ">"
SELECT name, price
FROM products
WHERE price > (
  SELECT AVG(price)
  FROM products
 );
 
-- name	price
-- Practical Fresh Shirt	876
-- Tasty Wooden Ball	796
-- Incredible Granite Mouse	989
-- Gorgeous Rubber Ball	801
-- Generic Fresh Computer	926
 ```

```SQL
-- Single column subquery for "IN"
SELECT id, product_id
FROM orders
WHERE product_id IN (
  SELECT id
  FROM products
  WHERE id > 98
);

-- id	product_id
-- 1	100
-- 2	99
-- 130	99
-- 204	100
-- 290	99
-- 406	100
-- 408	99
-- 414	100
-- 422	99
```

<!-- TOC --><a name="all-and-some-operators-inside-a-where-clause"></a>
#### ALL and SOME operators inside a WHERE clause

1. `ALL()`: Checks the condition against ALL values of  the column returned by a subquery (*i.e All the columns must pass the condition*)
2. `SOME()`: Checks the condition against SOME values of  the column returned by a subquery (*i.e at least one of the columns must pass the condition*)

Example:
```SQL
SELECT name
FROM phones
WHERE price > ALL(
    SELECT price
    FROM phones
    WHERE manufacturer = 'Samsung'
);
```
```SQL
SELECT name
FROM phones
WHERE price > SOME(
    SELECT price
    FROM phones
    WHERE manufacturer = 'Samsung'
);
```

<!-- TOC --><a name="correlated-subqueries"></a>
### Correlated subqueries

**Definition**: Correlated subqueries means we are ***referring to a row in the outer query in the inner/subquery***

**How does it work?**
1. The *outer query goes through every row of products one-by-one* at some point of processing
2. The subquery that then runs (say in the `WHERE` condition) will also follow the same logical steps of query ***- BUT! - it can get access to the current row in the outer query***
3. By ***using an alias for the outer query table*** (`AS`), it can access that row and perform operations (Ex: comparisons)

**Note**: Think of correlated subqueries as a ***double nested for-loop*** (Outside is the outer query row, inside is the subquery row)

Example: Fetching the name & price of the most expensive product in every department
```SQL
SELECT price, department, name
FROM products AS p1
WHERE p1.price = (
  SELECT MAX(price)
  FROM products AS p2
  WHERE p2.department = p1.department
); -- This subquery needs to return a single value due to the "=" in the outer WHERE clause

-- price	department	name
-- 412		Outdoors	Gorgeous Steel Towels
-- 328		Grocery		Gorgeous Concrete Towels
-- 989		Home		Incredible Granite Mouse
-- 801		Books		Gorgeous Rubber Ball
-- 945		Baby		Handcrafted Rubber Towels
-- ... ... ...
```
```SQL
-- Without using a JOIN or GROUP BY, 
-- print the number of orders for each product
SELECT name, (
  SELECT COUNT(*)
  FROM orders AS o1
  WHERE o1.product_id = p1.id
)
FROM products AS p1

-- name						count
-- Practical Fresh Shirt	7
-- Gorgeous Steel Towels	4
-- Rustic Plastic Bacon		5
-- Tasty Wooden Ball		4
-- Fantastic Soft Fish		4
-- Gorgeous Concrete Towels	12
-- ... ... ...
```

<!-- TOC --><a name="a-select-without-a-from"></a>
#### A SELECT without a FROM

**Trick**: USE `SELECT <subquery>` and you need not use a `FROM` statement

**Condition**: Subquery needs to return a *single value* (Same as the condition for writing any subquery in a SELECT clause i.e same constraint exist even for statements that use a `FROM`)

**Why?**: You will use it when you want to do some math around a result of a couple of different values combined together

Example:
```SQL
SELECT (
  SELECT MAX(price)
  FROM products
);

-- max
-- 991
```
```SQL
-- Math: Calculate the ratio between the max and the avg price of products
SELECT (
  SELECT MAX(price)
  FROM products
) / (
  SELECT AVG(price)
  FROM products
);

-- ?column?
-- 1.9870470996330680
```

<!-- TOC --><a name="using-sets-in-sql-queries"></a>
## Using sets in SQL queries

Set operations might be easier than complex query clauses when trying to combine results from multiple queries

1. `UNION`: Combines results from two separate queries
	- If the `UNION` finds identical rows then it keeps only only copy of that row in the result (Removes duplicates)

```SQL
-- Combine list of top four expensive products with the top four price to weight ratio ones
-- and remove the duplicates
(
  SELECT *
  FROM products
  ORDER BY price DESC
  LIMIT 4
)
UNION
(
  SELECT *
  FROM products
  ORDER BY price / weight DESC
  LIMIT 4
);

-- id	name	department	price	weight
-- 38	Awesome Fresh Keyboard	Home	982	30
-- 86	Refined Concrete Pants	Sports	724	2
-- 46	Incredible Granite Bacon	Music	982	9
-- 80	Small Fresh Gloves	Garden	991	8
-- 24	Small Plastic Soap	Beauty	345	1
-- 7	Incredible Granite Mouse	Home	989	2
-- 1	Practical Fresh Shirt	Toys	876	3

-- ^ Only 7 rows above. That means one duplicate row was removed
```

2. `UNION ALL`: Same as `UNION` but does ***not*** remove duplicates i.e preserves identical rows
```SQL
(
  SELECT *
  FROM products
  ORDER BY price DESC
  LIMIT 4
)
UNION ALL
(
  SELECT *
  FROM products
  ORDER BY price / weight DESC
  LIMIT 4
);

-- id	name	department	price	weight
-- 80	Small Fresh Gloves	Garden	991	8
-- 7	Incredible Granite Mouse	Home	989	2
-- 38	Awesome Fresh Keyboard	Home	982	30
-- 46	Incredible Granite Bacon	Music	982	9
-- 7	Incredible Granite Mouse	Home	989	2
-- 86	Refined Concrete Pants	Sports	724	2
-- 24	Small Plastic Soap	Beauty	345	1
-- 1	Practical Fresh Shirt	Toys	876	3

^ There are two identical rows (check for rows with id = 7)
```

**Note**: `UNION` or `UNION ALL` *work on queries that produce the same set of columns* (i.e same name and same data type)

**Other set operations:**
1. `INTERSECT`: Find common rows between two queries. Remove duplicates
2. `INTERSECT ALL`: Find common rows between two queries
3. `EXCEPT`: Find rows that are present in the first query but *not* in the second. Remove duplicates
4. `EXCEPT ALL`: Find rows that are present in the first query but *not* in the second

<!-- TOC --><a name="order-of-execution-of-a-select-statement"></a>
## Order of execution of a SELECT statement

The logical order of execution for a SQL SELECT statement differs from how queries are written—SQL processes clauses in a specific sequence internally, not from top to bottom.

**Order of Execution Steps**:
1. `FROM` / `JOIN` – Identify tables and combine data using joins.
2. `WHERE` – Filter rows before grouping.
3. `GROUP BY` – Group filtered rows as needed.
4. `HAVING` – Filter groups (after grouping).
5. `SELECT` – Project columns and expressions to result.
6. `DISTINCT` – Remove duplicate rows from results.
7. `ORDER BY` – Sort the results.
8. `LIMIT` / `OFFSET` – Restrict the number of rows returned.

**Mnemonic to help you remember**:
**"First We Go Home, See Delhi’s Old Lanes"**

- First (FROM / JOIN)
- We (WHERE)
- Go (GROUP BY)
- Home (HAVING)
- See (SELECT)
- Delhi’s (DISTINCT)
- Old (ORDER BY)
- Lanes (LIMIT / OFFSET)

<!-- TOC --><a name="postgresql-local-installation"></a>
## PostgreSQL local installation

(Mac installation guide) 

1. Check if you have PostgreSQL installed already: `psql` command
```bash
$ psql
zsh: command not found: psql # Means postgres was not installed
```

2. Install PostgreSQL via the Desktop app (Postgres.app from https://postgresapp.com/ > Downloads tab) or from Homebrew 

3. Configure the `$PATH` (See instructions on website above / check online for Homebrew installation). Restart the terminal.

4. Run `psql` command again
```bash
➜  ~ psql
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: No such file or directory
	Is the server running locally and accepting connections on that socket?
```
This means that postgres was installed but no connection was possible

5. Open Postgresapp (GUI), open it and click "initialize" the first time

6. Run `psql` command again
```bash
 psql
psql (18.0 (Postgres.app))
Type "help" for help.

pushkar=#
```
Now, postgres is running locally

<!-- TOC --><a name="pgadmin-on-macos"></a>
### pgAdmin on macOS

**pgAdmin** is a very popular tool to interact with and manage your postgres system running on Mac

Download link: http://pgadmin.org/download

- Download it
- Run DMG file and add to Applications
- Open it
- Will open a website on first load and automatically set the system user password as the master password
- Check the icon on the top bar of your mac to access pgAdmin after you close it
- Click on left sidebar > servers/register > create > server: Add the database we created and started  from the postgres app when we hit "initialize"
- Provide name "localhost" in the General and Connection tabs
- Provide your system username (instead of "postgres") and hit the create button! This will log you into postgres when you login to your Mac (No password - left blank - use only for learning)
- You can see your database details by clicking on a database with your username provided above! There will also be a database named 'postgres'. These are two defaults.

**What does pgAdmin do?**
1. Tool to manage and inspect Postgres database (Web based)
2. Connect to local or remote databases
3. Can view/change just about anything in Postgres (incl. data)

<!-- TOC --><a name="structure-of-a-postgres-server"></a>
## Structure of a Postgres server

- A Postgres server has:
	- One server running
		- Each server has one or more databases
			- Each database has one or more tables

***Generally, when working with an app, all your data lives inside tables of a single database. It is easier to work within a single database for most needs.***

<!-- TOC --><a name="fast-rules-for-storing-numbers"></a>
## Fast rules for storing numbers

With many data types in PostgreSQL, it can be *confusing* what to choose!

Here are some "fast rules" to help you select the right type:

1. Storing an `id`? Choose the **`SERIAL`** data type
2. Storing a number without a decimal? Choose the **`INTEGER`** data type
3. Storing very accurate, decimal point data? Choose **`NUMERIC`** data type (Ex: Bank balance, gms of gold, scientific calculations)
4. Storing less accurate decimal point data? Choose **`DOUBLE PRECISION`** data type (Ex: kgs of trash in a landfill, litres of water in a tank)

<!-- TOC --><a name="storing-text"></a>
## Storing text 

Data types and their uses:
1. **`CHAR(n)`**: Stores `n` characters always - even if it has to pad spaces
2. **`VARCHAR`**: Variable length character to store any length of string
3. **`VARCHAR(n)`**: Variable length character upto `n` characters. Auto-remove characters if crossing the `n` limit
4. **`TEXT`**: Store any length of string

* Use `VARCHAR(n)` when you want the database to enforce a maximum length constraint
* Use `TEXT` when you need to store very long strings or don't require a strict length limit

There is not much of a performance difference between the two!

<!-- TOC --><a name="storing-booleans"></a>
## Storing booleans

Fast rules:
1. Storing "truthy" values: Use **`TRUE`** (Ex: true, 1, on, t, yes, y)
2. Storing "falsy" values: Use **`FALSE`** (Ex: false, 0, off, f, no, n)
3. Storing unkown values: Use **`NULL`**

<!-- TOC --><a name="storing-date-and-time-values"></a>
## Storing date and time values

There are 3 data types:
1. `DATE`: Only date
2. `TIME`: Only time (Can add timezone options too)
3. `DATATIME`: Date + time combo

These 3 accept many string formats for date and/or time and are very flexible

**Intervals:** Think of intervals as a *duration of time* (`1 D`, `1D 20H 40M 30S`, etc)
- Use the **`INTERVAL`** data type
- We can add or subtract intervals i.e durations
- We can also add to or subtract from intervals on other data & time data types 
- Storing them inside tables might not be a good idea - but can use them for calculations!

<!-- TOC --><a name="row-level-validations-or-constraints"></a>
## Row level validations or constraints

<!-- TOC --><a name="is-the-value-defined"></a>
### Is the value defined?

Use `NOT NULL` on a column to make sure that a column must always have non-values values.

`NOT NULL` prevents a `NULL` entry for that column in a row being added to the table

1. When creating a table:
```SQL
CREATE TABLE products (
	id SERIAL PRIMARY KEY,
	name VARCHAR(50),
	department VARCHAR(50),
	price INTEGER NOT NULL, -- product should always have a price,
	weight INTEGER
);
```

2. When altering an existing table:
```SQL
ALTER TABLE products
ALTER COLUMN price
SET NOT NULL;
```
**Note**: You may get an error while altering if price already contains `NULL` values. There are two ways to solve it:
- Select all rows with `NULL` values for the column. Delete them (`SELECT` + `DELETE`)
- Update all rows with `NULL` values to a default value (`UPDATE...SET... WHERE`)

Once one of the above has been done, we can go ahead with our `ALTER` operation

**Note**: Most columns in table creations are marked `NOT NULL` since it is usually undesirable

<!-- TOC --><a name="do-we-have-a-default-value"></a>
### Do we have a default value?

Use `DEFAULT <value>` along side the column during (1) table creation or (2) table alteration.
For string defaults, use `''` as a wrapper around the text. Others (Ex: Numbers) do not need one

`DEFAULT` provides a predefined value when the row being added to the table is missing that particular column value

1. Table creation
```SQL
-- During table creation 
CREATE TABLE products (
	id SERIAL PRIMARY KEY,
	name VARCHAR(50),
	department VARCHAR(50),
	price INTEGER DEFAULT 99999,
	weight INTEGER
);
```
2. Table altering
```SQL
-- During table alteration
ALTER TABLE products
ALTER COLUMN price
SET DEFAULT 99999;
```

*^After this, we can add a new product to the table without a price specified and it will have 99999!*

<!-- TOC --><a name="is-the-value-unique"></a>
### Is the value unique?

Use the `UNIQUE` attribute on a column during (1) table creation or (2) table alteration

`UNIQUE` prevents duplicate entries for that column being added to the table

1. Table creation
```SQL
-- No two products should have the same name
CREATE TABLE products (
	id SERIAL PRIMARY KEY,
	name VARCHAR(50) UNIQUE,
	department VARCHAR(50),
	price INTEGER,
	weight INTEGER
);
```
2. Table altering
```SQL
-- Note that the syntax is different than the other alter commands
ALTER TABLE products
ADD UNIQUE(name);
```

<!-- TOC --><a name="adding-multicolumn-uniqueness"></a>
#### Adding multicolumn uniqueness

Use `,` comma separated column names
Ex:
```SQL
ALTER TABLE products
ADD UNIQUE(name, department);
```

**Note**: You may get an error while altering if price already contains *duplicate* values. There are two ways to solve it:
- Select all rows with `NULL` values for the column. Delete them (`SELECT` + `DELETE`)
- Update all rows with `NULL` values to a default value (`UPDATE...SET... WHERE`)

Once one of the above has been done, we can go ahead with our `ALTER` operation

<!-- TOC --><a name="dropping-a-constraint"></a>
### Dropping a constraint

Use the `DROP CONSTRAINT <constraint>` syntax in the `ALTER` statement.

The name of the constraint can be found in pgAdmin in the databases > `<dbname>` > Schemas > Public > Tables > `<tablename>` > Constraints (Once dropped, right click on the Constraints and hit refresh)

```SQL
ALTER TABLE products
DROP CONSTRAINT products_name_key;
```

<!-- TOC --><a name="is-value-passing-conditional-checks"></a>
### Is value passing conditional checks?

When inserting a row, we can check whether a column value matches certain criteria before we allow the row to be stored. Ex: Price of a product cannot be zero or less

Use the `CHECK ()` syntax

Usually, we can use comparison operators such as `>`, `<`, `>=`, `<=`, `!=`, `<>`. We can also use `IN`, `NOT IN`, and so on

**Note**: *We only have access to the row data that is being inserted* during the checks i.e we cannot have subqueries that can scan other rows from this or any other tables!

We can add the check during (1) table creation or (2) table alteration.

1. Table creation
```SQL
CREATE TABLE products (
	id SERIAL PRIMARY KEY,
	name VARCHAR(50) UNIQUE,
	department VARCHAR(50),
	price INTEGER CHECK (price > 0),
	weight INTEGER
);
```
2. Table altering
```SQL
-- Note that the syntax is different than the other alter commands
ALTER TABLE products
ADD CHECK(price > 0);
```

**Note**: You may get an error while altering if the row already contains data failing the check. There are two ways to solve it:
- Select all rows with `NULL` values for the column. Delete them (`SELECT` + `DELETE`)
- Update all rows with `NULL` values to a default value (`UPDATE...SET... WHERE`)

Once one of the above has been done, we can go ahead with our `ALTER` operation


<!-- TOC --><a name="multicolumn-checks"></a>
#### Multicolumn checks

Create a separate `CHECK` constraint not linked to a specific column during table creation or alteration

```SQL
CREATE TABLE orders (
	id SERIAL PRIMARY KEY,
	name VARCHAR(40) NOT NULL,
	created_at TIMESTAMP NOT NULL,
	est_delivery TIMESTAMP NOT NULL,
	CHECK (est_delivery > created_at)
);
```

<!-- TOC --><a name="typecasting-values"></a>
## Typecasting values

Use the `::<postgresdatatype>` syntax next to any data literal

Ex:
```SQL
INSERT INTO products (name, department, price, weight)
VALUES ('Straw', 'Steel', 400.566::INTEGER, 1); -- Price will be stored as `400`
```

<!-- TOC --><a name="app-vs-database-validations"></a>
## App vs database validations

Pros and cons exist in each approach

**App**: (Ex: Web app client)
- Easier to express ***more complex validations***
- Easier to ***apply new validation rules***
- Many ***libraries handle validations automatically***

**Database**:
- ***Valid still applied even if connected with a different client*** 
- ***Guarantees that validation is always applied***
- ***Can only apply new validation if all rows satisfy it*** (Need to be very careful adding new validations to a production database)

<!-- TOC --><a name="database-design"></a>
## Database design 

Practical examples of designing a database for your app

<!-- TOC --><a name="database-schema-design-tools"></a>
### Database schema design tools

1. dbdiagram.io (Code based)
2. drawsql.app
3. sqldbm.app
4. quickdatabasediagrams.com
5. ondras.zarovi.cz/sql/demo (Very good, open source)


