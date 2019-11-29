<<<<<<< HEAD
# Multi-GPU Computation
=======
# Multi-GPU Computation Implementation from Scratch
:label:`sec_multi_gpu`
>>>>>>> 1ec5c63... copy from d2l-en (#16)

In this section, we will show how to use multiple GPU for computation. For example, we can train the same model using multiple GPUs. As you might expect, running the programs in this section requires at least two GPUs. In fact, installing multiple GPUs on a single machine is common because there are usually multiple PCIe slots on the motherboard. If the NVIDIA driver is properly installed, we can use the `nvidia-smi` command to view all GPUs on the current computer.

```{.python .input  n=1}
!nvidia-smi
```

<<<<<<< HEAD
As we discussed in the [“Automatic Parallel Computation”](auto-parallelism.md) section, most operations can use all the computational resources of all CPUs, or all computational resources of a single GPU. However, if we use multiple GPUs for model training, we still need to implement the corresponding algorithms. Of these, the most commonly used algorithm is called data parallelism.
=======
As we discussed in
:numref:`sec_auto_para`, most operations can use all the computational resources of all CPUs, or
all computational resources of a single GPU. However, if we use multiple GPUs
for model training, we still need to implement the corresponding algorithms. Of
these, the most commonly used algorithm is called data parallelism.
>>>>>>> 1ec5c63... copy from d2l-en (#16)


## Data Parallelism

<<<<<<< HEAD
In the deep learning field, Data Parallelism is currently the most widely used method for dividing model training tasks among multiple GPUs. Recall the process for training models using optimization algorithms described in the [“Mini-batch Stochastic Gradient Descent”](../chapter_optimization/minibatch-sgd.md) section. Now, we will demonstrate how data parallelism works using mini-batch stochastic gradient descent as an example.
=======
In the deep learning field, Data Parallelism is currently the most widely used
method for dividing model training tasks among multiple GPUs. Recall the process
for training models using optimization algorithms described in
:numref:`sec_minibatch_sgd`. Now, we will demonstrate how data parallelism works using minibatch
stochastic gradient descent as an example.
>>>>>>> 1ec5c63... copy from d2l-en (#16)

Assume there are $k$ GPUs on a machine. Given the model to be trained, each GPU will maintain a complete set of model parameters independently. In any iteration of model training, given a random minibatch, we divide the examples in the batch into $k$ portions and distribute one to each GPU. Then, each GPU will calculate the local gradient of the model parameters based on the minibatch subset it was assigned and the model parameters it maintains. Next, we add together the local gradients on the $k$ GPUs to get the current minibatch stochastic gradient. After that, each GPU uses this minibatch stochastic gradient to update the complete set of model parameters that it maintains. :numref:`fig_data_parallel` depicts the minibatch stochastic gradient calculation using data parallelism and two GPUs.

![Calculation of minibatch stochastic gradient using data parallelism and two GPUs. ](../img/data-parallel.svg)
:label:`fig_data_parallel`

In order to implement data parallelism in a multi-GPU training scenario from scratch, we first import the required packages or modules.

```{.python .input  n=2}
import sys
sys.path.insert(0, '..')

import d2l
<<<<<<< HEAD
import mxnet as mx
from mxnet import autograd, nd
from mxnet.gluon import loss as gloss
import time
=======
from mxnet import autograd, gluon, np, npx
npx.set_np()
>>>>>>> 1ec5c63... copy from d2l-en (#16)
```

## Defining the Model

<<<<<<< HEAD
We use LeNet, introduced in the [“Convolutional Neural Networks (LeNet)”](../chapter_convolutional-neural-networks/lenet.md) section, as the sample model for this section. Here, the model implementation only uses NDArray.
=======
We use LeNet, introduced in :numref:`sec_lenet`, as the sample model for this section. Here, the model implementation only uses `ndarray`s.
>>>>>>> 1ec5c63... copy from d2l-en (#16)

```{.python .input  n=10}
# Initialize model parameters
scale = 0.01
W1 = np.random.normal(scale=scale, size=(20, 1, 3, 3))
b1 = np.zeros(20)
W2 = np.random.normal(scale=scale, size=(50, 20, 5, 5))
b2 = np.zeros(50)
W3 = np.random.normal(scale=scale, size=(800, 128))
b3 = np.zeros(128)
W4 = np.random.normal(scale=scale, size=(128, 10))
b4 = np.zeros(10)
params = [W1, b1, W2, b2, W3, b3, W4, b4]

# Define the model
def lenet(X, params):
    h1_conv = npx.convolution(data=X, weight=params[0], bias=params[1],
                              kernel=(3, 3), num_filter=20)
    h1_activation = npx.relu(h1_conv)
    h1 = npx.pooling(data=h1_activation, pool_type='avg', kernel=(2, 2),
                     stride=(2, 2))
    h2_conv = npx.convolution(data=h1, weight=params[2], bias=params[3],
                              kernel=(5, 5), num_filter=50)
    h2_activation = npx.relu(h2_conv)
    h2 = npx.pooling(data=h2_activation, pool_type='avg', kernel=(2, 2),
                     stride=(2, 2))
    h2 = h2.reshape(h2.shape[0], -1)
    h3_linear = np.dot(h2, params[4]) + params[5]
    h3 = npx.relu(h3_linear)
    y_hat = np.dot(h3, params[6]) + params[7]
    return y_hat

# Cross-entropy loss function
loss = gloss.SoftmaxCrossEntropyLoss()
```

## Synchronize Data Among Multiple GPUs

We need to implement some auxiliary functions to synchronize data among the multiple GPUs. The following `get_params` function copies the model parameters to a specific GPU and initializes the gradient.

```{.python .input  n=12}
def get_params(params, ctx):
    new_params = [p.copyto(ctx) for p in params]
    for p in new_params:
        p.attach_grad()
    return new_params
```

Try to copy the model parameter `params` to `gpu(0)`.

<<<<<<< HEAD
```{.python .input  n=5}
new_params = get_params(params, mx.gpu(0))
=======
```{.python .input  n=13}
new_params = get_params(params, d2l.try_gpu(0))
>>>>>>> 1ec5c63... copy from d2l-en (#16)
print('b1 weight:', new_params[1])
print('b1 grad:', new_params[1].grad)
```

Here, the data is distributed among multiple GPUs. The following `allreduce` function adds up the data on each GPU and then broadcasts it to all the GPUs.

```{.python .input  n=14}
def allreduce(data):
    for i in range(1, len(data)):
        data[0][:] += data[i].copyto(data[0].context)
    for i in range(1, len(data)):
        data[0].copyto(data[i])
```

Perform a simple test of the `allreduce` function.

<<<<<<< HEAD
```{.python .input  n=7}
data = [nd.ones((1, 2), ctx=mx.gpu(i)) * (i + 1) for i in range(2)]
print('before allreduce:', data)
=======
```{.python .input  n=16}
data = [np.ones((1, 2), ctx=d2l.try_gpu(i)) * (i + 1) for i in range(2)]
print('before allreduce:\n', data[0], '\n', data[1])
>>>>>>> 1ec5c63... copy from d2l-en (#16)
allreduce(data)
print('after allreduce:\n', data[0], '\n', data[1])
```

<<<<<<< HEAD
Given a batch of data instances, the following `split_and_load` function can split the sample and copy it to each GPU.

```{.python .input  n=8}
def split_and_load(data, ctx):
    n, k = data.shape[0], len(ctx)
    m = n // k  # For simplicity, we assume the data is divisible
    assert m * k == n, '# examples is not divided by # devices.'
    return [data[i * m: (i + 1) * m].as_in_context(ctx[i]) for i in range(k)]
```
=======
## Splitting a Data Batch into Multiple GPUs

The `utils` module in Gluon provides a function to evenly split an array into multiple parts along the first dimension, and then copy the $i^\mathrm{th}$ part into the $i^\mathrm{th}$ device. It is straightforward to implement, but we will use the pre-implemented version so later chapters can reuse the `split_batch` function we will define later.
>>>>>>> 1ec5c63... copy from d2l-en (#16)

Now, we try to divide the 4 data instances equally between 2 GPUs using the `split_and_load` function.

<<<<<<< HEAD
```{.python .input  n=9}
batch = nd.arange(24).reshape((6, 4))
ctx = [mx.gpu(0), mx.gpu(1)]
splitted = split_and_load(batch, ctx)
print('input: ', batch)
=======
```{.python .input  n=8}
data = np.arange(24).reshape(4, 6)
ctx = [npx.gpu(0), npx.gpu(1)]
splitted = gluon.utils.split_and_load(data, ctx)
print('input: ', data)
>>>>>>> 1ec5c63... copy from d2l-en (#16)
print('load into', ctx)
print('output:', splitted)
```

<<<<<<< HEAD
## Multi-GPU Training on a Single Mini-batch
=======
The `split_batch` function then splits both the features and labels.

```{.python .input  n=9}
# Saved in the d2l package for later use
def split_batch(X, y, ctx_list):
    """Split X and y into multiple devices specified by ctx."""
    assert X.shape[0] == y.shape[0]
    return (gluon.utils.split_and_load(X, ctx_list),
            gluon.utils.split_and_load(y, ctx_list))
```

## Multi-GPU Training on a Single Minibatch
>>>>>>> 1ec5c63... copy from d2l-en (#16)

Now we can implement multi-GPU training on a single minibatch. Its implementation is primarily based on the data parallelism approach described in this section. We will use the auxiliary functions we just discussed, `allreduce` and `split_and_load`, to synchronize the data among multiple GPUs.

```{.python .input  n=10}
def train_batch(X, y, gpu_params, ctx, lr):
    # When ctx contains multiple GPUs, mini-batches of data instances are
    # divided and copied to each GPU
    gpu_Xs, gpu_ys = split_and_load(X, ctx), split_and_load(y, ctx)
    with autograd.record():  # Loss is calculated separately on each GPU
        ls = [loss(lenet(gpu_X, gpu_W), gpu_y)
              for gpu_X, gpu_y, gpu_W in zip(gpu_Xs, gpu_ys, gpu_params)]
    for l in ls:  # Back Propagation is performed separately on each GPU
        l.backward()
    # Add up all the gradients from each GPU and then broadcast them to all
    # the GPUs
    for i in range(len(gpu_params[0])):
        allreduce([gpu_params[c][i].grad for c in range(len(ctx))])
    # The model parameters are updated separately on each GPU
    for param in gpu_params:
        d2l.sgd(param, lr, X.shape[0])  # Here, we use a full-size batch
```

## Training Functions

Now, we can define the training function. Here the training function is slightly different from the one used in the previous chapter. For example, here, we need to copy all the model parameters to multiple GPUs based on data parallelism and perform multi-GPU training on a single minibatch for each iteration.

```{.python .input  n=11}
def train(num_gpus, batch_size, lr):
    train_iter, test_iter = d2l.load_data_fashion_mnist(batch_size)
    ctx = [mx.gpu(i) for i in range(num_gpus)]
    print('running on:', ctx)
    # Copy model parameters to num_gpus GPUs
    gpu_params = [get_params(params, c) for c in ctx]
    for epoch in range(4):
        start = time.time()
        for X, y in train_iter:
<<<<<<< HEAD
            # Perform multi-GPU training for a single mini-batch
            train_batch(X, y, gpu_params, ctx, lr)
            nd.waitall()
        train_time = time.time() - start

        def net(x):  # Verify the model on GPU 0
            return lenet(x, gpu_params[0])

        test_acc = d2l.evaluate_accuracy(test_iter, net, ctx[0])
        print('epoch %d, time: %.1f sec, test acc: %.2f'
              % (epoch + 1, train_time, test_acc))
=======
            # Perform multi-GPU training for a single minibatch
            train_batch(X, y, gpu_params, ctx_list, lr)
            npx.waitall()
        timer.stop()
        # Verify the model on GPU 0
        animator.add(epoch+1, (d2l.evaluate_accuracy_gpu(
            lambda x: lenet(x, gpu_params[0]), test_iter, ctx[0]),))
    print('test acc: %.2f, %.1f sec/epoch on %s' % (
        animator.Y[0][-1], timer.avg(), ctx_list))
>>>>>>> 1ec5c63... copy from d2l-en (#16)
```

## Multi-GPU Training Experiment

We will start by training with a single GPU. Assume the batch size is 256 and the learning rate is 0.2.

```{.python .input  n=12}
train(num_gpus=1, batch_size=256, lr=0.2)
```

By keeping the batch size and learning rate unchanged and changing the number of GPUs to 2, we can see that the improvement in test accuracy is roughly the same as in the results from the previous experiment. Because of the extra communication overhead, we did not observe a significant reduction in the training time.

```{.python .input  n=13}
train(num_gpus=2, batch_size=256, lr=0.2)
```

## Summary

* We can use data parallelism to more fully utilize the computational resources of multiple GPUs to implement multi-GPU model training.
* With the same hyper-parameters, the training accuracy of the model is roughly equivalent when we change the number of GPUs.

## Exercises

* In a multi-GPU training experiment, use 2 GPUs for training and double the `batch_size` to 512. How does the training time change? If we want a test accuracy comparable with the results of single-GPU training, how should the learning rate be adjusted?
* Change the model prediction part of the experiment to multi-GPU prediction.

## [Discussions](https://discuss.mxnet.io/t/2383)

![](../img/qr_multiple-gpus.svg)