---
layout: post
title: 字符串的排列
tags:
- Sliding_Window
- Two_Pointers
categories: leetcode
description: 字符串的排列
---
给你两个字符串 s1 和 s2 ，写一个函数来判断 s2 是否包含 s1 的排列。如果是，返回 true ；否则，返回 false 。

换句话说，s1 的排列之一是 s2 的 子串 。


示例 1：
```
输入：s1 = "ab" s2 = "eidbaooo"
输出：true
解释：s2 包含 s1 的排列之一 ("ba").
```

示例 2：
```
输入：s1= "ab" s2 = "eidboaoo"
输出：false
```

提示：

- 1 <= s1.length, s2.length <= 104
- s1 和 s2 仅包含小写字母

## 思路一

<font color=red>滑动窗口</font>

针对s2，维护一个滑动窗口，窗口大小等于s1的宽度，只要***窗口内各个字母的出现次数与s1中各个字母的出现
次数保持一致***，则s2满足题意。

```java
class Solution {
    public boolean checkInclusion(String s1, String s2) {
        int n1 = s1.length();
        int n2 = s2.length();
        if(n1 > n2 ){
            return false;
        }

        int[] permutationS1 = new int[26];
        for(char c : s1.toCharArray()){
            permutationS1[c - 'a']++;
        }
       
        int[] permutationS2 = new int[26];
        for(int i = 0; i < n1; i++){
            char c = s2.charAt(i);
            permutationS2[c - 'a']++;
        }

        if(isEqual(permutationS1, permutationS2)){
            return true;
        }

        for(int i = 1; i <= n2 - n1; i++){
            int left = i;
            int right = left + n1 - 1;
            permutationS2[s2.charAt(left - 1) - 'a']--;
            permutationS2[s2.charAt(right) - 'a']++;
            if(isEqual(permutationS1, permutationS2)){
                return true;
            }
        }
        return false;
    }

    public boolean isEqual(int[] a, int[] b){
        if(a.length != b.length){
            return false;
        }
        for(int i = 0; i < a.length; i++){
            if(a[i] != b[i]){
                return false;
            }
        }
        return true;
    }
}
```

> 时间复杂度

S2的每个字符需要遍历一次(On))，遍历每个字符时，都需要判断窗口是否满足题意(O1)，故时间复杂度为O(n)。

***纠正***，S1也遍历了一次，应该是O(n1+n2)

## 思路二
针对思路一进行优化，对于每个窗口，都判断了一次是否与s1中各字母出现频次一致。

我们可以维护两个变量，
- 一个数组`cnt`，索引为字母编号(从0开始)，值为窗口内字母出现频次 - s1中相应字母出现频次。
- 一个标记int differ， 表示字母出现频次不同的字母个数。

## 思路三
<font color=red>双指针</font>

目标是找到一个`cnt`数组，元素个数为n1，且各个元素值都为0。

反过来，还可以在保证 `cnt`的值不为正的情况下，去考察是否存在一个区间，其长度恰好为 `n`。


>s1: abf
>
>s2: abgbaf

## 参考

- [字符串的排列](https://leetcode.cn/problems/permutation-in-string/)
- [最小覆盖子串](https://leetcode.cn/problems/minimum-window-substring/)

