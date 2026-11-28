# Full Text Searches - PG

- [Full Text Searches - PG](#full-text-searches---pg)
  - [PostgreSQL Full Text Search](#postgresql-full-text-search)
    - [Introduction to the PostgreSQL Full-Text Search](#introduction-to-the-postgresql-full-text-search)
      - [How the Full-Text Search Pipeline Works (Example Walkthrough)](#how-the-full-text-search-pipeline-works-example-walkthrough)
    - [PostgreSQL Full-Text Search Data Types](#postgresql-full-text-search-data-types)
      - [`tsvector`](#tsvector)
      - [`tsquery`](#tsquery)
      - [The Match Operator (`@@`)](#the-match-operator-)
    - [Using PostgreSQL Full-Text Search with Table Data](#using-postgresql-full-text-search-with-table-data)
      - [1) Setting up a sample table](#1-setting-up-a-sample-table)
      - [2) Simple full-text search](#2-simple-full-text-search)
      - [3) Full-text search with AND operator](#3-full-text-search-with-and-operator)
      - [4) Full-text search with OR operator](#4-full-text-search-with-or-operator)
      - [5) Full-text search with phrase](#5-full-text-search-with-phrase)
      - [6) Full-text search with negation](#6-full-text-search-with-negation)
    - [Full-Text Search Using GIN Indexes](#full-text-search-using-gin-indexes)
      - [Why Add a GIN Index If You Already Have a `tsvector` Column?](#why-add-a-gin-index-if-you-already-have-a-tsvector-column)
      - [How GIN Works Internally for Full-Text Search](#how-gin-works-internally-for-full-text-search)
      - [Expression Index Walkthrough & Why `WHERE to_tsvector(...)` Doesn't Recompute Every Row](#expression-index-walkthrough--why-where-to_tsvector-doesnt-recompute-every-row)
      - [Best Practices: When to Use (and When Not to Use) GIN Indexes](#best-practices-when-to-use-and-when-not-to-use-gin-indexes)
    - [Summary](#summary)
  - [The `pg_trgm` Extension](#the-pg_trgm-extension)
    - [What Is a Trigram and How Does It Work?](#what-is-a-trigram-and-how-does-it-work)
    - [Enable the `pg_trgm` extension](#enable-the-pg_trgm-extension)
    - [Example usage](#example-usage)
    - [Trigrams](#trigrams)
      - [Counting trigrams](#counting-trigrams)
      - [Computing similarity](#computing-similarity)
    - [Advanced text searching](#advanced-text-searching)
    - [Performance considerations](#performance-considerations)
    - [Conclusion](#conclusion)
  - [Choosing the Right Text Search in PostgreSQL](#choosing-the-right-text-search-in-postgresql)
    - [Quick Decision Diagram](#quick-decision-diagram)
    - [Comparison & Decision Matrix](#comparison--decision-matrix)
    - [Simple Real-World Examples](#simple-real-world-examples)
    - [PostgreSQL Search vs. Elasticsearch / OpenSearch](#postgresql-search-vs-elasticsearch--opensearch)

Sources:

1. [Neon article: Full text search](https://neon.com/postgresql/indexes/full-text-search)
2. [Neon article: pg_trgm extension](https://neon.com/docs/extensions/pg_trgm)

## PostgreSQL Full Text Search

**Summary**: In this tutorial, you will learn about PostgreSQL full-text search and how to use it to perform complex searches on text stored in the database.

### Introduction to the PostgreSQL Full-Text Search

In PostgreSQL, full-text search is a built-in feature that allows you to perform complex searches on text stored in the database.

Full-text search enables efficient searching of documents, articles, or any form of text-based content by indexing their words and phrases.

The following describes an overview of how PostgreSQL full-text search works:

- **Indexing**: PostgreSQL allows you to create indexes on columns with the `tsvector` data type. When creating a full-text search index, PostgreSQL analyzes text data and generates a specialized data structure called **`tsvector`**, which represents the indexed document in a format optimized for searches.
- **Text analysis**: For building a full-text index, PostgreSQL performs a text analysis process that involves tokenizing text into individual words or tokens, removing stop words (like `the`, `and`, ...), applying stemming (or lemmatization) to reduce words to their root forms, and performing other linguistic transformations to prepare the text for indexing.
- **Querying**: Once PostgreSQL creates the full-text index, you can perform full-text search queries using dedicated full-text search functions and operators. For example, you can search for specific words or phrases, apply boolean operators to combine search terms, and more.
- **Ranking**: Allows you to rank the search results based on their relevance to the query. PostgreSQL offers the `ts_rank()` function to calculate a rank score for each document based on its similarity to the search query. You can use the ranking feature to sort the search results based on relevance.
- **Highlighting**: PostgreSQL can generate snippets or summaries of documents containing the matching words from the search query.

#### How the Full-Text Search Pipeline Works (Example Walkthrough)

```text
Raw Documents:
  Doc 1: "The quick fox is jumping and jumping"
  Doc 2: "The fox jumps over the dog"

┌──────────────────────────────────────────────────────────────────────────┐
│ 1. TEXT ANALYSIS (to_tsvector)                                           │
│    Tokenize ──> Drop stop words (the, is, and, over) ──> Stem roots      │
│                                                                          │
│    Doc 1 ──> 'fox':3 'jump':5,7 'quick':2   (positions 5 & 7 for jump)   │
│    Doc 2 ──> 'dog':6 'fox':2    'jump':3                                 │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 2. INDEXING (GIN Inverted Index on tsvector)                             │
│    Maps every root lexeme directly to matching document IDs:             │
│                                                                          │
│    'dog'   ──> [ Doc 2 ]                                                 │
│    'fox'   ──> [ Doc 1, Doc 2 ]                                          │
│    'jump'  ──> [ Doc 1, Doc 2 ]                                          │
│    'quick' ──> [ Doc 1 ]                                                 │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 3. QUERYING (to_tsquery & @@ match operator)                             │
│    User searches: to_tsquery('fox & jumping')  ──>  'fox' & 'jump'       │
│    GIN lookup:    [ Doc 1, Doc 2 ]  ∩  [ Doc 1, Doc 2 ]                  │
│    Result:        Doc 1 and Doc 2 match (@@ evaluates to true)           │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 4. RANKING (ts_rank — sort by relevance)                                 │
│    #1  Doc 1  (higher score: 'jump' appears twice, at positions 5 & 7)   │
│    #2  Doc 2  (lower score:  'jump' appears once,  at position 3)        │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ 5. HIGHLIGHTING (ts_headline — generate matched snippets)                │
│    Doc 1 ──> "The quick <b>fox</b> is <b>jumping</b> and <b>jumping</b>" │
│    Doc 2 ──> "The <b>fox</b> <b>jumps</b> over the dog"                  │
└──────────────────────────────────────────────────────────────────────────┘
```

In practice, you often use the full-text search feature in applications including content management systems (`CMS`), document management systems, and so on to enable fast searches on a large volume of text.

### PostgreSQL Full-Text Search Data Types

PostgreSQL provides you with two data types unique to full-text search: `tsvector` and `tsquery`.

#### `tsvector`

`tsvector` is a data type that allows you to store preprocessed documents in a format optimized for efficient searching and retrieval of text.

A `tsvector` value contains a sorted list of lexemes (words) and their positions and weight within a document.

For example, Title `"Fast Queries"` (weight `A`) + Body `"Run fast queries"` (weight `B`) becomes:

```text
'fast':1A,4B  'queri':2A,5B  'run':3B
  │    │ │
  │    │ └── Weight ('A' = high priority, 'B' = normal; default 'D' is hidden)
  │    └──── Position (word #1 and word #4)
  └───────── Lexeme (root word, sorted A–Z)
```

- **Lexemes (sorted):** Root words (`Queries` → `'queri'`) let `"query"` match `"queries"` and keep lookups fast.
- **Positions:** Word lists alone can't tell `"fast queries"` apart from `"fast ... (500 words) ... queries"`. Positions (`1` next to `2`) make **phrase searches** (`<->`) possible.
- **Weights (`A`–`D`):** Keep casual body/footer mentions from tying with titles—tagging titles as `'A'` puts headline matches first.

Note that lexemes are words without the variation created by suffixes; for example, `watches`, `watched`, and `watching` all share the lexeme `watch`.

For example, the following uses the `to_tsvector()` function to convert the words `watches`, `watched`, and `watching` to `tsvector`:

```sql
SELECT to_tsvector('watches'),
       to_tsvector('watched'),
       to_tsvector('watching');
```

**Output:**

```text
 to_tsvector | to_tsvector | to_tsvector
-------------+-------------+-------------
 'watch':1   | 'watch':1   | 'watch':1
(1 row)
```

In this example, the `to_tsvector()` function converts the words into `tsvector` values. Instead of returning the original words, it returns the lexemes of these words, which is `watch`.

The following example uses the `to_tsvector()` function to convert a string to a `tsvector` value:

```sql
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.');
```

**Output:**

```text
                      to_tsvector
-------------------------------------------------------
 'brown':3 'dog':9 'fox':4 'jump':5 'lazy':8 'quick':2
(1 row)
```

In this example:

- Each entry in the `tsvector` value represents a word (lexeme) and its position within the string (or document). For example, the word `quick` appears at position 2, the word `brown` appears at position 3, and so on.
- Words are sorted in alphabetical order.
- Articles and stop words like `The` and `over` are omitted.

#### `tsquery`

`tsquery` is a data type that represents search queries in full-text searches. It allows you to specify search conditions containing the indexed document's words or phrases.

Additionally, a `tsquery` value can include search operators to refine the search conditions:

- **Boolean operators** (`&` for AND, `|` for OR, and `!` for NOT): Combine search terms and define logical relationships between them.
- **Phrase search** (`<->` / `<N>` in `to_tsquery()`, or `""` in `websearch_to_tsquery()`): Indicates that words must appear in the indexed document in the specified order (`<->` means right next to each other; `<N>` means `N` words apart).
- **Prefix search** (`:*`): A colon-asterisk after a word indicates that the search term should match any word starting with that prefix.
- **Negation** (`!` in `to_tsquery()`, or `-` in `websearch_to_tsquery()`): Excludes specific terms from the search results.
- **Grouping** (`()`): Parentheses group terms and operators so you control which conditions are evaluated together.

The `to_tsquery()` function converts a query string into a `tsquery` (and automatically stems words to their root lexemes). For example:

```sql
SELECT to_tsquery('jumping');
```

**Output:**

```text
 to_tsquery
------------
 'jump'
(1 row)
```

##### Easy Examples of Every `tsquery` Operator

All examples below search the exact same sentence (`tsvector`: `'brown':3 'dog':9 'fox':4 'jump':5 'lazy':8 'quick':2`):

> `"The quick brown fox jumps over the lazy dog."`

**1. AND (`&`) — Both words must exist anywhere in the text (any order)**

```sql
-- Matches (t): both 'fox' and 'dog' exist in the sentence
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('fox & dog');   -- t

-- No match (f): 'fox' exists, but 'cat' does not
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('fox & cat');   -- f
```

**2. OR (`|`) — At least one of the words must exist**

```sql
-- Matches (t): 'cat' is missing, but 'dog' exists
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('cat | dog');   -- t

-- No match (f): neither 'cat' nor 'bird' exists
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('cat | bird');  -- f
```

**3. Negation / NOT (`!`) — Exclude documents containing a word**

```sql
-- Matches (t): 'fox' exists AND 'cat' is NOT present
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('fox & !cat');  -- t

-- No match (f): 'fox' exists, but 'dog' IS present (so !dog fails)
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('fox & !dog');  -- f

-- Standalone negation (t): 'cat' is not in the sentence
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('!cat');        -- t
```

**4. Phrase Search (`<->`, `<N>`, and `websearch_to_tsquery`) — Exact word order & distance**

```sql
-- Matches (t): 'brown' (pos 3) is immediately followed by 'fox' (pos 4)
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('brown <-> fox');  -- t

-- No match (f): wrong order ('fox' comes AFTER 'brown', not before)
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('fox <-> brown');  -- f

-- No match (f): 'brown' sits in between 'quick' and 'fox'
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('quick <-> fox');  -- f

-- Matches (t): <2> means 2 positions apart ('quick' at 2, 'fox' at 4 -> 4 - 2 = 2)
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('quick <2> fox');  -- t

-- Tip: phraseto_tsquery() and websearch_to_tsquery() add <-> automatically:
SELECT phraseto_tsquery('quick brown fox');         -- 'quick' <-> 'brown' <-> 'fox'
SELECT websearch_to_tsquery('"quick brown fox"');   -- 'quick' <-> 'brown' <-> 'fox'
```

**5. Prefix Search (`:*`) — Match words starting with a prefix (great for autocomplete)**

```sql
-- Matches (t): 'quick' starts with 'qui'
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('qui:*');       -- t

-- Matches (t): 'brown' starts with 'bro'
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('bro:*');       -- t

-- No match (f): 'quick' ENDS with 'ick', it does not START with 'ick'
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('ick:*');       -- f
```

**6. Grouping (`()`) — Control which operators run first**

Just like multiplication runs before addition in math, **`!` runs first, then `<->`, then `&`, and `|` runs last**. Use parentheses whenever you mix `&` and `|`:

```sql
-- WITHOUT parentheses: read as fox | (cat & wolf)
-- Matches (t) because 'fox' alone satisfies the OR, even though 'wolf' is missing!
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('fox | cat & wolf');    -- t

-- WITH parentheses: must have ('fox' OR 'cat') AND MUST also have 'wolf'
-- No match (f) because 'wolf' is missing!
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('(fox | cat) & wolf');  -- f

-- Matches (t): has ('fox' OR 'cat') AND also has 'dog'
SELECT to_tsvector('The quick brown fox jumps over the lazy dog.')
    @@ to_tsquery('(fox | cat) & dog');   -- t
```

#### The Match Operator (`@@`)

The match operator (`@@`) evaluates whether the text in a document (`tsvector`) matches the terms specified in a search query (`tsquery`) and returns `true` if there is a match or `false` otherwise:

```sql
tsvector @@ tsquery
```

For example, the following statement uses the `@@` operator to determine if the string matches the `tsquery`:

```sql
SELECT
  to_tsvector(
    'The quick brown fox jumps over the lazy dog.'
  ) @@ to_tsquery('jumping') AS result;
```

**Output:**

```text
 result
--------
 t
(1 row)
```

It returns `true` (`t`) because the `tsvector` contains the word `jump`, which is the lexeme of the word `jumping`.

The following example uses the match operator (`@@`) to determine if the string contains the word `cat`:

```sql
SELECT
  to_tsvector(
    'The quick brown fox jumps over the lazy dog.'
  ) @@ to_tsquery('cat') AS result;
```

**Output:**

```text
 result
--------
 f
(1 row)
```

### Using PostgreSQL Full-Text Search with Table Data

Let's look at several examples of using full-text searches with boolean operators on a table.

#### 1) Setting up a sample table

First, create a new table called `posts`:

```sql
CREATE TABLE posts (
   id SERIAL PRIMARY KEY,
   title TEXT NOT NULL,
   body TEXT,
   body_search TSVECTOR
      GENERATED ALWAYS AS (to_tsvector('english', body)) STORED
);
```

In the `posts` table, `body_search` is a **generated column** with the data type `tsvector`.

Whenever you change data in the `body` column, PostgreSQL automatically converts it to a `tsvector` using the `to_tsvector()` function and stores it in the `body_search` column.

Second, insert some rows into the `posts` table:

```sql
INSERT INTO posts (title, body)
VALUES
    ('Introduction to PostgreSQL', 'This is an introductory post about PostgreSQL. It covers basic concepts and features.'),
    ('Advanced PostgresSQL Techniques', 'In this post, we delve into advanced PostgreSQL techniques for efficient querying and data manipulation.'),
    ('PostgreSQL Optimization Strategies', 'This post explores various strategies for optimizing PostgreSQL database performance and efficiency.');
```

Third, retrieve data from the `id` and `body_search` columns:

```sql
SELECT
  id,
  body_search
FROM
  posts;
```

**Output:**

```text
 id |                                                body_search
----+------------------------------------------------------------------------------------------------------------
  1 | 'basic':10 'concept':11 'cover':9 'featur':13 'introductory':4 'post':5 'postgresql':7
  2 | 'advanc':7 'data':14 'delv':5 'effici':11 'manipul':15 'post':3 'postgresql':8 'query':12 'techniqu':9
  3 | 'databas':9 'effici':12 'explor':3 'optim':7 'perform':10 'post':2 'postgresql':8 'strategi':5 'various':4
(3 rows)
```

#### 2) Simple full-text search

The following example uses the match operator (`@@`) to search for posts whose `body` contains the word `"PostgreSQL"`:

```sql
SELECT
  id,
  body
FROM
  posts
WHERE
  body_search @@ to_tsquery('PostgreSQL');
```

**Output:**

```text
 id |                                                   body
----+----------------------------------------------------------------------------------------------------------
  1 | This is an introductory post about PostgreSQL. It covers basic concepts and features.
  2 | In this post, we delve into advanced PostgreSQL techniques for efficient querying and data manipulation.
  3 | This post explores various strategies for optimizing PostgreSQL database performance and efficiency.
(3 rows)
```

#### 3) Full-text search with AND operator

The following example uses the AND operator (`&`) to search for posts whose `body` contains both `"PostgreSQL"` and `"techniques"` in any order:

```sql
SELECT
  id,
  body
FROM
  posts
WHERE
  body_search @@ to_tsquery('PostgreSQL & techniques');
```

**Output:**

```text
 id |                                                   body
----+----------------------------------------------------------------------------------------------------------
  2 | In this post, we delve into advanced PostgreSQL techniques for efficient querying and data manipulation.
(1 row)
```

#### 4) Full-text search with OR operator

The following example uses the OR operator (`|`) to search for posts whose `body` contains either the word `"efficient"` or `"optimization"`:

```sql
SELECT
  id,
  body
FROM
  posts
WHERE
  body_search @@ to_tsquery('efficient | optimization');
```

**Output:**

```text
 id |                                                   body
----+----------------------------------------------------------------------------------------------------------
  2 | In this post, we delve into advanced PostgreSQL techniques for efficient querying and data manipulation.
  3 | This post explores various strategies for optimizing PostgreSQL database performance and efficiency.
(2 rows)
```

#### 5) Full-text search with phrase

The following example searches for posts whose `body` contains the phrase `"PostgreSQL technique"`:

```sql
SELECT
  id,
  body
FROM
  posts
WHERE
  body_search @@ to_tsquery('''PostgreSQL technique''');
```

**Output:**

```text
 id |                                                   body
----+----------------------------------------------------------------------------------------------------------
  2 | In this post, we delve into advanced PostgreSQL techniques for efficient querying and data manipulation.
(1 row)
```

#### 6) Full-text search with negation

The following example searches for posts whose `body` does not contain the word `"efficient"`:

```sql
SELECT
  id,
  body
FROM
  posts
WHERE
  NOT body_search @@ to_tsquery('efficient');
```

**Output:**

```text
 id |                                         body
----+---------------------------------------------------------------------------------------
  1 | This is an introductory post about PostgreSQL. It covers basic concepts and features.
(1 row)
```

### Full-Text Search Using GIN Indexes

In PostgreSQL, **GIN** stands for **Generalized Inverted Index**. A GIN index is optimized for indexing composite values where a single row contains many searchable items—such as arrays, JSONB documents, and full-text search vectors (`tsvector`).

#### Why Add a GIN Index If You Already Have a `tsvector` Column?

Look back at the `body_search` generated column we created earlier. Storing `body_search` as a `tsvector` column saves PostgreSQL from having to tokenize and stem the text at query time—**but without an index, PostgreSQL still has to read every single row in the table one by one** (a **Sequential Scan**) to check `body_search @@ ...`.

On 3 rows or 1,000 rows, a sequential scan finishes in a millisecond. On 1,000,000 blog posts, reading every `tsvector` on disk takes seconds per search.

Adding a GIN index creates a reverse lookup map from **lexeme → matching rows**, turning an $O(N)$ table scan into an almost-instant index lookup. You have two ways to index full-text search:

1. **Index the stored `tsvector` column directly:**
   ```sql
   CREATE INDEX idx_posts_body_search ON posts USING GIN (body_search);
   ```
2. **Create a GIN expression index directly on the `TEXT` column** (skipping the extra `body_search` column altogether, as shown in the walkthrough below).

#### How GIN Works Internally for Full-Text Search

Think of a GIN index like the **index at the back of a textbook**: instead of flipping through every page (table row) looking for a word, you look up the word alphabetically in the back and immediately get the exact list of page numbers (row IDs) where it appears.

```text
GIN Index Structure (Built at INSERT / UPDATE time):

  B-Tree of Unique Lexemes          Posting Lists (Matching Heap Row IDs)
  ────────────────────────          ──────────────────────────────────────
  'advanc'      ──────────────────► [ Row 2 ]
  'basic'       ──────────────────► [ Row 1 ]
  'effici'      ──────────────────► [ Row 2, Row 3 ]
  'postgresql'  ──────────────────► [ Row 1, Row 2, Row 3 ]

Query: to_tsquery('postgresql & efficient')  ──>  'postgresql' & 'effici'
  1. Look up 'postgresql' ──> [ Row 1, Row 2, Row 3 ]
  2. Look up 'effici'     ──> [ Row 2, Row 3 ]
  3. Intersect both lists ──> [ Row 2, Row 3 ]  (Reads ONLY Rows 2 & 3 from table!)
```

One subtle detail: **a GIN index stores which rows contain each lexeme, NOT the word positions inside the text.** That keeps the index compact and makes `&` (AND) and `|` (OR) queries pure set intersections/unions inside the index. For phrase searches (`<->`), PostgreSQL still uses the GIN index to grab the small set of rows containing all the words, and then briefly checks the exact word positions on just those candidate rows.

#### Expression Index Walkthrough & Why `WHERE to_tsvector(...)` Doesn't Recompute Every Row

Let's see how an **expression index** works without a separate `body_search` column. First, drop the `posts` table and recreate it with only `title` and `body`:

```sql
DROP TABLE IF EXISTS posts;

CREATE TABLE posts (
   id SERIAL PRIMARY KEY,
   title TEXT NOT NULL,
   body TEXT
);
```

Second, insert rows into the `posts` table:

```sql
INSERT INTO posts (title, body)
VALUES
    ('Introduction to PostgreSQL', 'This is an introductory post about PostgreSQL. It covers basic concepts and features.'),
    ('Advanced PostgresSQL Techniques', 'In this post, we delve into advanced PostgreSQL techniques for efficient querying and data manipulation.'),
    ('PostgreSQL Optimization Strategies', 'This post explores various strategies for optimizing PostgreSQL database performance and efficiency.');
```

Third, create a GIN expression index on `to_tsvector('english', body)`:

```sql
CREATE INDEX body_fts
ON posts
USING GIN ((to_tsvector('english', body)));
```

Finally, search for `posts` whose `body` contains either `basic` or `advanced`:

```sql
SELECT
  id,
  body
FROM
  posts
WHERE
  to_tsvector('english', body) @@ to_tsquery('basic | advanced');
```

**Output:**

```text
 id |                                                   body
----+----------------------------------------------------------------------------------------------------------
  1 | This is an introductory post about PostgreSQL. It covers basic concepts and features.
  2 | In this post, we delve into advanced PostgreSQL techniques for efficient querying and data manipulation.
(2 rows)
```

At first glance, putting `to_tsvector('english', body)` inside `WHERE` looks expensive—as if PostgreSQL must tokenize and stem every `body` on the fly during the `SELECT`. **It does not!** Here is what actually happens:

1. **Precomputed on write, not on read:** When you `INSERT` or `UPDATE` a row, PostgreSQL runs `to_tsvector('english', body)` once and stores all its lexemes inside the `body_fts` GIN index.
2. **Exact expression matching by the query planner:** When PostgreSQL plans the `SELECT` query, it sees that `to_tsvector('english', body)` in your `WHERE` clause matches the **exact expression** used to build `body_fts`. Instead of scanning the table and calling `to_tsvector()`, it skips straight to a `Bitmap Index Scan` on `body_fts` to look up `'basic'` and `'advanc'`.

> **Important Gotcha — The Configuration Name Must Match Exactly:**
>
> An expression index only hits if the `WHERE` expression matches the index definition **character-for-character (including the language configuration)**:
>
> ```sql
> -- HITS the body_fts GIN index (exact match):
> WHERE to_tsvector('english', body) @@ to_tsquery('english', 'basic | advanced');
>
> -- MISSES the index! Falls back to a slow Sequential Scan converting every row on the fly:
> WHERE to_tsvector(body) @@ to_tsquery('basic | advanced');
> WHERE to_tsvector('simple', body) @@ to_tsquery('basic | advanced');
> ```
>
> In fact, PostgreSQL requires you to pass an explicit dictionary like `'english'` inside `CREATE INDEX`—writing `to_tsvector(body)` without `'english'` depends on session settings (`default_text_search_config`) and is not allowed in an index.

*(Note: The only time PostgreSQL recomputes `to_tsvector('english', body)` during a query with an expression index is on the **already-matched rows**—for example, if you call `ts_rank(to_tsvector('english', body), ...)` to score results, or to verify exact positions for a `<->` phrase match.)*

#### Best Practices: When to Use (and When Not to Use) GIN Indexes

**1. Stored `tsvector` Column + GIN Index vs. Expression GIN Index**

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ OPTION A: Save search_vec Column in the Table + Add a GIN Index         │
├─────────────────────────────────────────────────────────────────────────┤
│  Table Row:  [ id | title | body | search_vec (pre-processed words!) ]  │
│                                           │                             │
│  GIN Index:  points to matching rows ─────┘                             │
│                                                                         │
│  When searching:                                                        │
│   1. Find matching rows ──► GIN index finds them immediately            │
│   2. Rank / sort rows   ──► Reads search_vec directly (NO extra work)   │
│                                                                         │
│  Use when: You rank results (ts_rank), weight Title > Body, or          │
│            search exact phrases (<->). Uses slightly more disk space.   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│ OPTION B: Expression GIN Index ONLY (No Extra Column in the Table)      │
├─────────────────────────────────────────────────────────────────────────┤
│  Table Row:  [ id | title | body ]   (Smaller rows! No extra column)    │
│                                                                         │
│  GIN Index:  stores the processed words inside the index itself         │
│                                                                         │
│  When searching:                                                        │
│   1. Find matching rows ──► GIN index finds them immediately            │
│   2. Rank / sort rows   ──► Postgres must re-read & re-process body     │
│                             for the matched rows                        │
│                                                                         │
│  Use when: You want smaller tables and mostly just filter yes/no (@@).  │
└─────────────────────────────────────────────────────────────────────────┘
```

- **Use a generated `tsvector` column + GIN index** when you combine multiple columns with weights (e.g., `title` weighted higher than `body`), sort by `ts_rank()`, or run many phrase searches (`<->`).

  **Why it's needed for ranking:** A GIN index only remembers *which rows* contain a word—it does not store word counts, positions, or `'A'`/`'B'` weights. Without a stored `search_vec` column, `ts_rank()` would have to re-parse the raw `title` and `body` text of every matched row on the fly; storing `search_vec` gives `ts_rank()` those precomputed counts and weights immediately:
  ```sql
  ALTER TABLE posts ADD COLUMN search_vec TSVECTOR
      GENERATED ALWAYS AS (
          setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
          setweight(to_tsvector('english', COALESCE(body, '')), 'B')
      ) STORED;

  CREATE INDEX idx_posts_search_vec ON posts USING GIN (search_vec);

  -- Fast GIN lookup + fast ranking without re-tokenizing title or body:
  SELECT id, title, ts_rank(search_vec, q) AS rank
  FROM posts, to_tsquery('english', 'postgresql & optimization') AS q
  WHERE search_vec @@ q
  ORDER BY rank DESC;
  ```

  ```text
  Step-by-Step: How Weighted search_vec + GIN Works

  WHEN YOU SAVE A POST (Automatic!):
  ┌─────────────────────────────────────────────────────────────────────┐
  │ 1. Tag importance:                                                  │
  │    title: "PostgreSQL Tuning"  ──► Priority 'A' (High)              │
  │    body:  "Tuning tips"        ──► Priority 'B' (Normal)            │
  │                                                                     │
  │ 2. Postgres merges both into search_vec on the row:                 │
  │    search_vec = [ 'postgresql' (A), 'tip' (B), 'tune' (A & B) ]     │
  │                                                                     │
  │ 3. GIN index updates its word lookup map:                           │
  │    'postgresql' ──► [Post #1]                                       │
  │    'tune'       ──► [Post #1]                                       │
  └─────────────────────────────────────────────────────────────────────┘

  WHEN A USER SEARCHES & SORTS BY RELEVANCE:
  ┌─────────────────────────────────────────────────────────────────────┐
  │ 1. WHERE search_vec @@ ...   ──► GIN jumps straight to matching IDs │
  │ 2. ORDER BY ts_rank(...)     ──► Reads (A)/(B) tags from search_vec │
  │                                  (No need to re-read title or body) │
  │                                                                     │
  │ Result: A match in the Title (A) automatically ranks ABOVE          │
  │         a match found only in the Body (B)!                         │
  └─────────────────────────────────────────────────────────────────────┘
  ```

- **Use an expression GIN index (no stored `tsvector` column)** when you **don't need `ts_rank()`** and mostly filter with `@@`.

  - **How `ts_rank()` works:** While `@@` only answers *yes or no* ("does this post match?"), `ts_rank(vector, query)` calculates a **relevance score** (e.g., `0.06` vs. `0.24`) by counting **how often** the search words appear and checking **weights** (`'A'` title > `'B'` body). To count those matches, `ts_rank()` must inspect the full `tsvector` of every matched row.
  - **When `ts_rank()` IS needed:** Google-style search bars where results must be sorted by *"most relevant first"* (`ORDER BY ts_rank(...) DESC`).
  - **When `ts_rank()` is NOT needed (and why an expression GIN index wins):** In many apps, you sort by **date, price, or popularity** instead of relevance—for example, *"find recent support tickets mentioning `'refund'`, newest first."* Since you aren't calling `ts_rank()`, Postgres **never has to re-run `to_tsvector()` on the matched rows**, and omitting the stored `tsvector` column keeps your table much smaller (no duplicate word list stored on every row):

  ```sql
  -- Expression GIN index only (no extra column on the posts table):
  CREATE INDEX idx_posts_body_gin ON posts USING GIN (to_tsvector('english', body));

  -- Filtering with @@ and sorting by date (no ts_rank() needed!):
  SELECT id, title, created_at
  FROM posts
  WHERE to_tsvector('english', body) @@ to_tsquery('english', 'refund')
  ORDER BY created_at DESC
  LIMIT 20;
  ```

  ```text
  Why Expression GIN Index Wins When You Don't Need ts_rank():

    Table Row:  [ id | title | body | created_at ]   <── Smaller! No extra tsvector column
    GIN Index:  stores 'refund' ──► [Post #4, Post #9, Post #12]

    Query: "Newest posts mentioning 'refund'" (ORDER BY created_at DESC)
      1. GIN Index Lookup ──► Finds matching IDs immediately (no re-parsing!)
      2. Sort by Date     ──► Reads created_at directly; never needs to_tsvector() again!
  ```

**2. When NOT to Use a GIN Full-Text Index**

```text
Skip a tsvector GIN index in these 4 common situations:

  1. SMALL TABLE (e.g., 200 rows)
     [ All 200 rows fit right in memory ]
     ──► Scanning 200 rows directly is faster than checking an index first.

  2. HEAVY WRITE / LOG TABLE (10,000 inserts/sec, rarely searched)
     1 new log message (50 words) ──► Forces 50 separate updates in GIN!
     ──► Makes writes noticeably slower than a simple B-Tree or no index.

  3. "NOT" SEARCHES (e.g., WHERE NOT body_search @@ 'spam')
     Suppose 98 out of 100 posts do NOT have 'spam':
     ──► When almost every row matches, Postgres ignores the index and
         just reads the table from top to bottom anyway.

  4. TYPOS, PARTIAL WORDS, OR EXACT MATCHES
     ├─ Typo ("Posgres" vs "Postgres")   ──► Use pg_trgm extension
     ├─ Middle of word ("%post%")        ──► Use pg_trgm extension
     └─ Exact lookup (slug = 'intro')    ──► Use a normal B-Tree index
```

- **Very small tables (under a few thousand rows):** A sequential scan fits in RAM and is faster than maintaining and traversing an inverted index.
- **High-throughput, write-heavy tables (e.g., raw log ingestion that is rarely searched):** Updating a GIN index is noticeably slower than updating a B-Tree because a single document inserts/updates dozens or hundreds of lexeme entries. (PostgreSQL uses a `fastupdate` pending buffer by default to batch GIN writes, but heavy write tables still pay extra I/O and `VACUUM` cleanup cost.)
- **Negation-only queries (`WHERE NOT body_search @@ to_tsquery('efficient')`):** Excluding one word usually matches almost the whole table, so the query planner ignores the GIN index and scans the table sequentially anyway.
- **Substring (`%abc%`) or typo-tolerant (`Grate Expectaton`) searches:** Standard `tsvector` GIN indexes only match stemmed whole words, not arbitrary substrings or misspellings. Use the **`pg_trgm` extension** (`GIN` with `gin_trgm_ops`) for fuzzy/substring search, and a standard **B-Tree index** for exact equality (`=`) or prefix (`LIKE 'abc%'`) lookups.

### Summary

- Use PostgreSQL full-text search to perform complex linguistic searches on text stored in the database.
- Use the `tsvector` and `tsquery` data types for full-text searches.
- Use the match operator (`@@`) to check if documents match a query.
- Use a generated column with the `tsvector` type to store precomputed document vectors for full-text searches.
- Use `GIN` indexes to accelerate searches over `tsvector` expressions and columns.

## The `pg_trgm` Extension

_Improve Postgres text searches with the `pg_trgm` extension_

> Summary: The `pg_trgm` Postgres extension breaks strings into three-character trigrams to compute similarity scores, enabling fuzzy text search, proximity matching, and tolerance for misspellings beyond what `LIKE` and `ILIKE` support. Use it when queries must match approximate or misspelled input against large text columns without full-text search infrastructure. Supports GiST and GIN indexes, a configurable similarity threshold, and functions including `similarity`, `word_similarity`, and `strict_word_similarity`.

The `pg_trgm` extension enhances Postgres' ability to perform text searches by using trigram matching. Trigrams are groups of three consecutive characters taken from a string. By breaking down text into trigrams, Postgres can perform more efficient and flexible searches, such as similarity and proximity searches.

It's especially useful for fuzzy string matching and searching large bodies of text.

This guide covers how to enable `pg_trgm`, use it for text searches, and optimize queries.

**Note:** The `pg_trgm` extension is open-source and can be installed on any Postgres setup. Detailed information about the extension is available in the [PostgreSQL Documentation](https://www.postgresql.org/docs/current/pgtrgm.html).

### What Is a Trigram and How Does It Work?

Normally, databases check strings strictly: `"postgres" = "postgress"` is `false`, and `LIKE '%postgres%'` fails completely if a user makes even a single typo.

A **trigram** (*tri* = 3) solves this by chopping every word into overlapping **3-character pieces**—like sliding a 3-letter window across the word one step at a time.

**Example 1: Sliding a 3-letter window over `"apple"`**

```text
Word: "apple"

  [a p p] l e   ──►  "app"
   a [p p l] e  ──►  "ppl"
   a p [p l e]  ──►  "ple"
```

*(Under the hood, Postgres also pads words with spaces—`  apple `—so the beginning `"  a"`, `" ap"` and ending `"le "` get their own 3-character pieces too. We will inspect the exact padded chunks using `show_trgm()` below.)*

**Example 2: Why a typo still matches (`"postgres"` vs. `"postgress"`)**

Suppose a user accidentally types an extra `"s"` at the end of `"postgres"`:

```text
  "postgres"   ──►  pos, ost, stg, tgr, gre, res
  "postgress"  ──►  pos, ost, stg, tgr, gre, res, ess
                    └────── 6 out of 7 chunks match! ──────┘
```

Even with a typo—or a missing letter like `"aple"` vs. `"apple"`, or a swapped letter like `"expectaton"` vs. `"expectation"`—**most of the 3-letter building blocks still overlap**.

Postgres turns fuzzy matching into simple counting:
1. Break both strings into their sets of 3-letter chunks.
2. Count how many chunks they **share** vs. the **total unique chunks** across both.
3. Divide those two numbers to get a **similarity score between `0.0` (nothing in common) and `1.0` (identical)**. If the score is high enough (`>= 0.3` by default), Postgres treats it as a match!

### Enable the `pg_trgm` extension

Activate `pg_trgm` by running the `CREATE EXTENSION` statement in your Postgres client:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```

### Example usage

Let's say you're developing a database of books and you want to find books with similar titles. We first create a test table and insert some sample data, using the query below.

```sql
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title TEXT
);

INSERT INTO books (title)
VALUES
    ('The Great Gatsby'),
    ('The Grapes of Wrath'),
    ('Great Expectations'),
    ('War and Peace'),
    ('Pride and Prejudice'),
    ('To Kill a Mockingbird'),
    ('1984');
```

**Basic string matching**

The `pg_trgm` extension can help you do fuzzy matches on strings.

For example, the query below looks for titles that are similar to the misspelled phrase "Grate Expectation". The `%` operator, provided by `pg_trgm`, measures similarity between two strings based on trigrams, and returns results if the similarity is above a certain threshold.

```sql
SELECT *
FROM books
WHERE title % 'Grate Expectation';
```

This query returns the following:

```text
| id | title               |
|----|---------------------|
| 3  | Great Expectations  |
```

The similarity threshold can be adjusted by setting the `pg_trgm.similarity_threshold` parameter (default value is `0.3`).

### Trigrams

#### Counting trigrams

The `pg_trgm` module makes these assumptions about how to count trigrams in a text string:

- Only alphanumeric characters are considered.
- The string is lowercased before counting trigrams.
- Each word is assumed to be prefixed with two spaces and suffixed with one space.
- The set of trigrams output is deduplicated.

We can use the `show_trgm` function to see how `pg_trgm` counts trigrams in a string. Here is an example:

```sql
SELECT show_trgm('War and Peace'); -- {"  a","  p","  w"," an"," pe"," wa",ace,and,"ar ","ce ",eac,"nd ",pea,war}
```

#### Computing similarity

Given the set of trigrams for two strings `A` and `B`, `pg_trgm` computes the similarity score as the size of the intersection of the two sets divided by the size of the union of the two sets.

Here is an example.

```sql
SELECT show_trgm('War'), show_trgm('Bar'), similarity('War', 'Bar');
```

This query returns the following:

```text
| show_trgm              | show_trgm              | similarity |
|------------------------|------------------------|------------|
| {"  w"," wa","ar ",war} | {"  b"," ba","ar ",bar} | 0.14285715 |
```

There are 7 distinct trigrams across the two input strings and 1 trigram in common. So the similarity score comes out to be 1/7 (`0.14285715`).

### Advanced text searching

`pg_trgm` offers powerful tools for more complex text search requirements.

**Proximity search**

The `similarity` function provided by `pg_trgm` returns a number between 0 and 1, representing how similar the two strings are. By filtering on the similarity score, you can search for strings that are within the specified threshold.

```sql
SELECT title
FROM books
WHERE SIMILARITY(title, 'War and') > 0.3;
```

This query returns the following:

```text
| title         |
|---------------|
| War and Peace |
```

**Substring matching**

`pg_trgm` also provides functionality to match the input text value against substrings within the target string. The query below illustrates this:

```sql
SELECT
    word_similarity('apple', 'green apples'),
    strict_word_similarity('apple', 'green apples');
```

This query returns the following:

```text
| word_similarity | strict_word_similarity |
|-----------------|------------------------|
| 0.8333333       | 0.625                  |
```

The `word_similarity` function returns the maximum similarity score between the input string and any substring of the target string. The similarity score is still computed using trigrams. In this example, the first string `apple` matches with the substring `apple` in the target.

In contrast, the `strict_word_similarity` function only considers a subset of substrings from the target, namely only sequences of full words in the target string. That is, the first string `apple` matches the substring `apples` in the target, hence the lower score.

**Distance scores**

There are operators to calculate the `distance` between two strings, i.e., one minus the similarity score.

```sql
SELECT similarity('Hello', 'Halo') AS similarity, 'Hello' <-> 'Halo' AS distance;
```

This query returns the following:

```text
| similarity | distance  |
|------------|-----------|
| 0.22222222 | 0.7777778 |
```

Similarly, there are operators to compute the distance based on the `word_similarity` and `strict_word_similarity` functions.

### Performance considerations

While `pg_trgm` enhances text search capabilities, comparing trigrams across hundreds of thousands of rows on the fly can get slow without an index. Here are the key practical tips and traps to know:

- **Indexing (`GIN` vs. `GiST`):** You can create either a `GIN` or `GiST` trigram index to speed up `%`, `LIKE`, `ILIKE`, and regex (`~`, `~*`) searches:

  ```sql
  -- GIN: fastest for WHERE title % '...' and ILIKE '%...%' filtering
  CREATE INDEX trgm_idx_gin ON books USING GIN (title gin_trgm_ops);

  -- GiST: best when sorting by closest match with <-> and LIMIT
  CREATE INDEX trgm_idx_gist ON books USING GIST (title gist_trgm_ops);
  ```

  ```text
  Which Trigram Index Should You Pick?

    1. Searching with WHERE title % 'query' or WHERE title ILIKE '%sub%'?
       ──► Use GIN (gin_trgm_ops): fastest lookup for filtering matching rows.

    2. Finding "Top 5 closest matches" (ORDER BY title <-> 'query' LIMIT 5)?
       ──► Use GiST (gist_trgm_ops): lets Postgres grab the top 5 closest
           neighbors straight from the index without sorting the whole table!
  ```

- **Use `%` instead of `similarity() > 0.3` in `WHERE` (Huge Beginner Trap!):**
  Calling the `similarity()` function in your `WHERE` clause forces Postgres to compute the score for **every single row** and **ignores your trigram index**. Always use the `%` operator instead, and adjust the threshold setting if needed:

  ```sql
  -- SLOW (ignores the trigram index — scans every row!):
  SELECT title FROM books WHERE similarity(title, 'Grate Expectation') > 0.4;

  -- FAST (uses the GIN/GiST trigram index!):
  SET pg_trgm.similarity_threshold = 0.4;
  SELECT title FROM books WHERE title % 'Grate Expectation';
  ```

- **Accelerating `ILIKE '%middle%'` (and the 3-character minimum rule):**
  A normal B-Tree index only helps prefix searches (`title LIKE 'Great%'`) and breaks as soon as you put a wildcard at the front (`title ILIKE '%Expect%'`). A `pg_trgm` GIN index breaks the search word into 3-letter trigrams, making leading-wildcard searches fast too! However, **1- or 2-character searches like `ILIKE '%ex%'` don't have enough letters to form a full trigram**, so Postgres skips the index—in UI search bars, wait until the user types at least **3 characters** before querying.

- **Limiting results smartly (`LIMIT` + `<->`):**
  Simply slapping `LIMIT 5` onto `ORDER BY similarity(title, 'Grate') DESC` still forces Postgres to score and sort *every* matching row before keeping 5. Combine a `GiST` index with the `<->` distance operator (or filter first with `WHERE title % ...`) so Postgres stops early:

  ```sql
  -- FAST with GiST index: returns top 5 closest book titles directly
  SELECT title, title <-> 'Grate Expectation' AS distance
  FROM books
  ORDER BY title <-> 'Grate Expectation'
  LIMIT 5;
  ```

### Conclusion

`pg_trgm` offers a versatile set of tools for text processing and searching in Postgres. We went over the basics of the extension, including how to enable it and how to use it for fuzzy string matching and proximity searches.

## Choosing the Right Text Search in PostgreSQL

Choosing the wrong text search approach either gives users broken results ("no results" on a 1-letter typo) or slows down your database as tables grow. Here is a simple guide comparing every method covered in this document.

### Quick Decision Diagram

```text
What are you trying to search?
│
├── Exact start of a string (e.g., email/slug starts with "ada...")
│   └──► B-Tree Index + LIKE 'ada%'
│
├── Short text with typos or partial words (names, book titles, SKUs)
│   ├── Filtering matches (WHERE title % '...' or ILIKE '%sub%')
│   │   └──► pg_trgm + GIN Index (gin_trgm_ops)
│   └── "Top 5 closest suggestions" autocomplete (ORDER BY <-> LIMIT 5)
│       └──► pg_trgm + GiST Index (gist_trgm_ops)
│
└── Long text / articles / word roots ("running" matches "runs")
    ├── Need relevance score (ts_rank), Title > Body weights, or phrases
    │   └──► Stored tsvector Column + GIN Index
    └── Only filtering yes/no (@@) and sorting by date/price
        └──► Expression GIN Index only (no extra column)
```

### Comparison & Decision Matrix

| Approach | Best Use Case | Roots (`run` ↔ `running`) / Typos? | Scale: Storage & Memory | Scale: Speed & Writes | Industry Standard? |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`LIKE` / `ILIKE` + B-Tree** | Exact prefix (`'ada%'`) on usernames, slugs, emails | Roots: No<br>Typos: No | **Lowest** (tiny B-Tree; no extra column) | Very fast for `'prefix%'`; **breaks on `'%middle%'`** | Standard for exact/prefix lookups |
| **FTS: Expression `GIN` Index** | Filtering long text (`@@`) sorted by date/price (no `ts_rank`) | Roots: **Yes**<br>Typos: No | **Medium** (smaller table; lexemes stored only in index) | Fast reads up to millions of rows; slower writes than B-Tree | Standard for simple keyword filters |
| **FTS: Stored `tsvector` + `GIN`** | Blog/doc search bars needing `ts_rank()`, `setweight`, or phrases | Roots: **Yes**<br>Typos: No | **Higher** (stores `search_vec` in every row + GIN index) | Fast reads & ranking up to millions of rows; slower writes | **Default PG standard** for document search |
| **`pg_trgm` + `GIN` Index** | Fuzzy filtering (`%`) & leading-wildcard `ILIKE '%word%'` on titles/names | Roots: No<br>Typos: **Yes** | **High on long text** (many 3-letter chunks per row; keep to short/medium fields) | Very fast `%` & `ILIKE` lookups (3+ chars); heavier index memory | **Default PG standard** for typo/substring search |
| **`pg_trgm` + `GiST` Index** | Autocomplete dropdowns (`ORDER BY col <-> 'q' LIMIT 5`) | Roots: No<br>Typos: **Yes** | **Medium** (smaller index than trigram GIN) | Fast Top-N sorting (`LIMIT`); slightly slower exact `%` filtering | Standard for PG fuzzy autocomplete |

### Simple Real-World Examples

```sql
-- 1. Username prefix lookup -> Normal B-Tree (cheapest & fastest)
CREATE INDEX idx_users_name ON users (username text_pattern_ops);
SELECT * FROM users WHERE username LIKE 'ada%';

-- 2. Book title search with typos -> pg_trgm + GIN
CREATE INDEX idx_books_trgm ON books USING GIN (title gin_trgm_ops);
SELECT * FROM books WHERE title % 'Grate Expectation';

-- 3. Support tickets by keyword + newest first (no ts_rank) -> Expression GIN
CREATE INDEX idx_tickets_fts ON tickets USING GIN (to_tsvector('english', body));
SELECT id, created_at FROM tickets
WHERE to_tsvector('english', body) @@ to_tsquery('english', 'refund')
ORDER BY created_at DESC LIMIT 20;

-- 4. Blog search ranked by Title > Body -> Stored tsvector + GIN
ALTER TABLE posts ADD COLUMN search_vec TSVECTOR
    GENERATED ALWAYS AS (
        setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(body, '')), 'B')
    ) STORED;
CREATE INDEX idx_posts_fts ON posts USING GIN (search_vec);
SELECT id, title FROM posts, to_tsquery('english', 'postgres') q
WHERE search_vec @@ q ORDER BY ts_rank(search_vec, q) DESC;
```

### PostgreSQL Search vs. Elasticsearch / OpenSearch

A common question for engineers is: *"Should I just use PostgreSQL search, or set up Elasticsearch / OpenSearch?"* Here is an objective, practical comparison.

```text
Option 1: PostgreSQL Only (Default Starting Point)
  App ──► PostgreSQL (App Data + Search in ONE place)
          ✓ Zero sync bugs (new rows are searchable immediately)
          ✓ Easy SQL JOINs (e.g., filter by user permissions)
          ✗ All search load shares CPU/RAM with your main database

Option 2: PostgreSQL + Elasticsearch / OpenSearch (ES / OS)
  App ──► PostgreSQL (Source of truth) ──(sync worker / CDC)──► ES / OS Cluster
          ✓ Scales across many servers + rich search features
          ✗ Extra cost, ops overhead, and slight sync delay ("eventual consistency")
```

#### Objective Comparison by Factor

| Factor | PostgreSQL (`tsvector` + `pg_trgm`) | Elasticsearch / OpenSearch |
| :--- | :--- | :--- |
| **Setup & Maintenance** | **Very low** — already inside your database | **High** — separate cluster + data sync pipeline |
| **Data Freshness** | **Instant** (ACID — searchable the moment it commits) | **Delayed** (usually milliseconds to seconds behind PG) |
| **Typo + Synonyms + Roots Together** | **Basic** — FTS handles roots, `pg_trgm` handles typos (awkward to mix) | **Best-in-class** — combines typos, synonyms, and stemming seamlessly |
| **Relevance Ranking** | **Basic (`ts_rank`)** — counts words/weights, lacks true corpus-wide IDF | **Industry-grade (`BM25`)** — smart scoring + field/recency boosting |
| **Sidebar Facet Counts** (e.g., `Nike (42)`) | **Slow at scale** — requires extra `GROUP BY` queries | **Instant** — built-in aggregations return hits + counts together |
| **Scale (Storage & Memory)** | **Single-server bound** — great up to **~1M–10M rows** while index fits in RAM | **Horizontal sharding** — scales across servers to **100M+ docs / TBs** |
| **SQL `JOIN`s & Permissions** | **Trivial** — `JOIN` teams/roles right inside the search query | **Hard** — no real SQL joins; data must be flattened/duplicated |

#### Simple Example: When Does the Difference Actually Show Up?

- **Stay in PostgreSQL:** You are building a SaaS app where users search *their team's support tickets or blog posts*. You need `WHERE team_id = 42` permissions, you have 500,000 rows, and simple keyword/title matching works great.
- **Move to ES / OS:** You are building an **Amazon-style store** with 20 million products. When a user types `"runing shoos"`, you must fix the typo, match `"sneakers"` (synonym), rank popular in-stock shoes first (`BM25`), and show live sidebar counts (`Nike (42)`, `Size 10 (19)`, `Under $50 (11)`) in under 30ms.

#### Objective Rules of Thumb

- **Choose PostgreSQL when:**
  1. Your searchable data is under a **few million rows** (or your GIN indexes fit comfortably in RAM).
  2. You rely heavily on **SQL `JOIN`s, permissions, or instant consistency** (e.g., newly saved items must appear immediately).
  3. You have a small team and want **one database to back up, monitor, and pay for**.
- **Choose Elasticsearch / OpenSearch (or lighter Meilisearch / Typesense) when:**
  1. **Search IS your core product** (e-commerce catalog, job board, media discovery, multi-TB log analytics).
  2. You need **BM25 ranking, synonyms + typos together, or fast faceted sidebar counts** across millions of documents.
  3. Search traffic or index size starts **starving your main Postgres RAM/CPU** (and notice: even then, you can first combine PG `tsvector` + `pg_trgm` or move heavy search onto a Postgres read replica before adopting a whole new search engine!).
- **Golden Industry Rule:** **Always start with PostgreSQL.** Don't add Elasticsearch/OpenSearch on Day 1—only introduce a separate search cluster once you hit a real bottleneck in relevance features, faceted counts, or single-node RAM/CPU.
