# Data Types in PostgreSQL: A Complete Study Guide

Every column in a database table must declare a **data type**. A data type is a contract between you and the database engine that defines four things:

1. **What values are allowed** (e.g., whole numbers, text, calendar dates, true/false).
2. **How many bytes of memory and disk** each value consumes.
3. **How values are compared and sorted** (e.g., `10 > 2` numerically, whereas `'10' < '2'` alphabetically).
4. **Which operations and functions** can be run on the column (e.g., adding two numbers, extracting the month from a timestamp, or checking if an IP address belongs to a subnet).

Choosing the right data type prevents silent bugs—like rounding errors in financial calculations or broken timezone conversions—keeps storage compact, and allows indexes to work at full speed.

---

## Table of Contents

- [How PostgreSQL Stores Data Types on Disk](#how-postgresql-stores-data-types-on-disk)
- [Numeric Types](#numeric-types)
  - [Integers (`SMALLINT`, `INTEGER`, `BIGINT`)](#integers-smallint-integer-bigint)
  - [Auto-Incrementing Integers: `SERIAL` vs. `IDENTITY` and Sequence Gaps](#auto-incrementing-integers-serial-vs-identity-and-sequence-gaps)
  - [Arbitrary-Precision Numbers (`NUMERIC` / `DECIMAL`)](#arbitrary-precision-numbers-numeric--decimal)
  - [Floating-Point Numbers (`REAL`, `DOUBLE PRECISION`)](#floating-point-numbers-real-double-precision)
- [Character and Text Types (`CHAR`, `VARCHAR`, `TEXT`)](#character-and-text-types-char-varchar-text)
  - [Common String Operations](#common-string-operations)
- [Boolean (`BOOLEAN`)](#boolean-boolean)
- [Date and Time Types (`DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL`)](#date-and-time-types-date-time-timestamp-timestamptz-interval)
  - [Operations on Dates, Timestamps, and Intervals](#operations-on-dates-timestamps-and-intervals)
- [Universally Unique Identifiers (`UUID`)](#universally-unique-identifiers-uuid)
- [Binary Data (`BYTEA`)](#binary-data-bytea)
- [Enumerated Types (`ENUM`)](#enumerated-types-enum)
- [Array Types (`type[]`)](#array-types-type)
  - [Operations on Arrays](#operations-on-arrays)
- [JSON and JSONB (`JSON`, `JSONB`)](#json-and-jsonb-json-jsonb)
  - [Operations on JSONB](#operations-on-jsonb)
- [Range Types (`INT4RANGE`, `DATERANGE`, `TSTZRANGE`)](#range-types-int4range-daterange-tstzrange)
  - [Operations on Ranges](#operations-on-ranges)
- [Network Address Types (`INET`, `CIDR`, `MACADDR`)](#network-address-types-inet-cidr-macaddr)
  - [Operations on Network Types](#operations-on-network-types)
- [Geometric and Vector Types (`POINT`, `POLYGON`, `VECTOR`)](#geometric-and-vector-types-point-polygon-vector)
- [The Special Case of `NULL`](#the-special-case-of-null)
- [Type Casting (`::` and `CAST`) and Custom Domains](#type-casting--and-cast-and-custom-domains)
- [Which Data Type Should You Assign to Which Value?](#which-data-type-should-you-assign-to-which-value)
  - [Decision Flowchart: Picking the Right Data Type](#decision-flowchart-picking-the-right-data-type)
  - [Value-to-Type Reference Table](#value-to-type-reference-table)
  - [Common Traps to Avoid](#common-traps-to-avoid)

---

## How PostgreSQL Stores Data Types on Disk

Before diving into individual types, it helps to understand the three physical storage strategies PostgreSQL uses when writing a row (called a **tuple**) onto an 8 KB disk page:

1. **Fixed-width types** (`SMALLINT`, `INTEGER`, `BIGINT`, `BOOLEAN`, `DATE`, `TIMESTAMPTZ`, `UUID`): Always occupy an exact, predictable number of bytes inline inside the row.
2. **Variable-width inline types (`varlena`)** (`TEXT`, `VARCHAR`, `NUMERIC`, `BYTEA`, `JSONB`, arrays): Store a tiny 1-byte or 4-byte length header followed by the actual data bytes. Short values live right inside the row.
3. **TOAST (The Oversized-Attribute Storage Technique):** PostgreSQL pages are 8 KB (`8,192` bytes) and a single row cannot span across multiple main pages. Whenever a variable-length column makes a row exceed roughly **2 KB**, PostgreSQL automatically **compresses** the value—and if it is still too large, moves the chunks out to a hidden side table called a **TOAST table**, leaving only an 18-byte pointer inside your main row.

#### Diagram: Fixed-Width vs. `varlena` vs. TOAST Storage

```text
Main 8 KB Heap Page (Row Tuple):
┌─────────────┬──────────────┬────────────────────────┬───────────────────────────┐
│ id (BIGINT) │ active (BOOL)│ email (short varlena)  │ bio (long > 2 KB)         │
│   8 bytes   │    1 byte    │ [hdr][ada@example.com] │ [ 18-byte TOAST Pointer ] │
└─────────────┴──────────────┴────────────────────────┴─────────────┬─────────────┘
                                                                    │
                                                                    ▼
                                                Background TOAST Table (Compressed Chunks)
                                                [chunk 0][chunk 1][chunk 2]...
```

Because of **memory alignment** (CPUs read 8-byte values fastest at memory addresses divisible by 8), PostgreSQL may insert invisible padding bytes between columns of mixed sizes. Placing 8-byte fixed columns (`BIGINT`, `TIMESTAMPTZ`, `DOUBLE PRECISION`) first, 4-byte columns (`INTEGER`, `DATE`) next, 1-byte columns (`BOOLEAN`) after that, and variable-length columns (`TEXT`, `JSONB`) last can shrink wide tables by 10–20% with zero code changes.

---

## Numeric Types

Numbers in databases are divided into three distinct families because computers represent whole numbers, exact decimals, and scientific approximations very differently in binary hardware.

### Integers (`SMALLINT`, `INTEGER`, `BIGINT`)

Integers store **exact whole numbers** with no fractional or decimal component. PostgreSQL provides three signed integer sizes:

| Type | Aliases | Storage Size | Minimum Value | Maximum Value |
| :--- | :--- | :--- | :--- | :--- |
| `SMALLINT` | `INT2` | 2 bytes | `-32,768` | `32,767` |
| `INTEGER` | `INT`, `INT4` | 4 bytes | `-2,147,483,648` (~2.14 billion) | `2,147,483,647` |
| `BIGINT` | `INT8` | 8 bytes | `-9,223,372,036,854,775,808` (~9.22 quintillion) | `9,223,372,036,854,775,807` |

If you try to store a number outside a type's range, PostgreSQL immediately rejects the write with an `integer out of range` error rather than silently wrapping around.

Notice also that **PostgreSQL has no unsigned integer types** (like `UNSIGNED INT` in MySQL). All three integer types are signed so that subtracting two integers never causes underflow surprises or ambiguous type-promotion rules. If a column must never be negative (like `age` or `stock_quantity`), enforce it cleanly with `CHECK (stock_quantity >= 0)`.

#### Analogy

> **Analogy:** *Shipping boxes of fixed sizes.*
>
> A `SMALLINT` is a small jewelry box, `INTEGER` is a standard moving box, and `BIGINT` is a freight container. All three hold whole items without cutting anything in half, but trying to force a refrigerator into a jewelry box fails immediately.

#### Diagram: Integer Bit Layout

```text
SMALLINT (2 bytes = 16 bits):
┌───┬─────────────────────────────────────────────────────────┐
│ S │ 15 value bits                                           │  Range: -32,768 to +32,767
└───┴─────────────────────────────────────────────────────────┘
  └── Sign bit (0 = positive, 1 = negative via two's complement)

INTEGER (4 bytes = 32 bits):
┌───┬─────────────────────────────────────────────────────────┐
│ S │ 31 value bits                                           │  Range: ~ -2.14B to +2.14B
└───┴─────────────────────────────────────────────────────────┘

BIGINT (8 bytes = 64 bits):
┌───┬─────────────────────────────────────────────────────────┐
│ S │ 63 value bits                                           │  Range: ~ -9.22Q to +9.22Q
└───┴─────────────────────────────────────────────────────────┘
```

#### Code Example & Integer Operations

```sql
CREATE TABLE products (
    -- BIGINT is safe for primary keys that grow over many years
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- SMALLINT works well for bounded counts like rating 1..5 or retry counts
    warranty_months SMALLINT NOT NULL CHECK (warranty_months >= 0),
    -- INTEGER is plenty for inventory stock in a warehouse
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0)
);

INSERT INTO products (warranty_months, stock_quantity)
VALUES (24, 1500);

-- Arithmetic operations on integers:
SELECT
    7 / 2 AS truncated_quotient,      -- Returns 3 (integer division drops remainder!)
    7 % 2 AS remainder_modulo,        -- Returns 1 (modulo operator)
    7.0 / 2 AS decimal_quotient,      -- Returns 3.5000000000000000 (7.0 is NUMERIC)
    pg_typeof(SUM(stock_quantity))    -- SUM(INTEGER) automatically promotes to BIGINT!
FROM products;
```

Notice two helpful behaviors above:
1. Dividing two integers (`7 / 2`) performs **truncating integer division** (`3`), dropping the `.5` remainder. Cast one side to `NUMERIC` (`7::NUMERIC / 2`) whenever you want a decimal result.
2. When aggregating with `SUM()`, PostgreSQL automatically widens the return type (`SUM(INTEGER)` returns `BIGINT`, and `SUM(BIGINT)` returns `NUMERIC`) so large totals never overflow!

#### When to Use Integers

- **Primary keys and auto-incrementing IDs:** Use `BIGINT` by default. A busy table can exhaust `INTEGER`'s 2.14 billion limit faster than expected, and migrating a primary key from `INT` to `BIGINT` in production requires locking and rewriting the entire table.
- **Counts, quantities, ages, port numbers, HTTP status codes:** Use `INTEGER` or `SMALLINT`.
- **Do not use integers for:** Phone numbers, ZIP/postal codes, or credit card numbers. Those are identifiers, not quantities—leading zeros (`02139`) would be stripped if stored as integers.

### Auto-Incrementing Integers: `SERIAL` vs. `IDENTITY` and Sequence Gaps

In older PostgreSQL tutorials, you will often see `SERIAL` or `BIGSERIAL` used for primary keys. Neither is a true data type—`BIGSERIAL` is a **shorthand macro** that creates a regular `BIGINT` column, creates a background `SEQUENCE` counter, and sets `DEFAULT nextval(...)`.

Modern PostgreSQL (10+) recommends the SQL-standard **`IDENTITY`** syntax instead:
- **`GENERATED ALWAYS AS IDENTITY`:** Prevents accidental manual ID overrides (you can still override explicitly in migrations via `OVERRIDING SYSTEM VALUE`).
- **`GENERATED BY DEFAULT AS IDENTITY`:** Auto-generates an ID if omitted, while still allowing you to supply a custom ID when needed (common during data imports).

```sql
-- Legacy shorthand (avoid in new schemas):
CREATE TABLE legacy_orders (
    id BIGSERIAL PRIMARY KEY
);

-- Modern SQL-standard approach (preferred):
CREATE TABLE modern_orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

> **Beginner Trap — Why Do Auto-Incrementing IDs Have Gaps (`1, 2, 4`)?**
>
> A `SEQUENCE` generates numbers **outside** transaction rollback rules so concurrent writers never block each other waiting for the next ID. If Transaction #3 grabs `id = 3` and later rolls back (e.g., due to a failed validation or `ON CONFLICT DO NOTHING`), `3` is never reused—the next row gets `4`. **Gaps in `id` columns are completely normal and expected.** Never assume `MAX(id)` equals the row count, and never try to "fill" missing IDs.

---

### Arbitrary-Precision Numbers (`NUMERIC` / `DECIMAL`)

`NUMERIC` and `DECIMAL` are identical in PostgreSQL. They store numbers with **exact decimal precision**, meaning `0.1 + 0.2` is guaranteed to equal `0.3` down to the last digit. Unlike integers, `NUMERIC` is a **variable-size** type (taking roughly 2 bytes for every 4 decimal digits stored, plus a small header).

You can declare `NUMERIC` in three ways:
- **`NUMERIC(precision, scale)`:** Fixed total digits (`precision`) and digits after the decimal point (`scale`). For example, `NUMERIC(10, 2)` holds up to 10 digits total with 2 after the decimal point (max `99,999,999.99`).
- **`NUMERIC(precision)`:** Equivalent to `NUMERIC(precision, 0)`—stores exact whole numbers larger than `BIGINT`.
- **Unconstrained `NUMERIC`:** Stores values with whatever precision and scale you insert (up to ~131,072 digits before the decimal and ~16,383 after).

#### Analogy

> **Analogy:** *An accountant's paper ledger.*
>
> Every digit is written down explicitly in base-10 columns—dollars and cents never get rounded into binary fractions behind your back. It takes a little more effort to write and calculate than mental estimation, but the books always balance to the exact cent.

#### Diagram: `NUMERIC(6, 2)` Digit Allocation

```text
NUMERIC(6, 2)  ──>  6 total digits, 2 after the decimal point

  [ 9 ] [ 9 ] [ 9 ] [ 9 ]  .  [ 9 ] [ 9 ]
  └─────────┬───────────┘     └────┬────┘
     4 integer digits        2 scale digits
     (precision - scale)        (scale)

  1234.56   -> Valid
  1234.567  -> Rounded to 1234.57 (scale exceeded, rounded automatically)
  12345.67  -> ERROR: numeric field overflow (integer digits > 4)
```

#### Code Example & Rounding Behavior

```sql
CREATE TABLE invoices (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    subtotal NUMERIC(12, 2) NOT NULL,
    tax_rate NUMERIC(5, 4) NOT NULL, -- e.g., 0.0825 for 8.25%
    total NUMERIC(12, 2) GENERATED ALWAYS AS (ROUND(subtotal * (1 + tax_rate), 2)) STORED
);

INSERT INTO invoices (subtotal, tax_rate)
VALUES (19.99, 0.0825);

SELECT subtotal, tax_rate, total FROM invoices;
-- subtotal: 19.99 | tax_rate: 0.0825 | total: 21.64

-- Useful numeric functions & subtle rounding difference:
SELECT
    ROUND(2.5::NUMERIC) AS numeric_round,             -- 3 (half away from zero)
    ROUND(2.5::DOUBLE PRECISION) AS float_round,      -- 2 (IEEE 754 round-to-even!)
    TRUNC(19.999::NUMERIC, 2) AS truncated,           -- 19.99 (chops digits without rounding)
    CEIL(4.1::NUMERIC) AS ceiling_val,                -- 5
    FLOOR(4.9::NUMERIC) AS floor_val;                 -- 4
```

#### When to Use `NUMERIC` (and Why to Avoid the Legacy `MONEY` Type)

- **Money, prices, salaries, tax rates, financial balances:** Always use `NUMERIC` (with a separate `currency_code TEXT` column like `'USD'`)—or store exact minor units (cents) as `BIGINT` when extreme arithmetic throughput is needed.
- **Avoid PostgreSQL's legacy `MONEY` type:** `MONEY` formats values using the server's `lc_monetary` locale setting. Changing locale settings or dumping/restoring across servers with different locales can silently misinterpret `$100` vs `€100` or fail to parse!
- **Trade-off:** Arithmetic on `NUMERIC` is performed in software rather than directly in the CPU's floating-point unit, making it noticeably slower than `INTEGER` or `DOUBLE PRECISION` for millions of heavy math operations.

---

### Floating-Point Numbers (`REAL`, `DOUBLE PRECISION`)

`REAL` (`FLOAT4`, 4 bytes, ~6 decimal digits of precision) and `DOUBLE PRECISION` (`FLOAT8`, 8 bytes, ~15 decimal digits of precision) implement IEEE 754 binary floating-point arithmetic. They represent numbers in **base-2 scientific notation** (sign, exponent, mantissa), which means certain base-10 decimals like `0.1` cannot be represented exactly—they repeat infinitely in binary, just like `1/3` repeats as `0.3333...` in base 10.

#### Analogy

> **Analogy:** *A scientific slide rule or measuring tape.*
>
> You can measure across huge ranges—from the width of a hair to the distance between galaxies—very quickly. However, you are always working with an approximation; the last few digits may wobble slightly.

#### Diagram: Why `0.1 + 0.2 != 0.3` in Floating Point

```text
Base-10:   0.1  +  0.2  =  0.3 (exact)

Base-2 (IEEE 754 DOUBLE PRECISION):
  0.1  ──>  0.1000000000000000055511151231257827021181583404541015625
+ 0.2  ──>  0.2000000000000000111022302462515654042363166809082031250
─────────────────────────────────────────────────────────────────────────
  Sum  ──>  0.30000000000000004440892098500626161694526672363281250  (!= 0.3)
```

#### Code Example & Floating-Point Quirks

```sql
-- Demonstrating floating-point inaccuracy vs NUMERIC exactness:
SELECT
    (0.1::DOUBLE PRECISION + 0.2::DOUBLE PRECISION) = 0.3::DOUBLE PRECISION AS float_equal,
    (0.1::NUMERIC + 0.2::NUMERIC) = 0.3::NUMERIC AS numeric_equal;
-- float_equal: false | numeric_equal: true

-- Instead of exact equality (=) on floats, compare within a tiny tolerance (epsilon):
SELECT ABS((0.1::DOUBLE PRECISION + 0.2::DOUBLE PRECISION) - 0.3::DOUBLE PRECISION) < 1e-9 AS close_enough;
-- close_enough: true

-- Special floating-point values supported by REAL and DOUBLE PRECISION:
SELECT
    'Infinity'::DOUBLE PRECISION > 1e300 AS inf_larger,          -- true
    'NaN'::DOUBLE PRECISION = 'NaN'::DOUBLE PRECISION AS nan_eq, -- true in PostgreSQL!
    'NaN'::DOUBLE PRECISION > 'Infinity'::DOUBLE PRECISION;      -- true (sorts highest)
```

Notice a subtle PostgreSQL rule: unlike Python or JavaScript (where `NaN == NaN` is `false`), PostgreSQL defines **`NaN = NaN` as `TRUE`** and sorts `NaN` higher than `Infinity` so that `GROUP BY`, `DISTINCT`, `ORDER BY`, and B-Tree indexes remain deterministic.

#### When to Use Floating-Point Numbers

- **Sensor readings, temperatures, physics simulations, GPS coordinates, ML weights:** Data that is inherently approximate and benefits from hardware-accelerated math.
- **Never use `REAL` or `DOUBLE PRECISION` for money or exact equality comparisons (`WHERE price = 19.99`).**

---

## Character and Text Types (`CHAR`, `VARCHAR`, `TEXT`)

PostgreSQL has three main types for storing strings of Unicode characters (UTF-8):

1. **`CHAR(n)` / `CHARACTER(n)`:** Fixed-length string. If you insert `'US'` into a `CHAR(5)` column, PostgreSQL silently pads it with 3 trailing spaces (`'US   '`).
2. **`VARCHAR(n)` / `CHARACTER VARYING(n)`:** Variable-length string with a hard maximum length of `n` characters. No padding is added; strings longer than `n` raise an error.
3. **`TEXT`:** Variable-length string with no declared character limit (up to ~1 GB).

A surprising fact about PostgreSQL: under the hood, `CHAR(n)`, `VARCHAR(n)`, and `TEXT` all use the exact same internal `varlena` (variable-length) storage engine. There is **no performance advantage** to using `VARCHAR(255)` over `TEXT`.

#### Analogy

> **Analogy:** *Containers for written notes.*
>
> - `CHAR(10)` is a row of 10 letter tiles on a board—if your word is 4 letters long, you must fill the remaining 6 slots with blank tiles.
> - `VARCHAR(10)` is a sticky note that shrinks to fit your word, but refuses to let you write an 11th letter.
> - `TEXT` is a scroll of paper that unrolls only as far as your message needs.

#### Diagram: Characters vs. Bytes in UTF-8 & Storage Layout

```text
Value inserted: 'cat' (3 ASCII chars = 3 bytes)
  CHAR(6):     [ header ][ c ][ a ][ t ][   ][   ][   ]   (padded to 6 chars)
  VARCHAR(6):  [ header ][ c ][ a ][ t ]                   (3 chars stored, max 6 enforced)
  TEXT:        [ header ][ c ][ a ][ t ]                   (3 chars stored, no artificial cap)

Multi-byte UTF-8 character: '🚀'
  char_length('🚀')  = 1 character   (VARCHAR(1) ALLOWS '🚀'!)
  octet_length('🚀') = 4 bytes       (1 Unicode emoji takes 4 bytes on disk)
```

Notice that `VARCHAR(n)` limits **characters**, not bytes! Because PostgreSQL databases normally use UTF-8 encoding, English ASCII letters take 1 byte each, accented letters like `'é'` take 2 bytes, and emojis like `'🚀'` take 4 bytes.

Another important intermediate detail is **collation and case sensitivity**: unlike MySQL (which often defaults to case-insensitive equality), PostgreSQL string equality (`=`) is **case-sensitive** by default (`'Ada' = 'ada'` is `FALSE`). For case-insensitive columns like user emails, store normalized lowercase strings (`LOWER(email)`), create a functional index (`CREATE INDEX idx_users_lower_email ON users (LOWER(email));`), or use a case-insensitive ICU collation / `CITEXT` extension.

#### Code Example

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- ISO country codes are always 2 letters, though TEXT + CHECK is equally fast
    country_code CHAR(2) NOT NULL,
    -- Enforcing length via CHECK constraint on TEXT is easy to alter later
    username TEXT NOT NULL CHECK (char_length(username) BETWEEN 3 AND 30),
    email TEXT NOT NULL,
    bio TEXT
);

INSERT INTO users (country_code, username, email, bio)
VALUES ('US', 'ada_lovelace', 'Ada@Example.COM', 'First computer programmer.');

-- Watch out for CHAR(n) trailing spaces and byte vs. character length:
SELECT
    'A'::CHAR(4) = 'A'::TEXT AS char_equals_text,          -- true (trailing spaces ignored in =)
    char_length('A'::CHAR(4)) AS char_length_logical,      -- 1 (CHAR strips trailing spaces)
    octet_length('A'::CHAR(4)) AS char_bytes_on_disk,      -- 4 (actual padded bytes)
    char_length('🚀') AS emoji_chars,                      -- 1
    octet_length('🚀') AS emoji_bytes;                     -- 4
```

#### When to Use Which

- **Default choice:** Use `TEXT` for almost all string columns (names, emails, URLs, descriptions). If you need a length limit for business rules, add `CHECK (char_length(col) <= 255)`—unlike `VARCHAR(255)`, changing a `CHECK` constraint later does not require rewriting or heavily locking the table.
- **`VARCHAR(n)`:** Fine when integrating with ORMs or schemas that expect explicit column lengths, or when a rigid limit is part of a standard.
- **Avoid `CHAR(n)`:** The silent space-padding causes subtle bugs in application code and offers zero speed or space benefits in PostgreSQL.

### Common String Operations

```sql
SELECT
    -- || concatenates strings, but returns NULL if ANY operand is NULL:
    'Hello ' || NULL AS plus_null,                              -- NULL
    -- CONCAT() safely treats NULL as an empty string:
    CONCAT('Hello ', NULL, 'World') AS safe_concat,             -- 'Hello World'
    -- Case normalization and trimming specific characters or whitespace:
    LOWER(TRIM('  Ada@Example.COM  ')) AS clean_email,          -- 'ada@example.com'
    -- 1-based slicing and prefix/suffix helpers:
    SUBSTRING('PostgreSQL' FROM 1 FOR 8) AS sub_str,            -- 'PostgreS'
    LEFT('PostgreSQL', 4) AS prefix_4,                          -- 'Post'
    -- Splitting and replacing:
    SPLIT_PART('ada@example.com', '@', 2) AS domain,            -- 'example.com'
    REPLACE('2026-11-28', '-', '/') AS slash_date,              -- '2026/11/28'
    -- Pattern matching: LIKE (% = 0+ chars, _ = 1 char), ILIKE (case-insensitive), ~ (regex)
    'PostgreSQL' ILIKE 'post%' AS ilike_match,                  -- true
    'user_42' ~ '^user_[0-9]+$' AS regex_match;                 -- true

-- Combine multiple rows into one comma-separated string:
SELECT STRING_AGG(username, ', ' ORDER BY username) AS all_users
FROM users;
```

---

## Boolean (`BOOLEAN`)

`BOOLEAN` (1 byte) stores logical truth values. However, unlike booleans in most programming languages (which only have `true` and `false`), SQL databases use **three-valued logic**:

- `TRUE` (also accepts `'t'`, `'true'`, `'yes'`, `'y'`, `'1'`)
- `FALSE` (also accepts `'f'`, `'false'`, `'no'`, `'n'`, `'0'`)
- `NULL` (meaning *unknown* or *not provided*)

#### Analogy

> **Analogy:** *A survey checkbox with three states.*
>
> - Checked = `TRUE` ("Yes, I agree")
> - Unchecked = `FALSE` ("No, I decline")
> - Torn off / left blank = `NULL` ("We don't know what the person answered")
>
> If you ask "Did Person A and Person B give the same answer?" when both left it blank, the answer isn't "Yes"—it's "We don't know" (`NULL`).

#### Diagram: Three-Valued Logic Truth Table

```text
┌─────────┬─────────┬─────────────┬─────────────┬─────────────────┐
│    A    │    B    │   A AND B   │   A OR B    │  A = B          │
├─────────┼─────────┼─────────────┼─────────────┼─────────────────┤
│  TRUE   │  TRUE   │    TRUE     │    TRUE     │  TRUE           │
│  TRUE   │  FALSE  │    FALSE    │    TRUE     │  FALSE          │
│  TRUE   │  NULL   │    NULL     │    TRUE     │  NULL (Unknown) │
│  FALSE  │  NULL   │    FALSE    │    NULL     │  NULL (Unknown) │
│  NULL   │  NULL   │    NULL     │    NULL     │  NULL (Unknown) │
└─────────┴─────────┴─────────────┴─────────────┴─────────────────┘
```

#### Code Example

```sql
CREATE TABLE feature_flags (
    flag_name TEXT PRIMARY KEY,
    is_enabled BOOLEAN -- Nullable!
);

INSERT INTO feature_flags VALUES
    ('dark_mode', TRUE),
    ('beta_checkout', FALSE),
    ('legacy_ui', NULL);

-- WRONG way to find flags that are NOT enabled (misses NULL rows!):
SELECT flag_name FROM feature_flags WHERE is_enabled = FALSE;
-- Returns only 'beta_checkout'

-- RIGHT way to treat NULL and FALSE together, or explicitly check NULL:
SELECT flag_name FROM feature_flags WHERE is_enabled IS NOT TRUE;
-- Returns 'beta_checkout' and 'legacy_ui'

-- Direct boolean filtering & sorting (FALSE < TRUE < NULL in ascending order):
SELECT flag_name FROM feature_flags WHERE is_enabled; -- Shorthand for is_enabled = TRUE
SELECT flag_name, is_enabled FROM feature_flags ORDER BY is_enabled DESC NULLS LAST;

-- Boolean aggregate checks across groups:
SELECT
    BOOL_OR(is_enabled) AS any_flag_enabled,           -- true if at least one row is TRUE
    BOOL_AND(is_enabled) AS all_flags_enabled,         -- false unless every non-NULL row is TRUE
    COUNT(*) FILTER (WHERE is_enabled) AS enabled_count
FROM feature_flags;

-- Comparing two nullable booleans safely:
SELECT TRUE IS DISTINCT FROM NULL;   -- true
SELECT NULL IS DISTINCT FROM NULL;   -- false
```

> **Classic Beginner-to-Intermediate SQL Trap — `NOT IN` with `NULL`:**
>
> Because of three-valued logic, `WHERE id NOT IN (1, 2, NULL)` evaluates to `(id != 1) AND (id != 2) AND (id != NULL)`. Since `id != NULL` is always `NULL`, the whole `AND` expression can never become `TRUE`, and the query silently returns **zero rows**! Always filter out `NULL`s in subqueries inside `NOT IN`, or better yet, use `NOT EXISTS`.

#### When to Use `BOOLEAN`

- **Binary states:** `is_active`, `is_verified`, `has_paid`, `email_opt_in`.
- **Best practice:** Add `NOT NULL DEFAULT FALSE` (or `TRUE`) unless "unknown" is genuinely meaningful in your domain. Avoiding nullable booleans eliminates three-valued logic surprises.

---

## Date and Time Types (`DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL`)

Handling dates and times is one of the most error-prone areas in software engineering because of time zones, Daylight Saving Time (DST), leap years, and varying month lengths. PostgreSQL provides five dedicated types so you can model each temporal concept accurately:

| Type | Storage | What It Represents | Time Zone Aware? |
| :--- | :--- | :--- | :--- |
| `DATE` | 4 bytes | Calendar date (`2026-11-28`) | No |
| `TIME` | 8 bytes | Wall-clock time of day (`14:30:00`) | No |
| `TIMESTAMP` | 8 bytes | Date + time, no time zone (`2026-11-28 14:30:00`) | No |
| `TIMESTAMPTZ` | 8 bytes | Exact instant in time (`TIMESTAMP WITH TIME ZONE`) | Yes (stored as UTC) |
| `INTERVAL` | 16 bytes | A span or duration of time (`3 days 04:00:00`) | N/A |

A crucial detail about `TIMESTAMPTZ`: **it does not store the time zone name inside the column.** Instead, PostgreSQL converts whatever time zone you pass in into **UTC** for storage, and converts it back to your connection's local `TimeZone` setting when you query it. Always write date literals in unambiguous **ISO 8601 format** (`'YYYY-MM-DD'`, e.g., `'2026-11-28'`) rather than `'11/28/2026'`, which breaks if a server uses `DMY` instead of `MDY` `DateStyle`.

Also note why `TIMETZ` (`TIME WITH TIME ZONE`) is omitted from most modern schemas: a time of day *without a date* cannot know whether Daylight Saving Time is active, making `TIMETZ` ambiguous and best avoided in favor of `TIMESTAMPTZ` (or plain `TIME`).

#### Analogy

> **Analogy:** *Calendar pages, wristwatches, and stopwatches.*
>
> - `DATE` is a square on a wall calendar ("November 28").
> - `TIME` is a wall clock ("2:30 PM") without knowing which day or country you're in.
> - `TIMESTAMP` is a photo of a calendar and a wall clock ("Nov 28, 2:30 PM"), but you don't know if the photo was taken in Tokyo or New York.
> - `TIMESTAMPTZ` is a **universal cosmic instant** (e.g., the exact second a rocket launches)—everyone on Earth agrees on *when* it happened, even if New York calls it 9:30 AM and London calls it 2:30 PM.
> - `INTERVAL` is a **stopwatch + calendar duration** ("1 month and 2 hours"), not a point on the timeline.

#### Diagram: How `TIMESTAMPTZ` Works & `INTERVAL` Internals

```text
1. TIMESTAMPTZ Storage & Display:
Client in New York (EST, UTC-5) inserts: '2026-11-28 09:30:00-05'
           │
           ▼  (Converted to UTC on write)
┌─────────────────────────────────────────┐
│  Disk Storage (8 bytes, always UTC):    │
│       2026-11-28 14:30:00 UTC           │
└─────────────────────────────────────────┘
           │
           ├──► Read by New York client (TimeZone = 'America/New_York')
           │      Output: 2026-11-28 09:30:00-05
           │
           └──► Read by Tokyo client (TimeZone = 'Asia/Tokyo')
                  Output: 2026-11-28 23:30:00+09

2. Why INTERVAL Is 16 Bytes (Three Separate Parts):
┌───────────────────┬─────────────────┬─────────────────────────┐
│  months (4 bytes) │  days (4 bytes) │ microseconds (8 bytes)  │
└───────────────────┴─────────────────┴─────────────────────────┘
  Why separate? Because '1 month' != '30 days' and '1 day' != '24 hours' across DST!
```

Look closely at the `INTERVAL` diagram: storing months, days, and microseconds separately is why adding `INTERVAL '1 day'` across a Daylight Saving Time "spring forward" night keeps the same wall-clock time (`09:00` -> `09:00` next day, 23 real hours later), whereas adding `INTERVAL '24 hours'` adds exactly 24 physical hours (`09:00` -> `10:00` next day).

#### Code Example: Creating Temporal Columns

```sql
CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    -- Exact moment the event starts (always use TIMESTAMPTZ for instants)
    starts_at TIMESTAMPTZ NOT NULL,
    -- How long the event lasts
    duration INTERVAL NOT NULL,
    -- A person's birthday is a calendar date, not an instant in UTC
    organizer_birthday DATE
);

INSERT INTO events (title, starts_at, duration, organizer_birthday)
VALUES (
    'Database Workshop',
    '2026-11-28 14:00:00+00',
    INTERVAL '2 hours 30 minutes',
    '1995-04-12'
);
```

### Operations on Dates, Timestamps, and Intervals

Temporal types support rich arithmetic, truncation, extraction, and time-zone conversion operations.

#### 1. Date and Time Arithmetic (`+`, `-`) and Safe Range Filtering

Subtracting two timestamps produces an `INTERVAL`. Adding an `INTERVAL` to a timestamp produces a new timestamp.

```sql
SELECT
    starts_at,
    starts_at + duration AS ends_at,
    -- Time remaining until the event (returns an INTERVAL)
    starts_at - NOW() AS time_until_start,
    -- Adding 1 month respects calendar month lengths (Jan 31 + 1 month = Feb 28)
    DATE '2026-01-31' + INTERVAL '1 month' AS next_month
FROM events;
```

> **The Golden Rule for Querying Date Ranges (Avoid `BETWEEN` and `DATE(col)`!):**
>
> Beginners often write `WHERE starts_at BETWEEN '2026-11-01' AND '2026-11-28'`. That has a silent bug: `'2026-11-28'` casts to `2026-11-28 00:00:00`, so **every event on November 28 after midnight is excluded!** Meanwhile, writing `WHERE starts_at::DATE = '2026-11-28'` prevents PostgreSQL from using a normal B-Tree index on `starts_at`.
>
> Always use a **half-open range** (`>=` and `<`):
> ```sql
> -- Finds ALL events on Nov 28, 2026 (UTC) AND uses a B-Tree index on starts_at!
> SELECT title FROM events
> WHERE starts_at >= TIMESTAMPTZ '2026-11-28 00:00:00+00'
>   AND starts_at <  TIMESTAMPTZ '2026-11-29 00:00:00+00';
> ```

#### 2. Converting Time Zones with `AT TIME ZONE`

`AT TIME ZONE` has two opposite behaviors depending on the input type:
- On a **`TIMESTAMPTZ`**, it converts the UTC instant into a wall-clock `TIMESTAMP` (without timezone) in the target zone.
- On a **`TIMESTAMP`**, it attaches the target zone to that wall-clock time and returns a `TIMESTAMPTZ` (UTC instant).

```sql
SELECT
    -- Convert stored UTC instant to wall-clock time in Tokyo:
    starts_at AT TIME ZONE 'Asia/Tokyo' AS tokyo_wall_time,
    -- Interpret a naive timestamp as New York time and get the UTC TIMESTAMPTZ:
    TIMESTAMP '2026-11-28 09:00:00' AT TIME ZONE 'America/New_York' AS ny_instant;
```

#### 3. Bucketing with `DATE_TRUNC` and `DATE_BIN` (Essential for Analytics)

- **`DATE_TRUNC('unit', timestamp)`** zeroes out all smaller fields below `'unit'`, snapping timestamps to the start of the hour, day, week, month, or year.
- **`DATE_BIN(stride, timestamp, origin)`** (PostgreSQL 14+) buckets timestamps into arbitrary intervals like 15-minute or 5-minute windows—great for monitoring dashboards.

```sql
-- Count events grouped by month
SELECT
    DATE_TRUNC('month', starts_at) AS event_month,
    COUNT(*) AS total_events
FROM events
GROUP BY event_month
ORDER BY event_month;

-- Snap a timestamp to 15-minute buckets:
SELECT DATE_BIN(INTERVAL '15 minutes', TIMESTAMPTZ '2026-11-28 14:37:52+00', TIMESTAMPTZ '2026-11-28 00:00:00+00');
-- Returns: '2026-11-28 14:30:00+00'
```

#### 4. Extracting Parts (`EXTRACT`), `AGE`, and Unix Epoch Conversion

`EXTRACT(field FROM source)` pulls out a single numeric component (year, month, day of week, Unix epoch seconds). `AGE(t1, t2)` calculates a human-friendly symbolic interval in years, months, and days. `TO_TIMESTAMP(epoch_seconds)` converts a numeric Unix timestamp back into a `TIMESTAMPTZ`.

```sql
SELECT
    EXTRACT(YEAR FROM starts_at) AS event_year,
    -- DOW: 0 = Sunday, 6 = Saturday; ISODOW: 1 = Monday, 7 = Sunday
    EXTRACT(ISODOW FROM starts_at) AS iso_weekday,
    -- Convert TIMESTAMPTZ to Unix epoch seconds and back:
    EXTRACT(EPOCH FROM starts_at) AS unix_seconds,
    TO_TIMESTAMP(1795874400) AS back_from_epoch,
    -- Human-readable age from birthday:
    AGE(CURRENT_DATE, organizer_birthday) AS organizer_age
FROM events;
```

#### 5. Formatting (`TO_CHAR`), Parsing (`TO_TIMESTAMP`), and Generating Date Series

- **`TO_CHAR(timestamp, format)`** formats a timestamp into a human-readable string.
- **`TO_TIMESTAMP(text, format)`** parses custom date strings (like `'28/11/2026 14:30'`) into a `TIMESTAMPTZ`.
- **`NOW()` vs. `CLOCK_TIMESTAMP()`:** `NOW()` (identical to `CURRENT_TIMESTAMP`) returns the timestamp when the current transaction *started* (so multiple inserts in one transaction share the same timestamp). `CLOCK_TIMESTAMP()` returns the actual wall-clock time at each call.
- **`GENERATE_SERIES(start, stop, step)`** generates a continuous sequence of dates or timestamps—essential for filling in days with zero events in reports.

```sql
SELECT
    TO_CHAR(starts_at, 'YYYY-MM-DD HH24:MI TZ') AS formatted_date,
    TO_TIMESTAMP('28/11/2026 14:30', 'DD/MM/YYYY HH24:MI') AS parsed_ts,
    NOW() AS txn_start_time,
    CLOCK_TIMESTAMP() AS live_wall_time
FROM events;

-- Generate every day in November 2026 (great for LEFT JOINing daily metrics):
SELECT day::DATE
FROM generate_series(
    DATE '2026-11-01',
    DATE '2026-11-30',
    INTERVAL '1 day'
) AS day;
```

#### When to Use Temporal Types

- **`TIMESTAMPTZ`:** Use for **99% of timestamps** (`created_at`, `updated_at`, `order_placed_at`, `login_at`, scheduled meetings).
- **`TIMESTAMP` (without time zone):** Only use for abstract local wall-clock times where time zone is irrelevant or stored in a separate column (e.g., "store opens at 09:00 local time everywhere").
- **`DATE`:** Use for birthdays, holidays, invoice dates, and calendar days.
- **`INTERVAL`:** Use for durations, timeouts, SLA windows, and subscription lengths.

---

## Universally Unique Identifiers (`UUID`)

A `UUID` is a **128-bit (16-byte)** identifier formatted as 32 hexadecimal digits separated by hyphens, like `550e8400-e29b-41d4-a716-446655440000`. PostgreSQL stores it natively as 16 raw bytes—not as a 36-character `TEXT` string (which takes 37+ bytes and does not validate format)—making it compact and fast to compare.

PostgreSQL accepts UUIDs with uppercase, lowercase, braces, or omitted hyphens (`'550E8400E29B41D4A716446655440000'`), validates the structure on write, and always outputs canonical lowercase hyphenated form.

#### Analogy

> **Analogy:** *A pre-printed serial number generated from a cosmic lottery.*
>
> Sequential integer IDs are like a deli counter ticket machine: everyone must line up at one machine to get the next number (`101`, `102`, `103`). A UUID is like rolling a 38-digit die at your own desk—the odds of two servers rolling the same number are so astronomically small that you never need to coordinate.

#### Diagram: UUIDv4 (Random) vs. UUIDv7 (Time-Ordered)

```text
UUIDv4 (Completely Random - gen_random_uuid()):
  [ random ][ random ][ ver=4 ][ random ]
  ──> Inserts scatter randomly across B-Tree index pages (more disk I/O at scale)

UUIDv7 (Time-Ordered Prefix + Random Suffix - uuidv7() in PG 18+):
  [ 48-bit Unix ms timestamp ][ ver=7 ][ random bits ]
  ──> Inserts append sequentially at the right edge of the B-Tree (cache-friendly!)
```

#### UUID Versions at a Glance

- **UUIDv4:** Completely random (`gen_random_uuid()`, built into PostgreSQL 13+ with no extension needed). Ideal for opaque public IDs that leak zero creation-time info.
- **UUIDv5:** Deterministic SHA-1 hash of a namespace + name (`uuid_generate_v5` via `uuid-ossp`). Always produces the same UUID for the same input—great for idempotency keys.
- **UUIDv7:** Combines a millisecond timestamp prefix with random bits (`uuidv7()` natively in PostgreSQL 18+, or via SQL helper/extension in older versions). Gives you distributed uniqueness **and** sequential B-Tree insert performance!

#### Code Example

```sql
CREATE TABLE api_keys (
    -- gen_random_uuid() is built into PostgreSQL 13+ (generates UUIDv4)
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    label TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO api_keys (label) VALUES ('production-backend');

-- PostgreSQL normalizes uppercase/unhyphenated inputs automatically:
SELECT '550E8400E29B41D4A716446655440000'::UUID;
-- Returns: 550e8400-e29b-41d4-a716-446655440000
```

#### When to Use `UUID`

- **Public-facing IDs in URLs and APIs:** Prevents users from guessing `/api/orders/1001`, `/api/orders/1002` or inferring how many customers you have.
- **Distributed systems / client-generated IDs:** When mobile apps, microservices, or offline clients must generate IDs before talking to the central database.
- **Trade-off:** `UUID` (16 bytes) is twice the size of `BIGINT` (8 bytes), and random UUIDv4 primary keys can cause B-Tree index fragmentation on high-write tables (prefer UUIDv7 for high-throughput append tables, or keep `BIGINT` internally and expose a separate `UUID` `public_id`).

---

## Binary Data (`BYTEA`)

`BYTEA` ("byte array") stores raw, uninterpreted sequences of bytes—anything from cryptographic hashes and encryption keys to small binary blobs. Like `TEXT`, `BYTEA` is a `varlena` type that automatically triggers **TOAST compression and out-of-line storage** once a value exceeds ~2 KB.

#### Analogy

> **Analogy:** *A sealed opaque envelope.*
>
> Unlike `TEXT`, where the database checks that the contents are valid UTF-8 characters and knows how to sort them alphabetically, `BYTEA` doesn't care what's inside. It just holds the exact raw bytes you handed it and gives them back untouched.

#### Diagram: `TEXT` vs. `BYTEA`

```text
TEXT:   Input bytes ──> Validated as UTF-8 ──> Sorted by language collation rules
BYTEA:  Input bytes ──> Stored verbatim    ──> Sorted strictly byte-by-byte (0x00..0xFF)
```

#### Code Example & Binary Operations

```sql
CREATE TABLE file_checksums (
    file_id BIGINT PRIMARY KEY,
    -- Raw 32-byte SHA-256 hash takes 32 bytes in BYTEA vs 64 bytes as hex TEXT
    sha256_digest BYTEA NOT NULL CHECK (octet_length(sha256_digest) = 32)
);

-- Insert using PostgreSQL hex escape format (\x...), or built-in sha256() (PG 11+):
INSERT INTO file_checksums (file_id, sha256_digest)
VALUES
    (1, '\x2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824'),
    (2, sha256(convert_to('hello world', 'UTF8')));

-- Encode/decode between BYTEA, hex, base64, and UTF-8 TEXT:
SELECT
    file_id,
    encode(sha256_digest, 'hex') AS hex_digest,
    encode(sha256_digest, 'base64') AS base64_digest,
    get_byte(sha256_digest, 0) AS first_byte_0_to_255,       -- 0-based byte inspection!
    convert_from(decode('68656c6c6f', 'hex'), 'UTF8') AS text_back -- 'hello'
FROM file_checksums;
```

#### When to Use `BYTEA`

- **Cryptographic hashes, salts, binary tokens, small thumbnails, or serialized protobufs** (typically a few bytes to a few kilobytes).
- **Do NOT use `BYTEA` for large files** (videos, PDFs, high-res images). Store large files in object storage (like Amazon S3 or Google Cloud Storage) and store only the file's `TEXT` URL/key in PostgreSQL.

---

## Enumerated Types (`ENUM`)

An `ENUM` is a custom data type you define with a static, ordered list of allowed string labels. Internally, PostgreSQL stores each label as a compact **4-byte integer**, while letting you write and read human-readable strings in SQL.

#### Analogy

> **Analogy:** *A printed multiple-choice dropdown menu.*
>
> You can only pick one of the exact options printed on the form (`'pending'`, `'paid'`, `'shipped'`). You cannot scribble `'maybe_paid'` in the margin, and the options have a built-in order from top to bottom.

#### Diagram: `ENUM` Internal Mapping

```text
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');

  Label (What you see)     Internal ID (4 bytes on disk)
  ────────────────────     ─────────────────────────────
  'pending'       <─────>  1  (earliest in sort order)
  'paid'          <─────>  2
  'shipped'       <─────>  3
  'cancelled'     <─────>  4  (latest in sort order)
```

#### Code Example

```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');

CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);

INSERT INTO orders (status) VALUES ('shipped'), ('pending');

-- Sorting orders by ENUM definition order (NOT alphabetical!):
SELECT id, status FROM orders ORDER BY status ASC;
-- Returns: 'pending' first, then 'shipped'

-- Adding a new value later is easy (note the transaction rule below!):
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'shipped';

-- Inspect all allowed labels of an ENUM at runtime:
SELECT enum_range(NULL::order_status);
-- Returns: {pending,paid,shipped,refunded,cancelled}
```

> **Intermediate Gotcha — `ALTER TYPE ... ADD VALUE` Inside Migrations:**
>
> You **cannot** run `ALTER TYPE ... ADD VALUE 'refunded'` and immediately `INSERT` or `UPDATE` a row using `'refunded'` inside the **same transaction**. PostgreSQL throws `unsafe use of new value "refunded" of enum type`. Always commit the `ALTER TYPE` migration step before using the new label!

#### Comparing Modeling Options for Allowed Values

| Approach | Pros | Cons | Best For |
| :--- | :--- | :--- | :--- |
| **`ENUM` Type** | Compact (4 bytes); self-documenting; custom sort order | Hard to remove/rename values; transaction migration quirk | Fixed constants (`log_level`, `day_of_week`) |
| **`TEXT` + `CHECK`** | Easy to alter constraints; standard SQL; no type coupling | Slightly larger than 4 bytes; alphabetical sort only | Evolving app statuses (`'trial'`, `'active'`, `'paused'`) |
| **Lookup Table + `FK`** | Attach metadata (display name, color); instant edits via `INSERT`/`DELETE` | Requires a `JOIN` to read labels | User-editable categories, countries, roles |

#### When to Use `ENUM`

- **Truly static, rarely changing value sets** like `days_of_week`, `card_suit`, or `log_severity` (`debug`, `info`, `warn`, `error`).
- For business statuses that evolve frequently over product iterations, prefer `TEXT` with a `CHECK` constraint or a foreign-key lookup table.

---

## Array Types (`type[]`)

PostgreSQL allows **any** built-in or custom data type to become an array by appending `[]` (e.g., `TEXT[]`, `INTEGER[]`, `UUID[]`, even 2D matrices like `INTEGER[][]`). Every element in an array must share the **same data type**, and unlike most programming languages (which use 0-based indexing), **PostgreSQL arrays are 1-based by default**. Accessing an out-of-bounds index (like `tags[99]`) returns `NULL` rather than throwing an exception.

#### Analogy

> **Analogy:** *A labeled pill organizer inside a single table cell.*
>
> Instead of creating a separate child table just to store three tags for a blog post, you keep a small ordered tray right inside the `tags` column.

#### Diagram: 1-Based Array Indexing

```text
Column: tags TEXT[] = '{"postgres", "sql", "indexing"}'

  Index:    [ 1 ]        [ 2 ]       [ 3 ]        [ 99 ] (out of bounds)
          ┌──────────┬──────────┬────────────┐
  Value:  │ postgres │   sql    │  indexing  │     ──> returns NULL (no error)
          └──────────┴──────────┴────────────┘
            ▲
            └── Note: tags[1] is 'postgres', NOT tags[0]!
```

#### Code Example

```sql
CREATE TABLE articles (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    tags TEXT[] NOT NULL DEFAULT '{}'
);

-- Creating a GIN index makes @> and && searches instant across millions of rows:
CREATE INDEX idx_articles_tags_gin ON articles USING GIN (tags);

INSERT INTO articles (title, tags) VALUES
    ('Postgres Data Types', ARRAY['postgres', 'sql', 'beginner']),
    ('Indexing Deep Dive', '{"postgres", "performance"}'); -- Note string literal uses double quotes inside {}!
```

### Operations on Arrays

PostgreSQL has specialized operators for searching, modifying, and expanding arrays. Note an important index rule: **`@>` and `&&` can use a GIN index, whereas `'val' = ANY(tags)` normally cannot!**

```sql
-- 1. Indexing and slicing (1-based!):
SELECT title, tags[1] AS primary_tag, tags[1:2] AS first_two_tags
FROM articles;

-- 2. Containment (@>): Does this article have BOTH 'postgres' and 'sql'? (Uses GIN index!)
SELECT title FROM articles WHERE tags @> ARRAY['postgres', 'sql'];

-- 3. Overlap (&&): Does this article have ANY of 'performance' or 'security'? (Uses GIN index!)
SELECT title FROM articles WHERE tags && ARRAY['performance', 'security'];

-- 4. Check if a single value is in the array with ANY() / ALL():
SELECT title FROM articles WHERE 'beginner' = ANY(tags);
SELECT ARRAY[2, 4, 6] @> ARRAY[2] AND 10 > ALL(ARRAY[2, 4, 6]); -- true (10 > every element)

-- 5. Append, concatenate (||), replace, or remove elements on UPDATE:
UPDATE articles
SET tags = array_remove(tags || ARRAY['tutorial'], 'beginner')
WHERE id = 1;

-- 6. Measuring length & converting between strings and arrays:
-- array_length('{}'::TEXT[], 1) returns NULL, whereas cardinality('{}'::TEXT[]) returns 0!
SELECT
    title,
    cardinality(tags) AS tag_count,
    array_position(tags, 'sql') AS sql_index,         -- 1-based position (or NULL if missing)
    array_to_string(tags, ', ') AS tags_csv,
    string_to_array('a,b,c', ',') AS parsed_array     -- {'a','b','c'}
FROM articles;

-- 7. UNNEST: Expand an array into individual rows (e.g., to count tag popularity)
SELECT unnest(tags) AS tag, COUNT(*) AS article_count
FROM articles
GROUP BY tag
ORDER BY article_count DESC;

-- 8. ARRAY_AGG: Collapse multiple rows back into an array
SELECT ARRAY_AGG(title ORDER BY id) AS all_titles
FROM articles;
```

#### When to Use Arrays

- **Simple, bounded lists of primitives:** Article tags, user roles, or secondary phone numbers where you don't need extra attributes per item.
- **Do not use arrays for:** Relationships that need their own columns (e.g., order items with quantity and price) or lists that need foreign-key enforcement (`REFERENCES` does not work on array elements). Use a proper join table for those.

---

## JSON and JSONB (`JSON`, `JSONB`)

PostgreSQL supports storing semi-structured JSON documents directly inside a column using two types:

- **`JSON`:** Stores an exact copy of the raw input text. It preserves whitespace, key order, and even duplicate keys. Every time you query a field inside `JSON`, PostgreSQL must re-parse the text from scratch.
- **`JSONB` ("JSON Binary"):** Parses the JSON into a decomposed binary structure on write. It strips insignificant whitespace, removes duplicate keys (keeping the last one), and sorts keys for fast lookups. **It supports GIN indexing.**

#### Analogy

> **Analogy:** *A raw photocopy vs. a filed binder.*
>
> `JSON` is a photocopy of a paper form—you have to read the whole page with your eyes every time you want to find one number. `JSONB` takes the form, organizes every field into tabbed dividers in a binary binder, and lets you jump straight to any field in microseconds.

#### Diagram: `JSON` vs. `JSONB` Pipeline

```text
Input: '{"b": 2, "a": 1, "a": 9}'

JSON  ──> Stored as exact text: '{"b": 2, "a": 1, "a": 9}'
          (Slow to query, preserves formatting & duplicates)

JSONB ──> Parsed to binary tree: {"a": 9, "b": 2}
          (Fast to query, deduplicated, GIN-indexable)
```

#### Code Example

```sql
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY,
    -- Always prefer JSONB unless you have a legal/audit need for exact raw formatting
    preferences JSONB NOT NULL DEFAULT '{}'::jsonb
);

-- GIN index accelerates @>, ?, ?|, ?& queries:
-- (Use jsonb_path_ops if you only use @> containment and want a much smaller index!)
CREATE INDEX idx_profiles_prefs_gin ON user_profiles USING GIN (preferences);

INSERT INTO user_profiles (user_id, preferences) VALUES
    (1, '{"theme": "dark", "retries": 3, "notifications": {"email": true, "sms": false}, "roles": ["admin", "editor"]}'),
    (2, '{"theme": "light", "retries": null, "notifications": {"email": false, "sms": false}, "roles": ["viewer"]}');
```

### Operations on JSONB

Understanding the difference between `->` and `->>` is the most important part of querying `JSONB`:
- `->` returns a **`JSONB`** value (keeping quotes around strings, allowing further chaining).
- `->>` returns a **`TEXT`** value (stripping JSON quotes, ready for `WHERE` text comparisons or casting).

Also watch out for **JSON `null` vs. SQL `NULL`**: inside `'{"retries": null}'::jsonb`, `preferences -> 'retries'` returns the JSONB token `'null'::jsonb` (so `IS NULL` is **`FALSE`**!), whereas `preferences ->> 'retries'` converts JSON `null` to a real SQL `NULL`.

```sql
-- 1. Extract top-level and nested values:
SELECT
    user_id,
    preferences -> 'theme' AS theme_as_jsonb,               -- '"dark"' (JSONB)
    preferences ->> 'theme' AS theme_as_text,               -- 'dark'   (TEXT)
    preferences -> 'notifications' ->> 'email' AS email_opt -- 'true'   (TEXT)
FROM user_profiles;

-- 2. Nested path extraction (#> returns JSONB, #>> returns TEXT) and casting:
-- Note the parentheses around ->> / #>> before :: because :: binds tighter than -> !
SELECT
    preferences #>> '{notifications, email}' AS email_opt_text,
    (preferences #>> '{notifications, email}')::BOOLEAN AS email_opt_bool,
    (preferences ->> 'retries')::INTEGER AS retries_int,    -- NULL for user 2!
    jsonb_typeof(preferences -> 'retries') AS retries_type  -- 'number' vs 'null'
FROM user_profiles;

-- 3. Containment (@>): Find all users with dark theme (uses GIN index!)
SELECT user_id FROM user_profiles
WHERE preferences @> '{"theme": "dark"}';

-- 4. Key existence (?, ?| any key, ?& all keys):
SELECT user_id FROM user_profiles
WHERE preferences ? 'roles'
  AND preferences ?& ARRAY['theme', 'notifications'];

-- 5. Expanding JSON arrays or objects into rows (with CROSS JOIN LATERAL):
SELECT u.user_id, r.role_name
FROM user_profiles u
CROSS JOIN LATERAL jsonb_array_elements_text(u.preferences -> 'roles') AS r(role_name);

SELECT key, value FROM jsonb_each_text('{"a": 1, "b": "hello"}'::jsonb);

-- 6. Updating a nested key with jsonb_set(target, path, new_value):
-- (Note: intermediate objects like 'notifications' must already exist)
UPDATE user_profiles
SET preferences = jsonb_set(preferences, '{notifications, sms}', 'true'::jsonb)
WHERE user_id = 1;

-- 7. Merging top-level keys (||), removing top-level (-), or removing nested (#-):
UPDATE user_profiles
SET preferences = (preferences || '{"language": "en"}'::jsonb) - 'roles' #- '{notifications, sms}'
WHERE user_id = 2;

-- 8. SQL/JSON Path filtering (PostgreSQL 12+): Check deep array items cleanly
SELECT user_id FROM user_profiles
WHERE preferences @? '$.roles[*] ? (@ == "admin")';
```

#### When to Use `JSONB`

- **Dynamic, sparse, or user-defined attributes:** Product specs that vary wildly by category (laptop RAM vs. shirt size), webhook payloads, or user UI preferences.
- **Do not use `JSONB` as an excuse to avoid schema design:** Core relational fields (`user_id`, `email`, `status`, `price`, `created_at`) belong in regular typed columns where constraints, foreign keys, and compact storage work best.

---

## Range Types (`INT4RANGE`, `DATERANGE`, `TSTZRANGE`)

Range types represent a **continuous span of values** with a lower bound and an upper bound in a single column, instead of splitting `start_date` and `end_date` into two separate columns.

Built-in range types include:
- `INT4RANGE` / `INT8RANGE` (ranges of integers)
- `NUMRANGE` (ranges of `NUMERIC`)
- `DATERANGE` (ranges of `DATE`)
- `TSRANGE` / `TSTZRANGE` (ranges of `TIMESTAMP` / `TIMESTAMPTZ`)
- **Multiranges (PostgreSQL 14+):** `INT4MULTIRANGE`, `DATEMULTIRANGE`, `TSTZMULTIRANGE`—a single value holding multiple non-overlapping ranges (e.g., store opening hours `09:00–12:00` and `13:00–17:00` excluding lunch).

Range notation uses `[` or `]` for **inclusive** bounds, and `(` or `)` for **exclusive** bounds. You can also omit either bound to create an **unbounded range** (`'[2026-11-28, )'` means "starting Nov 28, with no end date yet"—far cleaner than checking `end_date IS NULL`).

An important intermediate rule is **discrete canonicalization**: for discrete types with a fixed step (`INT4RANGE`, `INT8RANGE`, `DATERANGE`), PostgreSQL automatically normalizes bounds to `[lower, upper)` on write. That means `[1, 5]` (`INT4RANGE(1, 5, '[]')`) is stored and printed as `[1, 6)` so `[1, 5] = [1, 6)` evaluates to `TRUE`! Continuous ranges (`NUMRANGE`, `TSTZRANGE`) preserve your exact bracket types.

#### Analogy

> **Analogy:** *A highlighted block on a timeline.*
>
> Instead of writing down two separate sticky notes ("Starts Tuesday" and "Ends Friday") and constantly writing complex `start <= other_end AND end >= other_start` math, a range type treats the entire highlighted block as one object that natively understands overlap, containment, and gaps.

#### Diagram: Inclusive `[` vs. Exclusive `)` Bounds & Unbounded Ranges

```text
Booking A: [2026-11-28 10:00, 2026-11-28 11:00)
Booking B: [2026-11-28 11:00, 2026-11-28 12:00)
Active Plan: [2026-11-28 12:00, )  <── unbounded upper end (active indefinitely)

  10:00               11:00               12:00               ∞
  ├────────────────────┼────────────────────┼──────────────────►
  [===== Booking A ===)
                       [===== Booking B ===)
                                            [=== Active Plan ===>
                       ▲
                       └── 11:00 belongs ONLY to Booking B, so A && B is FALSE!
```

#### Code Example

```sql
CREATE TABLE room_bookings (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_name TEXT NOT NULL,
    during TSTZRANGE NOT NULL,
    -- Prevent double-booking the same room for overlapping time ranges!
    -- (Requires: CREATE EXTENSION IF NOT EXISTS btree_gist;)
    EXCLUDE USING gist (room_name WITH =, during WITH &&)
);

INSERT INTO room_bookings (room_name, during) VALUES
    ('Conference Room A', '[2026-11-28 10:00+00, 2026-11-28 11:00+00)'),
    ('Conference Room A', '[2026-11-28 11:00+00, 2026-11-28 12:00+00)');
```

### Operations on Ranges

```sql
-- 1. Check if a specific timestamp falls inside a booking (@> containment):
SELECT room_name, during FROM room_bookings
WHERE during @> TIMESTAMPTZ '2026-11-28 10:30+00';

-- 2. Check if two ranges overlap (&&) or sit right next to each other (-|- adjacent):
SELECT room_name FROM room_bookings
WHERE during && TSTZRANGE '[2026-11-28 10:45+00, 2026-11-28 11:15+00)';

SELECT TSTZRANGE '[2026-11-28 10:00+00, 2026-11-28 11:00+00)'
   -|- TSTZRANGE '[2026-11-28 11:00+00, 2026-11-28 12:00+00)';  -- true (touch without gap)

-- 3. Inspect bounds and calculate duration:
SELECT
    lower(during) AS start_time,
    upper(during) AS end_time,
    upper_inf(during) AS is_open_ended,
    upper(during) - lower(during) AS duration
FROM room_bookings;

-- 4. Set operations: intersection (*), union (+), difference (-), and discrete canonicalization:
SELECT
    INT4RANGE(1, 5, '[]') AS normalized_discrete,      -- Returns [1,6)
    INT4RANGE(1, 10) * INT4RANGE(5, 15) AS overlap,    -- Returns [5,10)
    INT4RANGE(1, 5) + INT4RANGE(5, 10) AS merged,      -- Returns [1,10)
    INT4RANGE(1, 10) - INT4RANGE(5, 15) AS trimmed,    -- Returns [1,5)
    -- Subtracting a middle slice splits one range into two, so use a multirange (PG 14+):
    INT4MULTIRANGE(INT4RANGE(1, 10)) - INT4MULTIRANGE(INT4RANGE(4, 7)) AS split_multirange; -- {[1,4),[7,10)}
```

#### When to Use Range Types

- **Calendar bookings, hotel reservations, pricing tiers, IP/port ranges, validity windows** (`effective_from` to `effective_until`).
- Whenever you need the database itself to guarantee **no overlapping intervals** using an `EXCLUDE USING gist` constraint.

---

## Network Address Types (`INET`, `CIDR`, `MACADDR`)

PostgreSQL includes first-class types for networking data rather than forcing you to store IP addresses as plain `TEXT`:

- **`INET`:** Stores an IPv4 or IPv6 host address, optionally with its subnet mask (e.g., `192.168.1.42/24`). If you omit the mask (e.g., `'192.168.1.42'`), it defaults to `/32` for IPv4 or `/128` for IPv6 (a single host). Allows non-zero host bits to the right of the netmask.
- **`CIDR`:** Stores an IPv4 or IPv6 **network block** (e.g., `192.168.1.0/24`). Rejects values that have non-zero bits to the right of the netmask.
- **`MACADDR` / `MACADDR8`:** Stores 6-byte or 8-byte hardware MAC addresses (e.g., `08:00:2b:01:02:03`).

#### Analogy

> **Analogy:** *A street address vs. a postal zip code.*
>
> `INET` (`192.168.1.42/24`) is a specific house on a street, along with the knowledge of how big the neighborhood is. `CIDR` (`192.168.1.0/24`) is the entire neighborhood boundary itself—you wouldn't name a whole neighborhood after House #42.

#### Diagram: `INET` vs. `CIDR` Bit Validation

```text
Subnet mask /24 means the first 24 bits are the network, last 8 bits are the host:

  192 . 168 . 1 . 42 /24
  └─── 24 network bits ───┘  └─ host ─┘

  INET:  192.168.1.42/24  ──> VALID   (represents host .42 inside /24 network)
  CIDR:  192.168.1.42/24  ──> ERROR!  (CIDR requires host bits to be 0: 192.168.1.0/24)
```

#### Code Example

```sql
CREATE TABLE access_logs (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    client_ip INET NOT NULL,
    device_mac MACADDR
);

CREATE TABLE blocked_subnets (
    subnet CIDR PRIMARY KEY,
    reason TEXT NOT NULL
);

INSERT INTO access_logs (client_ip, device_mac) VALUES
    ('192.168.1.42', '08:00:2b:01:02:03'),
    ('2001:db8::1', NULL);

INSERT INTO blocked_subnets (subnet, reason) VALUES
    ('192.168.1.0/24', 'Internal scanner range');
```

### Operations on Network Types

| Operator | Meaning | Example | Result |
| :--- | :--- | :--- | :--- |
| `<<` | Strictly contained within subnet | `INET '192.168.1.42' << CIDR '192.168.1.0/24'` | `true` |
| `<<=` | Contained within or equal to | `CIDR '192.168.1.0/24' <<= CIDR '192.168.1.0/24'` | `true` |
| `>>` / `>>=` | Contains / contains or equals | `CIDR '192.168.0.0/16' >> INET '192.168.1.42'` | `true` |
| `&&` | Subnets overlap | `CIDR '192.168.1.0/24' && CIDR '192.168.0.0/16'` | `true` |

```sql
-- Check if a client IP is contained within (<<=) a blocked CIDR subnet:
SELECT a.client_ip, b.subnet, b.reason
FROM access_logs a
JOIN blocked_subnets b ON a.client_ip <<= b.subnet;

-- Inspect address parts & anonymize IPs for GDPR compliance (zero out last 8 bits):
SELECT
    client_ip,
    host(client_ip) AS ip_without_mask,                   -- '192.168.1.42' (TEXT)
    network(set_masklen(client_ip, 24)) AS anonymized_24, -- 192.168.1.0/24 (for IPv4 logs)
    broadcast(INET '192.168.1.42/24') AS broadcast_ip,    -- 192.168.1.255/24
    masklen(INET '192.168.1.42/24') AS prefix_len         -- 24
FROM access_logs;
```

#### When to Use Network Types

- **Audit logs, rate limiters, firewall allowlists/blocklists, VPN/router management:** Storing IPs as `INET`/`CIDR` uses less space than `TEXT` (7 bytes for IPv4, 19 bytes for IPv6), validates syntax automatically, sorts numerically (`10.0.0.2` before `10.0.0.10`), and enables subnet containment queries (`<<=`).

---

## Geometric and Vector Types (`POINT`, `POLYGON`, `VECTOR`)

PostgreSQL also supports multi-dimensional coordinate data:

1. **Built-in 2D Geometric Types (`POINT`, `LINE`, `BOX`, `CIRCLE`, `POLYGON`):** Represent flat 2D Euclidean shapes. For example, `POINT(x, y)` stores an `(x, y)` pair in 16 bytes. Note that these assume a *flat* plane—for real-world Earth latitude/longitude (which sits on a curved sphere, where 1 degree of longitude shrinks near the poles), always use the **PostGIS** extension (`GEOGRAPHY` / `GEOMETRY`).
2. **Vector Type (`VECTOR(n)` via `pgvector`):** Stores fixed-length arrays of floating-point numbers representing AI/ML embeddings (e.g., `VECTOR(1536)`), enabling semantic similarity search.

#### Analogy

> **Analogy:** *A map pin vs. a GPS coordinate vs. an AI fingerprint.*
>
> - `POINT` is a pin on a flat sheet of graph paper.
> - PostGIS `GEOGRAPHY` is a pin on a physical globe (accounting for Earth's curvature).
> - `VECTOR(n)` is a numerical "meaning fingerprint" of a sentence or image in high-dimensional space.

#### Diagram: 2D Point Distance vs. High-Dimensional Vector Similarity

```text
2D Geometry (POINT):                 AI Embedding (VECTOR(3)):
   y ▲                               "king"  = [ 0.82,  0.14,  0.55]
     │      ● (4, 3)                 "queen" = [ 0.79,  0.18,  0.52]  (small angle = similar)
     │    /                          "apple" = [-0.41,  0.90, -0.12]  (large angle = unrelated)
     └──┴───────► x
```

#### Code Example

```sql
CREATE TABLE warehouses (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    location POINT NOT NULL
);

INSERT INTO warehouses (name, location) VALUES
    ('North Hub', POINT(10, 20)),
    ('South Hub', POINT(3, 4));

-- <-> calculates Euclidean distance; <@ checks if a POINT sits inside a BOX or CIRCLE:
SELECT
    name,
    location <-> POINT(0, 0) AS distance_from_origin,
    location <@ BOX '((0,0),(5,5))' AS inside_south_zone
FROM warehouses
ORDER BY distance_from_origin ASC;
```

In `pgvector`, you choose the distance operator that matches how your embedding model was trained: `<->` for **L2 (Euclidean) distance**, `<=>` for **Cosine distance** (most common for text embeddings), and `<#>` for **Negative inner product**.

#### When to Use Geometric and Vector Types

- **Built-in `POINT`/`BOX`/`POLYGON`:** Simple 2D coordinate math (game grids, floor plans, CAD bounding boxes).
- **PostGIS (`GEOGRAPHY`):** Real-world maps, "find restaurants within 5 km," delivery zones.
- **`pgvector` (`VECTOR(n)`):** Semantic search, RAG (Retrieval-Augmented Generation), image/product recommendations.

---

## The Special Case of `NULL`

`NULL` is not a data type—it is a **special marker** that can appear in any column (unless constrained with `NOT NULL`) to indicate that a value is **missing, unknown, or inapplicable**.

`NULL` is **not** equal to `0`, `''` (empty string), or `FALSE`. In SQL, comparing anything to `NULL` using `=` or `!=` yields `NULL` (unknown), never `TRUE`. Because `WHERE` only keeps rows where the condition evaluates to `TRUE`, `WHERE col = NULL` always returns **zero rows**—you must use `IS NULL` or `IS NOT NULL`.

Three additional `NULL` rules trip up developers regularly:
1. **Sorting (`ORDER BY`):** PostgreSQL treats `NULL` as **larger than all real values**. `ORDER BY score DESC` puts `NULL`s *first* unless you add `NULLS LAST`.
2. **Aggregates ignore `NULL`:** `COUNT(*)` counts every row, whereas `COUNT(col)`, `SUM(col)`, and `AVG(col)` skip `NULL`s (`AVG` of `10`, `NULL`, `20` is `15`, not `10`).
3. **`UNIQUE` constraints allow multiple `NULL`s by default:** Because `NULL != NULL`, a standard `UNIQUE (email)` allows many rows with `NULL` email. In PostgreSQL 15+, add `UNIQUE NULLS NOT DISTINCT (col)` if you want at most one `NULL`.

#### Analogy

> **Analogy:** *A sealed mystery box vs. an empty jar.*
>
> `0` or `''` is an empty jar—you can look inside and know for certain it holds nothing. `NULL` is a sealed box with a question mark on it: you cannot say two sealed boxes contain the same thing (`NULL = NULL` is unknown).

#### Diagram: `NULL` Comparisons & Sorting

```text
  col = NULL       ──>  NULL (Unknown) ──> Filtered OUT by WHERE!
  col != NULL      ──>  NULL (Unknown) ──> Filtered OUT by WHERE!
  col IS NULL      ──>  TRUE / FALSE   ──> Works as expected
  col IS NOT NULL  ──>  TRUE / FALSE   ──> Works as expected

  Default Sort Order (ASC):   1, 2, 10, NULL   (NULLs sort highest -> last in ASC, FIRST in DESC!)
  Safe Leaderboard (DESC):    ORDER BY score DESC NULLS LAST  ──> 10, 2, 1, NULL
```

#### Code Example

```sql
-- Common NULL-handling functions:
SELECT
    -- COALESCE returns the first non-NULL value in the list:
    COALESCE(NULL, NULL, 'fallback') AS safe_value,          -- 'fallback'
    -- NULLIF(a, b) returns NULL if a = b, otherwise returns a (great for preventing division by zero):
    100 / NULLIF(0, 0) AS safe_division;                     -- NULL (instead of division-by-zero crash)
```

---

## Type Casting (`::` and `CAST`) and Custom Domains

Because PostgreSQL is strongly typed, you will frequently need to convert a value from one data type to another—for example, turning a `TEXT` query parameter `'42'` into an `INTEGER`, or turning a `TIMESTAMPTZ` into a `DATE`.

PostgreSQL supports two equivalent syntaxes:
- **PostgreSQL shorthand:** `value::target_type` (e.g., `'42'::INTEGER`)
- **SQL standard syntax:** `CAST(value AS target_type)` (e.g., `CAST('42' AS INTEGER)`)

PostgreSQL applies casts at three levels:
1. **Implicit casts:** Automatic in expressions (e.g., `INTEGER + NUMERIC` promotes the integer to `NUMERIC`).
2. **Assignment casts:** Automatic only when writing to a column (e.g., inserting `'2026-11-28'` into a `DATE` column).
3. **Explicit casts:** Required when converting between unrelated types (`'42'::INTEGER`), using `::` or `CAST(...)`.

#### Analogy

> **Analogy:** *Repacking an item into a different shipping container.*
>
> A string `'2026-11-28'` is just characters on paper. Casting it with `'2026-11-28'::DATE` hands it to the calendar inspector, who validates that November 28 exists and packs it into a 4-byte `DATE` container so you can do date math on it.

#### Diagram: Valid vs. Failing Casts & Operator Precedence

```text
  '42'::INTEGER         ──>  42        (Success: valid whole number)
  42::TEXT              ──>  '42'      (Success: any value can cast to TEXT)
  '3.14'::NUMERIC       ──>  3.14      (Success)
  '3.14'::INTEGER       ──>  ERROR!    (invalid input syntax for type integer)
  '2026-02-30'::DATE    ──>  ERROR!    (date/time field value out of range)

  Precedence Trap (:: binds VERY tightly!):
  doc ->> 'qty'::INTEGER     ──>  ERROR! Parses as doc ->> ('qty'::INTEGER)
  (doc ->> 'qty')::INTEGER   ──>  OK!    Extracts TEXT first, then casts to INTEGER
```

#### Code Example & Custom Domains

```sql
SELECT
    '42'::INTEGER + 8 AS sum_result,                  -- 50
    CAST(19.99 AS TEXT) AS price_text,                -- '19.99'
    NOW()::DATE AS today_only,                        -- '2026-11-28'
    -- Casting integer to NUMERIC before division avoids integer truncation:
    7::NUMERIC / 2 AS exact_half,                     -- 3.5000000000000000
    -- Safe cast check (PostgreSQL 16+) before casting dirty strings:
    pg_input_is_valid('abc', 'integer') AS is_valid;  -- false (no error thrown!)

-- Bonus: A DOMAIN lets you define a reusable constrained data type across tables:
CREATE DOMAIN positive_money AS NUMERIC(12, 2) CHECK (VALUE >= 0);

CREATE TABLE payments (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    amount positive_money NOT NULL
);
```

---

## Which Data Type Should You Assign to Which Value?

When designing a table, start from **what the data means** and **what operations you will perform on it**, rather than picking a generic type out of habit.

### Decision Flowchart: Picking the Right Data Type

```text
What kind of data are you storing?
│
├── Numbers?
│   ├── Whole numbers only?
│   │   ├── Auto-incrementing PK? ────────────────► BIGINT GENERATED ALWAYS AS IDENTITY
│   │   ├── Small bounded count (< 32k)? ─────────► SMALLINT
│   │   └── General count / quantity? ────────────► INTEGER
│   └── Decimals?
│       ├── Money / exact financial math? ────────► NUMERIC(p, s)
│       └── Approximate sensor / physics / ML? ───► DOUBLE PRECISION (or REAL)
│
├── Text or Identifiers?
│   ├── Fixed random/distributed unique ID? ──────► UUID (prefer v7 at high write volume)
│   ├── Small static set of labels? ──────────────► ENUM (or TEXT + CHECK)
│   └── General strings, emails, phone, ZIP? ─────► TEXT (+ CHECK if bounded)
│
├── True / False? ────────────────────────────────► BOOLEAN NOT NULL DEFAULT FALSE
│
├── Dates, Times, or Schedules?
│   ├── Exact moment an event happened? ──────────► TIMESTAMPTZ
│   ├── Calendar date only (birthday, holiday)? ──► DATE
│   ├── Duration / timeout ('30 days')? ──────────► INTERVAL
│   └── Start + End window (bookings)? ───────────► TSTZRANGE / DATERANGE
│
└── Collections, Binary, or Specialized Data?
    ├── Simple list of same-type primitives? ─────► type[] (e.g., TEXT[])
    ├── Flexible / nested key-value attributes? ──► JSONB
    ├── Raw bytes / hashes (< few KB)? ───────────► BYTEA (large files go in S3/GCS!)
    ├── IP hosts / network blocks? ───────────────► INET / CIDR
    └── AI embeddings? ───────────────────────────► VECTOR(n) (pgvector)
```

### Value-to-Type Reference Table

| Real-World Value | Recommended PostgreSQL Type | Why This Type? |
| :--- | :--- | :--- |
| **Primary Key / Auto-increment ID** | `BIGINT GENERATED ALWAYS AS IDENTITY` | Never overflows `INT`'s 2.1B limit; compact (8 bytes) and fast in B-Trees. |
| **Public-facing / Distributed ID** | `UUID` | Prevents ID enumeration in URLs; safe to generate without central coordination. |
| **Money, Prices, Balances** | `NUMERIC(12, 2)` (or `BIGINT` cents) | Exact base-10 math; zero floating-point rounding drift. |
| **Percentages / Tax Rates** | `NUMERIC(5, 4)` | Exact decimal representation (e.g., `0.0825` for 8.25%). |
| **Item Counts / Inventory** | `INTEGER` (or `SMALLINT` if `< 32,767`) | Exact whole numbers; fast CPU arithmetic. |
| **Sensor Readings / ML Weights** | `DOUBLE PRECISION` (or `REAL`) | Hardware-accelerated scientific math where tiny rounding error is fine. |
| **Names, Emails, URLs, Bios** | `TEXT` (+ `CHECK` for length if needed) | Same engine as `VARCHAR` without arbitrary caps that are hard to change later. |
| **Case-Insensitive Email / Handle** | `TEXT` + `LOWER(email)` unique index | Ensures `'Ada@x.com'` and `'ada@x.com'` cannot both register. |
| **Phone Numbers / ZIP / Currency Codes** | `TEXT` | Preserves leading zeros (`02139`), `+` signs, and ISO codes (`'USD'`). |
| **True/False Flags** | `BOOLEAN NOT NULL DEFAULT FALSE` | 1 byte; `NOT NULL` avoids 3-valued logic bugs. |
| **Event Timestamps (`created_at`)** | `TIMESTAMPTZ` | Stores exact UTC instant; immune to server/client timezone mismatches. |
| **Birthdays / Calendar Holidays** | `DATE` | Represents a calendar day without shifting across time zones. |
| **Durations / Timeouts / SLAs** | `INTERVAL` | Expresses spans like `'30 days'` or `'15 minutes'` with calendar awareness. |
| **Booking / Schedule Windows** | `TSTZRANGE` or `DATERANGE` | Models `[start, end)` in one column and prevents double-bookings via GiST. |
| **Non-Contiguous Time Windows** | `TSTZMULTIRANGE` (PG 14+) | Models split schedules (e.g., morning + afternoon shifts) in one column. |
| **Small Static Set of Labels** | `ENUM` (or `TEXT` + `CHECK`) | Enforces valid labels at the DB level; use `TEXT` + `CHECK` if labels change often. |
| **Simple List of Tags / Roles** | `TEXT[]` | Avoids an extra join table when items have no extra attributes. |
| **Dynamic / Sparse Metadata** | `JSONB` | Flexible schema with fast key/containment queries and GIN indexing. |
| **Cryptographic Hashes / Keys** | `BYTEA` | Stores raw bytes in half the space of hex `TEXT`. |
| **IP Addresses / Subnets** | `INET` / `CIDR` | Validates IPv4/IPv6 syntax and enables subnet containment (`<<=`). |
| **AI Embeddings** | `VECTOR(n)` (`pgvector`) | Native similarity operators (`<->`, `<=>`) and vector indexes (HNSW). |

### Common Traps to Avoid

1. **Using `FLOAT` / `DOUBLE PRECISION` (or legacy `MONEY`) for money:** `0.1 + 0.2` becomes `0.30000000000000004`, and `MONEY` breaks across locale settings. Always use `NUMERIC` (or integer cents in `BIGINT`).
2. **Using `TIMESTAMP` instead of `TIMESTAMPTZ`:** `TIMESTAMP` drops timezone context. If a server in Virginia writes `14:00` and a worker in Frankfurt reads it, you no longer know *when* it actually happened.
3. **Querying timestamps with `BETWEEN`:** `WHERE created_at BETWEEN '2026-11-01' AND '2026-11-28'` misses everything on Nov 28 after `00:00:00`. Always use `>= '2026-11-01' AND < '2026-11-29'`.
4. **Using `INTEGER` for high-volume primary keys:** Hitting `2,147,483,647` causes sudden production outages and painful migrations. Default to `BIGINT`.
5. **Using `VARCHAR(255)` out of habit:** In PostgreSQL, `TEXT` is just as fast and doesn't force an arbitrary 255-character cutoff.
6. **Storing everything in `JSONB`:** You lose foreign keys, per-column `NOT NULL` guarantees, compact column storage, and simple SQL readability. Use `JSONB` for genuinely unstructured parts of a row, not the whole schema.
7. **Storing large files (images, videos, PDFs) in `BYTEA`:** This bloats database backups, replication, and memory caches. Store files in object storage (S3/GCS) and keep the `TEXT` URL/key in PostgreSQL.
