---
title: 海量搜索服务架构搭建2-SolrCloud集群搭建
tags: zookeeper,solr,集群,SolrCloud
grammar_cjkRuby: true
---
>平台：linux

**第一步，安装solr**

* 1.解压solr，路径solr-4.9.1\example\webapps的solr.war拷贝到tomcat的webapp下面启动
* 2.将solr-4.9.1\example\lib\ext下面的jar包拷贝到solr在tomcat中解压的目录的WEB-INFO下面的lib里面
* 3.修改WEB-INFO下面的web.xml里面修改solr/home，这里要新建一个solrhome的路径，
注意，把注释去掉
![web.xml修改][1]
* 4.在新配置的home目录下讲原来解压的solr-4.9.1\example\solr下的所有文件拷贝过来
* 5.再启动tomcat，访问http://192.168.116.7:8080/solr/
![访问默认界面][2]
>注意，服务tomcat需要新开，不能配置其它干扰
>linux虚拟机需要关闭防火墙

**第二步、直接使用solr开发搜索应用**

* 配置solr文档，直接在线测试
![document][3]
* 开发应用
>由于solr是嵌入到虚拟机的tomcat中，所以需要访问虚拟机中的solr。而应用开发使用window平台。
	* 配置pom.xml，添加solr相关的jar包
	

``` stylus
<!-- solr客户端 -->
      <dependency>
          <groupId>org.apache.solr</groupId>
          <artifactId>solr-solrj</artifactId>
          <version>4.9.1</version>
      </dependency>
      <dependency>
          <groupId>org.apache.httpcomponents</groupId>
          <artifactId>httpclient</artifactId>
          <version>4.4.1</version>
      </dependency>
      <dependency>
          <groupId>org.apache.httpcomponents</groupId>
          <artifactId>httpcore</artifactId>
          <version>4.4.1</version>
      </dependency>
      <dependency>
          <groupId>org.apache.httpcomponents</groupId>
          <artifactId>httpmime</artifactId>
          <version>4.4.1</version>
      </dependency>
      <dependency>
          <groupId>org.noggit</groupId>
          <artifactId>noggit</artifactId>
          <version>0.6</version>
      </dependency>
      <dependency>
          <groupId>commons-io</groupId>
          <artifactId>commons-io</artifactId>
          <version>1.4</version>
      </dependency>
      <dependency>
          <groupId>org.codehaus.woodstox</groupId>
          <artifactId>stax2-api</artifactId>
          <version>3.1.4</version>
      </dependency>
      <dependency>
          <groupId>org.codehaus.woodstox</groupId>
          <artifactId>woodstox-core-asl</artifactId>
          <version>4.4.1</version>
      </dependency>
      <dependency>
          <groupId>org.slf4j</groupId>
          <artifactId>slf4j-simple</artifactId>
          <version>1.7.7</version>
      </dependency>
      <dependency>
          <groupId>commons-logging</groupId>
          <artifactId>commons-logging</artifactId>
          <version>1.0.4</version>
      </dependency>
```


----------
* 问题：IDEA的@Test注解 cannot resolve:版本不够
![版本不够][4]
* 测试
``` stylus
import org.apache.solr.client.solrj.SolrQuery;
import org.apache.solr.client.solrj.SolrServer;
import org.apache.solr.client.solrj.impl.HttpSolrServer;
import org.apache.solr.client.solrj.response.QueryResponse;
import org.apache.solr.common.SolrDocument;
import org.apache.solr.common.SolrDocumentList;
import org.apache.solr.common.SolrInputDocument;
import org.junit.Test;

public class SolrJTest {

    @Test
    public void addDocument() throws Exception {
        //创建一连接
        SolrServer solrServer = new HttpSolrServer("http://192.168.116.7:8080/solr");
        //创建一个文档对象
        SolrInputDocument document = new SolrInputDocument();
        document.addField("id", "test001");
        document.addField("title", "贸易战");
        document.addField("context", "中国与美国的贸易战已经...");
        //把文档对象写入索引库
        solrServer.add(document);
        //提交
        solrServer.commit();
    }

    @Test
    public void deleteDocument() throws Exception {
        //创建一连接
        SolrServer solrServer = new HttpSolrServer("http://192.168.116.7:8080/solr");
        //两种删除方式
        solrServer.deleteById("test001");
        //solrServer.deleteByQuery("*:*");
        solrServer.commit();
    }

    @Test
    public void queryDocument() throws Exception {
        SolrServer solrServer = new HttpSolrServer("http://192.168.116.7:8080/solr");
        //创建一个查询对象
        SolrQuery query = new SolrQuery();
        //设置查询条件
        query.setQuery("*:*");
        query.setStart(20);
        query.setRows(50);
        //执行查询
        QueryResponse response = solrServer.query(query);
        //取查询结果
        SolrDocumentList solrDocumentList = response.getResults();
        System.out.println("共查询到记录：" + solrDocumentList.getNumFound());
        for (SolrDocument solrDocument : solrDocumentList) {
            System.out.println(solrDocument.get("id"));
            System.out.println(solrDocument.get("title"));
            System.out.println(solrDocument.get("context"));
        }
    }
}
```


**构建SolrCloud集群**
>SolrCloud(solr 云)是Solr提供的分布式搜索方案，当你需要大规模，容错，分布式索引和检索能力时使用 SolrCloud。当一个系统的索引数据量少的时候是不需要使用SolrCloud的，当索引量很大，搜索请求并发很高，这时需要使用SolrCloud来满足这些需求。

![SolrCloud架构图][5]

* 物理结构
三个Solr实例（ 每个实例包括两个Core），组成一个SolrCloud。
* 逻辑结构
索引集合包括两个Shard（shard1和shard2），shard1和shard2分别由三个Core组成，其中一个Leader两个Replication，Leader是由zookeeper选举产生，zookeeper控制每个shard上三个Core的索引数据一致，解决高可用问题。
用户发起索引请求分别从shard1和shard2上获取，解决高并发问题。
* SolrCloud搭建
![架构配置实例][6]
* 环境配置
	* CentOS-6.5-i386-bin-DVD1.iso
	* jdk-7u72-linux-i586.tar.gz
	* apache-tomcat-7.0.57.tar.gz
	* zookeeper-3.4.6.tar.gz
	* solr-4.10.3.tgz
	* 服务器7台: 
		* zookeeper三台：192.168.0.5，192.168.0.6，192.168.0.7
		* Solr四台：192.168.0.1，192.168.0.2，192.168.0.3，192.168.0.4

* 第一步、zookeeper集群安装
	* 解压zookeeper 安装包`tar -zxvf zookeeper-3.4.6.tar.gz`将zookeeper-3.4.6拷贝到/opt/sxt/soft下并将目录 名改为zookeeper至此zookeeper的安装目录为/usr/local/zookeeper
	* 进入zookeeper文件夹,创建data 和logs，创建目录并赋于写权限，指定zookeeper的数据存放目录和日志目录
	* 拷贝zookeeper配制文件zoo_sample.cfg，拷贝zookeeper配制文件zoo_sample.cfg并重命名zoo.cfg `cp zoo_sample.cfg zoo.cfg`，修改zoo.cfg配置
![zoo.cfg配置][7]
	* 进入data（即上文的/opt/zookeeper）文件夹 建立对应的myid文件,例如server.1为192.168.0.5则 data文件夹下的myid文件内容为1;server.2为192.168.0.6则 data文件夹下的myid文件内容为2;依此类推
	* 启动全部：进入/usr/local/zookeeper /bin`./zkServer.sh start`；查看集群状态`./zkServer.sh status `。**刚启动可能会有错误，应为各服务器之间需要进行通信判断，等待一段时间，集群中其他节点一并起来后就正常了**
	![zookeeper配置完成效果][8]
	* 将所有的服务器都配置带solr的tomcat（参考上面的配置）
	![三台solr结果][9]
	* **solrCloud部署**
		* 这里将solr.zip文件直接复制到linux中，使用unzip解压（不能解压后在复制，这样文本文件在复制时，系统不一致会出错）
		* 执行下边的命令将/home/solr/conf下的配置文件上传到zookeeper：`sh /opt/sxt/soft/solr-4.9.1/example/scripts/cloud-scripts/zkcli.sh -zkhost 192.168.116.5:2181,192.168.116.6:2181,192.168.116.7:2181 -cmd upconfig -confdir /opt/sxt/soft/solr-4.9.1/example/solr/collection1/conf -confname myconf -solrhome /opt/sxt/soft/solr-4.9.1example/solr`**主要就是修改是哪个路径和三个ip**
		![运行结果][10]
		![验证结果][11]
		* 最后一步：每一台solr和zookeeper关联
		修改每一台solr的tomcat 的 bin目录下catalina.sh文件中加入DzkHost指定zookeeper服务器地址：`JAVA_OPTS="-DzkHost=192.168.116.5:2181,192.168.116.6:2181,192.168.116.7:2181"`,关闭tomcat，重启
		![访问验证][12]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522498773156.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522498807139.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522554426100.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522552060013.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522556782088.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522556956510.jpg
  [7]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522577656082.jpg
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522579749389.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522581110225.jpg
  [10]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522584034164.jpg
  [11]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522584365685.jpg
  [12]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1522585918078.jpg
