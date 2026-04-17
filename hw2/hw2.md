# Homework: Relational Design, Closures, and Decomposition

## Part 1: Computing Closures

### 1. Implement `closure(fds, xs)`
Here is the implementation of the closure algorithm in Python.

```python
def closure(fds, xs):
    """Compute the closure of xs over fds"""
    closure_set = set(xs)
    changed = True
    
    while changed:
        changed = False
        for lhs, rhs in fds:
            # If the left-hand side is fully in our closure, 
            # and there are new attributes on the right-hand side
            if lhs.issubset(closure_set) and not rhs.issubset(closure_set):
                closure_set.update(rhs)
                changed = True
                
    return closure_set
```

### 2. The Optimization (Discarding successfully applied FDs)
**Why is this correct?** The set of attributes in `closure_set` grows monotonically (it never shrinks). Once an FD $X \rightarrow Y$ successfully applies, all attributes in $Y$ are added to the closure. If we encounter $X \rightarrow Y$ again in a future loop, $Y$ is already guaranteed to be in the closure. Since sets do not keep duplicate elements, applying it a second time does absolutely nothing. 

### 3. Challenge: Asymptotic Complexity
* Let $n$ be the number of attributes and $m$ be the number of FDs.
* **Worst Case:** $O(m \cdot n^2)$ for this naive loop. 
    * *Input that causes this:* A chain of dependencies like $A_1 \rightarrow A_2$, $A_2 \rightarrow A_3, \dots, A_{n-1} \rightarrow A_n$. In the worst case, each full scan over the $m$ FDs only adds exactly 1 new attribute to the closure, forcing the `while` loop to run $O(n)$ times.
* **Best Case:** $O(m \cdot n)$ or simply $O(m)$ if all FDs are evaluated and successfully applied on the very first pass, requiring only one additional pass to verify `changed = False`.

---

## Part 2: Checking Dependencies and Armstrong Axioms

### 1. Implement `check(fds, fd)`
To check if an FD $X \rightarrow Y$ follows from $\Phi$, we simply compute the closure of $X$ and see if it contains $Y$.

```python
def check(fds, fd):
    """Check if fd follows from fds"""
    lhs, rhs = fd
    return rhs.issubset(closure(fds, lhs))
```

### 2. Proving Armstrong Axioms by hand using `check`
* **Reflexivity ($Y \subseteq X \implies X \rightarrow Y$):** The algorithm initializes `closure_set` as $X$. Since $Y \subseteq X$, $Y$ is immediately a subset of `closure_set`. `check` returns `True`.
* **Augmentation ($\{X \rightarrow Y\} \vdash X \cup Z \rightarrow Y \cup Z$):**
  The closure initializes with $X \cup Z$. Because $X \subseteq X \cup Z$, the FD $X \rightarrow Y$ applies, adding $Y$. The closure is now $X \cup Y \cup Z$. We check if $Y \cup Z$ is a subset of $X \cup Y \cup Z$, which it trivially is.
* **Transitivity ($\{X \rightarrow Y, Y \rightarrow Z\} \vdash X \rightarrow Z$):**
  The closure initializes with $X$. The FD $X \rightarrow Y$ applies, adding $Y$ (closure is $X \cup Y$). In the next iteration, since $Y$ is now in the closure, $Y \rightarrow Z$ applies, adding $Z$ (closure is $X \cup Y \cup Z$). We check if $Z \subseteq X \cup Y \cup Z$, which it is.

---

## Part 3: Superkeys and BCNF Decomposition

### 1. Implement Superkey Check
```python
def isSuperkey(X, fds, sigma):
    """Check if X is a superkey for the table attributes sigma"""
    return sigma.issubset(closure(fds, X))
```

### 2. Prove the Optimized Algorithm Correctness
The original algorithm looks for a non-trivial FD $X \rightarrow Y$ where $X$ is not a superkey. 
The optimized algorithm looks for $X$ such that $X^+ \neq X$ and $X^+ \neq S$.
* **Non-trivial FD:** If $X^+ \neq X$, it means the closure contains attributes not in $X$. Let $Y = X^+ \setminus X$. By definition of closure, $X \rightarrow Y$ holds, and because $X \cap Y = \emptyset$, it is non-trivial.
* **Not a superkey:** If $X^+ \neq S$, it means $X$ does not functionally determine all attributes in the table. Therefore, $X$ is not a superkey.
This perfectly matches the exact violation conditions for Boyce-Codd Normal Form (BCNF).

### 3. Implement `decompose(S, fds)`
```python
import itertools

def powerset(iterable):
    s = list(iterable)
    return itertools.chain.from_iterable(itertools.combinations(s, r) for r in range(len(s)+1))

def decompose(S, fds):
    """Recursively decompose table S based on fds"""
    S_set = set(S)
    
    for xs_tuple in powerset(S_set):
        xs = set(xs_tuple)
        if not xs:
            continue
            
        x_plus = closure(fds, xs)
        
        # Check BCNF violation: X+ != X and X+ != S
        if x_plus != xs and x_plus != S_set:
            # R1 is X+
            R1 = x_plus
            # R2 is S - (X+ - X)
            R2 = S_set.difference(x_plus.difference(xs))
            
            # Recursively decompose
            return decompose(R1, fds) + decompose(R2, fds)
            
    # If no violations found, return the current schema
    return [S_set]
```

### 4. Challenge: Nondeterministic Decomposition
* **Example Schema:** $R(A, B, C, D)$
* **FDs:** $A \rightarrow B$ and $B, C \rightarrow D$.
* **Scenario A (Decompose on $A \rightarrow B$ first):** Results in $R_1(A, B)$ and $R_2(A, C, D)$. The FD $B, C \rightarrow D$ is **destroyed** because $B$ and $C$ are no longer in the same table.
* **Scenario B (Decompose on $B, C \rightarrow D$ first):**
  Results in $R_1(B, C, D)$ and $R_2(A, B, C)$. Both FDs are preserved (though further decomposition may be needed). The final schemas will differ based on which FD you process first.

---

## Part 4: Lossless Joins and Conditional Independence

### 1. Lossless Decomposition Proofs
* **With FDs:** Decomposing $R$ along $X \rightarrow Y$ yields $R_1(X \cup Y)$ and $R_2(R \setminus Y)$. Because $R_1 \cap R_2 = X$, and $X \rightarrow Y$ holds, $X$ is a primary key for $R_1$. When joining on $X$, every row in $R_2$ matches exactly one unique row in $R_1$, preventing spurious tuples and perfectly recovering $R$.
* **With Independence:** If $Y \perp Z \mid X$, decomposing into $(X, Y)$ and $(X, Z)$ is lossless by the definition of Multi-Valued Dependency ($X \twoheadrightarrow Y$). Because $Y$ and $Z$ vary completely independently for a given $X$, the Cartesian product of $Y$ and $Z$ for a specific $X$ already exists in the original table.

### 2. Converse of the previous problem
If $t1 \bowtie t2 = t$, it **does not** imply $x \rightarrow y$ or $x \rightarrow z$. 
It **does** imply $y \perp z \mid x$ (Multi-Valued Dependency). There can be multiple $y$'s and multiple $z$'s for a single $x$, but the lossless join means every combination of those $y$'s and $z$'s for that $x$ exists in the original table.

### 3. Decomposing $x \rightarrow a_i$
* **How many tables?** $k$ tables: $(x, a_1), (x, a_2), \dots, (x, a_k)$.
* **How many rows?** If each decomposed table has $n$ rows, then $x$ takes $n$ distinct values (since $x$ is a key). The original table therefore has exactly **$n$ rows**.

### 4. Decomposing with Conditional Independence ($a_i \perp a_j \mid x$)
* **How many tables?** $k$ tables: $(x, a_1), (x, a_2), \dots, (x, a_k)$.
* **How many rows?** Because $x$ is not a primary key (there can be multiple $a_i$ per $