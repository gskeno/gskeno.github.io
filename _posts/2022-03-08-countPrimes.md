---
layout: post
title: 计数质数
tags:
- math
categories: leetcode
description: 计数质数
---
给定整数 n ，返回 所有小于非负整数 n 的质数的数量 。

示例 1 ：
```
输入：n = 10
输出：4
解释：小于 10 的质数一共有 4 个, 它们是 2, 3, 5, 7 。
```

示例 2 ：
```
输入：n = 0
输出：0
```

示例 3 ：
```
输入：n = 1
输出：0
```

提示：
- 0 <= n <= 5 * 106

----

## 分析

### 存粹暴力
对于n以内的每个正数i，都去判断i是否是质数。

时间复杂度为O(\\(n^2\\))

### 优化一
实现函数`isPrime(int x)` 判断x是否是质数时，不需要如下方式判断
```java
    public boolean isPrime(int x){
        for (int i = 2; i < x; i++) {
            if (x % i == 0){
                return false;
            }
        }
        return true;
    }
```
这会判断2到(x-1)内的每一个数是否能被x整除，事实上，只需要判断2至\\(x^{0.5}\\)内的每个数能否
被x整除即可。

反向思考，如果x是合数(除了能被1和自身整除，还能被其他数整除)，假设 x=M*N，那么M和N不可能同时>=
\\(x^{0.5}\\)，否则 M * N > x。

所以判断x是否是质数，i只需要从2遍历到\\(x^{0.5}\\)。
```java
    public boolean isPrime(int x){
        for (int i = 2; i * i <= x ; i++) {
            if (x % i == 0){
                return false;
            }
        }
        return true;
    }
```

基于此，整个解决方案的`时间复杂度`为\\(n \sqrt{n}\\)

### 优化二
基于数字之间的关联关系，`数值i的2倍，3倍，...都不是质数`。

我们令`int[] an = new int[n]`，
`an[i]=0`表示整数i是质数，`an[i]=1`表示整数i是合数，初始时，i都是质数。

算法进行过程中，`识别出合数并进行标记`。
```java
    public int countPrimes(int n) {
        int ans = 0;
        // 初始时， 认为都是质数，遍历过程中将N * i标记为合数
        int[] nums = new int[n];
        for (int i = 2; i < n; i++) {
            if (nums[i] == 0){
                ans++;
                // i的k倍都是合数
                for (int k = 2; k * i < n; k++) {
                    nums[k * i] = 1;
                }
            }
        }
        return ans;
    }
```
### 优化三
另外对于每个遍历到的`i`，不必从其2倍开始进行标记，可直接从`i * i`倍开始标记。因为`i * (i-1)`在遍历i-1时，已经被提前标记了。<font color=red>比如6是当i=2时就被标记为合数的(2的3倍)，所以当i=3时，不必再次标记6为合数(3的2倍)</font>
```java
if ((long) i * i < n) {
    for (int j = i * i; j < n; j += i) {
            isPrime[j] = 0;
    }
}
```

### 优化四
尽管优化三已经排除了一些重复的合数标记，比如6。但是有些数字还是会倍重复标记，比如 45还是会在遍历到
3和5时被重复标记。

所以，这次要彻底避免重复标记。
```java
class Solution {
    public int countPrimes(int n) {
        List<Integer> primes = new ArrayList<Integer>();
        int[] isPrime = new int[n];
        Arrays.fill(isPrime, 1);
        for (int i = 2; i < n; ++i) {
            if (isPrime[i] == 1) {
                primes.add(i);
            }
            for (int j = 0; j < primes.size() && i * primes.get(j) < n; ++j) {
                isPrime[i * primes.get(j)] = 0;
                if (i % primes.get(j) == 0) {
                    break;
                }
            }
        }
        return primes.size();
    }
}
```
这里可以保证<font color=red>每个合数只会被其最小的质因数标记一次</font>。比如遍历到i=10时，
20= i * 2会被标记为合数，但是30=i * 3不会立即被标记为合数；当遍历到i = 10 / 2 * 3 = 15时，
30= i * 2才会被标记为合数。这里我们注意到`20`，`30`都是因为其最小的质因数2才被标记为合数的。

时间复杂度为O(n)，因为每个数都只会被标记为合数一次。
 


## 参考

- [质数计数](https://leetcode.cn/problems/count-primes/)
- [Mathjax](https://simpleyyt.com/jekyll-jacman/opinion/2014/02/16/Mathjax-with-jekyll)
