---
title: 并发编程实战6-线程之间的通信-深入解析Condition源码
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

* Object类的几个方法
	* notify方法：只会随机唤醒一个wait线程，然后此wait线程将会继续执行 
	* notifyAll方法：会唤醒所有的wait线程，所有wait线程将会全部执行

* 显示锁的condition对象
	* 对于Object类的wait和notify方法有一定的缺陷，即无法精确唤醒指定的线程。所以引入了lock的condition对象，可以对不同的条件进行判断，来选择唤醒不同的线程
	``` stylus
	import java.util.concurrent.locks.Condition;
	import java.util.concurrent.locks.Lock;
	import java.util.concurrent.locks.ReentrantLock;
	
	/**
	 * @author: ZouTai
	 * @date: 2018/4/9
	 * @description:
	 */
	public class ConditionBoundedBuffer {
	    private Lock lock = new ReentrantLock();
	
	    /**
	     * 这里使用生产者-消费者模式
	     */
	    private Condition product = lock.newCondition();
	    private Condition consume = lock.newCondition();
	    private final int max = 10;
	    int count = 0;
	
	    public void put() throws InterruptedException {
	        lock.lock();
	        try {
	            while (count >= max) {
	                System.out.println(Thread.currentThread().getName()+"生产过量，停止生产...");
	                /**
	                 * 1、生产过量，停止生产：生产者等待
	                 */
	                product.await();
	            }
	            count++;
	            /**
	             * 2、唤醒消费者
	             */
	            consume.signal();
	            System.out.println(Thread.currentThread().getName()+"生产-库存变为："+count);
	        } finally {
	            lock.unlock();
	        }
	
	    }
	    public void take() throws InterruptedException {
	        lock.lock();
	        try {
	            while (count == 0) {
	                System.out.println(Thread.currentThread().getName()+"库存为空，无法购买...");
	                /**
	                 * 3、库存为空，无法购买：消费者等待
	                 */
	                consume.await();
	            }
	            count--;
	            System.out.println(Thread.currentThread().getName()+"消费-库存还剩："+count);
	            /**
	             * 4、唤醒生产者
	             */
	            product.signal();
	        } finally {
	            lock.unlock();
	        }
	    }
	}
	```


