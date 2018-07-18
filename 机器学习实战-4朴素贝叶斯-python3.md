---
title: 机器学习实战-4朴素贝叶斯-python3 
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

==这一章原理很简单，相关的知识到处都是，《数学之美》讲过，《统计学习方法》西瓜书都有详细的概述。但是就是一个简单的概率问题，如果真正遇到实际问题，却也并不是很好解决的。这其中往往是实际操作时往往和理论空想不同，数据的合理安排非常重要。贝叶斯的使用非常广泛，其实就目前而言，现实生活中很多的人工智能方面的处理其实就是用的贝叶斯，特别是对于商业数据的处理分析。比如搜索公司中的网页关联分析，社交软件中根据行为表现进行群体划分，以及新闻中的精准推送等。==


----------
一、贝叶斯公式

1.  这个定理解决了现实生活里经常遇到的问题：已知某条件概率，如何得到两个事件交换后的概率，也就是在已知P(A|B)的情况下如何求得P(B|A)。
2.  一般而言就是这个概率公式：P(B|A) = P(AB)/P(A) = P(A|B)P(B)/P(A)

二、贝叶斯原理和流程
* 由于这方面的讲述比较多，而且，在我看来，只要知道了上面的公式含义，基本上就知道了使用的方法。这里主要说明一点，原理虽然简单，但是直接操作可能会出现无从下手。所以最好的方式是直接看实例代码，这样更容易懂。之后再做其他的，流程其实是类似的。

三、样例1：文本情感分类

1. 目标：其实就是根据文章的评论，来区分评论的倾向，是积极0，还是否消极1；
2. 步骤：对于训练集，分别提取出积极消极对应的词汇；对于测试集，计算测试集词汇在训练集在积极和消极中的“比重”，来确定将其分为哪一类。
3. 代码：bayes.py

``` stylus

# 案例3：使用beyes从个人广告中获取区域倾向
# 书上说的有点婉转，使得不能很好的和本章联系；其实这里就是给你两个城市的人们说话的用词，
# 然后通过这些用词来区分某个人属于哪个城市，另外这间接的可以得出城市的词云库信息；
# 另外这种方式也是广告商们如何分析用户的行为信息来确定用户的年龄和职业来进行精确推送；还有一些婚恋网站匹配等
from unit04.filterEmail import *
import feedparser

# 这里使用了词袋模型
def calcMostFreq(vocabList,fullText):
    import operator
    freqDict = {}
    for word in vocabList:
        freqDict[word] = fullText.cout(word)
    sortedFreq = sorted(freqDict.items(),key=operator.itemgetter,reverse=True)
    return sortedFreq[:30]

def localWords(feed1,feed0):
    docList = [];classList = [];fullText = []
    minLen = min(len(feed1['entries']),len(feed0['entries']))
    for i in range(minLen):
        wordList = textParse(feed1['entries'][i]['summary'])
        docList.append(wordList)
        fullText.extend(wordList)
        classList.append(1)

        wordList = textParse(feed0['entries'][i]['summary'])
        docList.append(wordList)
        fullText.extend(wordList)
        classList.append(0)
    vocabList = createVocabSet(docList)
    top30Words = calcMostFreq(vocabList,fullText)
    # 去掉出现频率最高的那些词
    for pairW in top30Words:
        if pairW[0] in vocabList:
            vocabList.remove(pairW[0])
    trainSet = list(range(2 * minLen));
    testSet = []
    for i in range(20):
        import numpy
        randIndex = int(numpy.random.uniform(0,len(trainSet)))
        testSet.append(trainSet[randIndex])
        del(trainSet[randIndex])
    trainMat = [];trainClassList = []
    for docIndex in trainSet:
        trainMat.append(bagOfWordsVec(vocabList,docList[docIndex]))
        trainClassList.append(classList[docIndex])
    pSpam,p0V,p1V = trainNB0(array(trainMat),trainClassList)

    errorCount = 0
    for docIndex in testSet:
        testVec = bagOfWordsVec(vocabList,docList[docIndex])
        if classifyNB(testVec,p0V,p1V,pSpam) != classList[docIndex]:
            errorCount += 1
    print('the  rate is :',float(errorCount)/len(testSet))
    return pSpam,p0V,p1V

# 测试
# 说明：可能是由于网络访问的原因，这个无法完成
# ny = feedparser.parse('http://newyork.craiglist.org/stp/insex.rss')
# sf = feedparser.parse('http://sfbay.craiglist.org/stp/insex.rss')
ny = feedparser.parse('http://newyork.craiglist.org/stp/index.rss')
sf = feedparser.parse('http://sfbay.craiglist.org/stp/index.rss')
vocabList,pSF,pNY = localWords(ny,sf)

# 得到两个城市的词频
getTopWords(ny,sf)
```
测试代码：test.py

``` stylus
from unit04.bayes import *


postingList,classVec = loadDataSet()
vocabSet = createVocabSet(postingList)
print(vocabSet)
vocSetOfList = setOfWordsVec(postingList[0],vocabSet)
print(vocSetOfList)

trainMat = []
for rowlist in postingList:
    rowlistset = setOfWordsVec(rowlist,vocabSet)
    trainMat.append(rowlistset)
print(trainMat)
print(classVec)

pA,p0,p1 = trainNB0(trainMat, classVec)
print(p0)
print(p1)

print("test P64----classify")
testingNB() # 我的测试结果是the error rate is : 0.6，一直在这左右徘徊。不知道为什么书上的误差会这么底，我的可能错了？
```

四、样例2：垃圾邮件过滤/邮件分类
filterEmail.py

``` stylus
# 案例2：过滤垃圾邮件
from unit04.bayes import *
def textParse(bigString):
    import re
    sentence = re.split('\W+',bigString)
    return [word.lower for word in sentence if len(word) > 2] # 去掉长度小于2的单词

def spamTest(): # spam垃圾邮件
    # import os
    # os.chdir(r'E:/JavaEE_IJ_WorkSpace/MLInAction')
    docList = [];fullText = [];classList = []
    for i in range(1,26):
        wordList = textParse(open('email/spam/%d.txt' % i).read())
        docList.append(wordList)
        fullText.extend(wordList)
        classList.append(1)

        wordList = textParse(open('email/ham/%d.txt' % i).read())
        docList.append(wordList)
        fullText.extend(wordList)
        classList.append(0)
    vocabList = createVocabSet(docList)
    trainSet = list(range(50));testSet = []
    for i in range(10):
        import numpy
        randIndex = int(numpy.random.uniform(0,len(trainSet)))
        testSet.append(trainSet[randIndex])
        del(trainSet[randIndex])
    trainMat = [];trainClassList = []
    for docIndex in trainSet:
        trainMat.append(setOfWordsVec(vocabList,docList[docIndex]))
        trainClassList.append(classList[docIndex])
    pSpam,p0V,p1V = trainNB0(array(trainMat),trainClassList)

    errorCount = 0
    for docIndex in testSet:
        testVec = setOfWordsVec(vocabList,docList[docIndex])
        if classifyNB(testVec,p0V,p1V,pSpam) != classList[docIndex]:
            errorCount += 1
    print('the error rate is :',float(errorCount)/len(testSet))

spamTest()
```

五、样例3：根据某个人说话词语来判断所在的城市
getYourCityFromWord.py

``` stylus

# 案例3：使用beyes从个人广告中获取区域倾向
# 书上说的有点婉转，使得不能很好的和本章联系；其实这里就是给你两个城市的人们说话的用词，
# 然后通过这些用词来区分某个人属于哪个城市，另外这间接的可以得出城市的词云库信息；
# 另外这种方式也是广告商们如何分析用户的行为信息来确定用户的年龄和职业来进行精确推送；还有一些婚恋网站匹配等
from unit04.filterEmail import *
import feedparser

# 这里使用了词袋模型
def calcMostFreq(vocabList,fullText):
    import operator
    freqDict = {}
    for word in vocabList:
        freqDict[word] = fullText.cout(word)
    sortedFreq = sorted(freqDict.items(),key=operator.itemgetter,reverse=True)
    return sortedFreq[:30]

def localWords(feed1,feed0):
    docList = [];classList = [];fullText = []
    minLen = min(len(feed1['entries']),len(feed0['entries']))
    for i in range(minLen):
        wordList = textParse(feed1['entries'][i]['summary'])
        docList.append(wordList)
        fullText.extend(wordList)
        classList.append(1)

        wordList = textParse(feed0['entries'][i]['summary'])
        docList.append(wordList)
        fullText.extend(wordList)
        classList.append(0)
    vocabList = createVocabSet(docList)
    top30Words = calcMostFreq(vocabList,fullText)
    # 去掉出现频率最高的那些词
    for pairW in top30Words:
        if pairW[0] in vocabList:
            vocabList.remove(pairW[0])
    trainSet = list(range(2 * minLen));
    testSet = []
    for i in range(20):
        import numpy
        randIndex = int(numpy.random.uniform(0,len(trainSet)))
        testSet.append(trainSet[randIndex])
        del(trainSet[randIndex])
    trainMat = [];trainClassList = []
    for docIndex in trainSet:
        trainMat.append(bagOfWordsVec(vocabList,docList[docIndex]))
        trainClassList.append(classList[docIndex])
    pSpam,p0V,p1V = trainNB0(array(trainMat),trainClassList)

    errorCount = 0
    for docIndex in testSet:
        testVec = bagOfWordsVec(vocabList,docList[docIndex])
        if classifyNB(testVec,p0V,p1V,pSpam) != classList[docIndex]:
            errorCount += 1
    print('the  rate is :',float(errorCount)/len(testSet))
    return pSpam,p0V,p1V

# 测试
# 说明：可能是由于网络访问的原因，这个无法完成
# ny = feedparser.parse('http://newyork.craiglist.org/stp/insex.rss')
# sf = feedparser.parse('http://sfbay.craiglist.org/stp/insex.rss')
ny = feedparser.parse('http://newyork.craiglist.org/stp/index.rss')
sf = feedparser.parse('http://sfbay.craiglist.org/stp/index.rss')
vocabList,pSF,pNY = localWords(ny,sf)

# 得到两个城市的词频
getTopWords(ny,sf)
```


