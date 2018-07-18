---
title: 并发编程实战8-ThreadLocal原理与使用-线程局部变量
tags: ThreadLocal,弱引用,ThreadLocalMap
grammar_cjkRuby: true
---
>首先，我的理解，ThreadLocal只是一个公用对象，但是并不是完全用来作为线程之间共享的。原因在于，它只是一种公用变量模板，每个线程只是拥有它的复制版（线程死亡后，复制版也随之死亡），而不是直接使用公用变量，这样就避免了共享安全问题。但是，为什么不让每个线程直接去创建自己的实例变量呢？主要是因为，线程进来时它自己可能并不清楚需要哪些变量，而且在线程执行完毕，还需要自己去销毁这些变量，下的太繁杂。

举个例子：当我去吃火锅时（我就是一个线程），餐厅都会有一个菜单纸（ThreadLocal）。没有使用ThreadLocal之前，我们和其他客人公用一张菜单纸，这当然是不符合逻辑的，所以，餐厅需要给我复制一份，我就可以在上面选菜了（菜单-账单就相当于：ThreadLocal-value，即ThreadLocalMap（key-value））。当我就餐结束，结账后（线程结束），我的菜单也就作废了（此时垃圾回收）。即线程公用一张ThreadLocal，但是并不直接使用那个共用的ThreadLocal，而是自己复制一份ThreadLocal，使用完后，这个复制版也就没用了。

为了严谨，下文综合其他解释：

jdk1.8源码解释：ThreadLocal类用来提供线程内部的局部变量。这种变量在多线程环境下访问(通过get或set方法访问)时能保证各个线程里的变量相对独立于其他线程内的变量。ThreadLocal实例通常来说都是private static类型的，用于关联线程和线程的上下文。
	* 之所以使用static是因为ThreadLocal是ThreadLocalMap的key的弱引用，当没有强引用指向ThreadLocal，gc时就会被回收，而此时key将为null。

* 一、实例应用：

``` stylus
package com.sound.daytb3;

/**
 * @author: ZouTai
 * @date: 2018/4/10
 * @description: ThreadLOcal-线程局部变量
 * 即，每个线程将维护一个自己的变量，这个变量只对当前线程可以随意更改，其他线程不会影响当前线程变量的值
 */
public class ThreadLocalDemo {
    private static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            // 初始化值为0
            return new Integer(0);
        }
    };

    public int getNext() {
        int count = threadLocal.get();
        count++;
        threadLocal.set(count);
        return count;
    }

    public static void main(String[] args) {
        ThreadLocalDemo tld = new ThreadLocalDemo();
        new Thread() {
            @Override
            public void run() {
                while (true) {
                    System.out.println(Thread.currentThread().getName() + "" + tld.getNext());
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    System.out.println(Thread.currentThread().getName() + "" + tld.getNext());
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }.start();
    }
}

```

* 二、源码解析：几个重要的方法get、set、remove

``` stylus
/**1、
     * initialValue函数用来设置ThreadLocal的初始值
      */
    private static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            // 初始化值为0
            return new Integer(0);
        }
    };

    /**2、
     * 该函数用来获取与当前线程关联的ThreadLocal的值
     * 如果当前线程没有该ThreadLocal的值，则调用initialValue函数获取初始值返回。
     * @return
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocal.ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocal.ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    /**3、
     * 设置当前线程的ThreadLocal的值为value。
     * @param value
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocal.ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    /**4、
     * remove函数用来将当前线程的ThreadLocal绑定的值删除
     * 在某些情况下需要手动调用该函数，防止内存泄露。
     */
    public void remove() {
        ThreadLocal.ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null)
            m.remove(this);
    }
```


* 三、ThreadLocal内部的实现原理-原理改进
	* 我们知道ThreadLocal内部使用的是map，map有三个点（map节点node和两个分支key-value），这里面，我们也是有三个属性（当前线程Thread、ThreadLocal变量和变量值value）。显然，变量值就对应value；但是node和key分别对应什么？
	* 早期的版本中，1.3之前，jdk将当前线程Thread作为key，ThreadLocal作为node。之后，更改为Thread作为node，ThreadLocal作为key。之所以这样做有以下两个好处：
		* 1.减少map节点，能提高性能：由于实际很多时候，线程很多，但是ThreadLocal变量其实很少。这样设计之后每个Map的Entry数量变小了：之前是Thread的数量，现在是ThreadLocal的数量。
		* 2.减少内存使用量：当Thread销毁之后对应的ThreadLocalMap也就随之销毁了。


* 四、内存泄露问题
引起内存泄露的主要问题是由于ThreadLocalMap的key使用了对ThreadLocal的弱引用导致的。
	* ThreadLocalMap是使用ThreadLocal的弱引用作为Key的
	* ThreadLocal中的引用对象如下图：
	![引用图][1]
	* 内存泄露问题：如图，ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用引用他，那么系统gc的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：`ThreadLocal Ref -> Thread -> ThreaLocalMap -> Entry -> value`永远无法回收，造成内存泄露。

**为什么要使用弱引用？直接使用强引用不就可以了吗？**
其实强引用将导致另外一种内存泄漏问题。即ThreadLocal引用为null，导师线程却没有回收，将用于持有ThreadLocal的强引用，无法回收，这也会导致内存泄漏。比如使用线程池的时候，线程结束是不会销毁的，会再次使用的。就可能出现内存泄露。　　

**内存泄漏的防护措施**
>源码内部会检查ThreaLocalMap，会将key为null的entry都进行删除
转载：整理一下ThreadLocalMap的getEntry函数的流程：
	* 1、首先从ThreadLocal的直接索引位置(通过ThreadLocal.threadLocalHashCode & (len-1)运算得到)获取Entry e，如果e不为null并且key相同则返回e；
	* 2、如果e为null或者key不一致则向下一个位置查询，如果下一个位置的key和当前需要查询的key相等，则返回对应的Entry，否则，如果key值为null，则擦除该位置的Entry，否则继续向下一个位置查询
	* 在这个过程中遇到的key为null的Entry都会被擦除，那么Entry内的value也就没有强引用链，自然会被回收。仔细研究代码可以发现，set操作也有类似的思想，将key为null的这些Entry都删除，防止内存泄露。 但是光这样还是不够的，上面的设计思路依赖一个前提条件：要调用ThreadLocalMap的genEntry函数或者set函数。这当然是不可能任何情况都成立的，所以很多情况下需要使用者手动调用ThreadLocal的remove函数，手动删除不再需要的ThreadLocal，防止内存泄露。所以JDK建议将ThreadLocal变量定义成private static的，这样的话ThreadLocal的生命周期就更长，由于一直存在ThreadLocal的强引用，所以ThreadLocal也就不会被回收，也就能保证任何时候都能根据ThreadLocal的弱引用访问到Entry的value值，然后remove它，防止内存泄露。


参考链接：[https://zhuanlan.zhihu.com/p/20213204][2]
[http://www.sczyh30.com/posts/Java/java-concurrent-threadlocal/][3]
[https://www.cnblogs.com/onlywujun/p/3524675.html][4]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1523352180650.jpg
  [2]: https://zhuanlan.zhihu.com/p/20213204
  [3]: http://www.sczyh30.com/posts/Java/java-concurrent-threadlocal/
  [4]: https://www.cnblogs.com/onlywujun/p/3524675.html
