---
title: Tensorflow实战-8循环神经网络RNN
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>回顾之前学习的神经网络，无论是单层感知机，还是卷积神经网络亦或是深度神经网络，其实都有一个缺点：即预测值只和当前的输入有关或者说就是通过当前的值来预测状态结果，而不需要之前的值。但是对于时间等序列预测，比如天气预报，明天是否下雨不仅仅与今天的状态有关，还可能与之前几天甚至几十天的状态有关，因此这里就需要使用循环神经网络。卷积神经网络是在神经网络上加了一层卷积层，同样的道理，为了使用之前的序列，也需要加上一个输入（这里并不是层），即RNN的输入=（当前的值+之前的值），所以这样看来，之前的神经网络其实是“一维”数据输入（这里的一维是指影响效果只有一个），而RNN加上了之前的序列，但是之前的值又不能与当前的值混合输入，否则影响效果是相同的，所以就将之前的值传播方向定为由左向右，当前的值定为由下往上，相当于X和Y轴，这里是比喻，这样考虑的话，其实如果还有其他因素考虑，其实还可以再加维数，可能称作为多维循环神经网络。

**一、原理分析**

* 细看循环神经网络会发现并不是循环的，当时这个循环的词是从哪里来的？
	* 为了探究RNN的发现，我们可以联想之前学习过的模电，模电中的三极管是两个输入决定输入，在复杂的电路中，输出的电流可能有返回来影响当前的电流输入，所以，循环神经网络是这样的一个状态，如下图左边。这就相当于一个循环传递的效果。但是出于优化的考虑，目前的神经网络并不是这样循环的，却是图中右边所示的前向传播过程。
	![其中，t 是时刻， x 是输入层， s 是隐藏层， o 是输出层，矩阵 W 就是隐藏层上一次的值作为这一次的输入的权重。][1]

1. 简单RNN

	* 简单的RNN如上图所示，左边输入的是之前的序列状态，下边输入的是下一时刻的时间；上面输出的为下一时刻的预测状态，右边数出的是接下来的序列状态；这样就类似于一个滑动窗口（之前的序列）不停地向右滑动得到新的窗口和预测值，再利用新的窗口来计算下一个节点。
	* 最近在做单通道语音分离这块，其中讲到了短时傅里叶变换STFT，即假设声音信号在极短的时间内频率基本上是稳定的，所以就可以使用一个滑动窗口来移动分析每一个窗口。这里面有点类似可能，虽然远离差得很远。即之前的序列也是一种滑动窗口，我们认为在这个窗口类，序列遵循一定的规律，比如等差为2的序列，所以之后的序列就可以使用这个规律来求解，找到新的规律，又可以使用新的规律预测下一个的序列。（一点想法）

2. 长短时记忆（LSTM）

	* 对于简单地RNN有个缺陷，即我们如何确定是前几天的序列对之后有影响？而且，对于之前的序列，并不是所有的值都有作用。所以这里有了长短时记忆LTSM
	* 长短是记忆如下图所示，它通过输入们和遗忘门来决定哪些信息需要记忆，那些信息需要丢弃，这样，就可以得到最重要的信息。

**二、几种变种RNN**

1. 双向神经网络
	* 当前时刻的输出不仅与之前的状态有关而且还和之后的状态有关，比如英语中的完形填空，当前的空格词语，不仅仅和之前的语境有关还需要后面的语境信息

2. 深层神经网络 
	* 观察之前的神经网络，会发现在纵向上是单层的，为了增强模型的表达能力，可以将纵向上进行多层叠加

3. 补充
	* RNN只会在同一时刻t的不同层之间进行dropout，在不停地t之间不会dropout。
----------


>以下为两个实战例子：自然语言建模预测和sin函数预测

**三、自然语言建模**
我们要用 RNN 做这样一件事情，每输入一个词，循环神经网络就输出截止到目前为止，下一个最可能的词，如下图所示：
![enter description here][2]

首先，要把词表达为向量的形式：

建立一个包含所有词的词典，每个词在词典里面有一个唯一的编号。
任意一个词都可以用一个N维的one-hot向量来表示。
![enter description here][3]
这种向量化方法，我们就得到了一个高维、稀疏的向量，这之后需要使用一些降维方法，将高维的稀疏向量转变为低维的稠密向量。

为了输出 “最可能” 的词，所以需要计算词典中每个词是当前词的下一个词的概率，再选择概率最大的那一个。

因此，神经网络的输出向量也是一个 N 维向量，向量中的每个元素对应着词典中相应的词是下一个词的概率：
![enter description here][4]

为了让神经网络输出概率，就要用到 softmax 层作为输出层。

softmax函数的定义：
因为和概率的特征是一样的，所以可以把它们看做是概率。
![enter description here][5]
例：
![enter description here][6]

计算过程为：
![enter description here][7]
含义就是：
模型预测下一个词是词典中第一个词的概率是 0.03，是词典中第二个词的概率是 0.09。
语言模型如何训练？

把语料转换成语言模型的训练数据集，即对输入 x 和标签 y 进行向量化，y 也是一个 one-hot 向量


接下来，对概率进行建模，一般用交叉熵误差函数作为优化目标。

交叉熵误差函数，其定义如下：
![enter description here][8]

用上面例子就是：
![enter description here][9]
计算过程如下：

![enter description here][10]

有了模型，优化目标，梯度表达式，就可以用梯度下降算法进行训练了。


----------


**四、时间序列预测-sin函数预测**

* 通过10000个点来预测1000个点，其实这里实验不应该用这么多的值，已经失去了预测的意义。
sinPredict.py

``` stylus
""" 
@author: zoutai
@file: sinPredict.py
@time: 2017/12/13 
@description: 预测正弦函数
"""
import numpy as np
import tensorflow as tf
from tensorflow.contrib import rnn
# 加载matplotlib工具包画图
import matplotlib as mpl
from tensorflow.contrib.learn.python.learn.estimators.estimator import SKCompat
mpl.use('Agg')

from matplotlib import pyplot as plt

learn = tf.contrib.learn

HIDDEN_SIZE = 30 #LSTM中的隐藏节点个数
NUM_LAYERS = 2 #LSTM的层数

TIMESTEPS = 10 #循环神经网络的截断长度
TRAINING_STEPS = 10000 # 训练次数
BATCH_SIZE = 32 # batch大小

TRAINING_EXAMPLES = 10000 # 训练数据个数
TESTING_EXAMPLES = 1000 # 测试数据个数
SAMPLE_GAP = 0.01 # 采样间隔


#
# 生成数据集
# 通过以步长为1为单位，依次取（TIMESTEPS+1）个连续序列作为训练。
# 其中TIMESTEPS是训练数据的输入状态x，1位输出状态y
def generate_data(seq):
    '''

    :param seq: 总的用于训练的序列点
    :return:
        arrayX: 将序列的一部分作为输入状态
        arrayY： 序列的另一部分作为输出状态
    '''
    x = []
    y = []
    for i in range(len(seq)-TIMESTEPS-1):
        x.append([seq[i:i+TIMESTEPS]])
        y.append([seq[i+TIMESTEPS]])
    return np.array(x,dtype=np.float32),np.array(y,dtype=np.float32)

def LstmCell():
    lstm_cell = rnn.BasicLSTMCell(HIDDEN_SIZE,state_is_tuple=True)
    return lstm_cell

# 定义lstm模型
def lstm_model(X, y):
    cell = rnn.MultiRNNCell([LstmCell() for _ in range(NUM_LAYERS)])
    output, _ = tf.nn.dynamic_rnn(cell, X, dtype=tf.float32)
    output = tf.reshape(output, [-1, HIDDEN_SIZE])
    # 通过无激活函数的全连接层计算线性回归，并将数据压缩成一维数组结构
    predictions = tf.contrib.layers.fully_connected(output, 1, None)

    # 实际值和预测值
    labels = tf.reshape(y, [-1]) # 重塑为一维，即默认的一维
    predictions = tf.reshape(predictions, [-1])

    # 输出预测和损失值
    loss = tf.losses.mean_squared_error(predictions, labels)

    # 损失函数优化方法
    train_op = tf.contrib.layers.optimize_loss(loss, tf.contrib.framework.get_global_step(),
                                             optimizer="Adagrad",
                                             learning_rate=0.1)
    return predictions, loss, train_op

#建立深层模型
# Estimator里面还有一个叫SKCompat的类，如果使用x,y而不是input_fn来传参数的形式，需要用这个类包装一下：
# 第二个参数用于本地保存
regressor = SKCompat(learn.Estimator(model_fn = lstm_model, model_dir="Models/model_2"))

test_start = TRAINING_EXAMPLES * SAMPLE_GAP
test_end = (TRAINING_EXAMPLES + TESTING_EXAMPLES) * SAMPLE_GAP
train_X,train_y = generate_data(np.sin(np.linspace(0,test_start,TRAINING_EXAMPLES,dtype=np.float32)))
test_X,test_y = generate_data(np.sin(np.linspace(test_start,test_end,TESTING_EXAMPLES,dtype=np.float32)))
regressor.fit(train_X,train_y,batch_size=BATCH_SIZE,steps=TRAINING_STEPS)
predicted = [[pred] for pred in regressor.predict(test_X)]

# 计算MSE
rmse = np.sqrt(((predicted - test_y) ** 2).mean(axis=0))

fig = plt.figure()
plot_predicted, = plt.plot(predicted, label='predicted')
plot_test, = plt.plot(test_y, label='real_sin')
plt.legend([plot_predicted, plot_test],['predicted', 'real_sin'])
plt.show()
fig.savefig('sin.png')
print("Mean Square Error is:%f" % rmse[0])
```

后续补充
部分参考：[简书-不会停的蜗牛][11]
更多参考：[循环神经网络(RNN, Recurrent Neural Networks)介绍][12]
  [1]	* : ./images/1513305053264.jpg


  [1]: ./images/1513308067569.jpg
  [2]: ./images/1513308067569.jpg
  [3]: ./images/1513308079476.jpg
  [4]: ./images/1513308087280.jpg
  [5]: ./images/1513308095186.jpg
  [6]: ./images/1513308105877.jpg
  [7]: ./images/1513308145346.jpg
  [8]: ./images/1513308169472.jpg
  [9]: ./images/1513308224744.jpg
  [10]: ./images/1513308215413.jpg
  [11]: http://www.jianshu.com/p/39a99c88a565
  [12]: http://blog.csdn.net/heyongluoyao8/article/details/48636251
