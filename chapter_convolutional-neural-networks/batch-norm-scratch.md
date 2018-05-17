# 批量归一化——从零开始

这一节我们介绍一个让深层卷积网络训练更加容易的层：批量归一化（batch normalization） [1]。回忆在[“实战Kaggle比赛：预测房价和K折交叉验证”](../chapter_supervised-learning/kaggle-gluon-kfold.md)这一节里，我们对输入数据做了归一化处理，就是将每个特征在所有样本上的值转换成均值为0方差为1。这样所有数值都同样量级上，从而使得训练的时候数值更加稳定。

这个数据归一化预处理对于浅层模型来说通常足够了，因为通过几层网络的作用后，输出值通常不会出现剧烈变化。但对于深层神经网络来说，情况可能会比较复杂。因为每一层都对输入乘以权重后得到输出。当很多层这样的相乘累计在一起时，最终的输出可能极大或者极小，而且权重改变都可能带来输出的巨大变化。为了让训练稳定，通常我们每次只能对权重做很小的改动，即使用很小的学习率，进而导致收敛缓慢。

批量归一化层的提出是针对这个情况。它将一个批量里的输入数据进行归一化然后输出。如果我们将批量归一化层放置在网络的各个层之间，那么就可以不断的对中间输出进行调整，从而保证整个网络的数值稳定性。


## 批量归一化层

我们首先看将批量归一化层放置在全连接层后时的情况。假设这个全连接层对一个批量数据输出$n$个向量数据点 $X = \{x_1,\ldots,x_n\}$，其中$x_i\in\mathbb{R}^p$。我们可以计算数据点在这个批量里面的均值和方差，其均长度为$p$的向量：

$$\mu_X \leftarrow \frac{1}{n}\sum_{i = 1}^{n}x_i,$$
$$\sigma_X^2 \leftarrow \frac{1}{n} \sum_{i=1}^{n}(x_i - \mu_B)^2.$$

对于数据点 $x_i$，我们可以对它的每一个特征维进行归一化：

$$\hat{x_i} \leftarrow \frac{x_i - \mu_X}{\sqrt{\sigma_X^2 + \epsilon}},$$

这里$\epsilon$是一个很小的常数保证不除以0。在上面归一化的基础上，批量归一化层引入了两个可以学习的模型参数，拉升参数 $\gamma$ 和偏移参数 $\beta$。它们均为$p$长向量，并作用在$\hat{x_i}$上：

$$y_i \leftarrow \gamma \hat{x_i} + \beta \equiv \mbox{BN}_{\gamma,\beta}(x_i).$$

这里$Y = \{y_1, \ldots, y_n\}$是批量归一化层的输出。

如果批量归一化层是放置在卷基层后面，那么我们将通道维当做是特征维，空间维（高和宽）里的元素则当成是样本来计算（参考[“多输入和输出通道”](./channels.md)里我们对$1\times 1$卷积层的讨论）。

通常训练的时候我们使用较大的批量大小来获取更好的计算性能，这时批量内样本均值和方差的计算都较为准确。但在预测的时候，我们可能使用很小的批量大小，甚至每次我们只对一个样本做预测，这时我们无法得到较为准确的均值和方差。对此，批量归一化层的解决方法是维护一个移动平滑的样本均值和方差来在预测时使用。

下面我们通过NDArray来实现这个计算。

```{.python .input  n=72}
import sys
sys.path.insert(0, '..')
import gluonbook as gb
from mxnet import nd, gluon, init, autograd
from mxnet.gluon import nn

def batch_norm(X, gamma, beta, moving_mean, moving_var,
               eps, momentum):
    # 通过 autograd 来获取是不是在训练环境下。
    if not autograd.is_training():
        # 如果是在预测模式下，直接使用传入的移动平滑均值和方差。
        X_hat = (X - moving_mean) / nd.sqrt(moving_var + eps)
    else:        
        assert len(X.shape) in (2, 4)
        # 接在全连接层后情况，计算特征维上的均值和方差。
        if len(X.shape) == 2:
            mean = X.mean(axis=0)
            var = ((X - mean)**2).mean(axis=0)
        # 接在二维卷基层后的情况，计算通道维上（axis=1）的均值和方差。这里我们需要保持 X 
        # 的形状以便后面可以正常的做广播运算。
        else:
            mean = X.mean(axis=(0,2,3), keepdims=True)
            var = ((X - mean)**2).mean(axis=(0,2,3), keepdims=True)
        # 训练模式下用当前的均值和方差做归一化。
        X_hat = (X - mean) / nd.sqrt(var + eps)
        # 更新移动平滑均值和方差。
        moving_mean = momentum * moving_mean + (1.0 - momentum) * mean
        moving_var = momentum * moving_var + (1.0 - momentum) * var
    # 拉升和偏移
    Y = gamma * X_hat + beta
    return (Y, moving_mean, moving_var)
```

接下来我们自定义一个BatchNorm层。它保存参与求导和更新的模型参数`beta`和`gamma`。同时也维护移动平滑的均值和方差使得在预测时可以使用。

```{.python .input  n=73}
class BatchNorm(nn.Block):
    def __init__(self, num_features, num_dims, **kwargs):
        super(BatchNorm, self).__init__(**kwargs)
        shape = (1,num_features) if num_dims == 2 else (1,num_features,1,1)
        # 参与求导和更新的模型参数，分别初始化成 0 和 1。
        self.beta = self.params.get('beta', shape=shape, init=init.Zero())
        self.gamma = self.params.get('gamma', shape=shape, init=init.One())
        # 不参与求导的模型参数。全在 CPU 上初始化成 0。
        self.moving_mean = nd.zeros(shape)
        self.moving_variance = nd.zeros(shape)
    def forward(self, X):
        # 如果 X 不在 CPU 上，将 moving_mean 和 moving_varience 复制到对应设备上。
        if self.moving_mean.context != X.context:
            self.moving_mean = self.moving_mean.copyto(X.context)
            self.moving_variance = self.moving_variance.copyto(X.context)
        # 保存更新过的 moving_mean 和 moving_var。
        Y, self.moving_mean, self.moving_variance = batch_norm(
            X, self.gamma.data(), self.beta.data(), self.moving_mean, 
            self.moving_variance, eps=1e-5, momentum=0.9)
        return Y
```

## 使用批量归一化层的LeNet

下面我们修改[“卷积神经网络”](./lenet.md)这一节介绍的LeNet来使用批量归一化层。我们在所有的卷基层和全连接层与激活层之间加入批量归一化层，来使得每层的输出都被归一化。

```{.python .input  n=74}
net = nn.Sequential()
net.add(
    nn.Conv2D(6, kernel_size=5),
    BatchNorm(6, num_dims=4),
    nn.Activation('sigmoid'),
    nn.MaxPool2D(pool_size=2, strides=2),
    nn.Conv2D(16, kernel_size=5),
    BatchNorm(16, num_dims=4),
    nn.Activation('sigmoid'),
    nn.MaxPool2D(pool_size=2, strides=2),
    nn.Dense(120),
    BatchNorm(120, num_dims=2),
    nn.Activation('sigmoid'),   
    nn.Dense(84),
    BatchNorm(84, num_dims=2),
    nn.Activation('sigmoid'),
    nn.Dense(10)
)
```

使用更前同样的超参数，可以发现前面五个迭代周期的收敛有明显加速。

```{.python .input  n=77}
ctx = gb.try_gpu()
net.initialize(force_reinit=True, ctx=ctx, init=init.Xavier())
trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate': 1})
loss = gluon.loss.SoftmaxCrossEntropyLoss()
train_data, test_data = gb.load_data_fashion_mnist(batch_size=256)
gb.train(train_data, test_data, net, loss, trainer, ctx, num_epochs=5)
```

最后我们查看下第一个批量归一化层学习到了`beta`和`gamma`。

```{.python .input  n=60}
(net[1].beta.data().reshape((-1,)),
 net[1].gamma.data().reshape((-1,)))
```

## 小结

批量归一化层对网络中间层的输出做归一化，来使得深层网络学习时数值更加稳定。

## 练习

* 尝试调大学习率，看看跟前面的LeNet比，是不是可以使用更大的学习率。
* 尝试将批量归一化层插入到LeNet的其他地方，看看效果如何，想一想为什么。
* 尝试不不学习`beta`和`gamma`（构造的时候加入这个参数`grad_req='null'`来避免计算梯度），看看效果会怎么样。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/1253)

![](../img/qr_batch-norm-scratch.svg)

## 参考文献

[1] Ioffe, Sergey, and Christian Szegedy. "Batch normalization: Accelerating deep network training by reducing internal covariate shift." arXiv:1502.03167 (2015).
