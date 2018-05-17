# 网络中的网络：NiN

回忆我们介绍的LeNet、AlexNet和VGG均由两个部分组成，输入图片首先进入由卷积层构成的部分充分抽取空间特征，然后再进入由全连接层构成的部分来输出最终分类结果。AlexNet和VGG对LeNet的改进主要在于如何加深加宽这两部分。

这一节我们介绍网络中的网络（NiN）[1]。它提出了另外一个思路，它串联多个由卷积层和“全连接”层构成的小网络来构建一个深层网络。

## 网络中的网络

我们知道卷积层的输入和输出都是四维数组，而全连接层则是二维数组。如果想在全连接层后再接上卷积层，则需要将其输出转回到四维。回忆在[“多输入和输出通道”](channels.md)这一小节里，我们介绍了$1\times 1$卷积，它可以看成将空间维（高和宽）上每个元素当做样本，并作用在通道维上的全连接层。NiN使用$1\times 1$卷积层来替代全连接层使得空间信息能够传递到后面的层去。下图对比了NiN同AlexNet和VGG等网络的主要区别。

![对比NiN（右）和其他（左）](../img/nin.svg)

NiN中的一个构成块由一个卷积层外加两个充当全连接层的$1\times 1$卷积层构成。第一个卷积层我们可以设置它的超参数，而第二和第三卷积层则除了设置它们的输出通道与之前一致外，别的超参数的均使用了固定值。如果使用

```{.python .input  n=2}
import sys
sys.path.insert(0, '..')
import gluonbook as gb
from mxnet import nd, gluon, init
from mxnet.gluon import nn

def nin_block(num_channels, kernel_size, strides, padding):
    blk = nn.Sequential()
    blk.add(nn.Conv2D(num_channels, kernel_size, 
                      strides, padding, activation='relu'),
            nn.Conv2D(num_channels, kernel_size=1, activation='relu'),
            nn.Conv2D(num_channels, kernel_size=1, activation='relu'))
    return blk
```

NiN的卷积层超参数跟Alexnet类似，使用窗口分别为$11\times 11$、$5\times 5$和$3\times 3$的卷积层。卷积层后跟窗口为$3\times 3$有重叠的最大池化层。但除了使用NiN块外，还有一点重要的跟AlexNet的不同在于去除了最后的三个全连接层。取而代之的是使用输出通道数等于标签类数的卷积层，然后在每一个通道使用一个平均池化层来将这个通道里的数值平均成一个标量作为输出。这个设计显著的减小模型参数大小，从而鞥更好的避免了过拟合。但也可能会造成训练时收敛变慢。

```{.python .input  n=9}
net = nn.Sequential()
net.add(
    nin_block(96, kernel_size=11, strides=4, padding=0),
    nn.MaxPool2D(pool_size=3, strides=2),
    nin_block(256, kernel_size=5, strides=1, padding=2),
    nn.MaxPool2D(pool_size=3, strides=2),
    nin_block(384, kernel_size=3, strides=1, padding=1),
    nn.MaxPool2D(pool_size=3, strides=2),
    nn.Dropout(.5),
    # 标签类数是 10。
    nin_block(10, kernel_size=3, strides=1, padding=1),
    # 全局平均池化层将窗口形状自动设置成输出的高和宽。
    nn.GlobalAvgPool2D(),
    # 将四维的输出转成二维的输出，其形状为（批量大小，10）。
    nn.Flatten()
)

```

我们构建一个数据来查看每一层的输出大小。

```{.python .input}
X = nd.random.uniform(shape=(1,1,224,224))

net.initialize()

for layer in net:
    X = layer(X)
    print(layer.name, 'output shape:\t', X.shape)
```

## 获取数据并训练

跟Alexnet和VGG类似，但使用了更大的学习率。

```{.python .input}
ctx = gb.try_gpu()
net.initialize(force_reinit=True, ctx=ctx, init=init.Xavier())
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': .1})

loss = gluon.loss.SoftmaxCrossEntropyLoss()
train_data, test_data = gb.load_data_fashion_mnist(batch_size=128, resize=224)
gb.train(train_data, test_data, net, loss, trainer, ctx, num_epochs=3)
```

## 小结

NiN提供了两个重要的设计思路：

- 重复使用由卷积层和代替全连接层的$1\times 1$卷积层构成的基础块来构建深层网络；
- 去除了容易造成过拟合的全连接层，而是替代成由输出通道数为标签类数的卷积层和全局平均池化层作为输出。

虽然因为精度和收敛速度等问题NiN并没有像本章中介绍的其他网络那么被广泛使用，但NiN的设计思想影响了后面的一系列网络的设计。

## 练习

- 多用几个迭代周期来观察网络收敛速度。
- 为什么NiN块里要有两个$1\times 1$卷积层，去除一个看看？

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/1661)

![](../img/qr_nin-gluon.svg)

## 参考文献

[1] Lin, M., Chen, Q., & Yan, S. (2013). Network in network. arXiv preprint arXiv:1312.4400.
