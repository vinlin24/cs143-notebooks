# 2025-05-06/Lecture 11: Isolation Levels & Application Programming

## Isolation Levels

### Motivation

Previous lecture, we built up to **strict 2PL** as some ultimate solution to producing **conflict-serializable** schedules which also support **rollbacks**. However, strict 2PL is kind of expensive because it's quite restriction.

In real practice, full isolation is, while ensuring 100% correctness, too restrictive in terms of performance. Today we look at *generalizing* the concept of "isolation" to a hierarchy of **isolation levels**, which have varying degrees of strictness.

On the spectrum of **fast** (but possibly **incorrect**) to **slow** (but more **correct**):

1. None (chaos! 😱)
2. [Read Uncommitted](#isolation-level-1-read-uncommitted)
3. [Read Committed](#isolation-level-2-read-committed)
4. [Repeatable Read](#isolation-level-3-repeatable-read)
5. [Serializable](#isolation-level-4-serializable)

Different isolation levels solve one or more of certain types of anomalies, such as:

- [Dirty read/inconsistent read](#dirtyinconsistent-read)
- [Lost update](#lost-update)
- [Unrepeatable read](#unrepeatable-read)

### Dirty/Inconsistent Read

> [!IMPORTANT]
>
> A **dirty read**/**inconsistent read** is when reads within one transaction reflect updates from another uncommitted transaction.

This is also known as a [**write-read conflict**](https://en.wikipedia.org/wiki/Write–read_conflict).

For example, consider a scenario where a manager wants to balance project budgets. That is, they want to subtract some amount from a project and re-distribute it to another project(s).

This might sound familiar because it's a similar situation in Lecture 8 during one of the in-class activities where a volunteer was asked to do the same thing but with fake quiz scores, used to motivate the concept of **consistency** (ACID). As you recall, the **invariant** then is similar to the situation here: the *total* money should be the same.

Suppose the CEO comes along and wants to check the company's current balance in a series of queries (e.g. `SELECT SUM(budget) ...`). If we're lucky, the CEO sees nothing wrong:

1. CEO: checks company balance, sees it as some value `X`.
2. Manager: -$10M from project A.
3. Manager: +$7M to project B.
4. Manager: +$3M to project C.
5. CEO: checks company balance again, still sees it as `X`.

But consider if the CEO happened to check before the manager's transaction completed:

1. CEO: checks company balance, sees it as some value `X`.
2. Manager: -$10M from project A.
3. CEO: checks company balance again, sees it as `X - 10M`.
4. Manager: +$7M to project B.
5. Manager: +$3M to project C.

The CEO thinks the company lost $10M out of nowhere!

### Lost Update

> [!IMPORTANT]
>
> A **lost update** is an update *overwritten* by another transaction.

This is also known as a [**write-write conflict**](https://en.wikipedia.org/wiki/Write–write_conflict).

Example where this doesn't happen:

Suppose an initial state of `A = B = 100`. An **invariant** is that `A + B = 200`.

| $T_1$     | $T_2$     |
| --------- | --------- |
| W(A, 200) |           |
| W(B, 0)   |           |
|           | W(B, 200) |
|           | W(A, 0)   |

The final state is that `A = 0, B = 200`. The invariant that `A + B = 200` is intact.

An example where a **lost update** occurs:

Suppose the same initial state of `A = B = 100`:

| $T_1$     | $T_2$     |
| --------- | --------- |
| W(A, 200) |           |
|           | W(B, 200) |
| W(B, 0)   |           |
|           | W(A, 0)   |

The final state is that `A = B = 0`. The invariant that `A + B = 200` is violated!

Because writing `0` was the last thing to happen to both data items, we "lost an update". It's like `200` just vanished, similar to what [the CEO observed earlier](#dirtyinconsistent-read). **Consistency** is violated once again.

**ASIDE (Review):** This schedule isn't conflict-serializable. By inspection, you can't swap non-conflicting operations to make it serial (convince yourself why).

### Unrepeatable Read

> [!IMPORTANT]
>
> **Unrepeatable read** describes the situation where two reads give different results.

You can also think of this like a generalization/special case of [**inconsistent read**](#dirtyinconsistent-read).

Consider this schedule:

| $T_1$   | $T_2$ |           |
| ------- | ----- | --------- |
|         | R(A)  | A = 100   |
| W(A, 0) |       |           |
|         | R(A)  | **A = 0** |

We have two back-to-back reads in the same transaction ($T_2$) that produce different results. As the application writer, this isn't something you would expect because there were no writes between *your* reads.

**ASIDE (Review):** Also note that this schedule is *not* conflict-serializable.

### Isolation Level 1: Read Uncommitted

> [!IMPORTANT]
>
> A schedule in **read uncommitted** isolation level uses:
> - **Strict 2PL** for **writes**.
> - No locks at all for **reads**.

When is this isolation level suitable?

- When you require very fast reads (there are no locks slowing down reads).
- When you have few/no writes (so the accuracy problem is not a common case).
- When read accuracy is not critical (so in the slight chance reads aren't accurate, it's still not a big deal).

What does this isolation level solve?

- Because writes use strict 2PL, it solves the [**lost update** problem](#lost-update). You can't have interleaved writes on the same data item.

What does this isolation level *not* solve?

- You can still get [**dirty reads**](#dirtyinconsistent-read). Because reads do not claim locks, nothing stops writes from coming in and changing a data item in between two reads on that same data item. For example: $R_1(A) \to L_2(A), W_2(A), U_2(A) \to R_1(A)$ may result in the two reads on $A$ producing different results.

### Isolation Level 2: Read Committed

> [!IMPORTANT]
>
> A schedule in **read committed** isolation level uses:
> - **Strict 2PL** for writes.
> - On-demand read locks (*not* 2PL!). That is, every read is wrapped with a lock & unlock i.e. $L(X), R(X), U(X)$.

What does this isolation level solve?

- There are no more [**dirty reads**](#dirtyinconsistent-read). Because reads now need to lock their data item, if a write is currently in progress (and thus possesses its lock), a read cannot proceed until that write is both finished and committed (because of strict 2PL):

| $T_1$        | $T_2$          |
| ------------ | -------------- |
| W(A, 0)      |                |
|              | ~~L(A), R(A)~~ |
| COMMIT, U(A) |                |

Thus, we cannot read uncommitted writes from another transaction.

What does this isolation level *not* solve?

- It is still possible to have [**unrepeatable reads**](#unrepeatable-read).

To understand why, consider this example:

| $T_1$            | $T_2$            |
| ---------------- | ---------------- |
|                  | L(A), R(A), U(A) |
| L(A), W(A), U(A) |                  |
|                  | L(A), R(A), U(A) |

Firstly, is this schedule conflict-serializable? No. There are conflicts across the operations, so they cannot swap. We cannot swap them into a serial schedule.

The write still satisfies strict 2PL since we only have one write, and we lock before it and unlock after (and presumably commit right after).

For the reads, we lock on-demand (like wrapping the individual reads).

Thus, this example satisfies the constraints of the **read committed** isolation level, but because we have a write in between the two reads, we can still have **unrepeatable reads**.

When is this isolation level suitable?

- When we just need to guarantee that the read result is valid at *some* point (not necessarily then and there).
- As a concrete example, it's useful for online shops. Recall from your personal experience where you may have seen that an item is in stock, which lets you try to order it (i.e. attempt a write on the database), but *then* it shows that that item isn't available anymore. That is, the unrepeatable read is acceptable.

### Isolation Level 3: Repeatable Read

> [!IMPORTANT]
>
> A schedule in the **repeatable read** isolation level uses **strict 2PL** for both write and read locks.
>
> Such a schedule is **conflict-serializable**... but *not* **serializable**!

> [!CAUTION]
>
> We lied lol. Previously, we introduced **conflict-serializable** as a *subset* of **serializable**, but there was an implicit assumption behind when that's true. When this is *not* necessarily true is explained below regarding [the phantom read problem](#the-phantom-read-problem).

To clarify, what does it mean for both "strict 2PL write locks" and "strict 2PL read locks"?

- Case I: Strict 2PL just within write locks, as well as strict 2PL just within read locks (interleaved still allowed between the two sequences).
- Case II: Strict 2PL write & read locks together.

<!-- Consider this schedule, does this satisfy either?

| $T_1$            | $T_2$            |
| ---------------- | ---------------- |
|                  | L(A), R(A), U(A) |
| L(A), W(A), U(A) |                  |
|                  | L(A), R(A), U(A) |

No, there's no 2PL even if only within just reads. -->

Consider this schedule:

| $T_1$                    | $T_2$                    | $T_3$                    |
| ------------------------ | ------------------------ | ------------------------ |
|                          | L(A), R(A), U(A), COMMIT |                          |
| L(A), W(A), U(A), COMMIT |                          |                          |
|                          |                          | L(A), W(A), U(A), COMMIT |

Strict 2PL is upheld independently for reads: all locks occur before any unlocks in $T_2$, and we unlock at commit. Strict 2PL is upheld independently for writes too: all locks occur before any unlocks in both $T_1$ and $T_3$, and they unlock at commit. Strict 2PL is also upheld when considering reads and writes *together*. It turns out that for *strict* 2PL, it doesn't actually matter if we take the locking to mean Case I or Case II, but for the sake of consistency (no pun intended), we'll refer to Case II (locking *together*).

> [!NOTE]
>
> We can use strict 2PL with exclusive writes & shared read locks together and still guarantee conflict-serializable schedules. The proof is similar to the one in last lecture and left as an exercise.

### The Phantom Read Problem

> [!IMPORTANT]
>
> A **phantom read** is when a subsequent read on a table returns a different number of rows than it did previously.

Consider this timeline:

| SQL        | $T_1$            | $T_2$ | SQL      |
| ---------- | ---------------- | ----- | -------- |
| `SELECT *` | R(A), R(B)       |       |          |
|            |                  | W(C)  | `INSERT` |
| `SELECT *` | R(A), R(B), R(C) |       |          |

Suppose some table has data items `A` and `B`. The first `SELECT *` dutifully returns everything from the table, so we `R(A)` and `R(B)`.

Then, another transactions wants to *insert* another data item `C` into the table. Because `C` didn't even exist before, it doesn't contest any lock and is able to proceed with insertion.

The second `SELECT *` wants to touch all data items `A`, `B`, and `C`, so it grabs a lock for all data items, including the new item `C`, and dutifully `R(A)`, `R(B)`, and `R(C)`.

Our second query returned rows that weren't previously there&mdash;**phantom rows**. The symmetric case also happens when there's a *deletion* between the reads, in which case the subsequent query exhibits *missing* rows.

Phantom reads are still possible under strict 2PL! Thus, revisiting our previous definition of conflict-serializable:

> [!IMPORTANT]
>
> Conflict-serializable implies serializable **assuming there are no inserts**.

What gives? Why do we call this isolation level "repeatable read" when it looks like our reads aren't repeatable after all?

Repeatable read guarantees that your reads are repeatable assuming you're accessing the *same* data items. If repeatable read spans *multiple* data items, this definition no longer applies. This may also make it clearer:

> [!TIP]
>
> How is a **phantom read** different from an [**unrepeatable read**](#unrepeatable-read)? An unrepeatable read is with respect to a specific data item (row). A phantom read only applies to a *range* of records (like a whole table).
>
> Phantom reads are orthogonal to unrepeatable reads. Unrepeatable reads are due to changes to values *within* the rows. Phantom reads are due to changes in the *number* of rows itself. These two anomalies can happen at the same time, independently (if isolation level permits).

### Isolation Level 4: Serializable

To solve phantom reads (and thus also complete our coverage over all previously discussed anomalies), we lock entire *tables*.

Consider this example:

|          | $T_1$         | $T_2$          |
| -------- | ------------- | -------------- |
| SELECT * | L(T), R(T)    |                |
|          |               | ~~L(T), W(C)~~ |
| SELECT * | R(T), C, U(T) |                |

The insertion of `C` into the table `T` is denied because the whole table `T` is locked. No more phantom reads.

> [!IMPORTANT]
>
> A schedule in the **serializable** isolation level uses **strict 2PL** for both write and read locks (combined), and locks act on whole tables.

### Anomalies & Isolation Levels Summary

- ❎ means this anomaly *cannot* happen under this isolation level.
- ⚠️ means this anomaly *may* occur under this isolation level.

| Isolation Level  | Lost Update | Dirty Read | Non-Repeatable Read | Phantom Read |
| ---------------- | ----------- | ---------- | ------------------- | ------------ |
| Read Uncommitted | ❎           | ⚠️          | ⚠️                   | ⚠️            |
| Read Committed   | ❎           | ❎          | ⚠️                   | ⚠️            |
| Repeatable Read  | ❎           | ❎          | ❎                   | ⚠️            |
| Serializable     | ❎           | ❎          | ❎                   | ❎            |

## Security

### SQL Injection

[Relevant XKCD.](https://xkcd.com/327/) Little Bobby Tables is a classic! You'll quote this one to all your friends.

Basically, this vulnerability comes from the fact that when we use naive string concatenation to build queries, we're effectively giving the user the ability to hijack the query and run unintended SQL commands (hence, SQL *injection*). The injection is done by first "completing" the intended placeholder (for example, a username in a `WHERE` clause), ending that query, and then writing an arbitrary query of your design right after it.

For example, consider this query template:

```sql
SELECT * FROM user
WHERE name = 'x'
```

How can we use SQL injection to return *all* users? What should we input for `x`?

<details>
<summary>Expand for answer.</summary>

Let `x` be `' OR true --'`. The query becomes:

```sql
SELECT * FROM user
WHERE name = '' OR true --'
```

Note the trailing `--` to start a comment. This is to "discard" everything that came after the placeholder (e.g. the original closing quote) to make sure the query doesn't throw a syntax error.

Another solution is to do something like:

```sql
SELECT * FROM user
WHERE name = 'blah' OR name <> 'blah' --'
```

The catch if you do this is 3-valued logic involving `NULL` (see Lecture 3).

Another solution, using `UNION`:

```sql
SELECT * FROM user
WHERE name = 'blah' UNION SELECT * FROM user --'
```

This makes it so that it doesn't matter what the original query returns; we're getting *all* data from the second query anyway, which is then union'ed into the final result.

</details>

**Defenses to SQL Injection**

- Parameterized query
- Prepared statements
- Access control: different privilege levels, where more destructive operations are restricted to higher levels (e.g. only database admin can `DROP TABLE`, etc.).

Example of **parameterized queries** using Python & SQLite:

```python
import sqlite3

# Obtain a connection and cursor to the database.
# Don't worry about this for now.
con = sqlite3.connect("temp.db")
cur = con.cursor()

# Initialize the database with some dummy data.
cur.execute("insert into r values (1);")

#                      <-------query template------>  param
for row in cur.execute("select * from r where x = ?", (1,)):
    print(row)
```

Notice the query template has a `?` placeholder, which represents where the application is meant to interpolate specific data into. We then instantiate those parameters by passing in the values as a separate argument to the `execute()` API. This is what distinguishes parameterized queries from naive string concatenation. Instead of directly interpolating the values into the query string ourselves (e.g. using Python f-strings, etc.), we defer the interpolation to the database system, which can ensure that  the input is properly sanitized and prevent malicious injection.

**ASIDE:** An analogous attack on LLMs is **prompt injection**, where the user effectively "overrides" previous commands with their own prompt to make the LLM respond with unauthorized information/in an uncontrolled way.

### Passwords & Hashing

> [!IMPORTANT]
>
> *Never* store passwords in plain text. NEVER. Store database **hashes** instead, preferably using **salting** as well (both explained below).

Data leakage happens all the time. Always assume that *someone* might see your data at some point. Thus, we should make it so that *even* if an attacker gains access to the database, the information they *see* is useless to them. That is, we need to somehow obfuscate and anonymize the password information.

We do this with **hash functions**. A quick primer: a **hash function** is a function that takes in an input of arbitrary length and maps it to values of some other domain.

Hash functions have the following properties, desirable for password computation:

- **Deterministic**: $x = y \Longrightarrow h(x) = h(y)$
  - Given the same plaintext password, it computes the same hash value. This lets us compare login attempts with the stored source of truth.
- **Low collision**: $x \ne y \Longrightarrow h(x) \ne h(y)$ (that is, $\Pr(h(x) = h(y)) \approx 0$)
  - Different passwords should not hash to the same value. Otherwise, you could log in to someone else's account with your password!
- **Easy to compute**: $f(x) \in O(1)$
  - Important for performance. You don't want to keep your users waiting at the login page!
- But **hard to invert**: $f^{-1}(x) \in O(2^N)$
  - Even if an attacker obtains the hashed value, they shouldn't be able to easily compute what original plaintext password hashes to that value. If hashes weren't hard to invert, it kind of defeats the purpose of using them in the first place!

> [!NOTE]
>
> **ASIDE:** There was a lot of dispute in Quiz 10 regarding why a hash function for passwords should be "easy to compute". Intuitively, one would think that we should make it hard such that it hinders brute-force attacks. However, this is what salting already solves. Furthermore, another point (outside of the scope of this class) is actually a security concept: password checking should use [**constant-time programming**](https://www.chosenplaintext.ca/articles/beginners-guide-constant-time-cryptography.html) such as to not leak information about the password via [**side channel attacks**](https://en.wikipedia.org/wiki/Side-channel_attack). It's more difficult to do this if password computation is more expensive.

Instead of storing a table with raw passwords, store the **hashed** passwords.

In SQL, this looks something like:

```sql
SELECT * FROM user
WHERE name = 'x'
AND pw_hash = hash('y');
```

However, this alone is not enough. Another point of concern is that people are bad at passwords! They often reuse their password across different sites and/or use very commonly used passwords.

Since hash functions are **deterministic**, the same password will result in the same hash. We want this (otherwise we can't validate logins lol), but this also means the attacker can still gain information from a leaked database of hashed passwords. For example, if they see an often repeated password hash, it's likely a very popular password like `password`. This enables them to launch precomputed hash attacks like [rainbow table attacks](https://www.geeksforgeeks.org/understanding-rainbow-table-attack/) or [dictionary attacks](https://www.geeksforgeeks.org/what-is-a-dictionary-attack/).

The solution is to add a little something *before* computing the hash such that we produce different hash values despite being associated with the same original plaintext. This little something is called a **salt** (like "salting" your "hash" 😉). In pseudocode:

```python
salt = get_random()
salted_pw_hash = hash(pw, salt)
```

We store the password and salt together in the database.

Now, the attacker cannot immediately see if multiple users are using the same password. Furthermore, even if attackers reverse the hash, it would produce an entirely different plaintext password since the hash has the salt "baked into" it. This is also why storing the salt is fine. Its purpose isn't to be secret, just to ensure that same passwords hash to different values and defeat precomputed hash attacks/significantly slow down password cracking attempts.

The hash + salt approach is standard practice and solves both problems of using common passwords & reusing passwords across different sites!

### Privacy Laws

As application programmers, there is a lot more to keep in mind than just coding or database theory. We must ensure we are compliant with **privacy laws** too. Examples include:

- [HIPPA](https://en.wikipedia.org/wiki/Health_Insurance_Portability_and_Accountability_Act) for privacy of health data.
- [GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation) for EU law.
- [FERPA](https://en.wikipedia.org/wiki/Family_Educational_Rights_and_Privacy_Act) for privacy of education data.

A common practice is to "de-ID". That is, remove **personal identifiable information (PII)**.

However, a recent breakthrough by Harvard computer scientist [Latanya Sweeney](https://en.wikipedia.org/wiki/Latanya_Sweeney) actually showed that this isn't actually enough. This was demonstrated through the re-identification of Massachusetts governor [William Weld](https://en.wikipedia.org/wiki/Bill_Weld). The scenario was something like this:

*Public* data (approximately):

```
health_rc(zip, DoB, sex, diagnosis, procedure, ...)
voter(name, party, ..., zip, DoB, sex, ...)

Example tuple:
  ("Weld", "R.", ..., 12345, 02-30, M, ...)
```

Notice that there's no name, SSN, etc. (what you would traditionally consider PII). Furthermore, this is all *public* data.

There are only *three* people with the birthday `02-30`, Only of them were male (`M`). Of them, only one lived in the ZIP code `12345`. Sweeney was then able to use these constraints and join this data back with the health records to deduce the governor's health data! And all it costed her was $20 for the original public voter information.

![](assets/lec11/latanya_sweeney_join.png)

Another solution: use **aggregate** statistics. For example, a *histogram* of quiz stores. However, imagine if we watch this data change in real time as people submit quizzes? Imagine if you see a small bump in the histogram when you/your friend submits their quiz. Despite there being no PII in the collected data, the extra **context** still helps you deduce information.

Point: we need to care about **privacy in context**. De-ID'ing data alone is not enough. More formally, an analysis $\mathcal{M}$ on data should satisfy **differential privacy**:

![](assets/lec11/differential_privacy.png)

Mathematically, we quantify **indistinguishability** as a bounded ratio of probabilities:

$$\frac{\Pr(\mathcal{M}(D_1) \in O)}{\Pr(\mathcal{M}(D_2) \in O)} \le e^\varepsilon$$

Where the numerator is the probability of seeing **output** $O$ on **input** $D_1$ and the denominator is analogously the probability of seeing that same **output** $O$ on **input** $D_2$.
