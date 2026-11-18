# 5. Coding / Whiteboard

Algorithm rounds test: do you **clarify the problem**, **think out loud**, **write working code**, **discuss complexity**. For freshers/juniors it's mostly Easy–Medium.

## The UMPIRE process (narrate each step)
1. **U**nderstand: clarify (input/output, constraints, edge cases: empty, negative, duplicates, very large).
2. **M**atch: which pattern is this (array/string/hash/two-pointer/recursion...).
3. **P**lan: sketch the approach in words/pseudocode before typing.
4. **I**mplement: write it, explaining as you go.
5. **R**eview: run through 1–2 examples (including edge cases).
6. **E**valuate: time/space complexity, can it be optimized.

## Big-O to remember
- O(1) constant · O(log n) binary · O(n) scan · O(n log n) sort · O(n²) nested loops.
- Use a **hash map/set** to turn O(n²) → O(n) (O(1) membership).

## Common patterns (Easy–Medium)
- **Strings**: reverse, palindrome check, char counting, **anagram** (compare Counters). *(Tie-in: word-checker uses regex/frequency counting.)*
- **Arrays**: two-sum (hash map), max/min, merge intervals, move zeros.
- **Two pointers / sliding window**: max subarray sum, longest substring without repeats.
- **Hashing**: frequency counting, most frequent element (e.g. `Counter.most_common`).
- **Recursion / traversal**: factorial, Fibonacci (with memo), tree/graph traversal (DFS/BFS).
- **Sort + search**: binary search on a sorted array.

## Sample solutions (Python — whiteboard-friendly)

**Two-sum (O(n)):**
```python
def two_sum(nums, target):
    seen = {}                 # value -> index
    for i, x in enumerate(nums):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i
    return None
```

**Frequency count / most common:**
```python
from collections import Counter
def most_common_word(words):
    return Counter(words).most_common(1)[0][0]
```
*(Exactly the idea behind word-checker picking the most common font, and quan-quen-map's weighted suggestions.)*

**Palindrome check (two pointers):**
```python
def is_palindrome(s):
    i, j = 0, len(s) - 1
    while i < j:
        if s[i] != s[j]:
            return False
        i += 1; j -= 1
    return True
```

## Whiteboard tips
- **Don't code silently**: narrate your thinking — they grade reasoning.
- **Ask about edge cases first**: empty, single element, duplicates, overflow.
- If stuck, state the brute-force first then optimize ("I'll do O(n²) first, then use a hash to get O(n)").
- Use clear variable names; trace through a small example by hand.

## Sample questions
- "Complexity of your solution?" → state time + space.
- "Can you optimize?" → change the data structure (array→hash), remove a loop.
- "Handling very large input?" → avoid O(n²), consider streaming.

> 💡 Practicing ~20–30 Easy/Medium problems on LeetCode/HackerRank by pattern (strings, arrays, hashing, two pointers) is enough confidence for a fresher/junior coding round.
