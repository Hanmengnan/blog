---
title:  故障树分析（FTA）
tags:  [软件工程]
categories: [笔记]
date: 2020/5/22 20:46:25
---


> 故障树分析，就是选择某一故障作为顶层事件，逐步拆解事件为中间事件，直到事件无法拆解变为底层事件。拆解事件的过程可以画成一棵树，这棵树也就称为故障树，利用故障树分析底层事件对故障发生影响的重要程度称为故障树分析。

### 建树符号

#### 事件符号

![image.png](https://b3logfile.com/file/2020/05/image-93cd7958.png)

#### 逻辑符号

![image.png](https://b3logfile.com/file/2020/05/image-06e7d4cf.png)

#### 结构函数

- 底事件状态

    ```math
    x_i=
    \begin{cases}
    0,故障\\
    1,正常\\
    \end{cases}
    ```

- 顶事件状态
    > 可用与底事件有关的函数表示
    
  ```math
  T=\phi(x_1,x_2...x_n)

  \phi(T)=
  \begin{cases}
  0,故障\\
  1,正常\\
  \end{cases}
  ```

- 计算

    - 或门

    ```math
    \phi(T)=\sum_{i=1}^{n}{x_i}
    ```

    - 与门

    ```math
    \phi(T)=\prod_{i=1}^{n}{x_i}
    ```
    
- 化简
    ```math
    T=x_1(x_1+x_4)x_3=x_1x_3+x_1x_4x_3=x_1x_3
    ```
    
- 最小割集
    - 最小割集是指，是指让顶层事件发生的最少数量的底层事件集合
    - 例：
        - 顶层事件T可以表示为 
        ![image.png](https://b3logfile.com/file/2020/05/image-f2f1d995.png)
        - 化简为        
        ![image.png](https://b3logfile.com/file/2020/05/image-e8b97e7f.png)
        - 根据表达式为或运算，则可知有4个最小割集，为：
        ![image.png](https://b3logfile.com/file/2020/05/image-ce114013.png)



    
- 路集
    - 是割集的对偶集合，也即不触发发生顶层事件的底层事件集合


#### 顶层事件概率的计算
1. 将顶层事件表示为，最小割集的形式
    ```math
    T=\phi(x_1,x_2...x_n)
    ```
2. P为计算事件的概率，则
    ```math
    P(T)=\phi(P(x_1),P(x_2)...P(x_n))
    ```


#### 结构重要度计算
0. 设顶层事件T，底层事件共n个
1. 计算底层事件x发生的情况下，顶层事件T发生的情况共p个
2. 计算底层事件x不发生的情况下，顶层事件T发生的情况共q个
3. 则事件x的结构重要度为
    ```math
    I(x)=\frac{1}{2^{n-1}} [p-q]
    ```

#### 概率重要度计算
1. 列出所有底事件状态组合与对应的顶事件状态
2. 找到其中其他条件相同，底事件x取1时顶事件T为1，底事件x取0顶事件T为0的情况。
3. 计算概率
4. 例：
    - ![image.png](https://b3logfile.com/file/2020/05/image-d923c89c.png)
    - 组合**1001**、**1010**、**1110**满足条件
    - ![image.png](https://b3logfile.com/file/2020/05/image-07f0f890.png)


#### 关键重要度计算
```math
I_c(i) = \frac{P_i}{g(P)} I_g(i)

P_i为底事件i发生的概率

g(P)为顶事件发生的概率

I_g(i)为事件i的概率重要度
```
