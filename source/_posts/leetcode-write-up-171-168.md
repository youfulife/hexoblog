title: Excel Sheet Column Number Title
date: 2014-12-28 21:47:58
tags: leetcode
categories: leetcode
---

###LeetCode 171 题 
Related to question Excel Sheet Column Title
Given a column title as appear in an Excel sheet, return its corresponding column number.
For example:

    A -> 1
    B -> 2
    C -> 3
    ...
    Z -> 26
    AA -> 27
    AB -> 28 
<!-- more -->
解决方案如下：
``` python
class Solution:
    # @param s, a string
    # @return an integer
    def titleToNumber(self, s):
        length = len(s)
        num = 0
        j = 0
        for i in range(length-1, -1, -1):
            num += (ord(s[i])-64) * (26 ** j)
            j += 1
        return num
```
 
###LeetCode 168 题

Given a positive integer, return its corresponding column title as appear in an Excel sheet.
For example:

    1 -> A
    2 -> B
    3 -> C
    ...
    26 -> Z
    27 -> AA
    28 -> AB 
解决方案如下：
``` python
class Solution:
    # @return a string
    def convertToTitle(self, num):
        title = []
        if num > 0:
            while True:
                x = num % 26
                num //= 26
                if x == 0:
                    x = 26
                    num -= 1
                title.append(chr(x+64))
                if num == 0:
                    title.reverse()
                    return ''.join(title)
        else:
            return None
```

这两个题都是easy级别，目前官网尚未出来Solution， 上面两个答案提交是通过的。类似于10进制的一些求值，这里相当于是26进制，但是去除了0。