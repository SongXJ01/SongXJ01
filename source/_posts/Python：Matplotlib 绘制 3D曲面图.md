---
title: Python：Matplotlib 绘制 3D曲面图
date: 2020-04-13 13:59:02
copyright: true
tags: [Python, Matplotlib]
categories:
- 技术笔记
- Python
---

-----
Matplotlib 是 Python 的绘图库，它与 NumPy 一起使用，可以基本上实现 MATLAB 的绘图和计算功能，而且效率更高，速度更快。

今天主要说一下关于 Matplotlib 绘制三维图像，并实现一个可以多次使用的**函数模板**，直接复制调用即可使用。

----
## 1. 导入模块包
`numpy`和`matplotlib`是两个常规的基本模块。因为实现的是三维绘图，所以需要另外一个模块`Axes3D`，这是是 Matplotlib 里面专门用来画三维图的工具包。
```python
import numpy as np
from matplotlib import pyplot as plt
from mpl_toolkits.mplot3d import Axes3D
```

## 2. 图像的基本设置
这里包括对图中字体大小、图片长宽比、分辨率的调整，并将其转换为三维格式。
```python
	plt.rcParams.update({'font.size': 32})  # 统一设置图中字体大小
	fig = plt.figure(figsize=(20, 16), dpi=50)  # 设置图像大小和分辨率
	ax3 = Axes3D(fig)  # 将图像转换为3D模式
```

## 3. 处理数据，生成坐标矩阵
这里的`matrix`是一个二维列表，是 Python 的基本数据格式，需要将其先转化为`np.array`的格式，才能进行更多的操作。
另外要根据传入的二维数据**创建坐标矩阵**，这一点很重要。
```python
    # 绘制三维图像
    matrix = np.array(matrix)
    # 根据二维数据的长宽创建坐标矩阵
    arrX = np.arange(0, len(matrix[0]))
    arrY = np.arange(0, len(matrix))
    X, Y = np.meshgrid(arrX, arrY)  # 创建坐标矩阵
    print(X.shape, Y.shape, matrix.shape)
    ax3.plot_surface(X, Y, matrix, cmap='rainbow')
```

## 4. 设置坐标轴刻度，稀疏化坐标轴刻度
如果我们的数据是100 X 100 的二维矩阵，如果将所有的刻度都显示在坐标轴上，那么必会变得密密麻麻，所以我们需要将坐标刻度稀疏化，并用自己想要的方式展现出来。
使用`xticks(x, _x)`设置X坐标轴刻度（Y轴同理，但是不能设置Z轴）：
* 第一个参数：刻度值列表
* 第二个参数：需要展示的出来的经过处理的刻度值列表。这里可以对刻度自定义，比如统一扩大10倍或缩小十倍（本例中以0.1为步长取的数据，所以在刻度上要乘上步长，即缩小为0.1倍），同时也可以设置为字符串。
```python
    x = list(range(len(arrX)))[::int(len(arrX) / 10)]
    _x = [int(i * stepValue) for i in x]
    plt.xticks(x, _x)
```
## 5. 设置图片名称并保存
使用`xlabel`设置X轴名称（Y轴同理，但是不能设置Z轴）：
 * 第一个参数：X轴名称的字符串
 * 第二个参数（`labelpad`）：X轴名称与X轴之间的间隔距离

使用`savefig`储存图片，这里直接将需要储存的图片格式写在图片名称的字符串中即可。如果在 LaTeX中 使用推荐`.eps`格式，另外也可以储存为`.jpg`格式和`.png`格式.

```python
    plt.xlabel(keyX, labelpad=30)  # X轴名称
    plt.ylabel(keyY, labelpad=30)  # Y轴名称
    plt.title("The effect of {} and {} on GJBD".format(keyX, keyY), pad=20)  # 图形题目
    plt.savefig("./change_{}{}.eps".format(keyX, keyY))  # 保存图片
    plt.show()
```

-----
## 完整代码接口如下
三个参数如下：
* matrix : 二维列表格式的数据，形如[ [1,2,3], [4,5,6], [7,8,9] ]
* keyX : X轴名称
* keyY : Y轴名称

其中`stepValue`是二维数据的采集步长，可以根据实际情况自行修改。
```python
import numpy as np
from matplotlib import pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

''' 使用二维列表数据绘制三维曲面图
--- matrix : 二维数据（普通Python二维列表）
--- keyX : X轴名称（string）
--- keyY : Y轴名称（string）
'''
def figure_3D(matrix, keyX, keyY):
    plt.rcParams.update({'font.size': 32})  # 统一设置图中字体大小
    fig = plt.figure(figsize=(20, 16), dpi=50)  # 设置图像大小和分辨率
    ax3 = Axes3D(fig)  # 将图像转换为3D模式

    # 绘制三维图像
    matrix = np.array(matrix)
    # 根据二维数据的长宽创建坐标矩阵
    arrX = np.arange(0, len(matrix[0]))
    arrY = np.arange(0, len(matrix))
    X, Y = np.meshgrid(arrX, arrY)  # 创建坐标矩阵
    print(X.shape, Y.shape, matrix.shape)
    ax3.plot_surface(X, Y, matrix, cmap='rainbow')

    # 设置坐标轴刻度
	stepValue = 0.1    # 步长
    x = list(range(len(arrX)))[::int(len(arrX) / 10)]
    _x = [int(i * stepValue) for i in x]
    plt.xticks(x, _x)
    y = list(range(len(arrY)))[::int(len(arrY) / 5)]
    _y = [round(i * stepValue, 1) for i in y]
    plt.yticks(y, _y)

    # 设置坐标轴名称
    plt.xlabel(keyX, labelpad=30)  # X轴名称
    plt.ylabel(keyY, labelpad=30)  # Y轴名称
    plt.title("The effect of {} and {} ".format(keyX, keyY), pad=20)  # 图形题目
    plt.savefig("./change_{}{}.eps".format(keyX, keyY))  # 保存图片
    plt.show()

```
