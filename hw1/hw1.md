# Homework 1: SQL Exploration

## Part 1: Filtering with `WHERE`

### What does `SELECT *` mean?
`SELECT *` means select all columns from table. The `*` is a wildcard that saves you from typing everything out

### 1. The query outputs fewer rows than R
**Status:** Possible

**SQL Commands:**
```sql
CREATE TABLE R (x INTEGER);
INSERT INTO R (x) VALUES (1), (5);
SELECT * FROM R WHERE x < 3;
```

### 2. The query outputs more rows than R
**Status:** Impossible

**Reasoning:** You cannot output more rows than what you began with

### 3. The query outputs the same number of rows as R
**Status:** Possible

**SQL Code:**
```sql
-- Starting with a fresh table
DROP TABLE IF EXISTS R;
CREATE TABLE R (x INTEGER);
INSERT INTO R (x) VALUES (0), (1), (2);
SELECT * FROM R WHERE x < 3;
```


## Part 2: Row Transformations with `x + x`

Consider the query: `SELECT x + x FROM R`

### 1. The query outputs fewer rows than R
**Status:** Impossible

**Reasoning:** If you don't have a `WHERE` or `LIMIT` clause you are kept at exactly same number of rows as R as the output

### 2. The query outputs more rows than R
**Status:** Impossible

**Reasoning:** Same reason as 1

### 3. The query outputs the same number of rows as R
**Status:** Possible

**SQL:**
```sql
DROP TABLE IF EXISTS R;
CREATE TABLE R (x INTEGER);
INSERT INTO R (x) VALUES (10), (20), (30);
SELECT x + x FROM R;
```


## Part 3: `SELECT DISTINCT`

`SELECT DISTINCT x` identifies all unique values in column `x` and removes any duplicates from the final output

#### 1. The query outputs fewer rows than the GROUP BY query
**Status:** Impossible

**Reasoning:** Both `DISTINCT x` and `GROUP BY x` are operations that reduce a dataset based on the unique values of column `x`. If there are $k$ unique values for $x$, both queries will return exactly $k$ rows. There is no scenario where `DISTINCT` would find fewer unique values than `GROUP BY` on the same column

#### 2. The query outputs more rows than the GROUP BY query
**Status:** Impossible

**Why:** For the same reason as above, the "cardinality" (number of rows) of a set of unique values is fixed. `DISTINCT` cannot find "extra" unique values that `GROUP BY` would somehow miss

#### 3. The query outputs the same number of rows as the GROUP BY query
**Status:** Possible (Always True)

**SQL Code:**
```sql
CREATE TABLE R (x INTEGER, y INTEGER);
INSERT INTO R (x, y) VALUES (1, 10), (1, 20), (2, 50);

-- Query A:
SELECT DISTINCT x FROM R; -- Returns 2 rows (1 and 2)

-- Query B:
SELECT x, avg(y) FROM R GROUP BY x; -- Returns 2 rows (1 and 2)
```


## Part 4: Join Cardinality
**Query:** `SELECT * FROM R, S WHERE R.x = S.y`
*Let $r$ = size of $R$, $s$ = size of $S$, and $j$ = output size.*

### 1. Condition: $j \le r + s$
* **To Satisfy:** * `R = {1, 2}`, `S = {1, 3}`. Result: `(1,1)`.
    * $j=1, r=2, s=2$. $1 \le 4$ (True).
* **To Violate:** * `R = {1, 1, 1}`, `S = {1, 1, 1}`. Result: 9 rows of `(1,1)`.
    * $j=9, r=3, s=3$. $9 \le 6$ (False).

### 2. Condition: $j \le r \times s$
* **To Satisfy:** Any two tables will satisfy this.
* **To Violate:** IMPOSSIBLE
    * **Reason:** The Cartesian product ($r \times s$) represents every possible combination of rows between the two tables. Because the `WHERE` clause can only filter these existing combinations, the output size $j$ can never exceed the total number of possible pairings.

### 3. Condition: $j \ge \min(r, s)$
* **To Satisfy:** * `R = {1}`, `S = {1, 2}`. Result: `(1,1)`.
    * $j=1, r=1, s=2$. $1 \ge 1$ (True).
* **To Violate:** * `R = {1}`, `S = {2}`. Result: 0 rows.
    * $j=0, r=1, s=1$. $0 \ge 1$ (False).

### 4. Condition: $j \le \max(r, s)$
* **To Satisfy:** * `R = {1, 2}`, `S = {1, 2, 3}`. Result: `(1,1), (2,2)`.
    * $j=2, r=2, s=3$. $2 \le 3$ (True).
* **To Violate:** * `R = {1, 1}`, `S = {1, 1}`. Result: 4 rows of `(1,1)`.
    * $j=4, r=2, s=2$. $4 \le 2$ (False).


## Part 5: Linear Algebra in SQL

We can represent a vector $\mathbf{v} = [v_1, v_2, \dots, v_k]$ with a 2-column table:

| i | v |
| :--- | :--- |
| 1 | $v_1$ |
| 2 | $v_2$ |
| 3 | $v_3$ |
| ... | ... |
| k | $v_k$ |

Where the first column stores the index $i$, and the second column stores the value $\mathbf{v}[i]$. Similarly, we can represent a matrix $A$ with a 3-column table:

| i | j | A |
| :--- | :--- | :--- |
| 1 | 1 | $A_{11}$ |
| 1 | 2 | $A_{12}$ |
| 1 | 3 | $A_{13}$ |
| ... | ... | ... |
| 2 | 1 | $A_{21}$ |
| 2 | 2 | $A_{22}$ |
| 2 | 3 | $A_{23}$ |
| ... | ... | ... |
| k | k | $A_{kk}$ |

Where the $i$ column stores the row index, $j$ stores the column index, and $A_{ij}$ stores the matrix entry at row $i$, column $j$.

---

### 1. Point-wise Product
```sql
SELECT A.i, A.v * B.v 
FROM A, B 
WHERE A.i = B.i;
```

### 2. Dot Product
```sql
SELECT SUM(A.v * B.v) 
FROM A, B 
WHERE A.i = B.i;
```

### 3. Outer Product
```sql
SELECT A.i, B.i AS j, A.v * B.v 
FROM A, B;
```

### 4. Matrix Product
```sql
SELECT A.i, B.j AS k, SUM(A.val * B.val)
FROM A, B
WHERE A.j = B.i
GROUP BY A.i, B.j;
```

### 5. Sparse Tables

Queries do **NOT** need to change for sparse tables

**Reasoning:**  SQL joins naturally handle sparse data. Because we are using inner joins based on indices, any "missing" (zero) entry in a sparse table will simply result in no match during the join. Since $0 \times x = 0$, and the sum of a missing value is effectively zero, the results remain mathematically correct without any modification to the SQL logic.