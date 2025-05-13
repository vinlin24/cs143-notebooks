# 2024-05-08/Lecture 12: Hashing

### Query Plans (Revisited)

Let's revisit the *analytical* (as opposed to *transactional*) half of the course for a bit. Review Lecture 8 for the difference between the two.

Recall that you can use `EXPLAIN QUERY PLAN` in SQL to get a textual representation of its query plan:

```sh
# Example dataset, I don't have it.
sqlite3 imdb/imdb.db
```
```console
sqlite> .read 1a.sql
...
sqlite> EXPLAIN QUERY PLAN <paste query>
QUERY PLAN
|--SCAN mc
|--SEARCH ct USING INTEGER PRIMARY KEY (rowid=?)
|--SEARCH t USING INTEGER PRIMARY KEY (rowid=?)
|--BLOOM FILTER ON mi_idx (movie_id=?)
|--SEARCH mi_index USING AUTOMATIC COVERING INDEX (movie_id=?)
`--SEARCH it USING INTEGER PRIMARY KEY (rowid=?)
...
```

You might not recognize some concepts in this plan. We'll cover indexing later as well as [bloom filters](#bloom-filters).

We can also use the [Umbra online tool](https://umbra-db.com/interface/) to visualize in a web interface how a SQL query is transformed and optimized into an actual plan executed by the database. For this part of the lecture, we'll be using query 8 (Click **Load Query** and choose **8**).

Click on the query plan icon on the left sidebar (looks like a tree diagram). Notice on the sidebar we now have a bunch of steps you can click through.

**(1) Unoptimized plan**

This is like the raw SQL almost directly translated into something like an [abstract syntax tree (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree). It's like the initial step of translating SQL to its equivalent relational algebra as you may recall doing from hand earlier in the course. It's just that at this point, we have not applied any RA identities yet to optimize anything.

**(2) Expression simplification**

Maybe look at and factor some local expressions. Not much interesting. You can manually inspect the difference if you want.

**(3) Unnesting**

This corresponds to what we did in Lecture 6, where we were interested in un-nesting nested sub-queries.

You can select **Query 20** to see an example with nested sub-queries and explore how the tree changes.

**(4) Predicate pushdown**

Now we're getting into the real optimizations. We introduced predicate pushdown in Lecture 6 as a way to reduce the size of intermediate tables before joining them.

Notice how the selections now appear as low as possible (in the leaves).

What if we have a predicate that involves two tables? We push the predicate down to wherever they come together (the join node).
- If a selection predicate involves only one table, we just push it down to the leaf corresponding to that table.
- If it involves multiple tables, we push it down to where the tables are joined.

Predicate pushdown is an example of a **heuristic** optimization. We *assume* that it generally makes things faster.

**(5) Initial join tree**

The analyzer comes up with an initial ordering of the tree via some other heuristics.

Notice how as we go down these steps, numbers start appearing on some of the edges, and pictorially they have varying edge width. Those numbers are *estimates* of how many rows are matched from that subtree. Edges where there aren't any numbers means the database doesn't have a guess yet, and more will be estimated as we apply more steps.

**(6) Side-way information**

Another heuristic, and kind of complicated, so it's not covered in this class.

**(7) Operator reordering**

This is arranging **join order**. More on that later.

**(8) Early probing**

Another optimization.

**(9)  Common subtree elimination**

Up until now, notice how we have many `NOTSELECTEDYET` nodes. These correspond to points where we want to use some kind of join, but we don't know *which* algorithm to use yet. The decision to use a specific algorithm is called physical operator mapping. "Physical" as in contrast to "abstract" or "logical"&mdash;we know *what* it should compute, but we don't have a way *how* to do it yet (recall that this is due to the fact that SQL is **declarative**).

**(10)  Physical operator mapping**

Example algorithm/physical operator: **hashjoin**. More on [hashing](#hashing-continued) later.

This is now a **physical** plan (in contrast to a **logical** plan). We *could* now give this exact tree to an execution engine, which would traverse the tree bottom-up and dutifully apply all the operations specified by the tree nodes.

**(11) Cardinality estimator**

Estimate more numbers for the edges.

**ASIDE:** how are the estimates done? A little bit of math, mostly guessing. This is one of the biggest unsolved problems in databases. Don't worry too much about it for now.

**(12) Analyzed plan**

You can click the button in the bottom left: **Fetch Analyze Plan**. This actually executes the plan and gets the true numbers of rows returns at each step.

Notice that there's some (pretty bad) error between the *estimated* numbers and the true numbers obtained after execution. We see that the (now beautifully colored) tree has a **Est. Q-Error** field in the nodes now.

**Q-Error** is a way to quantify the error between the estimated and actual output, and it's calculated by taking the larger size over the smaller size.

Taking a step back: why do we care about joins so much?

- Joins are *expensive* because are the only operator that can produce a *larger* table than its operands table. What's more, it scales quadratically (as opposed to something like union, which is at most $N + M$). If we don't do it right, the size of the intermediates can quickly blow up.
- That then begs the question: why do we join so much instead of just using unions? We *need* joins because we need to reconcile information from different tables that were decomposed to reduce dependency (see: decomposition lecture).

## Hashing (Continued)

### Review

Recall:

> [!IMPORTANT]
>
> > *Never* store passwords in plain text. NEVER. Store database **hashes** instead, preferably using **salting** as well.

Recall the properties of a good hash function and why each of them are important:

> - **Deterministic**: $x = y \Longrightarrow h(x) = h(y)$
>   - Given the same plaintext password, it computes the same hash value. This lets us compare login attempts with the stored source of truth.
> - **Low collision**: $x \ne y \Longrightarrow h(x) \ne h(y)$ (that is, $\Pr(h(x) = h(y)) \approx 0$)
>   - Different passwords should not hash to the same value. Otherwise, you could log in to someone else's account with your password!
> - **Easy to compute**: $f(x) \in O(1)$
>   - Important for performance. You don't want to keep your users waiting at the login page!
> - But **hard to invert**: $f^{-1}(x) \in O(2^N)$
>   - Even if an attacker obtains the hashed value, they shouldn't be able to easily compute what original plaintext password hashes to that value. If hashes weren't hard to invert, it kind of defeats the purpose of using them in the first place!
>

### Motivation: Hashing to Optimize Joins

Suppose we have a query on two tables (not sure about this syntax):

$$Q(x, y, z) := T_1(x, y) \land T_2(y, z)$$
$$t_1.y = t_2.y$$

Recall the nested loop semantics of such a join:

```python
for t1 in T1:
    for t2 in T2:
        if t1.y == t2.y:
            print(t1, t2)
```

What if I tell you there's a **functional dependency** (bet you didn't think you'd see those again 😀): $Y \to Z$?

<!-- Review: What is a **key** for $T_2$? $Y$. -->

**CLAIM:** we can somehow leverage the FD to *remove* the inner loop. If you're familiar with Python and/or did HW2, you might recall that you can use a **hashmap** data structure (Python `dict`) to *associate* some key type to some value type&mdash;doesn't that sound great for representing a functional dependency $K \to V$?

```python
# Assume d2 is an initialized mapping of Y -> Z.
d2: dict[int, int]

for t1 in T1:
    # Since y functionally determines z, we can just perform a lookup
    # on what z is associated with the y we have.
    z = d2[t1.y]
    # Only one loop needed to get everything we need!
    print(t1.x, t1.y, z)
```

An important detail: notice that we've eliminated a loop, but this is only more efficient if the `d1[...]` lookup itself is better than just looping through `t2`. How fast is the `d1[...]` lookup? CS 32 is a prerequisite for this class, so you'll know the answer is $O(1)$, but bear with us for a bit as we use this to motivate **hashing** in general.

### Hashmaps Explained

What if the keys of our mapping happened to be the numbers in the range $[1, 100]$? You wouldn't need to store the key information at all since it can just be encoded *positionally* within some sequential container (e.g. an array):

```python
# (N.B.: it doesn't matter that it doesn't start at 0 since we can
#  just offset by the starting value when indexing)
#         1     2     3
table = ["a", "db", "cs", ...]
```

But what if we generalize it to the case where we have arbitrary keys which aren't necessarily contiguous? For example, $1, 3, 5, 42, ...$.

What if we want to find some key like `5`? If the keys are sorted, you can just use **binary search**. But if you recall, that has a time complexity of $O(\log N)$ where $N$ is the number of keys. This also assumes the domain is sorted, which itself incurs more overhead to maintain.

A solution that does better than $O(\log N)$ is to use **hashing**.

Consider a trivial (not very effective) hash function for a range of size $N=10$:

$$h(x) = x \mod 10$$

Suppose we have these keys: $(11, 32, 45, 63, 94)$.

We'll allocate an array of size 5 (using one-indexing for now, just to be consistent with lecture):

```
  1  2  3  4  5
[  |  |  |  |  ]
```

We can take the hash $h(\cdot)$ of each key to *map* each key value to a *position* within this array:

- $11 \mod 10 = 1$
- $32 \mod 10 = 2$
- $45 \mod 10 = 5$
- $63 \mod 10 = 3$
- $94 \mod 10 = 4$

```
  1  2  3  4  5
[11|32|63|94|45]
```

Now if you want to **look up** a key, you just apply the same hash function to find the position where the key *must* have been stored. That is, insertion and lookup are symmetric.

### Resolving Hash Collisions

Now suppose we have these values instead: $(11, 32, 42, 63, 94)$.

- $11 \mod 10 = 1$
- $32 \mod 10 = 2$
- $42 \mod 10 = 2$

Since two different keys $32 \ne 42$ mapped to the same hash $2$, we have a **collision**. To resolve this, we have a couple of options. The simplest is to generalize the elements within the array into buckets of **linked lists** that can store arbitrarily many keys.

```
 1 2 3 4 5
[ | | | | ]
 | |
 | +---> (32) -> (42) -> /
 +---> (11) -> /
```

Note that now insertion has to be modified too since each bucket can now correspond to *multiple* keys. But that's just a matter of:

1. Computing the hash like before to locate the *bucket*.
2. Then *iterating* through that bucket's linked list and *compare by equality* each element in the linked list until we find the key.

This method of conflict resolution is called [**closed-addressing** (or **open-hashing**)](https://programming.guide/hash-tables-open-vs-closed-addressing.html).

Another approach is to use [**open-addressing** (or **closed-hashing**)](https://programming.guide/hash-tables-open-vs-closed-addressing.html). The prototypical example of that is **linear probing.**

Suppose we want to insert the keys $11, 32, 63, 94, 42$.

We hash and store the first 4 like normal:

```
[11, 32, 63, 94, /]
```

42 has a collision with 32. We'll linearly scan the array until we find a vacancy:

```
[11, 32, 63, 94, /]
 ^
     ^
         ^
             ^
                 ^ insert!
[11, 32, 63, 94, 42]
```

What about lookup? It's similarly symmetric to insertion.

1. You would first hash and look at the bucket like before.
2. If the entry is not equal to the key, we just linearly scan until we find the key.

**ASIDE:** In practice, linear probing is used more often than linked lists. Why? Linked list entries are scattered across memory. If you've taken CS 33/CS 111, you learned that this does not exhibit good [**spatial locality**](https://en.wikipedia.org/wiki/Locality_of_reference), so the cache performance is poorer. Contrast this with linear probing, where we only have elements stored contiguously in an array data structure. Great locality!

However, you might notice a flaw in linear probing. Suppose we have this stream of keys: $11, 32, 63, 94, 42, 52$.

Insert 11, 32, 63, 94:

```
[11, 32, 63, 94,  /,  /, ...]
```

Insert 42:

```
 scan for vacancy
---------------->|
[11, 32, 63, 94,  /,  /, ...]
[11, 32, 63, 94, 42,  /, ...]
```

Insert 52:

```
   scan for vacancy
--------------------->|
[11, 32, 63, 94, 42,  /, ...]
[11, 32, 63, 94, 42, 52, ...]
```

This phenomenon is called **clustering**, when a collision increases the chance of *further* collisions.

How do we solve this?

- Firstly, a good hash function helps a lot (values in the domain should evenly distribute among the range).
- Secondly and more simply, just make the array larger (increase the number of buckets). This makes the final modulo mapping hash values to array indices less likely to cause collisions.

**ASIDE:** Another (less polite) open-addressing approach is [**cuckoo hashing**](https://en.wikipedia.org/wiki/Cuckoo_hashing): upon collision, kick out the current resident and use a different hash function to reallocate the evicted resident!
- A property of this is that lookup is $O(1)$ because you need to look at at most 2 places, compared to the $O(N)$ of linear probing.
- Insertion however is a bit more expensive since the eviction process might not terminate efficiently.

<!-- TODO: Explain this activity. -->
<!-- ### Activity: Throwing Bullets at a Target

A volunteer repeatedly throw NERF bullets to a target drawn on the projector as Professor records the closest distance to the bullseye so far. The main takeaway is that the closest distance is monotonically non-increasing (that is, more tries $\Longrightarrow$ closer to the target).

This is a similar idea to how we get an estimate of the number of distinct values (estimating `COUNT(DISTINCT ...)` on a stream of items).

Suppose we have a bunch of strings, and we want to get an estimate of the number of unique values:

```
"CS", "143", "DB", "UCLA", "Casa", "Kira", "CS", "CS", "142", "DB", "DB", "DB", "DB"
```

Suppose we hash the entries one by one:

- `h("CS") = n1`
- `h("143") = n2`
- ...

Keep track of the smallest number seen so far?

TODO: If we have n bits:

$$\frac{2^n}{min hash}$$ -->

## Bloom Filters

### Motivation

Hash tables are great because they provide a 100% correct answer in $O(1)$ time whether an element exists in it. However, the drawback is its $O(N)$ space requirement. *Every* element needs to be stored somewhere in the data structure.

Instead of storing *every single* value, we can design a data structure that *approximates* the behavior of a hash function:

- If the lookup returns NO, the element is *for sure* not in the collection.
- If the lookup returns YES, the element *might* be in the collection but it also might not be.

This is suitable for scenarios where it's acceptable to sacrifice a bit of accuracy in exchange for a much smaller space requirement. An example off the top of my head is maybe checking whether a user exists. We can first query the bloom filter, and if it returns NO, we know the user doesn't exist. If it returns yes, we then fall back to a traditional service that queries the full database to confirm existence. Notice that this can be easily generalized to any situation of "check something likely to be NO first, double check if it happens to be YES".

### How It Works

Neuron analogy: a naive theory is that we have one neuron for every single thing in life, like a neuron for every person you know, a neuron for every concept, etc. However, that would take too many neurons. Instead, we have a bunch of cells (**ensemble**) that light up that respond to different stimuli, and each cell might participate in multiple ensembles at once (probably? not a neuroscientist).

This is similar in principle to how a **bloom filter** data structure works. Suppose we want to map a bunch of database items into some vector using [**one-hot encoding**](https://en.wikipedia.org/wiki/One-hot):

```
0 0 0 0 1 0 0 0 0 0
        ^ "DB neuron"
```

```
0 0 0 0 0 0 0 1 0 0
              ^ "grandma neuron"
```

If we have 10 bits, we can encode $2^{10}$ distinct values.

A bloom filter is essentially a mapping from input values to a **bit vector**. We have a collection of hash functions $f_1, f_2, ..., f_k$ that all compute a hash on some database item and use those hash values to set specific bits within that bit vector e.g.:

```
0 1 1 0 0 0 0 1 0 1
  ^ ^         ^   ^
 f3 f2       f1   f4
```

This hash result is like the item's "signature". Intuitively, it's kind of like casting that item's "shadow" on the bit vector. Hopefully that helps explain the behavior:

- If the element we're looking up is missing a 1 where there's supposed to be one, there's no way that element could've been stored, so we are 100% sure the element does not exist in the data structure.
- If the element we're looking up has all of its 1s where it should be, it *could* still be a **false positive** because the combination of other inputs could happen to result in all of our lookup element's positions being set. It follows that this probability increases the more elements we store in our bloom filter.

Thus, we have a data structure that *approximates* the correctness of a traditional hash table but has a much smaller space requirement (tuned depending on the error probability you can tolerate).
