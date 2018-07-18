---
title:  并发编程实战13-同步中的四种锁synchronized、ReentrantLock、ReentrantReadWriteLock、StampedLock
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---



 1. synchronized同步锁
 	* synchronized属于悲观锁，直接对区域或者对象加锁，性能稳定，可以使用大部分场景。
 2. ReentrantLock可重入锁（Lock接口）
 	* 相对于synchronized更加灵活，可以控制加锁和放锁的位置
 	* 可以使用Condition来操作线程，进行线程之间的通信
 	* 核心类AbstractQueuedSynchronizer，通过构造一个基于阻塞的CLH队列容纳所有的阻塞线程，而对该队列的操作均通过Lock-Free（CAS）操作，但对已经获得锁的线程而言，ReentrantLock实现了偏向锁的功能。
 3. ReentrantReadWriteLock可重入读写锁（ReadWriteLock接口）
 	* 相对于ReentrantLock，对于大量的读操作，读和读之间不会加锁，只有存在写时才会加锁，但是这个锁是悲观锁
 	* ReentrantReadWriteLock实现了读写锁的功能
 	* ReentrantReadWriteLock是ReadWriteLock接口的实现类。ReadWriteLock接口的核心方法是readLock()，writeLock()。实现了并发读、互斥写。但读锁会阻塞写锁，是悲观锁的策略。
 4. StampedLock戳锁
 	* ReentrantReadWriteLock虽然解决了大量读取的效率问题，但是，由于实现的是悲观锁，当读取很多时，读取和读取之间又没有锁，写操作将无法竞争到锁，就会导致写线程饥饿。所以就需要对读取进行乐观锁处理。
 	* StampedLock加入了乐观读锁，不会排斥写入
 	* 当并发量大且读远大于写的情况下最快的的是StampedLock锁




----------


>StampedLock控制锁有三种模式（排它写，悲观读，乐观读），一个StampedLock状态是由版本和模式两个部分组成，锁获取方法返回一个数字作为票据stamp，它用相应的锁状态表示并控制访问。

StampedLock主要使用的是乐观读取，通过一个stamp变量来检测是否有读写冲突，每次乐观读取时会加入stamp校验。当存在写入操作时，写操作会改变stamp的状态，就会导致乐观读取失败，乐观读锁就会升级为悲观读锁

``` stylus
package com.sound.daytd5;

import java.util.concurrent.locks.StampedLock;

/**
 * @author: ZouTai
 * @date: 2018/4/17
 * @description:
 */
public class StampedLockDemo {
    private int balance;
    private final StampedLock stampedLock = new StampedLock();


    /**
     * 1、悲观写
     * writeLock()：典型的cas操作，如果STATE等于s,设置写锁位为1（s+WBIT）。
     * acquireWrite跟acquireRead逻辑类似，先自旋尝试、加入等待队列、直至最终Unsafe.park()挂起线程。
      */
    public void write(int i) {
        long stamp = stampedLock.writeLock();
        balance += i;
        stampedLock.unlockWrite(stamp);
    }

    /**
     * 2、悲观读
     * 乐观锁失败后锁升级为readLock()：尝试state+1,用于统计读线程的数量，
     * 如果失败，进入acquireRead()进行自旋，通过CAS获取锁。
     * 如果自旋失败，入CLH队列，然后再自旋，
     * 如果成功获得读锁，激活cowait队列中的读线程Unsafe.unpark(),
     * 最终依然失败，Unsafe().park()挂起当前线程。
     */
    public void read() {
        long stamp = stampedLock.readLock();
        int value = balance;
        stampedLock.unlockRead(stamp);
    }

    /**重点：！！！
     * 3、乐观读：当读取远远大于写入时，使用乐观锁
     * tryOptimisticRead()：如果当前没有写锁占用，返回state(后7位清0，即清0读线程数)，如果有写锁，返回0，即失败。
     */
    public void optimisticRead() {
        long stamp = stampedLock.tryOptimisticRead();
        int value = balance;
        // 校验这个戳是否有效validate()：比较当前stamp和发生乐观锁得到的stamp比较，不一致则失败。
        if(!stampedLock.validate(stamp)) {
            long readStamp = stampedLock.readLock();
            value = balance;
            stamp = readStamp;
        }
        stampedLock.unlockRead(stamp);
    }

    /**重点：！
     * 4、判断条件之后，再写
     * 存在读取和写入两个操作
     */
    public void conditionReadWrite(int state){
        // 首先读取
        long stamp = stampedLock.readLock();
        while (balance > 100){
            long writeStamp = stampedLock.tryConvertToWriteLock(stamp);
            // 步骤：a
            if(writeStamp!=0) {
                balance += state;
                stamp = writeStamp;
                break;
            } else {
                // 转换失败
                stampedLock.unlockRead(stamp);
                //显式直接进行写锁 然后再通过循环再试,回到 步骤：a
                stamp = stampedLock.writeLock();
            }
        }
    }

}

```


[同步中四种锁的性能比较][1]
[同步中的四种锁synchronized、ReentrantLock、ReentrantReadWriteLock、StampedLock][2]
[同步锁参考英文文档][3]


  [1]: http://colobu.com/2016/06/01/Java-8-StampedLocks-vs-ReadWriteLocks-and-Synchronized/
  [2]: http://www.cnblogs.com/dennyzhangdd/p/6925473.html
  [3]: https://www.javaspecialists.eu/talks/jfokus13/PhaserAndStampedLock.pdf
