---
layout: post
title: "The Only Binary Search You Ever Need - Why the Half-Open Interval Pattern is Mathematically Superior"
date: 2026-08-10 08:00:00 +0300
categories: [algorithms, competitive-programming]
tags: [binary-search, cpp, competitive-programming, math, algorithms]
math: true
---

If you have done competitive programming or solved LeetCode problems for more than a week, you have definitely been personally victimized by binary search. 

We have all been there. You write out a quick binary search during a contest, hit submit, and get `Time Limit Exceeded` because your code got stuck in an infinite loop on test case 4. Or worse, you get `Wrong Answer` on an edge case because you returned `low` instead of `high`, or forgot whether your loop condition was supposed to be `low <= high` or `low < high`. 

Then you spend the next 20 minutes randomly throwing `+ 1` and `- 1` at `mid`, `low`, and `high` like you are playing a slot machine, hoping the compiler gods take pity on you.

I used to do this all the time until I stopped using standard textbook binary search and switched exclusively to the half-open interval model: $[st, dr)$ with a `while (dr - st > 1)` termination condition. 

Once you understand the math behind this invariant-driven pattern, binary search stops being a guessing game. You will never write an infinite loop again, you will never wonder whether to add or subtract 1, and your code will work on the first submit every single time. 

Here is why this pattern is mathematically superior to every other binary search template.

---

## The Chaos of Standard Binary Search Templates

Before diving into the fix, let's break down why traditional binary search implementations are a total nightmare. 

Most textbooks teach binary search like this:

```cpp
// The textbook template full of edge-case landmines
int low = 0;
int high = n - 1;
while (low <= high) {
  int mid = low + (high - low) / 2;
  if (arr[mid] == target) {
    return mid;
  }
  if (arr[mid] < target) {
    low = mid + 1;
  } else {
    high = mid - 1;
  }
}
return -1;
```

This looks innocent, but it breaks down the moment you need to search for predicate boundaries (e.g., finding the lower bound, upper bound, or binary searching on answer spaces). 

Notice the mental load required here:
1. **Loop Condition:** Is it `low <= high` or `low < high`?
2. **Boundary Updates:** Do I do `low = mid + 1` or `low = mid`? `high = mid - 1` or `high = mid`?
3. **Mid Calculation:** Should `mid` round down `(low + high) / 2` or round up `(low + high + 1) / 2`? (If you pick the wrong combination with `low = mid`, your code loops infinitely on size 2 ranges!).
4. **Return Value:** When the loop finishes, where is your answer? Is it `low`? Is it `high`? What if the answer doesn't exist?

Every decision depends on micro-details that are trivial to mess up under contest time pressure.

---

## The $[st, dr)$ Model: Invariants over Guesswork

Instead of thinking about searching for a specific index, let's reframe binary search mathematically. 

Binary search is just finding the transition boundary in a monotonically partitioned boolean space. You have a search space where a predicate $P(x)$ evaluates to `false` for all values up to a point, and `true` for all values after (or vice versa).

$$\underbrace{\text{false, false, false, \dots, false}}_{st \text{ region}} \quad \vert \quad \underbrace{\text{true, true, \dots, true, true}}_{dr \text{ region}}$$

We maintain a half-open interval $[st, dr)$ with two fundamental **invariants**:

1. **$st$ is ALWAYS a known state where $P(st) = \text{false}$** (or the lower invalid region).
2. **$dr$ is ALWAYS a known state where $P(dr) = \text{true}$** (or the upper valid region).

Here, $st$ stands for *start* (inclusive bound) and $dr$ stands for *dreapta* / direction (exclusive upper bound).

### The Algorithm Template

```cpp
int st = MIN_BOUND - 1; // Guaranteed to be false (or invalid)
int dr = MAX_BOUND + 1; // Guaranteed to be true (or valid)

while (dr - st > 1) {
  int mid = st + (dr - st) / 2;
  if (check(mid)) {
    dr = mid;
  } else {
    st = mid;
  }
}

// Result: st is the LAST false, dr is the FIRST true!
```

Look at that code. 
- **NO `mid + 1`**
- **NO `mid - 1`**
- **NO checking if `low <= high`**
- **NO ambiguity about return values**

`st` is assigned `mid` directly. `dr` is assigned `mid` directly. 

Why can we do this without getting stuck in infinite loops? Because the math guarantees it.

---

## The Mathematical Proof of Termination and Progress

Let's prove why `dr - st > 1` strictly guarantees progress and prevents infinite loops.

### Lemma 1: $mid$ is strictly contained within $(st, dr)$

Assume $st, dr \in \mathbb{Z}$ and the loop condition holds: 

$$dr - st > 1 \implies dr - st \ge 2$$

We compute the midpoint using standard integer division (truncating towards zero):

$$mid = st + \lfloor \frac{dr - st}{2} \rfloor$$

Since $dr - st \ge 2$, we know that $\lfloor \frac{dr - st}{2} \rfloor \ge 1$. Therefore:

$$mid \ge st + 1 > st \implies mid > st$$

Now let's bound $mid$ from above. Since $dr - st \ge 2$, integer division satisfies $\lfloor \frac{k}{2} \rfloor < k$ for any integer $k \ge 2$. Thus:

$$\lfloor \frac{dr - st}{2} \rfloor < dr - st$$

Adding $st$ to both sides gives:

$$st + \lfloor \frac{dr - st}{2} \rfloor < dr \implies mid < dr$$

Combining both inequalities yields:

$$st < mid < dr$$

### Proof of Strict Shrinking

Because $st < mid < dr$:
* If $P(mid)$ is `true`, we update $dr' = mid$. Since $mid < dr$, the new upper bound is strictly smaller ($dr' < dr$).
* If $P(mid)$ is `false`, we update $st' = mid$. Since $st < mid$, the new lower bound is strictly larger ($st' > st$).

In both execution branches, the length of the interval $dr - st$ strictly decreases by at least 1 at every iteration. Because the range is finite, the algorithm is **guaranteed to terminate in $O(\log(dr - st))$ steps**. 

No edge case can cause $mid == st$ or $mid == dr$ while $dr - st > 1$, which makes infinite loops mathematically impossible.

---

## Exact Boundary Termination: Zero Guesswork

What happens when the loop finishes?

The loop condition is `while (dr - st > 1)`. The loop terminates when:

$$dr - st \le 1$$

Since $dr$ starts strictly greater than $st$ and decreases by integer steps, the loop stops at the **exact moment**:

$$dr - st = 1 \iff dr = st + 1$$

At termination, $st$ and $dr$ are two adjacent integers. 

Because our invariants were maintained at every single step:
- $P(st)$ is guaranteed to be `false` (the last `false` in the search space).
- $P(dr)$ is guaranteed to be `true` (the first `true` in the search space).

There is zero guesswork after the loop ends:
- Want the first index where condition is met (e.g., `std::lower_bound`)? Return `dr`.
- Want the last index where condition is NOT met? Return `st`.

---

## Real-World C++ Examples

Let's look at how clean this looks in actual code. (Note: using raw arrays for zero memory allocation overhead in CP).

### Example 1: Custom Lower Bound Implementation

Find the first index in a sorted array where `arr[i] >= target`.

```cpp
#include <iostream>

const int MAXN = 100005;
int arr[MAXN];

int custom_lower_bound(int n, int target) {
  // st = -1 (outside array bounds, check(-1) is conceptually false)
  // dr = n  (outside array bounds, check(n) is conceptually true)
  int st = -1;
  int dr = n;

  while (dr - st > 1) {
    int mid = st + (dr - st) / 2;
    if (arr[mid] >= target) {
      dr = mid;
    } else {
      st = mid;
    }
  }

  // dr is the first index where arr[dr] >= target
  return dr;
}
```

If all elements are smaller than `target`, `dr` remains `n`, naturally signaling that no element satisfied the condition. If all elements are $\ge$ `target`, `dr` becomes `0`. Edge cases disappear.

### Example 2: Binary Search on the Answer (Optimization Problems)

Consider a problem where you need to find the minimum allocation capacity $W$ such that items can be shipped within $K$ days.

```cpp
#include <iostream>

const int MAXN = 100005;
int weights[MAXN];

bool check(int capacity, int n, int k) {
  int days = 1;
  int current_sum = 0;

  for (int i = 0; i < n; i++) {
    if (weights[i] > capacity) {
      return false;
    }
    if (current_sum + weights[i] > capacity) {
      days++;
      current_sum = weights[i];
    } else {
      current_sum += weights[i];
    }
  }

  return days <= k;
}

int solve_min_capacity(int n, int k, int max_possible_weight) {
  // st = 0 is impossible/false (assuming weights > 0)
  // dr = max_possible_weight + 1 is guaranteed to be true
  int st = 0;
  int dr = max_possible_weight + 1;

  while (dr - st > 1) {
    int mid = st + (dr - st) / 2;
    if (check(mid, n, k)) {
      dr = mid;
    } else {
      st = mid;
    }
  }

  // dr holds the minimum valid capacity
  return dr;
}
```

Notice how intuitive the initialization is:
- Pick `st` as a value you know for a fact **fails** the test.
- Pick `dr` as a value you know for a fact **passes** the test.
- Run `while (dr - st > 1)` with `st = mid` and `dr = mid`.
- Return `dr`. That's literally it.

---

## Extension: Real / Floating-Point Binary Search

The beauty of this invariant model is that it transfers seamlessly to continuous binary search (searching over `double` or `float`).

When searching over real numbers, you don't even have to worry about step sizes. You just run the loop either for a fixed number of iterations (e.g., 80 iterations gives $\sim 10^{-24}$ precision) or until `dr - st > 1e-9`.

```cpp
double solve_floating_point(double low_bound, double high_bound) {
  double st = low_bound;
  double dr = high_bound;

  for (int iter = 0; iter < 80; iter++) {
    double mid = st + (dr - st) / 2.0;
    if (check_real(mid)) {
      dr = mid;
    } else {
      st = mid;
    }
  }

  return dr;
}
```

Because there is no `+ 1` or `- 1` anywhere in the logic, the code for real numbers and discrete integers is conceptually identical!

---

## Comparison Matrix

| Feature | Closed Interval `[lo, hi]` | Half-Open Interval `[st, dr)` |
| :--- | :--- | :--- |
| **Loop Condition** | `while (lo <= hi)` | `while (dr - st > 1)` |
| **Mid Updates** | `mid + 1` / `mid - 1` | `st = mid` / `dr = mid` |
| **Infinite Loop Risk** | High (if off-by-one on rounding) | Mathematically Impossible |
| **Return State** | Need to remember if `lo` or `hi` | `st` is last false, `dr` is first true |
| **Boundary Bounds** | Must fit inside $[0, N-1]$ | Safe setup with $-1$ and $N$ |
| **Mental Stress** | Extreme | Non-existent |

---

## Summary Cheat Sheet

Next time you need to write binary search, just follow these 4 steps:

1. **Define the predicate $P(x)$** such that the search space is partitioned into `false` then `true`.
2. **Initialize bounds:**
   - Set `st` to a safe out-of-bounds value where $P(st)$ is `false` (e.g., `-1`).
   - Set `dr` to a safe out-of-bounds value where $P(dr)$ is `true` (e.g., $N$).
3. **Loop while `dr - st > 1`:**
   - `mid = st + (dr - st) / 2`
   - If $P(mid)$ is `true`, `dr = mid`. Else `st = mid`.
4. **Extract answer:**
   - First `true` element is `dr`.
   - Last `false` element is `st`.

Stop memorizing special cases and stop guessing $+1$ / $-1$ offsets. Use $[st, dr)$ and let the invariants do the work for you!
