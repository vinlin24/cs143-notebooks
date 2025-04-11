# 2025-04-08/Lecture 3: More SQL

You can find all tables (unless otherwise specified) used in this lecture in the Lecture 3 database file shared by Professor before class on [the class website](https://remy.wang/cs143/), also [downloaded to this repository for your convenience](data/lec3.db):

```sh
sqlite3 data/lec3.db
```

Professor also introduced some resources for playing with SQLite on your own time:

- [SQLite Browser](https://sqlitebrowser.org), which implements a familiar spreadsheet-like GUI over the database while supporting SQL queries.
- [SQLite documentation](https://sqlite.org), which you can use as a source of truth for researching syntax, etc. For example, [this page](https://sqlite.org/lang_select.html) provides a visual diagram on the grammar of the `SELECT` clause.

## Keys

We're starting to get more into the **relational** part of relational databases! Firstly, we introduce some special columns called **keys**.

A **primary key (PK)** determines all other columns in a table. That is, all tuples in the table are functionally determined by the PK. It follows that the values in the PK must be *unique*. You can think of it as like the "identifying" attribute of each tuple. That's why "IDs" (like "user ID", "student ID", etc.) are often used as PKs in practice!

A **foreign key (FK)** references a PK in another table. Remember table decomposition & joins from last lecture? FKs and PKs introduce a more systematic relationship between tables. If table $A$ is related to table $B$, $A$ would like to associate each of its tuples with tuples in $B$. It does this through the FK-PK relationship. Some things to note about FKs:

- The referenced PK *must* exist. That is, it can't be `NULL`. More on `NULL` later.
- All values in the FK column must be values in the PK of the referenced table. This should make intuitive sense&mdash;the FK is supposed to be like labels directing you to items in the PK; it doesn't make sense for it to have a value outside of the PK's domain.

### Declaring Keys

Consider the tables in the DB file:

```console
sqlite> SELECT * FROM employers;
┌──────┬─────────┐
│ name │  addr   │
├──────┼─────────┤
│ UCLA │ LA      │
│ UW   │ seattle │
└──────┴─────────┘
sqlite> SELECT * from people;
┌─────────┬──────┬───────┬──────┐
│  name   │ addr │ phone │ job  │
├─────────┼──────┼───────┼──────┤
│ remy    │ ...  │ 123   │ UCLA │
│ zifan   │ ...  │ 234   │ UCLA │
│ vincent │ ...  │ 345   │ UCLA │
│ remy    │ ...  │ 123   │ UW   │
│ dan     │ ...  │ 456   │ UW   │
│ magda   │ ...  │ 567   │ UW   │
└─────────┴──────┴───────┴──────┘
sqlite> .schema employers
CREATE TABLE IF NOT EXISTS "employers" (
        "name"  TEXT,
        "addr"  TEXT
);
sqlite> .schema people
CREATE TABLE IF NOT EXISTS "people" (
        "name"  TEXT,
        "addr"  TEXT,
        "phone" INTEGER,
        "job"   TEXT
);
sqlite> INSERT INTO people VALUES ("remy", "...", 345, "USC");
sqlite> SELECT * FROM people;
┌─────────┬──────┬───────┬──────┐
│  name   │ addr │ phone │ job  │
├─────────┼──────┼───────┼──────┤
│ remy    │ ...  │ 123   │ UCLA │
│ zifan   │ ...  │ 234   │ UCLA │
│ vincent │ ...  │ 345   │ UCLA │
│ remy    │ ...  │ 123   │ UW   │
│ dan     │ ...  │ 456   │ UW   │
│ magda   │ ...  │ 567   │ UW   │
| remy    | ...  | 345   | USC  |
└─────────┴──────┴───────┴──────┘
```

Inserting a `job` that isn't `UCLA` or `UW` works just fine because this version of the table did not declare a foreign key constraint on the schema. As far as SQL is concerned, `job` is just like any other column in `people`. Similarly, `name` is just like any other column in `employers` since `name` wasn't declared as a primary key. Let's fix that:

```sql
-- Undo our blasphemy from earlier.
DELETE FROM people WHERE job="USC";

-- Remember to enable this so SQLite actually enforces foreign keys!
PRAGMA foreign_keys = ON;

CREATE TABLE employers_new(
  -- Declare name as a PRIMARY KEY.
  name TEXT PRIMARY KEY,
  addr TEXT
);

-- I'm using a shortcut here: insert values from another table.
INSERT INTO employers_new SELECT * FROM employers;

CREATE TABLE people_new(
  name TEXT,
  addr TEXT,
  phone TEXT,
  job TEXT,
  FOREIGN KEY(job) REFERENCES employers(name)
  --          ^^^ FK column   ^^^^^^^^^^^^^^^ PK table & column
);

INSERT INTO people_new SELECT * FROM people;
```

Now try inserting that USC row again:

```sql
sqlite> INSERT INTO people_new VALUES ("remy", "...", 345, "USC");
Parse error: foreign key mismatch - "people_new" referencing "employers"
```

We see that SQLite stops us since the foreign key value must reference an existing primary key value. Similarly, we can't add another employer tuple with `name="UCLA"`, even if the tuple differs in other columns. This is because `name` is a primary key. Primary keys are **unique** by design (you can think of it as implicitly having the `UNIQUE` constraint):

```sql
sqlite> INSERT INTO employers_new VALUES ("UCLA", "Mars");
Runtime error: UNIQUE constraint failed: employers_new.name (19)
```

### SQL Challenge: Joins & Keys

**QUESTION:** What happens if you join a table $T$ with itself on its PK?

You always get back the same number of rows as $T$. The columns simply get repeated. It's as if you made a copy of $T$ and then stuck it to the side of $T$.

| T1.col1 | T1.col2 | T1.col3 | T2.col1 | T2.col2 | T2.col3 |
| ------- | ------- | ------- | ------- | ------- | ------- |
| A       | B       | C       | A       | B       | C       |
| ...     | ...     | ...     | ...     | ...     | ...     |


**QUESTION:** How do we check that a FK is valid with SQL?

As usual, first break the problem down into easier queries. We can figure out how to combine them back together using either sub-queries or operators later.

This problem is equivalent to seeing if all the values in the FK are also values in the referenced PK. We can first join the tables on the FK=PK to get tuples where FK values *are* valid (they equal some PK value). The FK values that are *invalid*, if any, would then be any of them that do *not* appear in this joined table.

First let's insert some blasphemous data again so we can see it show up as an example of a FK *not* being valid (use the original `people` table since it doesn't have the SQL-enforced `FOREIGN KEY` constraint):

```sql
INSERT INTO people VALUES ("remy", "...", 345, "USC");
```

The join to find the valid FK values:

```sql
SELECT people.job
FROM people, employers
WHERE people.job = employers.name;
```
```console
┌──────┐
│ job  │
├──────┤
│ UCLA │
│ UCLA │
│ UCLA │
│ UW   │
│ UW   │
│ UW   │
└──────┘
```

We can then leverage the `EXCEPT` operator, which acts like **set difference** (NOTE: yes, *set* difference&mdash;it takes unique tuples). We want the jobs that appear in our FK (`people.job`) that *don't* appear in our above join:

```sql
SELECT job FROM people
EXCEPT
SELECT people.job
FROM people, employers
WHERE people.job = employers.name;
```
```console
┌─────┐
│ job │
├─────┤
│ USC │
└─────┘
```

And voila! We see the `job` that violates what would've been a foreign key constraint. Remove the USC tuple and try again and you should get back nothing:

```console
sqlite> DELETE FROM people WHERE job="USC";
sqlite> SELECT job FROM people
   ...> EXCEPT
   ...> SELECT people.job
   ...> FROM people, employers
   ...> WHERE people.job = employers.name;
sqlite>
```

> [!NOTE]
>
> My answer differs from Professor's version on the slides, where he uses the `people.name` column in both queries instead. This actually wouldn't give the correct value for the particular example we used since the USC tuple has `name=remy`, and since `remy` already exists in other tuples and `EXCEPT` takes the *set* difference, we get back an empty table when we should've gotten a row back for USC.

### SQL Challenge: People with Both Cats and Dogs?

Consider this table in the DB file:

```console
sqlite> SELECT * FROM pets;
┌──────┬───────────────┬─────┬─────────┬──────┬────────┐
│ name │     breed     │ age │ origin  │ kind │ person │
├──────┼───────────────┼─────┼─────────┼──────┼────────┤
│ casa │ tabby         │ 8   │ seattle │ cat  │ remy   │
│ kira │ tuxedo        │ 6   │ hawaii  │ cat  │ remy   │
│ toby │ border collie │ 17  │ seattle │ dog  │ remy   │
│ maya │ husky         │ 10  │ LA      │ dog  │ sam    │
└──────┴───────────────┴─────┴─────────┴──────┴────────┘
```

We want to be able to check that some person owns both a cat and dog. In table terms, we want `person`s that have tuples with `kind="cat"` or `kind="dog"`.

Once again, I'll lay out the initial attempts we did during lecture as an opportunity to introduce more SQL concepts/pitfalls.

Attempt 1:

```sql
SELECT pets.person FROM pets
WHERE pets.kind="cat" AND pets.kind="dog";
```

That won't work because that looks for tuples where the `kind` column has simultaneously the value `cat` and `dog`, which is impossible. You get an empty table back.

```sql
SELECT pets.person FROM pets
WHERE pets.kind="cat" OR pets.kind="dog";
```

This filters for rows whose `kind` has *either* `cat` or `dog`, which isn't what we want either.

Okay, let's take a step back. We know how to *individually* query for people that have cats or dogs:

<table>
<tr>
<td>

```sql
SELECT pets.person FROM pets
WHERE pets.kind="cat";
```

```console
┌────────┐
│ person │
├────────┤
│ remy   │
│ remy   │
└────────┘
```

</td>
<td>

```sql
SELECT pets.person FROM pets
WHERE pets.kind="dog";
```

```console
┌────────┐
│ person │
├────────┤
│ remy   │
│ sam    │
└────────┘
```

</td>
</tr>
</table>

We realize that we just want the *intersection* of the two resulting tables. Recall from the first lecture that we can just use the `INTERSECT` operator between two query expressions:

```sql
SELECT pets.person FROM pets
WHERE pets.kind="cat"
INTERSECT
SELECT pets.person FROM pets
WHERE pets.kind="dog";
```
```console
┌────────┐
│ person │
├────────┤
│ remy   │
└────────┘
```

As a bonus, `INTERSECT` automatically removes duplicates since it takes the **set intersection**.

### SQL Challenge: People with Both Cats and Dogs? ([Even Further Beyond](https://www.youtube.com/watch?v=8TGalu36BHA))

The above answer is valid, but there's a way to do this in *one* query using a **join**. Instead of using two "copies" of the table with two separate queries, we can just perform a self-join on `person`. [Recall from earlier](#sql-challenge-joins--keys) that this would return something akin to two copies concatenated side-by-side.

> [!NOTE]
>
> Erm ackshually 🤓: Here `person` *isn't* a PK (it isn't even `UNIQUE`, by inspection), so the self-join would actually return rows for the matches between *each* tuple with the same `person`. But no biggie, we can just `SELECT DISTINCT` to remove duplicates like always.

Remember earlier we had the problem of checking `kind="cat" AND kind="dog"` not doing what we wanted because we were conditioning on the *same* column&mdash;now we have *two* columns for each tuple to play with. We can check that one side's `kind="cat"` and the other side's `kind="dog"` because that would correspond to a tuple for a `person` who had both a cat and dog!

```sql
SELECT DISTINCT p1.person
FROM pets AS p1
JOIN pets AS p2
ON p1.person=p2.person
WHERE p1.kind="cat" AND p2.kind="dog";
```

Or, if you prefer the implicit join style:

```sql
SELECT DISTINCT p1.person
FROM pets AS p1, pets AS p2
WHERE p1.person = p2.person
AND p1.kind="cat" AND p2.kind="dog";
```

Note that since `person` isn't `UNIQUE`, we use `SELECT DISTINCT` to remove duplicates that arise during the join process.

### SQL Challenge: Pet Kind with Average Age > 10?

Consider the same `pets` table:

```console
sqlite> SELECT * FROM pets;
┌──────┬───────────────┬─────┬─────────┬──────┬────────┐
│ name │     breed     │ age │ origin  │ kind │ person │
├──────┼───────────────┼─────┼─────────┼──────┼────────┤
│ casa │ tabby         │ 8   │ seattle │ cat  │ remy   │
│ kira │ tuxedo        │ 6   │ hawaii  │ cat  │ remy   │
│ toby │ border collie │ 17  │ seattle │ dog  │ remy   │
│ maya │ husky         │ 10  │ LA      │ dog  │ sam    │
└──────┴───────────────┴─────┴─────────┴──────┴────────┘
```

We want the `kind`s where the average `age` within each is > 10. Some things should be triggering alarm bells right now: we want an "average", where that average is applied "within each" group of something (`kind` in this case). `GROUP BY`! We want to `GROUP BY kind`, and then do something with the grouped tuples.

There's something different this time. Notice that our aggregate function (`AVG()`) is actually part of our conditioning too now, *not* just the projection. We need the value returned from `AVG()` to determine which rows to discard.

It's tempting to do something like:

```sql
SELECT kind, AVG(age) FROM pets
WHERE AVG(age) > 10
GROUP BY kind;
```

However, this is illegal in SQL:

```console
Parse error: misuse of aggregate: AVG()
  SELECT kind, AVG(age) FROM pets WHERE AVG(age) > 10 GROUP BY kind;
                          error here ---^
```

This doesn't work because of the order SQL executes queries. Recall that the `WHERE` clause is the "row filter". At this point in the query, SQL is still determining which rows to include at all in the first place&mdash;how would it know the `AVG()` of all of the rows before that? There *is* a way to achieve something similar using a new clause, `HAVING`. More on that later.

Okay, when in doubt, you should know by now: break the problem down and solve the parts individually with easier queries. Let's start with something more familiar: how do we just find the average age of each kind of pet?

```sql
SELECT kind, AVG(age)
FROM pets
GROUP BY kind;
```
```console
┌──────┬──────────┐
│ kind │ AVG(age) │
├──────┼──────────┤
│ cat  │ 7.0      │
│ dog  │ 13.5     │
└──────┴──────────┘
```

Since only `dog` has an average `age` over 10, we know that in our final answer, we should only see something like this:

```console
┌──────┬──────────┐
│ kind │ AVG(age) │
├──────┼──────────┤
│ dog  │ 13.5     │
└──────┴──────────┘
```

Our problem has become just a matter of filtering from our intermediate table `WHERE` the average is over 10. Recall how to save the result of a query to another table:

```sql
CREATE TABLE averages AS
SELECT kind, AVG(age) AS a
FROM pets
GROUP BY kind;

-- Now we just filter for over 10:
SELECT * FROM averages
WHERE averages.a > 10.0;
```

But we want to be cool and solve it using one main query, so we'll use a sub-query:

```sql
WITH averages AS (SELECT kind, AVG(age) AS a FROM pets GROUP BY kind)
SELECT * FROM averages WHERE averages.a > 10.0;
```
```console
┌──────┬──────┐
│ kind │  a   │
├──────┼──────┤
│ dog  │ 13.5 │
└──────┴──────┘
```

### SQL Challenge: Pet Kind with Average Age > 10? ([Even Further Beyond](https://www.youtube.com/watch?v=8TGalu36BHA))

If we want to apply an aggregate function as part of our *conditioning*, we need to put it in another clause, the `HAVING` clause.

`HAVING` allows us to specify a condition with an aggregate function after we `GROUP BY` a column(s). In this case, we want to find `kind`s whose aggregate *something* satisfies some *condition*, so we `GROUP BY kind` and then `HAVING` our condition, in this case `AVG(age) > 10`:

```sql
SELECT kind, AVG(age) FROM pets
GROUP BY kind
HAVING AVG(age) > 10;
```
```console
┌──────┬──────────┐
│ kind │ AVG(age) │
├──────┼──────────┤
│ dog  │ 13.5     │
└──────┴──────────┘
```

Clean!

## Creating Temporary Tables in SQL

There are multiple ways to "create" tables from existing tables in SQL. The following SQL constructs are analogous to some familiar Python concepts:

| SQL                        | Python          |
| -------------------------- | --------------- |
| `WITH`                     | Local variable  |
| `CREATE TABLE`             | Global variable |
| `CREATE VIEW`              | Helper function |
| `CREATE MATERIALIZED VIEW` | N/A             |

- `WITH` is the "cleaner" version of a sub-query like we learned. We can execute some query expression and then alias the result into some table name that can be referenced elsewhere in (local to) the query.
- `CREATE TABLE` actually creates a new table physically on disk with all of the data from the query result. We likened this to variable assignment before. Because it's a full-fledged table, it's like a *global* variable since we can see it in `.tables`, reference it in other queries, etc.
- `CREATE VIEW` does *not* actually create a new table on disk but rather a **virtual table**. You can think of it as a `SELECT` query wrapped under a helper function, which will re-run the query every time the view is referenced again. This is useful if you want to abstract some complex query into something akin to an intermediate table for use in a larger query, just like how you would extract code out of a large function into a helper function in other programming languages.
- `MATERIALIZED VIEW` is like `CREATE VIEW` but the data is actually stored physically on disk, so the query does not need to be re-run every time it's referenced. Note that SQLite doesn't actually natively support materialized views.

### SQL Challenge: Average Age of Each Kind (Without `GROUP BY`)

We know how to find the average age of a *given*, single `kind`:

```sql
SELECT kind, AVG(age)
FROM pets
WHERE pets.kind="cat";
```
```console
┌──────┬──────────┐
│ kind │ AVG(age) │
├──────┼──────────┤
│ cat  │ 7.0      │
└──────┴──────────┘
```

How do we generalize this to *all* `kind`s at once? This is a bit tricky. We want to somehow:

1. Run some aggregate function `AVG()`...
2. On all `age` values of rows whose...
3. `kind` matches the `kind` in another column within the output table itself:

```
┌──────┬────────────────────────────────────────────┐
│ kind │ ???                                        │
├──────┼────────────────────────────────────────────┤
│ cat <-- 7.0 (AVG(age) of all rows with kind=cat)  │
│ dog <-- 13.5 (avg(age) of all rows with kind=dog) │
└──────┴────────────────────────────────────────────┘
```

It turns out we can actually run a **nested sub-query** *within a `SELECT` column* itself!

```sql
SELECT DISTINCT
  p1.kind,
  (SELECT AVG(p2.age) FROM pets AS p2 WHERE p2.kind=p1.kind)
  -- ^ Inner query is directly used as an output column of outer query!
FROM pets as p1;
```
```console
┌──────┬────────────────────────────────────────────────────────────┐
│ kind │ (SELECT AVG(p2.age) FROM pets AS p2 WHERE p2.kind=p1.kind) │
├──────┼────────────────────────────────────────────────────────────┤
│ cat  │ 7.0                                                        │
│ dog  │ 13.5                                                       │
└──────┴────────────────────────────────────────────────────────────┘
```

Consider what the inner query is doing:

```sql
(SELECT AVG(p2.age) FROM pets AS p2 WHERE p2.kind=p1.kind)
```

1. We're selecting rows from `pets`...
2. Where the `kind` is equal to the `kind` in the outer, other output column...
3. And taking the `AVG(age)` of all those rows.

Exactly what we wanted! This is like a cursed way to simulate `GROUP BY`.

> [!NOTE]
>
> Obviously, you wouldn't do this in a real query. This is just a pedagogical example to introduce nested sub-queries (...and also show how there's *usually* a better way to do things that can avoid nested sub-queries in the first place&mdash;they tend to be nasty and unreadable).

## `NULL`: The Problem Child of SQL

### SQL Challenges ("Why Is SQL Like This?" Edition)

**QUESTION:** Does this return `R`?

```sql
SELECT r1.x
FROM R as r1, R as r2
WHERE r1.x = r2.x;
```

Intuitively, when you join a table with itself and take the projection of just one side's columns, you should just get back the original table. However, this isn't necessarily true because of possible **duplicates**. Due to the "nested loop semantics" of how joins work, each copy will contribute its own match to the result table, so we get even more duplicates!

Try it for yourself:

```console
sqlite> CREATE TABLE s(x int);
sqlite> INSERT INTO s VALUES (1), (1);
sqlite> SELECT * FROM s;
┌───┐
│ x │
├───┤
│ 1 │
│ 1 │
└───┘
sqlite> SELECT s1.x FROM s AS s1, s AS s2 WHERE s1.x=s2.x;
┌───┐
│ x │
├───┤
│ 1 │
│ 1 │
│ 1 │
│ 1 │
└───┘
```

**QUESTION:** Does this return R?

```sql
SELECT *
FROM R
WHERE R.x = R.x;
```

Intuitively, `R.x = R.x` should just evaluate to `true`, making the `WHERE` clause degenerate and making our query just return the original table. However, `R.x = R.x` is *not* necessarily `true`, due to `NULL` values.

Consider this table in the DB file:

```console
sqlite> SELECT * FROM r;
┌───┐
│ x │
├───┤
│   │
└───┘
sqlite> SELECT COUNT(*) FROM r;
┌──────────┐
│ COUNT(*) │
├──────────┤
│ 1        │
└──────────┘
```

```console
sqlite> SELECT COUNT(*) FROM r WHERE r.x=r.x;
┌──────────┐
│ COUNT(*) │
├──────────┤
│ 0        │
└──────────┘
```

Because `r` has a `NULL` value, joining on `r.x=r.x` doesn't actually contribute that row to the output table! This behavior is [explained later](#operators-with-null).

**QUESTION:** Does this return R?

```sql
SELECT *
FROM R
WHERE R.x = R.x
OR R.x <> R.x;
```

Intuitively, the disjunction of a statement and its negation should be able to just be replaced with `true`. But once again, `NULL` causes issues:

```console
sqlite> SELECT * FROM r WHERE true;
┌───┐
│ x │
├───┤
│   │
└───┘
sqlite> SELECT * FROM r WHERE r.x=r.x OR r.x<>r.x;
sqlite>
```

**QUESTION:** Does this return R?

```sql
SELECT *
FROM R
WHERE null = null;
```

You can probably guess it by now lol.

```console
sqlite> SELECT * FROM r WHERE null=null;
sqlite>
```

In fact, none of these return anything:

```sql
SELECT *
FROM R
WHERE null <> null;

SELECT *
FROM R
WHERE NOT null <> null;
```

> `NULL` "transcends truth". *&mdash;Professor Remy Wang*

### Values in SQL

To understand why `NULL` is being a pain in the `A$$`, we need to first understand how values work in SQL.

There are two kinds of values in SQL:

- **Data values**
- **Logical values**

**Data values** are like numbers, dates, strings, etc.&mdash;the stuff you're actually interested in storing in your tables. A special data value is the `NULL` value, representing "nothingness" or "unknown".

**Logical values** are your familiar booleans true and false, used by the engine in intermediate calculations. A special logical value is the `UNKNOWN` value, representing "unknown". You can think of it as just the logical counterpart to `NULL` (or conversely, `NULL` is the data counterpart to `UNKNOWN`). Keeping track of *type* (data vs. logical) is important&mdash;it helps you remember when an expression evaluates to `NULL` vs. `UNKNOWN`.

> [!CAUTION]
>
> `UNKNOWN` doesn't actually exist in SQLite specifically. Where you would see `UNKNOWN`, you would see `NULL` instead.

### Operators in SQL

There are 3 kinds of operators in SQL:

**Data operators**:

```
      1             +             2
<data value> <data operator> <data value>
```

The entire expression evaluates to itself another **data value**.

**Predicates**:

```
      1               <             2
<data value> <predicate operator> <data value>
```

The entire expression evaluates to a **logical value**.

**Logical connectives**:

```
      true              AND               false
<logical value> <logical operator> <logical value>
```

The entire expression evaluates to itself another **logical value**.

Conveniently summarized:

| Operator kind | Example    | Input -> Output |
| ------------- | ---------- | --------------- |
| Data          | + - \* /   | data -> data    |
| Predicate     | > < =      | data -> logic   |
| Logical       | AND OR NOT | logic -> logic  |

Note that you can't go from logic -> data.

### Operators with `NULL`

These operators all behave as you would expect if `NULL` is *not* involved. When `NULL` *is* involved, there are special rules:

| Operator Kind | Example    | Output when `NULL` is >= 1 of the operands |
| ------------- | ---------- | ------------------------------------------ |
| Data          | + - \* /   | `NULL`                                     |
| Predicate     | > < =      | `UNKNOWN`                                  |
| Logical       | AND OR NOT | [3 Valued Logic](#3-valued-logic)          |

For **data**, it's intuitive: if either operand is `NULL` (it's "missing" or "unknown"), the result should also be "missing" or "unknown", so the expression evaluates to `NULL` as well.

For **predicate**, it's also intuitive: the same reason as for data, but because we're dealing with logical types, we have the logical equivalent of `NULL`, `UNKNOWN`.

> [!IMPORTANT]
>
> There is one exception for **predicates**: To explicitly check if `x` is `NULL`, you use: `x IS NULL`, which will return T/F on whether `x` is indeed `NULL`. It *won't* return `NULL` despite the RHS being `NULL`.

> [!IMPORTANT]
>
> Remember that `UNKNOWN` doesn't actually exist in SQLite specifically. What *would* be `UNKNOWN` is automatically converted to `NULL` when used as input to a data operator/predicate.

For **logical**, we apply **3 valued logic**:

### 3 Valued Logic

| AND     | true        | false     | UNKNOWN     |
| ------- | ----------- | --------- | ----------- |
| true    | true        | false     | **UNKNOWN** |
| false   | false       | false     | **false**   |
| UNKNOWN | **UNKNOWN** | **false** | **UNKNOWN** |

| OR      | true     | false       | UNKNOWN     |
| ------- | -------- | ----------- | ----------- |
| true    | true     | true        | **true**    |
| false   | true     | false       | **UNKNOWN** |
| UNKNOWN | **true** | **UNKNOWN** | **UNKNOWN** |

| NOT | true  | false | UNKNOWN     |
| --- | ----- | ----- | ----------- |
|     | false | true  | **UNKNOWN** |

At first glance, it looks like a lot of combinations to memorize. However, all of these are actually quite intuitive if you reason about them using Boolean algebra. The key idea comes down to whether the overall expression can *already be deduced from one operand*, or we actually need the `UNKNOWN` operand (and thus the overall expression continues to be "unknown").

- **True AND UNKNOWN**? The value of the unknown operand would've determined the value of the result, so the overall result is unknown.
- **False AND UNKNOWN**? False because it's a conjunction. It doesn't matter what the other operand is anyway.
- **True OR UNKNOWN**? True because it's a disjunction. It doesn't matter what the other operand is anyway.
- **False OR UNKNOWN**? The value of the unknown operand would've determined the value of the result, so the overall result is unknown.
- **UNKNOWN AND UNKNOWN**? Unknown. Neither of the operands are known.
- **UNKNOWN OR UNKNOWN**? Unknown. Neither of the operands are known.
-  **NOT UNKNOWN**? The value of the unknown operand would've determined the value of the result, so the overall result is unknown.

### Revisiting Examples

When in doubt, use [the `NULL` rules from earlier](#operators-with-null) and [3 valued logic](#3-valued-logic) to determine what the `WHERE` clause is actually saying and how it would affect the queries.

```sql
SELECT *
FROM R
WHERE R.x=R.x; -- NULL=NULL => UNKNOWN
```

Where `R.x` is [`NULL`, the `WHERE` clause evaluates to `UNKNOWN`](#operators-with-null), so that row is *excluded*. The return table is *not* necessarily the same as `R`.

```sql
SELECT *
FROM R
WHERE R.x = R.x
OR R.x <> R.x; -- NULL <> NULL => UNKNOWN
```

Same as above.

```sql
SELECT *
FROM R
WHERE null = null; -- NULL=NULL => UNKNOWN
```

Same as above. It's just explicit this time.

```sql
SELECT *
FROM R
WHERE null <> null; -- NULL <> NULL => UNKNOWN
```

Same as above. It's just explicit this time.

```sql
SELECT *
FROM R
-- NULL <> NULL => UNKNOWN
-- NOT UNKNOWN => still UNKNOWN
WHERE NOT null <> null;
```

This time we have a nested operation. Just follow through with the rules and you'll find when `WHERE` evaluates to `UNKNOWN`.
