# 稠密连接网络：DenseNet

上一节介绍的ResNet中的跨层连接设计引申出了数个后续工作。这一节我们介绍其中的一个：稠密连接网络（DenseNet） [1]。 它与ResNet的主要区别如下图演示。

![ResNet（左）对比DenseNet（右）。](../img/densenet.svg)

可以看出主要区别在于层B的输入不是像ResNet那样通过加法和输出合并，而是通过在通道维上的跟层B的输出进行合并。因为使用了拼接，所以底层的输出会单独保留的进入上面所有层。这是为什么称之为“稠密连接“的原因。

DenseNet的主要构建模块是稠密块和过渡块，前者定义了输入和输出是如何拼接的，后者则用来控制通道数不要过大。

## 稠密块

DeseNet使用了ResNet改良版的“批量归一化、激活和卷积”结构（参见上一节习题），我们首先实现这个结构。

```{.python .input  n=1}
import sys
sys.path.append('..')
import gluonbook as gb

from mxnet import nd, gluon, init
from mxnet.gluon import nn

def conv_block(num_channels):
    blk = nn.Sequential()
    blk.add(nn.BatchNorm(), nn.Activation('relu'),
            nn.Conv2D(num_channels, kernel_size=3, padding=1))
    return blk
```

稠密块里有多个卷积块，每个卷积块使用相同的输出通道数。但在前向计算时，我们将每个卷积块的输出在通道维上同其输出合并进入下一个卷积块。

```{.python .input  n=2}
class DenseBlock(nn.Block):
    def __init__(self, num_convs, num_channels, **kwargs):
        super(DenseBlock, self).__init__(**kwargs)
        self.net = nn.Sequential()
        for _ in range(num_convs):
            self.net.add(conv_block(num_channels))

    def forward(self, X):
        for blk in self.net:
            Y = blk(X)
            # 在通道维上将输入和输出合并。
            X = nd.concat(X, Y, dim=1)
        return Y
```

下面例子中我们定义一个有两个输出通道数为10的卷积块，使用通道数为3的输入时，我们会得到通道数为$3+2\times 10=23$的输入。卷积块的通道数影响了输出通道数相对于输入通道数的增长，因此也被成为之增长率（growth rate）。

```{.python .input  n=8}
blk = DenseBlock(2, 10)
blk.initialize()
X = nd.random.uniform(shape=(4,3,8,8))
Y = blk(X)
Y.shape
```

## 过渡块

每个稠密块都会带来通道数的增加。使用过多则会带过于复杂的模型复杂度。过渡块（transition block）则是用来通过$1\times1$卷积层来减小通道数。同时它使用步幅为2的平均池化层来将高宽减半来进一步降低复杂度。

```{.python .input  n=3}
def transition_block(num_channels):
    blk = nn.Sequential()
    blk.add(nn.BatchNorm(), nn.Activation('relu'),
            nn.Conv2D(num_channels, kernel_size=1),
            nn.AvgPool2D(pool_size=2, strides=2))
    return blk
```

我们对前面的稠密块的输出使用通道数为10的过渡块。

```{.python .input}
blk = transition_block(10)
blk.initialize()
blk(Y).shape
```

## DenseNet模型

DenseNet跟ResNet一样首先使用单卷积层和最大池化层：

```{.python .input}
net = nn.Sequential()
net.add(nn.Conv2D(64, kernel_size=7, strides=2, padding=3),
        nn.BatchNorm(), nn.Activation('relu'),
        nn.MaxPool2D(pool_size=3, strides=2, padding=1))
```

不同于ResNet接下来使用四个基于残差块的模块，DenseNet使用的是四个稠密块。同ResNet一样我们可以设置每个稠密块使用多少个卷积层，这里我们设成4，跟上一节的ResNet 18保持一致。稠密块里的卷积层通道数（既增长率）设成32，所以每个稠密块将增加128通道。

ResNet里通过步幅为2的残差块来在每个模块之间减小高宽，这里我们则是使用过渡块来减半高宽，并且减半输入通道数。

```{.python .input  n=5}
num_channels = 64  # 当前的数据通道数。
growth_rate = 32
num_convs_in_dense_blocks = [4, 4, 4, 4]

for i, num_convs in enumerate(num_convs_in_dense_blocks):
    net.add(DenseBlock(num_convs, growth_rate))
    num_channels += num_convs * growth_rate  # 上一个稠密的输出通道数。
    # 在稠密块之间加入通道数减半的过渡块。
    if i != len(num_convs_in_dense_blocks)-1:
        net.add(transition_block(num_channels//2))
```

最后同ResNet一样我们接上全局池化层和全连接层来输出。

```{.python .input}
net.add(nn.BatchNorm(), nn.Activation('relu'), 
        nn.GlobalAvgPool2D(), nn.Dense(10))
```

## 获取数据并训练

因为这里我们使用了比较深的网络，所以我们进一步把输入减少到$32\times 32$来训练。

```{.python .input}
ctx = gb.try_gpu()
net.initialize(force_reinit=True, ctx=ctx, init=init.Xavier())
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': 0.1})
loss = gluon.loss.SoftmaxCrossEntropyLoss()
train_data, test_data = gb.load_data_fashion_mnist(batch_size=256, resize=96)
gb.train(train_data, test_data, net, loss, trainer, ctx, num_epochs=5)
```

## 小结

不同于ResNet中将输入加在输出上完成跨层连接，DenseNet使用拼接来使得底部层的输出能够更独立的进入顶部层。

## 练习

- DesNet论文中提交的一个优点是其模型参数比ResNet更小，这是为什么？
- DesNet被人诟病的一个问题是内存消耗过多。真的会这样吗？可以把输入换成$224\times 224$，来看看实际（GPU）内存消耗。
- 实现 [1] 中的表1提出的各个DenseNet版本。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/1664)

![](../img/qr_densenet-gluon.svg)

## 参考文献

[1] Huang, Gao, et al. "Densely connected convolutional networks." CVPR. 2017.
