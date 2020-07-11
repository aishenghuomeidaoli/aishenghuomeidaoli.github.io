---
title: LeetCode算法
date: 2019-03-30 20:01:50
categories:
- 技术
- 算法
tags:
- 算法
---

此篇文章中文题干来自[LeetCode中国](https://leetcode-cn.com)，算法答案来自[Github-apachecn/awesome-algorithm](https://github.com/apachecn/awesome-algorithm)。

<!--more-->

## 1. [两数之和](https://leetcode-cn.com/problems/two-sum/)

给定一个整数数组`nums`和一个目标值`target`，请你在该数组中找出和为目标值的那两个整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

示例:

> 给定 nums = [2, 7, 11, 15], target = 9
因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]

解题方案:

* 时间复杂度O(N)，空间复杂度: O(N)


```
          2        7        11    15
         不存在   存在之中
lookup   {2:0}    [0，1]
```
建立字典 lookup 存放第一个数字，并存放该数字的 index
判断 lookup 中是否存在： target - 当前数字， 则表面 当前值和 lookup中的值加和为 target.
如果存在，则返回： target - 当前数字 的 index 和 当前值的 index

```python
class Solution(object):
    def twoSum(self, nums, target):
        """
        :type nums: List[int]
        :type target: int
        :rtype: List[int]
        """
        lookup = {}
        for i, num in enumerate(nums):
            if target - num in lookup:
                return [lookup[target-num], i]
            else:
                lookup[num] = i
```

## 2. [两数相加](https://leetcode-cn.com/problems/add-two-numbers/)

给出两个 非空 的链表用来表示两个非负的整数。其中，它们各自的位数是按照 逆序 的方式存储的，并且它们的每个节点只能存储 一位 数字。

如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。

您可以假设除了数字 0 之外，这两个数都不会以 0 开头。

示例：

>输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807

解题方案:

* 时间复杂度O(N)，空间复杂度O(1)

使用递归，每次算一位的相加

```python
class Solution:
    def addTwoNumbers(self, l1, l2):
        """
        :type l1: ListNode
        :type l2: ListNode
        :rtype: ListNode
        """
        if not l1 and not l2:
            return 
        elif not (l1 and l2):
            return l1 or l2
        else:
            if l1.val + l2.val < 10:
                l3 = ListNode(l1.val+l2.val)
                l3.next = self.addTwoNumbers(l1.next, l2.next)
            else:
                # l1.val 与 l2.val 相加，向下一位进位，即l1.next, l2.next, ListNode(1)相加
                l3 = ListNode(l1.val+l2.val-10)
                l3.next = self.addTwoNumbers(l1.next, self.addTwoNumbers(l2.next, ListNode(1)))
        return l3
```

## 3. [无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

* 思路1，时间复杂度O(N)，空间复杂度O(N)，slide window

```python
class Solution:
    def lengthOfLongestSubstring(self, s):
        """
        :type s: str
        :rtype: int
        """
        if not s or len(s) == 0:
            return 0
        
        l, r = 0, 0
        res, lookup = 0, set()
        while l < len(s) and r < len(s):
            if s[r] not in lookup:
                lookup.add(s[r])
                res = max(res, r-l+1)
                r += 1
            else:
                # set.discard 移除元素，元素不存在也不报错
                lookup.discard(s[l])
                l += 1
        return res
```

* 思路2，时间复杂度O(N)，空间复杂度O(N)

逐个字符迭代，使用hashmap，将每个迭代的字符作为键，索引为值进行存储，入当前字符出现过，则将上一次重复字符的后一位作为起点重新计算。

例如‘alxyzxlkjh’：

遍历至z时，起点索引为0，最大长度为5；

遍历至第二个x时，获取重复的x索引2，可将其后的字符作为起点，即起点索引为2+1，重新计算长度为3，与长度5比较；

遍历至第二个l时，如果将上一个l索引+1作为起点，则可能包含重复字符（x），所以当前迭代的起点索引为 上一个重复字符索引+1 与 上一轮迭代起点索引的最大值；

程序变量解释:

`res` 代表最大子串的长度;

`maps` 遍历时存放每一个字符的 index

`start` 是这一轮迭代的起点索引，若当前字符出现过，则start = maps.get(s[i]) + 1，若未出现过，start = 0，可用`maps.get(s[i], -1) + 1`表示。由于可能包含其他重复子串，所以 start = max(start, maps.get(s[i], -1) + 1)


当前子串长度为 i - start + 1, 所以只要求得start即可，
```python
class Solution:
    def lengthOfLongestSubstring(self, s):
        """
        :type s: str
        :rtype: int
        """
        res, start, n = 0, 0, len(s)
        maps = {}
        for i in range(n):
            start = max(start, maps.get(s[i], -1)+1)
            res = max(res, i - start + 1)
            maps[s[i]] = i
        return res
```

## 4. [寻找两个有序数组的中位数](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/)

给定两个大小为 m 和 n 的有序数组 nums1 和 nums2。

请你找出这两个有序数组的中位数，并且要求算法的时间复杂度为 O(log(m + n))。

你可以假设 nums1 和 nums2 不会同时为空。

示例 1:

nums1 = [1, 3]

nums2 = [2]

则中位数是 2.0

示例 2:

nums1 = [1, 2]

nums2 = [3, 4]

则中位数是 (2 + 3)/2 = 2.5

解决方案：

首先转成求A和B数组中第k小的数的问题, 然后用k/2在A和B中分别找。

比如k = 6, 分别看A和B中的第3个数, 已知 A1 < A2 < A3 < A4 < A5... 和 B1 < B2 < B3 < B4 < B5..., 如果A3 <＝ B3, 那么第6小的数肯定不会是A1, A2, A3, 因为最多有两个数小于A1, 三个数小于A2, 四个数小于A3。 关键点是从 k/2 开始来找。

B3至少大于5个数, 所以第6小的数有可能是B1 (A1 < A2 < A3 < A4 < A5 < B1), 有可能是B2 (A1 < A2 < A3 < B1 < A4 < B2), 有可能是B3 (A1 < A2 < A3 < B1 < B2 < B3)。那就可以排除掉A1, A2, A3, 转成求A4, A5, ... B1, B2, B3, ...这些数中第3小的数的问题, k就被减半了。每次都假设A的元素个数少, pa = min(k/2, lenA)的结果可能导致k == 1或A空, 这两种情况都是终止条件。

发问，为什么要从k/2开始寻找，依旧k = 6, 我可以比较A1 和 B5的关系么，可以这样做，但是明显的问题出现在如果A1 > B5，那么这个第6小的数应该存在于B6和A1中。

如果A1 < B5，这个时间可能性就很多了，比如A1 < A2 < A3 < A4 < B1 < B2，各种可能，无法排除元素，所以还是要从k/2开始寻找。

这个跟习题算法的区别是每次扔的东西明显少一些，但是k也在不断变小。下面的代码的时间复杂度是O(lg(m+n))

```python
class Solution(object):
    def findMedianSortedArrays(self, nums1, nums2):
        """
        :type nums1: List[int]
        :type nums2: List[int]
        :rtype: float
        """
        def findKth(A, B, k):
            if len(A) == 0:
                return B[k-1]
            if len(B) == 0:
                return A[k-1]
            if k == 1:
                return min(A[0], B[0])

            a = A[k // 2 - 1] if len(A) >= k // 2 else None
            b = B[k // 2 - 1] if len(B) >= k // 2 else None

            if b is None or (a is not None and a < b):
                return findKth(A[k // 2:], B, k - k // 2)
            return findKth(A, B[k // 2:], k - k // 2)  # 这里要注意：因为 k/2 不一定 等于 (k - k/2)
            
        n = len(nums1) + len(nums2)
        if n % 2 == 1:
            return findKth(nums1, nums2, n // 2 + 1)
        else:
            smaller = findKth(nums1, nums2, n // 2)
            bigger = findKth(nums1, nums2, n // 2 + 1)
            return (smaller + bigger) / 2.0
```

## 5. [最长回文子串](https://leetcode-cn.com/problems/longest-palindromic-substring/)

给定一个字符串 s，找到 s 中最长的回文子串。你可以假设 s 的最大长度为 1000。

示例 1：

输入: "babad"
输出: "bab"
注意: "aba" 也是一个有效答案。
示例 2：

输入: "cbbd"
输出: "bb"

解决方案：
思路1， 时间复杂度: O(N) 空间复杂度: O(N)

[Manacher算法](https://www.felix021.com/blog/read.php?2040)

```python
class Solution(object):
    def longestPalindrome(self, s):
        """
        :type s: str
        :rtype: str
        """
        def preProcess(s):
            if not s:
                return ['^', '&']
            T = ['^']
            for i in s:
                T += ['#', i]
            T += ['#', '$']
            return T
        T = preProcess(s)
        P = [0] * len(T)
        id, mx = 0, 0
        for i in range(1, len(T)-1):
            j = 2 * id - i
            if mx > i:
                P[i] = min(P[j], mx-i)
            else:
                P[i]= 0
            while T[i+P[i]+1] == T[i-P[i]-1]:
                P[i] += 1
            if i + P[i] > mx:
                id, mx = i, i + P[i]
        max_i = P.index(max(P))    #保存的是当前最大回文子串中心位置的index
        start = (max_i - P[max_i] - 1) // 2
        res = s[start:start+P[max_i]]
        return res
```
