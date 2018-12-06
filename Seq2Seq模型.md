---
title: Seq2Seq模型 
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

>seq2seq模型即通过序列预测序列，但是相对于传统单一深度学习系统，如CNN或者RNN，这些模型的输入输出都是固定的长度，比如图像识别中图像的大小。但是对于机器翻译或者语音对话而言，由于输入的序列文本大小可变，预测输出也是可变的，因而这种单一的格式很难适应。因此提出了seq2seq模型，这是一种编解码架构模型（encoder-decoder）

大体原理个人解释：
![enter description here](http://osiy4s0ad.bkt.clouddn.com/soundblog/1538049198313.png)

* 对于输入序列，假设序列长度统一为10；如：0123456789
* 第一步encoder，当我们对其使用encoder时，设其循环网络单元个数为50个，输出即50个状态，这些都将作为解码器decoder的隐藏状态输入，即解码器的先验知识。
* 第二步解码，解码部分的输入由两部分组成：encoder的隐藏状态输出state和解码器的输入。注意这里的解码器输入由于使用了目标序列——即最终结果序列。所以训练时的解码输入和预测时的有所不同。
	* 训练时，为了更快的训练，也为了防止误差，使用了目标序列作为先验条件。
	![训练时](http://osiy4s0ad.bkt.clouddn.com/soundblog/1538052679606.png)
	* 预测时，由于我们并不知道目标序列，所以这里的输入使用前一刻的输出作为输入，初始化为一个固定值。
	![预测时](http://osiy4s0ad.bkt.clouddn.com/soundblog/1538053530635.png)


----------


代码来源：[https://github.com/NELSONZHAO/zhihu/tree/master/basic_seq2seq?1521452873816](https://github.com/NELSONZHAO/zhihu/tree/master/basic_seq2seq?1521452873816)

详细步骤：

``` python
"""
@author: zoutai
@file: seq2seq.py 
@time: 2018/09/25 
@description: 基本seq2seq系统
"""

from distutils.version import LooseVersion
import tensorflow as tf
from tensorflow.python.layers.core import Dense
import os


# Check TensorFlow Version
assert LooseVersion(tf.__version__) >= LooseVersion('1.1'), 'Please use TensorFlow version 1.1 or newer'
print('TensorFlow Version: {}'.format(tf.__version__))


import numpy as np
import tensorflow as tf

# 读取数据
with open('data/letters_source.txt', 'r', encoding='utf-8') as f:
    source_data = f.read()

with open('data/letters_target.txt', 'r', encoding='utf-8') as f:
    target_data = f.read()

# 预览数据
print(source_data.split('\n')[:10],target_data.split('\n')[:10])


# 数据预处理
def extract_character_vocab(data):
    '''
    构造映射表：
    PAD用于填充
    UNK表示未知单词，比如你超过 VOCAB_SIZE范围的则认为未知单词，
    GO表示decoder开始的符号，
    EOS表示回答结束的符号，
    '''
    special_words = ['<PAD>', '<UNK>', '<GO>',  '<EOS>']

    set_words = list(set([character for line in data.split('\n') for character in line]))
    # 这里要把四个特殊字符添加进词典
    int_to_vocab = {idx: word for idx, word in enumerate(special_words + set_words)}
    vocab_to_int = {word: idx for idx, word in int_to_vocab.items()}

    return int_to_vocab, vocab_to_int


# 构造映射表
source_int_to_letter, source_letter_to_int = extract_character_vocab(source_data)
target_int_to_letter, target_letter_to_int = extract_character_vocab(target_data)

# 对字母进行转换：
'''
dict.get(key, default=None)
参数：
    key - - 字典中要查找的键。
    default - - 如果指定键的值不存在时，返回该默认值值。
'''
source_int = [[source_letter_to_int.get(letter, source_letter_to_int['<UNK>'])
               for letter in line] for line in source_data.split('\n')]
target_int = [[target_letter_to_int.get(letter, target_letter_to_int['<UNK>'])
               for letter in line] + [target_letter_to_int['<EOS>']] for line in target_data.split('\n')]

print(source_int[:10],target_int[:10])

## 构建模型
# 1.输入层
def get_inputs():
    '''
    模型输入tensor
    '''
    inputs = tf.placeholder(tf.int32, [None, None], name='inputs')
    targets = tf.placeholder(tf.int32, [None, None], name='targets')
    learning_rate = tf.placeholder(tf.float32, name='learning_rate')

    # 定义target序列最大长度（之后target_sequence_length和source_sequence_length会作为feed_dict的参数）
    target_sequence_length = tf.placeholder(tf.int32, (None,), name='target_sequence_length')
    max_target_sequence_length = tf.reduce_max(target_sequence_length, name='max_target_len')
    source_sequence_length = tf.placeholder(tf.int32, (None,), name='source_sequence_length')

    return inputs, targets, learning_rate, target_sequence_length, max_target_sequence_length, source_sequence_length



# 2.Encoder编码部分
# 在Encoder端，我们需要进行两步，第一步要对我们的输入进行Embedding，再把Embedding以后的向量传给RNN进行处理。
# 在Embedding中，我们使用[tf.contrib.layers.embed_sequence](https://www.tensorflow.org/api_docs/python/tf/contrib/layers/embed_sequence)，
# 它会对每个batch执行embedding操作。
def get_encoder_layer(input_data, rnn_size, num_layers,
                      source_sequence_length, source_vocab_size,
                      encoding_embedding_size):
    '''
    构造Encoder层

    参数说明：
    - input_data: 输入tensor
    - rnn_size: rnn隐层结点数量
    - num_layers: 堆叠的rnn cell数量
    - source_sequence_length: 源数据的序列长度
    - source_vocab_size: 源数据的词典大小
    - encoding_embedding_size: embedding的大小
    '''
    # Encoder embedding
    # returns: [batch_size, doc_length, embed_dim] - [128,7,15]/[?,?,15]
    encoder_embed_input = tf.contrib.layers.embed_sequence(input_data, source_vocab_size, encoding_embedding_size)

    # RNN cell
    def get_lstm_cell(rnn_size):
        lstm_cell = tf.contrib.rnn.LSTMCell(rnn_size, initializer=tf.random_uniform_initializer(-0.1, 0.1, seed=2))
        return lstm_cell

    cell = tf.contrib.rnn.MultiRNNCell([get_lstm_cell(rnn_size) for _ in range(num_layers)])

    # encoder_output = [batch_size, max_time, cell.output_size] - [128,256,50] 相当于特征扩大
    # encoder_state = [batch_size, cell.state_size] - [128,50]
    # 参考：https://blog.csdn.net/u010223750/article/details/71079036
    # [?,?,50]-[?,50]
    encoder_output, encoder_state = tf.nn.dynamic_rnn(cell, encoder_embed_input,
                                                      sequence_length=source_sequence_length, dtype=tf.float32)

    return encoder_output, encoder_state


## 3.Decoder 解码器输入
# 对target数据进行预处理
def process_decoder_input(data, vocab_to_int, batch_size):
    '''
    补充<GO>，并移除最后一个字符
    '''
    # cut掉最后一个字符
    # 参考：https://blog.csdn.net/banana1006034246/article/details/75092388
    # 假设data=[128,256] -> [128,255]
    ending = tf.strided_slice(data, [0, 0], [batch_size, -1], [1, 1])
    # [128,256] = [128,1] + [128,255]
    decoder_input = tf.concat([tf.fill([batch_size, 1], vocab_to_int['<GO>']), ending], 1)

    return decoder_input

## 4.解码层
# 对数据进行embedding
# 同样地，我们还需要对target数据进行embedding，使得它们能够传入Decoder中的RNN。
# Dense的说明在https://github.com/tensorflow/tensorflow/blob/master/tensorflow/python/layers/core.py
def decoding_layer(target_letter_to_int, decoding_embedding_size, num_layers, rnn_size,
                   target_sequence_length, max_target_sequence_length, encoder_state, decoder_input):
    '''
    构造Decoder层

    参数：
    - target_letter_to_int: target数据的映射表
    - decoding_embedding_size: embed向量大小
    - num_layers: 堆叠的RNN单元数量
    - rnn_size: RNN单元的隐层结点数量
    - target_sequence_length: target数据序列长度
    - max_target_sequence_length: target数据序列最大长度
    - encoder_state: encoder端编码的状态向量
    - decoder_input: decoder端输入
    '''
    # 1. Embedding
    target_vocab_size = len(target_letter_to_int)
    # 假设target_vocab_size=30,字母种类26+4=30
    # 生成随机嵌入矩阵[30,15]
    decoder_embeddings = tf.Variable(tf.random_uniform([target_vocab_size, decoding_embedding_size]))
    # 得到映射嵌入[128,?,15]/[128,length,15] = lookup([30,15],[128,legth])
    decoder_embed_input = tf.nn.embedding_lookup(decoder_embeddings, decoder_input)

    # 2. 构造Decoder中的RNN单元
    def get_decoder_cell(rnn_size):
        # rnn_size = 50
        decoder_cell = tf.contrib.rnn.LSTMCell(rnn_size,
                                               initializer=tf.random_uniform_initializer(-0.1, 0.1, seed=2))
        return decoder_cell

    # 多层RNN
    cell = tf.contrib.rnn.MultiRNNCell([get_decoder_cell(rnn_size) for _ in range(num_layers)])

    # 3. Output全连接层
    # 细胞数为target_vocab_size=30
    output_layer = Dense(target_vocab_size,
                         kernel_initializer=tf.truncated_normal_initializer(mean=0.0, stddev=0.1))

    # 4. Training decoder
    with tf.variable_scope("decode"):
        # 得到help对象
        training_helper = tf.contrib.seq2seq.TrainingHelper(inputs=decoder_embed_input,
                                                            sequence_length=target_sequence_length,
                                                            time_major=False)
        # 构造decoder
        training_decoder = tf.contrib.seq2seq.BasicDecoder(cell,
                                                           training_helper,
                                                           encoder_state,
                                                           output_layer)
        # returns: (final_outputs, final_state, final_sequence_lengths)
        # final_outputs是一个namedtuple，里面包含两项(rnn_outputs, sample_id)
        # rnn_output: [batch_size, decoder_targets_length, vocab_size]，保存decode每个时刻每个单词的概率矩阵，可以用来计算loss
        # sample_id: [batch_size], tf.int32，保存最终的编码结果。可以表示最后的答案（最可能的路径）
        # decoder_targets_length 即真是单词的长度，比如cmmoon为6；30代表每个细胞返回的概率
        # [128,?,30]-[128,?]
        training_decoder_output, _ , _ = tf.contrib.seq2seq.dynamic_decode(training_decoder,
                                                                       impute_finished=True,
                                                                       maximum_iterations=max_target_sequence_length)
    # 5. Predicting decoder
    # 与training共享参数
    with tf.variable_scope("decode", reuse=True):
        # 创建一个常量tensor并复制为batch_size的大小
        start_tokens = tf.tile(tf.constant([target_letter_to_int['<GO>']], dtype=tf.int32), [batch_size],
                               name='start_tokens')
        predicting_helper = tf.contrib.seq2seq.GreedyEmbeddingHelper(decoder_embeddings,
                                                                     start_tokens,
                                                                     target_letter_to_int['<EOS>'])
        predicting_decoder = tf.contrib.seq2seq.BasicDecoder(cell,
                                                             predicting_helper,
                                                             encoder_state,
                                                             output_layer)
        predicting_decoder_output, _, _ = tf.contrib.seq2seq.dynamic_decode(predicting_decoder,
                                                                         impute_finished=True,
                                                                         maximum_iterations=max_target_sequence_length)

    return training_decoder_output, predicting_decoder_output

# Seq2Seq
# 上面已经构建完成Encoder和Decoder，下面将这两部分连接起来，构建seq2seq模型
def seq2seq_model(input_data, targets, lr, target_sequence_length,
                  max_target_sequence_length, source_sequence_length,
                  source_vocab_size, target_vocab_size,
                  encoder_embedding_size, decoder_embedding_size,
                  rnn_size, num_layers):
    # 2.获取encoder的状态输出
    # [128,50]/[?,50] 128批，每个单词每个cell输出一个节点，共50个cell
    _, encoder_state = get_encoder_layer(input_data,
                                         rnn_size,
                                         num_layers,
                                         source_sequence_length,
                                         source_vocab_size,
                                         encoding_embedding_size)

    # 3.预处理后的decoder输入
    # [128,?]/[128,7]
    decoder_input = process_decoder_input(targets, target_letter_to_int, batch_size)

    # 4.将状态向量与输入传递给decoder
    training_decoder_output, predicting_decoder_output = decoding_layer(target_letter_to_int,
                                                                        decoding_embedding_size,
                                                                        num_layers,
                                                                        rnn_size,
                                                                        target_sequence_length,
                                                                        max_target_sequence_length,
                                                                        encoder_state,
                                                                        decoder_input)

    return training_decoder_output, predicting_decoder_output


# 超参数
# Number of Epochs
epochs = 60
# Batch Size
batch_size = 128
# RNN Size
rnn_size = 50
# Number of Layers
num_layers = 2
# Embedding Size
encoding_embedding_size = 15
decoding_embedding_size = 15
# Learning Rate
learning_rate = 0.001
save_path = '/home/zoutai/code/Exercise/tensorflow-exercises/basic_seq2seq/checkpoints'

# 构造graph
train_graph = tf.Graph()


## 定义模型运行结构
with train_graph.as_default():
    # 1.获得模型输入
    input_data, targets, lr, target_sequence_length, max_target_sequence_length, source_sequence_length = get_inputs()

    # 234.建立模型
    # predicting_decoder_output并没有使用？
    # [128,?,30] [128,?]
    training_decoder_output, predicting_decoder_output = seq2seq_model(input_data,
                                                                       targets,
                                                                       lr,
                                                                       target_sequence_length,
                                                                       max_target_sequence_length,
                                                                       source_sequence_length,
                                                                       len(source_letter_to_int),
                                                                       len(target_letter_to_int),
                                                                       encoding_embedding_size,
                                                                       decoding_embedding_size,
                                                                       rnn_size,
                                                                       num_layers)

    # tf.identity就是为了在图上显示/获取这个值而创建的虚拟节点
    training_logits = tf.identity(training_decoder_output.rnn_output, 'logits')
    predicting_logits = tf.identity(predicting_decoder_output.sample_id, name='predictions')

    print(training_logits,predicting_logits)
    masks = tf.sequence_mask(target_sequence_length, max_target_sequence_length, dtype=tf.float32, name='masks')

    # 5.定义优化器
    with tf.name_scope("optimization"):
        # Loss function
        # logits：尺寸[batch_size, sequence_length, num_decoder_symbols]
        # targets：尺寸[batch_size, sequence_length]，不用做one_hot。
        # weights：[batch_size, sequence_length]，即mask，滤去padding的loss计算，使loss计算更准确。
        # 注意这里的logits是概率矩阵，所以不需要维数相同，sequence_loss会自动计算误差值

        # 计算加权交叉熵损失
        # 交叉熵原理参考：https://blog.csdn.net/chaipp0607/article/details/73392175
        cost = tf.contrib.seq2seq.sequence_loss(
            training_logits,
            targets,
            masks)

        # Optimizer
        optimizer = tf.train.AdamOptimizer(lr)

        # Gradient Clipping 梯度裁剪：约束梯度值，防止梯度爆炸/跨度过大

        # 1.计算梯度
        # Returns: A list of (gradient, variable)
        # 参考：https://blog.csdn.net/jetFlow/article/details/80161354
        gradients = optimizer.compute_gradients(cost)
        capped_gradients = [(tf.clip_by_value(grad, -5., 5.), var) for grad, var in gradients if grad is not None]

        # 2.调用apply_gradients更新网络
        # 将梯度应用在变量上，返回此操作。即：使用该梯度进行权重更新
        train_op = optimizer.apply_gradients(capped_gradients)

## Batches
def pad_sentence_batch(sentence_batch, pad_int):
    '''
    对batch中的序列进行补全，保证batch中的每行都有相同的sequence_length

    参数：
    - sentence batch
    - pad_int: <PAD>对应索引号
    '''
    max_sentence = max([len(sentence) for sentence in sentence_batch])
    return [sentence + [pad_int] * (max_sentence - len(sentence)) for sentence in sentence_batch]


def get_batches(targets, sources, batch_size, source_pad_int, target_pad_int):
    '''
    定义生成器，用来获取batch，每次获取一批数据
    '''
    for batch_i in range(0, len(sources) // batch_size):
        start_i = batch_i * batch_size
        sources_batch = sources[start_i:start_i + batch_size]
        targets_batch = targets[start_i:start_i + batch_size]
        # 补全序列
        pad_sources_batch = np.array(pad_sentence_batch(sources_batch, source_pad_int))
        pad_targets_batch = np.array(pad_sentence_batch(targets_batch, target_pad_int))

        # 记录每条记录的长度
        targets_lengths = []
        for target in targets_batch:
            targets_lengths.append(len(target))

        source_lengths = []
        for source in sources_batch:
            source_lengths.append(len(source))
        # 这里使用yield代替return从而节约内存使用，因为yield使用生成模式，即迭代模式，相当于一个迭代器iterator
        # 每次返回下一个元素，而不是等所有元素生成完一起返回，大大降低内存使用
        # 参考：https://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/index.html
        yield pad_targets_batch, pad_sources_batch, targets_lengths, source_lengths


## 训练
def train():
    ## Train
    # 将数据集分割为train和validation；第一批作为验证，后面的作为训练
    train_source = source_int[batch_size:]
    train_target = target_int[batch_size:]
    # 留出一个batch进行验证
    valid_source = source_int[:batch_size]
    valid_target = target_int[:batch_size]
    # next函数参考：http://www.runoob.com/python/python-func-next.html
    (valid_targets_batch, valid_sources_batch, valid_targets_lengths, valid_sources_lengths) = next(
        get_batches(valid_target, valid_source, batch_size,
                    source_letter_to_int['<PAD>'],
                    target_letter_to_int['<PAD>']))

    display_step = 50  # 每隔50轮输出loss

    checkpoint = "trained_model.ckpt"

    # 添加使用GPU
    os.environ["CUDA_VISIBLE_DEVICES"] = '-1'  # use GPU with ID=0 ;use CPU with ID = -1
    config = tf.ConfigProto()
    config.gpu_options.per_process_gpu_memory_fraction = 0.98  # maximun alloc gpu98% of MEM
    config.gpu_options.allow_growth = True  # allocate dynamically

    with tf.Session(graph=train_graph,config=config) as sess:
        # 初始化全局变量
        sess.run(tf.global_variables_initializer())

        # 训练次数60
        for epoch_i in range(1, epochs + 1):
            # 分批训练10000/128-1=78.125-1=77
            for batch_i, (targets_batch, sources_batch, targets_lengths, sources_lengths) in enumerate(
                    get_batches(train_target, train_source, batch_size,
                                source_letter_to_int['<PAD>'],
                                target_letter_to_int['<PAD>'])):

                t_logits = sess.run(
                    [training_logits,targets],
                    {input_data: sources_batch,
                     targets: targets_batch,
                     lr: learning_rate,
                     target_sequence_length: targets_lengths,
                     source_sequence_length: sources_lengths})
                print(t_logits,targets)

                # run(
                #     fetches, 元组：一些用于计算结果的张量，用于去除计算结果tensor （比如我们要求姐y=w*x中的w，这里的fetches相当于y）
                #     feed_dict=None, 字典：输入变量占位符 （这里的feed_dict相当于x）
                #     options=None,
                #     run_metadata=None
                # )
                # 参考：http://wiki.jikexueyuan.com/project/tensorflow-zh/get_started/basic_usage.html
                # cost计算误差，train_op通过cost来更新权重
                _, loss = sess.run(
                    [train_op, cost],
                    {input_data: sources_batch,
                     targets: targets_batch,
                     lr: learning_rate,
                     target_sequence_length: targets_lengths,
                     source_sequence_length: sources_lengths})

                # 共77批
                if batch_i % display_step == 0:
                    # 计算validation loss
                    # 验证时不需要更新权重
                    validation_loss = sess.run(
                        [cost],
                        {input_data: valid_sources_batch,
                         targets: valid_targets_batch,
                         lr: learning_rate,
                         target_sequence_length: valid_targets_lengths,
                         source_sequence_length: valid_sources_lengths})

                    print('Epoch {:>3}/{} Batch {:>4}/{} - Training Loss: {:>6.3f}  - Validation loss: {:>6.3f}'
                          .format(epoch_i,
                                  epochs,
                                  batch_i,
                                  len(train_source) // batch_size,
                                  loss,
                                  validation_loss[0]))

        # 保存模型
        saver = tf.train.Saver()
        saver.save(sess, os.path.join(save_path, checkpoint))
        print('Model Trained and Saved')

## 预测
def source_to_seq(text):
    '''
    对源数据进行转换
    '''
    sequence_length = 7
    return [source_letter_to_int.get(word, source_letter_to_int['<UNK>']) for word in text] + [
        source_letter_to_int['<PAD>']] * (sequence_length - len(text))

def predict():
    os.environ["CUDA_VISIBLE_DEVICES"] = "-1"
    # 输入一个单词
    input_word = 'common'
    text = source_to_seq(input_word)

    checkpoint = "/trained_model.ckpt"

    loaded_graph = tf.Graph()

    with tf.Session(graph=loaded_graph) as sess:
        # 加载模型
        modelPath =save_path+checkpoint
        print(modelPath)
        loader = tf.train.import_meta_graph(modelPath + '.meta')
        loader.restore(sess, modelPath)
        input_data = loaded_graph.get_tensor_by_name('inputs:0')
        logits = loaded_graph.get_tensor_by_name('predictions:0')
        source_sequence_length = loaded_graph.get_tensor_by_name('source_sequence_length:0')
        target_sequence_length = loaded_graph.get_tensor_by_name('target_sequence_length:0')

        answer_logits = sess.run(logits, {input_data: [text] * batch_size,
                                          target_sequence_length: [len(input_word)] * batch_size,
                                          source_sequence_length: [len(input_word)] * batch_size})[0]

    pad = source_letter_to_int["<PAD>"]

    print('原始输入:', input_word)

    print('\nSource')
    print('  Word 编号:    {}'.format([i for i in text]))
    print('  Input Words: {}'.format(" ".join([source_int_to_letter[i] for i in text])))

    print('\nTarget')
    print('  Word 编号:       {}'.format([i for i in answer_logits if i != pad]))
    print('  Response Words: {}'.format(" ".join([target_int_to_letter[i] for i in answer_logits if i != pad])))

if __name__ == '__main__':
    # train or predict
    flag = 'train'
    if flag == 'train':
        train()
    else:
        predict()
```