---
title:  并发编程实战14-LongAdder统计加法器-计数器jdk8
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
传统的原子锁AtomicLong/AtomicInt虽然也可以处理大量并发情况下的计数器，但是由于使用了自旋等待，当存在大量竞争时，会存在大量自旋等待，而导致CPU浪费，而有效计算很少，降低了计算效率。

而LongAdder是根据ConcurrentHashMap这类为并发设计的类的基本原理——锁分段，通过维护一个计数数组cells来保存多个计数副本，每个线程只对自己的副本进行操作，最后汇总来得到最终结果。

>Adder的英文意思为加法器，从字面意思上就可以理解，LongAdder的作用是用来进行统计计算的。比如，我们需要一个计数器，加入计数值为N，当大量的线程访问时，N的值将会出现并发安全，但是，我们并不打算在每个单独的子线程中去查看这个N值，只是在所有的子线程完毕后，才会统计N的总数，即相当于一种并行计算，但是我们不在乎中间结果，只在乎，中间计算都进行了就可以。因此，我们对可以变量创建一个副本N'，每个线程都有这个副本，只对这个副本进行操作，最后，将所有的副本进行汇总就是最终的N值，即：`N=N1'+N2'+N3'...+Nn'`。

>详细原理：内部源码使用了一个cells的数组来保存每个线程的副本，第一个线程会初始化数组cells。新来到一个线程，会指向第一个cell，并检测是否有其他线程使用CAS，如果没有，则所有线程公用一个cel副本；否则，如果自旋等待，则当前线程新建一个cell副本，加入到数组cells，对副本进行操作。当所有操作完成后，使用sum函数就可以统计所有的操作。

**这里也用到了cas操作，主要是为了防止多线程公用一个cell，导致一个cell数据并发出错，另外注意LongAdder的并不是细粒度的，所以不能使用中间值**

 1. 线程副本cell
 LongAdder继承自Striped64类，主要逻辑方法在Striped64中。
 其中cell即为每个线程的副本变量，使用的是一个静态内部类Cell，维护了一个value变量
 

``` stylus
@sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```
问题：既然Cell这么简单，为什么不直接用long value？
我的理解：可能要设计到Java值引用问题，如果直接使用long value，在线程B进行数组扩容后，线程A修改的value值将无法反映到线程B；而使用Cell对象对value进行包装后，由于线程A只会修改Cell对象的value值，不会修改Cell对象，所以线程B的Cell中的value也会跟着改变，就不会出现值问题。更具体的了解，可能需要了解一下java的值传递（java没有引用传递）


 2.Striped64主体代码
 主方法为longAccumulate方法，主要处理数组的扩容和冲突检查

``` stylus
abstract class Striped64 extends Number {  
    @sun.misc.Contended static final class Cell { ... }  
  
    /** Number of CPUS, to place bound on table size */  
    static final int NCPU = Runtime.getRuntime().availableProcessors();  
  
    // cell数组，长度一样要是2^n，可以类比为jdk1.7的ConcurrentHashMap中的segments数组  
    transient volatile Cell[] cells;  
  
    // 累积器的基本值，在两种情况下会使用：  
    // 1、没有遇到并发的情况，直接使用base，速度更快；  
    // 2、多线程并发初始化table数组时，必须要保证table数组只被初始化一次，因此只有一个线程能够竞争成功，这种情况下竞争失败的线程会尝试在base上进行一次累积操作  
    transient volatile long base;  
  
    // 自旋标识，在对cells进行初始化，或者后续扩容时，需要通过CAS操作把此标识设置为1（busy，忙标识，相当于加锁），取消busy时可以直接使用cellsBusy = 0，相当于释放锁  
    transient volatile int cellsBusy;  
  
    Striped64() {  
    }  
  
    // 使用CAS更新base的值  
    final boolean casBase(long cmp, long val) {  
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);  
    }  
  
    // 使用CAS将cells自旋标识更新为1  
    // 更新为0时可以不用CAS，直接使用cellsBusy就行  
    final boolean casCellsBusy() {  
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);  
    }  
  
    // 下面这两个方法是ThreadLocalRandom中的方法，不过因为包访问关系，这里又重新写一遍  
  
    // probe翻译过来是探测/探测器/探针这些，不好理解，它是ThreadLocalRandom里面的一个属性，  
    // 不过并不影响对Striped64的理解，这里可以把它理解为线程本身的hash值  
    static final int getProbe() {  
        return UNSAFE.getInt(Thread.currentThread(), PROBE);  
    }  
  
    // 相当于rehash，重新算一遍线程的hash值  
    static final int advanceProbe(int probe) {  
        probe ^= probe << 13;   // xorshift  
        probe ^= probe >>> 17;  
        probe ^= probe << 5;  
        UNSAFE.putInt(Thread.currentThread(), PROBE, probe);  
        return probe;  
    }  
  
    /** 
     * 核心方法的实现，此方法建议在外部进行一次CAS操作（cell != null时尝试CAS更新base值，cells != null时，CAS更新hash值取模后对应的cell.value） 
     * @param x the value 前面我说的二元运算中的第二个操作数，也就是外部提供的那个操作数 
     * @param fn the update function, or null for add (this convention avoids the need for an extra field or function in LongAdder). 
     *     外部提供的二元算术操作，实例持有并且只能有一个，生命周期内保持不变，null代表LongAdder这种特殊但是最常用的情况，可以减少一次方法调用 
     * @param wasUncontended false if CAS failed before call 如果为false，表明调用者预先调用的一次CAS操作都失败了 
     */  
    final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {  
        int h;  
        // 这个if相当于给线程生成一个非0的hash值  
        if ((h = getProbe()) == 0) {  
            ThreadLocalRandom.current(); // force initialization  
            h = getProbe();  
            wasUncontended = true;  
        }  
        boolean collide = false; // True if last slot nonempty 如果hash取模映射得到的Cell单元不是null，则为true，此值也可以看作是扩容意向，感觉这个更好理解  
        for (;;) {  
            Cell[] as; Cell a; int n; long v;  
            if ((as = cells) != null && (n = as.length) > 0) { // cells已经被初始化了  
                if ((a = as[(n - 1) & h]) == null) { // hash取模映射得到的Cell单元还为null（为null表示还没有被使用）  
                    if (cellsBusy == 0) {       // Try to attach new Cell 如果没有线程正在执行扩容  
                        Cell r = new Cell(x);   // Optimistically create 先创建新的累积单元  
                        if (cellsBusy == 0 && casCellsBusy()) { // 尝试加锁  
                            boolean created = false;  
                            try {               // Recheck under lock 在有锁的情况下再检测一遍之前的判断  
                                Cell[] rs; int m, j;  
                                if ((rs = cells) != null && (m = rs.length) > 0 && rs[j = (m - 1) & h] == null) { // 考虑别的线程可能执行了扩容，这里重新赋值重新判断  
                                    rs[j] = r; // 对没有使用的Cell单元进行累积操作（第一次赋值相当于是累积上一个操作数，求和时再和base执行一次运算就得到实际的结果）  
                                    created = true;  
                                }  
                            } finally {  
                                cellsBusy = 0; 清空自旋标识，释放锁  
                            }  
                            if (created) // 如果原本为null的Cell单元是由自己进行第一次累积操作，那么任务已经完成了，所以可以退出循环  
                                break;  
                            continue;           // Slot is now non-empty 不是自己进行第一次累积操作，重头再来  
                        }  
                    }  
                    collide = false; // 执行这一句是因为cells被加锁了，不能往下继续执行第一次的赋值操作（第一次累积），所以还不能考虑扩容  
                }  
                else if (!wasUncontended) // CAS already known to fail 前面一次CAS更新a.value（进行一次累积）的尝试已经失败了，说明已经发生了线程竞争  
                    wasUncontended = true; // Continue after rehash 情况失败标识，后面去重新算一遍线程的hash值  
                else if (a.cas(v = a.value, ((fn == null) ? v + x : fn.applyAsLong(v, x)))) // 尝试CAS更新a.value（进行一次累积） ------ 标记为分支A  
                    break; // 成功了就完成了累积任务，退出循环  
                else if (n >= NCPU || cells != as) // cell数组已经是最大的了，或者中途发生了扩容操作。因为NCPU不一定是2^n，所以这里用 >=  
                    collide = false; // At max size or stale 长度n是递增的，执行到了这个分支，说明n >= NCPU会永远为true，下面两个else if就永远不会被执行了，也就永远不会再进行扩容  
                                     // CPU能够并行的CAS操作的最大数量是它的核心数（CAS在x86中对应的指令是cmpxchg，多核需要通过锁缓存来保证整体原子性），当n >= NCPU时，再出现几个线程映射到同一个Cell导致CAS竞争的情况，那就真不关扩容的事了，完全是hash值的锅了  
                else if (!collide) // 映射到的Cell单元不是null，并且尝试对它进行累积时，CAS竞争失败了，这时候把扩容意向设置为true  
                                   // 下一次循环如果还是跟这一次一样，说明竞争很严重，那么就真正扩容  
                    collide = true; // 把扩容意向设置为true，只有这里才会给collide赋值为true，也只有执行了这一句，才可能执行后面一个else if进行扩容  
                else if (cellsBusy == 0 && casCellsBusy()) { // 最后再考虑扩容，能到这一步说明竞争很激烈，尝试加锁进行扩容 ------ 标记为分支B  
                    try {  
                        if (cells == as) {      // Expand table unless stale 检查下是否被别的线程扩容了（CAS更新锁标识，处理不了ABA问题，这里再检查一遍）  
                            Cell[] rs = new Cell[n << 1]; // 执行2倍扩容  
                            for (int i = 0; i < n; ++i)  
                                rs[i] = as[i];  
                            cells = rs;  
                        }  
                    } finally {  
                        cellsBusy = 0; // 释放锁  
                    }  
                    collide = false; // 扩容意向为false  
                    continue; // Retry with expanded table 扩容后重头再来  
                }  
                h = advanceProbe(h); // 重新给线程生成一个hash值，降低hash冲突，减少映射到同一个Cell导致CAS竞争的情况  
            }  
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) { // cells没有被加锁，并且它没有被初始化，那么就尝试对它进行加锁，加锁成功进入这个else if  
                boolean init = false;  
                try {                           // Initialize table  
                    if (cells == as) { // CAS避免不了ABA问题，这里再检测一次，如果还是null，或者空数组，那么就执行初始化  
                        Cell[] rs = new Cell[2]; // 初始化时只创建两个单元  
                        rs[h & 1] = new Cell(x); // 对其中一个单元进行累积操作，另一个不管，继续为null  
                        cells = rs;  
                        init = true;  
                    }  
                } finally {  
                    cellsBusy = 0; // 清空自旋标识，释放锁  
                }  
                if (init) // 如果某个原本为null的Cell单元是由自己进行第一次累积操作，那么任务已经完成了，所以可以退出循环  
                    break;  
            }  
            else if (casBase(v = base, ((fn == null) ? v + x : fn.applyAsLong(v, x)))) // cells正在进行初始化时，尝试直接在base上进行累加操作  
                break;                          // Fall back on using base 直接在base上进行累积操作成功了，任务完成，可以退出循环了  
        }  
    }  
  
    // double的不讲，更long的逻辑基本上是一样的  
    final void doubleAccumulate(double x, DoubleBinaryOperator fn, boolean wasUncontended);  
  
    // Unsafe mechanics Unsafe初始化  
    private static final sun.misc.Unsafe UNSAFE;  
    private static final long BASE;  
    private static final long CELLSBUSY;  
    private static final long PROBE;  
    static {  
        try {  
            UNSAFE = sun.misc.Unsafe.getUnsafe();  
            Class<?> sk = Striped64.class;  
            BASE = UNSAFE.objectFieldOffset  
                (sk.getDeclaredField("base"));  
            CELLSBUSY = UNSAFE.objectFieldOffset  
                (sk.getDeclaredField("cellsBusy"));  
            Class<?> tk = Thread.class;  
            PROBE = UNSAFE.objectFieldOffset  
                (tk.getDeclaredField("threadLocalRandomProbe"));  
        } catch (Exception e) {  
            throw new Error(e);  
        }  
    }  
  
} 
```

 3.LongAdder的add()方法
 add(long x) 方法不仅仅实现加，参数为负就为减；调用longAccumulate来进行操作

``` stylus
    public void add(long x) {  
        Cell[] as; long b, v; int m; Cell a;  
        if ((as = cells) != null || !casBase(b = base, b + x)) {  
            boolean uncontended = true;  
            if (as == null || (m = as.length - 1) < 0 ||  
                (a = as[getProbe() & m]) == null ||  
                !(uncontended = a.cas(v = a.value, v + x)))  
                longAccumulate(x, null, uncontended);  
        }  
    }  
```


**jdk1.8的ConcurrentHashMap中，没有再使用Segment，使用了一个简单的仿造LongAdder实现的计数器，这样能够保证计数效率不低于使用Segment的效率。**


[jdk1.8 LongAdder源码学习][1]

 


  [1]: https://blog.csdn.net/u011392897/article/details/60480108
