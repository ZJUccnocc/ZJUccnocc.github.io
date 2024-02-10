# 斯坦福229：机器学习【P2 Liner Regression & Gradient Descent】

[【ML】斯坦福CS229：机器学习中英文字幕 by Andrew Ng](https://www.bilibili.com/video/BV19e411W7ga?p=2&vd_source=2db6e07e62ad98affe473bcc11f7d9ea)

关键词：线性回归，批量和随机梯度下降（一种拟合线性回归模型的算法），正规方程（一种拟合线性模型的方法）

## Defination

**线性回归**是一种最简单的监督学习的回归问题。将数据集拟合成线性的映射方式。

## Workflow

```
											 x
											 |
Training Set  ->   Learning algorithm  ->  y=h(x)  ->  "hypothesis"
											 |
											 y
```

### Step 1：How do you represent the hypothesis?

在我们的一元线性回归中，我们会有： `h(x)=ax+b` 。（从数学理论上来讲，应该称为 "affine function" 仿射函数，但是在机器学习中将其视为线性函数，不默认b=0）

在多元线性回归中，我们有：

$$
\begin{align}
h_\Theta(X)=\Theta_1x_1+\Theta_2x_2+\Theta_3x_3+ ...+\Theta_ix_i+...+\Theta_0(\Theta_i为参数) \\
=\sum_{j=0}^n\Theta_jx_j(x_0=1)
\end{align}
$$

从中我们可以归纳出参数矩阵和特征矩阵：

$$
\begin{align}
Parameter\ matrix:&\Theta=\begin{pmatrix} 
\Theta_0 \ ... \ \Theta_j \ ...\ \Theta_n
\end{pmatrix}^T&\\
Feature\ matrix:&X=\begin{pmatrix} 
x_0 \ ... \ x_j \ ...\ x_n
\end{pmatrix}^T (x_0=1)\\
Target\ Variable:&Y\\
需要注意的是&,\Theta 和 X是(n+1)维的向量
\end{align}
$$

我们再定义： $m$ 为训练示例的数量(即表格的行数) ,$n$ 为特征的数量(即输入量的数量) ，$(X,Y)$ 是一组训练示例，而 $(X^{(i)},Y^{(i)})$ 表示第 $i$ 组训练示例 $i∈[1,m] $

下面有一个例子方便理解：

| Size(平方米) | Number of Rooms（个） | Prize(万元) |
| ------------ | --------------------- | ----------- |
| 105          | 4                     | 156         |
| 120          | 5                     | 193         |
| 153          | 7                     | 254         |

$$
\begin{gather}
在本例子中，n=2，m=3\\
X^{(1)}=\begin{pmatrix} 
1 ,105 ,4\end{pmatrix}^T
\qquad Y^{(1)}= 156\\
X^{(2)}=\begin{pmatrix} 
1 ,120 ,5\end{pmatrix}^T
\qquad Y^{(2)}= 193\\
X^{(3)}=\begin{pmatrix} 
1 ,254 ,7\end{pmatrix}^T
\qquad Y^{(3)}= 254\\
\end{gather}
$$

### Step2：Choose $\Theta$ to form hypothesis

在有了定义和概念之后，我们下一步要做的事情是找到正确的 $\Theta$ ：

$$
\begin{gather}
Find \ \Theta \ s.t. \ h_\Theta(X)\ close\ to\ Y\\
其中h_{\Theta}(X)表示\ h\ 是一个与X，\Theta 相关的表达式\\
换言之，我们的目标是Find \ \Theta \ s.t.\sum_{i=1}^m(h_{\Theta}(X)-y)^2取得最小值\\
为了简化求导运算，我们可以在求和符号前添加一个常数\frac{1}{2}\\
因此我们的目标式子变为J(\Theta)=\frac{1}{2}\sum_{i=1}^m(h_{\Theta}(X)^{(i)}-y^{(i)})^2\\
我们称J(\Theta)为成本函数
\end{gather}
$$

> Q：为什么成本函数中使用的是平方误差而不是绝对误差？

> A：教授说这些和下周讲的更多监督学习问题相关，目前还不是很能看出这里的妙处，等“下周”会见分晓。但本人感觉这里的平方是为了方便后续求导。

### Step3：Find the algorithm to minimize the J (Gradient Descent)

这里我们使用一种称为**梯度下降（Gradient descent）**的算法。

算法的内容是：从Theta的一些值开始，比如将Theta初始化为零向量。然后持续地更改Theta来减少J。

![image-20240206131919429](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P2%20Liner%20Regression%20&%20Gradient%20Descent%E3%80%91img/image-20240206131919429.png)

我们可以将图中的曲面看作一个地形图，假设你在地形图上的任意一点，这时你向四面八方任意一个方向走一小步，一定存在一步能够使你以最快的下降。在每走一步之后都重复一遍最后一定能走到一个最低点。

但是聪明的你一定发现了这个最低点大概率是一个局部最优解，也就是当你初始化在地形图上的其他点时很有可能得到另一个局部最优解。但神奇的是，事实证明并不会出现局部最优解。证明如下：

> 当你按照 `J(θ)` 的函数绘制线性回归模型时，我们得到的图像和上图具有局部最优解的图像并不一致。因为我们 `J(θ)` 函数是一个个平方项的和，于是 `J(θ)` 的结果对于任意的θ都是一个二次函数的形式，所以他的图像看上去永远是这样的：
>
> ![image-20240206153512422](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P2%20Liner%20Regression%20&%20Gradient%20Descent%E3%80%91img/image-20240206153512422.png)
>
> 这样看上去就可以知道唯一的局部最优就一定是全局最优解了。

那么怎么去实现我们的Gradient Descent呢？

首先我们假设当前的 $\Theta_j$ 递推关系如下： 

$$
\Theta_{j} := \Theta_j-\alpha \frac{\partial J(\Theta)}{\partial \Theta_j}，其中\alpha是学习率(常数)\\
$$

$$
\begin{align}
∵\frac{\partial J(\Theta)}{\partial \Theta_j}=&(\frac{1}{2}\sum_{i=1}^m(h_{\Theta}(X^{(i)})-y^{(i)})^2)'\\
=&\sum_{i=1}^m[\frac{1}{2}(h_{\Theta}(X^{(i)})-y^{(i)})^2]'\\
=&\sum_{i=1}^m(h_{\Theta}(X^{(i)})-y^{(i)})\frac{\partial (h_{\Theta}(X^{(i)})-y^{(i)})}{\partial \Theta_j}\\
=&\sum_{i=1}^m(h_{\Theta}(X)^{(i)}-y^{(i)})·X_j^{(i)}
\end{align}\\
$$

$$
∴\Theta_{j} := \Theta_j-\alpha \sum_{i=1}^m(h_{\Theta}(X^{(i)})-y^{(i)})·X_j^{(i)}\ \ \ \ \ (j∈[0,n])
$$

接下来我们只需要重复上述步骤直到结果收敛.

据此，我们将 `α` 表征的量可以理解为一步的步长，如果 `α` 值过大则会导致结果不精确，但 `α` 值过小又会导致算法时长过长。因此我们通常会尝试多个 `α` 值并找到最有效的值。

下面举个简单的例子来可视化这个算法的运行过程。还是之前我们提到的一元监督学习的例子：

![image-20240206154113425](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P2%20Liner%20Regression%20&%20Gradient%20Descent%E3%80%91img/image-20240206154113425.png)

这是一个提供的数据集，可以看到在本例中**n=1**，m是图上的点的个数。当我们初始化 $\Theta_0=\Theta_1=0$ 时，得到的假设 $h_\Theta(X)=0$ ，也就是上图中的那条直线 $y=0$。

当我们开始梯度下降后，我们一步步调整向量 $\Theta$ 的值，使得图上的直线能够更好的表示数据集的分布情况。每一步的梯度下降都使得**所有点和假设直线之间的竖直距离的总和**减小，这也是为什么经过有限次的梯度下降，最后 $\Theta$ 一定会收敛的原因。

## 批量梯度下降与随机梯度下降

### 批量梯度下降

正如前面大篇幅讲述的情况一样，将一组示例数据看作一批数据，我们批量处理这一批批的数据，因此称为**批量梯度下降**。

批量梯度下降的**缺点**是为了根据你给出的数据集完成一次更新，哪怕只是计算一步的变化，你都需要计算一次$\Theta_{j} := \Theta_j-\alpha \sum_{i=0}^m(h_{\Theta}(X)-y)·X_j\\$​​ 之中的总和，于是每一步都会变得很慢。

尝试给出伪代码：

$$
\begin{align}
&Repeat\{  \\
&\ \ \ \ \ \ \ \ for\ j=0\ to\ n\{\\
&\ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \Theta_{j} := \Theta_j-\alpha \sum_{i=1}^m(h_{\Theta}(X^{(i)})-y^{(i)})·X_j^{(i)}\\
&\ \ \ \ \ \ \ \ \}\\
&\}
\end{align}
$$

然后贴上一点自己写的python代码（不确定是不是对的）

```python
for a in range ( times ):   #训练周期,一般是一个较大的数字。
    for j in range(n+1):	#在一轮训练周期内逐个调整Theta[j]
        grad = 0;
        for i in range(m):	#并在单次循环内
            hypothesis = 0	#根据X计算h的函数值
            for k in range(n+1): 
                hypothesis += Theta_matrix[k]*X[i][k] #累加得到第i个数据集下的偏差值
            grad += (hypothesis - y[i]) * X[i][j] #累加得到根据当前Theta_matrix计算得到的梯度值
        Theta[j] -= learning_rate*grad #根据梯度值调整第j个Theta
```

### 随机梯度下降

伪代码如下：

$$
\begin{align}
&Repeat\{  \\
&\ \ \ \ \ \ \ \ for\ j=0\ to\ n\{\\
&\ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ for\ i=1\ to\ m\{\\
&\ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \Theta_{j} := \Theta_j-\alpha (h_{\Theta}(X^{(i)})-y^{(i)})·X_j^{(i)}\\
&\ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \ \}\\
&\ \ \ \ \ \ \ \ \}\\
&\}
\end{align}
$$

不难看出，随机梯度下降并不会一遍遍重复计算 $\sum_{i=1}^m(h_{\Theta}(X^{(i)})-y^{(i)})·X_j^{(i)}$ ，而是选择只针对第 i 组示例中的值对 $\Theta$ 进行维护。

至于为什么这个算法是有效的，我们还是使用原先的假设。假设你在地形图上的任意一点初始化，我们只看一组数据而非数据集内的所有数据，这个时候我们也会向更低处迈出一步，但因为并没有将所有数据考虑进来，所以不一定是最速的下降方式。然后再接着看下一组数据，并重复......

所以当我们使用随机梯度下降时，他的路径可能更加随机和嘈杂，但是在整体上来看，他都是在想全局最优解前进。但与批量梯度下降不同的是，随机梯度下降**永远不会完全收敛**，只能达到一个最低水平，然后一圈圈的重复路径。但尽管如此，当你要处理的数据集极大时，我们还是会更多选择使用随机梯度下降。

> Q：能不能先使用随机梯度下降，再使用批量梯度下降？

> A：可以做到结合，比如在随机梯度下降中并不是针对一个示例进行计算，而是将巨量的数据分组进行计算。同样是不完全的，但是每次下降考虑的示例数量更多，下降的方向也就更精确。而这也是我们在实践中更常使用的方法。

> Q：什么时候停止随机梯度下降？

> A：通过监测 $J(\Theta)$ 的变化来判断是否停止。如果 $J(\Theta)$ 停止变化了，则可以停止。

## Normal equation（正态方程） 

！！注意这种方法仅适用于线性回归问题！！

### 一些标记

首先我们需要定义一下矩阵导数：

$$
\begin{gather}
假如我们有一个矩阵A∈\mathbb{R}^{n×n}，
并且存在映射f:\mathbb{R}^{n×n}→\mathbb{R}\\
则矩阵导数\nabla_A f(A)=
\left[\begin{array}{}
\frac{\partial f(A)}{\partial A_{11}} & \frac{\partial f(A)}{\partial A_{12}} &... & \frac{\partial f(A)}{\partial A_{1n}} \\
\frac{\partial f(A)}{\partial A_{21}} & \frac{\partial f(A)}{\partial A_{22}} &... & \frac{\partial f(A)}{\partial A_{2n}} \\
... & ... & ... & ...\\
\frac{\partial f(A)}{\partial A_{n1}} & \frac{\partial f(A)}{\partial A_{n2}} &... & \frac{\partial f(A)}{\partial A_{nn}} \\
\end{array}\right]_{n×n}∈\mathbb{R}^{n×n}\\
\end{gather}
$$

$$
\begin{align}
例如：有一次矩阵函数f(A)=&a_{11}A_{11}+a_{12}A_{12}+...+a_{1n}A_{1n}\\
+&a_{21}A_{21}+a_{22}A_{22}+...+a_{2n}A_{2n}\\
+&\ \ \ ...\ \ \ +\ \ \ ...\ \ \  +...+\ \ \ ...\ \ \ \\
+&a_{n1}A_{n1}+a_{n2}A_{n2}+...+a_{nn}A_{nn}\\
\end{align}\\
$$

$$
\begin{align}
则\nabla_A f(A)
=&
\left[\begin{array}{}
\frac{\partial f(A)}{\partial A_{11}} & \frac{\partial f(A)}{\partial A_{12}} &... & \frac{\partial f(A)}{\partial A_{1n}} \\
\frac{\partial f(A)}{\partial A_{21}} & \frac{\partial f(A)}{\partial A_{22}} &... & \frac{\partial f(A)}{\partial A_{2n}} \\
... & ... & ... & ...\\
\frac{\partial f(A)}{\partial A_{n1}} & \frac{\partial f(A)}{\partial A_{n2}} &... & \frac{\partial f(A)}{\partial A_{nn}} \\
\end{array}\right]_{n×n}\\
=&
\left[\begin{array}{}
a_{11} & a_{12} &... & a_{1n} \\
a_{21} & a_{22} &... & a_{2n} \\
... & ... & ... & ...\\
a_{n1} & a_{n2} &... & a_{nn} \\
\end{array}\right]_{n×n}\\
\end{align}\\
$$

有了矩阵倒数的定义之后我们可以将 $J(\Theta)$ 关于 $\Theta$ 求导：

$$
\nabla_\Theta J(\Theta)=
\left[\begin{array}{}
\frac{\partial J(\Theta)}{\partial \Theta_0}  \\
... \\
\frac{\partial J(\Theta)}{\partial \Theta_n} \\
\end{array}\right]_{(n+1)×1}
，\Theta ∈ \mathbb{R}^{n+1}
$$

根据导数的定义，我们可以知道当 $\nabla_\Theta J(\Theta)=\vec{0}$ 时，$J(\Theta)$ 可以取到极值（极大或极小）。又根据先前所说的$J(\Theta)$ 的图像可以知道这个时候的结果一定是全局最优解。

> 接下来我们给出一个有关矩阵的迹的定理但不予证明，感兴趣的读者可以自行证明（bushi）：
> 
> $$
> 有n阶矩阵A,B∈\mathbb{R}^{n×n}
> $$
> 
> 定理1：
> 
> $$
> 定义矩阵函数f(A)=tr(AB)，则\nabla_A J(A)=B^{T}
> $$
> 
> 定理2：
> 
> $$
> tr(AB)=tr(BA)，tr(ABC)=tr(CAB)
> $$
> 
> 定理3：（有点难证明，但是老师写这个的时候都是写一点看一点的，说明并没有特别重要）
> 
> $$
> \nabla_Atr(AA^TC)=CA+C^TA
> $$
> 
> 关于定理3，我们可以这么理解：$(ax^2)'=ax+ax$。只不过这里变成矩阵之后和转置相关了。

定理结束，我们回到正题。很显然我们需要重新给出矩阵形势下 $J(\Theta)$ 的定义：

$$
\begin{gather}
若X=\left[\begin{array}{}
X_{(1)}^T \\
X_{(2)}^T \\
... \\
X_{(m)}^T \\
\end{array}\right]_{m×(n+1)}
\Theta=\left[\begin{array}{}
\Theta_0 \\
\Theta_1 \\
... \\
\Theta_n \\
\end{array}\right]_{(n+1)×1}
Y=\left[\begin{array}{}
y_{(1)} \\
y_{(2)} \\
... \\
y_{(m)} \\
\end{array}\right]_{m×1}
\\
则矩阵形式的h_\Theta(X)=X\Theta=\left[\begin{array}{}
X_{(1)}^T\Theta \\
X_{(2)}^T\Theta \\
... \\
X_{(m)}^T\Theta \\
\end{array}\right]_{m×1}
=\left[\begin{array}{}
h_\Theta(X^{(1)}) \\
h_\Theta(X^{(2)}) \\
... \\
h_\Theta(X^{(m)}) \\
\end{array}\right]_{m×1}\\
\end{gather}
$$

$$
\begin{align}
最后矩阵形式的J(\Theta)=&\frac{1}{2}\sum_{i=1}^m(h_{\Theta}(X^{(i)})-y^{(i)})^2\\
=&\frac{1}{2}(X\Theta-Y)^T(X\Theta-Y)\\
=&\left[\begin{array}{}
h_\Theta(X^{(1)})-y^{(1)} \\	
h_\Theta(X^{(2)})-y^{(2)} \\
... \\
h_\Theta(X^{(m)})-y^{(m)} \\
\end{array}\right]_{m×1}\\
\end{align}
$$

于是我们进一步对 $\nabla_\Theta J(\Theta)=\vec{0}$ 进行化简：

$$
\begin{align}
\nabla_\Theta J(\Theta)=&\nabla_\Theta\frac{1}{2}(X\Theta-Y)^T(X\Theta-Y)\\
=&\frac{1}{2}\nabla_\Theta(\Theta^TX^T-Y^T)(X\Theta-Y)\\
=&\frac{1}{2}\nabla_\Theta(\Theta^TX^TX\Theta-\Theta^TX^TY-Y^TX\Theta +Y^TY)\\
=&\frac{1}{2}(X^TX\Theta+X^TX\Theta-X^TY-X^TY)\ \ (将四项对应求导)\\
=&X^TX\Theta-X^TY=\vec0\\
\end{align}
$$

根据上面的一段推到，最后我们可以得到的是：$X^TX\Theta=X^TY$。而这也就是我们这一部分所要讲的**正态方程**。

通过进一步移项我们可以得到：$\Theta=(X^TX)^{-1}X^TY$！

> Q:如果你的$X^TX$是不可逆的怎么办！？

> A：根据我们并不充足的线性代数知识，$X^TX∈\mathbb{R}^{(n+1)×(n+1)}$，若$X^TX$不可逆，则$R(X^TX)\neq n+1$，也就是说X中一定存在冗余的特征，或者某两个特征之间已经存在了固定的数量关系！



