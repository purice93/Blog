---
title: 机器学习实战-11关联分析Apriori算法-pytohn3
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
==感悟：这一章的理论很浅，浅到似乎只需要一个数学公式就能够表达。但是这一章却花费了我整整两天时间。有时候我在想，为什么总是有人很多人想得多做得少，理论很深却只会纸上谈兵。很长的一段时间我就是这种人，自以为智力还算可以，很多数学问题自认为很简单，看了很多书，学了很多知识，但真正向别人讲解时，却怎么也讲不清，其实最深处的原因，还是自身根本就没有熟练地理解其中的含义，有些东西并不是想当然的。其实会和熟差距是非常远的。这让我想起了高中时期，任何题目一眼飘过就能立马动笔，几乎遇不到不会做的试题。而这种情况的出现却是依靠高一高二不断地练习达到的。就像现在这样，其实道理很简单，但是我们还是站在别人的肩膀上行走，你并不是熟路人，瞬时的记忆不可能完成，如果没人提点，庞大的知识体系你根本无法构建，即便一个很小的问题你都不知从何开始。而，这本书中，很多地方都体现了很好地数据处理逻辑，如果给我们自己来组织，很可能不知道从哪着手，或者无法合理地安排函数结构，也可能无法达到强大的可修改性。其中的很多逻辑，在理论上很简单，但是通过代码来表达，不同的形式，也可能千差万别。==

----------
**一、关联分析**
 1. 什么是关联分析：关联分析就是你找出两个事物之间的联系。比如找出吸烟和肺癌之间的联系，或者典型的啤酒和尿布的故事。
 2. 两个指标：
 * 支持度：数据集中包含该项数据的比例
 * 可信度、置信度：通过P推断出H的可靠性概率：P->H = （P U H）/(P),可信度越高，相关度越大
 3. Apriori算法：
 * 原理：算法的主要作用是为了减少计算量。正原理，如果某个项是频繁的，则它所有的子项也是频繁的；同理，如果某个子项是非频繁的，那么它的所有超集也是非频繁的。这样，如果我们这些高计算频繁项，然后扩展对应的超集，这样就排除了非频繁的。具体的代码表达可能很复杂，需要判断各种情况。而且其中也存在一些可以简化计算的其他方式。
 * 代码1：函数逻辑 Apriori.py：
 

``` stylus

def loadDataSet():
    return [[1,3,4],[2,3,5],[1,2,3,5],[2,5]]

def loadDataSet2():
    return [[1,3,4,6],[2,3,5,6],[1,2,3,5,6],[2,5,6]]


# 得出数据集的单个元素Set
def createC1(dataSet):
    """
    :param dataSet:
    :return:
    """
    C1 = []
    for data in dataSet:
        for item in data:
            if not [item] in C1: #需要通过[]将元素封装为元组，便于后续作为一个整体进行集合运算
                C1.append([item])
    return C1

# 计算单个元素的支持度，即占比
def scanD(data, Ck, minSupport):
    """
    相当于遍历D，求出所有的Ck中的元素支持度大于minsupport的
    :return: 满足支持度的元素、所有元素-支持度的map
    :param data: dataSet
    :param Ck:元素集合
    :param minSupport:最小支持度阈值
    """
    eleMap = {} # 元素-元素总个数 的映射
    for oneData in data:
        for element in Ck:
            if element.issubset(oneData): # set子集
                # 这样写会报错，因为字典的key不可变，而set是可变的，不能作为字典{}的key，所以需要使用frozenset(冻结set，不可变)
                # TypeError: unhashable type: 'set' --> element
                if not eleMap.__contains__(element): # 这里与原书不同
                    eleMap[element] = 1
                else:
                    eleMap[element] += 1

    # 计算元素支持度
    # 注意：错误object of type 'map' has no len()
    # 原因：In Python 3, map returns an iterator not a list:
    # python3中map是一个迭代器，不能使用len(),需要将map转化为list才能使用
    # numsum = float(len(data))
    numsum = len(list(data)) # 这个地方报错是因为在其它代码处，使用了list作为变量！记住：永远不要使用关键词作为变量
    supportMap = {} # 所有元素支持度的值
    retList = [] # 满足支持度的元素
    for key in eleMap:
        try:
            support = eleMap[key]/float(numsum)
        except:
            support = 0.0
        if support >= minSupport:
            retList.insert(0,key) # 将key插入到第0个位置之前（即最开始位置）
        supportMap[key] = support
    return retList,supportMap

# 将组合Lk合称为每个集合为k个元素的集合
# 如[{1},{2},{3}]-->[{1,2},{2,3},{1,3}]
def aprioriGen(Lk, k):
    retList = []
    lenLk = len(Lk)
    for i in range(lenLk):
        # 注意：下面的逻辑复杂，需要参考书中p208
        # k-2,是为了将前k-2项相同时，集合合并；比如当k=2时，只有一个元素{1}、{2}，此时k-2=0，L1==L2,直接并集运算
        # k=3时，有两个元素{1,2}、{1,3}、{2,3}、{5,6}；由于只能合并为大小为k=3的集合，所以必然只有k-2项是相同的，也只需要组合这种情况
        # 此时组合只有{1,2,3}；
        for j in range(i+1,lenLk):
            L1 = list(Lk[i])[: k-2]
            L2 = list(Lk[j])[: k-2]
            # print '-----i=', i, k-2, Lk, Lk[i], list(Lk[i])[: k-2]
            # print '-----j=', j, k-2, Lk, Lk[j], list(Lk[j])[: k-2]
            L1.sort()
            L2.sort()
            # 第一次 L1,L2 为空，元素直接进行合并，返回元素两两合并的数据集
            # if first k-2 elements are equal
            if L1 == L2:
                # set union
                # print 'union=', Lk[i] | Lk[j], Lk[i], Lk[j]
                retList.append(Lk[i] | Lk[j])
    return retList

# 提升一级：计算所有可能集合的支持度
# 找出数据集 dataSet 中支持度 >= 最小支持度的候选项集以及它们的支持度。即我们的频繁项集。
def apriori(dataSet, minSupport=0.5):
    """apriori（首先构建集合 C1，然后扫描数据集来判断这些只有一个元素的项集是否满足最小支持度的要求。那么满足最小支持度要求的项集构成集合 L1。然后 L1 中的元素相互组合成 C2，C2 再进一步过滤变成 L2，然后以此类推，知道 CN 的长度为 0 时结束，即可找出所有频繁项集的支持度。）
    Args:
        dataSet 原始数据集
        minSupport 支持度的阈值
    Returns:
        L 频繁项集的全集
        supportData 所有元素和支持度的全集
    """
    # C1 即对 dataSet 进行去重，排序，放入 list 中，然后转换所有的元素为 frozenset
    C1 = createC1(dataSet)
    C1 = list(map(frozenset,C1))
    # print 'C1: ', C1
    # 对每一行进行 set 转换，然后存放到集合中
    D = list(map(set, dataSet))
    # print 'D=', D
    # 计算候选数据集 C1 在数据集 D 中的支持度，并返回支持度大于 minSupport 的数据
    L1, supportData = scanD(D, C1, minSupport)
    # print "L1=", L1, "\n", "outcome: ", supportData

    # L 加了一层 list, L 一共 2 层 list
    L = [L1]
    k = 2 # 一组合有两个元素为起点
    # 判断 L 的第 k-2 项的数据长度是否 > 0。第一次执行时 L 为 [[frozenset([1]), frozenset([3]), frozenset([2]), frozenset([5])]]。L[k-2]=L[0]=[frozenset([1]), frozenset([3]), frozenset([2]), frozenset([5])]，最后面 k += 1
    while (len(L[k-2]) > 0):
        # print 'k=', k, L, L[k-2]
        # 输出k个元素组合时，所对应的集合，并求解对应满足的支持度
        Ck = aprioriGen(L[k-2], k) # 例如: 以 {0},{1},{2} 为输入且 k = 2 则输出 {0,1}, {0,2}, {1,2}. 以 {0,1},{0,2},{1,2} 为输入且 k = 3 则输出 {0,1,2}
        # print 'Ck', Ck

        Lk, supK = scanD(D, Ck, minSupport) # 计算候选数据集 CK 在数据集 D 中的支持度，并返回支持度大于 minSupport 的数据
        # 保存所有候选项集的支持度，如果字典没有，就追加元素，如果有，就更新元素
        supportData.update(supK) # 理解update的含义
        # 下面是为了排除最后一个[]空值
        if len(Lk) == 0:
            break
        # Lk 表示满足频繁子项的集合，L 元素在增加，例如:
        # l=[[set(1), set(2), set(3)]]
        # l=[[set(1), set(2), set(3)], [set(1, 2), set(2, 3)]]
        L.append(Lk)
        k += 1
        # print 'k=', k, len(L[k-2])
    return L, supportData

# 输出关联项及对应的值；如p->h
def generateRules(L,supportData,minConf):
    bigRuleList = []
    for i in range(1,len(L)):
        for freqSet in L[i]:
            H1 = [frozenset([item]) for item in freqSet] # 这里的H1是freqSet的变形，起初看了半天没明白
            if i==1:
                calcCong(freqSet,H1,supportData,bigRuleList,minConf)
            else:
                ruleFromConseq(freqSet,H1,supportData,bigRuleList,minConf)
    return bigRuleList

def calcCong(freqSet,H,supportData,bigRuleList,minConf=0.7):
    prunedH = [] # 单词pruned-修剪
    for conseq in H:
        conf = supportData[freqSet]/supportData[freqSet-conseq]
        if conf >= minConf:
            print(freqSet-conseq,'-->',conseq,'is:',conf)
            bigRuleList.append((freqSet-conseq,conseq,conf))
            prunedH.append(conseq)
    return prunedH

# 这个是为了解决另一个情况，例如：{2，3，4}，我们需要找{2,3}-->{4},还需要知道{2}-->{3,4},即hmp1={2,3},m=2
def ruleFromConseq(freqSet,H,supportData,bigRuleList,minConf=0.7):
    length = len(H[0])
    if len(freqSet) > (length+1):
        sonH = aprioriGen(H,length+1)
        isSonH = calcCong(freqSet,sonH,supportData,bigRuleList,minConf)
        # 这里使用了一个数学逻辑，如果isSonH只有一个，就说明无法再划分
        #（定理：如果某条规则不满足可信度，左部所有的子集也不满足可信度）
        if len(isSonH) > 1: # 去掉这个语句也行，只是计算量增大
            ruleFromConseq(freqSet,isSonH,supportData,bigRuleList,minConf)
```

 * 代码2：测试函数 test.py：
 

``` stylus
# 另外我发现一个问题：党创建一个python项目时，会自动创建一个__init__.py文件，
# 这是默认初始化文件，如果直接运行这个文件，首先初始化会运行一次这个文件，之后会再次运行这个文件。也就是这个文件被运行了两次。
from unit11 import Apriori # 这个按照格式来，否则会报错，虽然错误不影响内部逻辑

dataSet = Apriori.loadDataSet2()
print(dataSet)

C1 = Apriori.createC1(dataSet)
print(C1)

D = map(set,dataSet)
# print(list(D))
# C = map(set,C1)
C = map(frozenset,C1)
# print(list(C))

# 注意通过list(map)来将map转为list，只能第一次有效，第二次返回的list为空（可能是由于map是迭代的，一次list遍历后，就到了末尾）
data1 = list(D)
label = list(C)
L1,suppDataMap = Apriori.scanD(data1,label,0.5)
print('满足的支持度元素是：'+str(L1))
print('所有元素的支持度是：'+str(suppDataMap))
print('--------p207')

L,suppDataMap2 = Apriori.apriori(data1)
for onel in L:
    print(onel)
    print(str(suppDataMap2))

print('-----11')
rules = Apriori.generateRules(L,suppDataMap2,0.5)
print(rules)

# 由于国会测试那个需要申请key，比较麻烦，就没写
print('发现毒蘑菇的相似特征')
# 毒蘑菇分类
mushDataSet = [line.split() for line in open('mushroom.dat').readlines()]
L,supportMapMush = Apriori.apriori(mushDataSet,0.3)

# 其中L为关联集合标签，标签中2代表是毒蘑菇，所以只需要判断那些标签含有2，就可以大体确定那些可能是毒蘑菇
# 注意数据中的数字代表特征，所有的特征从1开始标，不能重复
for row in L:
    for item in row:
        # set.intersection(item)
        # Return a new set with elements common to the set and all others.
        # 即返回一个包含此元素的新的set
        if item.intersection('2'):
            print(item)

```
以下链接可能解释还行，没看，可以参考一下；另外请参考书籍
相关链接：http://blog.csdn.net/u010859707/article/details/78180301

