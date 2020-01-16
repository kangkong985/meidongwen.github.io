---
layout: post
algo: true
comments: true
title:  "deep learning"
excerpt: "..."
tag:
- algo
- deeplearning
- vae
---





1. 1. loss是整体网络进行优化的目标， 是需要参与到优化运算，更新权值W的过程的
2. metric只是作为评价网络表现的一种“指标”， 比如accuracy，是为了直观地了解算法的效果，充当view的作用，并不参与到优化过程


在keras中实现自定义loss， 可以有两种方式

方式一 自定义 loss function
    def vae_loss(x, x_decoded_mean):
        xent_loss = objectives.binary_crossentropy(x, x_decoded_mean)
        kl_loss = - 0.5 * K.mean(1 + z_log_sigma - K.square(z_mean) - K.exp(z_log_sigma), axis=-1)
        return xent_loss + kl_loss
     
    vae.compile(optimizer='rmsprop', loss=vae_loss)


方式二 通过自定义一个keras的层（layer）来达到目的， 作为model的最后一层，最后令model.compile中的loss=None
    # Custom loss layer
    class CustomVariationalLayer(Layer):
        def __init__(self, **kwargs):
            self.is_placeholder = True
            super(CustomVariationalLayer, self).__init__(**kwargs)
     
        def vae_loss(self, x, x_decoded_mean_squash):
            x = K.flatten(x)
            x_decoded_mean_squash = K.flatten(x_decoded_mean_squash)
            xent_loss = img_rows * img_cols * metrics.binary_crossentropy(x, x_decoded_mean_squash)
            kl_loss = - 0.5 * K.mean(1 + z_log_var - K.square(z_mean) - K.exp(z_log_var), axis=-1)
            return K.mean(xent_loss + kl_loss)
     
        def call(self, inputs):
            x = inputs[0]
            x_decoded_mean_squash = inputs[1]
            loss = self.vae_loss(x, x_decoded_mean_squash)
            self.add_loss(loss, inputs=inputs)
            # We don't use this output.
            return x


​     
​    y = CustomVariationalLayer()([x, x_decoded_mean_squash])
​    vae = Model(x, y)
​    vae.compile(optimizer='rmsprop', loss=None)



在keras中自定义metric非常简单，需要用y_pred和y_true作为自定义metric函数的输入参数   点击查看metric的设置


注意事项：
1. keras中定义loss，返回的是batch_size长度的tensor， 而不是像tensorflow中那样是一个scalar
2. 为了能够将自定义的loss保存到model，以及可以之后能够顺利load model，需要把自定义的loss拷贝到keras.losses.py 源代码文件下，否则运行时找不到相关信息，keras会报错


有时需要不同的sample的loss施加不同的权重，这时需要用到sample_weight，例如

            # Class weights:
            # To balance the difference in occurences of digit class labels. 
            # 50% of labels that the discriminator trains on are 'fake'.
            # Weight = 1 / frequency
            cw1 = {0: 1, 1: 1}
            cw2 = {i: self.num_classes / half_batch for i in range(self.num_classes)}
            cw2[self.num_classes] = 1 / half_batch
            class_weights = [cw1, cw2]   # 使得两种loss能够一样重要

discriminator.train_on_batch(imgs, [valid, labels], class_weight=class_weights)









## Batch Normalization

UserWarning: No training configuration found in save file: the model was *not* compiled. Compile it manually
忽略

使用SMT32CUBE.AI 导入模型文件.h5并进行Analysize时，报错：bad marchal data
原因：使用了不支持的Lambda层

 If your data is in the form of symbolic tensors, you should specify the `steps` argument (instead of the `batch_size` argument, because symbolic tensors are expected to produce batches of input data).
 模型输出的结果是张量，如果要使用这个结果，则要先把他转化为数值
 batch_size=None,steps=1







# FAQ

------

unindent does not match any outer indentation level
缩进错误

------

name '*****' is not defined
可能是定义该变量的那一行缩进错误

------

Cannot interpret feed_dict key as Tensor: Can not convert a int into a Tensor
查看是否重用了占位符，有重用的地方，改另外的变量名称即可

------

Error converting shape to a TensorShape: int() argument must be a string or a number, not 'tuple'
修改变量名

------

too many indices for array
矩阵的shape

------

numpy.loadtxt
数据错误，直接numpy.loadtxt()函数内部参数使用None缺省，shell会显示错误

------

UnboundLocalError： local variable 'xxx' referenced before assignment
在函数外部已经定义了变量n，在函数内部对该变量进行运算，运行时会遇到了这样的错误：主要是因为没有让解释器清楚变量是全局变量还是局部变量。

------

Using a `tf.Tensor` as a Python `bool` is not allowed. Use `if t is not None:` instead of `if t:` to test if a tensor is defined, and use TensorFlow ops such as tf.cond to execute subgraphs conditioned on the value of a tensor.
这里的原因是tensorflow的tensor不再是可以直接作为bool值来使用了，需要进行判断。
如：if grad: 改为  if grad is not None:

------

UnboundLocalError： local variable 'xxx' referenced before assignment
在函数外部已经定义了变量n，在函数内部对该变量进行运算，运行时会遇到了这样的错误：主要是因为没有让解释器清楚变量是全局变量还是局部变量。

------

网络获取minist数据集失败
改为本地加载，使用numpy.load加载npz文件

------

NoneType' object has no attribute '_inbound_nodes'
https://stackoverflow.com/questions/44627977/keras-multi-inputs-attributeerror-nonetype-object-has-no-attribute-inbound-n%EF%BC%89%EF%BC%8C

------

loss=nan
输入的数据含有负数

------

global name '*' is not defined
If the model you want to load includes custom layers or other custom classes or functions, you can pass them to the loading mechanism via the custom_objects argument:
from keras.models import load_model

Assuming your model includes instance of an "AttentionLayer" class
model = load_model('my_model.h5', custom_objects={'AttentionLayer': AttentionLayer})
Alternatively, you can use a custom object scope:
from keras.utils import CustomObjectScope
with CustomObjectScope({'AttentionLayer': AttentionLayer}):
    model = load_model('my_model.h5')
Custom objects handling works the same way for load_model, model_from_json, model_from_yaml:
from keras.models import model_from_json
model = model_from_json(json_string, custom_objects={'AttentionLayer': AttentionLayer})



When passing a list as loss, it should have one entry per model outputs. The model ha







# VAE

## reference

https://kexue.fm/archives/5253

