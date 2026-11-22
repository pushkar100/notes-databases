# PostgreSQL JSON and JSONB

[Neon PostgreSQL Tutorial](https://neon.com/postgresql/tutorial/json)

- [PostgreSQL JSON](#postgresql-json)
   * [What is JSON](#what-is-json)
      + [Objects](#objects)
      + [Arrays](#arrays)
      + [Data types in JSON](#data-types-in-json)
   * [PostgreSQL JSON data types](#postgresql-json-data-types)
      + [JSON vs. JSONB](#json-vs-jsonb)
      + [How JSON and JSONB differ in practice](#how-json-and-jsonb-differ-in-practice)
         - [1) Storage and parsing: Verbatim text vs. Processed binary](#1-storage-and-parsing-verbatim-text-vs-processed-binary)
         - [2) Query performance: Repeated text parsing vs. Direct binary lookup](#2-query-performance-repeated-text-parsing-vs-direct-binary-lookup)
         - [3) Indexing: Expression-only vs. Whole-document GIN indexing](#3-indexing-expression-only-vs-whole-document-gin-indexing)
   * [When to use JSON/JSONB vs. stricter data types](#when-to-use-jsonjsonb-vs-stricter-data-types)
      + [Decision matrix](#decision-matrix)
      + [Quick rule of thumb](#quick-rule-of-thumb)
      + [Real-world SQL comparison: When to use which](#real-world-sql-comparison-when-to-use-which)
         - [1) Where strict relational columns shine (Transactional integrity)](#1-where-strict-relational-columns-shine-transactional-integrity)
         - [2) Where JSONB shines (Dynamic and sparse attributes)](#2-where-jsonb-shines-dynamic-and-sparse-attributes)
         - [3) The Anti-Pattern: Putting everything into JSONB](#3-the-anti-pattern-putting-everything-into-jsonb)
   * [PostgreSQL JSON examples](#postgresql-json-examples)
      + [1) Storing JSON objects example](#1-storing-json-objects-example)
      + [2) Storing JSON arrays example](#2-storing-json-arrays-example)
   * [Creating JSON data](#creating-json-data)
      + [1) Using to_json and to_jsonb](#1-using-to_json-and-to_jsonb)
      + [2) Creating JSON objects with json_build_object and jsonb_build_object](#2-creating-json-objects-with-json_build_object-and-jsonb_build_object)
      + [3) Creating JSON arrays with json_build_array and jsonb_build_array](#3-creating-json-arrays-with-json_build_array-and-jsonb_build_array)
   * [Searching JSON data](#searching-json-data)
      + [1) Containment operator (@>)](#1-containment-operator-)
      + [2) Contained-by operator (<@)](#2-contained-by-operator-)
      + [3) Key existence operators (?, ?|, ?&)](#3-key-existence-operators-)
   * [Querying JSON data](#querying-json-data)
      + [1) Extracting nested data with path operators (#> and #>>)](#1-extracting-nested-data-with-path-operators-and-)
      + [2) Querying with SQL/JSON path (jsonb_path_query)](#2-querying-with-sqljson-path-jsonb_path_query)
   * [Modifying JSON data](#modifying-json-data)
      + [1) Merging and updating with the concatenation operator (||)](#1-merging-and-updating-with-the-concatenation-operator-)
      + [2) Updating values with jsonb_set](#2-updating-values-with-jsonb_set)
      + [3) Inserting values with jsonb_insert](#3-inserting-values-with-jsonb_insert)
      + [4) Deleting keys and paths with - and #-](#4-deleting-keys-and-paths-with-and-)
   * [Working with JSON arrays](#working-with-json-arrays)
      + [1) Finding array length with jsonb_array_length](#1-finding-array-length-with-jsonb_array_length)
      + [2) Expanding arrays into rows with jsonb_array_elements and jsonb_array_elements_text](#2-expanding-arrays-into-rows-with-jsonb_array_elements-and-jsonb_array_elements_text)
      + [3) Accessing array elements with negative indexes](#3-accessing-array-elements-with-negative-indexes)
   * [Working with JSON objects](#working-with-json-objects)
      + [1) Getting object keys with jsonb_object_keys](#1-getting-object-keys-with-jsonb_object_keys)
      + [2) Expanding key-value pairs with jsonb_each and jsonb_each_text](#2-expanding-key-value-pairs-with-jsonb_each-and-jsonb_each_text)
      + [3) Unpacking objects into columns with jsonb_to_record](#3-unpacking-objects-into-columns-with-jsonb_to_record)
   * [Aggregating JSON data](#aggregating-json-data)
      + [1) Aggregating rows into a JSON array with jsonb_agg](#1-aggregating-rows-into-a-json-array-with-jsonb_agg)
      + [2) Aggregating key-value pairs into a JSON object with jsonb_object_agg](#2-aggregating-key-value-pairs-into-a-json-object-with-jsonb_object_agg)
   * [JSON utility functions](#json-utility-functions)
      + [1) Formatting JSON with jsonb_pretty](#1-formatting-json-with-jsonb_pretty)
      + [2) Checking types with jsonb_typeof](#2-checking-types-with-jsonb_typeof)
      + [3) Removing null fields with jsonb_strip_nulls](#3-removing-null-fields-with-jsonb_strip_nulls)
   * [Summary](#summary)

Last updated: 2026-06-19

**Summary**: in this tutorial, you will learn about JSON and how to work with JSON data in PostgreSQL using the PostgreSQL JSON and JSONB data types.

## What is JSON

JSON stands for JavaScript Object Notation. JSON is a lightweight data interchange format that is readable for humans and simple for computers to parse.

JSON is based on two main data structures **objects** and **arrays**:

### Objects

An object is defined as an unordered collection of key-value pairs enclosed in curly braces `{}`. Each pair includes:

- A key which is a string surrounded by double quotes (").
- A colon `:` that separates the key and value.
- A value that can be a string, a number, or even an object.

For example, the following illustrates a JSON object that represents a film:

```json
{ "title": "Chamber Italian", "release_year": 2006, "length": 117 }
```

The film object has three keys `title`, `release_year`, and `length` with associated values.

### Arrays

An array is an ordered list of values enclosed in square brackets `[]`. The values do not have to be the same type. Additionally, an array may contain values of any valid JSON data type including objects and arrays.

For example, the following shows an array that stores three film titles as strings:

```json
["Chamber Italian", "Grosse Wonderful", "Airport Pollock"]
```

### Data types in JSON

JSON supports some data types including:

- String: "Joe"
- Number: 100, 9.99, …
- Boolean: true and false.
- Null: null

JSON data can be particularly useful for creating configuration files or exchanging data between a client and a server.

## PostgreSQL JSON data types

PostgreSQL offers two data types for storing JSON:

- JSON – store an exact copy of the JSON text.
- JSONB – store the JSON data in binary format.

### JSON vs. JSONB

The following table illustrates the key differences between JSON and JSONB types:

| Feature           | JSON                                                            | JSONB                                                                |
| ----------------- | --------------------------------------------------------------- | -------------------------------------------------------------------- |
| Storage           | Textual representation (verbatim)                               | Binary storage format                                                |
| Size              | Typically larger because it retains the whitespace in JSON data | Typically smaller                                                    |
| Indexing          | Full-text search indexes                                        | Binary indexes                                                       |
| Performance       | Slightly slower                                                 | Generally faster                                                     |
| Query performance | Slower due to parsing                                           | Faster due to binary storage                                         |
| Parsing           | Parse each time                                                 | Parse once, store in binary format                                   |
| Data manipulation | Simple and easy                                                 | More complex                                                         |
| Ordering of keys  | Preserved                                                       | Not preserved                                                        |
| Duplicate keys    | Allow duplicate key, the last value is retained                 | Do not allow duplicate keys.                                         |
| Use cases         | Storing configuration data, log data, simple JSON documents     | Storing JSON documents where fast querying and indexing are required |

In practice, you should **use JSONB to store JSON data unless you have specialized requirements such as retaining the ordering of keys in the JSON documents**.

### How JSON and JSONB differ in practice

To better understand why JSONB is almost always preferred, let's explore the key differences with concrete examples:

#### 1) Storage and parsing: Verbatim text vs. Processed binary

- **`JSON`** stores an exact, character-for-character copy of the input text, preserving extra whitespace and duplicate keys.
- **`JSONB`** parses the text upon input, strips unneeded whitespace, discards duplicate keys (keeping only the last value), and stores the result in a decomposed binary format.

For example, observe what happens when you insert JSON with duplicate keys and irregular spacing:

```sql
SELECT
  '{"fruit": "apple",   "fruit": "banana"}'::JSON AS json_verbatim,
  '{"fruit": "apple",   "fruit": "banana"}'::JSONB AS jsonb_processed;
```

Output:

```text
            json_verbatim             |   jsonb_processed
--------------------------------------+---------------------
 {"fruit": "apple",   "fruit": "banana"} | {"fruit": "banana"}
(1 row)
```

The `JSON` type preserved the raw string exactly as entered, whereas `JSONB` stripped redundant spacing and deduplicated the `fruit` key.

#### 2) Query performance: Repeated text parsing vs. Direct binary lookup

- **`JSON`** values must be reparsed from scratch every time an expression accesses an attribute. If a table has 100,000 rows, running `SELECT data ->> 'city' FROM users;` requires PostgreSQL to parse 100,000 JSON strings character-by-character.
- **`JSONB`** stores internal binary offsets and pre-sorted keys. PostgreSQL skips directly to the target key in memory without parsing text, making reads and lookups dramatically faster.

#### 3) Indexing: Expression-only vs. Whole-document GIN indexing

Indexing is the most significant practical differentiator:

- **`JSON` cannot be indexed as a whole document**. You can only index a specific, pre-determined attribute using a standard B-tree expression index:

  ```sql
  -- For JSON columns, you must create an index for every specific field you query:
  CREATE INDEX idx_user_city ON users ((profile ->> 'city'));
  ```

- **`JSONB` supports GIN (Generalized Inverted Index)** over the entire document. A single GIN index indexes **all keys and values** simultaneously:

  ```sql
  -- Indexes the entire JSONB document in one step:
  CREATE INDEX idx_products_properties ON products USING gin (properties);
  ```

  With this single GIN index, PostgreSQL can instantly accelerate searches on any key or combination of keys using containment operators (`@>`):

  ```sql
  -- Efficiently uses the GIN index:
  SELECT * FROM products WHERE properties @> '{"color": "white"}';
  ```

For maximum search speed on containment queries, `JSONB` also supports the `jsonb_path_ops` operator class, which creates smaller, faster GIN indexes optimized specifically for the `@>` operator:

```sql
CREATE INDEX idx_products_properties_path ON products USING gin (properties jsonb_path_ops);
```

## When to use JSON/JSONB vs. stricter data types

PostgreSQL is a relational database first. While JSONB provides immense flexibility, you should not default to putting everything into a single JSON column.

The general rule of thumb: **Use strict relational columns by default; use JSONB for flexible, sparse, or external data.**

### Decision matrix

The following decision matrix will help you choose between standard relational columns and JSON/JSONB:

| Requirement / Scenario | Use Stricter Relational Columns                                                 | Use JSON / JSONB                                                    |
| :--------------------- | :------------------------------------------------------------------------------ | :------------------------------------------------------------------ |
| **Schema Stability**   | Fixed, well-defined, and predictable schema                                     | Dynamic, semi-structured, or frequently changing schema             |
| **Data Integrity**     | Needs database-level constraints (`FOREIGN KEY`, `NOT NULL`, `CHECK`, `UNIQUE`) | Flexible validation (handled primarily by application logic)        |
| **Row Attributes**     | Uniform (almost every row has the same fields)                                  | Sparse (different rows have completely different attributes)        |
| **Querying & Joins**   | Columns are frequently joined, grouped, or filtered                             | Data is usually read or written together as a single unit           |
| **Storage & Indexing** | Compact native storage; standard B-Tree indexes                                 | Slightly larger binary storage; requires GIN indexes for searching  |
| **Typical Examples**   | `user_id`, `email`, `price`, `status`, `created_at`                             | Product specs (RAM, shoe size), raw API responses, user preferences |

### Quick rule of thumb

- **Choose Relational (VARCHAR, INT, TIMESTAMP, etc.)** if you need database-enforced data types, foreign keys to other tables, or your schema rarely changes.
- **Choose JSONB** if you are storing third-party API payloads, arbitrary user settings, or polymorphic product attributes where each product has unique features.

### Real-world SQL comparison: When to use which

To understand how to apply this in practice, let's look at three common scenarios in database design:

#### 1) Where strict relational columns shine (Transactional integrity)

Consider an `orders` table. Every order must belong to a valid user, have a positive payment amount, and track its status and timestamp:

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users(id),
    total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

**Why strict relational columns are best here:**

- **Database constraints**: `CHECK (total_amount >= 0)` prevents invalid negative values.
- **Foreign keys**: `REFERENCES users(id)` guarantees an order cannot point to a non-existent customer.
- **Performance**: High-frequency queries like `WHERE status = 'pending' AND created_at > ...` run at maximum speed with standard B-tree indexes.

#### 2) Where JSONB shines (Dynamic and sparse attributes)

Now consider an e-commerce catalog that sells diverse products like laptops, clothing, and books. Each category has completely different specifications:

- Laptops have `cpu`, `ram`, and `storage`.
- Clothing has `color`, `size`, and `fabric`.
- Books have `isbn` and `page_count`.

Instead of creating dozens of empty/nullable columns (`cpu`, `ram`, `shoe_size`, `fabric`, etc.) or building an overly complicated table structure, use a hybrid table with a JSONB column:

```sql
CREATE TABLE catalog_items (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    category VARCHAR(50) NOT NULL,
    attributes JSONB
);

-- Inserting a laptop with hardware specifications
INSERT INTO catalog_items (name, price, category, attributes)
VALUES ('DevBook Pro', 1299.99, 'electronics', '{"cpu": "M3", "ram": "16GB", "storage": "512GB"}');

-- Inserting clothing with apparel specifications
INSERT INTO catalog_items (name, price, category, attributes)
VALUES ('Classic Hoodie', 49.99, 'clothing', '{"color": "navy", "size": "XL", "fabric": "cotton"}');
```

You can query the common relational columns directly while filtering inside the JSONB attributes:

```sql
SELECT
  name,
  price,
  attributes ->> 'cpu' AS cpu
FROM catalog_items
WHERE category = 'electronics'
  AND attributes @> '{"ram": "16GB"}';
```

Output:

```text
    name     |  price  | cpu
-------------+---------+-----
 DevBook Pro | 1299.99 | M3
(1 row)
```

#### 3) The Anti-Pattern: Putting everything into JSONB

A common beginner mistake is treating PostgreSQL like a pure document store and putting all data into a single JSONB column:

```sql
-- AVOID THIS PATTERN:
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    data JSONB -- {"email": "user@example.com", "role": "admin", "password_hash": "..."}
);
```

**Why this is problematic:**

- **No data integrity**: Typos like `{"emaiil": "..."}` will be accepted without warning.
- **No native constraints**: You cannot easily add `UNIQUE` or `NOT NULL` to `data ->> 'email'` without complex expression indexes.
- **Cumbersome queries**: Writing `data ->> 'role'` in every query is more verbose, harder to maintain, and slower than a native, typed column.

## PostgreSQL JSON examples

Let's take some examples of storing JSON data in the PostgreSQL database.

### 1) Storing JSON objects example

First, create a new table called `products`:

```sql
CREATE TABLE products(
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    properties JSONB
);
```

The `products` table includes three columns:

1. The `id` column is the primary key column that uniquely identifies the product.
2. The `name` column stores the product name.
3. The `properties` column stores various properties of a product such as size, and color.

Second, insert JSON data into the `products` table:

```sql
INSERT INTO products(name, properties)
VALUES('Ink Fusion T-Shirt','{"color": "white", "size": ["S","M","L","XL"]}')
RETURNING *;
```

Output:

```text
 id |        name        |                    properties
----+--------------------+---------------------------------------------------
  1 | Ink Fusion T-Shirt | {"size": ["S", "M", "L", "XL"], "color": "white"}
(1 row)
```

Third, [insert multiple rows](https://neon.com/postgresql/tutorial/postgresql-insert-multiple-rows) into the `products` table:

```sql
INSERT INTO products(name, properties)
VALUES('ThreadVerse T-Shirt','{"color": "black", "size": ["S","M","L","XL"]}'),
      ('Design Dynamo T-Shirt','{"color": "blue", "size": ["S","M","L","XL"]}')
RETURNING *;
```

Output:

```text
 id |        name         |                    properties
----+---------------------+---------------------------------------------------
  2 | ThreadVerse T-Shirt | {"size": ["S", "M", "L", "XL"], "color": "black"}
  3 | Design Dynamo T-Shirt | {"size": ["S", "M", "L", "XL"], "color": "blue"}
(2 rows)
```

Fourth, retrieve JSON data from the `products` table:

```sql
SELECT id, name, properties
FROM products;
```

Output:

```text
 id |         name          |                    properties
----+-----------------------+---------------------------------------------------
  1 | Ink Fusion T-Shirt    | {"size": ["S", "M", "L", "XL"], "color": "white"}
  2 | ThreadVerse T-Shirt   | {"size": ["S", "M", "L", "XL"], "color": "black"}
  3 | Design Dynamo T-Shirt | {"size": ["S", "M", "L", "XL"], "color": "blue"}
(3 rows)
```

Fifth, retrieve the product with the colors extracted from the JSON data in the `properties` column:

```sql
SELECT
  id,
  name,
  properties -> 'color' color
FROM
  products;
```

Output:

```text
 id |         name          |  color
----+-----------------------+---------
  1 | Ink Fusion T-Shirt    | "white"
  2 | ThreadVerse T-Shirt   | "black"
  3 | Design Dynamo T-Shirt | "blue"
(3 rows)
```

The `->` operator extracts a JSON object field by a key. In this example, we use the operator `->` to extract the color of the properties object:

```sql
properties -> 'color'
```

The values in the color column are surrounded by double quotes (").

To extract a JSON object field by a key as text, you can use the `->>` operator. For example:

```sql
SELECT
  id,
  name,
  properties ->> 'color' color
FROM
  products;
```

Output:

```text
 id |         name          | color
----+-----------------------+-------
  1 | Ink Fusion T-Shirt    | white
  2 | ThreadVerse T-Shirt   | black
  3 | Design Dynamo T-Shirt | blue
(3 rows)
```

Sixth, retrieve the products with the colors black and white using the `->>` operator in the [WHERE](https://neon.com/postgresql/tutorial/postgresql-where) clause:

```sql
SELECT
  id,
  name,
  properties ->> 'color' color
FROM
  products
WHERE
  properties ->> 'color' IN ('black', 'white');
```

Output:

```text
 id |        name         | color
----+---------------------+-------
  1 | Ink Fusion T-Shirt  | white
  2 | ThreadVerse T-Shirt | black
(2 rows)
```

### 2) Storing JSON arrays example

First, create a table called `contacts` for storing the contact persons:

```sql
CREATE TABLE contacts(
   id SERIAL PRIMARY KEY,
   name VARCHAR(255) NOT NULL,
   phones JSONB
);
```

The `phones` column has the JSONB data type. We'll store both work phone and personal phone numbers in a JSON array in the `phones` column.

Second, insert new rows into the `contacts` table:

```sql
INSERT INTO contacts(name, phones)
VALUES
   ('John Doe','["408-111-2222", "408-111-2223"]'),
   ('Jane Doe','["212-111-2222", "212-111-2223"]')
RETURNING *;
```

Output:

```text
 id |   name   |              phones
----+----------+----------------------------------
  1 | John Doe | ["408-111-2222", "408-111-2223"]
  2 | Jane Doe | ["212-111-2222", "212-111-2223"]
(2 rows)
```

Third, retrieve the contacts with the work phone numbers from the `contacts` table:

```sql
SELECT
  name,
  phones ->> 0 "work phone"
FROM
  contacts;
```

Output:

```
   name   |  work phone
----------+--------------
 John Doe | 408-111-2222
 Jane Doe | 212-111-2222
(2 rows)
```

The `->> index` extracts the index element in an array. In this example, we use the `->> 0` to extract the first elements in the `phones` array as text.

## Creating JSON data

PostgreSQL provides several built-in functions to construct JSON and JSONB data directly from SQL values, rows, and key-value pairs.

### 1) Using to_json and to_jsonb

The `to_json()` and `to_jsonb()` functions convert any SQL value, date, or record into its corresponding JSON representation:

```sql
SELECT
  to_jsonb(true) AS bool_val,
  to_jsonb('2026-06-19'::DATE) AS date_val;
```

Output:

```text
 bool_val |  date_val
----------+-------------
 true     | "2026-06-19"
(1 row)
```

### 2) Creating JSON objects with json_build_object and jsonb_build_object

The `jsonb_build_object()` function creates a JSONB object from a variable list of arguments where alternating arguments are keys and values:

```sql
SELECT jsonb_build_object(
  'title', 'PostgreSQL JSON Tutorial',
  'views', 1500,
  'is_published', true
) AS post;
```

Output:

```text
                                        post
------------------------------------------------------------------------------------
 {"views": 1500, "title": "PostgreSQL JSON Tutorial", "is_published": true}
(1 row)
```

### 3) Creating JSON arrays with json_build_array and jsonb_build_array

The `jsonb_build_array()` function creates a JSONB array from an arbitrary list of arguments:

```sql
SELECT jsonb_build_array('PostgreSQL', 16, true) AS tech_stack;
```

Output:

```text
        tech_stack
--------------------------
 ["PostgreSQL", 16, true]
(1 row)
```

## Searching JSON data

When querying JSONB columns, PostgreSQL offers dedicated containment and existence operators to filter data efficiently.

### 1) Containment operator (@>)

The containment operator `@>` tests whether the JSONB document on the left contains the JSONB document on the right:

```sql
SELECT id, name
FROM products
WHERE properties @> '{"color": "white"}';
```

Output:

```text
 id |        name
----+--------------------
  1 | Ink Fusion T-Shirt
(1 row)
```

In this example, PostgreSQL returns only the products whose `properties` object includes `"color": "white"`.

### 2) Contained-by operator (<@)

The `<@` operator tests whether the left JSONB value is completely contained within the right JSONB value:

```sql
SELECT '{"color": "white"}'::JSONB <@ properties AS is_contained
FROM products
WHERE id = 1;
```

Output:

```text
 is_contained
--------------
 true
(1 row)
```

### 3) Key existence operators (?, ?|, ?&)

You can check whether keys or array elements exist using key existence operators:

- `?` returns true if a specific top-level key or string element exists.
- `?|` returns true if _any_ of the specified keys exist.
- `?&` returns true if _all_ of the specified keys exist.

For example, check if each product's `properties` column has a `color` key:

```sql
SELECT
  name,
  properties ? 'color' AS has_color
FROM products;
```

Output:

```text
         name          | has_color
-----------------------+-----------
 Ink Fusion T-Shirt    | t
 ThreadVerse T-Shirt   | t
 Design Dynamo T-Shirt | t
(3 rows)
```

Check whether a JSON object contains any key from a list using `?|`:

```sql
SELECT '{"color": "red", "size": "M"}'::JSONB ?| array['color', 'discount'] AS has_any;
```

Output:

```text
 has_any
---------
 t
(1 row)
```

## Querying JSON data

In addition to basic key extraction (`->` and `->>`), PostgreSQL provides path operators and SQL/JSON path queries to navigate deeply nested data structures.

### 1) Extracting nested data with path operators (#> and #>>)

- `#> path` extracts a nested JSON value as JSON/JSONB using a text array representing the path.
- `#>> path` extracts the nested JSON value as plain text.

For example:

```sql
SELECT '{"item": {"name": "Laptop", "specs": {"ram": "16GB", "storage": "512GB"}}}'::JSONB #> '{item, specs}' AS specs;
```

Output:

```text
                 specs
---------------------------------------
 {"ram": "16GB", "storage": "512GB"}
(1 row)
```

To extract the RAM value directly as text:

```sql
SELECT '{"item": {"name": "Laptop", "specs": {"ram": "16GB", "storage": "512GB"}}}'::JSONB #>> '{item, specs, ram}' AS ram;
```

Output:

```text
 ram
------
 16GB
(1 row)
```

### 2) Querying with SQL/JSON path (jsonb_path_query)

PostgreSQL supports the SQL/JSON path standard through functions like `jsonb_path_query()`, allowing you to filter items using path expressions:

```sql
SELECT jsonb_path_query(
  '{"items": [{"name": "Pen", "price": 5}, {"name": "Notebook", "price": 15}]}',
  '$.items[*] ? (@.price > 10).name'
) AS expensive_item;
```

Output:

```text
 expensive_item
----------------
 "Notebook"
(1 row)
```

## Modifying JSON data

PostgreSQL provides operators and functions to update, merge, insert, and delete data in JSONB documents.

### 1) Merging and updating with the concatenation operator (||)

The `||` operator merges two JSONB objects. If a key already exists, its value is updated with the value from the right-hand object:

```sql
SELECT '{"color": "white", "size": "M"}'::JSONB || '{"size": "L", "price": 25}'::JSONB AS updated_product;
```

Output:

```text
                updated_product
-----------------------------------------------
 {"size": "L", "color": "white", "price": 25}
(1 row)
```

### 2) Updating values with jsonb_set

The `jsonb_set()` function updates or inserts a value at a specified path within a JSONB document:

Syntax:

```text
jsonb_set(target JSONB, path TEXT[], new_value JSONB [, create_missing BOOLEAN])
```

For example, update the `ram` value:

```sql
SELECT jsonb_set(
  '{"name": "Laptop", "specs": {"ram": "8GB"}}'::JSONB,
  '{specs, ram}',
  '"16GB"'
) AS updated_specs;
```

Output:

```text
                 updated_specs
-----------------------------------------------
 {"name": "Laptop", "specs": {"ram": "16GB"}}
(1 row)
```

### 3) Inserting values with jsonb_insert

The `jsonb_insert()` function inserts a new value at a given position without overwriting existing elements:

```sql
SELECT jsonb_insert(
  '["apple", "banana"]'::JSONB,
  '{1}',
  '"orange"'
) AS fruits;
```

Output:

```text
            fruits
-------------------------------
 ["apple", "orange", "banana"]
(1 row)
```

### 4) Deleting keys and paths with - and #-

- `- text` removes a key from an object or an element from an array.
- `#- path` removes an element or field at the specified nested path.

For example:

```sql
SELECT
  '{"color": "black", "material": "cotton", "discount": 10}'::JSONB - 'discount' AS removed_key,
  '{"specs": {"ram": "8GB", "gpu": "integrated"}}'::JSONB #- '{specs, gpu}' AS removed_path;
```

Output:

```text
               removed_key                |       removed_path
------------------------------------------+---------------------------
 {"color": "black", "material": "cotton"} | {"specs": {"ram": "8GB"}}
(1 row)
```

## Working with JSON arrays

PostgreSQL includes specialized functions to inspect array lengths and expand array elements into table rows.

### 1) Finding array length with jsonb_array_length

The `jsonb_array_length()` function returns the number of elements in the outermost JSON array:

```sql
SELECT
  name,
  jsonb_array_length(phones) AS phone_count
FROM contacts;
```

Output:

```text
   name   | phone_count
----------+-------------
 John Doe |           2
 Jane Doe |           2
(2 rows)
```

### 2) Expanding arrays into rows with jsonb_array_elements and jsonb_array_elements_text

- `jsonb_array_elements()` expands a JSON array into a set of JSONB values.
- `jsonb_array_elements_text()` expands a JSON array into a set of plain text values.

For example, unpack phone numbers into individual rows:

```sql
SELECT
  name,
  jsonb_array_elements_text(phones) AS phone
FROM contacts
WHERE name = 'John Doe';
```

Output:

```text
   name   |    phone
----------+--------------
 John Doe | 408-111-2222
 John Doe | 408-111-2223
(2 rows)
```

### 3) Accessing array elements with negative indexes

Negative indexes allow you to count elements backward from the end of an array (`-1` retrieves the last element):

```sql
SELECT
  name,
  phones ->> -1 AS "last phone"
FROM contacts;
```

Output:

```text
   name   |  last phone
----------+--------------
 John Doe | 408-111-2223
 Jane Doe | 212-111-2223
(2 rows)
```

## Working with JSON objects

PostgreSQL provides functions to inspect keys, decompose objects into key-value pairs, and unpack objects into tabular records.

### 1) Getting object keys with jsonb_object_keys

The `jsonb_object_keys()` function returns all keys of the outermost JSON object as a set of rows:

```sql
SELECT DISTINCT jsonb_object_keys(properties) AS property_key
FROM products;
```

Output:

```text
 property_key
--------------
 color
 size
(2 rows)
```

### 2) Expanding key-value pairs with jsonb_each and jsonb_each_text

The `jsonb_each_text()` function unpacks the outermost JSON object into rows with two columns: `key` and `value` (both as text):

```sql
SELECT key, value
FROM jsonb_each_text('{"color": "white", "brand": "Nike", "size": "M"}'::JSONB);
```

Output:

```text
  key  | value
-------+-------
 color | white
 brand | Nike
 size  | M
(3 rows)
```

### 3) Unpacking objects into columns with jsonb_to_record

The `jsonb_to_record()` function turns a JSON object into an arbitrary record with specified column types:

```sql
SELECT *
FROM jsonb_to_record('{"name": "T-Shirt", "price": 19.99, "in_stock": true}'::JSONB)
AS product(name VARCHAR, price NUMERIC, in_stock BOOLEAN);
```

Output:

```text
  name   | price | in_stock
---------+-------+----------
 T-Shirt | 19.99 | t
(1 row)
```

## Aggregating JSON data

PostgreSQL provides aggregate functions that combine multiple rows of data into a single JSON array or JSON object.

### 1) Aggregating rows into a JSON array with jsonb_agg

The `jsonb_agg()` function collects values or rows across multiple table rows into a single JSON array:

```sql
SELECT jsonb_agg(name) AS product_names
FROM products;
```

Output:

```text
                             product_names
------------------------------------------------------------------------
 ["Ink Fusion T-Shirt", "ThreadVerse T-Shirt", "Design Dynamo T-Shirt"]
(1 row)
```

You can also aggregate structured row objects:

```sql
SELECT jsonb_agg(
  jsonb_build_object('id', id, 'name', name)
) AS products_list
FROM products;
```

Output:

```text
                                             products_list
-------------------------------------------------------------------------------------------------------
 [{"id": 1, "name": "Ink Fusion T-Shirt"}, {"id": 2, "name": "ThreadVerse T-Shirt"}, {"id": 3, "name": "Design Dynamo T-Shirt"}]
(1 row)
```

### 2) Aggregating key-value pairs into a JSON object with jsonb_object_agg

The `jsonb_object_agg()` function aggregates key-value pairs from multiple rows into a single JSON object:

```sql
SELECT jsonb_object_agg(id, name) AS id_to_name
FROM products;
```

Output:

```text
                                               id_to_name
---------------------------------------------------------------------------------------------------------
 {"1": "Ink Fusion T-Shirt", "2": "ThreadVerse T-Shirt", "3": "Design Dynamo T-Shirt"}
(1 row)
```

## JSON utility functions

PostgreSQL provides handy utility functions for inspecting data types, formatting JSON, and cleaning null values.

### 1) Formatting JSON with jsonb_pretty

The `jsonb_pretty()` function formats a JSONB value as human-readable, indented text:

```sql
SELECT jsonb_pretty(properties)
FROM products
WHERE id = 1;
```

Output:

```text
           jsonb_pretty
-----------------------------------
 {                                +
     "size": [                    +
         "S",                     +
         "M",                     +
         "L",                     +
         "XL"                     +
     ],                           +
     "color": "white"             +
 }
(1 row)
```

### 2) Checking types with jsonb_typeof

The `jsonb_typeof()` function returns the type of the outermost JSON value as text (`object`, `array`, `string`, `number`, `boolean`, or `null`):

```sql
SELECT
  jsonb_typeof('{"color": "white"}'::JSONB) AS type_obj,
  jsonb_typeof('["S", "M"]'::JSONB) AS type_arr,
  jsonb_typeof('123'::JSONB) AS type_num;
```

Output:

```text
 type_obj | type_arr | type_num
----------+----------+----------
 object   | array    | number
(1 row)
```

### 3) Removing null fields with jsonb_strip_nulls

The `jsonb_strip_nulls()` function deletes all object keys that have a JSON `null` value:

```sql
SELECT jsonb_strip_nulls(
  '{"name": "T-Shirt", "color": "white", "discount": null}'::JSONB
) AS cleaned_data;
```

Output:

```text
             cleaned_data
---------------------------------------
 {"name": "T-Shirt", "color": "white"}
(1 row)
```

## Summary

- JSON stands for JavaScript Object Notation.
- Use `JSONB` to store JSON data for fast indexing and querying.
- Use `to_jsonb()`, `jsonb_build_object()`, and `jsonb_build_array()` to construct JSON data.
- Search JSON data using containment (`@>`, `<@`) and key existence (`?`, `?|`, `?&`) operators.
- Extract nested elements using path operators (`#>`, `#>>`) and `jsonb_path_query()`.
- Modify JSON documents using `||` (merge), `jsonb_set()`, `jsonb_insert()`, and `-` / `#-` (delete).
- Work with arrays using `jsonb_array_length()` and `jsonb_array_elements_text()`.
- Expand objects using `jsonb_object_keys()`, `jsonb_each_text()`, and `jsonb_to_record()`.
- Aggregate rows into JSON with `jsonb_agg()` and `jsonb_object_agg()`.
- Format and inspect JSON using `jsonb_pretty()`, `jsonb_typeof()`, and `jsonb_strip_nulls()`.
