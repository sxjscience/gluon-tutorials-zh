# 分类模型

前几节介绍的线性回归模型适用于输出为连续值的情景，例如输出为房价。在其他情景中，模型输出还可以是一个离散值，例如图片类别。对于这样的分类问题，我们可以使用分类模型，例如Softmax回归。和线性回归不同，Softmax回归的输出单元从一个变成了多个。本节以Softmax回归模型为例，介绍神经网络中的分类模型。Softmax回归是一个单层神经网络。


让我们考虑一个简单的分类问题。为了便于讨论，让我们假设输入图片的尺寸为$2 \times 2$，并设图片的四个特征值，即像素值分别为$x_1, x_2, x_3, x_4$。假设训练数据集中图片的真实标签为狗、猫或鸡，这些标签分别对应离散值$y_1, y_2, y_3$。举个例子，如果$y_1=0, y_2=1, y_3=2$，任意一张狗图片的标签记作0。



## Softmax运算

我们将一步步地描述Softmax回归是怎样对单个$2 \times 2$图片样本分类的。它将会用到Softmax运算。

设带下标的$w$和$b$分别为Softmax回归的权重和偏差参数。给定单个图片的输入特征$x_1, x_2, x_3, x_4$，我们有
$$
o_1 = x_1 w_{11} + x_2 w_{21} + x_3 w_{31} + x_4 w_{41} + b_1,\\
o_2 = x_1 w_{12} + x_2 w_{22} + x_3 w_{32} + x_4 w_{42} + b_2,\\
o_3 = x_1 w_{13} + x_2 w_{23} + x_3 w_{33} + x_4 w_{43} + b_3.
$$

图3.2用神经网络图描绘了上面的计算。
和线性回归一样，Softmax回归也是一个单层神经网络。和线性回归有所不同的是，Softmax回归输出层中的输出个数等于类别个数，因此从一个变成了多个。在Softmax回归中，$o_1, o_2, o_3$的计算都要依赖于$x_1, x_2, x_3, x_4$。所以，Softmax回归的输出层是一个全连接层。

![Softmax回归是一个单层神经网络](../img/softmaxreg.svg)

在得到输出层的三个输出后，我们需要预测输出分别为狗、猫或鸡的概率。不妨设它们分别为$\hat{y}_1, \hat{y}_2, \hat{y}_3$。下面，我们通过对$o_1, o_2, o_3$做Softmax运算，得到模型最终输出

$$
\hat{y}_1 = \frac{ \exp(o_1)}{\sum_{i=1}^3 \exp(o_i)},\\
\hat{y}_2 = \frac{ \exp(o_2)}{\sum_{i=1}^3 \exp(o_i)},\\
\hat{y}_3 = \frac{ \exp(o_3)}{\sum_{i=1}^3 \exp(o_i)}.
$$

由于$\hat{y}_1 + \hat{y}_2 + \hat{y}_3 = 1$且$\hat{y}_1 \geq 0, \hat{y}_2 \geq 0, \hat{y}_3 \geq 0$，$\hat{y}_1, \hat{y}_2, \hat{y}_3$是一个合法的概率分布。我们可将上面Softmax运算中的三式记作

$$\hat{y}_1, \hat{y}_2, \hat{y}_3 = \text{Softmax}(o_1, o_2, o_3).$$

我们有时把Softmax运算叫做Softmax层。


## 单样本分类的矢量计算表达式

为了提高计算效率，我们可以将单样本分类通过矢量计算来表达。在上面的图片分类问题中，假设Softmax回归的权重和偏差参数分别为

$$
\boldsymbol{W} = 
\begin{bmatrix}
    w_{11} & w_{12} & w_{13} \\
    w_{21} & w_{22} & w_{23} \\
    w_{31} & w_{32} & w_{33} \\
    w_{41} & w_{42} & w_{43}
\end{bmatrix},\quad
\boldsymbol{b} = 
\begin{bmatrix}
    b_1 & b_2 & b_3
\end{bmatrix},
$$




设$2 \times 2$图片样本$i$的特征为

$$\boldsymbol{x}^{(i)} = \begin{bmatrix}x_1^{(i)} & x_2^{(i)} & x_3^{(i)} & x_4^{(i)}\end{bmatrix},$$

输出层输出为
$$\boldsymbol{o}^{(i)} = \begin{bmatrix}o_1^{(i)} & o_2^{(i)} & o_3^{(i)}\end{bmatrix},$$

预测为狗、猫或鸡的概率分布为

$$\boldsymbol{\hat{y}}^{(i)} = \begin{bmatrix}\hat{y}_1^{(i)} & \hat{y}_2^{(i)} & \hat{y}_3^{(i)}\end{bmatrix}.$$


我们对样本$i$分类的矢量计算表达式为

$$
\boldsymbol{o}^{(i)} = \boldsymbol{x}^{(i)} \boldsymbol{W} + \boldsymbol{b},\\
\boldsymbol{\hat{y}}^{(i)} = \text{Softmax}(\boldsymbol{o}^{(i)}).
$$


## 小批量样本分类的矢量计算表达式


为了进一步提升计算效率，我们通常对小批量数据做矢量计算。广义上，给定一个小批量样本，其批量大小为$n$，输入个数（特征数）为$x$，输出个数（类别数）为$y$。设批量特征为$\boldsymbol{X} \in \mathbb{R}^{n \times x}$，批量标签$\boldsymbol{y} \in \mathbb{R}^{n \times 1}$。
假设Softmax回归的权重和偏差参数分别为$\boldsymbol{W} \in \mathbb{R}^{x \times y}, \boldsymbol{b} \in \mathbb{R}^{1 \times y}$。Softmax回归的矢量计算表达式为

$$
\boldsymbol{O} = \boldsymbol{X} \boldsymbol{W} + \boldsymbol{b},\\
\boldsymbol{\hat{Y}} = \text{Softmax}(\boldsymbol{O}),
$$

其中的加法运算使用了广播机制，$\boldsymbol{O}, \boldsymbol{\hat{Y}} \in \mathbb{R}^{n \times y}$且这两个矩阵的第$i$行分别为$\boldsymbol{o}^{(i)}$和$\boldsymbol{\hat{y}}^{(i)}$。


## 交叉熵损失函数

Softmax回归使用了交叉熵损失函数（cross-entropy loss）。以本节中的图片分类为例，真实标签狗、猫或鸡分别对应离散值$y_1, y_2, y_3$，它们的预测概率分别为$\hat{y}_1, \hat{y}_2, \hat{y}_3$。为了便于描述，设样本$i$的标签的被预测概率为$p_{\text{label}_i}$。例如，如果样本$i$的标签为$y_3$，那么$p_{\text{label}_i} = \hat{y}_3$。直观上，训练数据集上每个样本的真实标签的被预测概率越大（最大为1），分类越准确。假设训练数据集的样本数为$n$。由于对数函数是单调递增的，且最大化函数与最小化该函数的相反数等价，我们希望最小化

$$
\ell(\boldsymbol{\Theta}) = -\frac{1}{n} \sum_{i=1}^n \log p_{\text{label}_i},
$$
其中$\boldsymbol{\Theta}$为模型参数。该函数即交叉熵损失函数。在训练Softmax回归时，我们将使用优化算法来迭代模型参数并不断降低损失函数的值。


## 模型预测及评价

在训练好Softmax回归模型后，给定任一样本特征，我们可以预测每个输出类别的概率。通常，我们把预测概率最大的类别作为输出类别。如果它与真实类别（标签）一致，说明这次预测是正确的。在下一节的实验中，我们将使用准确率（accuracy）来评价模型的表现。它等于正确预测数量与总预测数量的比。

## 小结

* Softmax回归适用于分类问题。它使用Softmax运算输出类别的概率分布。
* Softmax回归是一个单层神经网络，输出个数等于分类问题中的类别个数。


## 练习

* 如果按本节Softmax运算的定义来实现它，可能会有什么问题？


## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/6403)

![](../img/qr_classification.svg)
