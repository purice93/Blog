---
title: 机器学习实战-5Logistic回归
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
几个关键词：逻辑回归、极大似然估计、激活函数
## 参考书籍：西瓜书P54,《统计学习方法》P77

1. 回归和分类的区别
回归是一种连续变量的预测，比如函数拟合，股票线等，人的年龄。分类的特征却不是连续变化的，比如西瓜的颜色是不能连续量化的

2. 线性回归
* 举个例子，如果我们现在有两个未知数x1,x2,即特征值；一个输出变量y，即分类结果；我们需要求解一个方程`w1x1+w2x2+b = y`,来拟合这种曲线来预测之后的曲线。一般而言我们是直接去方程组的，但是，这里数据量很大，而且我们的方程是用来预测的，不可能直接求出精确的值，这样会过拟合。
* 因而，我们考虑一种近似求解，通过训练来改变w的值，来拟合直线。这里我们可以把这个函数考虑为一个感知机。误差为`error = y - (w1x1+w2x2+b)` ，对其对其进行w求导得出梯度值，通过`w = w + grad(error,w)`来更新权值
* 但是对于二分类而言，x1、x2与y的关系并不是直接就能通过这种直线进行拟合的，所以需要使用激活函数来将之间变为曲线拟合，这和感知机中激活函数的原理是一样的，另外含有支持向量机中的核函数。
* 所以我们设 `z = w1x1+w2x2+b`，`h(z) = 1/(1+exp(-z)`;将输出结果归一化到0~1之间，同时拟合曲线。

3. 以下三种情况，分别是梯度下降和随机梯度，以及学习率变化的随机梯度
4. 梯度导数推导：https://my.oschina.net/tantexian/blog/1359191，这里使用的是对数损失函数
![enter description here][1]
![enter description here][2]

代码：logRegres.py

``` stylus
import math,random
import numpy as np
import matplotlib.pyplot as plt

# 将文本数据转化为矩阵训练数据
def loadDataSet():
    dataMat = []; labelMat = []
    file = open('testSet.txt')
    for line in file.readlines():
        line = line.strip().split()
        dataMat.append([1.0,float(line[0]),float(line[1])])
        labelMat.append([float(line[2])])
    return dataMat,labelMat

# 定义sigmoid激活函数
def sigmoid(inX):
    return 1.0/(1.0+np.exp(-inX))

# 实现简单地二分类、梯度下降、最大似然估计
def gradAscent(dataMatrix,classLabels):
    dataMatrix = np.mat(dataMatrix)
    labelMatrix  = np.mat(classLabels)
    m,n = np.shape(dataMatrix)
    alpha = 0.001
    maxCycles = 500
    weights = np.ones((n,1)) # n*1
    for k in range(maxCycles):
        h = sigmoid(dataMatrix*weights)
        error = labelMatrix - h
        #这里的公式是直接推导出来的即：梯度=XT * (Y-XW)
        weights = weights + dataMatrix.transpose() * error  
    return weights

# 改进1：随机梯度下降，学习率不变
# 注意：这里实际上并不是随机，只是每次都选一个样本训练，遍历完
# 随机是指，每次多选择部分数据进行训练。
def randomGradAscent(dataMatr,classLabels):
    dataMatrix = np.array(dataMatr)
    m,n = np.shape(dataMatrix)
    alpha = 0.01
    weights = np.ones(n)
    for i in range(m):
        h = sigmoid(sum(dataMatrix[i] * weights))
        error = classLabels[i] - h
        weights = weights + alpha * error * dataMatrix[i]
    return np.array(np.mat(weights).transpose()) # 由于这里的weights是横向数组，所以需要转换为统一的竖向数组

# 改进2：随机梯度下降,学习率变化！
# 注意：通过研究上面的梯度上升效率，发现，在最初是下降时很快的，学习率（即步长）可以适当加快；
# 后期由于误差减小，学习率变化变小；即可以设置学习率随着训练的次数逐渐减小
def randomGradAscentPlus(dataMatr,classLabels,num):
    dataMatrix = np.array(dataMatr)
    m,n = np.shape(dataMatrix)
    alpha = 0.01
    weights = np.ones(n)
    for j in range(num):
        dataIndex = list(range(m))
        for i in range(m):
            alpha = 4.0/(1.0+i+j)+0.01
            randomIndex = int(random.uniform(0,len(dataIndex)))
            h = sigmoid(sum(dataMatrix[randomIndex]*weights))
            error = classLabels[randomIndex] - h
            weights = weights + alpha * error * dataMatrix[i]
            del(dataIndex[randomIndex])
    return np.array(np.mat(weights).transpose())

# 画出分类图
def plotBestFit(weightMat):
    weights = np.array(weightMat)
    dataMat,labelMat = loadDataSet()
    n = np.shape(dataMat)[0] # 样本数
    xcord1 = [];ycord1 = []
    xcord2 = [];ycord2 = []
    for i in range(n):
        # 类别为1
        if labelMat[i][0] == 1:
            xcord1.append(dataMat[i][3]) # x,y分别对应特征值（看作是坐标）
            ycord1.append(dataMat[i][2])
        else:
            xcord2.append(dataMat[i][4])
            ycord2.append(dataMat[i][2])

    fig = plt.figure() # 创建一个视图
    ax = fig.add_subplot(1,1,1) # 添加一个面板
    ax.scatter(xcord1,ycord1,s=30,c='red',marker='s') # 创建一个x vs y二维图，s-size of point，c-color,maker-形状、s-square正方形
    ax.scatter(xcord2,ycord2,s=30,c='green')
    x = np.arange(-3.0,3.0,0.1) # 坐标轴范围
    y = (-weights[0][0]-weights[1][0]*x)/weights[2][0]
    ax.plot(x,y) # 模拟直线
    plt.xlabel('特征x1');plt.ylabel('特征x2') # 坐标标题
    plt.show() # 显示

# 从疝气病症预测病马的死亡率
# 处理缺失值

# 激活函数
def classifyVector(inX,weights):
    dataMatrix = np.array(inX)
    # 这里需要再将weights转化回来
    # np.array(np.mat(weights).transpose())
    w = np.array(np.mat(weights).transpose())[0]
    prop = sigmoid(sum(dataMatrix[0]*w))
    if prop > 0.5:
        return 1
    else:
        return 0

def colicTest():
    frTrain = open('horseColicTraining.txt')
    frTest = open('horseColicTest.txt')
    trainingSet = []; trainingLabels = []
    for line in frTrain.readlines():
        currLine = line.strip().split('\t')
        lineArr = []
        for i in range(21):
            lineArr.append(float(currLine[i]))
        trainingSet.append(lineArr)
        trainingLabels.append([float(currLine[21])])
    print(trainingSet)
    trainWeights = randomGradAscentPlus(trainingSet,trainingLabels,500)

    # 测试集
    errorCount = 0;numTestVec = 0;
    for testLine in frTest.readlines():
        numTestVec = numTestVec+1.0
        currLine = testLine.strip().split('\t')
        lineArr = []
        for i in range(21):
            lineArr.append(float(currLine[i]))
        print(lineArr)
        if int(classifyVector(lineArr,trainWeights)) != int(float(currLine[21])):
            errorCount = errorCount+1
    errorRate = errorCount / numTestVec
    print("the errorRate is: %f" % errorRate)
    return errorRate

# 多次测试求平均错误率
def mutiTest():
    numTests = 10;errorSum = 0.0
    for i in range(numTests):
        errorSum = errorSum + colicTest()
    print("after %d itertions,the errorRate mean is %f" % (numTests,errorSum/numTests))


# 补充知识点：
# 1.str.strip([chars]):剥夺，脱去，即去除str中含有的char字符，
# Return a copy of the string with the leading and trailing characters removed.

# 2.str.split(sep=None, maxsplit=-1),返回一个字符串隔离的list
# Return a list of the words in the string, using sep as the delimiter string.
# \t represents a tab 空格键
# \n represents a new line 换行
# \r represents a carriage return 回车

# 3.transpose():矩阵转置

# 4.注意矩阵mat、列表list、np.array之间的区别
# 对mat进行操作，需要先将mat转化为array，再进行操作，否则会出现：x and y must have same first dimension

# 5.float和str和int之间的转换，出错会：
# TypeError: ufunc 'multiply' did not contain a loop with signature matching types dtype('S32') dtype('S32') dtype('S32')

# 6.坑是真的多，感觉是在学python而不是机器学习。
```
测试代码：logTest.py

``` stylus
# 出现一些微小的错误，没事迷茫，应该是几个函数库的不同所引起的，比如random函数，math、numpy中的都有，
# 可能有一些区别，但是书上是用python2写的，和python3有些区别

from unit05 import logRegres
import numpy as np
dataX,labelY = logRegres.loadDataSet()

# weights1 = logRegres.gradAscent(dataX,labelY)
# logRegres.plotBestFit(weights1)

# weights2 = logRegres.randomGradAscent(dataX,labelY)
# logRegres.plotBestFit(weights2)

# weights3 = logRegres.randomGradAscentPlus(dataX,labelY,150)
# logRegres.plotBestFit(weights3)

logRegres.mutiTest()
```


  [1]: http://img.blog.csdn.net/20150721231518546?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast
  [2]: http://img.blog.csdn.net/20150723231350914?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center
  [3]: http://img.blog.csdn.net/20150721231518546?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast
  [4]: http://img.blog.csdn.net/20150721231518546?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast
