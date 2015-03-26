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

``` python
class Solution:
    # @param n, an integer
    # @return an integer
    def reverseBits(self, n):
        n = (n & 0x55555555) << 1 | (n & 0xAAAAAAAA) >> 1
        n = (n & 0x33333333) << 2 | (n & 0xCCCCCCCC) >> 2
        n = (n & 0x0F0F0F0F) << 4 | (n & 0xF0F0F0F0) >> 4
        n = (n & 0x00FF00FF) << 8 | (n & 0xFF00FF00) >> 8
        n = (n & 0x0000FFFF) << 16 | (n & 0xFFFF0000) >> 16
        return n
```

二分法求解，两个bit一组，对调其中相邻bit，然后再4个一组，编为2段，每个段有两个bit，每个段作为一个整体，对调相邻两段，以此类推。
性能再次提高：
600 / 600 test cases passed.
Status: Accepted
Runtime: 44 ms
Submitted: 0 minutes ago

更高效的查表法：
```
   static const unsigned char BitReverseTable256[256] = 
{
    #define R2(n)     n,     n + 2*64,     n + 1*64,     n + 3*64
    #define R4(n)     R2(n), R2(n + 2*16), R2(n + 1*16), R2(n + 3*16)
    #define R6(n)     R4(n), R4(n + 2*4 ), R4(n + 1*4 ), R4(n + 3*4 )
    R6(0), R6(2), R6(1), R6(3)
};
uint32_t reverseBits(uint32_t n) {
	uint32_t c;
	unsigned char * p = (unsigned char *) &n;
	unsigned char * q = (unsigned char *) &c;
	q[3] = BitReverseTable256[p[0]]; 
	q[2] = BitReverseTable256[p[1]]; 
	q[1] = BitReverseTable256[p[2]]; 
	q[0] = BitReverseTable256[p[3]];
	return c;
}
```
600 / 600 test cases passed.
Status: Accepted
Runtime: 4 ms
Submitted: 0 minutes ago

类似的各种bit hack技巧可以查书《hacker's delight》还有文章《Bit Twiddling Hacks》，里面各种各样的奇技淫巧。
