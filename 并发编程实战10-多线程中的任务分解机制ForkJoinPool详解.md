---
title: 并发编程实战10-多线程中的任务分解机制ForkJoinPool详解
tags: ForkJoin,模板,小书匠
grammar_cjkRuby: true
---
>Fork/Join 模式类似于MapReduce，也相当于一种分而治之的理念，或者说就像二分查找、二路归并算法。通过将一个大量的计算分解为许多的小计算，分而治之，然后再合并，同时，这些分出来的每个小计算都是并行进行的，这样就大大增大了CPU的利用率。

Fork/Join 模式有自己的适用范围。如果一个应用能被分解成多个子任务，并且组合多个子任务的结果就能够获得最终的答案，那么这个应用就适合用 Fork/Join 模式来解决。图 1 给出了一个 Fork/Join 模式的示意图，位于图上部的 Task 依赖于位于其下的 Task 的执行，只有当所有的子任务都完成之后，调用者才能获得 Task 0 的返回结果。可以说，Fork/Join 模式能够解决很多种类的并行问题。通过使用 Doug Lea 提供的 Fork/Join 框架，软件开发人员只需要关注任务的划分和中间结果的组合就能充分利用并行平台的优良性能。其他和并行相关的诸多难于处理的问题，例如负载平衡、同步等，都可以由框架采用统一的方式解决。这样，我们就能够轻松地获得并行的好处而避免了并行编程的困难且容易出错的缺点。
![图 1. Fork/Join 模式示意图][1]

在多线程并发编程中，有时候会遇到将大任务分解成小任务再并发执行的场景。Java 8新增的ForkJoinPool很好的支持了这个问题。ForkJoinPool是一种支持任务分解的线程池，当提交给他的任务“过大”，他就会按照预先定义的规则将大任务分解成小任务，多线程并发执行。 一般要配合可分解任务接口ForkJoinTask来使用，ForkJoinTask有两个实现它的抽象类：RecursiveAction和RecursiveTask，其区别是前者没有返回值，后者有返回值。

* 应用场景：
简单来说：如果你的问题能很容易分解成子问题，则比较适合ForkJoinPool。适合CPU密集型的场景。

* 实例：计算1+2+...+100;并行计算

``` stylus
package com.sound.daytc1;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;
import java.util.concurrent.RecursiveTask;

/**
 * @author: ZouTai
 * @date: 2018/4/10
 * @description: ForkJoin模式计算序列相加-二分法
 */
public class RecursiveTaskDemo extends RecursiveTask<Integer> {
    private int first;
    private int last;

    public RecursiveTaskDemo(int first, int last) {
        this.first = first;
        this.last = last;
    }

    @Override
    protected Integer compute() {
        System.out.println(Thread.currentThread().getName() + " ... ");
        /**
         * 这里面要写自己的划分逻辑
         * 构造ForkJoin
         */
        int sum = 0;
        // 拆分任务
        if (last - first <= 2) {
            // 计算
            for (int i = first; i <= last; i++) {
                sum += i;
            }
        } else {
            /**
             * 类似于分支递归思想
             */
            RecursiveTaskDemo demo01 = new RecursiveTaskDemo(first, (last + first) / 2);
            RecursiveTaskDemo demo02 = new RecursiveTaskDemo((last + first) / 2 + 1, last);

            // 执行
            demo01.fork();
            demo02.fork();

            Integer a = demo01.join();
            Integer b = demo02.join();

            sum = a + b;
        }
        return sum;
    }

    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool(3);
        Future<Integer> future = forkJoinPool.submit(new RecursiveTaskDemo(1, 100));
        System.out.println("处理其他程序...");
        try {
            System.out.println("计算的值为：" + future.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}

```









[IBM-JDK 7 中的 Fork/Join 模式][2]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1523368521354.jpg
  [2]: https://www.ibm.com/developerworks/cn/java/j-lo-forkjoin/
