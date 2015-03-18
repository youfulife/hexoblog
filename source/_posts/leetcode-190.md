title: Reverse Bits
date: 2015-03-18 13:10:11
tags: leetcode
categories: leetcode
---

Reverse bits of a given 32 bits unsigned integer.

For example, given input 43261596 (represented in binary as 00000010100101000001111010011100), return 964176192 (represented in binary as 00111001011110000010100101000000).

Follow up:
If this function is called many times, how would you optimize it?
<!-- more -->
一开始想了一个非常复杂的方法，东拼西凑终于搞出来了，写完了就忘了怎么写的了，性能也不是很好。

600 / 600 test cases passed.
Status: Accepted
Runtime: 66 ms
Submitted: 14 hours, 3 minutes ago

``` python
class Solution:
    # @param n, an integer
    # @return an integer
    def reverseBits(self, n):
        for i in range(16):
            low = (n & (0x1 << i)) >> i
            high = (n & (0x1 << (31-i))) >> (31-i)
            if low != high:
                if low == 1:
                    n |= (low << (31 - i))
                    n &= ~(0x1 << i)
                else:
                    n &= ~(0x1 << (31 - i))
                    n |= (high << i)
        return n
```

后来又想了一个比较简单的方法：

``` python
class Solution:
    # @param n, an integer
    # @return an integer
    def reverseBits(self, n):
        s = 0
        for i in range(32):
            if (n & (0x1 << i)) >> i:
                s += 0x1 << (31-i)
        return s
```

第二种性能要好一些：
600 / 600 test cases passed.
Status: Accepted
Runtime: 52 ms
Submitted: 13 hours, 44 minutes ago

