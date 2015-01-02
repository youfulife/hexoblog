title: Valid Palindrome
date: 2015-01-02 22:11:16
tags: leetcode
categories: leetcode
---

LeetCode 125题

原题如下：

	Given a string, determine if it is a palindrome, considering only alphanumeric characters and ignoring cases.
	For example,
	"A man, a plan, a canal: Panama" is a palindrome.
	"race a car" is not a palindrome.

	Note:
	Have you consider that the string might be empty? This is a good question to ask during an interview.
	For the purpose of this problem, we define empty string as valid palindrome.

<!-- more -->
这个题是笔试题中最常见的字符串相关题目，解题思路也非常简单，两个指针，一个指向开头，一个指向结尾，开始同时向中间移动，遇到非字母数字就跳过，并且做对比，直到两个指针相遇。
虽然思路比较明确，但是一些细节还是需要把握好，尤其是退出循环后对两个指针是否是合法字符的判断。
代码如下：
``` python
class Solution:
    # @param s, a string
    # @return a boolean
    def isPalindrome(self, s):
        if len(s) == 0:
            return True
        else:
            j = len(s) - 1
        i = 0
        while i < j:
            if s[i].lower() != s[j].lower():
                if s[i].isalnum() and s[j].isalnum():
                    return False
                if not s[i].isalnum():
                    i += 1
                if not s[j].isalnum():
                    j -= 1
            else:
                i += 1
                j -= 1
        if s[i].lower() == s[j].lower() or not s[i].isalnum() or not s[j].isalnum():
            return True
        else:
            return False
```
