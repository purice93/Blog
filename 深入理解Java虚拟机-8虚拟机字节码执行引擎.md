---
title: 深入理解Java虚拟机-8虚拟机字节码执行引擎 
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
参考：
https://zhuanlan.zhihu.com/p/28468115 
http://www.cnblogs.com/deman/p/5489895.html
8.1. 
* 概述：相对于物理机而言，Java虚拟机相当于一座架设在开发人员与物理机之间的“桥梁”。一端接受的是开发人员的字节码文件，另一端将字节码转化为物理机能够执行的机器码。这样，只要有java虚拟机存在的地方就可以使用虚拟机层面上的这种语言，达到“一次编写，处处运行”的目的。通过这个定义我们知道，一切可以编译出字节码的语言都可以获得这种“平台无关性”，也就是说像一些类Java语言比如Groovy Scala等，因为用他们也可以生成字节码，所以也可以用Java虚拟机来执行，也就具有了平台无关性。所以在后期，java虚拟机不单单只给java提供，其他相关的这种语言也能使用。
* 执行引擎是指java字节码的执行策略，一般分为两种：解释执行、编译执行。所有的java虚拟机执行引擎都是一致的，将字节码解析为机器码，本章主要讲方法的调用和字节码执行。


----------

8.2. 运行时栈帧结构
* 虚拟机栈：想一想我们学习递归时，方法都是一层一层迭代的，但是我们如何在子方法执行完成后，找打父方法。实际上是一种栈结构，递归就是一种栈的思想。对于虚拟机而言，也是通过这种思想来设计虚拟机方法的调用，方法的调用和调用结束相当于是栈的入栈和出栈操作。
* 栈帧：栈帧是虚拟机栈的单个元素，相当于一个方法，其中存储了方法的局部变量表、操作数栈、动态链接和方法返回地址、以及其他一些附加信息。
	* 在编译程序代码的时候，栈帧需要多大的局部变量表、多深的操作数栈都已经完全确定了，并且写入到方法表的Code属性之中，因此一个栈帧需要分配多大的内存，并不会受到运行期变量数据的影响，而仅仅取决于具体的虚拟机的实现。

8.2.1.局部变量表
* 方法的局部变量表包括方法的参数和方法内部定义的局部变量。java编译时，局部变量表的存储大小就已经确定。
* 局部变量槽(Slot)：用于存储局部变量，对于以下八种数据类型：boolean、byte、char、short、int、float、reference或returnAddress，要求一个Slot只能存储一个数据，他们的大小分别是1、1、1、2、4、4、4、4；一个slot一般就是4个字节(32位)，这样存储时恰好满足这种情况。但是Slot大小并不固定，取决于具体的虚拟机。
* 对于8字节(64位)数据类型long、double，通过高危对齐来使用两个Slot来存储。虽然是需要两次读写，但是由于局部变量表是建立在线程上的堆栈，是线程私有的，所以两个Slot是原子性操作，不会发生安全问题。
* 局部变量表的分配：局部变量表索引从低到高依次是：方法的调用实例->方法参数->方法内局部变量。杜对于静态方法，就没有调用实例。
* **Slot重用**：为了节省栈空间，局部变量表Slot可以重用。比如、方法内部的局部变量快{}，有效区域只是在块内，当出了块区域，就没效了，但是垃圾回收机制却不能立刻回收。因此，这里虚拟机设计，只要变量的计数器值超出了作用范围就可以让给其他变量来使用。注意，这里是**可以**，如果没有其他值来占用，就依然占有。参见书中p240。这里作者提了一种实际操作另外一种方式：既然是为了回收不用的内存，可以直接在使用块区域对象变量之后将对象引用置为null，这样人为操作要准确很多。
* 开发注意-变量初始化：java中类变量即使不赋值也会在类加载阶段进行初始化，但是对于局部变量，由于没有准备阶段，无法初始化，所以局部变量必须手动初始化（int i= 0而不是int i），否则编译报错。


----------
8.2.2. 操作数栈
* 局部变量表只是变量的一种存储，而操作数栈就是一个方法中的计算逻辑存储，就相当于我们中缀表达式计算器类似，比如方法需要求解a+b,分别将a、b入栈，遇到+，然后出栈计算。
* 同局部变量表一样，操作数栈的最大深度也是在编译时就确定了。32位栈容量为1,64位栈容量为2。操作类型必须为同一类型int-int。
* 虚拟机优化-共享数据：一般而言，两个栈帧元素是相互独立的。但为了节省空间，也可以共享存储空间。通过将一个栈帧的共享局部变量表和另一个栈帧的共享操作数栈共享同样的区域。


----------
8.2.3. 动态连接
* 每一个栈帧对应一个方法，因此每一个栈帧都有这个方法的引用。比如方法的重载和重写中，我们到底调用哪个方法可能只能在运行期确定（静态调用编译器就可以确定）。具体的参照8.3，目前还没有弄得很清楚。


----------
8.2.4 方法返回地址：方法在调用结束后，恢复上层方法的局部变量表和操作数栈，并将返回值压到调用者的操作数栈中，同调整PC计数器的值以指向后面一条指令。


----------
8.2.5. 附加信息：其余一些信息，如调试信息。


----------
8.3. 方法调用
目的：确定条用那个版本的方法（重载重写），编译器确定还是运行期确定？这里主要讲的就是多态性！
方法调用的主要任务就是确定被调用方法的版本（即调用哪一个方法），该过程不涉及方法具体的运行过程。按照调用方式共分为两类：
1. 解析调用一定是静态的过程，在编译期间就完全确定目标方法。
2. 分派调用既可能是静态，也可能是动态的，根据分派标准可以分为单分派和多分派。两两组合有形成了静态单分派、静态多分派、动态单分派、动态多分派

----------

8.3.1. 解析调用
* 方法调用就是常量池中的符号引用，在类加载的解析阶段，会将其中的一部分转化为直接引用。运行期转化的就是间接引用。我们这里的解析单指直接引用。即在真正运行之前就已经确定了方法的调用版本。
* 在Class文件中，所有方法调用中的目标方法都是常量池中的符号引用，在类加载的解析阶段，会将一部分符号引用转为直接引用，也就是在编译阶段就能够确定唯一的目标方法，这类方法的调用成为解析调用。此类方法主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可访问，因此决定了他们都不可能通过继承或者别的方式重写该方法，符合这两类的方法主要有以下几种：静态方法、私有方法、实例构造器、父类方法。虚拟机中提供了以下几条方法调用指令：
	* invokestatic：调用静态方法，解析阶段确定唯一方法版本
	* invokespecial：调用<init>方法、私有及父类方法，解析阶段确定唯一方法版本
	* invokevirtual：调用所有虚方法
	* invokeinterface：调用接口方法
	* invokedynamic：动态解析出需要调用的方法，然后执行
前四条指令固化在虚拟机内部，方法的调用执行不可认为干预，而invokedynamic指令则支持由用户确定方法版本。其中invokestatic指令和invokespecial指令调用的方法称为非虚方法，其余的（final修饰的除外[^footnote4]）称为虚方法。


----------
8.3.2 分派调用
* 1. 静态分派：所有依赖静态类型3来定位方法执行版本的分派成为静态分派，发生在**编译阶段**，典型应用是方法**重载**。

``` stylus
public class BB  
{  
    static abstract class Human  
    {       
    }  
    static class Man extends Human  
    {     
    }  
    static class Woman extends Human  
    {        
    }  
      
    public void sayHello(Human guy)  
    {  
        System.out.println("hello guy");  
    }  
    public void sayHello(Man guy)  
    {  
        System.out.println("hello man");  
    }  
    public void sayHello(Woman guy)  
    {  
        System.out.println("hello woman");  
    }  
      
    public static void main(String[] args)  
    {  
        BB b = new BB();  
        Human man = new Man();//静态类型为Human  
        Human woman = new Woman();  
          
        b.sayHello(man);  
        b.sayHello(woman);  
    }  
}  

``` stylus
执行结果是:
hello guy
hello guy
原因：虚拟机在重载时是通过参数的静态类型而不是实际类型作为判定依据，并且静态类型是编译期可知的，所以在编译阶段，javac编译器就根据参数的静态类型决定使用哪个重载版本，并把这个方法的符号引用写入invokevirtual指令的参数中。

```

* 在某些情况下，静态分配并不能明显的确定调用那个类型，因此这里使用了更加合适的版本，即匹配与之对应更合适的那个方法版本。顺序为：当前类型-》向上转换（char-int-long-float-double）-》自动装箱为对象类型-》装箱类实现了的接口类型-》父类-》可变长参数方法 （6步）具体如下：

``` stylus
public class Verload {  
      
    private static void sayHello(char arg){  
        System.out.println("hello char");  
    }  
  
    private static void sayHello(Object arg){  
        System.out.println("hello Object");  
    }  
      
    private static void sayHello(int arg){  
        System.out.println("hello int");  
    }  
      
    private static void sayHello(Character arg){  
        System.out.println("hello long");  
    }  

    private static void sayHello(char... arg){  
        System.out.println("hello long");  
    }

    private static void sayHello(Serializable arg){  
        System.out.println("hello long");  
    }
      
    public static void main(String[] args) {  
        sayHello('c');  
    }  
    //一次查找：char-int-long-Character-Serializable-Object-char...
}  
```


* 2. 动态分派:在运行期间根据实际类型4来确定方法执行版本的分派成为动态分派，发生在**运行期间**，典型的应用是方法的**重写**。即对象调用的是父方法还是子方法运行时通过对象来确定能够，而不是对象的接口引用。重载相反。

``` stylus
public class AA  
{  
    static abstract class Human  
    {  
        protected abstract void sayHello();  
    }  
      
    static class Man extends Human  
    {  
        @Override  
        protected void sayHello()  
        {  
            System.out.println("man say hello");  
        }  
    }  
    static class Woman extends Human  
    {  
        @Override  
        protected void sayHello()  
        {  
            System.out.println("woman say hello");  
        }  
    }  
      
    public static void main(String[] args)  
    {  
        Human man = new Man();  
        Human woman = new Woman();  
        man.sayHello();  
        woman.sayHello();  
        man = new Woman();  
        man.sayHello();  
    }  
}  
执行结果：
man say hello
woman say hello
woman say hello
原因：
invokevirtual指令有多态查找的机制，该指令的运行时解析过程步骤如下：
1.找到操作数栈顶的第一个元素所指向的对象的实际类型，记做c
2.如果在类型c中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找过程结束，不通过则返回java.lang.IllegalAccessError.
3.否则，按照继承关系从下往上依次对c的各个父类进行第二步的搜索和验证过程。
4.始终没找到合适的方法，抛出java.lang.AbstractMethodError异常。
这就是java语言中方法重写的本质。
```
* **单分派和多分派：

``` stylus
public class disPatch {
	static class QQ {
	}
	static class _360 {
	}
	public static class Father {
		public void hardChoice(QQ arg) {
			System.out.println("father choose qq");
		}
		public void hardChoice(_360 arg) {
			System.out.println("father choose _360");
		}
	}
	public static class Son extends Father {
		public void hardChoice(QQ arg) {
			System.out.println("son choose qq");
		}
		public void hardChoice(_360 arg) {
			System.out.println("son choose _360");
		}
	}
	public static void main(String[] args) {
		Father father = new Father();
		Father son = new Son();
		father.hardChoice(new _360());
		son.hardChoice(new QQ());
	}
}
```

	* 方法的调用者和参数统称为方法的宗量，根据总量的数量，将分派分为单分派的多分派。我这里的理解就是单分派和双分派，调用者为一个宗量，所有的方法参数统称为另一个宗量。
	* java实际上是静态多分派和动态单分派
	* 静态多分派：比如上面的例子，son.hardChoice(new QQ()); 我们需要确定调用类型是Son还是father，同时还需要确定方法参数是360还是QQ，这个都是需要编译器指引的。
	* 动态单分派：还是上面这个例子，由于编译期已经确定了方法的参数即hardChoice(new QQ())，只是不知道需要确定调用者，运行期son实际类型为Father
	* 一句话说明：就是重载是编译期的事，重写是运行期的事；**只有运行期才能够确定调用的是父类还是子类方法。**	

8.3.3. 动态语言支持
* java是一种编译型语言，能够编写时就查错，更加稳定，但是由于过多的检查和为了稳定导致不够简化，开发不方便；python等是一种解释性语言，清晰简洁开发灵活，但是只能运行时才知道错误在哪。
* 但是为了权衡这种关系，java希望虚拟机能够为其他动态语言提供一种支持，这样就能够使用其他语言灵活性的特点。
* java. lang. invoke 包
JDK 1. 7 实现了 JSR- 292， 新加入的 java. lang. invoke 包[ 2] 就是 JSR- 292 的一个重要组成部分。

``` stylus
static class ClassA{
    public void println(String s){
        System.out.println(s);
    }
}

public static void main(String[] args) throws Throwable {
    Object obj = System.currentTimeMillis() % 2 == 0 ? System.out:new ClassA();
    /* 无论obj最终是那个实现类，下面这句都能正确调用到println方法 */
    getPrintlnMH(obj).invokeExact("icyfenix");
    /* output:
     * icyfenix
     */
}

private static MethodHandle getPrintlnMH(Object receiver) throws Throwable{
    /* MethodType: 代表“方法类型”，包含了方法的返回值（methodType()的第一个参数）
     * 和具体参数（methodType()第二个及以后的参数） */
    MethodType mt = MethodType.methodType(void.class, String.class);
    /* lookup()方法来自于MethodHandles.lookup，
     * 这句的作用是在指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄。
     * 因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，
     * 也即是this指向的对象，这个参数以前是放在参数列表中进行传递的，而现在提供给了bindTo()方法来完成这件事情 */
    return MethodHandles.lookup().findVirtual(receiver.getClass(), "println", mt)
            .bindTo(receiver);
}
```
MethodHandle 的使用方法和效果与 Reflection 有众多相似之处，不过，它们还是有以下这些区别：
Reflection 是在模拟 Java 代码层次的方法调用，而 MethodHandle 是在模拟字节码层次的方法调用。
Reflection 是重量级，而 MethodHandle 是轻量级。
Reflection API 的设计目标是只为 Java 语言服务的，而 MethodHandle 则设计成可服务于所有 Java 虚拟机之上的语言，其中也包括 Java 语言。
* invokedynamic 指令
在某种程度上， invokedynamic 指令与 MethodHandle 机制的作用是一样的，都是为了解决原有 4 条" invoke" 指令方法分派规则固化在虚拟机之中的问题，把如何查找目标方法的决定权从虚拟机转嫁到具体用户代码之中。
* 掌控方法分派规则


----------
后续未看：转载
4、基于栈的字节码解析执行引擎
4.1、解析执行
Java 语言中， Javac 编译器完成了程序代码经过词法分析、语法分析到抽象语法树，再遍历语法树生成线性的字节码指令流的过程。因为这一部分动作是在 Java 虚拟机之外进行的，而解释器在虚拟机的内部，所以 Java 程序的编译就是半独立的实现。

4.2、基于栈的指令集与基于寄存器的指令集
Java 编译器输出的指令流，基本上[ 1] 是一种基于栈的指令集架构。

基于栈的指令集主要的优点就是可移植，寄存器由硬件直接提供[ 2]， 程序直接依赖这些硬件寄存器则不可避免地要受到硬件的约束。

栈架构指令集的主要缺点是执行速度相对来说会稍慢一些。

虽然栈架构指令集的代码非常紧凑，但是完成相同功能所需的指令数量一般会比寄存器架构多，因为出栈、入栈操作本身就产生了相当多的指令数量。更重要的是，栈实现在内存之中，频繁的栈访问也就意味着频繁的内存访问，相对于处理器来说，内存始终是执行速度的瓶颈。尽管虚拟机可以采取栈顶缓存的手段，把最常用的操作映射到寄存器中避免直接内存访问，但这也只能是优化措施而不是解决本质问题的方法。由于指令数量和内存访问的原因，所以导致了栈架构指令集的执行速度会相对较慢。

4.3、基于栈的解析执行过程
一段简单的算术代码的字节码表示 link
在 HotSpot 虚拟机中，有很多以" fast_" 开头的非标准字节码指令用于合并、替换输入的字节码以提升解释执行性能，而即时编译器的优化手段更加花样繁多[ 1]。

