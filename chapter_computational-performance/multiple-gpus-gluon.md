<<<<<<< HEAD
# Gluon Implementation for Multi-GPU Computation

In Gluon, we can conveniently use data parallelism to perform multi-GPU computation. For example, we do not need to implement the helper function to synchronize data among multiple GPUs, as described in the[“Multi-GPU Computation”](multiple-gpus.md) section, ourselves.
=======
# Concise Implementation of Multi-GPU Computation
:label:`sec_multi_gpu_gluon`

In Gluon, we can conveniently use data parallelism to perform multi-GPU
computation. For example, we do not need to implement the helper function to
synchronize data among multiple GPUs, as described in
:numref:`sec_multi_gpu`, ourselves.
>>>>>>> 1ec5c63... copy from d2l-en (#16)

First, import the required packages or modules for the experiment in this section. Running the programs in this section requires at least two GPUs.

```{.python .input  n=1}
import sys
sys.path.insert(0, '..')

import d2l
<<<<<<< HEAD
import mxnet as mx
from mxnet import autograd, gluon, init, nd
from mxnet.gluon import loss as gloss, nn, utils as gutils
import time
=======
from mxnet import autograd, gluon, init, np, npx
from mxnet.gluon import nn
npx.set_np()
>>>>>>> 1ec5c63... copy from d2l-en (#16)
```

## Initializing Model Parameters on Multiple GPUs

<<<<<<< HEAD
In this section, we use ResNet-18 as a sample model. Since the input images in this section are original size (not enlarged), the model construction here is different from the ResNet-18 structure described in the [“ResNet”](../chapter_convolutional-neural-networks/resnet.md) section. This model uses a smaller convolution kernel, stride, and padding at the beginning and removes the maximum pooling layer.

```{.python .input  n=2}
# This function is saved in the d2l package for future use
def resnet18(num_classes):
=======
In this section, we use ResNet-18 as a sample model. Since the input images in
this section are original size (not enlarged), the model construction here is
different from the ResNet-18 structure described in :numref:`sec_resnet`. This
model uses a smaller convolution kernel, stride, and padding at the beginning
and removes the maximum pooling layer.

```{.python .input  n=2}
# Saved in the d2l package for later use
def resnet18(num_classes):
    """A slightly modified ResNet-18 model."""
>>>>>>> 1ec5c63... copy from d2l-en (#16)
    def resnet_block(num_channels, num_residuals, first_block=False):
        blk = nn.Sequential()
        for i in range(num_residuals):
            if i == 0 and not first_block:
                blk.add(d2l.Residual(
                    num_channels, use_1x1conv=True, strides=2))
            else:
                blk.add(d2l.Residual(num_channels))
        return blk

    net = nn.Sequential()
    # This model uses a smaller convolution kernel, stride, and padding and
    # removes the maximum pooling layer
    net.add(nn.Conv2D(64, kernel_size=3, strides=1, padding=1),
            nn.BatchNorm(), nn.Activation('relu'))
    net.add(resnet_block(64, 2, first_block=True),
            resnet_block(128, 2),
            resnet_block(256, 2),
            resnet_block(512, 2))
    net.add(nn.GlobalAvgPool2D(), nn.Dense(num_classes))
    return net

net = resnet18(10)
```

Previously, we discussed how to use the `initialize` function's `ctx` parameter to initialize model parameters on a CPU or a single GPU. In fact, `ctx` can accept a range of CPUs and GPUs so as to copy initialized model parameters to all CPUs and GPUs in `ctx`.

```{.python .input  n=3}
ctx = [mx.gpu(0), mx.gpu(1)]
net.initialize(init=init.Normal(sigma=0.01), ctx=ctx)
```

Gluon provides the `split_and_load` function implemented in the previous section. It can divide a minibatch of data instances and copy them to each CPU or GPU. Then, the model computation for the data input to each CPU or GPU occurs on that same CPU or GPU.

```{.python .input  n=4}
<<<<<<< HEAD
x = nd.random.uniform(shape=(4, 1, 28, 28))
gpu_x = gutils.split_and_load(x, ctx)
=======
x = np.random.uniform(size=(4, 1, 28, 28))
gpu_x = gluon.utils.split_and_load(x, ctx)
>>>>>>> 1ec5c63... copy from d2l-en (#16)
net(gpu_x[0]), net(gpu_x[1])
```

Now we can access the initialized model parameter values through `data`. It should be noted that `weight.data()` will return the parameter values on the CPU by default. Since we specified 2 GPUs to initialize the model parameters, we need to specify the GPU to access parameter values. As we can see, the same parameters have the same values on different GPUs.

```{.python .input  n=5}
weight = net[0].params.get('weight')

try:
    weight.data()
except RuntimeError:
    print('not initialized on', mx.cpu())
weight.data(ctx[0])[0], weight.data(ctx[1])[0]
```

<<<<<<< HEAD
=======
Remember we define the `evaluate_accuracy_gpu` in :numref:`sec_lenet` to support evaluating on a single GPU, now we refine this implementation to support multiple devices.

```{.python .input  n=6}
# Saved in the d2l package for later use
def evaluate_accuracy_gpus(net, data_iter, split_f=d2l.split_batch):
    # Query the list of devices
    ctx_list = list(net.collect_params().values())[0].list_ctx()
    metric = d2l.Accumulator(2)  # num_corrected_examples, num_examples
    for features, labels in data_iter:
        Xs, ys = split_f(features, labels, ctx_list)
        pys = [net(X) for X in Xs]  # Run in parallel
        metric.add(sum(float(d2l.accuracy(py, y)) for py, y in zip(pys, ys)),
                   labels.size)
    return metric[0]/metric[1]
```

>>>>>>> 1ec5c63... copy from d2l-en (#16)
## Multi-GPU Model Training

When we use multiple GPUs to train the model, the `Trainer` instance will automatically perform data parallelism, such as dividing minibatches of data instances and copying them to individual GPUs and summing the gradients of each GPU and broadcasting the result to all GPUs. In this way, we can easily implement the training function.

```{.python .input  n=7}
def train(num_gpus, batch_size, lr):
    train_iter, test_iter = d2l.load_data_fashion_mnist(batch_size)
    ctx = [mx.gpu(i) for i in range(num_gpus)]
    print('running on:', ctx)
    net.initialize(init=init.Normal(sigma=0.01), ctx=ctx, force_reinit=True)
    trainer = gluon.Trainer(
        net.collect_params(), 'sgd', {'learning_rate': lr})
<<<<<<< HEAD
    loss = gloss.SoftmaxCrossEntropyLoss()
    for epoch in range(4):
        start = time.time()
        for X, y in train_iter:
            gpu_Xs = gutils.split_and_load(X, ctx)
            gpu_ys = gutils.split_and_load(y, ctx)
=======
    loss = gluon.loss.SoftmaxCrossEntropyLoss()
    timer, num_epochs = d2l.Timer(), 2
    animator = d2l.Animator('epoch', 'test acc', xlim=[1, num_epochs])
    for epoch in range(num_epochs):
        timer.start()
        for features, labels in train_iter:
            Xs, ys = d2l.split_batch(features, labels, ctx_list)
>>>>>>> 1ec5c63... copy from d2l-en (#16)
            with autograd.record():
                ls = [loss(net(gpu_X), gpu_y)
                      for gpu_X, gpu_y in zip(gpu_Xs, gpu_ys)]
            for l in ls:
                l.backward()
            trainer.step(batch_size)
<<<<<<< HEAD
        nd.waitall()
        train_time = time.time() - start
        test_acc = d2l.evaluate_accuracy(test_iter, net, ctx[0])
        print('epoch %d, time: %.1f sec, test acc %.2f' % (
            epoch + 1, train_time, test_acc))
=======
        npx.waitall()
        timer.stop()
        animator.add(epoch+1, (evaluate_accuracy_gpus(net, test_iter),))
    print('test acc: %.2f, %.1f sec/epoch on %s' % (
        animator.Y[0][-1], timer.avg(), ctx_list))
>>>>>>> 1ec5c63... copy from d2l-en (#16)
```

First, use a single GPU for training.

```{.python .input}
train(num_gpus=1, batch_size=256, lr=0.1)
```

Then we try to use 2 GPUs for training. Compared with the LeNet used in the previous section, ResNet-18 computing is more complicated and the communication time is shorter compared to the calculation time, so parallel computing in ResNet-18 better improves performance.

```{.python .input  n=10}
train(num_gpus=2, batch_size=512, lr=0.2)
```

## Summary

* In Gluon, we can conveniently perform multi-GPU computations, such as initializing model parameters and training models on multiple GPUs.

## Exercises

* This section uses ResNet-18. Try different epochs, batch sizes, and learning rates. Use more GPUs for computation if conditions permit.
* Sometimes, different devices provide different computing power. Some can use CPUs and GPUs at the same time, or GPUs of different models. How should we divide minibatches among different CPUs or GPUs?

## [Discussions](https://discuss.mxnet.io/t/2384)

![](../img/qr_multiple-gpus-gluon.svg)