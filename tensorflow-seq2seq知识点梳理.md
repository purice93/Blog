---
title: tensorflow-seq2seq知识点梳理
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

>接触python已有两年之久，零散地使用tensorflow也将近一年。但是是指今日，如果让我重新建立一个项目，我仍是无能为力。有时候，我会有一种感觉，python这种语言就像是一个无底洞，你永远不知道它在不同的场景中有多少不同的变化，更可怕的是，你无法知晓其中的错误，它是如此的“灵活”，以至于很多的检查都需要依赖程序员自己，而这些错误有时候是很难检查的，特别是对于初学者而言。但是，这又是一个很容易入门的语言，因为，不管你是什么专业的人，你都可以用10行以内的代码做一些简单地事情，而不需要太多的系统知识或者规范。

>由于python知识点太过散乱，往往是学了后面忘了前面，所以还是要写一些总结加深印象，由于系统的总结可能太过于漫无目的，而且时间成本过高，所以，暂且以一个小项目为首，将其中的流程知识点串起来

1. 项目背景
项目为一个seq2seq架构，seq2seq架构是一种序列到序列的预测模型，即通过输入和输出都是序列。通常是用在自然语言处理中，比如翻译系统，输入为一维中文，输出为一维英文。改进版也使用在音频处理和图像处理上（这里面需要将音频特征、图像转化为序列形式）。
>输入为一个单词，输出为此单词的字母顺序排列；比如输入为hello，输出则为ehllo

2. 占位变量tf.placeholder

``` python
inputs = tf.placeholder(tf.int32, [None, None], name='inputs')
```
tensorflow中，由于我们是首先建立训练模型，后定义真实的输入数据。所有需要占位符来表示需要输入的未知变量，解析如下：
Args:
    dtype: 数据类型-The type of elements in the tensor to be fed.
    shape: 数据维度，一般为[批大小，序列长度,]-The shape of the tensor to be fed (optional). If the shape is not
      specified, you can feed a tensor of any shape.
    name: 数据名字-用于在tensorboard显示标注-A name for the operation (optional).

 Returns:
    A `Tensor` that may be used as a handle for feeding a value, but not
    evaluated directly.

3. tf.contrib.layers.embed_sequence(ids, vocab_size,  embed_dim)
词嵌入，一种one-hot编码的改进版，即将单词映射成数字表示，然后将数字转换为嵌入矩阵向量。

Args:
	ids: 形状为[batch_size, doc_length]的int32或int64张量，也就是经过预处理的输入数据。
	vocab_size: 输入数据的总词汇量，指的是总共有多少类词汇，不是总个数
	embed_dim：想要得到的嵌入矩阵的维度(自主设定，表示嵌入广度)

 Returns: [batch_size, doc_length, embed_dim] 
 	Tensor of [batch_size, doc_length, embed_dim] with embedded sequences.
	
4. tf.nn.dynamic_rnn

``` javascript
encoder_output, encoder_state = tf.nn.dynamic_rnn(cell, encoder_embed_input,
                                                      sequence_length=source_sequence_length, 			dtype=tf.float32)
```
这里的dynamic_rnn即构造循环神经网络，但是相对于传统的RNN有一定的改进，对于一般的RNN，对于输入序列，我们是强制性统一到最大长度，对于短于最大长度的进行补齐，但是这样会带来一个计算问题，所以这里使用了dynamic，动态RNN，它增加了一sequence_length输入参数-序列长度，这样，计算时只取当前长度的数值进行计算，输出时只保留当前长度的输出
参考：https://blog.csdn.net/u010223750/article/details/71079036

5. tf.strided_slice

``` javascript
ending = tf.strided_slice(data, [0, 0], [batch_size, -1], [1, 1])
```
 Args:
    input_: A `Tensor`.
    begin: An `int32` or `int64` `Tensor`.
    end: An `int32` or `int64` `Tensor`.
    strides: An `int32` or `int64` `Tensor`.

对数据矩阵进行切片，strides一般为全1值，这里的意思相当于切除最后一个字母。
![enter description here](http://osiy4s0ad.bkt.clouddn.com/soundblog/1538035875521.png)

6. tf.nn.embedding_lookup
输入张量映射

``` javascript
# 得到映射嵌入[128,?,15]/[128,length,15] = lookup([30,15],[128,legth])
decoder_embed_input = tf.nn.embedding_lookup(decoder_embeddings, decoder_input)
```
第一个参数为映射字典，第二个参数为映射索引

7. tf.identity(input, name=None)
tf.identity就是为了在计算图获取这个值而创建的虚拟节点，只是用来获取中间值

8. tf.contrib.seq2seq.sequence_loss
Args:
	logits: 训练输出的概率矩阵-A Tensor of shape [batch_size, sequence_length, num_decoder_symbols] and dtype float. The logits correspond to the prediction across all classes at each timestep.
	targets: 真是输出值-A Tensor of shape [batch_size, sequence_length] and dtype int. The target represents the true class at each timestep.
用于计算加权交叉熵损失，[加权交叉熵参考](https://blog.csdn.net/chaipp0607/article/details/73392175)

9. 梯度计算和网络更新
	* 梯度计算：gradients = optimizer.compute_gradients(cost)
	* 网络更新：train_op = optimizer.apply_gradients(capped_gradients)
		将梯度应用在变量上，返回此操作。即：使用该梯度进行权重更新

10. tf.clip_by_value(grad, -5., 5.）
数据截断，这里主要为了梯度裁剪：约束梯度值，防止梯度爆炸/跨度过大。是梯度范围限制在-5~5

11. sess.run()
Args:
      fetches: 元组：一些用于计算结果的张量，用于去除计算结果tensor （比如我们要求姐y=w\*x中的w，这里的fetches相当于y）
      feed_dict: 字典：输入变量占位符 （这里的feed_dict相当于x）