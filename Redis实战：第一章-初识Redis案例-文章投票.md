---
title: Redis实战：第一章-初识Redis案例-文章投票
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

>redis全称REmote DIctionary Server，即远程字典服务，是一个由Salvatore Sanfilippo写的key-value存储系统。Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。它通常被称为数据结构服务器，因为值（value）可以是 字符串(String), 哈希(Map), 列表(list), 集合(sets) 和 有序集合(sorted sets)等类型。
>redis是NoSQL数据库，相对于其他数据库所不同的是，redis并不使用表结构，即并不会提前设计或建立表，而是在使用的时候，直接附带存储结构名。（一般都是字符串，在之后的几章中，为了方便描述，暂时称这种名也为表）

redis特点和应用场景参考博客：[六尺帐篷][1]
redis参考文档：[http://redisdoc.com/index.html][2]
**我理解的优势：**
* 快速存取：相对于一般的关系型数据库，redis数据表之间没有关联，查找和存取都异常的块
* 由于数据存量少，大多数时候是直接内存操作
* 相对于同属nosql的memcached，redis有两个主要优势：
	* 有两种写入硬盘的方式：1、快照-直接将所有数据写入硬盘；2、追加-追加一条数据到硬盘
	* 支持5中数据格式，更加适应多种不同的业务需求，更加灵活
* redis既可以用作主数据库，也可以用作辅助数据库，一般用来作为缓存数据库使用。常见于电商系统，登录系统等
* redis使用主从数据库配置，所以可以有效避免单台服务器的内存访问压力，这个逻辑相当于分布式服务器。

redis初始案例：文章投票（同类如：新闻点赞，网站发帖等）
* 文章投票的逻辑：对于一个网站，当用户访问时，我们希望用户看到的文章尽量是最近发表的或者是点赞最多的，这里就需要对文章进行评分排序；另外对于文章发布时间，我们也需要进行保存，以便提供根据发布时间来查看文章顺序，这两样都是需要redis的有序集合结构。
* 为了防止用户对文章进行多次投票，需要记录一个文章投票用户，使用简单地集合set
* 为了节约内存，一般情况下，文章发不超过一周，将不能进行投票，这里需要使用一个过期时间，或者直接定义时间进行计算。这个例子并不适用于这里，但是可以想象其他场景，比如一个跳蚤平台，当买家发布一周后，此时的交易很可能已经完成或者基本不可能完成了，也就没必要再占用网站页面，就让他自动死去。
* 对于删除文章或者添加文章，虽然redis对没单个操作提供原子性，但是多个操作在多线程情况下，可能会发生错误，所以需要使用redis事务处理，由于本章只是提供案例，所以暂时不使用事务逻辑。

详细逻辑代码如下：

``` stylus
import redis.clients.jedis.Jedis;
import redis.clients.jedis.ZParams;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

/**
 * @author: ZouTai
 * @date: 2018/6/27
 * @description: 第一章:初识redis-例子-对文章进行投票
 */
public class Chapter01 {
    /**
     * 一周总秒数；一次支持加432分；一页包含25篇文章
     */
    private static final int ONE_WEEK_IN_SECONDS = 7 * 86400;
    private static final int VOTE_SCORE = 432;
    private static final int ARTICLES_PER_PAGE = 25;

    public static final void main(String[] args) {
        new Chapter01().run();
    }

    public void run() {
        // 切换到指定的数据库，数据库索引号 index 用数字值指定，以 0 作为起始索引值。
        // 默认使用 0 号数据库。
        Jedis conn = new Jedis("localhost");
        conn.select(15);

        // 1、发表新文章
        String articleId = postArticle(
                conn, "username", "A title", "http://www.google.com");
        System.out.println("We posted a new article with id: " + articleId);
        System.out.println("Its HASH looks like:");

        // 2、输出所有文章
        Map<String, String> articleData = conn.hgetAll("article:" + articleId);
        for (Map.Entry<String, String> entry : articleData.entrySet()) {
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }
        System.out.println();

        // 3、对文章进行投票
        articleVote(conn, "other_user", "article:" + articleId);
        String votes = conn.hget("article:" + articleId, "votes"); // 获取文章投票数
        System.out.println("We voted for the article, it now has votes: " + votes);
        assert Integer.parseInt(votes) > 1;

        // 4、输出所有的文章数据
        System.out.println("The currently highest-scoring articles are:");
        List<Map<String, String>> articles = getArticles(conn, 1);
        printArticles(articles);
        assert articles.size() >= 1;

        // 5、文章分组
        addGroups(conn, articleId, new String[]{"new-group"});
        System.out.println("We added the article to a new group, other articles include:");
        articles = getGroupArticles(conn, "new-group", 1);
        printArticles(articles);
        assert articles.size() >= 1;
    }

    /**
     * @param conn
     * @param user
     * @param title
     * @param link
     * @return 发表新的文章
     */
    public String postArticle(Jedis conn, String user, String title, String link) {
        String articleId = String.valueOf(conn.incr("article:")); // 即文章id从1依次递增（文章表）
        String voted = "voted:" + articleId;
        conn.sadd(voted, user); // (投票表)
        conn.expire(voted, ONE_WEEK_IN_SECONDS); //（设置过期时间为一周）

        long now = System.currentTimeMillis() / 1000;
        String article = "article:" + articleId;
        HashMap<String, String> articleData = new HashMap<String, String>();
        articleData.put("title", title);
        articleData.put("link", link);
        articleData.put("user", user);
        articleData.put("now", String.valueOf(now));// （文章发表时间）
        articleData.put("votes", "1");
        conn.hmset(article, articleData);// （文章详细数据表）

        // 分数score用于对文章进行排序，排序由发表时间和投票数两个因素决定，
        // 默认一篇文章一天投票200次，则每一次投票相当于增加432秒的分数
        // 初始分数为发表时的当前时间（now+432）
        conn.zadd("score:", now + VOTE_SCORE, article); // （分数表-用于排序）
        conn.zadd("time:", now, article); // (时间表-用于通过发表时间查找)

        return articleId;
    }

    public void articleVote(Jedis conn, String user, String article) {
        // 判断投票文章是否过期（一个星期），过期不能投票
        // 判断方法：当前时间-过期时间>文章发表时间(过期)
        long cutoff = (System.currentTimeMillis() / 1000) - ONE_WEEK_IN_SECONDS;
        if (conn.zscore("time:", article) < cutoff) {
            return;
        }

        String articleId = article.substring(article.indexOf(':') + 1);
        if (conn.sadd("voted:" + articleId, user) == 1) { // 一个人只能投一次票，投过票会返回0
            conn.zincrby("score:", VOTE_SCORE, article); // 增加分数
            conn.hincrBy(article, "votes", 1); // 增加文章投票数
        }
    }


    public List<Map<String, String>> getArticles(Jedis conn, int page) {
        return getArticles(conn, page, "score:");
    }

    public List<Map<String, String>> getArticles(Jedis conn, int page, String order) {
        int start = (page - 1) * ARTICLES_PER_PAGE;
        int end = start + ARTICLES_PER_PAGE - 1;

        Set<String> ids = conn.zrevrange(order, start, end);
        List<Map<String, String>> articles = new ArrayList<Map<String, String>>();
        for (String id : ids) {
            Map<String, String> articleData = conn.hgetAll(id);
            articleData.put("id", id);
            articles.add(articleData);
        }

        return articles;
    }

    /**
     * 文章分类/分组
      */
    public void addGroups(Jedis conn, String articleId, String[] toAdd) {
        String article = "article:" + articleId;
        for (String group : toAdd) {
            conn.sadd("group:" + group, article);
        }
    }

    public List<Map<String, String>> getGroupArticles(Jedis conn, String group, int page) {
        return getGroupArticles(conn, group, page, "score:");
    }

    /**
     *
     * @param conn
     * @param group
     * @param page
     * @param order
     * @return
     * 1、zinterstore交集；
     *  (Article:id)与(Article:id-score)交集可以对分组文章进行排序
     *  即("group:" + group, order)-->("group:new-group", "score:")
     * 2、缓存：
     *  zinterstore太耗时，所以使用临时表，进行缓存（表key）
     */
    public List<Map<String, String>> getGroupArticles(Jedis conn, String group, int page, String order) {
        String key = order + group;
        if (!conn.exists(key)) { // 先判断是否有缓存
            ZParams params = new ZParams().aggregate(ZParams.Aggregate.MAX);
            conn.zinterstore(key, params, "group:" + group, order);
            conn.expire(key, 60); // 缓存1分钟，一分钟后删除表
        }
        return getArticles(conn, page, key);
    }

    private void printArticles(List<Map<String, String>> articles) {
        for (Map<String, String> article : articles) {
            System.out.println("  id: " + article.get("id"));
            for (Map.Entry<String, String> entry : article.entrySet()) {
                if (entry.getKey().equals("id")) {
                    continue;
                }
                System.out.println("    " + entry.getKey() + ": " + entry.getValue());
            }
        }
    }
}

```
```
结果如下：
We posted a new article with id: 2
Its HASH looks like:
  link: http://www.google.com
  votes: 1
  title: A title
  user: username
  now: 1530349369

We voted for the article, it now has votes: 2
The currently highest-scoring articles are:
  id: article:2
    link: http://www.google.com
    votes: 2
    title: A title
    user: username
    now: 1530349369
  id: article:1
    link: http://www.google.com
    votes: 2
    title: A title
    user: username
    now: 1530109319
We added the article to a new group, other articles include:
  id: article:2
    link: http://www.google.com
    votes: 2
    title: A title
    user: username
    now: 1530349369
  id: article:1
    link: http://www.google.com
    votes: 2
    title: A title
    user: username
    now: 1530109319

Process finished with exit code 0

```


  [1]: https://www.jianshu.com/p/238372c25669
  [2]: http://redisdoc.com/index.html
