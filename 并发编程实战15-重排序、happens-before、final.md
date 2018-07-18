---
title: 并发编程实战15-重排序、happens-before
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
* 指令重排序
为了优化CPU的运行效率，在条件允许的情况下，直接运行当前有能力立即执行的后续指令，避开获取下一条指令所需数据时造成的等待3。通过乱序执行的技术，处理器可以大大提高执行效率。
比如：对于如下代码

``` stylus
int a = 10 // 1 
int b = 100 // 2
int c = a // 3
```
实际的执行过程可能会是：1-3-2，而不是：1-2-3；因为第一步获取a的值后，第三部仍然需要使用，此时，由于第二步并不会干扰单线程下程序了逻辑，将会直接执行3，再执行2.避免二次读取a值。（只是说明可能的原理，例子并不一定正确）

* 数据依赖性（as-if-serial）
	* As-if-serial语义的意思是，所有的动作(Action)5都可以为了优化而被重排序，但是必须保证它们重排序后的结果和程序代码本身的应有结果是一致的。
即
	* 即如果后续逻辑计算需要依赖之前的某个值a，那么a这个值的计算步骤不能跳过。如上代码：第3步，c的值需要使用a，所以不能跳过1直接执行3

* happens-before 规则
语义：如果A先发生于B，那么A所做的所有改变都能被B看到
Happens-before是用来指定两个操作之间的执行顺序。提供跨线程的内存可见性。在Java内存模型中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必然存在happens-before关系。

遵循的规则：
	* 程序顺序规则：一个线程中的每个操作，happens- before 于该线程中的任意后续操作。
	* 监视器锁规则：对一个监视器锁的解锁，happens- before 于随后对这个监视器锁的加锁。
	* volatile变量规则：对一个volatile域的写，happens- before 于任意后续对这个volatile域的读。
	* 传递性：如果A happens- before B，且B happens- before C，那么A happens- before C。


* 锁和volatile的内存语义：（JMM）
	* 锁的获取：首先清空当前线程value内存，从主存中获取最新值；锁的释放：将当前线程内存value刷新到主内存
	* volatile的读写与锁的获取和释放对应，原理类似

* final域的重排序规则
对于final域，编译器和处理器要遵守两个重排序规则

1> 在构造函数内对一个final域的写入，与随后把这个构造函数的引用赋值给一个引用变量，两个操作不能重排序
2> 初次读一个包含final域对象的引用，和随后初次读这个final域，这两个操作不能重排序  


[美团点评博客-Java内存访问重排序的研究][1]
[深入理解Java内存模型（五）——锁][2]
[java多线程学习(九)final的内存语义][3]


  [1]: https://tech.meituan.com/java-memory-reordering.html
  [2]: http://www.infoq.com/cn/articles/java-memory-model-5
  [3]: https://blog.csdn.net/ditto_zhou/article/details/78738197
