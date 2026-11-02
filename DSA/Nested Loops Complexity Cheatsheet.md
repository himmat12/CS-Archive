# Nested Loops Complexity Cheat Sheet  
## Disjoint → Additive vs Overlapping → Multiplicative

---

## Core Principle

> **Don't look at loop nesting; look at total work.**  
> Ask: *How many times is each element touched in total across the whole run?*

---

## 1. Disjoint / Amortized Scans → Additive Complexity (\(O(n)\))

### Pattern

- Outer loop: \(O(n)\)
- Inner loop(s): total steps across **all** outer iterations = \(O(n)\)
- Each element is touched a constant number of times overall.

### Result

\[
O(n) + O(n) = O(n)
\]

### Typical Signs

- Inner loop runs only from "starts" of disjoint structures (sequences, segments, groups).
- Each element belongs to exactly one such structure.
- You can argue: "Sum of all inner-loop lengths ≤ \(n\)."

### Example: Longest Consecutive Sequence

```python
nums_set = set(nums)
longest = 0
for n in nums_set:                  # O(n)
    if n - 1 not in nums_set:       # start of a sequence
        length = 1
        while n + length in nums_set:  # walks this sequence
            length += 1
        longest = max(longest, length)
# Total while-steps over all starts = O(n)
# Overall: O(n)
```

### Why It's Additive

- Outer `for`: one check per element → \(O(n)\).
- Inner `while`: each element walked at most once in total → \(O(n)\).
- Total = \(O(n) + O(n) = O(n)\).

---

## 2. Overlapping / Unbounded Scans → Multiplicative Complexity (\(O(n \cdot t)\) or \(O(n^2)\))

### Pattern

- Outer loop: \(n\) iterations.
- For many outer iterations, inner loop can do up to \(t\) steps.
- Inner scans can heavily overlap; same elements re-scanned many times.
- No global bound like "total inner steps ≤ \(O(n)\)."

### Result

\[
\text{Total work} = O(n \cdot t)
\]

- If \(t = n\) → \(O(n^2)\).

### Typical Signs

- For each element (or query), you scan a window/segment that can be \(O(n)\).
- No disjointness: sequences/segments overlap arbitrarily.
- Worst case: inner loop does \(O(n)\) work for each of \(n\) outer iterations.

### Example 1: Naive Double Loop

```python
for i in range(n):          # n iterations
    for j in range(n):      # up to n iterations each time
        do_O1_work()
# Total: n * n = O(n^2)
```

### Example 2: Per-Start Scan Without Disjointness

```python
# t starting points, each can scan up to n elements
for s in starts:            # t iterations
    x = s
    while x in nums_set:    # up to O(n) per start, overlapping
        x += 1
# Worst case: O(t * n); if t ≈ n → O(n^2)
```

---

## 3. Amortized Inner Work → Still Additive

### Pattern

- Some outer iterations trigger expensive inner work.
- But expensive cases are rare / limited.
- Total inner work over all outer iterations = \(O(n)\).

### Result

\[
O(n) + O(n) = O(n) \quad \text{(amortized)}
\]

### Typical Signs

- Occasional big re-scans or re-builds, but they "pay for" many cheap operations.
- Classic in dynamic arrays, union-find, some two-pointer/sliding-window proofs.

### Mental Model

> "Yes, this `while` can be long sometimes, but over the whole algorithm the sum of all those lengths is only \(O(n)\)."

---

## 4. Quick Decision Checklist

Ask these in order:

1. **How many outer iterations?**  
   - Usually \(n\), sometimes \(t\).

2. **For each outer iteration, how many inner steps in the worst case?**  
   - Constant? → likely \(O(n)\).  
   - Up to \(t\)? → candidate for \(O(n \cdot t)\).

3. **Do inner scans overlap or are they disjoint?**  
   - **Disjoint** (each element in one segment/sequence):  
     - Sum of inner lengths ≤ \(n\) → additive → \(O(n)\).
   - **Overlapping** (same elements re-scanned many times):  
     - No global bound → multiplicative → \(O(n \cdot t)\) or \(O(n^2)\).

4. **Can you bound the total number of inner steps across all outer iterations?**  
   - Yes, by \(O(n)\) → additive/amortized → \(O(n)\).  
   - No, worst case is "\(n\) outer × up to \(t\) inner" → multiplicative.

---

## 5. Rule-of-Thumb Formulas

### Disjoint + Global Bound on Inner Steps

\[
\text{Time} = O(\text{outer}) + O(\text{total inner}) = O(n) + O(n) = O(n)
\]

### Overlapping, No Global Bound

\[
\text{Time} = O(\text{#outer} \times \text{max inner per outer}) = O(n \cdot t)
\]

- If \(t = n\) → \(O(n^2)\).

---

## One-Liner to Remember

> **Disjoint / amortized → add.**  
> **Overlapping / unbounded → multiply.**