# 2025-04-01 Introduction

## Motivation

### Why Study Databases?

Most relevant reason to you CS majors right now: SQL is a top programming language in the job market.

There are about a trillion ($10^{12}$) instances of SQLite alone out there right now, used in many industries including Android (smartphones), vehicles, web applications like QuickBooks, etc. Almost every software application needs some form of persistent storage, after all.

### Why Databases?

Databases serve as a way to *store*, *analyze*, and *share* data.

What can we use as databases? Consider this analogy to ancient times:
- Ancient civilizations (e.g. Egyptians) would write on papyrus to record data, akin to what how we use *spreadsheets* today.
- However, papyrus is not very durable and can become ripped, tattered, etc. over time.
- Now consider an alternative our ancestors also used: inscribing text on a stone tablet. This is much more durable, but now it's very heavy and inconvenient.

In modern times, software like Microsoft Excel & Word are like our version of papyrus and/or stone tablets. Why can't we just use them as databases?

You technically could: you *can* store, analyze, and share data saved on them. But imagine a typical software application:
- Your application may have *lots* of data. A spreadsheet or word document is not meant to be an arbitrary large sink of automated data.
- Yes, they don't stop you from throwing a lot of data into them anyway. But think about the last time you opened a massive spreadsheet/document and how laggy it was, both when launching and navigating it.
- There are likely multiple, many points of access needed between your application and its persistent storage. These files aren't optimized for many concurrent users.
- Another point not mentioned: such software isn't meant to be a source of structured data, so it would be difficult to design queries or parsers that work with their specific binary format. For example: a table is human-readable in a Word document, but think about how you would access that table programmatically from your application. Databases should make sense to both humans and machines.

Thus, we should specifically use **databases** for our applications.

### Good Properties of Databases

Databases should be:

- **Scalable**: should support an arbitrarily large amount of data.
- **Efficient**: should support fast operations on the data.
- **Durable**: should keep the data reliably and indefinitely.
- **Concurrent**: should support accessing the data from multiple places at once.

## Tables

Before building up to a full database, consider its smaller building block: **tables** (aka **relations**). A table represents the storage of one type of entity, like "pets", "customers", "products", etc. These are similar in concept to a spreadsheet (or one sheet/tab within a larger Google Sheets document).

Anatomy of a table:
- Every row (aka **tuple**) has the same length.
- Every column has the same length.
- Every column has a **data type**. All values within the same column must have that same data type.

If you're into linear algebra, you can think of a table as an $n \times k$ matrix: $n$ $k$-tuples.

A **schema** is like the definition of the table structure, akin to the "header row" of familiar tabular data but also with type information for every column.

> [!WARNING]
>
> The *order* of the columns in a schema *usually* does not matter, but there may be some language features where it does. An example is when [inserting data rows](#inserting-data-into-a-table).

A **database**, loosely speaking, is just many tables together. This is analogous to an entire Google Sheets document (which can have multiple sheets/tabs within it).It represents the larger context of its respective software application (like how "customers" and "products" tables combined might comprise some "store" database).

Tabular design if great because it's simple: intuitive for humans and easy for machines. Given a table of pet information, it's easy to "eyeball" queries like "how many pets are from Seattle?", "how many cats are from Seattle?", "how many cats older than 3 years old are from Seattle?", etc.

### Addressing Scalability

**Distributed Storage**

What if the table is too large? Split it over multiple machines!

For example, if there are 9 database rows and 3 machines, consider storing 3 rows on each machine.

**Distributed Compute**

What if the computer is too slow? Add more computers!

Once again, suppose there are 9 database rows and 3 machines, and suppose we want to add numbers from every row. In this case, we can simply *parallelize* the summation by summing a subset of the rows on each machine and then sum those results together.

This does complicate if the data has dependencies, but this is a good proof of concept for now.

### Addressing Concurrency

How do we avoid conflicting reads & writes? Recall from OS/systems programming that we can use **locks**. We can also partition the table and apply locking to a subset of the table each time.

### Addressing Durability

Use replication: make copies of critical regions. This introduces **redundancy** (the good kind), so if one copy gets corrupted, the data isn't lost.

## SQL Crash Course

### Getting Started with SQLite

We'll use **SQLite** as our specific flavor of SQL. To drop into a REPL, simply invoke at the command line:

```sh
sqlite3
```

This connects to a **transient** (in-memory) database. This is good for throwaway exploration. We can also use an actual persistent database by passing in an existing database file:

```sh
sqlite3 pets.db
```

When in the REPL, it's useful to know some commands.

To list the tables in the database:

```console
sqlite> .table
pets
sqlite> .tables  -- Same as .table
pets
```

To show the schema of a table:

```console
sqlite> .schema pets
CREATE TABLE pets (name text, breed text, age int, origin text, kind text);
```

To delete an entire table:

```console
sqlite> DROP TABLE pets;
```

This is actually a full-fledged SQL command, but we have not covered all of SQL yet, so I'll include it here so you know how to clean up unwanted tables during your exploratory programming.

### Creating a Table from a Schema

You can create a table with `CREATE TABLE` followed by the table name and a comma-separated list of column definitions:

```sql
CREATE TABLE pets (
  -- <column name> <data type>
  name TEXT,
  age INT,
  bday DATE,
  kind TEXT,
);
```

> [!NOTE]
>
> By *convention*, keywords are capitalized, but SQL isn't actually case-sensitive. It also isn't whitespace-sensitive, so feel free to split your commands over multiple lines for readability.

### Inserting Data into a Table

You can insert into an existing table with `INSERT INTO` with the table name and a full tuple of values, representing one row:

```sql
INSERT INTO pets VALUES
  ("casa", 8, "2017-01-01", "cat");
```

Note that here order matters&mdash;the values are matched one-to-one with the ordering of columns defined in the schema (`CREATE TABLE`).

Beware some syntactical pitfalls! Consider these problematic inserts:

```sql
INSERT INTO pets VALUES
  (casa, 8, 2017-01-01, cat);
-- (name, age, bday, kind)
```

This gives a syntax error. Strings must be quoted. Here we go:

```sql
INSERT INTO pets VALUES
  ("casa", 8, 2017-01-01, "cat");
```

The syntax error goes away, but now Casa's `bday` shows up as `2015`. This is because `2017-01-01` is treated like an expression of 3 integers: 2017 minus 1 minus 1. Dates are actually stored as **strings** in SQLite, just with a special format. Thus, we need to quote them too: `"2017-01-01"`.

Finally, consider this nonsense:

```sql
INSERT INTO pets VALUES (1, 2, 3, 4);
```

This actually works too unfortunately. SQLite simply *cast*s the values into the types defined in the schema. So be careful! Later, we'll learn more about **integrity constraints** that can mitigate this a bit.

### Querying Data from a Table

To read data from a table, we use `SELECT`:

```sql
SELECT * FROM pets;
```

Here `*` means "all columns", so return a result table with every column from the schema (no filtering).

> [!TIP]
> You can use `.mode box` in the SQLite REPL to make the query result tables look nicer:
>
> ```console
> sqlite> .mode box
> sqlite> SELECT * FROM pets;  -- This'll look pretty now.
> ```

You can use `LIMIT` to take the first $N$ rows from the result:

```sql
SELECT * FROM pets LIMIT 3;
```

Instead of `*`, you can specify the result table to only contain a subset of columns. For example, only taking the `age` column:

```sql
SELECT age FROM pets;
```

Multiple columns work too, of course:

```sql
SELECT name, age FROM pets;
```

### Querying Tables with Constraints

`SELECT` filters columns, but we can use `WHERE` to filter rows. That is, only include in the result set rows that satisfy some condition.

For example, only taking the `age` column and for rows where the `kind` is `"cat"`:

```sql
SELECT age FROM pets
WHERE kind="cat";
```

We can actually have a full-fledged **expression** in the `SELECT` clause:

```sql
SELECT age * 7 FROM pets WHERE kind="dog";
```

This would return the values of the `age` column but with every value multiplied by 7 (i.e. a dog's "human age"):

| age \* 7 |
| -------- |
| 14       |
| 21       |

Note that the column "name" gets renamed with the expression.

Another example: getting the birthday of cats over the age of 5. We demonstrate `AND` for logical conjunction and familiar comparison operators like `>`:

```sql
SELECT bday
FROM pets
WHERE kind="cat" AND age > 5;
```

Querying with the inverse of the condition is as simple as tacking on `NOT` for logical negation:

```sql
SELECT bday
FROM pets
WHERE NOT(kind="cat" AND age > 5);
```

There's `OR` for logical disjunction too:

```sql
SELECT bday
FROM pets
WHERE kind="dog" OR kind="cat";
```

**ASIDE:** Sneak peak of a **sub-query**. We can run a nested inner query first, returning a table that can then be checked against using `IN` by the outer query:

```sql
SELECT * FROM pets
WHERE "cat" IN (SELECT kind FROM pets);
```

This is advanced, covered more later.

### Aggregation

We can use **aggregate functions** to reduce values in a column(s) to a singular value. We apply them within the `SELECT` expression. For example:

```sql
SELECT COUNT(kind) FROM pets;
```

This would output a singular number, something like:

| COUNT(kind) |
| ----------- |
| 5           |

Note that this returns the count *including duplicates*. To only consider **distinct** values, use the `DISTINCT` keyword. For example, this returns the `kind` column but only with unique values:

```sql
SELECT DISTINCT kind from pets;
```

| kind |
| ---- |
| rat  |
| cat  |
| dog  |

Getting the *count* of distinct `kind`s would look like:

```sql
SELECT COUNT(DISTINCT kind) from pets;
```

> [!TIP]
>
> Where does `DISTINCT` go? You can think of `DISTINCT` as a qualifier on the *column*, `kind` in this case, so they go together. That is, don't think of `SELECT DISTINCT` as one operator. `SELECT DISTINCT` is really `SELECT` applied on a `DISTINCT column`.

More aggregate functions (they do what you think they do):

```sql
SELECT SUM(age) FROM pets;
SELECT MIN(age) FROM pets;
SELECT MAX(age) FROM pets;
SELECT AVG(age) FROM pets;
```

### Grouping

What if we want to aggregate by a specific column? For example, instead of finding the average age across *all* pets, we want to find the average age on a per-`kind` basis (what's the average age of all dogs, average age of all cats, etc. all in one result table). We can use `GROUP BY`, which effectively first groups the rows into their own imaginary sub-tables and then applies the function. For example:

```sql
SELECT kind, AVG(age)
FROM pets
GROUP BY kind;
```

| kind | AVG(age) |
| ---- | -------- |
| rat  | 2.0      |
| cat  | 7.0      |
| dog  | 13.5     |

### Query General Form

The general form of a SQL query (thus far):

```sql
SELECT e(columns)
FROM table
WHERE condition(s)
GROUP BY column(s);
```

`e(columns)` just means an arbitrary expression `e` of the columns. It could include math (e.g. `* 7`), aggregate functions (e.g. `SUM()`).

### SQL Semantics

Intuitively, you can visualize SQL queries thus far as a `for` loop:

```sql
SELECT age, name
FROM pets
WHERE 7 * age < 70;
```

```python
for p in pets:
  if 7 * p.age < 70:      # WHERE predicate.
    print(p.age, p.name)  # SELECT columns.
```

If you're familiar with formal logic, the logical semantics of SQL can be expressed as:

```
(age, name) in output
  <=>
exists p in pets:
  p.age=age & p.name=name & 7*age < 70
```

Example:

```
(10, maya) in output
  <=>
exists ip in pets:
  p.age=10 & p.name=maya & 7*10 < 70
```

We see in this case, the last condition `(7*10 < 70)` is false, so `(10, maya)` is NOT in the output.

### Queries Intuition

The query process can be seen as a composition of steps. Consider this example:

```sql
SELECT age, name
FROM pets
WHERE 7*age < 70;
```

The `WHERE` first "filters" *rows*, returning an intermediate sub-table with only the rows satisfying the condition. Note that this sub-table still has the same *schema*. It has the same number of *columns* as the original table.

The `SELECT` then "filters" *columns*, only keeping the ones specified in the expression. Note that the result has the same number of *rows* as the previous step.

## Relational Algebra Crash Course

**Relational algebra** is a way of mathematically modeling operations in a relational database. Here $T$ refers to a table and $t, r$ refer to tuples (rows) within $T$.

### Selection

$$\sigma_p(T) = \lbrace t | t \in T \land p(t) \rbrace$$

Analogous SQL operation: `WHERE`. We only "keep" tuples $t$ that satisfy some predicate $p(t)$.

### Projection

$$\pi_{x,y,z}(T) = \lbrace (t.x, t.y, t.z) | t \in T \rbrace$$

Analogous SQL operation: `SELECT DISTINCT`. We keep every tuple $t$ in $T$, but just *map* it to a specific permutation $x, y, z$ of columns.

> [!IMPORTANT]
>
> Note that it's `SELECT DISTINCT`, not just `SELECT`. This is just due to the nature of sets, which are defined to not have duplicates.

An **extended projection** is one where the column mapping can also support an arbitray expression `e` on it:

$$\pi_{e(x,y,z)}(T) = \lbrace e(t.x, t.y, t.z) | t \in T \rbrace$$

For example:

```sql
SELECT YEAR(bday) + age -- (Pretend a YEAR() function exists)
FROM pets;
```

The corresponding extended projection:

$$\pi_{year(bday)+age} = \lbrace \verb|YEAR|(bday) + age | t \in T \rbrace$$

### Union

$$T_1 \cup T_2 = \lbrace t | t \in T_1 \lor r \in T_2 \rbrace$$

Analogous SQL operator: `UNION`. Note that just like in set theory, the SQL `UNION` operator would discard duplicates. For example, this would simply return the original `pets` table:

```sql
SELECT * FROM pets
UNION
SELECT * FROM pets;
```

To keep duplicates, use `UNION ALL`:

```sql
SELECT * FROM pets
UNION ALL
SELECT * FROM pets;
```

### Intersection

$$T_1 \cap T_2 = \lbrace t | t \in T_1 \land r \in T_2 \rbrace$$

Analogous SQL operator: `INTERSECT`.

### Aggregation

$$\gamma_{F(x)}(T) = F(T.x)$$

Analogous SQL: aggregate functions like `SUM()`, `MAX()`, etc.

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
<script type="text/x-mathjax-config">
  MathJax.Hub.Config({ tex2jax: {inlineMath: [['$', '$']]}, messageStyle: "none" });
</script>
