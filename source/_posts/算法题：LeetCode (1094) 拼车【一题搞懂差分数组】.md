---
title: 算法题：LeetCode (1094) 拼车【一题搞懂差分数组】                                                       
date: 2022-09-27 09:09:36
copyright: true
tags: [算法题, Java]
categories:
- 技术笔记
- 算法题
---            



&emsp;&emsp;车上最初有 `capacity` 个空座位，车只能向一个方向行驶，给定整数 `capacity` 和一个数组 `trips` ,  `trip[i] = [numPassengersi, fromi, toi]` 表示第 `i` 次旅行有 `numPassengersi` 乘客，接他们和放他们的位置分别是 `fromi` 和 `toi` 。这些位置是从汽车的初始位置向东的公里数。
<!--more-->

当且仅当你可以在所有给定的行程中接送所有乘客时，返回 `true`，否则请返回 `false`。                                                                                                     

> 来源：力扣（LeetCode） 
> 链接：[https://leetcode.cn/problems/car-pooling](https://leetcode.cn/problems/car-pooling)


### 示例

* 示例 1：

```shell
输入：trips = [[2,1,5],[3,3,7]], capacity = 4
输出：false
```

* 示例 2：

```shell
输入：trips = [[2,1,5],[3,3,7]], capacity = 5
输出：true
```


<br /> <br /> 

## 解题思路

**差分数组**：差分数组主要的适用场景是对原始数组进行频繁的区间增减操作，这个时候适用差分数组能够快速的完成，同时能够快速获得更新后的数组各个位置的值。

以`trips = [[2,1,5],[3,3,7]], capacity = 4`为例，数组变化如下：
![乘客情况](/images/算法题_拼车/乘客情况.png)


使用差分数组修改上图所示的数组，结果如下：
![修改为差分数组](/images/算法题_拼车/修改为差分数组.png)


<br /> <br /> 

## 代码（Java）

```java
public class LeetCode_1094_carPooling {
    public boolean carPooling(int[][] trips, int capacity) {
        int[] diff = new int[1001];
        for (int[] t : trips) {
            int passengers = t[0], from = t[1], to = t[2];
            diff[from] += passengers;
            diff[to] -= passengers;
        }
        int num = 0;    // 差分求和
        for (int i : diff) {
            num += i;
            if (num > capacity) {
                return false;
            }
        }
        return true;
    }
}

```

<br/><br/><br/><br/>