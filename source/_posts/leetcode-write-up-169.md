title: Majority Element
date: 2014-12-28 22:08:37
tags: leetcode
categories: leetcode
---
LeetCode 169 题：

Given an array of size n, find the majority element. The majority element is the element that appears more than ⌊ n/2 ⌋ times. You may assume that the array is non-empty and the majority element always exist in the array.

找出一个数组中超过半数的那个元素。可以假设数组非空，而且一定有一个元素个数超过数组总数的一半。

这个问题有一个专门的算法来解决 **Moore voting algorithm**, 看了之后感觉神清气爽。具体算法细节和数学证明请自行google。
先来最经典的解决方案：
``` python
class Solution:
    # @param num, a list of integers
    # @return an integer
    def majorityElement(self, num):
        elem = self.find_element(num)
        count = 0
        for x in num:
            if x == elem:
                count += 1
        if count > len(num)//2:
            return elem
        else:
            return None

    def find_element(self, L):
        mark = 0
        for i in range(len(L)):
            if mark == 0:
                mark = 1
                elem = L[i]
            elif elem == L[i]:
                mark += 1
            else:
                mark -= 1
        return elem
```
<!-- more -->
下面是LeetCode上给出的各种解决方案：
Majority Element
1. **Runtime: O(n2) — Brute force solution**: Check each element if it is the majority element.
2. **Runtime: O(n), Space: O(n) — Hash table**: Maintain a hash table of the counts of each element, then find the most common one.
3. **Runtime: O(n log n) — Sorting**: As we know more than half of the array are elements of the same value, we can sort the array and all majority elements will be grouped into one contiguous chunk. Therefore, the middle (n/2th) element must also be the majority element.
4. **Average runtime: O(n), Worst case runtime: Infinity — Randomization**: Randomly pick an element and check if it is the majority element. If it is not, do the random pick again until you find the majority element. As the probability to pick the majority element is greater than 1/2, the expected number of attempts is < 2.
5. **Runtime: O(n log n) — Divide and conquer**: Divide the array into two halves, then find the majority element A in the first half and the majority element B in the second half. The global majority element must either be A or B. If A == B, then it automatically becomes the global majority element. If not, then both A and B are the candidates for the majority element, and it is suffice to check the count of occurrences for at most two candidates. The runtime complexity, T(n) = T(n/2) + 2n = O(n log n).
6. **Runtime: O(n) — Moore voting algorithm**: We maintain a current candidate and a counter initialized to 0. As we iterate the array, we look at the current element x:

    If the counter is 0, we set the current candidate to x and the counter to 1.
    If the counter is not 0, we increment or decrement the counter based on whether x is the current candidate.
After one pass, the current candidate is the majority element. Runtime complexity = O(n).
7. **Runtime: O(n) — Bit manipulation**: We would need 32 iterations, each calculating the number of 1's for the ith bit of all n numbers. Since a majority must exist, therefore, either count of 1's > count of 0's or vice versa (but can never be equal). The majority number’s ith bit must be the one bit that has the greater count.
**Update (2014/12/24)**: Improve algorithm on the O(n log n) sorting solution: We do not need to 'Find the longest contiguous identical element' after sorting, the n/2th element is always the majority.

给出一个提交通过的Hash table的算法：
``` python
class Solution2:
    # @param num, a list of integers
    # @return an integer
    def majorityElement(self, num):
        dict1 = {}
        for x in num:
            if x in dict1:
                dict1[x] += 1
            else:
                dict1[x] = 1
        values = dict1.values()
        max_value = max(values)
        for x in dict1:
            if dict1[x] == max_value:
                return x
```
给出一个由于超时未通过的Bitmap的算法，注意bitmap算法要考虑有负数的特殊情况：
``` python
#Runtime: O(n) — Bit manipulation
class Solution7:

    def majorityElement(self, num):
        major = 0
        mid = len(num) // 2
        for i in range(32):
            count_1bit = 0
            for x in num:
                if x < 0:
                    x = abs(x)
                count_1bit += (x >> i) & 0x1
            if count_1bit > mid:
                major += 2 ** i
        negative_count = 0
        for x in num:
            if x < 0:
                negative_count += 1
        if negative_count > mid:
            major -= major * 2
        return major
```
先进行快速排序也没有通过：
``` python
# Time Limit Exceeded
#Runtime: O(n log n) — Sorting
class Solution2:
    # @param num, a list of integers
    # @return an integer
    def majorityElement(self, num):
        self.quick_sort(num, 0, len(num))
        return num[len(num)//2]

    def quick_sort(self, L, low, high):
        if low == high:
            return
        store_index = low+1
        for i in range(low+1, high):
            if L[i] < L[low]:
                L[i], L[store_index] = L[store_index], L[i]
                store_index += 1
        L[low], L[store_index-1] = L[store_index-1], L[low]
        self.quick_sort(L, low, store_index-1)
        self.quick_sort(L, store_index, high)
```
这个问题出现在很多公司的面试题中，包括《编程之美》中也有这个题目的变形， 而且在现实项目中也经常会用到类似的要求。