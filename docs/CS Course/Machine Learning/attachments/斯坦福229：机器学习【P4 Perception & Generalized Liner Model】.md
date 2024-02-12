# 斯坦福229：机器学习【P4 Perceptron & Generalized Liner Model】

[【ML】斯坦福CS229：机器学习中英文字幕 by Andrew Ng](https://www.bilibili.com/video/BV19e411W7ga/?p=4&spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=2db6e07e62ad98affe473bcc11f7d9ea)

关键词：Perceptron感知器，Exponential Family指数族，广义线性模型，多分类问题的Softmax回归解法。

## Percptron algorithm 感知器算法

在上节课中我们学到了逻辑回归的原理，其中比较核心的就是 Sigmoid 函数。

$$
g(x)=\frac{1}{1+e^{-x}}
$$

而我们这里所提到的感知器算法使用的函数则是另一种函数:

$$
g(x)=\left \{ 
\begin{align}
1,\ x\ge0 \\
0,\ x< 0
\end{align}\right \}
$$

这时的 $h_\Theta(X)=g(\Theta^TX)$​ 也会相应的发生变化。 但不变的是他们的迭代规则是相同的：

$$
\Theta_j := \Theta_j + \alpha\sum_{i=1}^m[y^{(i)}-h_\Theta(X^{(i)})]X_{j}^{(i)}
$$

我们在这个迭代公式中可以发现 $y^{(i)}∈\{0,1\}$ ，而同样的 $h_\Theta(X^{(i)})∈\{0,1\}$ 。因此 $y^{(i)}-h_\Theta(X^{(i)})$ 有可能是 $\{-1,0,1\}$ 。这说明当 $y^{(i)}-h_\Theta(X^{(i)})=0$ 时，表示我们的估计 $h_\Theta(X^{(i)})$ 和 $y^{(i)}$ 是一致的。同样如果不一致会导致这个项结果为 $\{-1,1\}$ 。从代数角度来看，如果这一项是0的话也恰好说明针对当前训练示例 $i$ 的训练结果是正确的，并没有对参数集产生影响，而如果结果错误了会采取一定的修正。

具体感知器的修正过程不好表述，请原谅我在此处不做过多的阐述，如果小伙伴们听不懂视频中老师讲的修正过程，可以去[这里](https://zhuanlan.zhihu.com/p/492867531)看看~

总结一下，感知器算法本质上可以理解为用一个超平面将两类点在多维坐标系上分开，这个超平面表示为 $\Theta^TX$ ，当一个训练示例被超平面正确区分时，由于感知器函数的特性，导致这一次的 $\Theta$ 不会有任何变化。如果区分错误的话，对于 $\Theta_j$ 将会出现一个 $\alpha X_j^{(i)}$​ 的偏差调整使得该示例正确地回到超平面的某一边，而具体符号则会和这个示例的点在原超平面的上方或下方来确定。

但是显然这个算法是具有一定限制的，比如下图这个训练示例集就很难找到一个完美的分离边界来对这个情况做出区分。因此我们很少将这个算法用于实践中。但不得不承认的是他有一个很好的几何性质。

![image-20240212141717493](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P4%20Perception%20&%20Generalized%20Liner%20Model%E3%80%91img/image-20240212141717493.png)

## Exponential Family

指数组是一类概率分布，他们也与我们之后即将探讨的广义线性模型（GLM）相关。

这一族的概率密度函数可以写成：

$$
P(y;\eta)=b(y)·exp(\eta^TT(y)-a(\eta))=\frac{b(y)exp({\eta^TT(y)})}{e^{a(\eta)}}
$$

其中的 $y$ 是数据集，在这里使用 $y$ 而不是 $x$ 的原因是我们要使用指数组为数据输出建模。
 $\eta$ 是自然参数。可以是个标量也可以是个矢量。
 $T(y)$ 是充分统计量(Sufficient statistic)。其标矢性应与 $\eta$ 一致。而在今天这节课上我们的 $T(y)=y$ .
 $b(y)$ 是基本措施函数(Base measure)，是一个只和 $y$ 有关的函数，不能包含 $\eta$.
 $a(\eta)$ 是区分函数(log-partition function)，是一个只和 $、eta$ 有关的函数，不能包含 $y$​ .

我们之所以称他是一个族，是因为函数 $T,b,a$ 都是我们可以自行选择的函数，只要概率密度函数的积分为1即可。

### 伯努利分布的指数族归一化

比如我们常说的Bernoulli分布（二项分布），假设 $\phi$ 是某事件发生的概率，则有：

$$
\begin{align}
P(y;\phi)=&\phi^y·(1-\phi)^{(1-y)}\\
=&exp(log(\ \phi^y·(1-\phi)^{(1-y)}\ ))\\
=&exp(\ y·log(\frac{\phi}{1-\phi})+log(1-\phi)\ )\\
\end{align}
$$

至此我们可以发现：

$$
\begin{gather}
b(y)= 1\\
\eta=log\frac{\phi}{1-\phi} \Rightarrow \phi=\frac1{1+e^{-\eta}}\\
T(y)=y\\
a(\eta)=-log(1-\phi)=-log(1-\frac1{1+e^{-\eta}})=log(1+e^\eta)
\end{gather}
$$

聪明的你可以发现，这个指数族成员归一化至指数族的关键步骤就在于取 $exp(log(...)$ .除此之外还有一点在于， $\phi$ 长得很像我们的 Sigmoid 函数，这不是巧合，后续在GLM的介绍中会解释原因。

### 高斯分布的指数族归一化

我们知道高斯分布和均值与方差有关，我们假设方差 $\sigma ^2$ 为1。则其概率密度函数为：

$$
\begin{align}
P(y;\mu)=&\frac{1}{\sqrt{2\pi}}\exp(-\frac{{(y-\mu)}^2}{2})\\
=&\frac{1}{\sqrt{2\pi}}·e^{-\frac{y^2}{2}}exp(\mu y-\frac12\mu^2)
\end{align}
$$

同样对比可以发现：

$$
\begin{gather}
b(y)= \frac{1}{\sqrt{2\pi}}·e^{-\frac{y^2}{2}}\\
\eta=\mu\\
T(y)=y\\
a(\eta)=\frac12\mu^2
\end{gather}
$$

### 指数族的数学特性

1. 如果我们在指数族中执行最大似然性的取对数时会有一定优势：
   当存在指数族符合以上所述条件，即是分布可以由自然参数 $\eta$ 参数化时， $\eta$​ 的最大似然函数是凹函数。

2. $$
   E[y;\eta] = \frac{\partial}{\partial \eta}a(\eta)
   $$

3. $$
   Var[y;\eta]=\frac{\partial^2}{\partial \eta^2}a(\eta)
   $$

第而第三条性质被留作老师的课后作业了，我们也不给出具体证明了（其实是懒+不会）。

### 更多

指数族中除了高斯和伯努利以外，还有泊松分布，beta分布，gamma分布等等。

我们往往在训练示例集的 $y$ 呈现某特征时对应选择某分布来将其归一化入广义线性模型：

| $y$ 的特征 | 选择的分布         |
| ---------- | ------------------ |
| 实数集     | 高斯分布           |
| 二进制集   | 伯努利分布         |
| 非负整数集 | 泊松分布           |
| 正实数集   | Gamma分布/指数分布 |
| 概率分布集 | Beta分布           |

其中的概率分布集主要出现在贝叶斯机器学习或贝叶斯统计之中。

## 广义线性模型

### 一些假设与定义

同样我们需要为这个模型提出一些假设：

$$
\begin{align}
&(1)y|x;\Theta是指数族的一员\\
&(2)\eta=\Theta^TX ,\ \ \Theta,X ∈\Bbb R^n\\
&(3)在测试时，我们给出的预测h_\Theta(X)= E[y|X;\Theta]
\end{align}
$$

![image-20240212161857093](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P4%20Perception%20&%20Generalized%20Liner%20Model%E3%80%91img/image-20240212161857093.png)

如上图，也就是说我们的测试示例中的 $X$ 经过一个参数集 $\Theta$ 的线性模型处理后得到一个自然参数 $\eta$ 也就是 $\Theta^TX$ ，并根据测试示例的 $y$ 选择指数族中的某一种分布，最后用 $\eta$​ 参数化我们的后可以得出我们的假设。而我们在机器学习中学习调整的实际上是 $\Theta$​ 而不是指数族中的自然参数或任何相关的函数。

### 学习阶段

在学习阶段我们所需要做的就是求最大似然函数（其原因也在之前线性回归的概率解释部分阐述过）：

$$
\max_\Theta \{\ log[\ P(y^{(i)};\Theta^TX^{(i)})\ ]\ \}
$$

具体我们是先分析来简化运算，然后决定是使用梯度上升/下降的办法来求最大似然函数。

事实证明，无论是哪类指数族的分布或是哪种GLM，我们学习更新的规则都是一样的：

$$
\Theta_j :=\Theta_j+\alpha \sum_{i=1}^m(y^{(i)}-h_\Theta(X^{(i)}))X_j^{(i)}
$$

你根据训练示例 $y$ 的分布选择合适的假设函数 $h_\Theta(X)=E[y|X;\Theta]$ 后给参数集 $\Theta$ 一个初始量后，即可开始你的学习过程。

> 补充一些术语：
>
> 我们知道 $\eta$ 是自然参数，则 $\mu=E[y;\eta]=g(\eta)$ 称为规范相应函数。通过反解得到 $\eta=g^{-1}(\mu)$ 可以称为规范连接函数。注意到 $g(\eta)$ 在指数族中存在对应的表现形式：
> 
> $$
> g(\eta)=\frac\partial{\partial\eta}a(\eta)
> $$
> 
> 至今我们已出现了三种参数：
>
> | 模型参数 | 自然参数 | 规范参数                                                    |
> | -------- | -------- | ----------------------------------------------------------- |
> | $\Theta$ | $\eta$   | $\phi-伯努利$<br />$\mu,\sigma ^2-高斯$<br />$\lambda-泊松$ |

### 广义线性模型下的逻辑回归

在逻辑回归中我们应该有：

$$
\begin{align}
\phi=&\frac1{1+e^{-\eta}}\\
a(\eta)=&-log(1-\phi)=log(1+e^\eta)\\
h_\Theta(X)=&E[y|X;\Theta]=\frac{\partial a(\eta)}{\partial\eta}=\frac{\partial a(\eta)}{\partial \phi}·\frac{\partial \phi}{\partial\eta}\\
=&\frac{-1}{1-\phi}·\frac{-e^{-\eta}}{(1+e^{-\eta})^2}\\
=&\frac{1}{1-\phi}·\phi^2·(1-\frac1\phi)=\phi=\frac1{1+e^{-\eta}}=\frac1{1+e^{-\Theta^TX}}
\end{align}
$$

经过这一串的推导，我们知道最初选择 Sigmoid 函数作为我们的假设函数其实是有原因的。一切都和原先的假设契合起来了。

### 广义线性模型下的线性回归

这一部分老师没讲，但是为了巩固，我还是花点时间敲敲公式好了：

$$
\begin{align}
\eta=&\mu\\
a(\eta)=&\frac12\mu^2\\
h_\Theta(X)=&E[y|X;\Theta]\\
=&\frac{\partial a(\eta)}{\partial\eta}=\eta=\Theta^TX
\end{align}
$$

## 多分类问题的Softmax Regression

在广义线性回归中，Softmax回归也是其中的一种也就是由非负整数训练示例集引申出来的对应回归方法。我们在此不再介绍具体根据广义线性回归得出的推导过程，接下来我们主要介绍另外一种定义方法，来试图观测其本质——交叉熵最小化法。

### 一些假设预定义

#### 简单定义

为了可视化我们的训练示例，我们暂时将训练示例集定为 $X∈ \Bbb R^2，y∈\{\Box,\triangle,\bigcirc\}$ ，并且分布如下：

![image-20240212173935490](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P4%20Perception%20&%20Generalized%20Liner%20Model%E3%80%91img/image-20240212173935490.png)

我们要做的就是随便在平面上取一点，需要你通过学习猜测他是哪种现状。

#### 抽象泛化定义

$$
\begin{gather}
k是类的个数\\
X^{(i)}∈\Bbb R^{n}\\
y^{(i)}=[\{0,1\}^k]∈\Bbb R^{1×k}\\
e.g.\ 在简单示例中的三角形可以表示为[0,1,0]而正方形可以表示为[1,0,0]\\
k个\Theta_{class}∈\Bbb R^{n}，每个类都有一组参数\\
我们有\Theta=\begin{bmatrix}
\Theta_{class_1}^T\\
\Theta_{class_2}^T\\
...\\
\Theta_{class_k}^T
\end{bmatrix}_{k×n}
\end{gather}
$$

> Q：这样定义的 $y$ 显然会造成很多数据的冗余，比如 $[0,1,1]$​ 这些有多个1的表示就成了没有意义的内容，为什么要这样定义呢？

> A：这是为了能够将多分类问题转化为针对每一个类的分类问题，也就是说在每一个类上我们探讨的情况又被抽象成是或不是的问题（{0，1}），这样一来就可以将这个新的回归向我们已有的基础上靠近。

### 学习过程

这里的 $\Theta_{class}^TX$ 可以用先前分类问题的几何表现来理解，若 $\Theta_{class}^TX>0$ 表示是class类中的，相反则是类外的。

![image-20240212183823931](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P4%20Perception%20&%20Generalized%20Liner%20Model%E3%80%91img/image-20240212183823931.png)

假设我们在坐标平面内取一点，由此我们可以得到一个针对该点的三个值：$\Theta_{\bigcirc}^TX^{(i)}\ \ \Theta_{\triangle}^TX^{(i)}\ \ \Theta_{\Box}^TX^{(i)}$​ ，而这三个值一定有正有负，并且有大有小。我们可以称这三个值为类的概率分布。

再通过熟悉的取 exp 操作，我们可以得到一组都是正数的结果：

![image-20240212183727841](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P4%20Perception%20&%20Generalized%20Liner%20Model%E3%80%91img/image-20240212183727841.png)

再经过标准化操作后，让纵轴的含义变为$\frac{exp(\Theta_{class}^TX^{(i)})}{\sum_{j\ in\ AllClass}\Theta_{j}^TX^{(i)}}$ ，于是你就有所有类的高度之和为1的图了。我们不妨将其认作 $y$ 的概率密度函数，即 $\hat P(y)$​ 。而这也可以作为我们的假设输出。既然现在的假设输出变为了概率分布形式的，那么我们也应当调整我们的 $y$ 来适应这个形式，于是之前对 $y$ 的定义就开始发挥作用了：

![image-20240212185450581](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P4%20Perception%20&%20Generalized%20Liner%20Model%E3%80%91img/image-20240212185450581.png)

真实的值将是一个只有一条柱子的概率分布图，也就是 $P(y)=[0,1,0]$ 的形式。而我们的学习目标就是最大限度的减少这两个分布之间的差距。标准的说，就是使两个分布之间的交叉熵最小。

### 如何获得最小交叉熵

我们定义交叉熵：

$$
CrossEnt(P,\hat P)=-\sum_{y\ in\ AllClass}P(y)log\hat P(y)
$$

在本例中之有 $P_\triangle=1$ ，于是可以进一步化简为：

$$
\begin{align}
CrossEnt(P,\hat P)=&-\sum_{y\ in\ AllClass}P(y)log\hat P(y)\\
=& -log\hat P(y_\triangle)=-log\frac{exp(\Theta_\triangle^TX)}{\sum_{j\in\{\triangle,\Box,\bigcirc\}}exp(\Theta_j^TX)}\\
=&log[exp(\Theta_{\triangle}^TX)+exp(\Theta_\bigcirc^TX)+exp(\Theta_{\Box}^TX)]\ -\ \Theta_\triangle^TX
\end{align}
$$

为了求交叉熵的最小值，我们需要对其使用批量梯度下降。

本节课程到此戛然而止了！估计是设置好的时间到了，后续课程没有录进去了。（悲）以下内容属于苯人yy，正确性有待考究。

苯人感觉 $CrossEnt(P,\hat P)$ 是一个标量，我们无法令标量对矢量求导，于是我们对原有的批量梯度下降做一点修正：

$$
\Theta_j:=\Theta_j-\alpha\sum_{i=1}^m\frac\partial{\partial\Theta_j^TX^{(i)}}CrossEnt(P,\hat P^{(i)}),j\in AllClass
$$

$$
\begin{align}
\frac{\partial CrossEnt(P,\hat P^{(i)})}{\partial\Theta_j^TX^{(i)}}=&\frac{\partial \{ log[exp(\Theta_{\triangle}^TX)+exp(\Theta_\bigcirc^TX)+exp(\Theta_{\Box}^TX)]\ -\ \Theta_\triangle^TX\} }{\partial\Theta_j^TX}\\
=&\frac{\partial \{ log[\sum_{k\ in\ All}exp(\Theta_{k}^TX)]\}}{\partial\{\sum_{k\ in\ All}exp(\Theta_{k}^TX )\}}·\frac{\partial \{exp(\Theta_{j}^TX  )\}}{\partial\Theta_j^TX}-(j=\triangle?1:0)\\
=&\frac{exp(\Theta_{j}^TX )}{\sum_{k\in AllClass}exp(\Theta_{k}^TX)}-(j=\triangle?1:0)
\end{align}
$$

$$
故\Theta_j:=\Theta_j-\alpha\sum_{i=1}^m\frac{exp(\Theta_{j}^TX^{(i)} )}{\sum_{k\in AllClass}exp(\Theta_{k}^TX^{(i)})}-(j=\triangle?:0)
$$

至此完成了对 $\Theta$ 迭代公式的求解！（完结撒花）
