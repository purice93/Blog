---
title: 并发编程实战9-Java中的并发工具类
tags: CountDownLatch,CyclicBarrier,Semaphore,Exchanger
grammar_cjkRuby: true
---
>在JDK的并发包里提供了几个非常有用的并发工具类。CountDownLatch、CyclicBarrier和Semaphore工具类提供了一种并发流程控制的手段，Exchanger工具类则提供了在线程间交换数据的一种手段。

一、等待多线程完成的CountDownLatch

CountDownLatch允许一个或多个线程等待其他线程完成操作。
比如我们现在需要一个计算c=a+b总和的任务，但是其中a有a+10,且b有b+100;总和的计算需要等待a和b全部计算完成才能开始，所以需要等待。显然，最简单的方法，可以直接使用join方法来实现。如下：

``` stylus
package com.sound.daytb4;

/**
 * 1、join实现等待
 */
public class JoinCountDownLatchTest {
    int a = 1;
    int b = 2;
    int c = 0;
    public static void main(String[] args) throws InterruptedException {
        JoinCountDownLatchTest demo = new JoinCountDownLatchTest();
        Thread cal1 = new Thread(new Runnable() {
            @Override
            public void run() {
                demo.a = demo.a + 10;
                System.out.println("计算第一部分，结果为a = "+demo.a);
            }
        });
        Thread cal2 = new Thread(new Runnable() {
            @Override
            public void run() {
                demo.b = demo.b + 100;
                System.out.println("计算第二部分，结果为b = "+demo.b);
            }
        });
        cal1.start();
        cal2.start();
        cal1.join();
        cal2.join();
        System.out.println("等待前两部分计算完...");
        System.out.println("将第一部分和第二部分相加为，c = "+ (demo.a+demo.b));
    }
    /**
     *
     计算第一部分，结果为a = 11
     计算第二部分，结果为b = 102
     等待前两部分计算完...
     将第一部分和第二部分相加为，c = 113
     */
}
```
**CountDownLatch与join的区别**
但是，相对于join，CountDownLatch功能更多，或者说是更加灵活，可以在内部随时完成。
考虑一种情况，对于上面的逻辑，假如，函数中还有其他计算，但是这些计算不算影响c的计算，所以就需要提前唤醒c，而不需要再等待了。

``` stylus
package com.sound.daytb4;

import java.util.concurrent.CountDownLatch;

/**
 * 2、countDownLatch实现等待
 */
public class CountDownLatchTest {
    static CountDownLatch countDownLatch = new CountDownLatch(2);
    int a = 1;
    int b = 2;
    int c = 0;
    public static void main(String[] args) throws InterruptedException {
        CountDownLatchTest demo = new CountDownLatchTest();
        new Thread(new Runnable() {
            @Override
            public void run() {
                demo.a = demo.a + 10;
                System.out.println("计算第一部分，结果为a = "+demo.a);
                countDownLatch.countDown();
                demo.b = demo.b + 100;
                System.out.println("计算第二部分，结果为b = "+demo.b);
                countDownLatch.countDown();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("继续计算其他任务");
            }
        }).start();
        countDownLatch.await();
        System.out.println("等待前两部分计算完...");
        System.out.println("将第一部分和第二部分相加为，c = "+ (demo.a+demo.b));
    }

    /**
     *
     计算第一部分，结果为a = 11
     计算第二部分，结果为b = 102
     等待前两部分计算完...
     将第一部分和第二部分相加为，c = 113
     继续计算其他任务
     */
}
```
**CountDownLatch内部逻辑**
CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完
成，这里就传入N。
当我们调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法
会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个
点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个
CountDownLatch的引用传递到线程里即可。

二、同步/循环屏障CyclicBarrier

CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。
这相当于我们平时开会，屏障就相当于会议设置的固定人数和会议内容，每个人相当于一个线程，只有人到齐了，会议才会开始。
例子如下：

``` stylus
package com.sound.daytb5;

import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {

	public void meeting(CyclicBarrier barrier) {
		System.out.println(Thread.currentThread().getName() + " 到达会议室，等待开会..");

		if(Thread.currentThread().getName().equals("Thread-7")) {
			System.out.println("Thread-7 出车祸了，到不了了，会议将无法开始");
			try {
				Thread.sleep(10000);
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			barrier.reset();
		}

		try {
			barrier.await();
		} catch (Exception e) {
			e.printStackTrace();
		}

	}

	public static void main(String[] args) {
		CyclicBarrierDemo demo = new CyclicBarrierDemo();

		// 定义会议人数:10 和 内容run(){}
		CyclicBarrier barrier = new CyclicBarrier(10, new Runnable() {
			@Override
			public void run() {
				System.out.println("好！我们开始开会...");
			}
		});

		for (int i = 0; i < 12; i++) {
			new Thread(new Runnable() {
				@Override
				public void run() {
					demo.meeting(barrier);
				}
			}).start();
		}

		// 监控等待线程数
		new Thread(new Runnable() {

			@Override
			public void run() {
				while(true) {
					try {
						Thread.sleep(1000);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("等待的线程数 " + barrier.getNumberWaiting());
					System.out.println("屏障是否损坏or异常？ " + barrier.isBroken());
				}
			}
		}).start();
	}

}

```

二.1、CyclicBarrier和CountDownLatch的区别
CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。

三、控制并发线程数的Semaphore
Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。
把它比作是控制流量的红绿灯。比如××马路要限制流量，只允许同时有一百辆车在这条路上行使，其他的都必须在路口等待，所以前一百辆车会看到绿灯，可以开进这条马路，后面的车会看到红灯，不能驶入××马路，但是如果前一百辆中有5辆车已经离开了××马路，那么后面就允许有5辆车驶入马路，这个例子里说的车就是线程，驶入马路就表示线程在执行，离开马路就表示线程执行完成，看见红灯就表示线程被阻塞，不能执行。
	* 应用场景：Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，就可以使用Semaphore来做流量控制。如下：
``` stylus
package com.sound.daytb5;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

public class SemaphoreTest {
    private static final int THREAD_COUNT = 30;
    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);
    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                        System.out.println("save data");
                        s.release();
                    } catch (InterruptedException e) {
                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```
**与线程池的区别**
在上面的代码中，我们其实可以直接创建一个大小为10的线程池，这样不就好了吗？其实这两个东西完全不一样。即Semaphore是为了解决资源共享冲突的并发数目，而线程池只是一个线程池，池子里的线程并不一定是资源冲突的。就相当于，线程池是一个家，家里有猫、有狗、有猪，但是猫和猪并不会因为吃的东西而打架，它两吃的食物不一样。但是，同一个猪圈的猪却不同了，猪圈就相当于Semaphore信号量，它们都是吃一个槽里的食物，当然会打起来，所以就需要并发控制，比如，由于资源有限，一个猪圈里面只有10个槽，所以，每个猪圈就限制只能住10头猪，等这些猪长大了杀了（线程死亡），其他猪才能再进来。

* 四、线程间交换数据的Exchanger
Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。
	* Exchanger的应用场景：
		* Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。
		* Exchanger也可以用于校对工作，比如我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否录入一致。

``` stylus
package com.sound.daytb5;

import java.util.concurrent.Exchanger;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExchangerTest {
    private static final Exchanger<String> exgr = new Exchanger<String>();
    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println("银行流水A开始执行...");
                    String A = "银行流水A"; // A录入银行流水数据
                    exgr.exchange(A);
                } catch (InterruptedException e) {
                }
            }
        });
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println("银行流水B开始执行...");
                    String B = "银行流水B";// B录入银行流水数据
                    String A = exgr.exchange("B");
                    System.out.println("A和B数据是否一致：" + A.equals(B) + "，A录入的是："
                            + A + "，B录入是：" + B);
                } catch (InterruptedException e) {
                }
            }
        });
        threadPool.shutdown();
    }
}
```


