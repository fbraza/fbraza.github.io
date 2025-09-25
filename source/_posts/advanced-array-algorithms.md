---
title: When arrays make me cry
summary: A brief overview of some advanced array algorithms
date: "2025-09-22"
tags:
    - Data structure
    - Algorithms
---

In a previous article we quickly glanced through array fundamentals. We saw how array are presented in RAM, the difference between static and dynamic arrays and how an array can be used to build a data structure called a stack. Here we are going to dive a bit more deeper into different algortihmic patterns for arrays and their solutions.

## The Kadane's algorithm

Imagine you have a list of numbers, something like daily changes in your bank account. We have some positive (you earned money) and some negative (you spent money). You want to find the best consecutive sequence of days where your total gain was the highest. This is exactly what the Kadane’s Algorithm does. Instead of checking every possible sequence (which would be slow), it scans the list just once. At each step, it asks:

- Is it better to start a new sequence here?
- Or to add this number to the sequence I’m already building?”

It keeps track of the highest sum it has seen so far and, by the end of the list, we get our answer.

This algorithm while simple is widely used in finance, signal processing, or anywhere you need to find the “best contiguous segment” in a series of numbers.

### The brute force

In the leetcode language, problem should be framed as followed.

> Find a non-empty sub-array with the largest sum

Given the array `[4, -1, 2, -7, 3, 4]` we can quikcly find a brute force approach using two pointers `i` and `j`.

```python
def brut_force(nums: list[int]) -> int:
    max_sum = nums[0]

    for i in range(len(nums)):
        current_sum = 0
        for j in range(i, len(nums)):
            current_sum += nums[j]
            max_sum = max(max_sum, current_sum)

    return max_sum
```

The tome complexity of this algorithm is `O(n^2)`. Each loop is `O(n)` so we have `O(n * n)`.

### The kadane's way

The idea behind the kadane's algorithm is pretty simple. If your sum is below `0`. Reset your max to `0` and keep adding the element of the list until the end. The code will look like this.

```python
def kadane(nums: list[int]) -> int:
    max_sum = nums[0]
    current_sum = 0

    for num in nums:
        current_sum = max(0, current_sum + num)
        max_sum = max(current_sum, 0)

    return max_sum
```

This algorithm is very smart and allows you to obtain the answer in `O(n)`. Simple and efficient.

### Let's suffer

A set of exercises to practice the concept.

- [maximum-subarray](https://leetcode.com/problems/maximum-subarray/)
- [maximum-sum-circular-subarray](https://leetcode.com/problems/maximum-sum-circular-subarray/)
- [longest-turbulent-subarray](https://leetcode.com/problems/longest-turbulent-subarray/)

## Sliding window

The Kadane’s algorithm finds the maximum sum of any contiguous subarray in one pass. But looking at it closer, you may realize that what the code is doing is ust managing a window.
Indeed, t does not recalculate from zero every time. Instead, it evaluates to “Keep going… or start fresh here?” If the running sum is good (namely based on your rules, before was not `<` to `0`), it extends. If it goes below `0`, it resets. This is the pattern of a sliding with the right edge always moving ahead. We can acutally rewrite the kadane algorithm by expleciting the widnow:

```python

def kadane_with_sliding(nums: list[int]) -> list[int]:
    """
    return the index of the sliding window with the highest sum
    """
    max_sum = nums[0]
    cur_sum = 0
    max_left, max_right = 0, 0
    left = 0

    for right in range(len(nums)):
        if cur_sum < 0:
            cur_sum = 0
            left = right

        cur_sum += nums[right]
        if cur_sum > 0:
            max_sum = cur_sum
            max_left, max_right = left, right

    return [max_left, max_right]
```

And once you see it here, you will see windows everywhere:

- Longest substring without repeats? Sliding window.
- Max average over k elements? Sliding window.
- Highest-scoring DNA segment? Sliding window again.

### Problems with static windows

In some situation your windows will have a defined size. A typical problem maybe:

>Given an array, return True if there are two element withing a window of size k that are equal

There are ways of brute forcing it with two `for` loops but the trick here to use the right data structure: a hashSet (`set()` in python).

>In Python, the set data structure is an unordered collection of unique elements. It is implemented using a hash table, which allows for average-case constant time complexity (O(1)) for operations such as insertions, deletions, and membership tests. **REMEMBER:** `O(1)` does not mean "one operation". It means that the speed of operation is not dependent of the number of element present in your data structure.

```python
def find_duplicate(nums: list[int]):
    window = set()
    left = 0

    for right in range(len(nums)):
        if right - left + 1 > k:
            window.remove(nums[left])
            left += 1
        if nums[right] in window:
            return True
        window.add(nums[right])

    return False
```

We could acutally have done the same with a dictionnary.

```python
def find_duplicate(nums: list[int]):
    window = {}
    left = 0

    for right in range(len(nums)):
        if right - left + 1 > k:
            del window[nums[left]]
            left += 1
        if window.get(nums[right], False):
            return True
        window[nums[right]] = True

    return False
```

### Practice with these exercises

- [contains-duplicate-ii](https://leetcode.com/problems/contains-duplicate-ii/)
- [number-of-sub-arrays-of-size-k-and-average-greater-than-or-equal-to-threshold](https://leetcode.com/problems/number-of-sub-arrays-of-size-k-and-average-greater-than-or-equal-to-threshold/)

### Problems with dynamic windows

At Sunrise, I built an algorithm to measure hypoxic burden. The idea was to scan the SpO2 values. For that we used a dymanic windows:

- When SpO2 drops below a threshold (e.g., 90%)
- Next, we keep it open as long as levels stay low
- Then we close it only when oxygen recovers and stabilizes.
- Finally we calculate severity (area under the curve) before resetting the window.

Knowing how to play with dynamic sliding window is very convenient and can be applied to many other situations:

- You need to fin enriched regions in ChIP-seq or ATAC-seq data? Slide a window, but adjust its size based on local signal-to-noise.
- You are detecting bursts of activity? Expand the window while traffic is high for analysis and contract it when it the traffic descrease.

This pattern is common. Let's see some basic problem examples and the code to solve them.

>Find the length of the longest subarray with identical values

```python
def find_longest_subarray(nums: list[int]):
    length = 0
    left = 0

    for right in range(len(nums)):
        if nums[left] != nums[right]:
            left = right

        length = max(length, right - left + 1)

    return length
```

>Find the length of the smallest subarray where the sum of its elements is greater than or equal to a given target. Assume all values are positive.

```python
def find_min_subarray(nums: list[int], target: int):
    length = float('inf')
    left = 0
    current_sum = 0

    for right in range(len(nums)):
        current_sum += nums[right]

        while current_sum >= target:
            length = min(length, right - left + 1)
            current_sum -= nums[left]
            left += 1

    return length if length != float('inf') else 0
```

### Suffering a bit with these exercises

- [minimum-size-subarray-sum](https://leetcode.com/problems/minimum-size-subarray-sum/)
- [longest-substring-without-repeating-characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)
- [longest-repeating-character-replacement](https://leetcode.com/problems/longest-repeating-character-replacement/)
