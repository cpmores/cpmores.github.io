+++
title = 'CNN 图像分类底层构建'
date = 2026-02-01T14:27:06+08:00
draft = false
tags = ['ML']

+++

2026 年还写 CNN 的 Hello World 博客，是不是有些明日黄花。

内容被反反复复再生产，更不用提其生产工作也在被 AI 自身所替代。但是很可惜，理解的过程 AI 并不能替代我完成，AI Infra 推陈出新，谁又能把我的血肉硬件更新换代呢。

最近为了参与 Kubeflow 还是不得不开始着手学习一些基本概念（虽然对其应用层面并不感冒），想着写一个 CNN 图像分类，当然还是用 MNIST ，不使用第三方框架（Flux 之类），实际写起来感觉网络上的资源还是强理论弱实践，更要命的是，矩阵求偏导的过程显得神乎其神，代码实现时似乎无从下手，还有实际代码上的小细节是理论证明中所没有包含的。本文就浅浅记录一下完成一个完整的图像分类的工作流程。理论部分推荐：https://zhuanlan.zhihu.com/p/687501772

## 数据导入 dataLoader

数据镜像：https://gitee.com/fmubai/mnist-mirror

四个文件分别用于训练和测试：

```julia
function read_mnist_images(filename)::ClassifyImagesMatrix
    open(filename) do io
        # get meta
        magic_number = ntoh(read(io, Int32))
        num_images = ntoh(read(io, Int32))
        img_rows = ntoh(read(io, Int32))
        img_cols = ntoh(read(io, Int32))

        # read real images
        images = Array{UInt8}(undef, img_cols, img_rows, num_images)
        read!(io, images)

        images_float = permutedims(images, (2, 1, 3))
        normalized = Float32.(images_float) ./ 255.0f0

        return ClassifyImagesMatrix(normalized, img_rows, img_cols, num_images, filename)
    end
end

function read_mnist_labels(filename)::ClassifyLabelsMatrix
    open(filename) do io
        magic_number = ntoh(read(io, Int32))
        num_labels = ntoh(read(io, Int32))
        labels = Array{UInt8}(undef, num_labels)
        read!(io, labels)
        return ClassifyLabelsMatrix(labels, num_labels, filename)
    end
end
```

文件使用的是网络序，还得手动转为字节序，当然标签和图片是一一对应的。

## 设计层级 Layers

既然是自己手写，当然怎么简单怎么来，使用以下层级：

+ 卷积层
+ ReLU
+ 池化层
+ Flatten
+ 全连接层
+ Softmax
+ Loss

思路其实和函数拟合差不多，将图片输入卷积层，卷积层使用卷积操作提取信息，通过激活函数进行非线性化，在池化层进一步降阶，多进行几次然后展开为一维向量，进入全连接层，降阶到 10 ，就可以拿给 Softmax 转为概率，去预测标签了，Loss 就是与实际的偏移程度，反向传播将 Loss 对于每一层输入以及参数的偏导层层传递（当然是为了减少计算开销），进行一次网络参数的修改（在这个架构中只有卷积层、全连接层、Softmax 实际上是有参数需要修改的）。

---

为了增加模型的灵活性，使用 Chain 模式抽象层来构建模型（ 感慨 Julia 的 abstract type 太美好了 ）

```julia
abstract type AbstractLayer end

struct LayerCache
    input,
    output,
    extra::Dict{Symbol,Any}
end

# needs implement
forward(layer::AbstractLayer, x) = error("Not implemented for $(typeof(layer))")
backward(layer::AbstractLayer, cache::LayerCache, grad_output) =
    error("Not implemented for $(typeof(layer))")

struct ConvLayer <: AbstractLayer
    weights,
    bias,
    stribe::Int,
    padding::Int,
    kernel_size::Tuple{Int,Int}


    function ConvLayer(num_in_channel::Int, num_out_channel::Int, stribe::Int=1,
        padding::Int=0, kernel_size::Tuple{Int,Int}=(2, 2))::ConvLayer
        weight = randn(Float32, kernel_size..., num_in_channel, num_out_channel)
        bias = zeros(Float32, num_out_channel)
        return ConvLayer(weight, bias, stribe, padding, kernel_size)
    end
end

struct ReLU <: AbstractLayer
end

struct MaxPoolLayer <: AbstractLayer
    size
end

struct Flatten <: AbstractLayer
end

struct DenseLayer <: AbstractLayer
    weights,
    bias

    function DenseLayer(num_input_channels::Int, num_output_channels::Int)
        weights = randn(Float32, num_output_channels, num_input_channels)
        bias = zeros(Float32, num_output_channels)
        return DenseLayer(weights, bias)
    end
end
```

除了各层本身的属性和参数外，还需要缓存各层输入输出的数据（反向传播时需要加入计算），像池化层（这里使用最大池），还需要记录额外信息（如最大值的位置）。



