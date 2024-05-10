---
title: STL库中SET的原理
date: 2023-11-09 18:36:01
copyright: true
tags: [C++, STL库, 集合]
categories:
- 技术笔记
- C++
---

&emsp;&emsp;`std::set` 是C++标准库中的容器之一，它基于红黑树实现。`std::set` 利用红黑树的特性来实现有序的插入、查找和删除操作，并且具有较好的平均和最坏情况下的时间复杂度。
当向 `std::set` 插入元素时，它会按照特定的比较函数（`bool less<T>::operator() const(const T &lhs, const T & rhs)`）将新元素插入到红黑树的适当位置，以保持树的有序性质。插入操作的平均时间复杂度为 $O(\log n)$，其中$n$是 `std::set` 中元素的数量。查找操作（`find()`）使用红黑树的性质，通过比较函数在树中进行二分查找，查找操作的平均时间复杂度为 $O(\log n)$。

<!--more-->

&emsp;&emsp;但是当我们把 struct 放入 `std::set` 会有什么后果呢？因为 `std::set` 需要在插入到时候排序，所以需要重载  struct 的比较运算符，这个时候就出现问题了，首先我们定义一个结构体 `Person` ：

```cpp
struct Person {
    Person(int _ID, string _name, int _age) : ID(_ID), age(_age), name(_name) {}
    int ID;
    int age;
    string name;
};
```
&emsp;&emsp;当我们直接插入到 `std::set<Person>` 中时，会报 complier error 的错误，因此简单补写一个比较运算符重载，如下：

```cpp
bool operator<(const Person &lhs, const Person &rhs) {
    return lhs.age < rhs.age;
}
```
&emsp;&emsp;OK，编译起来没有问题，但是我们运行测试一下下面的 `find` 操作就会发现问题

```cpp
#include <iostream>
#include <set>

using namespace std;

struct Person {
    Person(int _ID, string _name, int _age) : ID(_ID), age(_age), name(_name) {}
    int ID;
    int age;
    string name;
};

bool operator<(const Person &lhs, const Person &rhs) {
    return lhs.age < rhs.age;
}

int main() {
    set<Person> person;
    for (int i = 0; i < 1000; ++i) {
        Person p_tmp(i, "sxj", 10);
        person.insert(p_tmp);
    }
    Person p(2000, "sxj", 10);
    auto it = person.find(p);
    if (it != person.end())
        cout << "Find Person --- ID: " << it->ID << "  name: " << it->name << "  age: " << it->age;
    else
        cout << "Can't find" << endl;
    return 0;
}
```
 
![运行结果](/SongXJ01/images/STL库中SET的原理/运行结果.png)

&emsp;&emsp;明明不在 `set` 中的 **ID-2000** 的 `Person` 也可以被找到。造成这个结果的原因是我们所提供的 `operator<() `，当`Person` `p1`、`p2`，在 `p1<p2` 与 `p2<p2` 都不成立时，`find` 就会判断 `p1` 和 `p2` 是同一个 `Person` ，因此会造成这样的错误结果。

&emsp;&emsp;**解决方案**就是补充完整我们的比较运算符重载，**完整代码如下**：

```cpp
#include <iostream>
#include <set>

using namespace std;

struct Person {
    Person(int _ID, string _name, int _age) : ID(_ID), age(_age), name(_name) {}
    int ID;
    int age;
    string name;
};

bool operator<(const Person &lhs, const Person &rhs) {
    if (lhs.ID < rhs.ID) return true;
    if (lhs.ID > rhs.ID) return false;
    if (lhs.name < rhs.name) return true;
    if (lhs.name > rhs.name) return false;
    return lhs.age < rhs.age;
}

int main() {
    set<Person> person;
    for (int i = 0; i < 1000; ++i) {
        Person p_tmp(i, "sxj", 10);
        person.insert(p_tmp);
    }
    Person p(2000, "sxj", 10);
    auto it = person.find(p);
    if (it != person.end())
        cout << "Find Person --- ID: " << it->ID << "  name: " << it->name << "  age: " << it->age;
    else
        cout << "Can't find" << endl;
    return 0;
}
```


参考：[才哥第十四讲 std:set (二）](https://mp.weixin.qq.com/s?__biz=MzkxMzQ5NTI2Mg==&mid=2247483878&idx=1&sn=d155daf0a7125c05c44c639711c254a6&chksm=c17d8457f60a0d41e248ebfd6342c89476eaa710178889458d2442ec74561de1ac13b0f0614a&scene=132&exptype=timeline_recommend_article_extendread_samebiz#wechat_redirect)



<br/><br/><br/><br/>

