---
title: 推荐系统 Factorization Machines 因子分解机
date: 2021-11-30
tags: [机器学习,推荐系统]
cover: https://s1.ax1x.com/2021/12/09/oW7RRP.jpg
mathjax: true
---

## **前言**

因子分解机 (Factorization Machines) 是CTR预估的重要模型之一。要讲述因子分解机FM，就避不开**逻辑回归LR**（logistic Rgeression）和**矩阵分解MF**（Matrix Factorization）。

转自[推荐系统玩家 之 因子分解机FM（Factorization Machines） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/153500425)



## 1.因子分解机的演变
传统的推荐模型
![img](https://pic4.zhimg.com/80/v2-bd724d20893bb295c5107411397ed197_720w.jpg)

上图是传统的推荐系统模型的分类，逻辑回归LR模型在传统的推荐模型中占据着非常重要的位置。而因子分解机的出现，也来源于**逻辑回归和矩阵分解**的演化。

首先我们知道，**逻辑回归**的是对所有特征的一个线性加权组合, 然后再加入Sigmoid逻辑函数：

$$
\large \hat{y}(x)=w_{0}+w_{1} x_{1}+\ldots+w_{n} x_{n}=w_{0}+\sum_{i=1}^{n} w_{i} x_{i} \tag{1}
$$
对比与矩阵分解，虽然逻辑回归模型已经**不单单**考虑了用户的行为特征，也可以加入年龄，性别，物品的属性，时间，地点等等特征。但是逻辑回归表达能力仍然差的原因是仅仅用了每个单一的特征，而没有考虑到**特征之间的关系**。

那有没有模型可以加入特征之间的联系呢？

**POLY2模型（Degree-2 Polynomial Margin）**则在LR的基础上采用了一种**暴力的特征组合模式**，即将所有特征两两相交，因此原来的LR模型就变成了：
$$
\large \hat{y}(x)=w_{0}+\sum_{i=1}^{n} w_{i} x_{i}+\sum_{i=1}^{n-1} \sum_{j=i\llcorner 1}^{n} w_{i j} x_{i} x_{j} \tag{2}
$$
其中$\Large w_{i j}$​​是特征组合的$ (i, j)$​​权重。

也就是说，**POLY2模型**将所有的特征两两相交，暴力的组合了特征。这样以来，就增加了特征的维度，考虑到了特征之间的关系。但是同时，暴力的组合带来的是维度的增加。在机器学习中，我们普遍使用**One-hot**编码，这样的暴力组合，就使得复杂度从$\large n$​​​​上升到了 ,$\large n^2$​​​​ 特征维度也上升，同时数据极度稀疏，在训练过程中很难收敛。

那么有没有方法可以处理稀疏数据，同时保持特征之间的联系呢？

## 2.因子分解机

用户对电影打分的onehot编码

![img](https://pic1.zhimg.com/80/v2-a3c949bc9467e00b578f81c81108b4b0_720w.jpg)

上图是电影背景下，用户对电影打分的onehot编码形式。每行代表一个样本，每列都代表一个特征。同时特征可以分为五个部分：用户ID，电影ID，用户对其他电影的打分（归一化），时间信息，以及对上一次电影的打分。

首先我们直观的看下，为什么**POLY2模型**在很多情况下是不适用的。

$$
\large \hat{y}(x)=w_{0}+\sum_{i=1}^{n} w_{i} x_{i}+\sum_{i=1}^{n-1} \sum_{j=i+1}^{n} w_{i j} x_{i} x_{j} \tag{3}
$$
**POLY2模型**的表达式如上， $\Large w_{0}$是常数项， $\Large w_{i}$是一阶特征的系数， $\Large w_{i}w_{j}$是二阶特征的系数，也就是交叉特征的系数。而由于在数据中，不是每个特征组合都有相互作用，因此， $\Large x_{i}\cdot x_{j}$为0。

举个例子来说，如上图中，我们要估计用户A对电影ST星际迷航（Star Trek）的相互作用，来预测A都ST的打分。显然，上图中A列乘以ST列是等于0的。也就是说，交叉项  $\Large x_{i}\cdot x_{j}$ 为0了，那么对应的  $\Large w_{i}w_{j}$ 在梯度更新时，$\Large\frac{\partial \hat{y}(x)}{\partial w_{i, j}}=x_{i} x_{j}=0$，因此也就会导致  $\Large w_{i}w_{j}$ 的训练不充分且不准确，进而影响模型的效果。

因此，**因子分解机**的应运而生。

因子分解机的**优势**是为每个特征学习了一个**隐权重向量（latent vector），**在特征交叉的时候，用这两个特征隐向量内积作为交叉特征的权重。如何理解这个含义呢？

在这里我们就要提一嘴矩阵分解。矩阵分解其实是将一个稀疏矩阵R分解为两个矩阵内积的形式，通过内积回乘，就能够得到一个满秩的矩阵。如果是电影的评分矩阵，也就是可以对未知的电影评分进行预测。具体可以看这篇文章：



而因子分解机FM就是在POLY2模型的基础上，融合了矩阵分解的思想。即，对二阶交叉特征的系数以矩阵分解的方式调整，让系数不再是独立无关的，同时解决数据稀疏导致的无法训练参数的问题：

$$
\large\hat{y}(\mathbf{x}):=w_{0}+\sum_{i=1}^{n} w_{i} x_{i}+\sum_{i=1}^{n} \sum_{j=i+1}^{n}\left\langle\mathbf{v}_{i}, \mathbf{v}_{j}\right\rangle x_{i} x_{j} \tag{4}
$$
其中，$\large w_{0} \in \mathbb{R}, \quad \mathbf{w} \in \mathbb{R}^{n}, \quad \mathbf{V} \in \mathbb{R}^{n \times k}, V$是一个$\large n * k$的向量矩阵，$\large n$是特征$\large x$的个数，$\large k$是个待确定的参数。

$\large \left\langle\mathbf{v}_{i}, \mathbf{v}_{j}\right\rangle$ 表示点乘,即向量对应位置乘积的和

$$
\large \left\langle\mathbf{v}_{i}, \mathbf{v}_{j}\right\rangle:=\sum_{f=1}^{k} v_{i, f} \cdot v_{j, f} \tag{5}
$$
由矩阵分解可知，对任意一个正定矩阵$\large \mathbf{W}$，都可以找到一个矩阵$\large \mathbf{V}$，$\large \mathbf{}$且在矩阵$\large \mathbf{V}$维度$\large k$足够大的情况下使得$\large \mathbf{W}=\mathbf{V} \cdot \mathbf{V}^{t}$成立。因此，通过矩阵分解用两个向量 $\Large v_{i}$ 和 $\Large v_{j}$ 的内积近似原先矩阵$\large \mathbf{W}$。

其次，在拆解为 $\large\mathbf{v}_{i}$ 和之 $\large\mathbf{v}_{j}$ 后，参数更新时是对这两个向量分别更新的，那么在更新时，对于向量，我$\large\mathbf{v}_{i}$们不需要寻找到一组 $\Large x_{i}$和 $\Large x_{j}$同时不为0，我们只需要在$\large x_{i} \neq 0$的情况下，找到任意一个样本$\large x_{k} \neq 0$即可通过 $\Large x_{i}x_{k}$ 来更新 $\Large v_{i}$ 。

我们举个例子来理解下上面的定义：

在商品推荐的场景下，样本有两个特征，分别是类品和品牌。某个训练样本的特征组合是（足球，阿迪达斯）。在POLY2模型中，只有当“足球”和“阿迪达斯”同时出现在一个训练样本中时，模型才能够学到这个组合特征对应的权重。而在因子分解机FM中，“足球”的的隐向量也可以根据（足球，耐克）进行更新。“阿迪达斯”的隐向量也可以根据（篮球，阿迪达斯）更新，由此一来，就大幅度的降低了模型对稀疏性的要求。

更极端的情况，对于一个从未出现的组合（篮球，耐克），因为模型已经学习了“篮球”和“耐克”的隐向量，因此就具备了更新权重  $\Large w$​ 的能力，使其泛化能力大大提高。

## 3.降低时间复杂度

公式（3）的时间复杂度为 $O\left(k n^{2}\right)$ , 我们可以对二阶交叉特征进行化简，使时间复杂度降低到 $O\left(k n\right)$ 。

$$
\large \sum_{i=1}^{n} \sum_{j=i+1}^{n}\left\langle\mathbf{v}_{i}, \mathbf{v}_{j}\right\rangle x_{i} x_{j} \tag{6}
$$
![img](https://pic1.zhimg.com/80/v2-c3984c164afcb18d65ce0b3ad5f41b74_720w.jpg)

首先矩阵A中的上三角，红色方框部分代表公式（6），也就是因子分解机中二阶交叉项部分。由图可以看出，他是矩阵的全部元素减去对角线元素之和得到的, 即：

![img](https://pic1.zhimg.com/80/v2-3fa1f4caf873a21f6a9e59fdb769c3c0_720w.jpg)

$$
\large \sum_{i=1}^{n} \sum_{j=i+1}^{n}\left\langle\mathbf{v}_{i}, \mathbf{v}_{j}\right\rangle x_{i} x_{j}=\frac{1}{2}\left(\sum_{i=1}^{n} \sum_{j=1}^{n}\left\langle\mathbf{v}_{i}, \mathbf{v}_{j}\right\rangle x_{i} x_{j}-\sum_{i=1}^{n}\left\langle\mathbf{v}_{i}, \mathbf{v}_{i}\right\rangle x_{i} x_{i}\right)
$$
其中， $\large \sum_{i=1}^{n} \sum_{j=1}^{n}\left\langle\mathbf{v}_{i}, \mathbf{v}_{j}\right\rangle x_{i} x_{j}$​ 是矩阵全部元素之和。  $\large \sum_{i=1}^{n}\left\langle\mathbf{v}_{i}, \mathbf{v}_{i}\right\rangle x_{i} x_{i}$​代表对角线元素之和。

因为 ：

$$
\large \left\langle\mathbf{v}_{i}, \mathbf{v}_{i}\right\rangle=\sum_{f=1}^{k} v_{i, f}, v_{j, f}
$$
因此继续化简：

$$
\large =\frac{1}{2}\left(\sum_{i=1}^{n} \sum_{j=1}^{n} \sum_{f=1}^{k} v_{i, f} v_{j, f} x_{i} x_{j}-\sum_{i=1}^{n} \sum_{f=1}^{k} v_{i, f} v_{i, f} x_{i} x_{i}\right)
$$
$$
\large =\frac{1}{2} \sum_{f=1}^{k}\left(\left(\sum_{i=1}^{n} v_{i, f} x_{i}\right)\left(\sum_{j=1}^{n} v_{j, f} x_{j}\right)-\sum_{i=1}^{n} v_{i, f}^{2} x_{i}^{2}\right)
$$

$$
\large =\frac{1}{2} \sum_{f=1}^{k}\left(\left(\sum_{i=1}^{n} v_{i, f} x_{i}\right)^{2}-\sum_{i=1}^{n} v_{i, f}^{2} x_{i}^{2}\right) \tag{7}
$$

那么，**因子分解机的二阶表达式**就为：
$$
\large \hat{y}(x):=w_{0}+\sum_{i=1}^{n} w_{i} x_{i}+\frac{1}{2} \sum_{f=1}^{k}\left(\left(\sum_{i=1}^{n} v_{i, f} x_{i}\right)^{2}-\sum_{i=1}^{n} v_{i, f}^{2} x_{i}^{2}\right) \tag{8}
$$
其复杂度为  $O\left(k n\right)$​​  。考虑到特征的稀疏性，尽管  $\large n$​​可能很大，但很多  $\Large x_{i}$​​ 都是零。因此其实际复杂度应该是 $O(k \bar{n})$​​ ,其中  $\large \bar{n}$​​表示样本不为零的特征维度数量的平均值。

## 4.因子分解机FM求解

我们对公式（8）求偏导，可以计算因子分解模型对参数的梯度：

- 当参数为  $\large w_{i}$​​ 时：

$$
\large \frac{\partial \hat{y}(x)}{\partial w_{0}}=1 \tag{9}
$$



- 当参数为  $\large w_{i}$ 时，只跟它相关的  $\large x_{i}$ 有关:

$$
\large \frac{\partial \hat{y}(x)}{\partial w_{i}}=x_i \tag{10}
$$

当参数为   $\large v_{i,f}$​ 时:

$$
\large \begin{aligned} \frac{\partial \hat{y}(x)}{\partial v_{i, f}} &=\frac{\partial \frac{1}{2}\left(\left(\sum_{i=1}^{n} v_{i, f} x_{i}\right)^{2}-v_{i, f}^{2} x_{i}^{2}\right)}{\partial v_{i, f}} \\ &=\frac{1}{2}\left(2 x_{i} \sum_{i=1}^{n} v_{i, f} x_{i}-2 v_{i, f} x_{i}^{2}\right) \\ &=x_{i} \sum_{j=1}^{n} v_{j, f} x_{j}-v_{i, f} x_{i}^{2} \end{aligned} \tag{11}
$$

因此，**因子分解机FM模型对参数的梯度为**：

$$
\large \frac{\partial \hat{y}(x)}{\partial \theta}= \begin{cases}1, & \text { if } \theta \text { is } w_{0} \\ x_{i}, & \text { if } \theta \text { is } w_{i} \\ x_{i} \sum_{j=1}^{n} v_{j, f} x_{j}-v_{i, f} x_{i}^{2} & \text { if } \theta \text { is } v_{i, f}\end{cases} \tag{12}
$$

## 5.损失函数选取及算法流程

至此，我们就推导出了因子分解机的表达式以及参数的梯度。那么，损失函数在这里我们以以下两个为例, 并用梯度下降法求解：

### 回归问题：平方差损失函数

$$
\large Loss =\frac{1}{2} \sum_{i=1}^{n}\left(\hat{y}_{i}-y_{i}\right)^{2}
$$

求偏导得：

$$
\large \frac{\partial L}{\partial \hat{y}(x)}=(\hat{y}(x)-y)
$$
平方损失函数的梯度为：

$$
\large \frac{\partial L}{\partial \theta}=(\hat{y}(x)-y) * \frac{\partial \hat{y}(x)}{\partial \theta}
$$


### 分类问题：对数损失函数

$$
\large L o s s=\frac{1}{2} \sum_{i=1}^{n}-\ln \left(\sigma\left(\hat{y}_{i} y_{i}\right)\right)^{2}
$$

其中：

$$
\large \sigma(\hat{y} y)=\frac{1}{1+e^{-\hat{y} y}}
$$

$$
\large \frac{\partial(\sigma(\hat{y} y))}{\partial \hat{y}}=\sigma(\hat{y} y) *[1-\sigma(\hat{y} y)] * y
$$
对数函数下的梯度为：

$$
\large \frac{\partial L}{\partial \theta}=\frac{1}{\sigma(\hat{y} y)} * \sigma(\hat{y} y) *\left[1-\sigma(\hat{y} y] * y * \frac{\partial \hat{y}(x)}{\partial \theta}\right.
$$
$$
\large =[1-\sigma(\hat{y} y)] * y * \frac{\partial \hat{y}(x)}{\partial \theta}
$$



### 算法流程(以对数损失函数为例）

1. 初始化权重 $\large w_{0}, w_{1}, \ldots, w_{n}$​​ 和矩阵  $\mathbf{V}$​​
2. 对每一个样本：

$$
\large w_{0}=w_{0}-\alpha[1-\sigma(\hat{y} y)] * y
$$

对特征 $i \in(1, \ldots, n)$​ :

$$
\large w_{i}=w_{i}-\alpha[1-\sigma(\hat{y} y)] * y * x_{i}
$$
对  $f \in(1, \ldots, k)$​ :

$$
\large v_{i, f}=v_{i, f}-\alpha[1-\sigma(\hat{y} y)] * y *\left[x_{i} \sum_{j=1}^{n} v_{j, f} x_{j}-v_{i, f} x_{i}^{2}\right]
$$


3. 重复步骤2，直到满足终止条件。

## 参考文章

[Matrix Factorization 学习记录（一）：基本原理及实现_Multiangle's Notepad-CSDN博客](https://blog.csdn.net/u014595019/article/details/80586438)

[16.9. Factorization Machines — Dive into Deep Learning 0.17.0 documentation (d2l.ai)](http://www.d2l.ai/chapter_recommender-systems/fm.html)

[推荐系统中的矩阵分解总结_Rnan_prince的博客-CSDN博客_推荐系统矩阵分解](https://blog.csdn.net/qq_19446965/article/details/82079367?tdsourcetag=s_pctim_aiomsg)

[推荐系统之矩阵分解家族 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/35262187)

[推荐系统玩家 之 因子分解机FM（Factorization Machines） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/153500425)

[推荐系统玩家 之 矩阵分解(Matrix Factorization)的基本方法 及其 优缺点 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/145120275)

[分解机(Factorization Machines)推荐算法原理 - 刘建平Pinard - 博客园 (cnblogs.com)](https://www.cnblogs.com/pinard/p/6370127.html)

[(FM) Factorization Machines | 算法花园 (xiang578.com)](http://xiang578.com/post/fm.html)

[浅谈张量分解（一）：如何简单地进行张量分解？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/24798389)

[浅谈张量分解（二）：张量分解的数学基础 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/24824550)

[浅谈张量分解（三）：如何对稀疏矩阵进行奇异值分解？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/25512080)

[论文阅读 - Factorization Machines - wy-ei - 博客园 (cnblogs.com)](https://www.cnblogs.com/wy-ei/p/11534427.html)

《深度学习推荐系统》王喆

