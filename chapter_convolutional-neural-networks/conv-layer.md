# 二维卷积层

卷积神经网络是指主要由卷积层（convolutional layer）组成的网络。因为它最常用来处理图片数据，其有高和宽两个空间维度（彩色图片的颜色通道维度将在之后小节讨论），所以最常用到的是二维卷积层。本小节本节我们将介绍简单形式的二维卷积层的是怎么工作的。

## 二维相关运算符

虽然卷积层得名于卷积运算符（convolution），但我们常用更加直观的相关运算符（correlation）来实现卷积层。一个二维相关运算符将一个二维核（kernel）数组作用在一个二维输入数据上来计算一个二维数组输出。下图演示了如何对一个`(3, 3)`形状的输入`X`作用`(2, 2)`形状的核`K`来计算输出`Y`。

![二维相关运算符，高亮了计算第一个输出元素所使用的输入和核数组元素。](../img/correlation.svg)

可以看到输出`Y`的形状是`(2, 2)`，且第一个元素是由`X`的左上的`(2, 2)`子数组与核数组按元素相乘后再相加得来。即`Y[0, 0] = (X[0:2, 0:2] * K).sum()`，这里`X`、`K`和`Y`的类型都是NDArray。接下来我们将`X`上子数组向左一列来计算`Y`的第二列第一个元素。以此类推计算得到所有结果。

下面我们将上述过程实现在`corr2d`函数里，它接受`X`和`K`，输出`Y`。

```{.python .input}
import sys
sys.path.append('..')
import gluonbook as gb
from mxnet import autograd, nd
from mxnet.gluon import nn

def corr2d(X, K):
    h, w = K.shape
    Y = nd.zeros((X.shape[0]-h+1, X.shape[1]-w+1))
    for i in range(Y.shape[0]):
        for j in range(Y.shape[1]):
            Y[i, j] = (X[i:i+h, j:j+w]*K).sum()
    return Y
```

构造上图中的数据来测试实现的正确性。

```{.python .input}
X = nd.array([[0,1,2], [3,4,5], [6,7,8]])
K = nd.array([[0,1], [2,3]])
corr2d(X, K)
```

## 二维卷积层

二维卷积层就是将输入和其维护的核数组，也称作卷积核，做相关运算，然后加上一个标量偏差来得到输出。它的模型参数包括了卷积核和标量偏差。在训练的时候，我们通常首先对卷积核进行随机初始化，然后不断迭代更新卷积核和偏差来拟合数据。

下面的我们基于`corr2d`函数来实现一个自定义的二维卷基层。在初始化函数里我们声明`weight`和`bias`这两个模型参数，前向计算函数则是直接调用`corr2d`再加上偏差。

```{.python .input  n=70}
class Conv2D(nn.Block):
    def __init__(self, kernel_size, **kwargs):
        super(Conv2D, self).__init__(**kwargs)
        self.weight = self.params.get('weight', shape=kernel_size)
        self.bias = self.params.get('bias', shape=(1,))

    def forward(self, x):
        return corr2d(x, self.weight.data()) + self.bias.data()
```

你也许会好奇既然称之为卷积层，为什么不使用卷积运算符呢？其实卷积运算的计算与二维相关运算类似，唯一的区别是反向的将核数组跟输入做乘法，即`Y[0, 0] = (X[0:2, 0:2] * K[::-1, ::-1]).sum()`。但是因为在卷基层里`K`是学习而来而，所以不论是正向还是反向访问都可以。

## 图片物体边缘检测

下面我们来看一个应用卷积层的简单应用：检测图片中物体的边缘，即找到像素变化的位置。首先我们构造一张$6\times 8$的图，它中间4列为黑（0），其余为白（1）。

```{.python .input  n=66}
X = nd.ones((6, 8))
X[:, 2:6] = 0
X
```

然后我们构造一个形状为`(1, 2)`的卷积核，使得其作用在相同的横向相邻元素上输出为0，否则输出非0。

```{.python .input  n=67}
K = nd.array([[1, -1]])
```

对`X`作用我们设计的核`K`后可以发现，从白到黑的边缘我们检测成了1，从黑到白则是-1，其余全是0。

```{.python .input  n=69}
Y = corr2d(X, K)
Y
```

这里我们可以看到卷积层通过重复的使用`K`来有效的发掘局部空间特征。

## 通过数据学习核数组

最后我们来看一个例子，它使用前面的`X`和`Y`来学习我们构造的`K`。我们首先构造一个卷积层，将其卷积核初始化成随机数组。然后在每一个迭代里，我们使用平方误差来比较`Y`和卷积层的输出，然后计算梯度来更新权重。

虽然我们之前构造了Conv2D类，但由于`corr2d`使用了对单个元素赋值（`[i, j]=`）的操作会导致无法自动求导，下面我们使用Gluon提供的Conv2D类来实现这个例子。

```{.python .input  n=83}
# 构造一个输出通道是 1（将在后面小节介绍通道），核数组形状是 (1，2) 的二维卷基层。
conv2d = nn.Conv2D(1, kernel_size=(1, 2))
conv2d.initialize()

# 二维卷基层使用 4 维输入输出，格式为（批量大小，通道数，高，宽），这里批量和通道均为 1.
X = X.reshape((1, 1, 6, 8))
Y = Y.reshape((1, 1, 6, 7))

for i in range(10):
    with autograd.record():
        Y_hat = conv2d(X)
        loss = (Y_hat - Y) ** 2
        if i % 2 == 1:
            print('batch %d, loss %.3f' % (i, loss.sum().asscalar()))
    loss.backward()
    # 为了简单起见这里忽略了偏差。
    conv2d.weight.data()[:] -= 3e-2 * conv2d.weight.grad()
```

可以看到10次迭代后误差已经降到了一个比较小的值，现在来看一下学习到的核。

```{.python .input}
conv2d.weight.data().reshape((1,2))
```

我们看到学到的核与我们之前定义的`K`非常接近。

## 小结

- 二维卷基层的核心计算是二维相关运算。在最简单的形式下，它对二维输入数据和卷积核做相关运算然后加上偏差。
- 我们可以设计卷积核来检测图片中的边缘，同时也可以通过数据来学习它。

## 练习

- 构造一个`X`它有水平方向的边缘，如何设计`K`来检测它？如果是对角方向的边缘呢？
- 试着对我们构造的`Conv2D`进行自动求导，会有什么样的错误信息？
- 在Conv2D的`forward`函数里，将`corr2d`替换成`nd.Convolution`使得其可以求导。
- 试着将conv2d的核构造成`(2, 2)`，会学出什么样的结果？
- 如果通过变化输入和核的矩阵来将相关运算表示成一个矩阵乘法。
- 如何构造一个全连接层来进行物体边缘检测？

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/6314)

![](../img/qr_conv-layer.svg)
