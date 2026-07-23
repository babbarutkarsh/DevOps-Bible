---
title: DSA for DevOps & SRE
nav_order: 72
description: "Data structures & algorithms for DevOps/SRE coding rounds — the patterns and problems top companies actually ask, with worked solutions."
---

# DSA for DevOps & SRE — Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [Why Coding Rounds for SRE/DevOps?](#why-coding-rounds-for-sredevops)
- [What to Expect](#what-to-expect)
- [Pattern Overview](#pattern-overview)
- [Arrays & Hashing](#arrays--hashing)
- [Two Pointers](#two-pointers)
- [Sliding Window](#sliding-window)
- [Stack](#stack)
- [Intervals](#intervals)
- [Binary Search](#binary-search)
- [Trees & Graphs](#trees--graphs)
- [Heap / Priority Queue](#heap--priority-queue)
- [Ops-Flavored Coding Problems](#ops-flavored-coding-problems)
- [How to Approach the Coding Round](#how-to-approach-the-coding-round)
- [Study Plan](#study-plan)
- [Key Resources](#key-resources)

---

## Why Coding Rounds for SRE/DevOps?

Top product companies (Google SRE, Meta Production Engineering, Amazon, Netflix, Datadog) include a **coding round** in DevOps/SRE interviews to assess:

- **Scripting fluency** — Can you automate complex operational tasks?
- **Problem-solving under constraints** — Real outages require clear thinking and efficient algorithms.
- **Data structure fundamentals** — Logs, metrics, and infrastructure state are data; you need to manipulate them efficiently.

**SRE coding rounds differ from SWE:**
- **Difficulty:** Easy to Medium (occasionally Medium-Hard for senior roles). Rarely hard graph algorithms.
- **Language freedom:** Python and Go are most common (pick what you're fastest in).
- **Focus:** Working code + edge cases + complexity analysis. Brute force → optimize is expected; exotic tricks are not.

---

## What to Expect

| Round Type | Duration | Focus | Example Topics |
|------------|----------|-------|----------------|
| **Phone screen** | 45-60 min | 1-2 Easy problems | Arrays, strings, hashing |
| **Onsite coding** | 45-60 min | 1 Medium or 2 Easy-Medium | Two pointers, sliding window, intervals, stack |
| **Ops-flavored** | 45-60 min | Practical "code this tool" | Parse logs, rate limiter, LRU cache, merge intervals |

**Language choice:**
- **Python** — Fast to write, built-in data structures (dict, set, heapq, collections)
- **Go** — If you're applying for Go-heavy teams (Kubernetes, cloud infra)
- **Bash/awk** — Rarely allowed for algorithmic rounds, but know it for system design

---

## Pattern Overview

| Pattern | Use When | Complexity | Example Problems |
|---------|----------|------------|------------------|
| **Two Pointers** | Sorted array, palindrome, pair sum | O(n) time, O(1) space | Two Sum II, Container With Most Water |
| **Sliding Window** | Substring, subarray, "longest/shortest X" | O(n) time, O(k) space | Longest Substring Without Repeating |
| **Hashing** | Lookup, count, existence check | O(n) time, O(n) space | Two Sum, Anagram Check, Group Anagrams |
| **Stack** | Matching pairs, monotonic sequences | O(n) time, O(n) space | Valid Parentheses, Next Greater Element |
| **Intervals** | Merge, overlap, schedule | O(n log n) time | Merge Intervals, Meeting Rooms |
| **Binary Search** | Sorted data, "find X in O(log n)" | O(log n) time | Classic binary search, search rotated array |
| **BFS/DFS** | Graph traversal, grid search | O(V+E) time | Islands, shortest path, connected components |
| **Heap** | Top-K, running median, priority | O(n log k) time | Top K Frequent, Merge K Sorted |

---

## Arrays & Hashing

### 🔥 Q: Two Sum

**Problem:** Given an array of integers `nums` and an integer `target`, return indices of two numbers that add up to `target`.

**Approach:** Use a hash map to store `{value: index}`. For each element, check if `target - element` exists in the map.

**Complexity:** O(n) time, O(n) space

```python
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        diff = target - num
        if diff in seen:
            return [seen[diff], i]
        seen[num] = i
    return []

# Test
print(two_sum([2, 7, 11, 15], 9))  # [0, 1]
```

**Edge cases:** No solution, duplicate values, negative numbers.

---

### 🔥 Q: Valid Anagram

**Problem:** Check if two strings are anagrams (contain same characters with same frequencies).

**Approach:** Use a hash map (Counter) to count character frequencies, or sort both strings and compare.

**Complexity:** O(n) time, O(1) space (26 letters)

```python
from collections import Counter

def is_anagram(s, t):
    return Counter(s) == Counter(t)

# Test
print(is_anagram("anagram", "nagaram"))  # True
print(is_anagram("rat", "car"))          # False
```

**Go version:**
```go
func isAnagram(s string, t string) bool {
    if len(s) != len(t) {
        return false
    }
    counts := make(map[rune]int)
    for _, c := range s {
        counts[c]++
    }
    for _, c := range t {
        counts[c]--
        if counts[c] < 0 {
            return false
        }
    }
    return true
}
```

---

### ⭐ Q: Group Anagrams

**Problem:** Group strings that are anagrams together.

**Approach:** Use sorted string as key, map to list of anagrams.

**Complexity:** O(n * k log k) time (n strings, k = avg length), O(n*k) space

```python
from collections import defaultdict

def group_anagrams(strs):
    anagrams = defaultdict(list)
    for s in strs:
        key = ''.join(sorted(s))
        anagrams[key].append(s)
    return list(anagrams.values())

# Test
print(group_anagrams(["eat","tea","tan","ate","nat","bat"]))
# [['eat', 'tea', 'ate'], ['tan', 'nat'], ['bat']]
```

---

### ⭐ Q: Product of Array Except Self

**Problem:** Return an array where `output[i]` is the product of all elements except `nums[i]`. No division allowed.

**Approach:** Two passes — left products, then right products.

**Complexity:** O(n) time, O(1) space (output array doesn't count)

```python
def product_except_self(nums):
    n = len(nums)
    output = [1] * n
    
    # Left products
    left = 1
    for i in range(n):
        output[i] = left
        left *= nums[i]
    
    # Right products
    right = 1
    for i in range(n - 1, -1, -1):
        output[i] *= right
        right *= nums[i]
    
    return output

# Test
print(product_except_self([1, 2, 3, 4]))  # [24, 12, 8, 6]
```

---

### 🔥 Q: Top K Frequent Elements

**Problem:** Given an array, return the k most frequent elements.

**Approach:** Count frequencies with hash map, then use heap or bucket sort.

**Complexity:** O(n log k) time with heap, O(n) with bucket sort

```python
from collections import Counter
import heapq

def top_k_frequent(nums, k):
    counts = Counter(nums)
    return [num for num, _ in counts.most_common(k)]

# Alternative: heap-based
def top_k_frequent_heap(nums, k):
    counts = Counter(nums)
    return heapq.nlargest(k, counts.keys(), key=counts.get)

# Test
print(top_k_frequent([1,1,1,2,2,3], 2))  # [1, 2]
```

---

### ⭐ Q: Missing Number

**Problem:** Given array containing n distinct numbers in range [0, n], find the missing number.

**Approach:** Use math (expected sum - actual sum) or XOR.

**Complexity:** O(n) time, O(1) space

```python
def missing_number(nums):
    n = len(nums)
    expected_sum = n * (n + 1) // 2
    actual_sum = sum(nums)
    return expected_sum - actual_sum

# XOR approach (more robust against overflow)
def missing_number_xor(nums):
    xor = len(nums)
    for i, num in enumerate(nums):
        xor ^= i ^ num
    return xor

# Test
print(missing_number([3, 0, 1]))  # 2
```

---

## Two Pointers

### ⭐ Q: Move Zeroes

**Problem:** Move all zeroes to the end while maintaining order of non-zero elements.

**Approach:** Two pointers — one tracks position for next non-zero, one scans array.

**Complexity:** O(n) time, O(1) space

```python
def move_zeroes(nums):
    # Pointer for next non-zero position
    left = 0
    for right in range(len(nums)):
        if nums[right] != 0:
            nums[left], nums[right] = nums[right], nums[left]
            left += 1

# Test
nums = [0, 1, 0, 3, 12]
move_zeroes(nums)
print(nums)  # [1, 3, 12, 0, 0]
```

---

### 🔥 Q: Container With Most Water

**Problem:** Given array of heights, find two lines that form a container with maximum area.

**Approach:** Two pointers from ends, move the shorter pointer inward.

**Complexity:** O(n) time, O(1) space

```python
def max_area(height):
    left, right = 0, len(height) - 1
    max_water = 0
    
    while left < right:
        width = right - left
        h = min(height[left], height[right])
        max_water = max(max_water, width * h)
        
        # Move shorter pointer
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    
    return max_water

# Test
print(max_area([1,8,6,2,5,4,8,3,7]))  # 49
```

**Go version:**
```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    maxWater := 0
    
    for left < right {
        width := right - left
        h := min(height[left], height[right])
        area := width * h
        if area > maxWater {
            maxWater = area
        }
        
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return maxWater
}

func min(a, b int) int {
    if a < b { return a }
    return b
}
```

---

### ⭐ Q: Reverse a String

**Problem:** Reverse a string in-place.

**Approach:** Two pointers from both ends, swap and move inward.

**Complexity:** O(n) time, O(1) space

```python
def reverse_string(s):
    left, right = 0, len(s) - 1
    while left < right:
        s[left], s[right] = s[right], s[left]
        left += 1
        right -= 1

# Test
chars = ['h', 'e', 'l', 'l', 'o']
reverse_string(chars)
print(chars)  # ['o', 'l', 'l', 'e', 'h']
```

---

### 💡 Q: Trapping Rain Water

**Problem:** Given elevation map, compute how much water can be trapped after raining.

**Approach:** Two pointers with left_max and right_max tracking.

**Complexity:** O(n) time, O(1) space

```python
def trap(height):
    if not height:
        return 0
    
    left, right = 0, len(height) - 1
    left_max, right_max = height[left], height[right]
    water = 0
    
    while left < right:
        if left_max < right_max:
            left += 1
            left_max = max(left_max, height[left])
            water += left_max - height[left]
        else:
            right -= 1
            right_max = max(right_max, height[right])
            water += right_max - height[right]
    
    return water

# Test
print(trap([0,1,0,2,1,0,1,3,2,1,2,1]))  # 6
```

---

## Sliding Window

### 🔥 Q: Longest Substring Without Repeating Characters

**Problem:** Find length of longest substring without repeating characters.

**Approach:** Sliding window with hash set to track characters in current window.

**Complexity:** O(n) time, O(min(n, m)) space (m = charset size)

```python
def length_of_longest_substring(s):
    char_set = set()
    left = 0
    max_len = 0
    
    for right in range(len(s)):
        # Shrink window until no duplicate
        while s[right] in char_set:
            char_set.remove(s[left])
            left += 1
        
        char_set.add(s[right])
        max_len = max(max_len, right - left + 1)
    
    return max_len

# Test
print(length_of_longest_substring("abcabcbb"))  # 3 ("abc")
print(length_of_longest_substring("pwwkew"))    # 3 ("wke")
```

---

### 🔥 Q: Maximum Subarray (Kadane's Algorithm)

**Problem:** Find contiguous subarray with largest sum.

**Approach:** Kadane's algorithm — track current sum, reset if negative.

**Complexity:** O(n) time, O(1) space

```python
def max_subarray(nums):
    max_sum = nums[0]
    current_sum = nums[0]
    
    for num in nums[1:]:
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)
    
    return max_sum

# Test
print(max_subarray([-2,1,-3,4,-1,2,1,-5,4]))  # 6 ([4,-1,2,1])
```

---

## Stack

### 🔥 Q: Valid Parentheses (Balanced Parentheses)

**Problem:** Check if string with `()[]{}` is valid (properly nested).

**Approach:** Use stack to match opening/closing brackets.

**Complexity:** O(n) time, O(n) space

```python
def is_valid(s):
    stack = []
    mapping = {')': '(', ']': '[', '}': '{'}
    
    for char in s:
        if char in mapping:
            # Closing bracket
            top = stack.pop() if stack else '#'
            if mapping[char] != top:
                return False
        else:
            # Opening bracket
            stack.append(char)
    
    return len(stack) == 0

# Test
print(is_valid("()[]{}"))    # True
print(is_valid("([)]"))      # False
print(is_valid("{[]}"))      # True
```

---

### ⭐ Q: String Compression

**Problem:** Compress string using counts of repeated characters (e.g., "aabcccccaaa" → "a2b1c5a3").

**Approach:** Iterate and count consecutive characters.

**Complexity:** O(n) time, O(1) space (if modifying in-place)

```python
def compress(chars):
    write = 0
    i = 0
    
    while i < len(chars):
        char = chars[i]
        count = 0
        
        # Count consecutive same characters
        while i < len(chars) and chars[i] == char:
            i += 1
            count += 1
        
        # Write character
        chars[write] = char
        write += 1
        
        # Write count if > 1
        if count > 1:
            for digit in str(count):
                chars[write] = digit
                write += 1
    
    return write

# Test
chars = ['a','a','b','b','c','c','c']
length = compress(chars)
print(chars[:length])  # ['a','2','b','2','c','3']
```

---

## Intervals

### 🔥 Q: Merge Intervals

**Problem:** Merge overlapping intervals.

**Approach:** Sort by start time, then merge overlapping.

**Complexity:** O(n log n) time, O(n) space

```python
def merge_intervals(intervals):
    if not intervals:
        return []
    
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    
    for current in intervals[1:]:
        last = merged[-1]
        if current[0] <= last[1]:
            # Overlap — merge
            last[1] = max(last[1], current[1])
        else:
            # No overlap
            merged.append(current)
    
    return merged

# Test
print(merge_intervals([[1,3],[2,6],[8,10],[15,18]]))
# [[1,6],[8,10],[15,18]]
```

---

## Binary Search

### 🔥 Q: Binary Search

**Problem:** Find target in sorted array.

**Approach:** Classic binary search.

**Complexity:** O(log n) time, O(1) space

```python
def binary_search(nums, target):
    left, right = 0, len(nums) - 1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    
    return -1

# Test
print(binary_search([1,2,3,4,5,6], 4))  # 3
print(binary_search([1,2,3,4,5,6], 7))  # -1
```

---

## Trees & Graphs

### 🔥 Q: Number of Islands (BFS/DFS on Grid)

**Problem:** Count number of islands in a 2D grid ('1' = land, '0' = water).

**Approach:** DFS/BFS from each unvisited land cell.

**Complexity:** O(m*n) time, O(m*n) space (recursion stack)

```python
def num_islands(grid):
    if not grid:
        return 0
    
    def dfs(i, j):
        if (i < 0 or i >= len(grid) or 
            j < 0 or j >= len(grid[0]) or 
            grid[i][j] != '1'):
            return
        
        grid[i][j] = '0'  # Mark visited
        dfs(i+1, j)
        dfs(i-1, j)
        dfs(i, j+1)
        dfs(i, j-1)
    
    count = 0
    for i in range(len(grid)):
        for j in range(len(grid[0])):
            if grid[i][j] == '1':
                count += 1
                dfs(i, j)
    
    return count

# Test
grid = [
    ["1","1","0","0","0"],
    ["1","1","0","0","0"],
    ["0","0","1","0","0"],
    ["0","0","0","1","1"]
]
print(num_islands(grid))  # 3
```

---

### ⭐ Q: Palindromic Substrings

**Problem:** Count number of palindromic substrings in a string.

**Approach:** Expand around center for each possible center (n single chars + n-1 pairs).

**Complexity:** O(n²) time, O(1) space

```python
def count_substrings(s):
    def expand(left, right):
        count = 0
        while left >= 0 and right < len(s) and s[left] == s[right]:
            count += 1
            left -= 1
            right += 1
        return count
    
    total = 0
    for i in range(len(s)):
        # Odd length (single center)
        total += expand(i, i)
        # Even length (pair center)
        total += expand(i, i+1)
    
    return total

# Test
print(count_substrings("aaa"))  # 6 ("a","a","a","aa","aa","aaa")
```

---

## Heap / Priority Queue

### ⭐ Q: Array Left Rotation

**Problem:** Rotate array left by d positions.

**Approach:** Reverse three parts: [0:d], [d:n], entire array.

**Complexity:** O(n) time, O(1) space

```python
def rotate_left(arr, d):
    n = len(arr)
    d = d % n  # Handle d > n
    
    def reverse(start, end):
        while start < end:
            arr[start], arr[end] = arr[end], arr[start]
            start += 1
            end -= 1
    
    reverse(0, d-1)
    reverse(d, n-1)
    reverse(0, n-1)
    return arr

# Test
print(rotate_left([1,2,3,4,5], 2))  # [3,4,5,1,2]
```

---

## Ops-Flavored Coding Problems

### 🔥 Q: Parse Logs and Find Top K IPs

**Problem:** Given log lines with IP addresses, return top K most frequent IPs.

**Approach:** Hash map to count, heap to extract top K.

**Complexity:** O(n + k log n) time

```python
from collections import Counter
import heapq

def top_k_ips(logs, k):
    """
    logs: ["192.168.1.1 GET /api", "192.168.1.2 POST /login", ...]
    """
    ips = [line.split()[0] for line in logs]
    counts = Counter(ips)
    return heapq.nlargest(k, counts.keys(), key=counts.get)

# Test
logs = [
    "192.168.1.1 GET /api",
    "192.168.1.2 POST /login",
    "192.168.1.1 GET /health",
    "192.168.1.3 GET /",
    "192.168.1.1 POST /data"
]
print(top_k_ips(logs, 2))  # ['192.168.1.1', '192.168.1.2']
```

---

### ⭐ Q: Merge Overlapping Incident Time Intervals

**Problem:** Given incident start/end times, merge overlapping incidents.

**Approach:** Same as Merge Intervals.

**Complexity:** O(n log n) time

```python
def merge_incidents(incidents):
    """
    incidents: [(start_time, end_time), ...]
    Times in epoch seconds or ISO format (convert to comparable)
    """
    if not incidents:
        return []
    
    incidents.sort(key=lambda x: x[0])
    merged = [list(incidents[0])]
    
    for start, end in incidents[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    
    return [tuple(x) for x in merged]

# Test
incidents = [(1, 5), (3, 7), (10, 12), (11, 15)]
print(merge_incidents(incidents))  # [(1, 7), (10, 15)]
```

---

### 🔥 Q: Rate Limiter (Token Bucket)

**Problem:** Implement a token bucket rate limiter.

**Approach:** Track last refill time and token count.

**Complexity:** O(1) per allow() call

```python
import time

class RateLimiter:
    def __init__(self, capacity, refill_rate):
        """
        capacity: max tokens
        refill_rate: tokens per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()
    
    def allow(self):
        now = time.time()
        elapsed = now - self.last_refill
        
        # Refill tokens
        self.tokens = min(self.capacity, 
                         self.tokens + elapsed * self.refill_rate)
        self.last_refill = now
        
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

# Test
limiter = RateLimiter(capacity=10, refill_rate=2)  # 2 tokens/sec
print(limiter.allow())  # True
print(limiter.allow())  # True
```

**Go version:**
```go
type RateLimiter struct {
    capacity    float64
    refillRate  float64
    tokens      float64
    lastRefill  time.Time
}

func NewRateLimiter(capacity, refillRate float64) *RateLimiter {
    return &RateLimiter{
        capacity:   capacity,
        refillRate: refillRate,
        tokens:     capacity,
        lastRefill: time.Now(),
    }
}

func (r *RateLimiter) Allow() bool {
    now := time.Now()
    elapsed := now.Sub(r.lastRefill).Seconds()
    
    r.tokens = math.Min(r.capacity, r.tokens + elapsed*r.refillRate)
    r.lastRefill = now
    
    if r.tokens >= 1 {
        r.tokens -= 1
        return true
    }
    return false
}
```

---

### 🔥 Q: LRU Cache

**Problem:** Implement Least Recently Used cache with O(1) get and put.

**Approach:** Hash map + doubly linked list.

**Complexity:** O(1) get, O(1) put

```python
class Node:
    def __init__(self, key, val):
        self.key = key
        self.val = val
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> Node
        # Dummy head and tail
        self.head = Node(0, 0)
        self.tail = Node(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head
    
    def _remove(self, node):
        node.prev.next = node.next
        node.next.prev = node.prev
    
    def _add_to_head(self, node):
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node
    
    def get(self, key):
        if key not in self.cache:
            return -1
        node = self.cache[key]
        self._remove(node)
        self._add_to_head(node)
        return node.val
    
    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        
        node = Node(key, value)
        self.cache[key] = node
        self._add_to_head(node)
        
        if len(self.cache) > self.capacity:
            # Evict LRU (tail.prev)
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]

# Test
cache = LRUCache(2)
cache.put(1, 1)
cache.put(2, 2)
print(cache.get(1))    # 1
cache.put(3, 3)        # Evicts key 2
print(cache.get(2))    # -1 (not found)
```

---

## How to Approach the Coding Round

**Framework (REACTC):**

1. **Repeat/Clarify** — Restate the problem, ask about constraints (input size, edge cases, expected output format).
2. **Examples** — Walk through 2-3 examples (normal case, edge case, large input).
3. **Approach** — Describe the pattern/algorithm at a high level before coding.
4. **Code** — Write clean, correct code. Use meaningful variable names.
5. **Test** — Walk through your code with an example. Check edge cases (empty input, single element, duplicates, negative numbers).
6. **Complexity** — State time and space complexity. Discuss optimizations if asked.

**Tips:**
- **Talk out loud** — Explain your thought process as you code.
- **Brute force first** — If stuck, start with O(n²) solution, then optimize.
- **Edge cases matter** — Empty input, single element, all same, all different, negative/zero, very large n.
- **Syntax doesn't matter** — Pseudocode is fine if you're not fluent; interviewers care about logic.
- **Ask questions** — "Can I assume the array is non-empty?" "Should I handle Unicode?" "Is the input sorted?"

---

## Study Plan

**Week 1-2: Foundations**
- Arrays, hashing, two pointers (15 problems)
- Focus: Two Sum, Move Zeroes, Container With Most Water, Valid Anagram

**Week 3-4: Intermediate**
- Sliding window, stack, intervals (15 problems)
- Focus: Longest Substring, Valid Parentheses, Merge Intervals

**Week 5-6: Advanced**
- Binary search, trees/graphs, heap (15 problems)
- Focus: Binary Search variations, Number of Islands, Top K Frequent

**Week 7: Ops-flavored + Mock Interviews**
- LRU Cache, Rate Limiter, Log Parsing
- Practice on [Pramp](https://www.pramp.com) or [interviewing.io](https://interviewing.io)

**Daily:**
- 1-2 problems
- Review solutions even if you solve it (learn cleaner approaches)
- Track patterns in a notebook

---

## Key Resources

- **NeetCode** — https://neetcode.io (curated 150 problems by pattern, video solutions)
- **LeetCode Patterns** — https://seanprashad.com/leetcode-patterns
- **Grokking the Coding Interview** — https://www.educative.io/courses/grokking-coding-interview
- **Tech Interview Handbook** — https://www.techinterviewhandbook.org/algorithms/study-cheatsheet
- **LeetCode Easy/Medium** — Filter by "Top Interview Questions" and "Frequency"
- **Blind 75** — Curated list of 75 must-do problems (Google it)
- **AlgoExpert** — Paid but comprehensive (100+ problems + video explanations)
