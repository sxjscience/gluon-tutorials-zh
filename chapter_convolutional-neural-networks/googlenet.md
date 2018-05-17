# 含并行连结的网络：GoogLeNet

在2014年的Imagenet竞赛中，一个名叫GoogLeNet [1]的网络结构大放光彩。它虽然在名字上是向LeNet致敬，但在网络结构上已经很难看到LeNet的影子。GoogLeNet吸收了NiN的网络嵌套网络的想法，在此基础上做了很大的改进。在随后的几年里研究人员持续对它进行改进，提出了数个被广泛使用的版本。本小节将介绍这个模型系列的一个版本。

## Inception 块

GoogLeNet中的基础卷积块叫做Inception，得名于同名电影《Inception》，寓意梦中嵌套梦。比较上一节介绍的NiN，这个基础块在结构上更加复杂。

![Inception块。](../img/inception.svg)

由上图可以看出，Inception里有四个并行的线路。前三个通道里使用窗口大小分别是$1\times 1$、$3\times 3$和$5\times 5$的卷基层来抽取不同空间尺寸下的信息。其中中间两个线路会对输入先作用$1\times 1$卷积来将减小输入通道数，以此减低模型复杂度。第四条线路则是使用$3\times 3$最大池化层，后接$1\times 1$卷基层来变换通道。四条线路都使用了合适的填充来使得输入输出高宽一致。最后我们将每条线路的输出在通道维上合并在一起，输入到接下来的层中去。

Inception块中可以自定义的超参数是每个层的输出通道数，以此我们来控制模型复杂度。

```{.python .input  n=1}
import sys
sys.path.insert(0, '..')
import gluonbook as gb

from mxnet import nd, init, gluon
from mxnet.gluon import nn

class Inception(nn.Block):
    # c1 - c4 为每条线路里的层的输出通道数。
    def __init__(self, c1, c2, c3, c4, **kwargs):
        super(Inception, self).__init__(**kwargs)
        # 线路 1，单 1 x 1 卷积层。
        self.p1_1 = nn.Conv2D(c1, kernel_size=1, activation='relu')
        # 线路 2，1 x 1 卷积层后接 3 x 3 卷积层。
        self.p2_1 = nn.Conv2D(c2[0], kernel_size=1, activation='relu')
        self.p2_2 = nn.Conv2D(c2[1], kernel_size=3, padding=1, 
                              activation='relu')
        # 线路 3，1 x 1 卷积层后接 5 x 5 卷积层。
        self.p3_1 = nn.Conv2D(c3[0], kernel_size=1, activation='relu')
        self.p3_2 = nn.Conv2D(c3[1], kernel_size=5, padding=2,
                              activation='relu')
        # 线路 4，3 x 3最大池化层后接 1 x 1 卷积层。
        self.p4_1 = nn.MaxPool2D(pool_size=3, strides=1, padding=1)
        self.p4_2 = nn.Conv2D(c4, kernel_size=1, activation='relu')

    def forward(self, x):
        p1 = self.p1_1(x)
        p2 = self.p2_2(self.p2_1(x))
        p3 = self.p3_2(self.p3_1(x))
        p4 = self.p4_2(self.p4_1(x))
        # 在通道维上合并输出
        return nd.concat(p1, p2, p3, p4, dim=1)
```

## GoogLeNet模型

GoogLeNet跟VGG一样，在主体卷积部分中使用五个模块，每个模块之间使用步幅为2的$3\times 3$最大池化层来减小输出高宽。第一模块使用一个64通道的$7\times 7$卷基层。

```{.python .input  n=2}
b1 = nn.Sequential()
b1.add(
    nn.Conv2D(64, kernel_size=7, strides=2, padding=3, activation='relu'),
    nn.MaxPool2D(pool_size=3, strides=2)
)
```

第二模块使用两个卷基层，首先是64通道的$1\times 1$卷基层，然后是将通道增大3倍的$3\times 3$卷基层。它对应Inception块中的第二线路。

```{.python .input}
b2 = nn.Sequential()
b2.add(
    nn.Conv2D(64, kernel_size=1),
    nn.Conv2D(192, kernel_size=3, padding=1),
    nn.MaxPool2D(pool_size=3, strides=2)
)
```

第三模块串联两个完整的Inception块。第一个Inception块的输出通道数为256,其中四个线路的输出通道比例为2：4：1：1。且第二、三线路先分别将输入通道减小2倍和12倍后再进入第二层卷基层。第二个Inception块输出通道数增至480，每个线路比例为4：6：3：2。且第二、三线路先分别减少2倍和8倍通道数。

```{.python .input}
b3 = nn.Sequential()
b3.add(
    Inception(64, (96, 128), (16, 32), 32),
    Inception(128, (128, 192), (32, 96), 64),
    nn.MaxPool2D(pool_size=3, strides=2)
)
```

第四模块更加复杂，它串联了五个Inception块，其输出通道分别是512、512、512、528和832。其线路的通道分配类似之前，$3\times 3$卷积层线路输出最多通道，其次是$1\times 1$卷基层线路，之后是$5\times 5$卷基层和$3\times 3$最大池化层线路。其中线两个线路都会先按比减小通道数。这些比例在各个Inception块中都略有不同。

```{.python .input}
b4 = nn.Sequential()
b4.add(
    Inception(192, (96, 208), (16, 48), 64),
    Inception(160, (112, 224), (24, 64), 64),
    Inception(128, (128, 256), (24, 64), 64),
    Inception(112, (144, 288), (32, 64), 64),
    Inception(256, (160, 320), (32, 128), 128),
    nn.MaxPool2D(pool_size=3, strides=2)
)
```

第五模块有输出通道数为832和1024的两个Inception块，每个线路的通道分配使用同前的原则，但具体数字又是不同。因为这个模块后面紧跟输出层，所以它同NiN一样使用全局平均池化层来将每个通道高宽变成1。最后我们将输出变成二维数组后加上一个输出大小为标签类数的全连接层作为输出。

```{.python .input}
b5 = nn.Sequential()
b5.add(
    Inception(256, (160, 320), (32, 128), 128),
    Inception(384, (192, 384), (48, 128), 128),
    nn.GlobalAvgPool2D()
)

net = nn.Sequential()
net.add(b1, b2, b3, b4, b5, nn.Flatten(), nn.Dense(10))
```

因为这个模型相计算复杂，而且修改通道数不如VGG那样简单。本节里我们将输入高宽从224降到96来加速计算。下面演示各个模块之间的输出形状变化。

```{.python .input  n=3}
X = nd.random.uniform(shape=(1,1,96,96))

net.initialize()

for layer in net:
    X = layer(X)
    print(layer.name, 'output shape:\t', X.shape)
```

## 获取数据并训练

我们使用高宽为96的数据来训练。

```{.python .input}
ctx = gb.try_gpu()
net.initialize(force_reinit=True, ctx=ctx, init=init.Xavier())
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': .1})

train_data, test_data = gb.load_data_fashion_mnist(batch_size=128, resize=96)

loss = gluon.loss.SoftmaxCrossEntropyLoss()
gb.train(train_data, test_data, net, loss, trainer, ctx, num_epochs=3)
```

## 小结

Inception定义了一个有四条线路的子网络。它通过不同窗口大小的卷基层和最大池化层来并行抽取信息，使用$1\times 1$卷基层减低通道数来减少模型复杂度。GoogLeNet则精细的将多个Inception块和其他层串联起来。其通道分配比例是在ImageNet数据集上通过大量的实验得来。这个使得GoogLeNet和它的后继者一度是ImageNet上最高效的模型之一，即在给定同样的测试精度下计算复杂度更低。

## 练习

1. GoogLeNet有数个后续版本，尝试实现他们并运行看看有什么不一样。本小节介绍的是最先的版本 [1]。[2] 加入批量归一化层（后一小节将介绍），[3] 对Inception块做了调整。[4] 则加入了残差连接（后面小节将介绍）。
2. 对比AlexNet、VGG和NiN、GoogLeNet的模型参数大小。分析为什么后两个网络可以显著减小模型大小。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/1662)

![](../img/qr_googlenet-gluon.svg)

## 参考文献

[1] Szegedy, Christian, et al. "Going deeper with convolutions." CVPR, 2015.

[2] Ioffe, Sergey, and Christian Szegedy. "Batch normalization: Accelerating deep network training by reducing internal covariate shift." arXiv:1502.03167 (2015).

[3] Szegedy, Christian, et al. "Rethinking the inception architecture for computer vision." CVPR. 2016.

[4] Szegedy, Christian, et al. "Inception-v4, inception-resnet and the impact of residual connections on learning." AAAI. 2017.
