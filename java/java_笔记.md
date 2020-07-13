- [java](#java)
  - [HashMap](#hashmap)
  - [Collections.synchronizedMap()](#collectionssynchronizedmap)
  - [ConcurrentHashMap](#concurrenthashmap)
  - [ArrayList](#arraylist)
  - [LinkedList](#linkedlist)
  - [多线程](#多线程)
  - [jvm->GC](#jvm-gc)
  - [语法](#语法)
    - [泛型](#泛型)
      - [调用者决定方法返回值的类型](#调用者决定方法返回值的类型)
  - [反射](#反射)
    - [反射的缺点](#反射的缺点)
  - [命令](#命令)


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