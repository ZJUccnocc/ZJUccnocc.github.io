# 斯坦福229：机器学习【P3 Locally Weighted & Logiscal Regression】

[【ML】斯坦福CS229：机器学习中英文字幕 by Andrew Ng](https://www.bilibili.com/video/BV19e411W7ga/?p=3&vd_source=2db6e07e62ad98affe473bcc11f7d9ea)

关键词：局部加权回归（Locally weighted regression），线性回归的概率解释，逻辑回归（Logistic Regression），牛顿逻辑回归算法

## Locally Weighted Regression

局部加权回归是在线性回归的基础上对线性回归进行修改，使之可以拟合非线性函数。

那么不妨先回忆一下线性回归。

### 引入

在线性回归中我们假设每一个特征x都是一次的形式，所以我们有

$$
h_\Theta(X^{(i)})=\Theta_0X_0^{(i)}+\Theta_1X_1^{(i)}+...+\Theta_nX_n^{(i)}
$$

但有时候我们需要的不一定是线性的关系。我们可能会希望他是二次函数，甚至带有根号的函数......

聪明的你或许会想到可以通过重新赋值 $X_j$ 来解决：

$$
\begin{align}
假设有h_\Theta(X^{(i)})=&\Theta_0+\Theta_1X^{(i)}+\Theta_2{(X^{(i)})}^2+\Theta_3\sqrt{X^{(i)}}\\
=&\Theta_0X_0^{(i)}+\Theta_1X_1^{(i)}+\Theta_2X_2^{(i)}+\Theta_3X_3^{(i)}
\end{align}
$$

也就是说令 $X_1^{(i)}=X^{(i)},X_2^{(i)}={(X^{(i)})}^2,X_3^{(i)}=\sqrt{X^{(i)}}$ 即可得到我们熟悉的线性回归形式。

没错这确实是一种办法，但是我们这节课要学的另一种方法也可以很好的解决这个问题——**局部加权线性回归**。

### 术语补充

在正式开始学习局部加权线性回归前，我们需要补充一些有关机器学习的术语。

- Parametric Learnig Algorithm 参数学习算法

  在机器学习中我们通常会区别**参数**与**非参数**的学习算法。在参数学习算法中你会填入一些**固定的**参数集来适应你的数据集，比如我们之前用到的 $\Theta$ 。所以我们先前学习的线性回归是一个参数学习算法。

- Non-parametric Learning Algorithm 非参数学习算法

  非参数学习算法的特点是，训练示例/参数的数量会保持增长。局部加权线性回归将是我们学习的第一个非参数学习算法。在局部加权线性回归中是数据/参数的数量是呈线性增长的。

总结一下，**在参数学习算法中**，不论你的训练示例的个数有多大，当你确定好了你的参数集（$\Theta$）之后，你就可以把那些训练示例从你的计算机内存中删去，只依靠输入的X和已经计算得到的 $\Theta$ 来进行预测。但是**在非参数学习算法中**，你所需的储存空间将会随训练示例个数的增加而线性增长。因此我们可以知道这类算法并不适用于非常庞大的数据集。

### 比较线性回归和局部加权线性回归

![image-20240208135500715](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P3%20Locally%20Weighted%20&%20Logiscal%20Regression%E3%80%91img/image-20240208135500715.png)

还是从简单的情况入手，上图表示一套学习的训练示例，横轴表示一个特征X，纵轴表示输出值Y，坐标上的点对应了X与Y的关系。现在坐标轴上标出了 $x_0$ 表示要预测的点的位置

在线性回归问题中，对于如图所示的训练示例集，我们需要做的实际上就是找到 $\Theta$ 使得 $\frac12\sum_{i=1}^m(\Theta^Tx^{(i)}-y^{(i)})^2$ 取到最小值。然后将 $x_0$ 输入到我们的预测 $h_\Theta(x)$ 中，得到我们的预测结果。 

然而如果想要使用局部加权线性回归的话，我们还需要做更多：对 $x_0$ 附近的几组训练示例着重观察，同时结合所有的训练示例，最后总结出一条直线来进行预测。如下图：

![image-20240208141049592](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P3%20Locally%20Weighted%20&%20Logiscal%20Regression%E3%80%91img/image-20240208141049592-1707372650538-1.png)

相信聪明的你已经发现了，在局部加权线性回归中，与普通线性回归不同的是我们的成本函数 $J(\Theta)$ 。

### 局部加权回归的实现

我们只需要对 $J(\Theta)$ 做出如下修正：

$$
J(\Theta) = \sum_{i=1}^mw^{(i)}(\Theta^Tx^{(i)}-y^{(i)})^2\\
$$

其中 $w^{(i)}$ 是权重函数，通常我们令 $w^{(i)}$ 如下：

$$
w^{(i)}=\exp(-\frac{(x^{(i)}-x_0)^2}{2})
$$

这个权重函数具有的特点是：

$$
若\lvert x^{(i)}-x_0 \rvert 越小(最小值为0),则权重函数越接近1\\
若\lvert x^{(i)}-x_0 \rvert 越大，则权重函数越小(最小值为0)
$$

也就是说，$x^{(i)}$ 距离 $x_0$ 越远，则这个训练示例在 $J(\Theta)$ 中的影响就越小。聪明的你应该不难发现，这个函数曲线很类似正态分布的曲线。（但并非正态分布，因为积分后不会得到1）

然而实际上，在应用中我们并不会对所有训练示例进行计算，因为这将带来极大的时间开销，我们可以选择x周围一段长度为 $\tau$ 的范围按加权的方式计算。通过调整 $\tau$​ 你可以选择更宽的或者更窄的窗口。在引入了 $\tau$ 之后，我们的公式修正为：

$$
w^{(i)}=\exp(-\frac{(x^{(i)}-x_0)^2}{2\tau^2})
$$

> Q：如果 $x_0$ 超出了已有训练示例的 $x$ 的范围会出现什么情况？

> A：实践中可以看到这个方法还是可以用的，只是稍微不那么精确。

### 使用场景

我们通常在 $n$ 并不大的情况下使用局部加权线性回归，因为n变大时， $w^{(i)}$ 的运算次数就会增多，带来比较庞大的时间花销。

另外，当我们的训练示例集呈现明显的非线性关系时，可以采取这种方法来使结果更加精确。

## 线性回归的概率解释

### 一些假设与定义

根据之前学到的知识，我们可以作出假设：

$$
y^{(i)}=h_\Theta(X)+Error\_term=\Theta^TX^{(i)}+\varepsilon^{(i)}\ \ \ \ \ (1)
$$

其中的 $Error\_term$ 的意思是**未建模的影响项**，进一步可以理解为没有被 $X$ 囊括的其他影响项（未考虑到的，或是随机噪声）造成的偏差。为了能够量化这一偏差，我们假设 $\varepsilon^{(i)}$ 服从正态分布 $\mathcal N(0,\sigma^2)$。学过一些概率论与数理统计的小伙伴应该能联想到，这就意味着他的概率密度函数：

$$
P(\varepsilon^{(i)})=\frac{1}{\sqrt{2\pi}\sigma}\exp(-\frac{{(\varepsilon^{(i)})}^2}{2\sigma^2})
$$

并且我们还要假设 $\varepsilon^{(i)}$ 是独立同一分布（Independently and Identically Distributed，后文简称IID）的，这意味着任意一组**训练示例的误差项都是独立的**。而通过对（1）式简单的移项可以知道：

$$
P(y^{(i)}|X^{(i)};\Theta))=\frac{1}{\sqrt{2\pi}\sigma}\exp(-\frac{{(\Theta^TX^{(i)}-y^{(i)})}^2}{2\sigma^2})\\
\Rightarrow y^{(i)}|X^{(i)};\Theta \sim\mathcal N(\Theta^TX^{(i)},\sigma^2)
$$

也就是说我们给定了 $X$ 和 $\Theta$ 之后，特定的 $y$ 出现的概率的均值将是 $\Theta^TX^{(i)}$，方差将是 $\sigma^2$ 。需要注意的是，    "$y^{(i)}|X^{(i)};\Theta$" 这种写法表示：随机变量 $y^{(i)}$ 是由 $X^{(i)}$ （ $\Theta$ 参数化后）给定的。也就是说只有 $y^{(i)}$ 是随机的，而 $X^{(i)}$ 和 $\Theta$ 都是参数。

我们再定义参数 $\Theta$ 的**似然性(likelihood)**: $\mathcal{L}(\Theta)=P(\overline{y}|X,\Theta)$。其中的 **$\overline y$ 表示所有y的概率之和**。以防这时候有小伙伴忘记了似然概念，我简单地解释一下，似然本质上也是一种可能性。这里之所以用似然，主要是因为我们将 $\Theta$​​ 视为参数而非随机变量。更多的信息不妨[【点击链接】](https://www.zhihu.com/question/50828855)了解~

### 对参数似然性的处理

现在若在给定训练示例集 $X$ 与 $\overline{y}$ 的情况下，想知道哪个参数集才是最好的呢？ 我们不难想到：在哪个参数下，数据 $X$ 与 $\overline{y}$ 更有可能出现，则这个参数就是最好的，**也就是把最大化似然函数$\mathcal L(\Theta)$的那个参数看成是最好的**。注意我们这里把参数看成是变的。所以我们需要选择合适的 $\Theta$ 使得 $\mathcal L(\Theta)$ 最大。我们接着化简：

$$
\begin{align}
\\\mathcal{L}(\Theta)=&P(\overline{y}|X,\Theta)\\
=&\prod_{i=1}^mP(y^{(i)}|x^{(i)};\Theta)\\
=&\prod_{i=1}^m\frac{1}{\sqrt{2\pi}\sigma}\exp(-\frac{{(\Theta^TX^{(i)}-y^{(i)})}^2}{2\sigma^2})\\
\end{align}
$$

为了选择合适的 $\Theta$ 来最大化 $\mathcal L(\Theta)$，学过概率论与数理统计的小伙伴一定想起了最大似然估计（Maximum Likelihood Estimation，后文简称MLE），也就是通过取对数使得我们能够更好地求似然函数的最大值以及其条件。我们都知道对数函数在定义域内是单调递增的，所以最大的 $\mathcal l(\Theta)$ 和最大的 $\mathcal L(\Theta)$ 在同一刻出现。于是有： 

$$
\begin{align}
两边取对数:\mathcal{l}(\Theta)=&\ln[\mathcal{L}(\Theta)]\\
=&\sum_{i=1}^m\{\ln(\frac{1}{{\sqrt{2\pi}\sigma}})+\ln[exp(-\frac{{(\Theta^TX^{(i)}-y^{(i)})}^2}{2\sigma^2})]\}\\
=&m\ln\frac{1}{{\sqrt{2\pi}\sigma}}+\sum_{i=1}^m-\frac{{(\Theta^TX^{(i)}-y^{(i)})}^2}{2\sigma^2}
\end{align}
$$

通过简单地分析：

$$
\begin{align}
\mathcal L(\Theta)_{max} &\Leftrightarrow l(\Theta)_{max}\\
&\Leftrightarrow \{\sum_{i=1}^m-\frac{{(\Theta^TX^{(i)}-y^{(i)})}^2 }{2\sigma^2}\}_{max} \\
&\Leftrightarrow \{\sum_{i=1}^m\frac{{(\Theta^TX^{(i)}-y^{(i)})}^2 }{2\sigma^2}\}_{min} \\
&\Leftrightarrow \{\frac12\sum_{i=1}^m\{{(\Theta^TX^{(i)}-y^{(i)})}^2 \}_{min} \\
&\Leftrightarrow J(\Theta)_{min}
\end{align}
$$

到了这一步我们就成功将概率形式和矩阵形式以及基本形式三者相统一了。

## 逻辑回归解决分类问题

简单回顾一下，分类问题和回归问题是两种情况。分类问题的主要特点就是 $y$ 的取值是离散的。其中最简单的情况就是 $y∈\{0,1\}$​ 。

### 一些假设与定义

在这种情况下我们显然不会使用线性回归，那么我们接下来要介绍的算法叫做逻辑回归算法。 和线性回归算法一样，我们这里有个 $h_\Theta(X)∈[0,1]$。那么我们需要给出这个函数的定义：

$$
h_\Theta(X)=g(\Theta^TX)=\frac1{1+e^{-\Theta^TX}}\\
我们定义其中的\ g(x)=\frac1{1+e^{-x}}为逻辑函数
$$

不难发现这个 $g(x)$ 的图像如下：

![image-20240209124327205](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P3%20Locally%20Weighted%20&%20Logiscal%20Regression%E3%80%91img/image-20240209124327205.png)

函数 $g(x)∈(0,1)$ 恰好满足我们对 $h_\Theta(X)$ 取值范围的需求，而且 $g(x)$ 还有着很好的中心对称性。由于我们原先的为回归问题的假设函数 $h_\Theta(X)$ 所选择的与 $X$ 呈线性关系。于是我们可以得到： $h_\Theta(X)\propto \Theta^TX$ 。而逻辑函数 $g(x)$ 所做的就是将可能超出 $(0,1)$ 这个范围的 $\Theta^TX$ 控制的在 $(0,1)$ 之间。

> Q：为什么选择 $g(x)=\frac1{1+e^{-x}}$​ 作为逻辑函数？拥有相同特性的函数应该还有很多吧。

> A： 在未来的学习中我们会遇到一种实用性更广的算法——广义线性模型。而其有一种从这个函数中推到的方法。

接下来我们将假设数据具有如下分布：

$$
P(y=1|X;\Theta) = h_\Theta(X)\\
显然有 \ P(y=0|X;\Theta) = 1- h_\Theta(X)\\
$$

### 利用概率解释推进目标

根据这两条式子，聪明的你应该有办法将其归一化为一条式子：

$$
P(y|X;\Theta) = [h_\Theta(X)]^{y}·[1-h_\Theta(X)]^{1-y}，y∈\{0,1\}
$$

同样我们尝试求出 $\Theta$ 的似然性：

$$
\begin{align}
\\\mathcal{L}(\Theta)=&P(\overline{y}|X,\Theta)\\
=&\prod_{i=1}^mP(y^{(i)}|x^{(i)};\Theta)\\
=&[h_\Theta(X^{(i)})]^{y^{(i)}}·[1-h_\Theta(X^{(i)})]^{1-y^{(i)}}
\\
\end{align}
$$

同样的我们用最大似然估计：

$$
\begin{align}
两边取对数:\mathcal{l}(\Theta)=&\ln[\mathcal{L}(\Theta)]\\
=&\sum_{i=1}^m\{\ {y^{(i)}}\ln [h_\Theta(X^{(i)})]+(1-y^{(i)})\ln[1-h_\Theta(X^{(i)})]\ \}\\
\end{align}
$$

还是同样，我们需要尝试找到 $\Theta$ 使得 $\mathcal l(\Theta)$ 最大。

### 批量梯度上升法求最优解

这里我们选择使用的方法正是之前在线性回归中用过的批量梯度算法。不同的是这里的 $\mathcal l(\Theta)$ 是一个凸函数（形如 $f(x)=a\ln x+(1-a)ln(1-x),x∈(0,1)$ 的函数具有对称性，且是凸函数，求导可证，此处不予证明），因此这里使用的是批量梯度上升算法（由前可知当函数是凹函数时，梯度下降有全局最优解；同理当函数是凸函数时梯度上升有全局最优解）：

$$
\Theta_{j}:=\Theta_j+\alpha \frac{\partial \mathcal{l}(\Theta)}{\partial \Theta_j}
$$

由于视频中老师并没有给出$\frac{\partial \mathcal{l}(\Theta)}{\partial \Theta_j}$具体的计算过程而只给出了答案，我在下面尝试推导：

$$
∵h_\Theta(X)=g(\Theta^TX)\\
\begin{align}
有\frac{\partial \mathcal l(\Theta)}{\partial \Theta_j}=&
\frac{\partial \mathcal l(\Theta)}{\partial h_\Theta(X^{(i)})}·
\frac{\partial h_\Theta(X^{(i)})}{\partial \Theta^TX^{(i)}}·
\frac{\partial \Theta^TX^{(i)}}{\partial \Theta_j}\\
=&\sum_{i=1}^m\{(\frac{y^{(i)}}{h_\Theta(X^{(i)})}-\frac{1-y^{(i)}}{1-h_\Theta(X^{(i)})})·
\frac{\partial {g(t)}}{\partial t}·X_j^{(i)}\}\ \ \ (其中\ t=\Theta^TX)\\
而\frac{\partial {g(t)}}{\partial t}=&\frac{e^{-t}}{(1+e^{-t})^2}=\frac{1}{1+e^{-t}}·(1-\frac{1}{1+e^{-t}})=g(t)·(1-g(t))\\
=&h_\Theta(X^{(i)})·(1-h_\Theta(X^{(i)}))\\
\end{align}\\
∴\frac{\partial \mathcal l(\Theta)}{\partial \Theta_j}=\sum_{i=1}^m[y^{(i)}-h_\Theta(X^{(i)})]·X_j^{(i)}
$$

真是一场酣畅淋漓的求导啊！（bushi）接下来我们再代回批量梯度上升的式子可以得到：

$$
\Theta_{j}:=\Theta_j+\alpha \sum_{i=1}^m[y^{(i)}-h_\Theta(X^{(i)})]·X_j^{(i)}
$$

显然这个式子和线性回归的结果是一样的，只是一个是梯度上升，一个是梯度下降而已。但聪明的你应该会问：为什么呢？为什么我们一开始说不要在分类问题下使用线性回归的方法，但是现在却得到了基本一致的算法呢？

**如果用以后的知识来回答**，答案是回归问题和分类问题本质上都是广义线性模型的一种分支，所以他们会存在一致性。**如果用迄今为止的知识来回答**，答案是我们在 $h_\Theta(X)$ 上做了手脚。在线性回归问题中我们的 $h_\Theta(X)$ 就是 $\Theta^TX$ ，而分类问题中的 $h_\Theta(X)$ 却是 $g(\Theta^TX)$​​ 的改写。

> Q：既然分类问题和回归问题具有同一性，那么分类问题是否也有线性回归中存在的正态方程呢？

> A：实际上是没有的。在回归问题中，正态方程为你提供了一个简便的找到 $\Theta$ 的方法，

## 牛顿逻辑回归算法

牛顿逻辑回归的优势在于：梯度上升需要运行几百次甚至几千次的迭代次数才能得到比较好的 $\Theta $ 值，而牛顿逻辑回归算法只需要十来次即可得到相同程度的 $\Theta$​ 值。也就是说牛顿逻辑回归的效率更高。但缺点在于每次迭代的成本会更高一些。

### 我们的目标

我们想要求出最合适的 $\Theta$ 只需要最大化 $\mathcal l(\Theta)$ ，也就是找到 $\Theta$ 使得 $\mathcal l'(\Theta)=0$ 。而在牛顿算法实际上就是帮助我们： **假设有函数$f$，如何能够使 $f(\Theta)=0$ 的 $\Theta$ 。** 所以牛顿算法中的 $f(x)$ 本质上就是上文中的 $l'(\Theta)$​ 。

### 原理图

![image-20240209155620558](./%E6%96%AF%E5%9D%A6%E7%A6%8F229%EF%BC%9A%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0%E3%80%90P3%20Locally%20Weighted%20&%20Logiscal%20Regression%E3%80%91img/image-20240209155620558.png)

我们随机初始化一个 $\Theta_0$ ，第一次迭代过程就是在 $(\Theta_0,f(\Theta_0))$ 处做切线，取切线与坐标轴的交点为 $\Theta_1$ ，并对 $\Theta_1$​ 做相同处理迭代。

### 数学推导过程

#### 当 $\Theta ∈ \Bbb R^{1×1}$

$$
我们设 \Delta = \Theta^{(0)}-\Theta^{(1)}\ \ (1)\\
则显然有f'(\Theta^{(0)})=\frac{f(\Theta^{(0)})}{\Delta}\ \ (2)\\
$$

将(2)带入(1)有：

$$
\Theta^{(1)} = \Theta^{(0)}-\Delta=\Theta^{(0)}-\frac{f(\Theta^{(0)})}{f'(\Theta^{(0)})}
$$

由此我们可以得出牛顿逻辑回归的递推式：

$$
\Theta^{(t+1)} =\Theta^{(t)}-\frac{f(\Theta^{(t)})}{f'(\Theta^{(t)})}
$$

至此，我们再将最初的假设带回本式：

$$
\Theta^{(t+1)} :=\Theta^{(t)}-\frac{l'(\Theta^{(t)})}{l''(\Theta^{(t)})}
$$

#### 当 $\Theta ∈ \Bbb R^{(n+1)×1}$​

略去推导过程给出结果：

$$
\Theta^{(t+1)} := \Theta^{(t)}-{H^{(t)}}^{-1}\nabla_{\Theta^{(t)}} l\\
$$

其中的$\nabla_\Theta l$表示矩阵函数 $l$ 对矩阵 $\Theta$ 求导。

而其中的$H∈\Bbb R^{(n+1)×(n+1)} $,叫做Hessian矩阵。Hessian矩阵的特点在于：

$$
H_{ij}=\frac{\partial^2l}{\partial\Theta_i\partial\Theta_j}
$$

我们不难发现，当 $\Theta $​ 的维度上升时，一个大矩阵求逆将是需要很大开销的事情。而这甚至只是一次迭代所需要的开销。因此当你的 $\Theta$ 的维数并不大时，我们较多选择牛顿法进行回归，但如果你的 $\Theta$ 维度上升至上千甚至上万时，其效率甚至不如批量梯度上升法。

### Quadratic Convergence 二次收敛

牛顿回归算法具有二次收敛的特性。这表示如果在一次迭代中牛顿法的 $\Delta =0.01 $，那么在该点的0.01的范围内一定存在那个最优解。并且你获得的有效位数会在单次迭代中加倍，也就是说下一次迭代的 $\Delta$ 将会达到 $10^{-4}$ 数量级。这就是二次收敛。

牛顿法的二次收敛特性也使得牛顿法可以极快的收敛到全局最优解.
