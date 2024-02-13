---
title: Hexo博客技能树
date: 2023-05-23 16:36:49
copyright: true
tags: [前端, Hexo, 个人博客]
categories:
- 技术笔记
- 前端技术栈

top: 99
---


&emsp;&emsp;Hexo博客技能树汇总。在本篇文章中将会汇总本网站支持的各种技能。

<!--more-->

# 绘制动态图表

基于插件 `hexo-tag-chart` 实现单优雅的数据可视化。
安装方式：

```shell
npm i hexo-tag-chart -S
```

{% chart 90% 300 %}
    {
    type: 'line',
    data: {
    labels: ['January', 'February', 'March', 'April', 'May', 'June', 'July'],
    datasets: [{
        label: 'My First dataset',
        backgroundColor: 'rgb(255, 99, 132)',
        borderColor: 'rgb(255, 99, 132)',
        data: [0, 10, 5, 2, 20, 30, 45]
        }]
    },
    options: {
        responsive: true,
        title: {
        display: true,
        text: 'Chart.js Line Chart'
        }
    }
}
{% endchart %}

<br/>

上面这个样例可以通过以下代码来实现：

```js
{% chart 90% 300 %}
    {
    type: 'line',
    data: {
    labels: ['January', 'February', 'March', 'April', 'May', 'June', 'July'],
    datasets: [{
        label: 'My First dataset',
        backgroundColor: 'rgb(255, 99, 132)',
        borderColor: 'rgb(255, 99, 132)',
        data: [0, 10, 5, 2, 20, 30, 45]
        }]
    },
    options: {
        responsive: true,
        title: {
        display: true,
        text: 'Chart.js Line Chart'
        }
    }
}
{% endchart %}
```


<br/><br/><br/><br/>



# 支持LaTeX数学公式

安装方式：

```shell
npm un hexo-renderer-marked -S && npm i hexo-renderer-markdown-it-katex -S
```

$$
\left[\begin{array}{lll}
    a_{11} & a_{12} & a_{13} \\
    a_{21} & a_{22} & a_{23} \\
    a_{31} & a_{32} & a_{33}
    \end{array}\right] .*\left[\begin{array}{lll}
    b_{11} & b_{12} & b_{13} \\
    b_{21} & b_{22} & b_{23} \\
    b_{31} & b_{32} & b_{33}
    \end{array}\right]=\left[\begin{array}{lll}
    a_{11} b_{11} & a_{12} b_{12} & a_{13} b_{13} \\
    a_{21} b_{21} & a_{22} b_{22} & a_{23} b_{23} \\
    a_{31} b_{31} & a_{32} b_{32} & a_{33} b_{33}
\end{array}\right]
$$

<br/>

上面这个样例可以通过以下代码来实现：

```latex
$$
\left[\begin{array}{lll}
    a_{11} & a_{12} & a_{13} \\
    a_{21} & a_{22} & a_{23} \\
    a_{31} & a_{32} & a_{33}
    \end{array}\right] .*\left[\begin{array}{lll}
    b_{11} & b_{12} & b_{13} \\
    b_{21} & b_{22} & b_{23} \\
    b_{31} & b_{32} & b_{33}
    \end{array}\right]=\left[\begin{array}{lll}
    a_{11} b_{11} & a_{12} b_{12} & a_{13} b_{13} \\
    a_{21} b_{21} & a_{22} b_{22} & a_{23} b_{23} \\
    a_{31} b_{31} & a_{32} b_{32} & a_{33} b_{33}
\end{array}\right]
$$
```

# 支持 Mermaid 流程图



```mermaid
classDiagram
Class01 <|-- AveryLongClass : Cool
Class03 *-- Class04
Class05 o-- Class06
Class07 .. Class08
Class09 --> C2 : Where am i?
Class09 --* C3
Class09 --|> Class07
Class07 : equals()
Class07 : Object[] elementData
Class01 : size()
Class01 : int chimp
Class01 : int gorilla
Class08 <--> C2: Cool label
```


<br/>

上面这个样例可以通过以下代码来实现：
```
classDiagram
Class01 <|-- AveryLongClass : Cool
Class03 *-- Class04
Class05 o-- Class06
Class07 .. Class08
Class09 --> C2 : Where am i?
Class09 --* C3
Class09 --|> Class07
Class07 : equals()
Class07 : Object[] elementData
Class01 : size()
Class01 : int chimp
Class01 : int gorilla
Class08 <--> C2: Cool label
```
