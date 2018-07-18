---
title: Redis实战：第二章-使用redis构建web应用
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>对于一个高并发电商网站，如果使用传统的关系型数据库，由于关系型数据库在并发量达到100万时，效率将大大降低，比如对于一个电商网站，用户登录浏览商品，在很短的时间内，用户可能会浏览很多商品，而这些商品就是用户的兴趣点，为了分析用户的潜在需求，我们需要记录所有的访问数据，难点在于，如果有100万个用户都在这样操作，传统的关系型数据库将很难处理这么多的请求，将造成大量的数据丢失，所以，为了快速响应这些请求，redis的快速读写功能就能使用上。

* redis实现购物车[参考博客][1]
购物车的实现有三种方法：
	* 1直接使用关系型数据库，这样不用担心数据的丢失，但是高并发下速度慢
	* 2使用浏览器客户端自带的cookie，缺点，可能禁用cookie，另外如果用户在没有登陆的情况下，添加购物车，之后又由另外一个人登录，那么此时的购物车就是别人的了
	* 3redis缓存，速度快；缺点浪费大量服务器内存，但是一般这种情况，电商系统还是承受的住的，毕竟资源并不高，别人的文章说的，我想的话，这也就是个字段保存

* 网页缓存
早期的网站开发，前端的网页由于包含后台的数据，整个网页都是动态生成的，比如jsp；但是实际上，对于大部分网页而言，其中90%的网页都是固定不变的或者只改变一次，所以，就需要使用缓存来临时保存这些网页，而不用反复的动态生成而浪费时间

* 数据行缓存，对于一个网页，有时候，网页只有其中的部分数据会变，但是网页整体不变。比如，电商促销活动页面，其中固定放出一定的商品用于促销，此时由于包含了商品信息，数据随时可能改变，所以不能使用上面的网页缓存；但是，如果直接访问数据库，速度又受限，这时就可以使用redis进行数据行缓存，只缓存指定的数据，比如促销的商品信息

**详细逻辑代码：**

``` stylus
import com.google.gson.Gson;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Tuple;

import java.net.MalformedURLException;
import java.net.URL;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

/**
 * @author: ZouTai
 * @date: 2018/6/28
 * @description: 第二章、使用redis构建web应用
 * @create: 2018-06-28 10:06
 */
public class Chapter02 {
    public static void main(String[] args) throws InterruptedException {
        new Chapter02().run();
    }

    private void run() throws InterruptedException {
        // 1、创建连接，选择数据库15
        Jedis conn = new Jedis("localhost");
        conn.select(15);

        // 1、登录和cookie缓存 2、购物车实现 3、网页缓存 4、数据行缓存
        testLoginCookies(conn);
        testShoppingCartCookies(conn);
        testCacheRequest(conn);
        testCacheRows(conn);

    }

    /**
     * 2.3 网页内容缓存
     *
     * @param conn
     */
    private void testCacheRequest(Jedis conn) {
        System.out.println("\n----- testCacheRequest -----");
        String token = UUID.randomUUID().toString();
        CallBack callBack = new CallBack() {
            @Override
            public String call(String request) {
                return "content for " + request;
            }
        };

        // 登录
        updateToken(conn, token, "username", "item3");
        String url = "http://test.com/?item=itemX";
        System.out.println("We are going to cache a simple request against " + url);

        String result = cacheRequest(conn, url, callBack);
        System.out.println("We got initial content:\n" + result);
        System.out.println();
        assert result != null;


        System.out.println("To test that we've cached the request, we'll pass a bad callback");
        String result2 = cacheRequest(conn, url, null);
        System.out.println("We ended up getting the same response!\n" + result2);
        assert result.equals(result2);

        assert !canCache(conn, "http://test.com/");
        assert !canCache(conn, "http://test.com/?item=itemX&_=1234536");
    }

    /**
     * 对请求进行缓存处理
      */
    private String cacheRequest(Jedis conn, String request, CallBack callback) {
        // 不能缓存的请求，直接调用回调函数，返回提示信息
        // 回调函数，即通过外部类来实现计算并返回给当前函数
        if (!canCache(conn, request)) {
            return callback != null ? callback.call(request) : null;
        }

        String pageKey = "cache:" + hashRequest(request); // 将请求转为hash码
        String content = conn.get(pageKey);  // 获取连接内容（若存在）

        // 若不存在，则写入数据库
        if (content == null && callback != null) {
            content = callback.call(request);
            // 将值 value 关联到 key ，并将 key 的生存时间设为 seconds
            conn.setex(pageKey, 300, content);
        }
        return content;

    }

    private String hashRequest(String request) {
        return String.valueOf(request.hashCode());
    }

    /**
     * 判断请求是否能被缓存
      */
    private boolean canCache(Jedis conn, String request) {
        try {
            URL url = new URL(request);
            HashMap<String, String> params = new HashMap<String, String>();
            // getQuery获取url参数，这里即“item=itemX”
            if (url.getQuery() != null) {
                for (String param : url.getQuery().split("&")) {
                    // limit设置分割份数，即强制默认两份，如果出现“item=itemX=2”，做“item”和“itemX=2”处理
                    String[] pair = param.split("=", 2);
                    // 将请求参数添加到parames，如果没有设置参数的值，即length==1，则设置为null
                    params.put(pair[0], pair.length == 2 ? pair[1] : null);
                }
            }

            String itemId = extractItemId(params); // 获取参数的值itemX
            // 如果参数值为null或者不进行缓存，则返回false
            if (itemId == null || isDynamic(params)) {
                return false;
            }

            // zrank:有序集成员按 score 值递增(从小到大)顺序排列
            Long rank = conn.zrank("viewed:", itemId);
            // 有商品被访问过：即网页被访问
            return rank != null && rank < 10000;
        } catch (MalformedURLException mue) {
            return false;
        }
    }

    /**
     * 用于测试，即请求参数含有"_"的，不进行缓存
     */
    private boolean isDynamic(HashMap<String, String> params) {
        return params.containsKey("_");
    }

    private String extractItemId(HashMap<String, String> params) {
        return params.get("item");
    }

    /**
     * 2、使用缓存实现购物车
      */
    private void testShoppingCartCookies(Jedis conn) throws InterruptedException {
        System.out.println("\n----- testShoppingCartCookies -----");
        String token = UUID.randomUUID().toString();
        System.out.println("We'll refresh our session...");
        updateToken(conn, token, "username", "itemX");
        System.out.println("And add an item to the shopping cart");

        // 2.1 添加到购物车
        addToCart(conn, token, "item3", 3);

        // 2.2 输出所有的购物车
        Map<String, String> r = conn.hgetAll("cart:" + token);
        System.out.println("Our shopping cart currently has:");
        for (Map.Entry<String, String> entry : r.entrySet()) {
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }
        System.out.println();
        assert r.size() >= 1;

        // 2.3 删除所有的对话
        System.out.println("Let's clean out our sessions and carts");
        CleanFullSessionsThread thread = new CleanFullSessionsThread(0);
        thread.start();
        Thread.sleep(1000);
        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()) {
            throw new RuntimeException("The clean sessions thread is still alive?!?");
        }

        // 2.4 输出购物车数据
        r = conn.hgetAll("cart:" + token);
        System.out.println("Our shopping cart now contains:");
        for (Map.Entry<String, String> entry : r.entrySet()) {
            System.out.println("  " + entry.getKey() + ": " + entry.getValue());
        }
        assert r.size() == 0;

        // 3、 网页缓存
        testCacheRows(conn);
    }

    /**
     * 缓存行数据
     * @param conn
     */
    private void testCacheRows(Jedis conn) throws InterruptedException {
        System.out.println("\n----- testCacheRows -----");
        System.out.println("First, let's schedule caching of itemX every 5 seconds");
        scheduleRowCache(conn, "item3", 5);  // 缓存5分钟

        System.out.println("Our schedule looks like:");
        // Tuple含有一个score字段用于保存分数比较
        Set<Tuple> scheduleSet = conn.zrangeWithScores("schedule:", 0, -1); // 查找所有分数大于0的，max参数无
        System.out.println("按过期时间进行排序输出：");
        for (Tuple tuple : scheduleSet) {
            System.out.println(" " + tuple.getElement() + "," + tuple.getScore());
        }
        assert scheduleSet.size() != 0;

        System.out.println("We'll start a caching thread that will cache the data...");
        CacheRowsThread thread = new CacheRowsThread();
        thread.start();
        Thread.sleep(1000);
        System.out.println("Our cached data looks like:");
        String r = conn.get("inv:itemX");
        System.out.println(r);
        assert r != null;
        System.out.println();

        System.out.println("We'll check again in 5 seconds...");
        Thread.sleep(5000);
        System.out.println("Notice that the data has changed...");
        String r2 = conn.get("inv:itemX");
        System.out.println(r2);
        System.out.println();
        assert r2 != null;
        assert !r.equals(r2);

        System.out.println("Let's force un-caching");
        scheduleRowCache(conn, "itemX", -1);
        Thread.sleep(1000);
        r = conn.get("inv:itemX");
        System.out.println("The cache was cleared? " + (r == null));
        assert r == null;

        thread.quit();
        Thread.sleep(2000);
        if (thread.isAlive()){
            throw new RuntimeException("The database caching thread is still alive?!?");
        }
    }

    private void scheduleRowCache(Jedis conn, String item, int delay) {
        // 分别记录缓存总时间和缓存截止时间
        conn.zadd("delay:", delay, item);
        conn.zadd("schedule:", System.currentTimeMillis() / 1000, item);
    }

    /**
     * 添加到购物车
      */
    private void addToCart(Jedis conn, String token, String item, int count) {
        if (count <= 0) {
            conn.hdel("cart:" + token, item); // 用于移除购物车，直接移除商品
        } else {
            conn.hset("cart:" + token, item, String.valueOf(count));
        }
    }

    /**
     * 1、登录和cookie缓存
     * @param conn
     * @throws InterruptedException
     */
    private void testLoginCookies(Jedis conn) throws InterruptedException {
        System.out.println("test LoginCookies:");
        // 1.1 生成令牌：代表一次登录（未退出）
        String token = UUID.randomUUID().toString();

        // 1.2 一次商品浏览（可以多次）
        updateToken(conn, token, "username", "item1"); // 访问商品，可以多次
        System.out.println("We just logged-in/updated token: " + token);
        System.out.println("For user: 'username'");
        System.out.println();

        // 1.3 查找当前登录用户
        System.out.println("What username do we get when we look-up that token?");
        String rs = checkToken(conn, token);
        System.out.println(rs);
        System.out.println();
        assert rs != null;


        // 1.4 启动cookie检查线程，防止cookie过多
        System.out.println("Let's drop the maximum number of cookies to 0 to clean them out");
        System.out.println("We will start a thread to do the cleaning, while we stop it later");
        CleanSessionsThread thread = new CleanSessionsThread(0);
        thread.start();
        Thread.sleep(1000);
        thread.quit(); // 停止线程
        Thread.sleep(2000);
        if (thread.isAlive()) {
            throw new RuntimeException("The clean sessions thread is still alive?!?");
        }

        // 1.5 检查是否还有token
        long s = conn.hlen("login:");
        System.out.println("The current number of sessions still available is: " + s);
        assert s == 0;
    }

    /**
     * 注意：这里有三个表：
     * 1、login：记录登录标记-登录用户（token-user）
     * 2、recent：记录登录标记-登陆时间（token-timestamp）
     * 3、viewed：记录商品被访问次数（item-int）
     * 4、"viewed:" + token：记录商品被访问时间（item-timestamp）
     *
     * @param conn
     * @param token
     * @param user
     * @param item  浏览商品，生成令牌token记录
     */
    private void updateToken(Jedis conn, String token, String user, String item) {
        Long timestamp = System.currentTimeMillis() / 1000;
        // 2、写入登录表，表名为login，key-token，value-user
        conn.hset("login:", token, user); // 登录令牌对应user
        conn.zadd("recent:", timestamp, token); // 添加到当前登录的cookie目录
        if (item != null) {
            conn.zadd("viewed:" + token, timestamp, item); // 增加一个商品时间戳，
            // 移除旧的记录，只保留用户最近访问的25个商品
            // 注意这里反着取，即删除0，-mix，-mix+1，-mix+2....-27,-26
            conn.zremrangeByRank("viewed:" + token, 0, -26);
            // 新建当前商品访问分数为-1，每访问一次，分数-1；
            // 访问次数越多分数越少，排序越靠前
            conn.zincrby("viewed:", -1, item);
        }
    }

    private String checkToken(Jedis conn, String token) {
        return conn.hget("login:", token);
    }

    private class CleanSessionsThread extends Thread {
        private Jedis conn;
        private int limit; // 总令牌数量
        private boolean quit; // 是否停止

        public CleanSessionsThread(int limit) {
            this.conn = new Jedis("localhost");
            conn.select(15);
            this.limit = limit;
        }

        @Override
        public void run() {
            while (!quit) {
                long size = conn.zcard("recent:"); // 获取当前所有的cookie令牌
                // 未超过cookie数量限制
                if (size <= limit) {
                    try {
                        sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    continue; // 继续检查
                }
                // 删除多余的，但是删除量不大于100
                // 由于是按时间排序的，所以是从前往后删除
                long endIndex = Math.min(size - limit, 100);
                Set<String> tokenSet = conn.zrange("recent:", 0, endIndex - 1);
                String[] tokens = tokenSet.toArray(new String[tokenSet.size()]);

                // 为即将删除的令牌构建键名，用于删除
                conn.zrem("recent:", tokens); // 1删除登录记录-时间
                conn.hdel("login:", tokens); // 2删除登录记录-用户
                // 删除登录记录对应的所有商品访问信息表
                ArrayList<String> sessionsKeys = new ArrayList();
                for (String token : tokens) {
                    sessionsKeys.add("viewed:" + token);
                }
                conn.del(sessionsKeys.toArray(new String[sessionsKeys.size()]));
            }
        }

        public void quit() {
            this.quit = true;
        }
    }

    private class CleanFullSessionsThread extends Thread {
        private Jedis conn;
        private int limit;
        private boolean quit;

        public CleanFullSessionsThread(int limit) {
            this.conn = new Jedis("localhost");
            this.conn.select(15);
            this.limit = limit;
        }

        public void quit() {
            quit = true;
        }

        @Override
        public void run() {
            while (!quit) {
                long size = conn.zcard("recent:");
                if (size <= limit) {
                    try {
                        sleep(1000);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                    }
                    continue;
                }

                long endIndex = Math.min(size - limit, 100);
                Set<String> sessionSet = conn.zrange("recent:", 0, endIndex - 1);
                String[] sessions = sessionSet.toArray(new String[sessionSet.size()]);

                ArrayList<String> sessionKeys = new ArrayList<String>();
                for (String sess : sessions) {
                    sessionKeys.add("viewed:" + sess);
                    sessionKeys.add("cart:" + sess);
                }

                conn.del(sessionKeys.toArray(new String[sessionKeys.size()]));
                conn.hdel("login:", sessions);
                conn.zrem("recent:", sessions);
            }
        }
    }

    private class CacheRowsThread extends Thread {
        private Jedis conn;
        private boolean quit;

        public CacheRowsThread() {
            this.conn = new Jedis("localhost");
            conn.select(15);
        }

        @Override
        public void run() {
            while (!quit) {
                Set<Tuple> sSet = conn.zrangeWithScores("schedule:", 0, -1);
                Tuple next = sSet.size() > 0 ? sSet.iterator().next() : null;
                long now = System.currentTimeMillis() / 1000;
                // 如果为null或者没有过期，则等待重复检查
                if (next == null || next.getScore() > now) {
                    try {
                        sleep(50);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    continue;
                }

                // 如果有过期，则删除对应数据和表
                String item = next.getElement();
                double delay = conn.zscore("schedule:", item);
                if (delay <= 0) {
                    conn.zrem("delay:", item);
                    conn.zrem("schedule:", item);
                    continue;
                }

                // 封装一个网页数据类
                Inventory row = new Inventory(item);
                conn.zadd("delay:", now + delay, item);
                Gson gson = new Gson();
                conn.set("inv:" + item, gson.toJson(row), item); // 设置对应的网页

            }
        }

        public void quit() {
            this.quit = true;
        }


    }

    private class Inventory {
        private String item;
        private String data;
        private long time;

        public Inventory(String item) {
            this.item = item;
            this.data = "data to cache...";
            this.time = System.currentTimeMillis() / 1000;
        }
    }

    public interface CallBack {
        public String call(String request);
    }
}

```


  [1]: https://www.cnblogs.com/wang-meng/p/5854773.html
