title: Rotate Array 
date: 2015-03-04 15:08:27
tags: leetcode
categories: leetcode
---

Rotate an array of n elements to the right by k steps.
For example, with n = 7 and k = 3, the array [1,2,3,4,5,6,7] is rotated to [5,6,7,1,2,3,4].

Note:
Try to come up as many solutions as you can, there are at least 3 different ways to solve this problem.
<!-- more -->
练习python中，使用python来做几行代码可以搞定，当然可能性能会差一点：

第一次提交：
33 / 33 test cases passed.
Status: Accepted
Runtime: 93 ms

``` Python
class Solution:
    # @param nums, a list of integer
    # @param k, num of steps
    # @return nothing, please modify the nums list in-place.
    def rotate(self, nums, k):
        length = len(nums)
        k = k % length
        temp1 = nums[-k:]
        temp2 = nums[:(length-k)]
        temp3 = temp1 + temp2
        for i in range(length):
            nums[i] = temp3[i]
```
稍微修改了一点，减少了一次遍历，性能提高了大概10%：
33 / 33 test cases passed.
Status: Accepted
Runtime: 85 ms

``` Python
class Solution:
    # @param nums, a list of integer
    # @param k, num of steps
    # @return nothing, please modify the nums list in-place.
    def rotate(self, nums, k):
        length = len(nums)
        k %= length
        temp = nums[-k:]
        temp2 = nums[:(length-k)]
        for i in range(k):
            nums[i] = temp[i]
        for i in range(length-k):
            nums[i+k] = temp2[i]
```
