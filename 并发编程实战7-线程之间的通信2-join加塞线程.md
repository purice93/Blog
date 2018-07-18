---
title: 并发编程实战7-线程之间的通信2-join加塞线程
tags: join,加塞线程,并发编程
grammar_cjkRuby: true
---

>thread.Join把指定的线程加入到当前线程，可以将两个交替执行的线程合并为顺序执行的线程。比如在线程B中调用了线程A的Join()方法，直到线程A执行完毕后，才会继续执行线程B。

t.join();      //使调用线程 t 在此之前执行完毕。
t.join(1000);  //等待 t 线程，等待时间是1000毫秒

* 一、为什么要用join()方法：
在很多情况下，主线程生成并起动了子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程处理完其他的事务后，需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到join()方法了。
	* join()的作用是：“等待该线程终止”，这里需要理解的就是该线程是指的主线程等待子线程的终止。也就是在子线程调用了join()方法后面的代码，只有等到子线程结束了才能执行。

* 二、实例：

``` stylus
/**
 * @author: ZouTai
 * @date: 2018/4/10
 * @description: join-加塞线程，即，向线程加入内部线程，且内部线程，执行完，主线程才能继续。
 * 相当于将加塞线程合并入主线程
 *
 *
 */
public class JoinDemo01 {
    public void a(Thread joinThread){
        System.out.println("线程a开始...");
        joinThread.start();
        try {
            joinThread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("线程a执行完毕");
    }

    public void b() {
        System.out.println("加塞线程开始执行...");
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("加塞线程执行完毕");
    }

    public static void main(String[] args) {
        JoinDemo01 joinDemo01 = new JoinDemo01();
        Thread jThread = new Thread() {
            @Override
            public void run() {
                joinDemo01.b();
            }
        };
        joinDemo01.a(jThread);
    }

    /** result is :
     线程a开始...
     加塞线程开始执行...
     加塞线程执行完毕
     线程a执行完毕
     */
}
```

* 三、源码解析

``` stylus

// join方法：注意isAlive和wait对象线程的区别，不同
if (millis == 0) {
            while (isAlive()) { // 1、判断当前线程是否存货（即加塞线程）
                wait(0); // 2、使wait监视器对象线程等待（即主线程）；为什么1、2对象不同，需要了解wait方法，如下
            }
        } 
		
* This method should only be called by a thread that is the owner
* of this object's monitor. 
* 即wait方法是被当前线程的监视器调用，即当前运行的线程，即上面的主线程
* 监视器：即当前运行方法join所在的线程，而不是调用join方法的线程对象joinThread；而isAlive方法才是指的isAlive对象
public final native void wait(long timeout) throws InterruptedException;
```


