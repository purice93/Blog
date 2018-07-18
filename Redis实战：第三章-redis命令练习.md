---
title: Redis实战：第三章-redis命令练习
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>redis有5种数据结构，分别为字符串、列表、集合、散列、有序集合；相当于java中的String、list、set、hashmap、sortset(z)；另外redis不仅仅只有有序集合提供排序，对于另外四种结构，redis还提供了sort命令，可以更具指定的格式进行排序；redis也支持事务处理、发布和订阅功能，并且可以设置过期时间。

``` stylus
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPubSub;
import redis.clients.jedis.Pipeline;
import redis.clients.jedis.Response;
import redis.clients.jedis.SortingParams;
import redis.clients.jedis.ZParams;
import java.util.HashMap;
import java.util.List;
import java.util.Set;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * @author: ZouTai
 * @date: 2018/6/29
 * @description: Redis命令
 * @create: 2018-06-29 09:05
 */
public class Chapter03 {
    public static void main(String[] args) throws InterruptedException {
        new Chapter03().run();
    }

    private void run() throws InterruptedException {
        Jedis conn = new Jedis("localhost");
        conn.select(15);

        testString(conn);
    }

    private void testString(final Jedis conn) throws InterruptedException {
        /**
         * 1.1 字符串简单操作
         */
        //conn.set("key","0");
        conn.del("key"); // 删除数据
        String value = conn.get("key"); // 获取值
        long number = conn.incr("key"); // 自增1，并获取最新值
        conn.incrBy("key", 10); // 增加指定值
        conn.set("key", "15"); // 设置值
        conn.incr("key"); // 自增1
        assert number == 1;


        long len = conn.append("key", "hello");
        value = conn.get("key");
        conn.substr("key", 0, 3);
        conn.setrange("key", 5, "world"); // 从偏移量开始的值设置为指定值
        conn.setbit("key2", 2, "1");
        conn.setbit("key2", 7, "1");
        value = conn.get("key2");
        // "!"的编码为33，即0100001
        System.out.println(value);


        /**
         * 2、列表操作
         */
        len = conn.rpush("list-key", "last");
        len = conn.lpush("list-key", "first");
        conn.lpush("list-key", "new list");
        List<String> arr = conn.lrange("list-key", 0, -1);
        value = conn.lpop("list-key");
        value = conn.rpop("list-key");
        conn.rpush("list-key", "a", "b", "c");
        conn.ltrim("list-key", 2, -1); // 截取指定片段，包含偏移量
        arr = conn.lrange("list-key", 0, -1);

        // 2.2 列表阻塞操作
        conn.del("list");
        conn.del("list2");
        conn.rpush("list", "item1");
        conn.rpush("list", "item2");
        conn.rpush("list2", "item3");

        // 超时参数 timeout 接受一个以秒为单位的数字作为值。
        // 超时参数设为 0 表示阻塞时间可以无限期延长(block indefinitely) 。
        conn.brpoplpush("list2", "list", 1);
        arr = conn.lrange("list", 0, -1);
        System.out.println(arr); // [item3, item1, item2]

        // 若list2为null，则返回null
        value = conn.brpoplpush("list2", "list", 1);
        assert value == null;

        // 从左向右检查所有列表，每次返回一个元素
        arr = conn.blpop(1, "list", "list2");
        System.out.println(arr); // [list, item3]


        /**
         * 3、集合
         */
        long isOk = 0;
        boolean success = false;
        Long count = 0L;
        conn.sadd("set-key", "a", "b", "c");

        // 移除包含的元素，成功则返回1；如果一个没有，则返回0
        conn.srem("set-key", "c", "d");
        isOk = conn.srem("set-key", "c");
        count = conn.scard("set-key"); // 返回集合数量
        Set<String> sets = conn.smembers("set-key");

        // 转移元素
        conn.smove("set-key", "set-key2", "a");
        System.out.println(conn.smembers("set-key2"));


        // 3.1 组合和关联集合
        conn.sadd("skey1", "a", "b", "c", "d");
        conn.sadd("skey2", "c", "d", "e", "f");
        sets = conn.sdiff("skey1", "skey2");// 1中有，2中没有
        sets = conn.sinter("skey1", "skey2"); // 交集
        sets = conn.sunion("skey1", "skey2"); // 并集


        /**
         * 4、散列
         */
        conn.hmset("hash-key", new HashMap<String, String>() {
            {
                put("k1", "v1");
                put("k2", "v2");
                put("k3", "v3");
            }
        });
        arr = conn.hmget("hash-key", "k1", "k2");
        len = conn.hlen("hash-key");
        isOk = conn.hdel("hash-key", "k1"); // 返回0-1

        // 4.2 批量操作
        final StringBuffer lStr = new StringBuffer();
        for (int i = 0; i < 1000; i++) {
            lStr.append("a");
        }
        conn.hmset("hash-key2", new HashMap<String, String>() {
            {
                put("short", "hello");
                put("long", lStr.toString());
            }
        });

        conn.hkeys("hash-key2");
        conn.hexists("hash-key2", "short");
        conn.hincrBy("hash-key2", "num", 1);
        conn.hexists("hash-key2", "num");


        /**
         * 5、有序集合
         */
        conn.zadd("zset-key", new HashMap<String, Double>() {
            {
                put("a", 3.0);
                put("b", 2.0);
                put("c", 1.0);
            }
        });
        double num = 0;
        len = conn.zcard("zset-key");
        num = conn.zincrby("zset-key", 3, "c");
        num = conn.zscore("zset-key", "b");
        num = conn.zrank("zset-key", "a");
        num = conn.zcount("zset-key", 0, 3); // 指定分数内的数量

        isOk = conn.zrem("zset-key", "c");
        sets = conn.zrangeByScore("zset-key", 0, -1);

        // 5.2 交集和并集

        conn.zadd("zset1", new HashMap<String, Double>() {
            {
                put("a", 3.0);
                put("b", 2.0);
                put("c", 1.0);
            }
        });
        conn.zadd("zset2", new HashMap<String, Double>() {
            {
                put("b", 4.0);
                put("c", 5.0);
                put("d", 6.0);
            }
        });
        isOk = conn.zinterstore("zsetI", "zset1", "zset2");
        sets = conn.zrange("zsetI", 0, -1);

        isOk = conn.zunionstore("zsetU", "zset1", "zset2");
        sets = conn.zrange("zsetU", 0, -1);

        // 设置为取最小数，而不是相加
        ZParams zParams = new ZParams();
        zParams.aggregate(ZParams.Aggregate.MIN);
        isOk = conn.zunionstore("zsetU0", zParams, "zset1", "zset2");
        sets = conn.zrange("zsetU0", 0, -1);

        conn.zadd("zset3", new HashMap<String, Double>() {
            {
                put("e", 11.0);
            }
        });

        isOk = conn.zunionstore("zsetU2", "zset1", "zset2", "zset3");
        sets = conn.zrange("zsetU2", 0, -1);


        /**
         * 6、发布和订阅
         */
        runPubsub(conn);

        /**
         * 7、排序：sort命令
         */
        conn.rpush("sort-input", "23", "15", "110", "7");
        arr = conn.sort("sort-input");
        arr = conn.sort("sort-input", new SortingParams().alpha());
        conn.hset("d-7", "field", "5");
        conn.hset("d-15", "field", "1");
        conn.hset("d-23", "field", "9");
        conn.hset("d-110", "field", "3");
        arr = conn.sort("sort-input", new SortingParams().by("d-*->field"));
        SortingParams sps = new SortingParams();
        sps.by("d-*->field");
        sps.get("d-*->field");
        arr = conn.sort("sort-input", sps);


        /**
         * 7、事务处理
         */
        conn.del("notrans");
        conn.del("trans");
        System.out.println("7、事务处理：");
        ExecutorService singleThreadPool = new ThreadPoolExecutor(6, 10, 5, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
        Runnable noTRunnable = new Runnable() {
            public void run() {
                System.out.println(conn.incr("notrans"));
                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
//                System.out.println(conn.incrBy("notrans", -1)); 报线程安全错误
            }
        };
        Runnable TRunnable = new Runnable() {
            public void run() {
                // 为什么要新建连接？？
                Jedis conn = new Jedis("localhost");
                conn.select(15);

                Pipeline pipeline = conn.pipelined();
                pipeline.multi();//开启事务
                pipeline.incr("trans"); // 步骤1
                try {
                    Thread.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                pipeline.incrBy("trans", -1);// 步骤2


                /**
                 * 这里需要注意，pipeline.exec()结果保存了每一步的数据，所以输出会有两个值，
                 * 分别为步骤1和步骤2的结果
                 */
                Response<List<Object>> txresult = pipeline.exec();//提交事务
                pipeline.sync();//关闭pipeline
                //结束pipeline，并开始从相应中获得数据
                List<Object> responses = txresult.get();
                if (responses == null || responses.isEmpty()) {
                    throw new RuntimeException("Pipeline error: no response...");
                }
                // 只输出最总的结果
                System.out.println(responses.get(responses.size() - 1));
//                for(Object resp : responses){
//                    System.out.println("Response:" + resp.toString());//注意，此处resp的类型为Long
//                }
            }
        };
        System.out.println("不使用事务：");
        for (int i = 0; i < 3; i++) {
            singleThreadPool.execute(noTRunnable);
        }
        Thread.sleep(5);
        System.out.println("-----------");


        Thread.sleep(10);
        System.out.println("使用事务：");
        for (int i = 0; i < 3; i++) {
            singleThreadPool.execute(TRunnable);
        }
        Thread.sleep(5);


        /**
         * 8、为键设置过期时间
         */
        conn.set("key", "value");
        value = conn.get("key");
        conn.expire("key", 2);
        Thread.sleep(2);
        value = conn.get("key");

        conn.set("key2", "value2");
        conn.expire("key2", 100);// 注意sleep时间为millis不是second
        Thread.sleep(3000);
        long time = conn.ttl("key2"); // 查看还有多久过期
        System.out.println("过期时间还剩：" + time);

    }

    private void runPubsub(final Jedis conn) throws InterruptedException {
        // 建议使用线程池
        ExecutorService singleThreadPool = new ThreadPoolExecutor(6, 10, 5, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
        Runnable myRunnable = new Runnable() {
            Jedis conn1 = new Jedis();

            @Override
            public void run() {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                for (int i = 0; i < 3; i++) {
                    conn1.publish("channel", String.valueOf(i));
                    try {
                        Thread.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        singleThreadPool.execute(myRunnable);
        JedisPubSub pubSub = new JedisPubSub() {
            int count = 0;

            @Override
            public void onMessage(String channel, String message) {
                count++;
                System.out.println("channel:" + channel + "---" + "receives message :" + message);
                // 收取两个信息后，关闭订阅
                if (count == 2) {
                    unsubscribe(); // 内部会自动调用onUnsubscribe函数
                }
                //this.unsubscribe();//取消订阅
            }

            @Override
            public void onSubscribe(String channel, int subscribedChannels) {
                System.out.println("channel:" + channel + "---" + "is been subscribed:" + subscribedChannels);
            }

            @Override
            public void onUnsubscribe(String channel, int subscribedChannels) {
                System.out.println("channel:" + channel + "---" + "is been unsubscribed:" + subscribedChannels);

            }
        };
        conn.subscribe(pubSub, "channel");
    }
}

```


