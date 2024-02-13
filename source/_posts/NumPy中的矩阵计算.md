---
title: NumPy中的矩阵计算
date: 2023-09-21 22:27:29
copyright: true
tags: [矩阵计算, Python, NumPy]
categories:
- 技术笔记
- 矩阵计算

---


&emsp;&emsp;在本篇文章中将深入探讨NumPy库中矩阵点乘与叉乘的具体运算法则。
<!--more-->




<br/><br/>

# 点乘（Element-wise multiplication）

&emsp;&emsp;对于两个维度完全一样的矩阵用 multiply 做乘法，那么它们就是进行对应位置元素之间的乘法（Element-Wise Product），得到一个同样维度的矩阵输出，用符号 `.*` 表示（等价于`numpy`中的`*`符号）。

$\left[\begin{array}{lll}
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
\end{array}\right]$

## 标量 .* 向量 or 矩阵（Scalar .* Vector or Matrix）
&emsp;&emsp;设有行向量 $a=\left[1,2,3\right]$，将其转置得到 $a^T=\left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right]$, 和矩阵 $M=\left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
7 & 8 & 9
\end{array}\right]$。标量点乘向量或矩阵，则代表标量乘向量或矩阵中的每一个元素，在这种情况下左乘和右乘是等价的。

【例子】

$2 \; .* \; a = 2 \; .* \; \left[1,2,3\right] = \left[2,4,6\right]$

$2 \; .* \; a^T = 2 \; .* \; \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right]=\left[\begin{array}{l}
2 \\
4 \\
6 
\end{array}\right]$

$2 \; .* \; M = 2 \; .* \; \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
7 & 8 & 9
\end{array}\right] = \left[\begin{array}{lll}
2 & 4 & 6 \\
8 & 10 & 12 \\
14 & 16 & 18
\end{array}\right]$

## 矩阵 .* 矩阵 （Matrix .* Matrix）

&emsp;&emsp;设有矩阵 $M=\left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
7 & 8 & 9
\end{array}\right]$，矩阵之间的点乘仅支持维度完全相同的两个矩阵相乘，其本质为两个矩阵对应元素相乘，在这种情况下左乘和右乘是等价的。

$M \; .* \; M = \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
7 & 8 & 9
\end{array}\right] \; .* \; \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
7 & 8 & 9
\end{array}\right] = \left[\begin{array}{lll}
1 & 4 & 9 \\
16 & 25 & 36 \\
49 & 64 & 81
\end{array}\right]$

## 向量 .* 向量 （Vector .* Vector）

&emsp;&emsp;设有行向量 $a=\left[1,2,3\right]$，$b=\left[4,5,6\right]$，$c=\left[1,2\right]$。若两个维度相同的向量点乘，则代表对应位置的元素对应相乘，若两个向量方向不同（一个为行向量，一个为列向量）则对各自长度不做要求，分别补齐成相同形状后对应元素相乘。在这种情况下左乘和右乘是等价的。

$a \; .* \; b = \left[1,2,3\right] \; .* \; \left[4,5,6\right] = \left[4,10,18\right]$

$c \; .* \; b^T = \left[1,2\right] \; .* \; \left[\begin{array}{l}
4 \\
5 \\
6 
\end{array}\right]= \left[\begin{array}{lll}
1 & 2  \\
1 & 2  \\
1 & 2 
\end{array}\right] \; .* \; \left[\begin{array}{lll}
4 & 4  \\
5 & 5  \\
6 & 6 
\end{array}\right]= \left[\begin{array}{lll}
4 & 8  \\
5 & 10  \\
6 & 12 
\end{array}\right]$

$c^T \; .* \; b = \left[\begin{array}{l}
1 \\
2 \\
\end{array}\right] \; .* \; \left[4,5,6\right] = \left[\begin{array}{lll}
1 & 1 & 1 \\
2 & 2 & 2 \\
\end{array}\right] \; .* \; \left[\begin{array}{lll}
4 & 5 & 6 \\
4 & 5 & 6 
\end{array}\right]= \left[\begin{array}{lll}
4 & 5 & 6 \\
8 & 10 & 12 
\end{array}\right]$

## 向量 .* 矩阵 （Vector .* Matrix or Matrix .* Vector）
&emsp;&emsp;（这里的沿用数学书写习惯使用“`.*`”代表 NumPy 中的“`*`”）
&emsp;&emsp;设有行向量 $a=\left[1,2,3\right]$, 和矩阵 $M=\left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right]$。如果是行向量和矩阵点乘，则其长度必须和矩阵的列数相同，纵向补齐成和矩阵维度相同后对应元素相乘；如果是列向量和矩阵点乘，则其长度必须和矩阵的行数相同，横向补齐成和矩阵维度相同后对应元素相乘。在这种情况下左乘和右乘是等价的。

$a \; .* \; M = \left[1,2,3\right] \; .* \; \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right]= \left[\begin{array}{lll}
1 & 2 & 3 \\
1 & 2 & 3 \\
\end{array}\right] \; .* \; \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right]= \left[\begin{array}{lll}
1 & 4 & 9 \\
4 & 10 & 18 \\
\end{array}\right]$

$a^T \; .* \; M^T = \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right] \; .* \; \left[\begin{array}{lll}
1 & 4  \\
2 & 5  \\
3 & 6
\end{array}\right] = \left[\begin{array}{lll}
1 & 1  \\
2 & 2  \\
3 & 3 
\end{array}\right] \; .* \; \left[\begin{array}{lll}
1 & 4  \\
2 & 5  \\
3 & 6 
\end{array}\right]= \left[\begin{array}{lll}
1 & 4  \\
4 & 10  \\
9 & 18 
\end{array}\right]$

$a \; .* \; M^T = \left[1,2,3\right] \; .* \; \left[\begin{array}{lll}
1 & 4  \\
2 & 5  \\
3 & 6
\end{array}\right] = Invalid$

$a^T \; .* \; M = \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right] \; .* \;\left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right] = Invalid$

$M \; .* \; a = \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right] \; .* \;\left[1,2,3\right] =  \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right]  \; .* \; \left[\begin{array}{lll}
1 & 2 & 3 \\
1 & 2 & 3 \\
\end{array}\right] = \left[\begin{array}{lll}
1 & 4 & 9 \\
4 & 10 & 18 \\
\end{array}\right]$

$M^T \; .* \; a^T =  \left[\begin{array}{lll}
1 & 4  \\
2 & 5  \\
3 & 6
\end{array}\right]  \; .* \; \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right] = \left[\begin{array}{lll}
1 & 4  \\
2 & 5  \\
3 & 6 
\end{array}\right] \; .* \; \left[\begin{array}{lll}
1 & 1  \\
2 & 2  \\
3 & 3 
\end{array}\right] = \left[\begin{array}{lll}
1 & 4  \\
4 & 10  \\
9 & 18 
\end{array}\right]$

$M^T  \; .* \; a = \left[\begin{array}{lll}
1 & 4  \\
2 & 5  \\
3 & 6
\end{array}\right] \; .* \; \left[1,2,3\right]  = Invalid$

$M \; .* \; a^T = \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right] \; .* \;  \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right]= Invalid$


<br/><br/>

---

<br/><br/>

# 叉乘（dot）

&emsp;&emsp;矩阵叉乘，$\mathrm{m} \times \mathrm{n}$矩阵乘以 $\mathrm{n} \times \mathrm{k}$矩阵会得到一个$\mathrm{m} \times \mathrm{k}$的矩阵，用符号 `*` 表示（等价于`numpy`中的`@`符号）。

$A^{\mathrm{m} \times \mathrm{n}}  \quad * \quad B^{\mathrm{n} \times \mathrm{k}} \quad=\quad C^{\mathrm{m} \times \mathrm{k}}$

## 标量 * 向量 or 矩阵（Scalar * Vector or Matrix）

&emsp;&emsp;标量作为一个维度为$\mathrm{1} \times \mathrm{1}$的矩阵，按照矩阵乘法仅支持右乘行向量，或左乘列向量。设有行向量 $a=\left[1,2,3\right]$，将其转置得到列向量 $a^T=\left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right]$, 

$2^{(1\times1)} \; * \; a^{(1\times3)} = 2 \; * \; \left[1,2,3\right] = \left[2,4,6\right]$

$a^{T(3\times1)}\; * \; 2^{(1\times1)} = \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right] \; * \; 2 =\left[\begin{array}{l}
2 \\
4 \\
6 
\end{array}\right]$

## 矩阵 * 矩阵 （Matrix * Matrix）

&emsp;&emsp;设有矩阵 $M_1=\left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right]$，$M_2=\left[\begin{array}{ll}
1 & 2 \\
3 & 4 \\
5 & 6 \\
\end{array}\right]$。这种情况下按照矩阵运算法则进行计算。

$M_1^{(2\times3)} \; * \; M_2^{(3\times2)} = \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right] \; * \; \left[\begin{array}{ll}
1 & 2 \\
3 & 4 \\
5 & 6 \\
\end{array}\right] = \left[\begin{array}{ll}
22 & 28 \\
49 & 64
\end{array}\right]$

$M_1^{T(3\times2)} \; * \; M_2^{T(2\times3)} = \left[\begin{array}{ll}
1 & 4 \\
2 & 5 \\
3 & 6 \\
\end{array}\right] \; * \;   \left[\begin{array}{lll}
1 & 3 & 5 \\
2 & 4 & 6 \\
\end{array}\right] = \left[\begin{array}{lll}
9 & 19 & 29 \\
12 & 26 & 40 \\
15 & 33 & 51
\end{array}\right]$

## 向量 * 向量 （Vector * Vector）
&emsp;&emsp;设有行向量 $a=\left[1,2,3\right]$，$b=\left[4,5,6\right]$。这种情况下按照矩阵运算法则进行计算。

$a^{(1\times3)} \; * \; b^{T(3\times1)} = \left[1,2,3\right] \; * \; \left[\begin{array}{l}
4 \\
5 \\
6 
\end{array}\right]= 32$

$a^{T(3\times1)} \; * \; b^{(1\times3)} = \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right] \; * \; \left[4,5,6\right] = \left[\begin{array}{lll}
4 & 5 & 6 \\
8 & 10 & 12 \\
12 & 15 & 18
\end{array}\right]$

## 向量 * 矩阵 （Vector * Matrix or Matrix * Vector）
&emsp;&emsp;设有行向量 $a=\left[1,2,3\right]$, 和矩阵 $M=\left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right]$。这种情况下按照矩阵运算法则进行计算。

$a^{(1\times3)} \; * \; M^{T(3\times2)} = \left[1,2,3\right] \; * \; \left[\begin{array}{ll}
1 & 2 \\
3 & 4 \\
5 & 6 \\
\end{array}\right] = \left[14,32\right]$

$M^{(2\times3)} \; * \; a^{T(3\times1)} = \left[\begin{array}{lll}
1 & 2 & 3 \\
4 & 5 & 6 \\
\end{array}\right]  \; * \; \left[\begin{array}{l}
1 \\
2 \\
3 
\end{array}\right] = \left[\begin{array}{l}
14 \\
32
\end{array}\right]$


<br/><br/><br/><br/>
