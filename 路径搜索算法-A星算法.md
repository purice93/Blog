---
title: 路径搜索算法-A星算法
tags: 启发式算法,Astar,最短路径
grammar_cjkRuby: true
---
>A*算法的核心就是f(M)=g(M)+h(M)，g(M)表示当前节点到初始节点的最短路径大小，h(M)表示当前节点到终止节点的可能路径大小（即虚拟的，如欧氏距离，曼哈顿距离等）。即不仅仅考虑当前节点到初始节点的最短路径，还要考虑离终点的距离。而一般的路径查找方法只会考虑当前节点到初始节点的路径

![路径选择][1]

**两个关键集合open和close**
![open和close][2]
简单一句话即：open 往close 插入元素

open：已经搜索的区域
close：比较已经搜索的区域中的数值f，找到最好的插入close
即close中保存的是最短路径节点，open中是待选节点



``` stylus
""" 
@author: zoutai
@file: Astar.py 
@time: 2018/04/23 
@description: 
"""

# 地图
import math

mapStr = [
    '############################################################',
    '#..........................................................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#.......S.....................#............................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#.............................#............................#',
    '#######.#######################################............#',
    '#....#........#............................................#',
    '#....#........#............................................#',
    '#....##########............................................#',
    '#..........................................................#',
    '#..........................................................#',
    '#..........................................................#',
    '#..........................................................#',
    '#..........................................................#',
    '#...............................##############.............#',
    '#...............................#........E...#.............#',
    '#...............................#............#.............#',
    '#...............................#............#.............#',
    '#...............................#............#.............#',
    '#...............................###########..#.............#',
    '#..........................................................#',
    '#..........................................................#',
    '############################################################']


class Node:
    """
    用于节点的表示，parent用来在成功的时候回溯路径（相当于一个链表）
    """

    def __init__(self, parent, x, y, dist):
        self.parent = parent
        self.x = x
        self.y = y
        self.dist = dist # 这里只是g(M)


class A_Star:

    # 初始化地图长和宽，开始终止节点，open集合和close集合，
    # 路径集合path用于最后反向遍历找出路径
    def __init__(self, start, end, w=60, h=30):
        self.start = start
        self.end = end

        self.width = w
        self.height = h

        self.open = []
        self.close = []
        self.path = []

    def find_path(self):
        # 1、初始化开始节点
        p = Node(None, self.start[0], self.start[1], 0.0)
        while True:
            # 扩展F值最小的节点
            self.extend_round(p)

            # 如果开open列表为空，表示能走的路全走完了，无法走了，表示不存在路径，返回（封闭空间）
            if not self.open:
                return

            # 获取F值最小的节点（最短路径）
            idx, p = self.get_best()

            # 如果是终点，直接返回
            if self.end[0]==p.x and self.end[1]==p.y:
                # 画路径
                while p:
                    self.path.append((p.x, p.y))
                    p = p.parent
                return

            # 不是终点，则将此节点压入关闭列表，并从开放列表里删除（即这是最好的节点）
            self.close.append(p)
            del self.open[idx]

    def extend_round(self, p):
        # 可以从8个方向走
        xs = (-1, 0, 1, -1, 1, -1, 0, 1)
        ys = (-1, -1, -1, 0, 0, 1, 1, 1)
        # 只能走上下左右四个方向
        #        xs = (0, -1, 1, 0)
        #        ys = (-1, 0, 0, 1)
        for x, y in zip(xs, ys):

            new_x, new_y = x + p.x, y + p.y

            # 判断1：是否越界或者撞墙
            if new_x < 0 or new_x >= self.width or new_y < 0 or new_y >= self.height \
                    or MapData[new_y][new_x] == '#':
                continue

            # 生成新的节点和g(M)值，get_cost可以默认为1，这里为了考虑斜线和直线不同
            newNode = Node(p, new_x, new_y, p.dist+self.get_cost(p,new_x,new_y))

            # 新节点在close列表中，表示回环了，之前路径更好，忽略当前节点
            # 比如c：a->b->c->b
            if self.node_in_close(newNode):
                continue

            # 新节点是否在open列表中
            i = self.node_in_open(newNode)

            # -1:如果不在，比较更新open中的此节点数值
            if i!=-1:
                if self.open[i].dist > newNode.dist:
                    self.open[i].parent = p
                    self.open[i].dist = newNode.dist
                continue

            # 否则，则加入open列表，用于备选比较
            # 将新节点放入open中（第一次访问）
            self.open.append(newNode)

    def get_best(self):
        best = None
        bv = 1000000  # 如果你修改的地图很大，可能需要修改这个值
        bi = -1
        for idx, i in enumerate(self.open):
            f = i.dist + math.sqrt(
                (self.end[0] - i.x) * (self.end[0] - i.x)
                + (self.end[1] - i.y) * (self.end[1] - i.y)) * 1.2
            value = f  # 获取F值
            if value < bv:  # 比以前的更好，即F值更小
                best = i
                bv = value
                bi = idx
        return bi, best

    def get_searched(self):
        searched = []
        for i in self.open:
            searched.append((i.x, i.y))
        for i in self.close:
            searched.append((i.x, i.y))
        return searched

    def get_cost(self, p, new_x, new_y):
        """
        上下左右直走，代价为1.0，斜走，代价为1.4
        """
        if p.x == new_x or p.y == new_y:
            return 1.0
        return 1.4


    def node_in_close(self, node):
        for i in self.close:
            if node.x == i.x and node.y == i.y:
                return True
        return False

    def node_in_open(self, node):
        for i, n in enumerate(self.open):
            if node.x == n.x and node.y == n.y:
                return i
        return -1


# 查找字符所在位置
def get_symbol_XY(s):
    for y, line in enumerate(MapData):
        try:
            x = line.index(s)
        except:
            continue
        else:
            break
    return [x, y]

def search():
    # 初始化整个程序
    a_star = A_Star(START, END,len(MapData[0]),len(MapData))
    # 从开始几点查找路径
    a_star.find_path()

    # 标记已搜索区域为'-'
    # 已搜索区域=open+close
    searched = a_star.get_searched()
    for x, y in searched:
        MapData[y][x] = '-'

    # 标记路径为'>'
    path = a_star.path
    for x, y in path:
        MapData[y][x] = '>'

    # 打印最短路径长度和搜索区域长度
    print("path length is %d" % (len(path)))
    print("searched squares count is %d" % (len(searched)))

    # 恢复开始和终止节点的标记
    MapData[START[1]][START[0]] = 'S'
    MapData[END[1]][END[0]] = 'E'

if __name__ == '__main__':
    # 因为python里string不能直接改变某一元素，所以用MapData来存储搜索时的地图
    MapData = []
    for line in mapStr:
        MapData.append(list(line))

    # 初始化起始和终点坐标[x,y]
    START = get_symbol_XY('S')
    END = get_symbol_XY('E')

    # 开始搜索路径
    search()

    # 打印搜索后的地图
    for line in MapData:
        print(''.join(line))

```

参考：
[http://www.voidcn.com/article/p-yaynhyke-cm.html][3]
[https://blog.csdn.net/zgwangbo/article/details/52078338][4]
[https://blog.csdn.net/v_july_v/article/details/6093380][5]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524625051456.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524627585681.jpg
  [3]: http://www.voidcn.com/article/p-yaynhyke-cm.html
  [4]: https://blog.csdn.net/zgwangbo/article/details/52078338
  [5]: https://blog.csdn.net/v_july_v/article/details/6093380
