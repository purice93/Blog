---
title: python网络数据采集-第5章存储数据
tags: 数据采集,存储,pymysql,email,csv
grammar_cjkRuby: true
---

5.1 媒体文件简述
>网络上的资源很多，有图片，视频，常规文件rar\zip等，由于网络爬去的数据量大，如果直接保存，相对只保存对应的链接字符串，有很多缺陷:1、由于下载，导致爬取速度慢；2、消耗存储空间；3、而且还要实现文件下载的方法，繁琐；优点：1、防止由于盗链改变导致的信息丢失（盗链即网页内部通往外部网页的链接），盗链所连接的信息可能变化

简单地文件下载

``` stylus
# 1、单个文件下载
from urllib.request import urlretrieve
from urllib.request import urlopen
from bs4 import BeautifulSoup


'''
爬去网页示例内容
<a href="/" id="logo" rel="home" title="Home">
    <img alt="Home" src="http://www.pythonscraping.com/sites/default/files/lrg_0.jpg"/>
'''


html = urlopen("http://www.pythonscraping.com")
bsObj = BeautifulSoup(html,"html5lib")
print(bsObj)

# 注意这里find格式映射关系
imageLocation = bsObj.find("a",{"id":"logo"}).find("img")["src"]
print(imageLocation)

# 下载图片
urlretrieve(imageLocation,"img01.jpg")
```


``` stylus
# 批量爬取文件并下载
import os
from urllib.request import urlopen
from urllib.request import urlretrieve

from bs4 import BeautifulSoup

'''
爬去网页示例内容
<a href="/" id="logo" rel="home" title="Home">
    <img alt="Home" src="http://www.pythonscraping.com/sites/default/files/lrg_0.jpg"/>
'''

# 为何这里写的如此复杂？
# 在于我们希望将下载的资源根据所在的主url，来进行相对应的文件夹

baseurl = "http://pythonscraping.com"
downloadDirectory = "download"

html = urlopen("http://www.pythonscraping.com")
bsObj = BeautifulSoup(html, "html5lib")
downloadList = bsObj.findAll(src=True)
print(len(downloadList))


# 获取文件夹绝对路径的url
def getAbsoluteURL(baseUrl, downloadUrl):
    url = ""
    if downloadUrl.startswith("http://www."):
        url = "http://" + downloadUrl[11:]
    if baseurl not in url:
        return None
    return url


# 获取文件夹绝对路径（用于地址保存）
def getDownloadPath(baseurl, fileurl, downloadDirectory):
    # 含有？的需要删除？之后的字符串（下同）
    if "?" in fileurl:
        endIndex = fileurl.index("?")
        fileurl = fileurl[:endIndex]
    path = fileurl.replace(baseurl, "")
    path = downloadDirectory + path
    directory = os.path.dirname(path)

    if not os.path.exists(directory):
        os.makedirs(directory)
    return path

for download in downloadList:
    fileurl = getAbsoluteURL(baseurl, download["src"])
    #  'NoneType' object has no attribute 'replace'
    if fileurl is not None:
        if "?" in fileurl:
            endIndex = fileurl.index("?")
            fileurl = fileurl[:endIndex]
        print(fileurl)
        urlretrieve(fileurl, getDownloadPath(baseurl, fileurl, downloadDirectory))
```


5.2 把数据存储到csv中
> 这里的意思是将Html中的表格数据存储到csv表格中。1、找到一个table标签及其对应的id；2、找到所有的tr标签，即所有的行信息；3、遍历所有的行信息，找到每行对应的td或者th标签，写入csv中

``` stylus
# 例1：简单地csv文件写入
# 文件如果不存在会自动创建
import csv
csvFile = open("files/test.csv","w+")
try:
    writer = csv.writer(csvFile)
    writer.writerow(('number1','number2','number3'))
    for i in range(10):
        writer.writerow((i,i+2,i*2))
finally:
    csvFile.close()


# 例2，网页信息写入csv
from urllib.request import urlopen
from bs4 import BeautifulSoup
import codecs

html = urlopen("https://baike.baidu.com/item/%E5%9B%BD%E5%86%85%E7%94%9F%E4%BA%A7%E6%80%BB%E5%80%BC/31864?fromtitle=GDP&fromid=41201")
bsObj = BeautifulSoup(html,"html5lib")

# print(bsObj)

table = bsObj.findAll("table",{"class":"table-view log-set-param"})[1]
rows = table.findAll("tr")

'''
excel打开csv文件，可以识别编码“GB2312”，但是不能识别“utf-8”,数据库里的字符串编码是utf-8.因此：

当从csv读取数据（data）到数据库的时候，需要先把GB2312转换为unicode编码，然后再把unicode编码转换为utf-8编码：data.decode('GB2312').encode('utf-8')

当从数据库读取数据（data）存到csv文件的时候，需要先把utf-8编码转换为unicode编码，然后再把unicode编码转换为GB2312编码：data.decode('utf-8').encode('GB2312')
'''

# windows下使用GB2312
# linux下使用utf-8
csvFile = open("files/gdpList.csv",'wt',newline='',encoding="GB2312")
writer = csv.writer(csvFile)


# 另外注意pycharm中打开csv文件肯能感觉是乱的，其实就是以逗号分开的，没有问题
try:
    for row in rows:
        csvRow = []
        for cell in row.findAll('td'):
            csvRow.append(cell.getText())
        writer.writerow(csvRow)
finally:
    csvFile.close()
```


5.3 python连接mysql
>这里的连接和java都是一个道理，但是相比java要简便很多。安装pymysql库后自动导入就可以使用pymysql

``` stylus
'''
# 1、简单实验：

import pymysql

conn = pymysql.Connect("localhost","root","root","pythontes")

cur = conn.cursor()
cur.execute("select * from pages")


# fetchone():"""Fetch the next row"""
# fetchall():"""Fetch all the rows"""
for row in cur.fetchall():
    print(row)
    cur.close()
conn.close()


# 以上输出为：
# ('a1', 'liuBei', '123')
# ('a2', 'guanYu', '123')
# ('a3', 'zhangSanFei', '123')
'''


# 2、网页保存到mysql
# 数据库存储网页信息
# 还是上次那个GDP排名数据
from urllib.request import urlopen
from bs4 import BeautifulSoup
import codecs
import pymysql

html = urlopen("https://baike.baidu.com/item/%E5%9B%BD%E5%86%85%E7%94%9F%E4%BA%A7%E6%80%BB%E5%80%BC/31864?fromtitle=GDP&fromid=41201")
bsObj = BeautifulSoup(html,"html5lib")

# print(bsObj)

table = bsObj.findAll("table",{"class":"table-view log-set-param"})[1]
rows = table.findAll("tr")

# 输出到数据库mysql
conn = pymysql.connect("localhost","root","root","pythontes")

# 错误记录1：UnicodeEncodeError: 'latin-1' codec can't encode character
# 原因：windows系统编码不同
# 参考链接：https://stackoverflow.com/questions/3942888/unicodeencodeerror-latin-1-codec-cant-encode-character
conn.set_charset("utf8")
cur = conn.cursor()

try:
    tag = True
    for row in rows:
        # 这里需要跳过第一行说明部分
        if tag:
            tag = False
            continue

        csvRow = []
        for cell in row.findAll('td'):
            csvRow.append(cell.getText())
        print(csvRow)

        '''
        错误记录2：
        由于爬取的网页都是字符串类型，但是数据库不支持格式转换写入，
        即(%d,int(csvRow[0]))转换无效，所以只能将所有的类型保存为字符串类型
        '''

        # 一种写法
        # cur.execute("insert into gdpList values(%s,%s,%s,%s,%s,%s)",(csvRow[0],csvRow[1],csvRow[2],csvRow[3],csvRow[4],csvRow[5]))

        # 另一种写法
        sql = "insert into gdpList values(%s,%s,%s,%s,%s,%s)"
        cur.execute(sql,csvRow)

        # 提交
        cur.connection.commit()
finally:
    cur.close()
    conn.close()
```

**数据库实现网络图的爬取（深度爬取）**
简要谈谈数据库的优化：

* id和主键：首先最好每一个表都要有一个id字段，对数据查找都是有帮助的；关键字一般创建为单一不变的值，最能代表此行数据的字段
* 索引：索引用该改善数据的查找优化，根据不同的数据形式设定对应的索引，索引相当于字典，比如我们可以对创建一个字段的前10个字符作为索引：1、ALTER TABLE table_name ADD INDEX index_name (column_list（10）) ；2、CREATE INDEX index_name ON table_name (column_list（10）)。
* 分表来节省空间和时间：单个数据表可能存在大量的重复数据，比如统计一个镇的所有居民信息，对于村这个字段，可能很多人都是在一个村，所以存在大量的重复。因此分为：镇-村表，和村-居民表，要节省不少空间，同时查询速度也快

>以下为爬取深度网络，并将网络链接对应信息保存；使用了两个数据库表

``` stylus
# 建数据库表
CREATE TABLE `pages` (
`id` INT NOT NULL AUTO_INCREMENT,
`url` VARCHAR(255) NOT NULL,
`created` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
PRIMARY KEY (`id`)
)

CREATE TABLE `links` (
`id` INT NOT NULL AUTO_INCREMENT,
`fromPageId` INT NOT NULL,
`toPageId` INT NOT NULL,
`created` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
PRIMARY KEY (`id`)
)
```


``` stylus
""" 
@author: zoutai
@file: linkToLink.py 
@time: 2018/01/24 
@description: 爬取一个深度为4的网络，保存网络之间的连接关系（两两对应）
"""

import re
from urllib.request import urlopen

import pymysql
from bs4 import BeautifulSoup

conn = pymysql.connect("localhost", "root", "root", "pythontes")
conn.set_charset("utf8")
cur = conn.cursor()

# 定义一个set，对page进行去重
pages = set()


def insertLink(fromPageId, toPageId):
    cur.execute("select * from links where fromPageId = %s and toPageId = %s", (int(fromPageId), int(toPageId)))
    if cur.rowcount == 0:
        cur.execute("insert into links (fromPageId,toPageId) values (%s,%s)", (int(fromPageId), int(toPageId)))
        conn.commit()


# 返回当前url存储的id
def insertPage(pageUrl):
    cur.execute("select * from pages where url = %s", (pageUrl))
    if cur.rowcount == 0:
        cur.execute("insert into pages (url) values (%s)", pageUrl)
        conn.commit()
        return cur.lastrowid
    else:
        return cur.fetchone()[0]


def getLinks(pageUrl, recursionLevel):
    global pages
    if recursionLevel < 0:
        return
    pageId = insertPage(pageUrl)
    html = urlopen("https://en.wikipedia.org" + pageUrl)
    bsObj = BeautifulSoup(html, "html5lib")
    for link in bsObj.findAll("a", href=re.compile("^(/wiki/)((?!:).)*$")):
        insertLink(pageId, insertPage(link.attrs["href"]))
        if link.attrs['href'] not in pages:
            newPage = link.attrs['href']
            pages.add(newPage)
            getLinks(newPage, recursionLevel - 1)


getLinks("/wiki/Kevin_Bacon", 2)

```


5.4 发送Email数据，使用163的SMTP邮件服务器

""" 
@author: zoutai
@file: sendEmail.py 
@time: 2018/01/24 
@description: 邮件发送
"""

# 完全参考：[链接][1]

``` stylus
from smtplib import SMTP
from email.mime.text import MIMEText
from email.header import Header

def send_email(SMTP_host, from_addr, password, to_addrs, subject, content):
    email_client = SMTP(SMTP_host)
    email_client.login(from_addr, password)
    # create msg
    msg = MIMEText(content,'plain','utf-8')
    msg['Subject'] = Header(subject, 'utf-8')#subject
    msg['From'] = 'soundslow<18392960379@163.com>'
    msg['To'] = "530337704@qq.com"
    email_client.sendmail(from_addr, to_addrs, msg.as_string())

    email_client.quit()

if __name__ == "__main__":
    send_email("smtp.163.com","18392960379@163.com","zouzixuang1992","530337704@qq.com","来自163的邮件","今晚的卫星有点亮")
```



  [1]: https://segmentfault.com/q/1010000005643494
