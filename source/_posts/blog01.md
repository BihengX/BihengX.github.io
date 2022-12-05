---
title: MOOSE多物理场耦合平台入门学习记录（二）- 牛顿法求解非线性方程组
date: 2022-07-05 20:11:18
tags: [偏微分方程数值解, 有限差分法]
index_img: /img/hengshan.jpeg
banner_img: /img/hengshan.jpeg
comment: 'valine'
---

放在首页的话：本人撰写博客最主要的目的是整理自己对研究对象的认识，积累经验，加深理解。自身水平并不高，很多都是靠自学。如果有大佬看出其中的错误或者有新的角度理解，还请留言指点，非常感谢，这对于我水平提高有很大的帮助。

# 牛顿法的简介
NEWTON Solver是MOOSE平台三大求解器之一（NEWTON、JFNK、PJFNK），简要了解牛顿法的基本原理对于理解MOOSE平台的底层机制有一定帮助，也能帮助判断使用NEWTON solver收敛失败的原因。本博文主要参考这篇文章(https://zhuanlan.zhihu.com/p/139595720)，以及一个非常简洁明了的教学视频(https://www.bilibili.com/video/BV19Q4y1S7dR?spm_id_from=333.337.search-card.all.click&vd_source=b154dc71d458a24aaa3478de44369f37)。

## 牛顿法求解非线性方程

大多数数值分析的教科书都是首先介绍牛顿法求解非线性方程。牛顿法的基本思想来自于泰勒公式，假设现存一个非线性方程$y = f(x) = b$，首先将方程转化为求根问题，即$F(x) = f(x)-b=0$。在某一点$x_0$处，进行泰勒展开，可以得到：
$$
F(x) = F(x_0) + F^{'}(x - x_0) + \frac{F^{''}(x_0)}{2}(x - x_0)^2 + O(\Delta x) \tag{1}
$$
对于线性方程，仅取一阶近似就可以满足要求，可以得到:
$$
\begin{array}{c}
    F(x_0) + F^{'}(x - x_0) = 0 \\
    x = x_0 - \frac{F(x_0)}{F^{'}(x_0)}
\end{array} \tag{2}
$$
显然此时的$x$并不可能直接就是方程的根，但是可以重复迭代，使得$x$不断逼近方程根，直到满足要求。从图像上看，其过程就是，在一个方程所表达的曲线上选取一个点$x_0$，然后在该点做与曲线的切线，切线与x轴相交的点，其坐标值就是下一步迭代的$x_1$，然后再$x_1$所对应曲线上的点，继续做切线，切线与x轴相交的点$x_3$，反复重复上述过程，直到$x_n$达到预设误差的要求。

但是，对于需要求解函数的极值问题，取一阶近似就不能满足要求了，需要将其转化为求解其导数为0问题，再使用牛顿法，因此，常见求解非线性方程选取的是二阶近似：
$$
\begin{array}{c}
    F(x) = F(x_0) + F^{'}(x - x_0) + \frac{F^{''}(x_0)}{2}(x - x_0)^2 = 0
\end{array} \tag{3}
$$
对上式，左右再进行求导可以得到:
$$
\begin{array}{c}
    F^{'}(x_0) + F^{''}(x - x_0) = 0 \\
    x = x_0 - \frac{F^{'}(x_0)}{F^{''}(x_0)} 
\end{array} \tag{4}
$$
可以看到，牛顿法使用的前提，是曲线导数存在，而在实际使用中，求导数是一个非常难处理的问题，可以使用差分来近似。

## 牛顿法求解非线性方程的编程实现

以方程$x^2 - 2x  -3 =0, x>0$为例，其一阶导数和二阶导数分别是$2x-2, 2$，代码如下：
``` python
    import numpy as np
    import matplotlib.pyplot as plt
    from math import *
    def F(x):
        return x*x - 2*x - 3
    def firstOD(x): #一阶导数
        return 2*x - 2

    def secondOD(x): #二阶导数
        return 2
    # 以下为一阶近似牛顿法
    x0 = 0.0 #初始值
    err_limit = 1e-10 #误差限
    err_iter = 1.0
    iter_num = 0
    while(err_iter > err_limit):
        xn = x0 - F(x0)/firstOD(x0)
        err_iter = abs(xn - x0)
        x0 = xn
        iter_num = iter_num + 1
    print('The iter num for Newton method 1st is ' + str(iter_num))
```

示例方程确实太简单了，也可以看到，单纯使用牛顿法编程并不复杂（前提是导数易求解和易计算），还有就是初值的影响非常大，第一个是初值选取不当，容易陷入初值陷阱，计算失败。第二个就是可能迭代不到想要的解，如上面的方程，应该有两个根，如果初值选取小于1.0，则是求解的-1的根，但是实际想要的是$x>0$的解。因为牛顿法单次只能搜寻某个局部区间的解。因此在使用牛顿法前，对方程的几何特性最好要有一定把握。

## 牛顿法求解非线性方程组
如果是方程组，那么解就不是一个单个的数，而是一个解向量$[x_0,x_1,x_2,\cdots,x_n]$，对应方程组为:
$$
\begin{bmatrix}
    f_{1}(x_0,x_1,x_2,\cdots,x_n)  \\
    f_{2}(x_0,x_1,x_2,\cdots,x_n) \\
    f_{3}(x_0,x_1,x_2,\cdots,x_n) \\
      \vdots \\
    f_{n}(x_0,x_1,x_2,\cdots,x_n)
\end{bmatrix} = 
\begin{bmatrix}
    0 \\
    0 \\
    0 \\
    \vdots \\
    0
\end{bmatrix} \tag{5}
$$
对左侧矩阵中的每一项使用多元泰勒展开，与前述过程类似，取一阶梯度，写成迭代格式，可以得到：
$$
\begin{array}{c}
    F(X^{k}) + \nabla F(X^{k})(X^{k+1} - X^{k}) = 0 \\
    (X^{k+1} - X^{k}) =  - \nabla F(X^{k})^{-1}F(X^{k})
\end{array} \tag{6}
$$
上式中，$\nabla F(X^{k})$ 就是常见的Jacbobian矩阵，注意，上式中，已经不是简单的乘除法，而是涉及到矩阵求逆运算，并不满足交换律。X也是一个解向量，而不是一个单个数值。Jacbobian矩阵的形式如下：
$$
\begin{bmatrix}
    \frac{\partial f_1}{\partial x_0} & \frac{\partial f_1}{\partial x_1} & \cdots &  \frac{\partial f_1}{\partial x_n} \\
    \frac{\partial f_2}{\partial x_0} & \frac{\partial f_2}{\partial x_1} & \cdots  & \frac{\partial f_2}{\partial x_n} \\
    \vdots&\vdots& \ddots & \vdots \\
    \frac{\partial f_n}{\partial x_0} & \frac{\partial f_n}{\partial x_1} & \cdots  &\frac{\partial f_n}{\partial x_n}
\end{bmatrix}
$$

## 牛顿法求解非线性方程组的编程实现
以如下方程为示例，其几何意义就是一直线与圆相交的两点。这里仅考虑y>0的解：
$$
\begin{cases}
    x^2 + y^2 = 16 \\
    y -\sqrt{3}x = 0
\end{cases} \tag{7}
$$
待求解方程和Jacobian矩阵如下：
$$
\begin{cases}
    x^2 + y^2 -16 = 0 \\
    -\sqrt{3}x + y -4 =0
\end{cases},
\begin{bmatrix}
    2x & 2y \\
    -\sqrt{3} & 1
\end{bmatrix}
$$
代入6式中的迭代表达式：
``` python

#以下为牛顿法求解方程组程序
    import numpy as np
    import matplotlib.pyplot as plt
    from math import *
    def f11(x): #四个偏导数
        return 2*x[0]

    def f12(x):
        return 2*x[1]

    def f21(x):
        return -1*sqrt(3)

    def f22(x):
        return 1

    def err_cal(a,b): #计算误差，不同的误差统计方法函数不同，这里采用平方和
        n = np.size(a,axis=0)
        err_sum = 0
        for i in range(n):
            err_sum = err_sum + (a[i] - b[i])**2
        return sqrt(err_sum)

    def Jacobian_cal(x):
        n = np.size(x,axis=0)
        tempm = np.zeros([n,n])
        tempm[0][0] = f11(x)
        tempm[0][1] = f12(x)
        tempm[1][0] = f21(x)
        tempm[1][1] = f22(x)
        return tempm
    err_limit = 1e-5
    err_iter = 1
    xi = np.zeros([2,1])
    xi[0] = 3.0
    xi[1] = 2.0
    JM = np.zeros([4,4])
    iter_num = 0
    while(err_iter > err_limit):
        JM = Jacobian_cal(xi)
        xi_old = xi.copy()
        F = np.zeros([2,1])
        F[0] = xi[0]**2 + xi[1]**2 - 16
        F[1] = -sqrt(3)*xi[0] + xi[1]#计算F(x^k)
        print(F)
        tempjms = np.linalg.inv(JM) #对Jacobian矩阵求逆
        Rn = -1*tempjms.dot(F)
        xi = xi + Rn #计算xi+1，作为下一步迭代初始值
        err_iter = err_cal(xi_old,xi) #计算迭代误差
        iter_num = iter_num + 1
    print(xi)  #输出结果
    print(iter_num)
```
在牛顿法中，Jacobian矩阵的性质决定了迭代的结果好坏，有可能会出现如下情况：

1. Jacobian矩阵迭代过程中产生奇异，不可以再求逆，迭代就会失败。
2. 由于求解导数，以及矩阵求逆本身就是很复杂的运算，需要耗费大量计算资源，很容易出现错误，这与方程本身的特性相关。例如，上面方程如果x和y的系数互为相反数，则难以迭代出正确解。
3. 初值选取不当，造成了后面一系列迭代计算的恶化。

如果准确求解Jacobian矩阵并且保持迭代过程中矩阵有良好的计算特性是应用牛顿法的大问题。同时也可以看到，牛顿法求解方程组是所有方程同步求解的，并不是类似Picard迭代一样，完成一个场的计算再代入另一个场计算，有利于考虑不同物理场方程之间的高度非线性耦合关系，但是计算要求也更高。