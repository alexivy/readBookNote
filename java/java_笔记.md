- [java](#java)
  - [HashMap](#hashmap)
  - [Collections.synchronizedMap()](#collectionssynchronizedmap)
  - [ConcurrentHashMap](#concurrenthashmap)
  - [ArrayList](#arraylist)
  - [LinkedList](#linkedlist)
  - [String StringBuffer StringBuilder](#string-stringbuffer-stringbuilder)
  - [多线程](#多线程)
  - [jvm->GC](#jvm-gc)
  - [语法](#语法)
    - [泛型](#泛型)
      - [调用者决定方法返回值的类型](#调用者决定方法返回值的类型)
  - [反射](#反射)
    - [反射的缺点](#反射的缺点)
  - [命令](#命令)
- [设计模式](#设计模式)
- [代码执行顺序](#代码执行顺序)
- [异常](#异常)
- [不可变对象](#不可变对象)
- [ThreadLocal](#threadlocal)
- [实现单例的几种方法](#实现单例的几种方法)
- [接口 interface](#接口-interface)
- [线程 进程](#线程-进程)
- [设计模式](#设计模式-1)


# java  

无特殊说明均针对java-8  

## HashMap  
DEFAULT_LOAD_FACTOR=0.75（float型）  
数据结构是：node数组链表红黑树，链表尾插法（保持它的前后顺序，避免多线程下可能出现的环）  
插入新的kv对时会判断是否大于TREEIFY_THRESHOLD，大于就将其树化。还会判断元素数是否大于阈值，大于就会resize扩容，初始容量是16，每次扩容到原来的2倍。  
不是线程安全的

## Collections.synchronizedMap()  
线程安全，内部维护了一个mutex锁，对map操作前会用synchronized关键字锁对象来达到线程安全。  

## ConcurrentHashMap
利用synchronized和cas实现。put元素时会先cas尝试，失败后会Synchronized操作，性能比Collections.synchronizedMap好一些。

## ArrayList  
数据结构是object数组，每次扩容到原来的1.5倍（向下取整），数组所以在特定位置的操作会伴随元素的复制，适合写读，不适合频繁删除中间元素的情况  

## LinkedList  
底层是链表，不适合频繁读，但删除中间元素时性能比ArrayList好。

## String StringBuffer StringBuilder  
String，final char value[]存储字符串，声明的是不可变对象，每次对String对象的操作都会生成新的String对象。效率低下，浪费内存空间。  
StringBuffer 和 StringBuilder都继承自抽象类AbstractStringBuilder，它们的对象能够被多次修改，并且不产生新的未使用对象。AbstractStringBuilder实现了CharSequence和Appendable接口。使用char[] value存储数据。  
StringBuffer线程安全，synchronized修饰方法。  
StringBuiler性能比Buffer稍高，但线程不安全。  
  
  
  
## 多线程  
java自带的线程池底层都是ThreadPoolExecutor实现。它有7个参数：```int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler```  

## jvm->GC  
年轻代，老年代，元空间。  
年轻代：eden survivor1、2区（8：1：1），新对象会先放入eden，eden满时会minor gc，垃圾回收器都是复制清除算法。  
老年代：大对象会直接进入老年代，CMS垃圾回收器的流程：（标记清除算法，内存碎片的问题）  



## 语法  

### 泛型  

#### 调用者决定方法返回值的类型
getMax方法返回值的类型由调用者决定，那么就这么去定义 getMax 方法：  

    public <A> A getMax() {
        //...
        return (A)result;
    }

MyBatis中的例子：  
    被调函数：  
    public \<T\> T getMapper(Class\<T\> type) {
        return configuration.\<T\>getMapper(type, this);
    }
    调用者部分：
    UserMapper userMapper = session.getMapper(UserMapper.class);


## 反射  

反射的作用：  

1. 在运行时分析类的能力。  
2. 在运行时查看对象，例如，编写一个toString方法供所有类使用  
3. 实现通用的数组操作代码。  
4. 利用Method对象，这个对象很像C++中的函数指针。  

### 反射的缺点  

反射包括了一些动态类型，所以JVM无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被 执行的代码或对性能要求很高的程序中使用反射。  

使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如Applet，那么这就是个问题了。。  

由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用－－代码有功能上的错误，降低可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。  

## 命令  

***jps***   
虚拟机进程状况工具  
<details>
        <summary>点击查看折叠的内容</summary>
        <!-- 内容与标签间空一行 -->
    
    jps -l
    示例：
    D:\IDEA_Projects\javaTest>jps -l
    7848 org.jetbrains.jps.cmdline.Launcher
    11836 LC200619
    6860 jdk.jcmd/sun.tools.jps.Jps
    9484
    
</details>

***jinfo***   
java配置状况工具  

***jmap***   
内存映像工具  

<details>
        <summary>点击查看折叠的内容</summary>
        <!-- 内容与标签间空一行 -->
    
    jmap -histo[:live] <pid>  或者：jmap --help查看更多具体情况
    示例：
    D:\IDEA_Projects\javaTest>jmap -histo 11836

    num     #instances         #bytes  class name
    ----------------------------------------------
    1:          1374        1546560  [B
    2:           767        1530584  [I
    3:          9384        1024616  [C
    4:          7410         177840  java.lang.String
    5:           781          89064  java.lang.Class
    6:          1631          76192  [Ljava.lang.Object;
    ----------------------------------------------
    以上class name中：
    [C 等价于 char[]
    [S 等价于 short[]
    [I 等价于 int[]
    [B 等价于 byte[]
    [[I 等价于 int[][]
    
</details>


***jstat***   
统计信息监视工具  

***jstack***   
堆栈异常跟踪工具  

jvisualvm
jconsole


# 设计模式  
建造者模式：将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。  
适配器模式：将一个类的接口转换成另外一个客户希望的接口。Adapter 模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。  
桥接模式：将抽象部分与它的实现部分分离，使它们都可以独立地变化。  
代理模式：为其他对象提供一种代理以控制对这个对象的访问。  


# 代码执行顺序  
finally在return 关键字前执行。  
static代码块在类主动使用的首次初始化时执行。  
对类的主动使用：  
1. 创建类的实例  
2. 访问某个类或者接口的未被final修饰的静态常量，或对静态变量赋值。  
3. 调用类的静态方法。  
4. 反射（Class.forName（"a"））  
5. 初始化类的子类  
6. java虚拟机启动时被标明为启动类的类  

# 异常  
在Java程序设计语言中，异常对象都是派生于Throwable类的一个实例。  
Error类层次结构描述了Java运行时系统的内部错误和资源耗尽错误。应用程序不应该抛出这种类型的对象。如果出现了这样的内部错误，除了通告给用户，并尽力使程序安全地终止之外，再也无能为力了。这种情况很少出现。  
对于Exception，由程序错误导致的异常属于RuntimeException；而程序本身没有问题，但由于像I/O错误这类问题导致的异常属于其他异常。  

Java语言规范将派生于Error类或RuntimeException类的所有异常称为非受查（unchecked）异常，所有其他的异常称为受查（checked）异常。编译器将核查是否为所有的受查异常提供了异常处理器。  


RuntimeException的异常包含下面几种情况：·错误的类型转换。·数组访问越界。·访问null指针。  

不是派生于RuntimeException的异常包括：·试图在文件尾部后面读取数据。·试图打开一个不存在的文件。·试图根据给定的字符串查找Class对象，而这个字符串表示的类并不存在。

# 不可变对象  
参考：[此处](https://www.cnblogs.com/dolphin0520/p/10693891.html)  
总结一下，不可变对象或者不允许对其属性进行操作，或者对其属性的修改会返回一个新对象。通过反射的方法可以改变不可变对象。（final属性也可以通过反射改变）  
String就是最常用的不可变对象。  
不可变对象的使用场景：1、并发编程；2、防止对象被意外修改；3、避免在集合类的使用过程中出现错误。  


# ThreadLocal  
参考：[此处](https://www.cnblogs.com/xzwblog/p/7227509.html)  
Thread对象有属性threadLocalMap（是ThreadLocal的内部类），threadLocalMap中entry  
ThreadLocal对象set和get时均是从ThreadLocalMap中取值，Key为ThreadLocal的实例tl，存取时会获得实例tl的hashCode，由hashCode和map的len（2的倍数）来确定（方法是按位与，这种情况下与取模相同）在map中entry数组的下标。  
map中entry extends 弱引用，tl的弱引用，若tl没有外部的强引用 gc时tl会变为null，entry.get()方法会返回null，map的set或getEntry时会删除get方法返回null的节点，但若没有执行这两个方法且线程迟迟不结束，entry.value就会占用内存且无法被使用，所以建议手动调用tl.remove方法。  


# 实现单例的几种方法  
1、synchronized 构造方法  
2、将单例作为类的static属性，在类加载时单例会被赋值  
3、double check ：在构造方法中对类加锁，枷锁前后两次检测是否为null  
4、将单例作为类静态内部类InnerClass中的static属性，在get方法中返回InnerClass.instance，与2不同的是类加载时不会创建单例，调用get方法时才会创建。  
5、枚举类。  


# 接口 interface  
接口中的方法都自动地被设置为public一样，接口中的域将被自动设为publicstatic final。  
可以为接口方法提供一个默认实现。必须用default修饰符标记这样一个方法。  
Java SE 8中，允许在接口中增加静态方法。  

超类优先，接口有默认实现时需要明确。  


# 线程 进程  
线程是调度的最小单位，进程是资源分配的最小单位。  
每个进程拥有自己的一整套变量，而线程则共享数据，线程间通讯效率更高，创建、撤销一个线程比启动新进程的开销要小得多。  

# 设计模式  
设计模式可以分为三大类：创建型模式（Creational Patterns）、结构型模式（Structural Patterns）、行为型模式（Behavioral Patterns）。  
创建型模式：
工厂模式（Factory Pattern）
抽象工厂模式（Abstract Factory Pattern）
单例模式（Singleton Pattern）
建造者模式（Builder Pattern）
原型模式（Prototype Pattern）
[参考](https://www.runoob.com/design-pattern/design-pattern-intro.html)