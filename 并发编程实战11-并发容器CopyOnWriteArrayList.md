---
title: 并发编程实战11-并发容器CopyOnWriteArrayList
tags: CopyOnWriteArrayList,模板,小书匠
grammar_cjkRuby: true
---
一、同步容器和并发容器
在jdk早期，为了解决并发安全问题，引入了同步容器Vector和Hashtable。在JDK1.2中，引入了同步封装类，可以由Collections.synchronizedXxxx等方法创建，可以直接对ArrayList进行封装以达到同步。但是，同步容器有一个问题，过于严格，即完全串行执行，导致即便是在复合操作下，并没有线程安全问题，也会加锁。
常见复合操作如下：
	* 迭代：反复访问元素，直到遍历完全部元素；
	* 跳转：根据指定顺序寻找当前元素的下一个（下n个）元素；
	* 条件运算：例如若没有则添加等；

同步容器对所有容器状态的访问都串行化，严重降低了并发性；当多个线程竞争锁时，吞吐量严重下降；

java5.0之后提供了多种并发容器来改善同步容器的性能，如ConcurrentHashMap、CopyOnWriteArrayList、CopyOnWriteArraySet、ConcurrentSkipListMap等；这些容器通过进行优化以达到适应不同需求的目的，并没有完全串行化。


|  一般容器   |   同步容器  |  并发容器   |
| -----  | --- | --- |
|  ArrayList   |  Vector   |   CopyOnWriteArrayList  |
|   HashMap  |  HashTable   |   ConcurrentHashMap  |


一般容器线程不安全；同步容器线程安全，但是过于严格；为了满足实际需求，合理适应需求，增加了并发容器，更好的适应并发条件，即同步容器为了安全，并发容器为了安全的同时，达到高并发的效率。

二、源码分析：CopyOnWriteArrayList
在前面了解读写锁时就已经知道，在实际情况中，往往读取更多，写入很少，而读取数据并不影响数据的安全，所以读数据完全可以并发进行，而只将写数据进行严格加锁。

* 当修改时（添加和移除），就对数组进行复制，此时要加锁。
* 当读取时，不加锁，直接读取

``` stylus
/** The lock protecting all mutators */
// 1、用于加锁的可重入锁
final transient ReentrantLock lock = new ReentrantLock();
/** The array, accessed only via getArray/setArray. */
// 2、存取数据的volatile数组
private transient volatile Object[] array;
```

3、添加元素
```
// 3、添加元素
 public boolean add(E e) {
        final ReentrantLock lock = this.lock;                       // 获取独占锁
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);// 重新生成一个新的数组实例，并将原始数组的元素拷贝到新数组中
            newElements[len] = e;                                   // 添加新的元素到新数组的末尾
            setArray(newElements);                                  // 更新底层数组
            return true;
        } finally {
            lock.unlock();
        }
    }
```
有两点必须清楚：
 	* 第一，在”添加操作“开始前，获取独占锁(lock)，若此时有需要线程要获取锁，则必须等待；在操作完毕后，释放独占锁(lock)，此时其它线程才能获取锁。通过独占锁，来防止多线程同时修改数据！此时还是可以读取，只是读取的是原来的数组。这段时间内，难道不发生安全问题吗？
	* 第二，操作完毕时，会通过setArray()来更新volatile数组。对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入；这样，每次添加元素之后，其它线程都能看到新添加的元素。

4、删除数据
remove数据和add类似，也是复制副本

``` stylus
public E remove(int index) {  
    final ReentrantLock lock = this.lock;  
    lock.lock();  
    try {  
        Object[] elements = getArray();  
        int len = elements.length;  
        E oldValue = get(elements, index); // 获取volatile数组中指定索引处的元素值  
        int numMoved = len - index - 1;  
        if (numMoved == 0) // 如果被删除的是最后一个元素，则直接通过Arrays.copyOf()进行处理，而不需要新建数组  
            setArray(Arrays.copyOf(elements, len - 1));  
        else {  
            Object[] newElements = new Object[len - 1];  
            System.arraycopy(elements, 0, newElements, 0, index);    // 拷贝删除元素前半部分数据到新数组中  
            System.arraycopy(elements, index + 1, newElements, index, numMoved);// 拷贝删除元素后半部分数据到新数组中  
            setArray(newElements); // 更新volatile数组  
        }  
        return oldValue;  
    } finally {  
        lock.unlock();  
    }  
} 
```


5、读取数据
将底层volatile数组指定索引处的元素返回即可。
``` stylus 
public E get(int index) {
        return get(getArray(), index);
    }
private E get(Object[] a, int index) {
        return (E) a[index];
    }
```
6、遍历元素

``` stylus
    public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
    }
    public ListIterator<E> listIterator() {
        return new COWIterator<E>(getArray(), 0);
    }
    public ListIterator<E> listIterator(final int index) {
        Object[] elements = getArray();
        int len = elements.length;
        if (index<0 || index>len)
            throw new IndexOutOfBoundsException("Index: "+index);

        return new COWIterator<E>(elements, index);
    }

    private static class COWIterator<E> implements ListIterator<E> {
        private final Object[] snapshot; // 保存数组的快照，是一个不可变的对象
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }
        public void remove() {
            throw new UnsupportedOperationException();
        }
        public void set(E e) {
            throw new UnsupportedOperationException();
        }
        public void add(E e) {
            throw new UnsupportedOperationException();
        }
    }
```
如上容器的迭代器中会保存一个不可变的Object数组对象，那么在进行遍历这个对象时就不需要再进一步的同步。在每次修改时，都会创建并重新发布一个新的窗口副本，从而实现了可变性。如上迭代器代码中保留了一个指向volatile数组的引用，由于不会被修改，因此多个线程可以同时对它进行迭代，而不会彼此干扰或与修改容器的线程相互干扰。
与之前的ArrayList实现相比，CopyOnWriteArrayList返回迭代器不会抛出ConcurrentModificationException异常，即它不是fail-fast机制的！

三、CopyOnWrite的应用场景
CopyOnWrite并发容器用于读多写少的并发场景。比如缓存；白名单，黑名单，商品类目的访问和更新场景，假如我们有一个搜索网站，用户在这个网站的搜索框中，输入关键字搜索内容，但是某些关键字不允许被搜索。这些不能被搜索的关键字会被放在一个黑名单当中，黑名单每天晚上更新一次。当用户搜索时，会检查当前关键字在不在黑名单当中，如果在，则提示不能搜索。

使用CopyOnWriteMap需要注意两件事情：

	* 1. 减少扩容开销。根据实际需要，初始化CopyOnWriteMap的大小，避免写时CopyOnWriteMap扩容的开销。
	
	* 2. 使用批量添加。因为每次添加，容器每次都会进行复制，所以减少添加次数，可以减少容器的复制次数。如使用上面代码里的addBlackList方法。

CopyOnWrite的缺点：CopyOnWrite容器有很多优点，但是同时也存在两个问题，即内存占用问题和数据一致性问题。所以在开发的时候需要注意一下。
	* 内存占用问题。因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Yong GC和Full GC。之前我们系统中使用了一个服务由于每晚使用CopyOnWrite机制更新大对象，造成了每晚15秒的Full GC，应用响应时间也随之变长。针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，如ConcurrentHashMap。
	* 数据一致性问题/最终一致。CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。即只能保证数据的最终一致性，而不能保证实时一致。最终一致对于分布式系统也非常重要，它通过容忍一定时间的数据不一致，提升整个分布式系统的可用性与分区容错性。当然，最终一致并不是任何场景都适用的，像火车站售票这种系统用户对于数据的实时性要求非常非常高，就必须做成强一致性的。

链接：https://www.nowcoder.com/questionTerminal/95e4f9fa513c4ef5bd6344cc3819d3f7
来源：牛客网

**补充：**
一：快速失败（fail—fast）
	
* 在用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行了修改（增加、删除、修改），则会抛出Concurrent Modification Exception。
* 原理：迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个 modCount 变量。集合在被遍历期间如果内容发生变化，就会改变modCount的值。每当迭代器使用hashNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedmodCount值，是的话就返回遍历；否则抛出异常，终止遍历。
* 注意：这里异常的抛出条件是检测到 modCount！=expectedmodCount 这个条件。如果集合发生变化时修改modCount值刚好又设置为了expectedmodCount值，则异常不会抛出。因此，不能依赖于这个异常是否抛出而进行并发操作的编程，这个异常只建议用于检测并发修改的bug。
* 场景：java.util包下的集合类都是快速失败的，不能在多线程下发生并发修改（迭代过程中被修改）。

二：安全失败（fail—safe）(克隆容器Copy)

* 采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。
* 原理：由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发Concurrent Modification Exception。
* 场景：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。


[Java 7之多线程并发容器 - CopyOnWriteArrayList][1]
[CopyOnWriteArrayList详解][2]






  [1]: https://blog.csdn.net/mazhimazh/article/details/19210547
  [2]: https://blog.csdn.net/caomiao2006/article/details/53232687
