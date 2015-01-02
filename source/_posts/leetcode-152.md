title: Maximum Product Subarray 
date: 2015-01-02 22:50:03
tags: leetcode
categories: leetcode
---

LeetCode 152题

原题如下：

	Find the contiguous subarray within an array (containing at least one number) which has the largest product.

	For example, given the array [2,3,-2,4],
	the contiguous subarray [2,3] has the largest product = 6.
	
最长子序列问题第一想到的就是动态规划法。LeetCode给出的 Solution 也是动态规划。

Besides keeping track of the largest product, we also need to keep track of the smallest product. Why? The smallest product, which is the largest in the negative sense could become the maximum when being multiplied by a negative number.
<!-- more -->
Let us denote that:

	f(k) = Largest product subarray, from index 0 up to k.
Similarly,

	g(k) = Smallest product subarray, from index 0 up to k.
Then,

	f(k) = max( f(k-1) * A[k], A[k], g(k-1) * A[k] )
	g(k) = min( g(k-1) * A[k], A[k], f(k-1) * A[k] )
There we have a dynamic programming formula. Using two arrays of size n, we could deduce the final answer as f(n-1). Since we only need to access its previous elements at each step, two variables are sufficient.

代码如下：
``` python
class Solution:
    # @param A, a list of integers
    # @return an integer
    def maxProduct(self, A):
        if A:
            if len(A) == 1:
                return A[0]
            else:
                cur_max = 1
                cur_min = 1
                result = 0
                for i in range(len(A)):
                    tmp_max = max_three(A[i], cur_max * A[i], cur_min * A[i])
                    tmp_min = min_three(A[i], cur_min * A[i], cur_max * A[i])
                    cur_max = tmp_max
                    cur_min = tmp_min
                    result = max(result, cur_max)
                return result
        else:
            return None
            
def max_three(a, b, c):
    max = a if a>b else b
    if c > max:
        max = c
    return max


def min_three(a, b, c):
    min = a if a<b else b
    if c < min:
        min = c
    return min
```