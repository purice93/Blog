---
title: 遗传算法求解TSP问题
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>TSP问题（Travelling Salesman Problem）即旅行商问题，又译为旅行推销员问题、货郎担问题，是数学领域中著名问题之一。假设有一个旅行商人要拜访n个城市，他必须选择所要走的路径，路径的限制是每个城市只能拜访一次，而且最后要回到原来出发的城市。路径的选择目标是要求得的路径路程为所有路径之中的最小值。

[参考：用遗传算法求解TSP问题][1]
[相关博客][2]

由于网上的结果只有路径图，而没有写明distance值，所以没法参照
![迭代80000次结果，distance=147.0864047407135][3]
``` stylus
""" 
@author: zoutai
@file: TSP_Solve.py 
@time: 2018/04/22 
@description: 旅行商问题求解
"""
import math
import random
import matplotlib.pyplot as plt

def geneticoptimize(popsize=50, step=1, mutprob=0.2, elite=0.2, mixiter=80000):

    # # 生成随机DNA
    # list1 = [i for i in range(lengthDNA)]
    # random.shuffle(list1)
    # print(list1)

    # 变异(交换)
    def mutate(list1):
        index1 = random.randint(0,lengthDNA - 1)
        index2 = random.randint(0,lengthDNA - 1)
        list1[index1], list1[index2] = list1[index2], list1[index1]
        return list1
    # 重组
    def cross(list1, list2):
        index1 = random.randint(0, lengthDNA - 1)
        index2 = random.randint(index1, lengthDNA - 1)
        tempGene = list2[index1:index2]
        newDNA = []
        temp = 0
        for gene in list1:
            if index1==temp:
                newDNA.extend(tempGene)
                temp+=1
            if gene not in tempGene:
                newDNA.append(gene)
                temp+=1
        return newDNA
    # 距离
    def distance(list1):
        distance = 0.0
        # i从-1到32,-1是倒数第一个
        if not isinstance(list1, list):
            print()
        index1, index2 = 0,0
        for i in range(len(list1)-1):
            index1, index2 = list1[i], list1[i + 1]
            # print(index1,index2)
            # print(i)
            distance += math.sqrt((float(cities[index1][4]) - float(cities[index2][5])) ** 2 + (float(cities[index1][6]) - float(cities[index2][7])) ** 2)

        # index2 = list1[len(list1)-1]
        distance += math.sqrt(
            (float(cities[index2][8]) - float(cities[0][9])) ** 2 + (float(cities[index2][10]) - float(cities[0][11])) ** 2)

        return distance


    cities = []
    for line in open('distanceMatrix.txt', 'r', encoding="utf8"):
        # 按照tab键分割
        cities.append(line.strip().split('\t'))
    lengthDNA = len(cities)

    # 开始
    # 第一步：初始化种群
    pop = []
    for i in range(popsize):
        list1 = [j for j in range(lengthDNA)]
        random.shuffle(list1)
        pop.append(list1.copy())

    # 进化选择数
    toplite = int(popsize * elite)

    # 进化选择，排序
    for i in range(mixiter):
        # 排序，进行物种进化选择
        distances = [(distance(v),v) for v in pop]
        distances.sort()
        ranked = [v for (s,v) in distances]

        pop = ranked[:toplite]
        while len(pop)<popsize:
            if random.random()<mutprob:
                c = random.randint(0,toplite)
                pop.append(mutate(ranked[c]))
            else:
                c1 = random.randint(0,toplite)
                c2 = random.randint(0,toplite)
                pop.append(cross(ranked[c1],ranked[c2]))
        print(distances[0][0])

    list1, distance = distances[0][12],distances[0][0]
    
    # 画图
    plt.figure()
    xlist = []
    ylist = []
    for i in list1:
        xlist.append(float(cities[i][13]))
        ylist.append(float(cities[i][14]))
    xlist.append(float(cities[list1[0]][15]))
    ylist.append(float(cities[list1[0]][16]))
    plt.plot(xlist,ylist,color='r')
    plt.scatter(xlist, ylist, color='b')
    plt.title('TSP_gen:'+str(distance))
    plt.show()
    return distances[0][17],distances[0][0]

list1, distance = geneticoptimize()
print(distance,list1)
```


  [1]: https://blog.csdn.net/taonull/article/details/45972393
  [2]: https://blog.csdn.net/soundslow/article/details/80043652
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524455963525.jpg
  [4]: https://blog.csdn.net/taonull/article/details/45972393
  [5]: https://blog.csdn.net/taonull/article/details/45972393
  [6]: https://blog.csdn.net/soundslow/article/details/80043652
  [7]: https://blog.csdn.net/soundslow/article/details/80043652
  [8]: https://blog.csdn.net/taonull/article/details/45972393
  [9]: https://blog.csdn.net/taonull/article/details/45972393
  [10]: https://blog.csdn.net/soundslow/article/details/80043652
  [11]: https://blog.csdn.net/soundslow/article/details/80043652
  [12]: https://blog.csdn.net/taonull/article/details/45972393
  [13]: https://blog.csdn.net/taonull/article/details/45972393
  [14]: https://blog.csdn.net/soundslow/article/details/80043652
  [15]: https://blog.csdn.net/taonull/article/details/45972393
  [16]: https://blog.csdn.net/soundslow/article/details/80043652
  [17]: https://blog.csdn.net/taonull/article/details/45972393
