---
title: STL库中可能存在的内存溢出与脏数据问题
date: 2023-10-26 21:06:01
copyright: true
tags: [C++, 内存溢出, STL库]
categories:
- 技术笔记
- C++
---




&emsp;&emsp;STL库中的 `vector` 是我们使用最频繁的STD容器之一。它具有广泛的应用，并且在性能方面表现出色。然而，其存在一种潜在问题，即**溢出**。由于 `vector` 在使用下标访问元素时不会检查索引是否越界，因此很可能导致溢出错误的出现。这种错误被严格定义为一种内存的`"dirty read / dirty write"`。

<!--more-->

**下面的代码就是模拟这样的一种情况：**
&emsp;&emsp;首先在内存中声明一个长度为 16 的 `vector` 并命名为 `tmp`，用来在内存中写入藏数据。然后紧跟着声明一个长度为 16 的全 0 `vector` 。这里构建一个名为 `dirty_write` 的函数，其中向随机产生的索引中写入值，因为这里产生的索引不一定在 16 以内，所以会随机写入脏数据。


```cpp
#include <iostream>
using namespace std;

void dirty_write(std::vector<double> &args) {
    int r = rand() % 40 - 10;
    args[r] = rand() % 101 - 50;
}

int main() {
    vector<double> tmp(16);
    vector<double> zero_vec(16, 0);
    cout << "Before: " << endl;
    for (int i = 0; i < zero_vec.size(); ++i) {
        cout << zero_vec[i] << " ";
    }
    for (int i = 0; i < 10000; i++) {
        dirty_write(tmp);
    }
    cout << "\nAfter: " << endl;
    for (int i = 0; i < zero_vec.size(); ++i) {
        cout << zero_vec[i] << " ";
    }
    return 0;
}
```

&emsp;&emsp;上面的代码中的 `zero_vec` 初始化一个长度为 16 的全 0 数组，但实际运行结果是是这样的:
![运行结果](/images/STL库中可能存在的内存溢出与脏数据问题/运行结果.png)


&emsp;&emsp;经过查询发现 STL 中有在运行中检查 `index "__n"` 是否在范围内的代码，并通过 `_LIBCPP_DEBUG` 控制，默认时 `_LIBCPP_DEBUG` 被设置为 0 检查 assert 不开启，但是可以在编译时手动开启这个技能，即在g++ 编译时加上如下参数：

```bash
g++ -D_LIBCPP_DEBUG=1 ...
```


参考：[才哥第十四讲 std:set (二）](https://mp.weixin.qq.com/s?__biz=MzkxMzQ5NTI2Mg==&mid=2247483878&idx=1&sn=d155daf0a7125c05c44c639711c254a6&chksm=c17d8457f60a0d41e248ebfd6342c89476eaa710178889458d2442ec74561de1ac13b0f0614a&scene=132&exptype=timeline_recommend_article_extendread_samebiz#wechat_redirect)



<br/><br/><br/><br/>

