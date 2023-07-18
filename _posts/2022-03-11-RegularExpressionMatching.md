---
layout: post
title: 正则表达式匹配
tags:
- DP
categories: leetcode
description: 正则表达式匹配
---
给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。

'.' 匹配任意单个字符
'*' 匹配零个或多个前面的那一个元素
所谓匹配，是要涵盖 整个 字符串 s的，而不是部分字符串。


示例 1：

>输入：s = "aa", p = "a"
>输出：false
>解释："a" 无法匹配 "aa" 整个字符串。

示例 2:

>输入：s = "aa", p = "a*"
>输出：true
>解释：因为 '*' 代表可以匹配零个或多个前面的那一个元素, 在这里前面的元素就是 'a'。因此，字符串 "aa" 可被视为 'a' 重复了一次。

示例 3：

>输入：s = "ab", p = ".*"
>输出：true
>解释：".*" 表示可匹配零个或多个（'*'）任意字符（'.'）。

提示：

- 1 <= s.length <= 20
- 1 <= p.length <= 20
- s 只包含从 a-z 的小写字母。
- p 只包含从 a-z 的小写字母，以及字符 . 和 *。
- 保证每次出现字符 * 时，前面都匹配到有效的字符

## 分析

- 记s的长度为M, p的长度为N。
- 定义f[i][j] 表示s的前i个字符与p的前j个字符是否匹配。
  - f[0][0]=true，s的前0个字符是空串，p也是，空串与空串是匹配的。
  - f[M][N]就是`最终答案`。
- 外层遍历s，下标i从0到M；内层遍历p，下标j从1到N。
  - 如果p[j-1]不是特殊字符`*`，常规计算f[i][j]。
  - 如果p[j-1]是特殊字符`*`
    - 丢弃p[j-1]和p[j-2]两个字符，与s[i]匹配。即`f[i][j] = f[i][j-2]`;
    - 如果p[j-2]与s[i-1]相等，将s[i-1]丢弃，再与p[j]匹配，即`f[i][j] |= f[i-1][j]`。

## 代码
```java
class Solution {
    public boolean isMatch(String s, String p) {
       int M = s.length();
       int N = p.length();
       boolean[][] f = new boolean[M+1][N+1];
       // 空串是相等的
       f[0][0] = true;
       for(int i = 0; i <= M; i++){
           for(int j = 1; j <= N; j++){
               // 第i个字符与第j个字符比较

               if(p.charAt(j-1) != '*'){
                   // 如s= xzzz a ; p=xz* a
                   f[i][j] = equal(s, p , i, j)&&f[i-1][j-1];
               }else{
                   f[i][j] = f[i][j-2];
                   if(equal(s, p, i, j-1)){
                       // 如s=   taaa ;  p=  ta*  ;
                       f[i][j] |= f[i-1][j];
                   }
               }
           }
       }
       return f[M][N];
    }
    // s的第i个字符与p的第j个字符是否匹配，i，j都从1开始计算
    // s的第0个字符只与p的第0个字符匹配，除外，s的第0个字符不再与p匹配
    public boolean equal(String s, String p, int i, int j){
        if( i == 0 || j == 0){
            return false;
        }
        if(p.charAt(j-1) == '.'){
            return true;
        }
        return s.charAt(i-1) == p.charAt(j-1);

    }
}
```


## 参考
- [正则表达式匹配](https://leetcode.cn/problems/regular-expression-matching/)
