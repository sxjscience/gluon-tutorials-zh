# 多层感知机——从零开始

我们已经从上一章里了解了多层感知机的原理。下面我们一起来动手实现一个多层感知机。首先导入实现所需的包或模块。

```{.python .input}
import sys
sys.path.append('..')
import gluonbook as gb
from mxnet import autograd, gluon, nd
from mxnet.gluon import loss as gloss
```

## 获取和读取数据

我们继续使用Fashion-MNIST数据集。我们将使用多层感知机对图片进行分类。

```{.python .input  n=1}
batch_size = 256
train_iter, test_iter = gb.load_data_fashion_mnist(batch_size)
```

## 定义模型参数

我们在[“Softmax回归——从零开始”](../chapter_crashcourse/softmax-regression-scratch.md)一节里已经介绍了，Fashion-MNIST数据集中图片尺寸为$28 \times 28$，类别数为10。本节中我们依然使用长度为$28 \times 28 = 784$的向量表示每一张图片。因此，输入个数为784，输出个数为10。实验中，我们设超参数隐藏单元个数为256。

```{.python .input  n=2}
num_inputs = 784
num_outputs = 10
num_hiddens = 256

W1 = nd.random.normal(scale=0.01, shape=(num_inputs, num_hiddens))
b1 = nd.zeros(num_hiddens)
W2 = nd.random.normal(scale=0.01, shape=(num_hiddens, num_outputs))
b2 = nd.zeros(num_outputs)
params = [W1, b1, W2, b2]

for param in params:
    param.attach_grad()
```

## 定义激活函数

这里我们使用ReLU作为隐藏层的激活函数。

```{.python .input  n=3}
def relu(X):
    return nd.maximum(X, 0)
```

## 定义模型

同Softmax回归一样，我们通过`reshape`函数将每张原始图片改成长度为`num_inputs`的向量。然后我们将上一节多层感知机的矢量计算表达式翻译成代码。

```{.python .input  n=4}
def net(X):
    X = X.reshape((-1, num_inputs))
    H = relu(nd.dot(X, W1) + b1)
    return nd.dot(H, W2) + b2
```

## 定义损失函数

为了得到更好的数值稳定性，我们直接使用Gluon提供的包括Softmax运算和交叉熵损失计算的函数。

```{.python .input  n=6}
loss = gloss.SoftmaxCrossEntropyLoss()
```

## 训练模型

训练多层感知机的步骤和之前训练Softmax回归的步骤没什么区别。我们直接调用`gluonbook`包中的`train_cpu`函数，它的实现已经在[“Softmax回归——从零开始”](../chapter_crashcourse/softmax-regression-scratch.md)一节里介绍了。

这里设超参数迭代周期为5，学习率为0.5。

```{.python .input  n=8}
num_epochs = 5
lr = 0.5
gb.train_cpu(net, train_iter, test_iter, loss, num_epochs, batch_size,
             params, lr)
```

## 小结

* 我们可以通过手动定义模型及其参数来实现简单的多层感知机。
* 当多层感知机的层数较多时，本节的实现方法会显得较繁琐，特别在定义模型参数上。

## 练习

- 改变 `num_hiddens`超参数的值，看看对结果有什么影响。
- 试着加入一个新的隐藏层，看看对结果有什么影响。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/739)

![](../img/qr_mlp-scratch.svg)
