# Data Types in PostgreSQL: A Complete Study Guide

Every column in a database table must declare a **data type**. A data type is a contract between you and the database engine that defines four things:

1. **What values are allowed** (e.g., whole numbers, text, calendar dates, true/false).
2. **How much memory and disk space** each value uses.
3. **How values are compared and sorted** (e.g., `10 > 2` numerically, whereas `'10' < '2'` alphabetically).
4. **Which operations work on the column** (e.g., adding numbers, extracting the month from a timestamp, or checking whether a tag exists in a list).

Choosing the right data type prevents silent bugs—like rounding errors in money or broken time zones—keeps tables fast, and lets indexes work properly.

---

## Table of Contents

- [Numeric Types](#numeric-types)
  - [Integers (`SMALLINT`, `INTEGER`, `BIGINT`)](#integers-smallint-integer-bigint)
  - [Auto-Incrementing IDs (`IDENTITY` vs. `SERIAL`)](#auto-incrementing-ids-identity-vs-serial)
  - [Exact Decimals (`NUMERIC` / `DECIMAL`)](#exact-decimals-numeric--decimal)
  - [Floating-Point Numbers (`REAL`, `DOUBLE PRECISION`)](#floating-point-numbers-real-double-precision)
- [Text and Character Types (`TEXT`, `VARCHAR`, `CHAR`)](#text-and-character-types-text-varchar-char)
  - [Common String Operations](#common-string-operations)
- [Boolean (`BOOLEAN`)](#boolean-boolean)
- [Date and Time Types (`DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL`)](#date-and-time-types-date-time-timestamp-timestamptz-interval)
  - [Operations on Dates, Timestamps, and Intervals](#operations-on-dates-timestamps-and-intervals)
- [Universally Unique Identifiers (`UUID`)](#universally-unique-identifiers-uuid)
- [JSON and JSONB (`JSON`, `JSONB`)](#json-and-jsonb-json-jsonb)
  - [Operations on JSONB](#operations-on-jsonb)
- [Array Types (`type[]`)](#array-types-type)
  - [Operations on Arrays](#operations-on-arrays)
- [Enumerated Types (`ENUM`)](#enumerated-types-enum)
- [Binary Data (`BYTEA`)](#binary-data-bytea)
- [Range Types (`DATERANGE`, `TSTZRANGE`, `INT4RANGE`)](#range-types-daterange-tstzrange-int4range)
  - [Operations on Ranges](#operations-on-ranges)
- [Network Address Types (`INET`, `CIDR`)](#network-address-types-inet-cidr)
- [Geometric and Vector Types (`POINT`, `VECTOR`)](#geometric-and-vector-types-point-vector)
- [The Special Value `NULL`](#the-special-value-null)
- [Type Casting (`::` and `CAST`)](#type-casting--and-cast)
- [Which Data Type Should You Assign to Which Value?](#which-data-type-should-you-assign-to-which-value)
  - [Quick Decision Flowchart](#quick-decision-flowchart)
  - [Value-to-Type Reference Table](#value-to-type-reference-table)
  - [Common Beginner Traps to Avoid](#common-beginner-traps-to-avoid)

---

## Numeric Types

Databases separate numbers into three distinct families because computers store whole numbers, exact decimal fractions, and scientific approximations in completely different ways.

### Integers (`SMALLINT`, `INTEGER`, `BIGINT`)

Integers store **exact whole numbers** with no decimal point. PostgreSQL offers three sizes (all signed, meaning they hold both negative and positive values):

| Type | Aliases | Size | Minimum Value | Maximum Value |
| :--- | :--- | :--- | :--- | :--- |
| `SMALLINT` | `INT2` | 2 bytes | `-32,768` | `32,767` |
| `INTEGER` | `INT`, `INT4` | 4 bytes | `-2,147,483,648` (~2.14 billion) | `2,147,483,647` |
| `BIGINT` | `INT8` | 8 bytes | `-9,223,372,036,854,775,808` (~9.22 quintillion) | `9,223,372,036,854,775,807` |

If you try to insert a number larger or smaller than a column's range, PostgreSQL stops immediately with an `integer out of range` error instead of silently wrapping around. If you want to forbid negative numbers (such as stock counts), add `CHECK (stock_quantity >= 0)`.

#### Analogy

> **Analogy:** *Shipping boxes of fixed sizes.*
>
> A `SMALLINT` is a small jewelry box, an `INTEGER` is a moving box, and a `BIGINT` is a cargo container. They all hold whole items without cutting anything into pieces, and if an item doesn't fit, the lid refuses to close.

#### Diagram: Integer Sizes

```text
SMALLINT (2 bytes):  [ ± | 15 bits ]   ──>  up to 32,767
INTEGER  (4 bytes):  [ ± | 31 bits ]   ──>  up to ~2.14 billion
BIGINT   (8 bytes):  [ ± | 63 bits ]   ──>  up to ~9.22 quintillion
```

#### Code Example

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    warranty_months SMALLINT NOT NULL CHECK (warranty_months >= 0),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0)
);

INSERT INTO products (warranty_months, stock_quantity)
VALUES (24, 1500);

-- Watch out for integer division: dividing two integers drops the remainder!
SELECT
    7 / 2 AS integer_division,   -- Returns 3 (NOT 3.5!)
    7 % 2 AS remainder,          -- Returns 1
    7.0 / 2 AS decimal_division; -- Returns 3.5000000000000000
```

#### When to Use Integers

- **Primary keys:** Use `BIGINT` by default. Growing tables can outgrow `INTEGER`'s 2.14 billion limit, and changing a primary key column later in production is painful.
- **Counts, quantities, ages, HTTP status codes:** Use `INTEGER` (or `SMALLINT` for small bounded values).
- **Never use integers for:** Phone numbers or ZIP/postal codes—integers strip leading zeros (`02139` becomes `2139`).

### Auto-Incrementing IDs (`IDENTITY` vs. `SERIAL`)

In older tutorials you will often see `SERIAL` or `BIGSERIAL`. Those are legacy shorthands that quietly create a background counter (`SEQUENCE`) attached to an `INTEGER` or `BIGINT` column. Modern PostgreSQL uses the standard **`GENERATED ALWAYS AS IDENTITY`** syntax instead, which is cleaner and prevents accidental manual ID overwrites.

```sql
-- Modern preferred way:
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

Notice that auto-incrementing IDs can occasionally leave **gaps** (like `1, 2, 4`). If an insert fails or rolls back after claiming `3`, PostgreSQL does not reuse `3` so other concurrent writes never get blocked waiting on it. Gaps are completely normal.

---

### Exact Decimals (`NUMERIC` / `DECIMAL`)

`NUMERIC` and `DECIMAL` are two names for the exact same type in PostgreSQL. They store numbers in **base-10** so every decimal digit stays exact—meaning `0.1 + 0.2` always equals `0.3`.

You usually define it as `NUMERIC(precision, scale)`:
- **Precision:** Total number of digits allowed (before and after the decimal point).
- **Scale:** Number of digits allowed *after* the decimal point.

For example, `NUMERIC(10, 2)` holds up to 10 digits total, with 2 decimal places (up to `99,999,999.99`). Leaving out the numbers (`NUMERIC`) lets PostgreSQL store arbitrary lengths.

#### Analogy

> **Analogy:** *An accountant's ledger.*
>
> Dollars and cents are written cleanly in decimal columns. It takes a bit more work than mental estimation, but your accounts always balance down to the exact cent.

#### Diagram: `NUMERIC(6, 2)` Layout

```text
NUMERIC(6, 2)  ──>  6 digits total, 2 after the decimal point

  [ 9 ] [ 9 ] [ 9 ] [ 9 ]  .  [ 9 ] [ 9 ]
  └─────── 4 digits ─────┘    └─ 2 digits ─┘

  1234.56   ──> Stored as 1234.56
  1234.567  ──> Rounded to 1234.57
  12345.67  ──> ERROR (too many digits before the decimal point)
```

#### Code Example

```sql
CREATE TABLE invoices (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    subtotal NUMERIC(12, 2) NOT NULL,
    tax_rate NUMERIC(5, 4) NOT NULL -- e.g., 0.0825 for 8.25%
);

INSERT INTO invoices (subtotal, tax_rate) VALUES (19.99, 0.0825);

SELECT
    subtotal,
    ROUND(subtotal * (1 + tax_rate), 2) AS total, -- 21.64
    TRUNC(19.999::NUMERIC, 2) AS chopped_decimal  -- 19.99 (chops without rounding)
FROM invoices;
```

#### When to Use `NUMERIC`

- **Money, prices, balances, tax rates:** Always use `NUMERIC` (paired with a currency column like `'USD'`)—never use floating-point numbers or PostgreSQL's legacy locale-sensitive `MONEY` type.
- **Exact calculations** where even tiny rounding errors are unacceptable.

---

### Floating-Point Numbers (`REAL`, `DOUBLE PRECISION`)

`REAL` (4 bytes, ~6 decimal digits of accuracy) and `DOUBLE PRECISION` (8 bytes, ~15 decimal digits of accuracy) store numbers in **binary scientific notation**. Because computers work in base 2, simple base-10 decimals like `0.1` repeat forever in binary (just like `1/3` repeats as `0.3333...` in decimal), causing tiny rounding differences.

#### Analogy

> **Analogy:** *A measuring tape or scientific slide rule.*
>
> It handles huge or tiny measurements very fast using your CPU's math hardware, but you accept that the last decimal digits are slight approximations.

#### Diagram: Why `0.1 + 0.2 != 0.3` in Floating Point

```text
In Decimal:        0.1  +  0.2  =  0.3 (exact)

In DOUBLE PRECISION (Base-2):
  0.1  ──>  0.10000000000000000555...
+ 0.2  ──>  0.20000000000000001110...
─────────────────────────────────────
  Sum  ──>  0.30000000000000004440...  (NOT equal to 0.3!)
```

#### Code Example

```sql
SELECT
    (0.1::DOUBLE PRECISION + 0.2::DOUBLE PRECISION) = 0.3::DOUBLE PRECISION AS float_equal,
    (0.1::NUMERIC + 0.2::NUMERIC) = 0.3::NUMERIC AS numeric_equal;
-- float_equal: false | numeric_equal: true

-- To compare floating-point numbers safely, check if they are close enough:
SELECT ABS((0.1::DOUBLE PRECISION + 0.2::DOUBLE PRECISION) - 0.3) < 1e-9 AS close_enough; -- true
```

#### When to Use Floating-Point Numbers

- **Temperatures, sensor readings, physics math, machine learning weights:** Values that are already approximations and benefit from fast CPU math.
- **Never use them for money or exact `=` checks.**

---

## Text and Character Types (`TEXT`, `VARCHAR`, `CHAR`)

PostgreSQL gives you three types for storing text (UTF-8 strings):

1. **`TEXT`:** Variable-length string with no character cap.
2. **`VARCHAR(n)`:** Variable-length string that throws an error if you exceed `n` characters.
3. **`CHAR(n)`:** Fixed-length string that silently pads shorter text with trailing spaces (e.g., `'US'` in `CHAR(5)` becomes `'US   '`).

Under the hood, PostgreSQL stores all three using the exact same engine. **`VARCHAR(255)` is not faster or smaller than `TEXT` in PostgreSQL.** When text grows very large (over ~2 KB), PostgreSQL automatically compresses it in the background.

Also note that `VARCHAR(n)` counts **characters**, not bytes. In UTF-8, English letters take 1 byte, accented letters like `'é'` take 2 bytes, and emojis like `'🚀'` take 4 bytes.

#### Analogy

> **Analogy:** *Three ways to write a note.*
>
> - `CHAR(10)` is a row of 10 letter slots—if your word is 4 letters, you fill the other 6 slots with blank space tiles.
> - `VARCHAR(10)` is a note card that only takes as much ink as you write, but stops you at 10 characters.
> - `TEXT` is a notepad that grows only as long as what you actually write.

#### Diagram: Storage & UTF-8 Characters vs. Bytes

```text
Inserting 'cat':
  CHAR(6):     [ c ][ a ][ t ][   ][   ][   ]   (padded with 3 spaces)
  VARCHAR(6):  [ c ][ a ][ t ]                   (3 chars stored, max 6 enforced)
  TEXT:        [ c ][ a ][ t ]                   (3 chars stored, no artificial cap)

Multi-byte character '🚀':
  char_length('🚀')  = 1 character   (Fits inside VARCHAR(1)!)
  octet_length('🚀') = 4 bytes       (Takes 4 bytes on disk)
```

#### Code Example

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- Use TEXT + CHECK constraint so changing length limits later is easy
    username TEXT NOT NULL CHECK (char_length(username) BETWEEN 3 AND 30),
    email TEXT NOT NULL,
    bio TEXT
);

INSERT INTO users (username, email, bio)
VALUES ('ada_lovelace', 'Ada@Example.COM', 'First computer programmer.');
```

#### When to Use Each Text Type

- **`TEXT` (Default choice):** Use `TEXT` for almost everything (names, emails, URLs, descriptions, phone numbers, ZIP codes). Add a `CHECK (char_length(col) <= 255)` constraint when you want a business limit.
- **`VARCHAR(n)`:** Use only when external tools or conventions specifically require a declared limit.
- **Avoid `CHAR(n)`:** Trailing spaces create subtle bugs in comparisons and offer no speed benefit.

### Common String Operations

By default in PostgreSQL, string matching with `=` is **case-sensitive** (`'Ada' = 'ada'` is `false`). Use `LOWER()` to normalize emails or usernames, and `ILIKE` for case-insensitive searches.

```sql
SELECT
    -- || returns NULL if ANY piece is NULL; CONCAT() safely treats NULL as ''
    'Hello ' || NULL AS becomes_null,                          -- NULL
    CONCAT('Hello ', NULL, 'World') AS safe_join,             -- 'Hello World'
    -- Clean and split strings:
    LOWER(TRIM('  Ada@Example.COM  ')) AS clean_email,        -- 'ada@example.com'
    SPLIT_PART('ada@example.com', '@', 2) AS email_domain,    -- 'example.com'
    SUBSTRING('PostgreSQL' FROM 1 FOR 8) AS first_8_chars,    -- 'PostgreS' (1-based!)
    REPLACE('2026-11-28', '-', '/') AS formatted,             -- '2026/11/28'
    -- Wildcards: % matches any characters, _ matches 1 character
    'PostgreSQL' LIKE 'Post%' AS exact_case_match,            -- true
    'PostgreSQL' ILIKE 'post%' AS case_insensitive_match;     -- true

-- Join multiple rows into one comma-separated string:
SELECT STRING_AGG(username, ', ' ORDER BY username) AS user_list FROM users;
```

---

## Boolean (`BOOLEAN`)

`BOOLEAN` uses **1 byte** to store truth values. Unlike programming languages where booleans are strictly `true` or `false`, SQL columns can also be `NULL` (meaning *unknown* or *not answered*). This is called **three-valued logic**.

PostgreSQL accepts `'true'`/`'t'`/`'1'`/`'yes'` for `TRUE`, and `'false'`/`'f'`/`'0'`/`'no'` for `FALSE`.

#### Analogy

> **Analogy:** *A survey question with a checkbox.*
>
> - Checked = `TRUE` ("Yes")
> - Unchecked = `FALSE` ("No")
> - Left blank = `NULL` ("We don't know")

#### Diagram: Three-Valued Logic

```text
┌─────────┬─────────┬─────────────┬─────────────┐
│    A    │    B    │   A AND B   │   A OR B    │
├─────────┼─────────┼─────────────┼─────────────┤
│  TRUE   │  TRUE   │    TRUE     │    TRUE     │
│  TRUE   │  FALSE  │    FALSE    │    TRUE     │
│  TRUE   │  NULL   │    NULL     │    TRUE     │  (TRUE OR anything is still TRUE)
│  FALSE  │  NULL   │    FALSE    │    NULL     │  (FALSE AND anything is still FALSE)
│  NULL   │  NULL   │    NULL     │    NULL     │
└─────────┴─────────┴─────────────┴─────────────┘
```

#### Code Example

```sql
CREATE TABLE feature_flags (
    flag_name TEXT PRIMARY KEY,
    is_enabled BOOLEAN -- Nullable
);

INSERT INTO feature_flags VALUES
    ('dark_mode', TRUE),
    ('beta_checkout', FALSE),
    ('legacy_ui', NULL);

-- Shorthand for is_enabled = TRUE:
SELECT flag_name FROM feature_flags WHERE is_enabled;

-- Careful: is_enabled = FALSE misses the NULL row!
-- Use IS NOT TRUE when you want both FALSE and NULL:
SELECT flag_name FROM feature_flags WHERE is_enabled IS NOT TRUE;
```

> **Watch Out for `NOT IN` with `NULL`:**
>
> Because of three-valued logic, `WHERE id NOT IN (1, 2, NULL)` silently returns **zero rows** (`id != NULL` evaluates to `NULL`, ruining the whole `AND` chain). Filter out `NULL`s first or use `NOT EXISTS`.

#### When to Use `BOOLEAN`

- **Yes/no flags:** `is_active`, `is_verified`, `has_paid`, `is_deleted`.
- **Best practice:** Add `NOT NULL DEFAULT FALSE` unless "unknown" is truly meaningful—this avoids three-valued logic headaches altogether.

---

## Date and Time Types (`DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL`)

Dates and times are tricky because of time zones, Daylight Saving Time (DST), and varying month lengths. PostgreSQL provides five types so each temporal concept has an exact home:

| Type | Size | What It Stores | Time Zone Aware? |
| :--- | :--- | :--- | :--- |
| `DATE` | 4 bytes | Calendar date (`2026-11-28`) | No |
| `TIME` | 8 bytes | Clock time of day (`14:30:00`) | No |
| `TIMESTAMP` | 8 bytes | Date + time without a time zone | No |
| `TIMESTAMPTZ` | 8 bytes | Exact moment in time (`TIMESTAMP WITH TIME ZONE`) | Yes (stored as UTC) |
| `INTERVAL` | 16 bytes | A length or duration of time (`'2 hours 30 minutes'`) | N/A |

A key thing to understand about `TIMESTAMPTZ`: **it does not save the time zone name inside each row.** Instead, PostgreSQL converts any incoming timestamp into **UTC** for storage, and automatically shifts it to your database connection's `TimeZone` when you read it back. Always write dates in unambiguous **ISO 8601** format (`'2026-11-28'`), not `'11/28/2026'`.

#### Analogy

> **Analogy:** *A calendar, a wall clock, a cosmic instant, and a stopwatch.*
>
> - `DATE` is a square on a wall calendar ("Nov 28").
> - `TIME` is a wall clock ("2:30 PM") without knowing what day or country it is.
> - `TIMESTAMP` is a photo of a calendar and clock ("Nov 28, 2:30 PM") with no clue which country took the photo.
> - `TIMESTAMPTZ` is an **exact universal moment** (like the second a rocket launches)—New York calls it 9:30 AM and Tokyo calls it 11:30 PM, but both refer to the exact same instant.
> - `INTERVAL` is a **duration** ("2 hours 30 minutes"), not a point on the calendar.

#### Diagram: How `TIMESTAMPTZ` & `INTERVAL` Work

```text
1. How TIMESTAMPTZ handles time zones:
  New York user inserts:  '2026-11-28 09:30:00-05'
           │
           ▼  (Converted to UTC on disk)
  ┌──────────────────────────────────┐
  │ Stored UTC: 2026-11-28 14:30:00  │
  └──────────────────────────────────┘
           │
           ├──► New York user reads it back:  2026-11-28 09:30:00-05
           └──► Tokyo user reads it back:     2026-11-28 23:30:00+09

2. Why INTERVAL stores 3 parts separately:
  [ months ] + [ days ] + [ microseconds ]
  Because '1 month' varies (28–31 days), and '1 day' can be 23 or 25 hours on DST switch days!
```

#### Code Example

```sql
CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    starts_at TIMESTAMPTZ NOT NULL,
    duration INTERVAL NOT NULL,
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

#### 1. Adding & Subtracting Time (`+`, `-`)

Adding an `INTERVAL` to a timestamp moves it forward or backward. Subtracting two timestamps gives you an `INTERVAL`.

```sql
SELECT
    starts_at,
    starts_at + duration AS ends_at,
    starts_at - NOW() AS time_until_start,             -- Returns an INTERVAL
    DATE '2026-01-31' + INTERVAL '1 month' AS feb_end  -- Smart calendar math: 2026-02-28!
FROM events;
```

#### 2. Filtering Date Ranges Safely (Avoid `BETWEEN`!)

Beginners often write `WHERE starts_at BETWEEN '2026-11-01' AND '2026-11-28'`. That silently fails because `'2026-11-28'` becomes `2026-11-28 00:00:00` (midnight), **skipping every event during the day on November 28**.

Always use `>=` for the start and `<` for the *next* day—this catches every microsecond of November 28 **and** allows indexes to run at full speed:

```sql
SELECT title FROM events
WHERE starts_at >= TIMESTAMPTZ '2026-11-28 00:00:00+00'
  AND starts_at <  TIMESTAMPTZ '2026-11-29 00:00:00+00';
```

#### 3. Converting Time Zones with `AT TIME ZONE`

`AT TIME ZONE` shifts a `TIMESTAMPTZ` into a target city's local clock time (or attaches a time zone to a plain `TIMESTAMP`).

```sql
SELECT
    starts_at AT TIME ZONE 'Asia/Tokyo' AS tokyo_local_time,
    TIMESTAMP '2026-11-28 09:00:00' AT TIME ZONE 'America/New_York' AS ny_as_utc
FROM events;
```

#### 4. Grouping by Day, Week, or Month with `DATE_TRUNC`

`DATE_TRUNC('unit', timestamp)` zeroes out smaller units—turning `'2026-11-28 14:37:52+00'` into `'2026-11-28 14:00:00+00'` for `'hour'`, or `'2026-11-01 00:00:00+00'` for `'month'`. This is how you build daily or monthly charts.

```sql
SELECT
    DATE_TRUNC('month', starts_at) AS event_month,
    COUNT(*) AS total_events
FROM events
GROUP BY event_month
ORDER BY event_month;
```

#### 5. Extracting Pieces (`EXTRACT`, `AGE`) and Unix Epochs

```sql
SELECT
    EXTRACT(YEAR FROM starts_at) AS event_year,
    EXTRACT(ISODOW FROM starts_at) AS weekday_1_mon_to_7_sun,
    EXTRACT(EPOCH FROM starts_at) AS unix_seconds,   -- Convert to Unix epoch seconds
    TO_TIMESTAMP(1795874400) AS from_unix_seconds,   -- Convert Unix seconds back to TIMESTAMPTZ
    AGE(CURRENT_DATE, organizer_birthday) AS age     -- Friendly '31 years 7 mons 16 days'
FROM events;
```

#### 6. Formatting Strings (`TO_CHAR`), Parsing (`TO_TIMESTAMP`), and Generating Calendars

```sql
SELECT
    TO_CHAR(starts_at, 'YYYY-MM-DD HH24:MI') AS pretty_date,
    TO_TIMESTAMP('28/11/2026 14:30', 'DD/MM/YYYY HH24:MI') AS parsed_date
FROM events;

-- Generate every calendar day in November 2026 (great for LEFT JOINing daily analytics):
SELECT day::DATE
FROM generate_series(DATE '2026-11-01', DATE '2026-11-30', INTERVAL '1 day') AS day;
```

#### When to Use Temporal Types

- **`TIMESTAMPTZ`:** Your default for **almost every timestamp** (`created_at`, `updated_at`, `paid_at`, meetings).
- **`DATE`:** Birthdays, holidays, and calendar dates with no time of day.
- **`TIMESTAMP` (without TZ):** Rare—only for abstract wall-clock times (like "stores open at 9:00 AM local time everywhere").
- **`INTERVAL`:** Durations, timeouts, and subscription periods (`'30 days'`).

---

## Universally Unique Identifiers (`UUID`)

A `UUID` is a **128-bit (16-byte)** unique ID displayed as 32 hex digits with hyphens, like `550e8400-e29b-41d4-a716-446655440000`. Storing it as `UUID` takes **16 bytes** and validates the format automatically, whereas storing it as `TEXT` wastes 37+ bytes.

#### Analogy

> **Analogy:** *A pre-printed serial number from a giant lottery.*
>
> Sequential IDs (`1, 2, 3`) work like a deli ticket dispenser—everyone has to ask one central machine for the next number. A UUID lets any server or mobile device generate its own ID independently, with virtually zero chance of a collision.

#### Diagram: Random UUIDv4 vs. Time-Ordered UUIDv7

```text
UUIDv4 (gen_random_uuid() — completely random):
  [ random ][ random ][ ver=4 ][ random ]
  ──> Random writes jump all over index pages at huge scale

UUIDv7 (uuidv7() in PG 18+ — time-ordered prefix + random suffix):
  [ 48-bit timestamp ][ ver=7 ][ random ]
  ──> Writes append neatly in order, making indexes much happier
```

#### Code Example

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    label TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO api_keys (label) VALUES ('backend-service');
SELECT id, label FROM api_keys;
```

#### When to Use `UUID`

- **Public IDs in URLs and APIs:** Stops users from guessing `/orders/1001`, `/orders/1002` or counting how many customers you have.
- **Distributed / offline IDs:** When multiple services or client apps generate IDs before syncing.
- **Trade-off:** Twice as big as `BIGINT` (16 bytes vs. 8 bytes), and random UUIDv4 primary keys are slightly slower to index on massive multi-million-row tables (prefer UUIDv7 for high-volume tables).

---

## JSON and JSONB (`JSON`, `JSONB`)

PostgreSQL lets you store flexible JSON documents directly inside a column using two types:

- **`JSON`:** Saves the raw text exactly as you typed it (including extra spaces and key order). Every time you query inside it, PostgreSQL has to re-read the text from top to bottom.
- **`JSONB` ("JSON Binary"):** Parses the JSON into an optimized binary format on save. It removes wasted spaces, drops duplicate keys, supports **GIN indexes**, and queries much faster. **Always use `JSONB`.**

#### Analogy

> **Analogy:** *A photocopy vs. a tabbed binder.*
>
> `JSON` is a photocopy of a paper form—you have to scan the whole page every time you want one answer. `JSONB` files every field into tabbed dividers so PostgreSQL can jump straight to any key instantly.

#### Diagram: `JSON` vs. `JSONB`

```text
Input: '{"b": 2, "a": 1, "a": 9}'

JSON  ──> Saved verbatim as raw text: '{"b": 2, "a": 1, "a": 9}'
JSONB ──> Parsed into fast binary:    {"a": 9, "b": 2}  (deduplicated & indexable!)
```

#### Code Example

```sql
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY,
    preferences JSONB NOT NULL DEFAULT '{}'::jsonb
);

-- GIN index makes @> and ? queries super fast:
CREATE INDEX idx_profiles_prefs ON user_profiles USING GIN (preferences);

INSERT INTO user_profiles (user_id, preferences) VALUES
    (1, '{"theme": "dark", "retries": 3, "notifications": {"email": true, "sms": false}, "roles": ["admin", "editor"]}'),
    (2, '{"theme": "light", "retries": null, "notifications": {"email": false, "sms": false}, "roles": ["viewer"]}');
```

### Operations on JSONB

The #1 rule when reading `JSONB` is knowing the difference between `->` and `->>`:
- **`->`** returns a **`JSONB`** value (keeps JSON quotes; used to keep drilling deeper).
- **`->>`** returns plain **`TEXT`** (strips quotes; used for comparisons and casting).

Also watch out for JSON `null`: `preferences -> 'retries'` returns `'null'::jsonb` (which is **not** SQL `NULL`!), while `preferences ->> 'retries'` converts it into a normal SQL `NULL`.

```sql
-- 1. Extract values with -> and ->>:
SELECT
    user_id,
    preferences -> 'theme' AS as_jsonb,                    -- '"dark"'
    preferences ->> 'theme' AS as_text,                    -- 'dark'
    preferences -> 'notifications' ->> 'email' AS email,   -- 'true' (TEXT)
    -- Parentheses are required before :: because :: binds tighter than ->>
    (preferences ->> 'retries')::INTEGER AS retries_int    -- 3 or NULL
FROM user_profiles;

-- 2. Nested path shortcut (#> returns JSONB, #>> returns TEXT):
SELECT preferences #>> '{notifications, email}' FROM user_profiles;

-- 3. Containment (@>): Find users with dark theme (uses GIN index!)
SELECT user_id FROM user_profiles
WHERE preferences @> '{"theme": "dark"}';

-- 4. Key existence (?): Does the object have a 'roles' key?
SELECT user_id FROM user_profiles WHERE preferences ? 'roles';

-- 5. Turn a JSON array into rows with jsonb_array_elements_text():
SELECT u.user_id, r.role
FROM user_profiles u
CROSS JOIN LATERAL jsonb_array_elements_text(u.preferences -> 'roles') AS r(role);

-- 6. Update nested values (jsonb_set), merge keys (||), or delete keys (-):
UPDATE user_profiles
SET preferences = jsonb_set(preferences, '{notifications, sms}', 'true'::jsonb)
WHERE user_id = 1;

UPDATE user_profiles
SET preferences = (preferences || '{"language": "en"}'::jsonb) - 'roles'
WHERE user_id = 2;
```

#### When to Use `JSONB`

- **Dynamic or varying fields:** Product specs that differ per category (shirt sizes vs. laptop RAM), user UI settings, or third-party webhook payloads.
- **Do not use `JSONB` for core schema columns:** Keep IDs, emails, prices, status, and timestamps as regular typed columns so foreign keys and constraints protect your data.

---

## Array Types (`type[]`)

You can turn any PostgreSQL type into an array by adding `[]` (e.g., `TEXT[]`, `INTEGER[]`, `UUID[]`). Every element in an array must have the **same type**, and **PostgreSQL arrays start at index `1`**, not `0`! Asking for an index that doesn't exist (`tags[99]`) simply returns `NULL`.

#### Analogy

> **Analogy:** *A pill organizer inside a single table cell.*
>
> Instead of creating a whole extra child table just to hold 3 tags for a blog post, you keep an ordered tray right inside the `tags` column.

#### Diagram: 1-Based Array Indexing

```text
Column: tags TEXT[] = ARRAY['postgres', 'sql', 'indexing']

  Index:    [ 1 ]        [ 2 ]       [ 3 ]
          ┌──────────┬──────────┬────────────┐
  Value:  │ postgres │   sql    │  indexing  │
          └──────────┴──────────┴────────────┘
            ▲
            └── tags[1] is 'postgres' (NOT tags[0]!)
```

#### Code Example

```sql
CREATE TABLE articles (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    tags TEXT[] NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_articles_tags ON articles USING GIN (tags);

INSERT INTO articles (title, tags) VALUES
    ('Postgres Data Types', ARRAY['postgres', 'sql', 'beginner']),
    ('Indexing Deep Dive', ARRAY['postgres', 'performance']);
```

### Operations on Arrays

```sql
-- 1. Read elements & slices (1-based):
SELECT title, tags[1] AS first_tag, tags[1:2] AS first_two FROM articles;

-- 2. Containment (@>) & Overlap (&&) — both use a GIN index!
SELECT title FROM articles WHERE tags @> ARRAY['postgres', 'sql'];      -- Has BOTH
SELECT title FROM articles WHERE tags && ARRAY['performance', 'ai'];   -- Has ANY

-- 3. Check a single item with ANY():
SELECT title FROM articles WHERE 'beginner' = ANY(tags);

-- 4. Modify arrays (append, remove, concatenate with ||):
UPDATE articles
SET tags = array_remove(tags || ARRAY['tutorial'], 'beginner')
WHERE id = 1;

-- 5. Count items safely with cardinality() (returns 0 for empty '{}', unlike array_length() which returns NULL):
SELECT title, cardinality(tags) AS tag_count FROM articles;

-- 6. Expand array into rows (UNNEST) & collapse rows back into an array (ARRAY_AGG):
SELECT unnest(tags) AS tag, COUNT(*) AS uses FROM articles GROUP BY tag;
SELECT ARRAY_AGG(title ORDER BY id) AS all_titles FROM articles;
```

#### When to Use Arrays

- **Simple lists of values:** Article tags, permissions, or secondary phone numbers that don't need extra columns per item.
- **Use a separate join table instead** whenever items need extra properties (like quantity and unit price on an order item) or foreign-key references.

---

## Enumerated Types (`ENUM`)

An `ENUM` is a custom type defining a fixed list of allowed text labels. Behind the scenes, PostgreSQL stores each label as a compact **4-byte number**, while letting you write and read readable words.

#### Analogy

> **Analogy:** *A fixed dropdown menu.*
>
> You can only choose from the exact options on the menu (`'pending'`, `'paid'`, `'shipped'`). Nobody can type in a typo like `'shiped'`, and the options sort in the exact order you listed them.

#### Diagram: `ENUM` Storage & Sort Order

```text
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');

  Word shown in SQL        Stored on disk (4 bytes)
  ─────────────────        ────────────────────────
  'pending'       <─────>  1  (sorts first)
  'paid'          <─────>  2
  'shipped'       <─────>  3
  'cancelled'     <─────>  4  (sorts last)
```

#### Code Example

```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'cancelled');

CREATE TABLE customer_orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    status order_status NOT NULL DEFAULT 'pending'
);

INSERT INTO customer_orders (status) VALUES ('shipped'), ('pending');

-- Sorts by ENUM definition order ('pending' before 'shipped'), NOT alphabetically!
SELECT id, status FROM customer_orders ORDER BY status ASC;

-- Adding a value (must be committed before inserting rows with the new value):
ALTER TYPE order_status ADD VALUE 'refunded' AFTER 'shipped';
```

#### When to Use `ENUM`

- **Static, rarely changing sets of labels** like `log_level` (`debug`, `info`, `warn`, `error`).
- **Caveat:** Removing or renaming an `ENUM` label later is awkward, and you can't use a newly added label inside the same database transaction. If your status options change often, **`TEXT` with a `CHECK` constraint** (or a lookup table) is much easier to modify.

---

## Binary Data (`BYTEA`)

`BYTEA` ("byte array") stores raw sequences of binary bytes—such as SHA-256 hashes, encryption keys, or small binary tokens—without trying to interpret them as readable text. Like `TEXT`, large `BYTEA` values over ~2 KB are automatically compressed in the background.

#### Analogy

> **Analogy:** *A sealed opaque envelope.*
>
> Unlike `TEXT`, which checks that every character is valid UTF-8 and knows alphabetical order, `BYTEA` simply holds the exact bytes you put inside and hands them back untouched.

#### Diagram: `TEXT` vs. `BYTEA`

```text
TEXT:   Bytes ──> Checked as UTF-8 characters ──> Sorted alphabetically
BYTEA:  Bytes ──> Stored as-is (0x00..0xFF)   ──> Sorted byte-by-byte
```

#### Code Example

```sql
CREATE TABLE file_checksums (
    file_id BIGINT PRIMARY KEY,
    -- Raw 32-byte SHA-256 hash takes 32 bytes instead of 64 bytes as hex text
    sha256_digest BYTEA NOT NULL CHECK (octet_length(sha256_digest) = 32)
);

-- Insert hex bytes (\x...) or compute sha256() directly:
INSERT INTO file_checksums (file_id, sha256_digest)
VALUES (1, sha256(convert_to('hello world', 'UTF8')));

SELECT file_id, encode(sha256_digest, 'hex') AS hex_digest FROM file_checksums;
```

#### When to Use `BYTEA`

- **Cryptographic hashes, salts, small keys, or tiny serialized blobs** (bytes to kilobytes).
- **Never store large files (images, PDFs, videos) in `BYTEA`:** They bloat backups and slow down database caches. Store files in object storage (S3 / Google Cloud Storage) and save only the `TEXT` file key/URL in PostgreSQL.

---

## Range Types (`DATERANGE`, `TSTZRANGE`, `INT4RANGE`)

Instead of splitting a start and end into two separate columns (`starts_at` and `ends_at`), a range type stores an entire **continuous span** in a single column:
- `DATERANGE` (dates), `TSTZRANGE` (timestamps with time zone), `INT4RANGE` (integers), `NUMRANGE` (decimals).

Ranges use `[` / `]` for **inclusive** bounds and `(` / `)` for **exclusive** bounds. The standard pattern is `[start, end)` (inclusive start, exclusive end), so two back-to-back bookings like `[10:00, 11:00)` and `[11:00, 12:00)` don't collide at `11:00`. Omitting one side (`'[2026-11-28, )'`) represents an open-ended range with no end date yet.

#### Analogy

> **Analogy:** *A highlighted block on a timeline.*
>
> Instead of comparing two sticky notes (`start_a <= end_b AND end_a >= start_b`) in every query, a range column acts like one block on a calendar that naturally knows if two events overlap.

#### Diagram: Half-Open `[start, end)` Ranges

```text
Booking A: [10:00, 11:00)
Booking B: [11:00, 12:00)

  10:00               11:00               12:00
  ├────────────────────┼────────────────────┤
  [===== Booking A ===)
                       [===== Booking B ===)
                       ▲
                       └── 11:00 belongs only to Booking B (no overlap!)
```

#### Code Example

```sql
CREATE TABLE room_bookings (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_name TEXT NOT NULL,
    during TSTZRANGE NOT NULL,
    -- Database-enforced double-booking protection! (Requires: CREATE EXTENSION btree_gist;)
    EXCLUDE USING gist (room_name WITH =, during WITH &&)
);

INSERT INTO room_bookings (room_name, during) VALUES
    ('Room A', '[2026-11-28 10:00+00, 2026-11-28 11:00+00)'),
    ('Room A', '[2026-11-28 11:00+00, 2026-11-28 12:00+00)');
```

### Operations on Ranges

```sql
-- 1. Does a timestamp fall inside the booking? (@> containment)
SELECT room_name FROM room_bookings
WHERE during @> TIMESTAMPTZ '2026-11-28 10:30+00';

-- 2. Do two ranges overlap? (&&)
SELECT room_name FROM room_bookings
WHERE during && TSTZRANGE '[2026-11-28 10:45+00, 2026-11-28 11:15+00)';

-- 3. Read start, end, and duration:
SELECT lower(during) AS starts_at, upper(during) AS ends_at, upper(during) - lower(during) AS length
FROM room_bookings;
```

#### When to Use Range Types

- **Room bookings, hotel stays, subscription periods, pricing tiers:** Anywhere you work with start-to-end spans or want PostgreSQL to automatically reject overlapping bookings.

---

## Network Address Types (`INET`, `CIDR`)

PostgreSQL natively understands IPv4 and IPv6 addresses:
- **`INET`:** A single host IP address with an optional subnet mask (e.g., `'192.168.1.42'` or `'192.168.1.42/24'`).
- **`CIDR`:** An entire network block where all host bits must be zero (e.g., `'192.168.1.0/24'`).

#### Analogy

> **Analogy:** *A house address vs. a neighborhood boundary.*
>
> `INET` (`192.168.1.42/24`) is House #42 inside a neighborhood. `CIDR` (`192.168.1.0/24`) represents the whole neighborhood block itself.

#### Diagram: `INET` vs. `CIDR`

```text
  192.168.1.42/24   ──>  INET: VALID   (specific host .42 in a /24 subnet)
  192.168.1.42/24   ──>  CIDR: ERROR!  (CIDR requires .0 host bits: 192.168.1.0/24)
```

#### Code Example

```sql
CREATE TABLE access_logs (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    client_ip INET NOT NULL
);

INSERT INTO access_logs (client_ip) VALUES ('192.168.1.42'), ('2001:db8::1');

-- Check if a client IP falls inside (<<=) a network subnet:
SELECT client_ip
FROM access_logs
WHERE client_ip <<= CIDR '192.168.1.0/24';
```

#### When to Use `INET` / `CIDR`

- **Audit logs, IP allowlists/blocklists, rate limiting:** Takes less space than `TEXT`, rejects invalid IPs, sorts numerically (`10.0.0.2` before `10.0.0.10`), and lets you match subnets cleanly with `<<=`.

---

## Geometric and Vector Types (`POINT`, `VECTOR`)

1. **Built-in 2D Geometry (`POINT`, `BOX`, `POLYGON`):** Flat `(x, y)` coordinates on a 2D plane. Note that Earth is curved—for real GPS latitude/longitude distances in kilometers, use the **PostGIS** extension (`GEOGRAPHY`) instead.
2. **AI Vectors (`VECTOR(n)` via `pgvector`):** Fixed-length lists of numbers representing AI embeddings (e.g., `VECTOR(1536)`) for semantic search.

#### Analogy

> **Analogy:** *A pin on graph paper vs. an AI meaning fingerprint.*
>
> `POINT` marks an `(x, y)` spot on a flat grid, while `VECTOR(n)` captures the meaning of text or images as coordinates so similar meanings sit close together.

#### Diagram: 2D Grid vs. Vector Similarity

```text
2D Point:                         AI Embeddings (VECTOR):
  y ▲    ● (4, 3)                   "king"  = [ 0.82,  0.14,  0.55]
    │  /                            "queen" = [ 0.79,  0.18,  0.52]  (close!)
    └─────► x                       "apple" = [-0.41,  0.90, -0.12]  (far!)
```

#### Code Example

```sql
CREATE TABLE warehouses (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    location POINT NOT NULL
);

INSERT INTO warehouses (name, location) VALUES ('North Hub', POINT(10, 20)), ('South Hub', POINT(3, 4));

-- <-> calculates straight-line distance:
SELECT name, location <-> POINT(0, 0) AS distance
FROM warehouses
ORDER BY distance ASC;
```

#### When to Use Geometric & Vector Types

- **`POINT` / `BOX`:** Flat 2D layouts (floor plans, game grids). Use **PostGIS** for real-world Earth maps.
- **`VECTOR(n)` (`pgvector`):** AI semantic search (`<=>` cosine distance, `<->` Euclidean distance) and recommendations.

---

## The Special Value `NULL`

`NULL` is not a data type—it is a marker meaning **"missing, unknown, or not applicable"** that any column can hold unless marked `NOT NULL`.

`NULL` is **never** equal to `0`, `''`, or `FALSE`. In SQL, comparing anything to `NULL` with `=` or `!=` returns `NULL` (unknown), which `WHERE` treats as false—so `WHERE col = NULL` always returns **0 rows**.

Three quick `NULL` rules to remember:
1. **Always check with `IS NULL` / `IS NOT NULL`.**
2. **Sorting (`ORDER BY`):** PostgreSQL treats `NULL` as the largest value—so `ORDER BY score DESC` puts `NULL` rows at the **top** unless you add `NULLS LAST`.
3. **Aggregates:** `COUNT(*)` counts every row, while `COUNT(col)`, `SUM(col)`, and `AVG(col)` skip `NULL`s (`AVG` of `10, NULL, 20` is `15`).

#### Analogy

> **Analogy:** *A sealed mystery box vs. an empty jar.*
>
> `0` or `''` is an empty jar—you know for sure there's nothing inside. `NULL` is a sealed box with a question mark: you can't claim two sealed boxes match (`NULL = NULL` is unknown).

#### Diagram: `NULL` Comparisons & Sorting

```text
  col = NULL       ──>  NULL (Unknown)  ──> Filtered OUT by WHERE!
  col IS NULL      ──>  TRUE / FALSE    ──> Works properly

  ORDER BY score DESC             ──>  NULL, 100, 50, 10   (NULLs sneak to the top!)
  ORDER BY score DESC NULLS LAST  ──>  100, 50, 10, NULL   (What you usually want)
```

#### Code Example

```sql
SELECT
    -- COALESCE picks the first non-NULL value:
    COALESCE(NULL, NULL, 'guest') AS display_name,   -- 'guest'
    -- NULLIF(a, b) turns matching values into NULL (prevents divide-by-zero errors!):
    100 / NULLIF(0, 0) AS safe_division;             -- NULL instead of crashing
```

---

## Type Casting (`::` and `CAST`)

PostgreSQL lets you explicitly convert a value from one type to another using either PostgreSQL's `::` shorthand or standard SQL `CAST(...)`.

#### Analogy

> **Analogy:** *Repacking an item into the right container.*
>
> `'2026-11-28'` is just plain text. Casting it with `'2026-11-28'::DATE` checks that the date exists on the calendar and turns it into a real `DATE` so you can add days or extract the month.

#### Diagram: Casting & Operator Precedence

```text
  '42'::INTEGER      ──>  42        (Success)
  42::TEXT           ──>  '42'      (Success)
  'abc'::INTEGER     ──>  ERROR!    (Invalid number text)

  Watch parentheses with JSONB! (:: binds tighter than ->>):
  doc ->> 'qty'::INTEGER    ──> ERROR! Tries to cast the key name 'qty' to INTEGER
  (doc ->> 'qty')::INTEGER  ──> OK!    Extracts 'qty' as TEXT first, then casts
```

#### Code Example

```sql
SELECT
    '42'::INTEGER + 8 AS sum_result,        -- 50
    CAST(19.99 AS TEXT) AS price_text,      -- '19.99'
    NOW()::DATE AS today_only,              -- '2026-11-28'
    7::NUMERIC / 2 AS exact_half;           -- 3.5000000000000000
```

---

## Which Data Type Should You Assign to Which Value?

Start from **what the value means** and **how you'll query or calculate it**.

### Quick Decision Flowchart

```text
What kind of data are you storing?
│
├── Numbers?
│   ├── Whole numbers?
│   │   ├── Auto-incrementing PK? ─────────────► BIGINT GENERATED ALWAYS AS IDENTITY
│   │   └── Counts / quantities? ──────────────► INTEGER (or SMALLINT if < 32k)
│   └── Decimals?
│       ├── Money / exact financial math? ─────► NUMERIC(p, s)
│       └── Approximate sensor / ML math? ─────► DOUBLE PRECISION (or REAL)
│
├── Text or IDs?
│   ├── Unpredictable / distributed ID? ───────► UUID
│   ├── Small fixed list of statuses? ─────────► ENUM (or TEXT + CHECK)
│   └── Names, emails, URLs, phones, ZIPs? ────► TEXT (+ CHECK if bounded)
│
├── True / False? ─────────────────────────────► BOOLEAN NOT NULL DEFAULT FALSE
│
├── Dates, Times, or Windows?
│   ├── Exact moment something happened? ──────► TIMESTAMPTZ
│   ├── Calendar day only (birthday)? ─────────► DATE
│   ├── Duration ('30 days', '2 hours')? ──────► INTERVAL
│   └── Start + End window (bookings)? ────────► TSTZRANGE / DATERANGE
│
└── Lists, Objects, or Specialized Data?
    ├── Simple list of primitives (tags)? ─────► type[] (e.g., TEXT[])
    ├── Flexible key-value attributes? ────────► JSONB
    ├── Raw hashes / tiny binary (< few KB)? ──► BYTEA (large files go in S3/GCS!)
    ├── IP addresses / subnets? ───────────────► INET / CIDR
    └── AI embeddings? ────────────────────────► VECTOR(n) (pgvector)
```

### Value-to-Type Reference Table

| Real-World Value | Recommended Type | Why? |
| :--- | :--- | :--- |
| **Primary Key (Auto-increment)** | `BIGINT GENERATED ALWAYS AS IDENTITY` | Won't hit `INT`'s 2.1B limit; compact and fast. |
| **Public / Distributed ID** | `UUID` | Unpredictable in URLs; safe across multiple servers. |
| **Money, Prices, Balances** | `NUMERIC(12, 2)` | Exact base-10 math with zero rounding drift. |
| **Tax Rates / Percentages** | `NUMERIC(5, 4)` | Exact decimal precision (e.g., `0.0825`). |
| **Item Counts / Inventory** | `INTEGER` (or `SMALLINT`) | Fast exact whole numbers. |
| **Sensor / Scientific Readings** | `DOUBLE PRECISION` | Fast CPU floating-point math for approximate data. |
| **Names, Emails, URLs, Bios** | `TEXT` (+ `CHECK` if needed) | Just as fast as `VARCHAR(n)` without rigid limits. |
| **Phone Numbers / ZIP Codes** | `TEXT` | Keeps leading zeros (`02139`) and `+` signs intact. |
| **True/False Flags** | `BOOLEAN NOT NULL DEFAULT FALSE` | Compact 1-byte flag; `NOT NULL` prevents `NULL` bugs. |
| **Event Timestamps (`created_at`)** | `TIMESTAMPTZ` | Stores UTC instants safely across all time zones. |
| **Birthdays / Holidays** | `DATE` | Calendar day that doesn't shift across time zones. |
| **Durations / Timeouts** | `INTERVAL` | Expresses lengths like `'15 minutes'` or `'1 month'`. |
| **Reservations / Schedules** | `TSTZRANGE` / `DATERANGE` | Stores `[start, end)` together and prevents overlaps. |
| **Fixed Small Status Set** | `ENUM` (or `TEXT` + `CHECK`) | Enforces valid labels; use `TEXT + CHECK` if labels change often. |
| **Simple Tag / Role Lists** | `TEXT[]` | Keeps simple lists in one column without extra joins. |
| **Dynamic / Sparse Attributes** | `JSONB` | Fast, indexable storage for varying JSON structures. |
| **Password Hashes / Tokens** | `BYTEA` | Compact raw binary storage. |
| **IP Addresses / Subnets** | `INET` / `CIDR` | Validated IP storage with subnet matching (`<<=`). |
| **AI Embeddings** | `VECTOR(n)` (`pgvector`) | Fast similarity search (`<=>`, `<->`). |

### Common Beginner Traps to Avoid

1. **Using `FLOAT` / `DOUBLE PRECISION` for money:** `0.1 + 0.2` becomes `0.30000000000000004`. Always use `NUMERIC`.
2. **Using `TIMESTAMP` instead of `TIMESTAMPTZ`:** `TIMESTAMP` throws away time zone context, causing silent bugs whenever servers and users are in different zones.
3. **Querying timestamps with `BETWEEN`:** `BETWEEN '2026-11-01' AND '2026-11-28'` cuts off everything on Nov 28 after midnight. Always use `>=` and `<`.
4. **Using `INTEGER` for primary keys on growing tables:** Hitting 2.14 billion causes sudden write failures. Default to `BIGINT`.
5. **Using `VARCHAR(255)` out of habit:** In PostgreSQL, `TEXT` has identical performance without arbitrary cutoffs.
6. **Dumping whole tables into `JSONB`:** You lose foreign keys and clean per-column constraints. Keep core columns typed and use `JSONB` only for genuinely flexible fields.
7. **Storing big files in `BYTEA`:** Keep images, PDFs, and videos in object storage (S3/GCS) and store only the `TEXT` URL/key in PostgreSQL.
