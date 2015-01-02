title: Factorial Trailing Zeroes
date: 2014-12-30 21:58:23
tags: leetcode
categories: leetcode
---

题目如下：

	Given an integer n, return the number of trailing zeroes in n!
	Note: Your solution should be in logarithmic time complexity.
![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_QQ截图20141230230957.png)

<!-- more -->

第一种方法，先算出n的阶乘，然后通过对10循环取余得到0的个数。很明显时间复杂度为O(n)，提交之后由于超时通不过。
``` python
class Solution1:
    # @return an integer
    def trailingZeroes(self, n):
        ret = 1
        zero = 0
        for i in range(1, n+1):
            ret *= i
        while ret > 0 and ret % 10 == 0:
            ret //= 10
            zero += 1
        return zero
```

第二种方法，如果讲阶乘中所有的数都分解因子，结尾的0的个数是有5*2得到的。
由于2的个数肯定大于5的个数，所以最终问题就是求分解因子后5的个数。时间复杂度依然为O(n), 但是提交之后是能通过的。
``` python
class Solution2:
    # @return an integer
    def trailingZeroes(self, n):
        zero = 0
        for i in range(5, n+1, 5):
            while i % 5 == 0:
                i //= 5
                zero += 1
        return zero
```

第三种方法，既然已经知道了0的个数就是其中5的个数，那么可以根据下面的公式求解：

![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_ba407ce5567558a041a1f8760473aebd.png)

where k must be chosen such that

![](http://7sbqk1.com1.z0.glb.clouddn.com/youfu_blog_3ffa38f54212d1d319e13f075dd388b6.png)

下面给出一个直观的分析链接：http://www.purplemath.com/modules/factzero.htm

``` python
class Solution3:
    # @return an integer
    def trailingZeroes(self, n):
        zero = 0
        k = 0
        while (5 ** k) <= n:
            k += 1
        for i in range(1, k):
            zero += n // (5 ** i)
        return zero
```
明显第三个算法的时间复杂度为O(logN)

