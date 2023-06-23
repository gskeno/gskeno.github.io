---
layout: post
title: 移掉K位数字
tags:
- Monotonic_Queue
- Greedy
categories: leetcode
description: 移掉K位数字保证值最小
---
给你一个以字符串表示的非负整数 num 和一个整数 k ，移除这个数中的 k 位数字，使得剩下的数字最小。请你以字符串形式返回这个最小的数字。

示例 1 ：
```
输入：num = "1432219", k = 3
输出："1219"
解释：移除掉三个数字 4, 3, 和 2 形成一个新的最小的数字 1219 。
```

示例 2 ：
```
输入：num = "10200", k = 1
输出："200"
解释：移掉首位的 1 剩下的数字为 200. 注意输出不能有任何前导零。
```

示例 3 ：
```
输入：num = "10", k = 2
输出："0"
解释：从原数字移除所有的数字，剩余为空就是 0 。
```

提示：

>- 1 <= k <= num.length <= 105
>
>- num 仅由若干位数字（0 - 9）组成
>
>- 除了 0 本身之外，num 不含任何前导零

----

## 分析
原来的非负正数可以用下图来表示

<img src="/assets/img/leetcode/removeKDigits.jpg" width="500"/>

横坐标表示num中每个数字从左到右的位置编号，纵坐标表示num中每个数字的值。

从num中删除k个数字后，剩下的数字个数是固定的，如何使<font color=red>剩下的数字组成的正数最小呢` </font>？

剩下的数字组成的正数也可以用类似上图的线段来表示，节点个数是确定的，线段上升越平缓，对应的正数也就越小。

我们需要尽量<font color=red>削去值比较大的节点 </font>。

- 1.当nums[i] > nums[i+1]时，应该削去 nums[i]这个节点，例如删除图中的C节点
- 2.上一步执行完之后，如果i节点前的nums[i-1]节点也满足 nums[i-1] > nums[i+1]节点，
  也应该削去i-1节点，重复操作删除i+1节点前且值比其大的节点，直至不再满足条件，例如图中的B节点也要被删除，但A节点不需要删除。

上述第二步的操作就像维护一个单调栈，当前遍历到的元素要入栈时，栈顶中比当前元素值大的元素全部要出栈，
以保证当前元素入栈后，从栈底向栈顶看，是一个<font color=red>单调不减栈</font>。

这道题其实用单调队列更方便，因为最后还要从栈底遍历到栈顶，输出移除K数字后的最小正数

## 代码

```java
    public String removeKdigits(String num, int k) {
        Deque<Character> deque = new LinkedList<>();
        for (int i = 0; i < num.length(); i++) {
            char c = num.charAt(i);
            while (k > 0 && !deque.isEmpty() && deque.peekLast() > c){
                deque.pollLast();
                k--;
            }
            deque.offerLast(c);
        }
        while (k-- > 0){
            deque.pollLast();
        }
        boolean leadingZero = true;
        StringBuilder sb = new StringBuilder();
        while (!deque.isEmpty()){
            char c = deque.pollFirst();
            if (c == '0' && leadingZero){
                continue;
            }
            leadingZero = false;
            sb.append(c);
        }
        return sb.length() == 0 ? "0" : sb.toString();
    }
```


## 参考

- [移掉K位数字](https://leetcode.cn/problems/remove-k-digits/)
