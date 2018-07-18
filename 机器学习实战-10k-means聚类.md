---
title: 机器学习实战-10k-means聚类
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>一句话总结，即将数据集分为k个类别。原理：1、首先选取k个初始点作为中心；然后遍历所有的点，对于每一个点，计算k个中心中到这个点距离最小的中心，然后将这个点划分到这个中心；3、当所有的点划分完毕，重新计算类别的计算中心（将均值作为中心），直到中心点不再变化。但是这种方式可能陷入局部最小解，为了克服这个缺陷，提出了二分k-means算法，这种方式类似于二分法。即先划分两类，然后，再将其中一类划分为两类，这样就扩展到三类，一次类推。对于到底选取哪一类进行扩张，衡量刻度是到质心的总距离，选取最小的那一个。


----------


一、应用场景
这是一种无监督的学习，一般用来划分，实际生活中常常遇到，比如村落的划分，经常是把村落聚集的一个划分为一个村落。同样比如美国的大选，对于某一些地区的选民，很显然有一部分选民很大程度上是一致的态度，如果能找到他们这一个群里，汇集起来，就可以专门针对，来进行宣传。同样，对于推荐系统等，找到某一类共同特制的人群，也可以进行定点推销。


----------

二、聚类算法
由于原理简单，只是动手做了原理部分，没有亲自实现案列部分。
![第二部分实验图-书上未给代码][1]
代码：kMeans.py

``` stylus
""" 
@author: zoutai
@file: kMeans.py 
@time: 2017/11/20 
@description: K-均值聚类
"""

from numpy import *

def loadDataSet(fileName):      #general function to parse tab -delimited floats
    dataMat = []                #assume last column is target value
    fr = open(fileName)
    for line in fr.readlines():
        curLine = line.strip().split('\t')
        fltLine = list(map(float,curLine)) #map all elements to float()
        dataMat.append(fltLine)
    return dataMat

# 距离计算函数：欧氏距离
def distEclud(vecA,vecB):
    return sqrt(sum(power(vecA-vecB,2)))

def randCent(dataSet,k):
    n = shape(dataSet)[1]
    centroids = mat(zeros((k,n)))
    for j in range(n):
        minJ = min(dataSet[:,j])
        rangJ = float(max(dataSet[:,j])-minJ)
        centroids[:,j] = minJ + rangJ * random.rand(k,1) # 随机数范围0-1，矩阵为k*1
    return centroids

#
def kMeans(dataSet,k,distMeas=distEclud,createCent=randCent):
    m = shape(dataSet)[0]
    clusterAssment = mat(zeros((m,2)))
    centroids = createCent(dataSet,k)
    clusterChanged = True
    while clusterChanged:
        clusterChanged = False
        for i in range(m):
            minDist = inf;minIndex = -1
            for j in range(k):
                distJI = distEclud(centroids[j,:],dataSet[i,:])
                if distJI < minDist:
                    minDist = distJI
                    minIndex = j
            # 是否更新质心，重新扫描。（即如果全部的点最后都不改变了，则停止学习）
            if clusterAssment[i,0] != minIndex:
                clusterChanged = True
            clusterAssment[i,:] = minIndex,minDist**2 # 记录点所属的簇
        print(centroids)
        # 更新质心
        for cent in range(k):
            ptsInClust = dataSet[nonzero(clusterAssment[:,0]==cent)[0]] # 第cent簇的点集合
            centroids[cent,:] = mean(ptsInClust,axis=0) #取均值作为质心
    return centroids,clusterAssment

def biKmeans(dataSet,k,disMeas=distEclud):
    m = shape(dataSet)[0]
    clusterAssment = mat(zeros((m,2)))
    # 1，先求出所有点的质心
    centroid0 = mean(dataSet,axis=0).tolist()[0]
    centList = [centroid0] # 保存所有的质心

    # 遍历，求出每个点到质心的距离^2
    for j in range(m):
        clusterAssment[j,1] = distEclud(mat(centroid0),dataSet[j,:])**2 # ^2是错误的

    while len(centList) < k:
        lowestSSE = inf

        # 2,接下来，对已经划分完成的，尝试将每一个k类二次划分，找出最好的，就相当于增加了一个类
        for i in range(len(centList)):
            ptsInCurrCluster = dataSet[nonzero(clusterAssment[:,0].A==i)[0],:] # 生成所有第i类的数据集datasetI
            # 不知道哪里逻辑错了，这里需要判断空
            if len(ptsInCurrCluster)==0:
                continue
            centroidIMat,splitClustAss = kMeans(ptsInCurrCluster,2,disMeas) # 求出新的子划分

            # 计算新的总误差大小：sum = 1第I个新的子划分+2（原始的划分-第I个划分）
            # 1.第I个新的子划分误差
            firstSplit = sum(splitClustAss[:,1])
            secondSplit = sum(clusterAssment[nonzero(clusterAssment[:,0].A != i)[0],1])
            print("firstSplit is :",firstSplit,"secondSplit is ；",secondSplit)

            # 如果小于原始误差，则划分保存
            if firstSplit + secondSplit < lowestSSE:
                bestCentToSplit = i
                bestNewCents = centroidIMat
                bestClustAss = splitClustAss.copy()
                lowestSSE = firstSplit+secondSplit

        # 更新分配结果，将上面得到的最好结果保存:
        # 将新划分的两类划分给原始集合：第一类划分下标为被分解的簇bestCentToSplit，第二类，为新的簇，即len(centList)
        bestClustAss[nonzero(bestClustAss[:,0].A == 0)[0],0] = bestCentToSplit
        bestClustAss[nonzero(bestClustAss[:,0].A == 1)[0],0] = len(centList)

        print("the bestCentToSplit is",bestCentToSplit)

        # 将第bestCentToSplit个修改；增加另外一个
        centList[bestCentToSplit] = bestNewCents[0,:]
        centList.append(bestNewCents[1,:])
        # 赋均值
        clusterAssment[nonzero(clusterAssment[:,0].A == bestCentToSplit)[0],:] = bestClustAss
    return centList,clusterAssment


import urllib
import json


def geoGrab(stAddress, city):
    apiStem = 'http://where.yahooapis.com/geocode?'  # create a dict and constants for the goecoder
    params = {}
    params['flags'] = 'J'  # JSON return type
    params['appid'] = 'aaa0VN6k'
    params['location'] = '%s %s' % (stAddress, city)
    url_params = urllib.urlencode(params)
    yahooApi = apiStem + url_params  # print url_params
    print(yahooApi)
    c = urllib.urlopen(yahooApi)
    return json.loads(c.read())


from time import sleep


def massPlaceFind(fileName):
    fw = open('places.txt', 'w')
    for line in open(fileName).readlines():
        line = line.strip()
        lineArr = line.split('\t')
        retDict = geoGrab(lineArr[1], lineArr[2])
        if retDict['ResultSet']['Error'] == 0:
            lat = float(retDict['ResultSet']['Results'][0]['latitude'])
            lng = float(retDict['ResultSet']['Results'][0]['longitude'])
            print(lineArr[0], lat, lng)
            fw.write(line,'\t', lat,'\t', lng,'\n')
        else:
            print("error fetching")
        sleep(1)
    fw.close()


def distSLC(vecA, vecB):  # Spherical Law of Cosines
    a = sin(vecA[0, 1] * pi / 180) * sin(vecB[0, 1] * pi / 180)
    b = cos(vecA[0, 1] * pi / 180) * cos(vecB[0, 1] * pi / 180) * \
        cos(pi * (vecB[0, 0] - vecA[0, 0]) / 180)
    return arccos(a + b) * 6371.0  # pi is imported with numpy


import matplotlib
import matplotlib.pyplot as plt


def clusterClubs(numClust=5):
    numClust = numClust+1 # 不知道哪里逻辑错了，必须+1，最后会有一个空集
    datList = []
    for line in open('places.txt').readlines():
        lineArr = line.split('\t')
        datList.append([float(lineArr[4]), float(lineArr[3])])
    datMat = mat(datList)
    myCentroids, clustAssing = biKmeans(datMat, numClust, disMeas=distSLC)
    fig = plt.figure()
    rect = [0.1, 0.1, 0.8, 0.8]
    scatterMarkers = ['s', 'o', '^', '8', 'p', \
                      'd', 'v', 'h', '>', '<']
    axprops = dict(xticks=[], yticks=[])
    ax0 = fig.add_axes(rect, label='ax0', **axprops)
    imgP = plt.imread('Portland.png')
    ax0.imshow(imgP)
    ax1 = fig.add_axes(rect, label='ax1', frameon=False)
    for i in range(numClust):
        ptsInCurrCluster = datMat[nonzero(clustAssing[:, 0].A == i)[0], :]
        markerStyle = scatterMarkers[i % len(scatterMarkers)]
        ax1.scatter(ptsInCurrCluster[:, 0].flatten().A[0], ptsInCurrCluster[:, 1].flatten().A[0], marker=markerStyle,
                    s=90)


    newDataSet = zeros((len(myCentroids),2))
    newDataSet = [list(array(myCentroids[i])[0]) for i in range(len(myCentroids))]
    ax1.scatter(mat(newDataSet)[:, 0].flatten().A[0], mat(newDataSet)[:, 1].flatten().A[0], marker='+', s=300)
    plt.show()

```

画图代码：plotTest.py

``` stylus
""" 
@author: zoutai
@file: plotTest.py 
@time: 2017/11/22 
@description: 
"""

# 画这个图费了好长时间，主要的问题还是在于各种数据类型的转换和匹配，真是想说python真是个垃圾语言，对初学者很难适应

import matplotlib.pyplot as plt
from numpy import *
def plotGraph(dataSet,centList):
    fig = plt.figure()
    ax = fig.add_subplot(111)

    newDataSet = zeros((len(centList),2))
    newDataSet = [list(array(centList[i])[0]) for i in range(len(centList))]
    # for i in range(len(centList)):
    #     newDataSet.(array(centList[i])[0])
    newMat = mat(newDataSet)
    print(newMat)
    # ax.plot(dataSet[:,0],dataSet[:,1],marker = 'x')
    # ax.plot(centList[:,0],centList[:,1],marker = '+',color='r')
    ax.scatter(dataSet[:,0].flatten().A[0],dataSet[:,1].flatten().A[0],marker='^',s=90)
    ax.scatter(newMat[:,0].flatten().A[0],newMat[:,1].flatten().A[0],marker='o',s=90)
    plt.show()



```

测试代码：test.py

``` stylus
""" 
@author: zoutai
@file: test.py 
@time: 2017/11/21 
@description: 
"""
from unit10.kMeans import *
from unit10.plotTest import *
from numpy import *

# test1
dataMat = mat(loadDataSet('testSet.txt'))
myCentroids,clustAssing = kMeans(dataMat,4)
print(myCentroids)

# test2
# dataMat3 = mat(loadDataSet('testSet2.txt'))
# centList,newAssment = biKmeans(dataMat3,3)
# print(centList)
# plotGraph(dataMat3,centList)
# # dataLast = row_stack(dataMat3,mat(centList))
# print(centList)


# test3
clusterClubs(5)

```


  [1]: ./images/Figure_1-2.png "Figure_1-2"
