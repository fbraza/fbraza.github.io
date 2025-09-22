---
title: When arrays make me cry
summary: A brief overview of some advanced array algorithms
date: "2025-09-22"
tags:
    - Data structure
    - Algorithms
---

In a previous article we quickly glance through array fundamentals. We saw how array are presented in RAM, the difference between static and dynamic arrays and how an array can be ued to build a data structure called stack. Here we are going to dive a bit more deeper into different algortihmic pattern that how can used when dealing with arrays.

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

Please feel free to play around with the embedded python tutor below.

<iframe
  width="800"
  height="500"
  frameborder="0"
  src="https://pythontutor.com/iframe-embed.html#code=def%20brut_force%28nums%3A%20list%5Bint%5D%29%20-%3E%20int%3A%0A%20%20%20%20max_sum%20%3D%20nums%5B0%5D%0A%20%20%20%20%0A%20%20%20%20for%20i%20in%20range%28len%28nums%29%29%3A%0A%20%20%20%20%20%20%20%20current_sum%20%3D%200%0A%20%20%20%20%20%20%20%20for%20j%20in%20range%28i,%20len%28nums%29%29%3A%0A%20%20%20%20%20%20%20%20%20%20%20%20current_sum%20%2B%3D%20nums%5Bj%5D%0A%20%20%20%20%20%20%20%20%20%20%20%20max_sum%20%3D%20max%28max_sum,%20current_sum%29%0A%20%20%20%20%0A%20%20%20%20return%20max_sum%0A%20%20%20%20%0A%0Anums%20%3D%20%5B4,%20-1,%202,%20-7,%203,%204%5D%0Abrut_force%28nums%29&codeDivHeight=400&codeDivWidth=350&cumulative=false&curInstr=25&heapPrimitives=nevernest&origin=opt-frontend.js&py=311&rawInputLstJSON=%5B%5D&textReferences=false">
</iframe>

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

### To practice

- https://leetcode.com/problems/maximum-subarray/
- https://leetcode.com/problems/maximum-sum-circular-subarray/
- https://leetcode.com/problems/longest-turbulent-subarray/