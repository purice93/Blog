---
title: 海量搜索服务架构搭建1-基于spring的搜索服务
tags: lucene,solr,中文分词
grammar_cjkRuby: true
---


>相当于一个百度搜索系统


**几个名词解释**

* Lucene简介
	* 1.什么是lucene？
	Lucene是一个全文搜索框架，而不是应用产品。因此它并不像http://www.baidu.com/ 或者google Desktop那么拿来就能用，它只是提供了一种工具让你能实现这些产品。
	* 2.lucene能做什么？
	要回答这个问题，先要了解lucene的本质。实际上lucene的功能很单一，说到底，就是你给它若干个字符串，然后它为你提供一个全文搜索服务，告诉你你要搜索的关键词出现在哪里。知道了这个本质，你就可以发挥想象做任何符合这个条件的事情了。你可以把站内新闻都索引了，做个资料库；你可以把一个数据库表的若干个字段索引起来，那就不用再担心因为“%like%”而锁表了；你也可以写个自己的搜索引擎……
	* 3.你该不该选择lucene 
	下面给出一些测试数据，如果你觉得可以接受，那么可以选择。 
	测试一：250万记录，300M左右文本，生成索引380M左右，800线程下平均处理时间300ms。 
	测试二：37000记录，索引数据库中的两个varchar字段，索引文件2.6M，800线程下平均处理时间1.5ms。
	* 4.lucene为什么这么快
		* 倒排索引
		* 压缩算法
		* 二元搜索
	* 4.1 倒排索引
	根据属性的值来查找记录。这种索引表中的每一项都包括一个属性值和具有该属性值的各记录的地址。由于不是由记录来确定属性值，而是由属性值来确定记录的位置，因而称为倒排索引(invertedindex)
	![单词——文档矩阵][1]
	* 5.lucene的工作方式 
	lucene提供的服务实际包含两部分：一入一出。所谓入是写入，即将你提供的源（本质是字符串）写入索引或者将其从索引中删除；所谓出是读出，即向用户提供全文搜索服务，让用户可以通过关键词定位源
		* 写入流程 
		源字符串首先经过analyzer处理，包括：分词，分成一个个单词；去除stopword（可选）。将源中需要的信息加入Document的各个Field中，并把需要索引的Field索引起来，把需要存储的Field存储起来。将索引写入存储器，存储器可以是内存或磁盘。
		* 读出流程 
		用户提供搜索关键词，经过analyzer处理。对处理后的关键词搜索索引找出对应的Document。用户根据需要从找到的Document中提取需要的Field。
	


* 第一步，创建索引和查询索引
	* IDEA创建webapp工程，导入lucene响应jar包
	![导入5个lucene包][2]
	* 创建两个目录：文章资源目录、索引文件目录
	* 文章资源目录可以直接爬取`wget -o /tmp/wget.log -P /root/data --no-parent --no-verbose -m -D www.bjsxt.com -N --convert-links --random-wait -A html,HTML http://www.bjsxt.com	`
	![两个文件目录][3]
	* 两个函数：创建索引CreateIndex、查找索引SearchIndex
	* 测试结果
	![结果][4]

``` stylus
package com.sxt.lucene;

import org.apache.commons.io.FileUtils;
import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.document.Field;
import org.apache.lucene.document.LongField;
import org.apache.lucene.document.StringField;
import org.apache.lucene.document.TextField;
import org.apache.lucene.index.IndexWriter;
import org.apache.lucene.index.IndexWriterConfig;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.apache.lucene.util.Version;
import org.junit.Test;

import java.io.File;
import java.io.IOException;
/**
 * @author: ZouTai
 * @date: 2018/3/28
 * @description: 创建索引
 */

public class CreateIndex {

    // 静态变量，资源位置
    static String dataDir = "E:/JavaEE_IJ_WorkSpace/lucene/Data/data";
    static String indexDir = "E:/JavaEE_IJ_WorkSpace/lucene/Data/index";

    @Test
    public void createIndex() {
        try {
            // 文件和分析器
            Directory dir = FSDirectory.open(new File(indexDir));
            Analyzer analyzer = new StandardAnalyzer(Version.LUCENE_4_9);

            // 写入索引配置
            IndexWriterConfig indexWriterConfig = new IndexWriterConfig(Version.LUCENE_4_9, analyzer);
            indexWriterConfig.setOpenMode(IndexWriterConfig.OpenMode.CREATE_OR_APPEND);
            IndexWriter indexWriter = new IndexWriter(dir, indexWriterConfig);

            // 遍历文件
            File file = new File(dataDir);
            File[] files = file.listFiles();

            for(File f : files) {
                Document document = new Document();
                // 文件名、内容、最后修改时间
                document.add(new StringField("filename", f.getName(), Field.Store.YES));
                document.add(new TextField("content", FileUtils.readFileToString(f), Field.Store.YES));
                document.add(new LongField("lastModify", f.lastModified(), Field.Store.YES));
                indexWriter.addDocument(document);
            }
            indexWriter.close();

        } catch (IOException e) {
            e.printStackTrace();
        }


    }

}
```


----------


```
package com.sxt.lucene;

import org.apache.lucene.analysis.Analyzer;
import org.apache.lucene.analysis.standard.StandardAnalyzer;
import org.apache.lucene.document.Document;
import org.apache.lucene.index.DirectoryReader;
import org.apache.lucene.index.IndexReader;
import org.apache.lucene.queryparser.classic.QueryParser;
import org.apache.lucene.search.IndexSearcher;
import org.apache.lucene.search.Query;
import org.apache.lucene.search.ScoreDoc;
import org.apache.lucene.search.TopDocs;
import org.apache.lucene.store.Directory;
import org.apache.lucene.store.FSDirectory;
import org.apache.lucene.util.Version;
import org.junit.Test;

import java.io.File;

/**
 * @author: ZouTai
 * @date: 2018/3/29
 * @description: 查询索引
 */
public class SearchIndex {

    @Test
    public void searchIndex() {

        try {
            Directory directory = FSDirectory.open(new File(CreateIndex.indexDir));
            IndexReader indexReader = DirectoryReader.open(directory);
            IndexSearcher indexSearcher = new IndexSearcher(indexReader);

            Analyzer analyzer = new StandardAnalyzer(Version.LUCENE_4_9);
            QueryParser queryParser = new QueryParser(Version.LUCENE_4_9, "content", analyzer);
            Query query = queryParser.parse("form");

            TopDocs topDocs = indexSearcher.search(query, 10);
            ScoreDoc[] scoreDocs = topDocs.scoreDocs;
            for (ScoreDoc sd : scoreDocs) {
                int docId = sd.doc;
                Document document = indexReader.document(docId);
                System.out.println(document.get("filename"));

            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

* 中文分词的支持
	* 目前有11大中文分词器，本文使用的是IKAnalyzer。
	* 中文分词器不同于英文单词的单个独立性，需要对多义词单词进行识别，这歌就需要基于统计学来判断，比如“请把手拿开”中的“把手”，既可以作为自个名字，也可以作为动词，这里作为动词，所以，有时候，需要通过上下文或者广泛的概率来判断，那个是最可能的。
	* 导包：目前的包管理情况
	![jar包][5]
	* 构建SpringMVC项目结构：
		* 导入Springmvc的8个包：spring7+jstl
		* 配置xml文件：web.xml和spring-servlet.xml
		![enter description here][6]
	* 编写相应的mvc的java文件
	* 这里注意几个问题：
		* webapp转web，需要配置tomcat容器[参考][7]
		* 启动报错Intellij idea 出现错误 error:java: 无效的源发行版: 8[spring3.2对应为jdk1.7][8]
		* (拒绝访问。)：资源文件被占用
		* 流程
		![流程图][9]
		![效果图][10]

[项目源码-第一步-基本网站的搭建Spring][11]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522376629096.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522377097705.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522377229608.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522377636768.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522378234231.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522378656778.jpg
  [7]: https://blog.csdn.net/weixin_38410429/article/details/71773667
  [8]: https://blog.csdn.net/leixingbang1989/article/details/51985601
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522482496532.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522482802915.jpg
  [11]: https://github.com/purice93/lucene.git
