---
title: Redis实战：第五章-使用Redis构建支持程序
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

>本章主要讲解redis的使用案例，相对于以往的技术，redis在这些领域将大大简化或者提高程序的便利和稳定。比如日志记录，相对于以往的文件记录方式将更加灵活，便于数据操作，

1. 日志记录
>以往的日志记录采用本地文件存储的方式，这种方式有一种弊端，由于是文本存储，各个服务器之间很难协调，很难对多个数据文件进行聚合，导致之后的数据分析，将显得很不方便，相对而言，由于redis数据库可以在不同的服务器之间通信，加上自带数据处理函数，所以将更加方便。

对不同的数据分析，不同的日志有不同的好处，为了方便管理或者处理，需要对相应的日志做分类处理。比如对于一个网站而言，日志是很多的，如果大量记录是不现实的，但是，很多情况下，比如我们服务器突然出现问题，或者说现在我们需要立刻查看最近的操作的数据情况，这是，我们就可以使用最新的日志，这些日志只会记录最近发生的响应，而不会记录很久之前的数据情况，这样就便于我们查错，或者查看最近的操作情况。

* 最新日志：分别记录最近的debug\info\warning\error\critical等日志情况，便于查看（这里使用的是列表存储，方便添加和删除日志）
* 常见日志：常见日志即我们认为重要的日志，比如用户的操作，加入购物车，购买等信息。这些都是很重要的日志信息，可用于数据挖掘。但是文章中并没有给出这个例子，而是，对不同的日志信息进行排序，对每个动作进行打分，从而只保存经常操作的那些动作，即表明这些操作是最重要的，有很高的商业价值。（同样是列表）
2. 计数器和数据统计
>计数器是什么？这里的计数器并不是电子电路里面的时钟计数器，而是一种数据记录，比如用户的登陆次数，百度上某个医疗广告的点击量，整个百度网页每个渲染页面的点击量等等，这些都是很重要的价值信息。能够从中发掘大量的商业价值。
>数据统计。在统计学中，有几个重要的数据衡量指标，分别是最大值、最小值、平均值、数量、总分数、标准差等等；比如对一个网站进行优化，我么你需要记录网站各个页面的响应时间，这些数据有利于我们有正对性的进行优化。
	
* 本行计数器案例主要模拟，分别在1秒、5秒、一分钟、一小时等，网站的点击量，由于需要进行排序，所以使用的是有序表和哈希集结合
* 数据统计主要是模拟网页的加载时间

3. IP地址等excel本地数据查询
>我们经常会使用搜索，但是这里搜索并不是百度搜索，而是有现成的数据。比如我们一个存储数据的excel表，亦或是文本等等。例如，有时候我们会查询某个ip所对应的网址，但是这个数据很大，如果存储在一般的关系型数据库，查询将会有点慢，因此，这时候就可以使用redis数据库，大大加快查询速度。

4. 服务配置信息动态更改
>开发一个应用会存在很多配置文件，只写配置文件并不是一成不变，以往的人工配置，会出现，一旦一个配置文件发生改变，就需要人工取重新配置，这样大大降低了效率。因此，我们这里考虑使用程序自动化配置，即将配置信息保存在redis中，将其写入到程序中作为守护线程，每当配置文件发生改变都会自动进行配置

* 文中的样例为：redis配置redis的连接配置，会将redis的连接信息保存在redis中，自动检查新的配置文件，然后更新配置到程序中

详细代码：

``` stylus
import com.google.gson.Gson;
import com.google.gson.reflect.TypeToken;
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;
import org.javatuples.Pair;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.Pipeline;
import redis.clients.jedis.Transaction;
import redis.clients.jedis.Tuple;
import redis.clients.jedis.ZParams;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileReader;
import java.io.IOException;
import java.text.Collator;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

/**
 * @author: ZouTai
 * @date: 2018/7/6
 * @description:
 * @create: 2018-07-06 15:23
 */
public class Chapter05 {
    public static final String INFO = "info";
    public static final SimpleDateFormat TIMESTAMP =
            new SimpleDateFormat("EEE MMM dd HH:00:00 YYYY");
    public static final SimpleDateFormat ISO_FORMAT =
            new SimpleDateFormat("yyyy-MM-dd'T'HH:00:00");
    public static final Collator COLLATOR = Collator.getInstance();
    private static final int[] PRECISION = new int[]{1, 5, 60, 300, 3600, 18000, 86400};

    public static void main(String[] args) throws InterruptedException {
        new Chapter05().run();
    }

    private void run() throws InterruptedException {
        Jedis conn = new Jedis("localhost");
        conn.select(15);

        // 测试1：redis记录日志
//        testLogRecent(conn);
//        testLogCommon(conn);

        // 测试2：计数器和统计数据
//        testCounters(conn);
//        testStats(conn);
//        testAccessTime(conn);

        // 测试3：查找ip所在的城市及其详细信息
//        testIpLookup(conn);

        // 测试4：服务发现与配置
//        testIsUnderMaintenance(conn);
        testConfig(conn);
    }

    private void testConfig(Jedis conn) throws InterruptedException {
        System.out.println("首先，创建一个配置文件map：");
        Map<String, Object> config = new HashMap<String, Object>();
        config.put("db", 15);
        // 添加到redis中
        setConfig(conn, "redis", "test", config);
        Jedis conn2 = redisConnection("test");
        System.out.println("新的连接是否存在：" + (conn2.info() != null));
    }

    private static final Map<String, Jedis> REDIS_CONNECTIONS = new HashMap<String, Jedis>();
    private static final Map<String, Map<String, Object>> CONFIGS =
            new HashMap<String, Map<String, Object>>();
    private static final Map<String, Long> CHECKED = new HashMap<String, Long>();

    private Jedis redisConnection(String component) {
        Jedis configConn = REDIS_CONNECTIONS.get("config");
        if (configConn == null) {
            configConn = new Jedis("localhost");
            configConn.select(15);
            REDIS_CONNECTIONS.put("config", configConn);
        }
        String key = "config:redis:" + component;
        Map<String, Object> oldConfig = CONFIGS.get(key);
        Map<String, Object> newConfig = getConfig(configConn, "redis", component);
        // 判断配置文件是否相等，不相等，更改当前配置文件
        if (!newConfig.equals(oldConfig)) {
            Jedis conn = new Jedis("localhost");
            conn.select(((Double) newConfig.get("db")).intValue());
            REDIS_CONNECTIONS.put(key, conn);
        }
        return REDIS_CONNECTIONS.get(key);
    }

    /**
     * 从redis获取新的配置文件，与当前程序中的配置文件CONFIGS对比
     */
    @SuppressWarnings("unchecked")
    private Map<String, Object> getConfig(Jedis conn, String type, String component) {
        String key = "config:" + type + ":" + component;
        long wait = 1000;
        if (CHECKED.get(key) == null || CHECKED.get(key) < System.currentTimeMillis() - wait) {
            CHECKED.put(key, System.currentTimeMillis());
            String value = conn.get(key);
            Map<String, Object> config = null;
            if (value != null) {
                Gson gson = new Gson();
                config = (Map<String, Object>) gson.fromJson(value, new TypeToken<Map<String, Object>>() {
                }.getType());
            } else {
                config = new HashMap<String, Object>();
            }
            CONFIGS.put(key, config);
        }
        return CONFIGS.get(key);
    }

    private void setConfig(Jedis conn, String type, String component, Map<String, Object> config) {
        Gson gson = new Gson();
        conn.set("config:" + type + ":" + component, gson.toJson(config));
    }

    private void testIsUnderMaintenance(Jedis conn) throws InterruptedException {
        boolean flag = false;
        flag = isUnderMaintenance(conn);
        System.out.println("是否在维护:" + flag);
        conn.set("is-under-maintenance", "yes");
        flag = isUnderMaintenance(conn);
        System.out.println("改变后，是否在维护:" + flag);
        Thread.sleep(1000);
        flag = isUnderMaintenance(conn);
        System.out.println("停留1秒钟，是否在维护:" + flag);

        conn.del("is-under-maintenance");
        Thread.sleep(1000);
        flag = isUnderMaintenance(conn);
        System.out.println("清除后，是否在维护:" + flag);

    }


    private long lastChecked;
    private boolean underMaintenance;

    private boolean isUnderMaintenance(Jedis conn) {
        if (lastChecked < System.currentTimeMillis() - 1000) {
            String flag = conn.get("is-under-maintenance");
            underMaintenance = "yes".equals(flag);
        }
        return underMaintenance;
    }

    private void testIpLookup(Jedis conn) {
        String cwd = System.getProperty("user.dir");
        File blocks = new File(cwd + "/GeoLiteCity-Blocks.csv");
        File locations = new File(cwd + "/GeoLiteCity-Location.csv");
        if (!blocks.exists()) {
            System.out.println("文件不存在：" + blocks);
        }
        if (!locations.exists()) {
            System.out.println("文件不存在：" + locations);
        }
        System.out.println("将IP数据载入到redis：");
//        importIpsToRedis(conn, blocks);
        long ipSum = conn.zcard("ip2cityId:");
        System.out.println("IP数量为：" + ipSum);

        System.out.println("将城市数据载入redis：");
//        importCitiesToRedis(conn, locations);
        long citySum = conn.hlen("cityId2City:");
        System.out.println("城市数量为：" + citySum);


        System.out.println("随机查找ip");
        for (int i = 0; i < 5; i++) {
            String ip = randomOctet(255) + "."
                    + randomOctet(256) + "."
                    + randomOctet(256) + "."
                    + randomOctet(256);
            String cityMessage = Arrays.toString(findCityByIp(conn, ip));
            System.out.println("所在城市信息为：");
            System.out.println(cityMessage);
        }
    }

    private void importCitiesToRedis(Jedis conn, File file) {
        FileReader reader = null;
        Gson gson = new Gson();
        try {
            reader = new FileReader(file);
            CSVParser parser = new CSVParser(reader, CSVFormat.DEFAULT);
            for (CSVRecord record : parser) {
                if (record.size() < 4 || !Character.isDigit(record.get(0).charAt(0))) {
                    continue;
                }
                String cityId = record.get(0);

                String country = record.get(1);
                String region = record.get(2);
                String city = record.get(3);
                String postalCode = record.get(4);
                String latitude = record.get(5);
                String longitude = record.get(6);
                String metroCode = record.get(7);
                String areaCode = record.get(8);
                String json = gson.toJson(new String[]{
                        country, region, city, postalCode, latitude, longitude, metroCode, areaCode
                });
                conn.hset("cityId2City:", cityId, json);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                reader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private String[] findCityByIp(Jedis conn, String ipAddress) {
        long score = ipToScore(ipAddress);
        Set<String> rangs = conn.zrevrangeByScore("ip2cityId:", score, 0, 0, 1);
        if (rangs.size() == 0) {
            return null;
        }
        String cityId = rangs.iterator().next();
        cityId = cityId.substring(0, cityId.indexOf("_"));
        return new Gson().fromJson(conn.hget("cityId2City:", cityId), String[].class);
    }

    private String randomOctet(int max) {
        return String.valueOf((int) (Math.random() * max));
    }

    private void importIpsToRedis(Jedis conn, File file) {
        FileReader fileReader = null;
        try {
            int number = 0;
            fileReader = new FileReader(file);
            CSVParser parser = new CSVParser(fileReader, CSVFormat.DEFAULT);
            for (CSVRecord csvRecord : parser) {
                String startIp = csvRecord.get(0);
                if (startIp.toLowerCase().indexOf('i') != -1) {
                    continue;
                }
                long score = 0;
                if (startIp.indexOf('.') != -1) {
                    score = ipToScore(startIp);
                } else {
                    score = Long.parseLong(startIp, 10);
                }
                System.out.println(number);
                String cityIp = csvRecord.get(2) + "_" + number;
                number++;
                conn.zadd("ip2cityId:", score, cityIp);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                fileReader.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private long ipToScore(String ipAddress) {
        long score = 0;
        // "."需要转义
        for (String str : ipAddress.split("\\.")) {
            score = score * 256 + Integer.parseInt(str, 10);
        }
        return score;
    }

    /**
     * 记录程序执行时长
     *
     * @param conn
     */
    private void testAccessTime(Jedis conn) throws InterruptedException {
        AccessTimer timer = new AccessTimer(conn);
        for (int i = 0; i < 5; i++) {
            timer.setStart(System.currentTimeMillis());
            Thread.sleep((long) (Math.random() * 4000 + 1));
            timer.stop("request-" + i);
        }
        System.out.println("输出所有的分数，只缓存最多的100个：");
        Set<Tuple> stats = conn.zrangeWithScores("slowest:AccessTime", 0, -1);
        for (Tuple tuple : stats) {
            System.out.println(tuple.getElement() + " -- " + tuple.getScore());
        }

    }

    private void testStats(Jedis conn) {
        List<Object> rs = null;
        System.out.println("写入数据：");
        for (int i = 0; i < 5; i++) {
            int value = (int) (Math.random() * 11 + 5);
            rs = updateState(conn, "context", "type", value);
        }
        HashMap<String, Double> stats = getStats(conn, "context", "type");
        System.out.println("打印数据报表：");
        System.out.println(stats);
    }

    private HashMap<String, Double> getStats(Jedis conn, String context, String type) {
        String keys = "stats:" + context + ":" + type;
        Set<Tuple> datas = conn.zrangeWithScores(keys, 0, -1);
        HashMap<String, Double> stats = new HashMap<String, Double>();
        for (Tuple tuple : datas) {
            stats.put(tuple.getElement(), tuple.getScore());
        }
        stats.put("average", stats.get("sum") / stats.get("count"));
        // 标准差的公式需要自行推导
        double numerator = stats.get("sumsq") - Math.pow(stats.get("sum"), 2) / stats.get("count");
        double count = stats.get("count");
        stats.put("stddev", Math.pow(numerator / (count > 1 ? count - 1 : 1), .5));
        return stats;
    }

    private List<Object> updateState(Jedis conn, String context, String type, long value) {
        String destination = "stats:" + context + ":" + type;
        String startKey = destination + ":start";
        int timeout = 5000;
        long end = System.currentTimeMillis() + timeout;
        while (System.currentTimeMillis() < end) {
            conn.watch(startKey);
            long now = System.currentTimeMillis();
            String hourstart = ISO_FORMAT.format(new Date());
            String existing = conn.get(startKey);
            Transaction trans = conn.multi();
            if (existing != null && COLLATOR.compare(existing, now) < 0) {
                trans.rename(destination, destination + ":last");
                trans.rename(startKey, destination + ":plast");
                trans.set(startKey, hourstart);
            }

            String tkey1 = UUID.randomUUID().toString();
            String tkey2 = UUID.randomUUID().toString();
            trans.zadd(tkey1, value, "min");
            trans.zadd(tkey1, value, "max");
            trans.zunionstore(destination, new ZParams().aggregate(ZParams.Aggregate.MIN), destination, tkey1);
            trans.zunionstore(destination, new ZParams().aggregate(ZParams.Aggregate.MAX), destination, tkey2);
            trans.del(tkey1, tkey2);
            trans.zincrby(destination, 1, "count");
            trans.zincrby(destination, value, "sum");
            trans.zincrby(destination, value * value, "sumsq");
            List<Object> results = trans.exec();
            if (results == null) {
                continue;
            }
            return results.subList(results.size() - 3, results.size());
        }
        return null;
    }

    private void testCounters(Jedis conn) throws InterruptedException {
        conn.del("common:test:info");
        long now = System.currentTimeMillis() / 1000;
        for (int i = 0; i < 10; i++) {
            int count = (int) (Math.random() * 5) + 1; // 随机计数加量大小
            updateCounter(conn, "test", count, now + i);
        }

        System.out.println("测试1秒钟的：");
        ArrayList<Pair<Integer, Integer>> counter = getCounter(conn, "test", 1);
        for (Pair<Integer, Integer> count : counter) {
            System.out.println("  " + count);
        }
        assert counter.size() >= 10;


        System.out.println("测试5秒钟的：");
        counter = getCounter(conn, "test", 5);
        for (Pair<Integer, Integer> count : counter) {
            System.out.println("  " + count);
        }
        assert counter.size() >= 2;

        CleanCountersThread thread = new CleanCountersThread(0, 2 * 86400);
        thread.start();
        Thread.sleep(10000);
        thread.quit();
        thread.interrupt();
        counter = getCounter(conn, "test", 86400);
        System.out.println("Did we clean out all of the counters? " + (counter.size() == 0));
        assert counter.size() == 0;
    }

    private ArrayList<Pair<Integer, Integer>> getCounter(Jedis conn, String name, int precision) {
        String hash = String.valueOf(precision) + ":" + name;
        Map<String, String> datas = conn.hgetAll("count:" + hash);
        ArrayList<Pair<Integer, Integer>> results = new ArrayList<Pair<Integer, Integer>>();
        for (Map.Entry<String, String> entry : datas.entrySet()) {
            results.add(new Pair<Integer, Integer>(
                    Integer.parseInt(entry.getKey()),
                    Integer.parseInt(entry.getValue())
            ));
        }
        Collections.sort(results); // 之所有组装，是为了进行排序，将旧的数据放在前面（按时间）
        return results;
    }

    private void updateCounter(Jedis conn, String name, int count, long now) {
        Transaction trans = conn.multi();
        for (int prec : PRECISION) {
            // 计数器的开始时间（有多个）
            long pnow = (now / prec) * prec;
            String hash = String.valueOf(prec) + ":" + name;
            trans.zadd("known:", 0, hash); // 有序表-记录分数
            trans.hincrBy("count:" + hash, String.valueOf(pnow), count); // 哈希表记录每个日志信息：（起始时间-计数次数）
        }
        trans.exec();
    }

    /**
     * 测试1.2
     * 记录重要日志，并筛选
     *
     * @param conn
     */
    private void testLogCommon(Jedis conn) {
        conn.del("common:test:info");
        System.out.println("添加测试日志");
        for (int count = 1; count < 6; count++) {
            for (int i = 0; i < count; i++) {
                logCommon(conn, "test", "message-" + count);
            }
        }
        Set<Tuple> commonsets = conn.zrevrangeWithScores("common:test:info", 0, -1);
        for (Tuple tuple : commonsets) {
            System.out.println(tuple.getElement() + " : " + tuple.getScore());
        }
        assert commonsets.size() >= 5 : "日志数量不够，出错";
    }

    private void logCommon(Jedis conn, String name, String message) {
        logCommon(conn, name, message, INFO, 5000);
    }

    private void logCommon(Jedis conn, String name, String message, String severity, int timeout) {
        String commonDest = "common:" + name + ":" + severity;
        String startKey = commonDest + ":start";
        long end = System.currentTimeMillis() + timeout;
        while (System.currentTimeMillis() < end) {
            conn.watch(startKey);
            String existing = conn.get(startKey);
            Transaction trans = conn.multi(); // 创建事务
            String hourStart = ISO_FORMAT.format(new Date());

            /**
             * if用于判断，当过了一个小时时，将数据进行持久化存储。即程序只会记录最近一个小时之内的日志，很久以前的将删除
             * 更确切的说：“start”记录当前一个小时的，“last”记录上一个小时的（完整的一个小时）
             * commonDest用于保存日志分数值
             * startKey用于保存日志最后存储的时间（按所在小时记录）
             */
            if (existing != null && COLLATOR.compare(existing, hourStart) < 0) {
                trans.rename(commonDest, commonDest + ":last");
                trans.rename(startKey, commonDest + ":pstart");
                trans.set(startKey, hourStart);
            }

            trans.zincrby(commonDest, 1, message); // 添加日志，加分

            // 同步到最近的日志，减少客户端服务器交互：因为如果这里不处理，返回给客户端，客户端又要请求服务器保存。
            // 多了一次往返
            String recentDest = "recent:" + name + ':' + severity;
            trans.lpush(recentDest, TIMESTAMP.format(new Date()) + ' ' + message);
            trans.ltrim(recentDest, 0, 99);
            List<Object> results = trans.exec();
            if (results == null) {
                continue;
            }
            return;
        }
    }

    /**
     * 测试1.1
     * 记录登录日志
     *
     * @param conn
     */
    private void testLogRecent(Jedis conn) {
        conn.del("recent:test:info");
        for (int i = 0; i < 5; i++) {
            logRecent(conn, "test", "this message is : " + i);
            System.out.println("\n");
        }
    }

    private void logRecent(Jedis conn, String name, String message) {
        logRecent(conn, name, message, INFO);
        List<String> recentLogs = conn.lrange("recent:test:info", 0, -1);
        System.out.println("当前所有的登陆日志有：");
        for (String log : recentLogs) {
            System.out.println(log);
        }
        assert recentLogs.size() >= 5 : "日志数量不够，出错";
    }

    private void logRecent(Jedis conn, String name, String message, String severity) {
        String destination = "recent:" + name + ":" + severity;
        Pipeline pipe = conn.pipelined();
        pipe.lpush(destination, TIMESTAMP.format(new Date()) + " " + message);
        pipe.ltrim(destination, 0, 99); // 截取信息，只保留前100条
        pipe.sync(); // 同步管道数据，获取所有的响应值
    }

    private class CleanCountersThread extends Thread {
        private Jedis conn;
        private int sampleCount = 100;
        private boolean quit;
        private long timeOffset; // used to mimic a time in the future.

        public CleanCountersThread(int sampleCount, int timeOffset) {
            this.conn = new Jedis("localhost");
            this.conn.select(15);
            this.sampleCount = sampleCount;
            this.timeOffset = timeOffset;
        }

        public void quit() {
            quit = true;
        }

        @Override
        public void run() {
            int passes = 0; // 清理的次数
            while (!quit) {
                long start = System.currentTimeMillis();
                int index = 0; // 记录表下标[1,5,60...86400]
                while (index < conn.zcard("known:")) {
                    Set<String> oneSet = conn.zrange("known:", index, index);
                    index++;// 处理了一个
                    // 如果表中无数据，则返回
                    if (oneSet.size() == 0) {
                        break;
                    }
                    String onehash = oneSet.iterator().next();
                    int prec = Integer.parseInt(onehash.substring(0, onehash.indexOf(':')));
                    int numPrec = (int) Math.floor(prec / 60);
                    if (numPrec == 0) {
                        numPrec = 1;
                    }

                    // 对更新在60秒之内的，为防止清理多次，即只清理一次，需要计数判断
                    // 此时numPrec==1，下列计算将不等于0，不再进行删除
                    if ((passes % numPrec) != 0) {
                        continue;
                    }

                    String hkey = "count:" + onehash;
                    ArrayList<String> keyLists = new ArrayList<String>(conn.hkeys(hkey));
                    Collections.sort(keyLists);
                    String cutoff = String.valueOf((System.currentTimeMillis() + timeOffset) / 1000 - sampleCount * prec);
                    int delNum = bisectRight(keyLists, cutoff);

                    // 删除数据
                    System.out.println(hkey.toString() + "删除数量为：" + delNum);
                    if (delNum != 0) {
                        conn.hdel(hkey, keyLists.subList(0, delNum).toArray(new String[0]));
                        // 如果删除的数据刚好等于数据的总数，则表明表已经清空，需要同时删除有序表
                        if (delNum == keyLists.size()) {
                            conn.watch(hkey);
                            if (conn.hlen(hkey) == 0) {
                                Transaction trans = conn.multi();
                                trans.zrem("known:", onehash);
                                trans.exec();
                                index--;// 直接删除了，需要处理index
                            }
                        } else {
                            conn.unwatch(); // 不为空时，解除监控
                        }
                    }
                }

                // 清理一次+1
                passes++;
                // 使程序60秒执行一次

                // 程序持续时间
                long duration = Math.min(System.currentTimeMillis() + timeOffset - start + 1000, 60000);
                // 多余的时间停留一下
                try {
                    sleep(Math.max(60000 - duration, 1000));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }

        private int bisectRight(ArrayList<String> keyLists, String cutoff) {
            int index = Collections.binarySearch(keyLists, cutoff);
            return index > 0 ? index + 1 : Math.abs(index) - 1;
        }
    }

    private class AccessTimer {
        private Jedis conn;
        private long start;

        public AccessTimer(Jedis conn) {
            this.conn = conn;
        }

        public void setStart(long start) {
            this.start = start;
        }

        /**
         * 记录停留时间-即类似执行时间
         */
        public void stop(String context) {
            long delta = System.currentTimeMillis() - start;
            List<Object> stats = updateState(conn, context, "AccessTime", delta / 1000);
            double average = (Double) stats.get(1) / (Double) stats.get(0);
            Transaction trans = conn.multi();
            trans.zadd("slowest:AccessTime", average, context);
            trans.zremrangeByRank("slowest:AccessTime", 0, -101);
            trans.exec();
        }
    }
}

```


