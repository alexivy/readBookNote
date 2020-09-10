- [java](#java)
  - [六种数字类型](#六种数字类型)
  - [java中的多态](#java中的多态)
  - [hashCode equals](#hashcode-equals)
  - [==](#)
  - [volatile详解](#volatile详解)
  - [synchronized](#synchronized)
  - [锁](#锁)
    - [公平锁 非公平锁](#公平锁-非公平锁)
    - [共享锁与排他锁](#共享锁与排他锁)
  - [集合](#集合)
  - [TreeSet](#treeset)
  - [TreeMap](#treemap)
  - [HashMap](#hashmap)
    - [快速失败（fail - fast）](#快速失败fail---fast)
  - [Collections.synchronizedMap()](#collectionssynchronizedmap)
  - [ConcurrentHashMap](#concurrenthashmap)
    - [安全失败（ fail — safe ）](#安全失败-fail--safe-)
  - [ArrayList](#arraylist)
  - [LinkedList](#linkedlist)
  - [String StringBuffer StringBuilder](#string-stringbuffer-stringbuilder)
  - [多线程](#多线程)
  - [jvm->GC](#jvm-gc)
  - [语法](#语法)
    - [泛型](#泛型)
      - [Java泛型的实现方法：类型擦除](#java泛型的实现方法类型擦除)
      - [调用者决定方法返回值的类型](#调用者决定方法返回值的类型)
  - [反射](#反射)
    - [反射的缺点](#反射的缺点)
  - [命令](#命令)
- [代码执行顺序](#代码执行顺序)
- [异常](#异常)
- [不可变对象](#不可变对象)
- [ThreadLocal](#threadlocal)
- [实现单例的几种方法](#实现单例的几种方法)
- [接口 interface](#接口-interface)
- [线程 进程](#线程-进程)
- [设计模式](#设计模式)
- [红黑树](#红黑树)
- [面向对象与面向过程](#面向对象与面向过程)


# java  

无特殊说明均针对java-8  

## 六种数字类型  
byte 8位，有符号，-128~127，默认0。byte b=1;  
short 16位，有符号，-32768~32767，默认0。short s=1;  
int 32位，有符号，-(2^31)~2^31-1，默认0。int i=1;  
long 64位，有符号，-(2^63)~2^63-1，默认0L。long a=1L;  
float 32位，单精度浮点数。默认0.0f。float a=1.0f;  
double 64位，双精度，默认0.0d。double a=1.0;  

作为参数时，传入1则会调用声明参数为int型的方法，1f会调用float的方法，1.0会调用double的方法。  

声明类型为x的参数，会按照范围和精度调用方法。  
按前面列出的顺序，列在前变量类型的会寻找声明参数类型顺序在后面的第一个方法。例子如下  

```
byte a = 1;
func(a);

void func(byte a){} //最优先

void func(int a){} //次优先

void func(double a){} //最末

```

## java中的多态  
静态分派（与重载相关）  
方法参数如果有明确的声明类型或被强转类型A（静态类型），则按照其类型A去确定调用的方法。  
调用静态方法时也按照静态类型去调用。  

动态分派（与重写相关）  
调用对象某一方法时的执行invokevirtual指令的运行时解析过程大致分为以下几步：  
1）获得实际类型C。找到操作数栈顶的第一个元素所指向的对象的实际类型，记作C。  
2）在C中找合适（描述符和简单名称都符合且访问权限没问题的）的方法，找到就执行。如果在类型C中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的直接引用，查找过程结束；不通过则返回java.lang.IllegalAccessError异常。  
3）递归在其父类中找方法。按照继承关系从下往上依次对C的各个父类进行第二步的搜索和验证过程。  
4）没找到，抛异常。如果始终没有找到合适的方法，则抛出java.lang.AbstractMethodError异常。  

但子类中的字段名若与父类相同，则访问子类字段，通过super访问父类字段。  

总结：java对象调用方法，调用者看其实际类型，参数看其静态类型。  

java为类型在方法区中建立一个虚方法表，使用虚方法表索引来代替元数据查找以提高性能，虚方法表中存放着各个方法的实际入口地址。如果某个方法在子类中没有被重写，那子类的虚方法表中的地址入口和父类相同方法的地址入口是一致的，都指向父类的实现入口。如果子类中重写了这个方法，子类虚方法表中的地址也会被替换为指向子类实现版本的入口地址。  
具有相同签名的方法，在父类、子类的虚方法表中都应当具有一样的索引序号，这样当类型变换时，仅需要变更查找的虚方法表，就可以从不同的虚方法表中按索引转换出所需的入口地址。虚方法表一般在类加载的连接阶段进行初始化，准备了类的变量初始值后，虚拟机会把该类的虚方法表也一同初始化完毕。  


## hashCode equals  
object中的默认实现：  
equals默认实现是``` return (this == obj);//是比较两个对象的内存地址```  
hashCode是native方法，不同jdk版本不同，多数情况为内存地址转化成的int值。[native源码](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FJetBrains%2Fjdk8u_hotspot%2Fblob%2Fmaster%2Fsrc%2Fshare%2Fvm%2Fruntime%2Fsynchronizer.cpp%23L560)  
为什么重写equals方法就一定要重写hashCode方法呢？  
在java8的源码中有要求equals方法判断两个对象相等，那么hashCode值一定要相同：  
```
If two objects are equal according to the {@code equals(Object)}
method, then calling the {@code hashCode} method on each of
the two objects must produce the same integer result.
```
如果不重写，可能导致如下情况：  a.equals(b)==true ; (a.hasCode() != b.hashConde())== true ;  
这种情况下可能会引发一些错误，如会导致java中的HashMap中可能同时存在a和b两个key。  

## ==   
比较的是两个对象在内存中的地址是否相同。举例如下：  
```
int a=1;
Integer b=Integer.parseInt("1");
Integer c=new Integer("1");
System.out.println("a==b? "+(a==b));
System.out.println("a==c? "+(a==c));
System.out.println("b==c? "+(b==c));
输出如下：
a==b? true
a==c? true
b==c? false
```

## volatile详解  
volatile关键字为实例域的同步访问提供了一种免锁机制。  
volatile的特性：三点，保证可见性、但不保证原子性、禁止指令重排序优化。  
“可见性”是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。通过线程修改后立即将新值写回主内存（assign store操作紧密相连），线程每次使用值前都必须先从主内存刷新最新的值（load use操作紧密相连）。  

不保证原子性，所以不是线程安全的，举例子就是多线程多次对volatile修饰的域自增，最后的结果小于预期值。  
20个线程循环各自对volatile int i操作i++，各1000次，线程安全的话是20000，实则小于20000。  
原因是，A线程在i++前，会先到主内存中取i=233，B在此时也去取了i=233，A作i++，然后将234写回到主内存中，B作i++，然后也将234写回到主内存中，覆盖了A的修改，类似于数据库中的丢失修改（数据库中一般是加s锁避免）。  

有volatile修饰的变量，赋值后（前面mov%eax，0x150(%esi)这句便是赋值操作）多执行了一个“lock addl$0x0，(%esp)”操作，这个操作的作用相当于一个内存屏障，指重排序时不能把后面的指令重排序到内存屏障之前的位置。禁止指令重排序的意义：指令重排序会干扰程序的并发执行。java中有一些规定两项操作之间的偏序关系的原则称为先行发生原则，其一程序次序规则是指，在一个线程内，按照控制流顺序，书写在前面的操作先行发生于书写在后面的操作。注意，这里说的是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。  

## synchronized  
Java中的每一个对象都有一个内部锁，synchronized关键字就是去获取内部的对象锁。  
synchronized修饰静态方法时，锁加在类上；修饰非静态方法时，锁加在实例对象上。  

特点有：
1.可重入性（方法a中调用方法b，b可以获得a获得的锁）；  
2.不可中断性（锁被其他线程持有时只能等待，lock类有中断的能力 ***MORE*** ）。  

Java虚拟机的指令集中有monitorenter和monitorexit两条指令来支持synchronized关键字的语义。  
moniterenter指令以栈顶元素作为锁开启同步。  
minitorexit指令推出同步。  
为了保证在方法异常完成时monitorenter和monitorexit指令依然可以正确配对执行，编译器会自动产生一个异常处理程序，这个异常处理程序声明可处理所有的异常，它的目的就是用来执行monitorexit指令。  

## 锁  
[此文很详细](https://www.cnblogs.com/jyroy/p/11365935.html)。  

对象头中会存储锁标志位。  


锁升级。  
四种锁状态：无锁，偏向锁，轻量级锁，重量级锁。  


无锁阶段：所有线程都能访问并修改同一资源，但同时只有一个线程能修改成功，修改操作在循环内进行直到成功。CAS即是无锁。  


偏向锁：一段代码一直被同一线程访问，那么该线程会获得偏向锁，降低获取锁的代价。（在只有一个线程执行同步代码块时能够提高性能）  
当一个线程访问同步代码时会获得锁，会在对象头中存储锁偏向的线程id，线程进入和退出时不再通过CAS操作来加解锁，而是通过对象头来检测当前线程是否持有偏向锁。 ？ 只有其他线程来竞争偏向锁时，持有偏向锁的线程才会释放锁，在全局安全点时释放偏向锁，先暂停持有偏向锁的线程，根据锁对象是否处于被锁定状态，将锁状态变为无锁或轻量级锁。 ？ 

轻量级锁：当锁是偏向锁的时候，被其他线程访问就会升级为轻量级的锁，其他线程会通过自旋的方式尝试获取锁，不会阻塞。  
获得过程：对象处于无锁状态，先在栈帧中建立锁记录用于存储对象mark word的拷贝。 然后cas尝试将对象mark word更新为指向锁记录的指针，并将锁记录里的owner指针指向对象的mark word。更新成功后就拥有了该对象的轻量级锁，并将对象的锁标志为设为“00”（表示轻量级锁）。  
多线程竞争轻量级锁时，若只有一个等待线程，则自旋等待（避免线程调度的开销，线程阻塞和唤醒），自旋超过一定次数或者有第三个线程来访，则升级为重量级的锁。  

重量级锁：升级为重量级锁后，等待锁的线程都会进入阻塞状态。  


### 公平锁 非公平锁  
公平锁，线程按申请锁的顺序（等待队列中的位置）来获得锁，等待队列中除第一个线程以外的所有线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。  

非公平锁，线程加锁时直接抢占式获取锁，如果锁刚好可用线程就可以无需阻塞直接获取到锁，没获取到才会到等待队列尾等待。吞吐率较高：因为线程有几率不阻塞直接获取锁，不必唤醒所有线程。  

### 共享锁与排他锁  
类似数据库中的s锁与x锁，java中有ReentrantReadWriteLock。  


## 集合  

队列：循环数组队列_ArrayDeque，链表队列_LinkedList。  

HashSet迭代过程中是无序的。  

java中的集合。  
![image](/image/202008261354java.PNG)  

优先级队列，数据结构是堆。  


## TreeSet  

TreeSet是一个有序集合。在对有序集合进行遍历时，每个值将自动地按照排序后的顺序呈现。排序是用树结构（红黑树）完成的。如果树中包含n个元素，查找新元素的正确位置平均需要log2n次比较。  

## TreeMap  



## HashMap  

DEFAULT_LOAD_FACTOR=0.75（float型）  
数据结构是：node数组链表红黑树，链表尾插法（保持它的前后顺序，避免多线程下可能出现的环）  
插入新的kv对时会判断是否大于TREEIFY_THRESHOLD，大于就将其树化。还会判断元素数是否大于阈值（与负载因子相关），大于就会resize扩容，初始容量是16，每次扩容到原来的2倍。  
不是线程安全的。why？：java 8中多线程put会丢失元素，detail：当两个线程同时向一个空位插入值时，它们可能均检测到table[i]==null 然后分别执行 table[ i]=Value_A，table[ i]=Value_B，此时其中一个线程写入的元素就会丢失。java 7中还有可能出现多线程resize()时产生环的情况，由于采用的是头插法，resize过程中元素的前后顺序会改变，detail：resize()时会调用transfer()函数，transfer()函数中会将table数组中的元素迁移到newTable，若线程A进行transfer时表的i位置为[ {3}->{7}->null ]，执行e={3}后，线程B执行，完成了transfer，将i位置改为了[ {7}->{3}->null ]，然后A继续执行，next=e.next(=null)，e.next=newTable[ i]（{3}.next={7}），然后完成了transfer，此时i位置就为[ {7}->{3}->{7}->... ]产生了环。  

### 快速失败（fail - fast）  
此外用迭代器遍历HashMap中的元素时，若集合在遍历期间发生了修改（自己或其他线程进行修改），就会抛出Concurrent Modification Exception，这时java.util包下的集合类都有的快速失败机制。  
每个迭代器都维护一个独立的计数值。在每个迭代器方法的开始处检查自己改写操作的计数值是否与集合的改写操作计数值一致。  
原理就是这个过程中使用一个 modCount 变量，集合在被遍历期间如果内容发生变化，就会改变modCount的值，每当迭代器使用hashNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedmodCount值，是的话就继续遍历；否则抛出异常，终止遍历。  
但由于多次操作集合后modCount的值可能变回去，因此即使其他线程修改了集合，也可能不会抛出前述的并发修改异常。  

## Collections.synchronizedMap()  
线程安全，内部维护了一个mutex锁，对map操作前会用synchronized关键字锁对象来达到线程安全。  

## ConcurrentHashMap
利用synchronized和cas实现它的并发安全性。put元素时会先cas尝试，失败后会Synchronized操作，性能比Collections.synchronizedMap好一些。  
Node的值和next采用了volatile去修饰，保证了可见性，并且也引入了红黑树，在链表大于一定值的时候会转换（默认是8）。  
ConcurrentHashMap的put操作大致可以分为以下步骤：  

根据 key 计算出 hashcode 。

判断是否需要进行初始化。

即为当前 key 定位出的 Node，如果为空表示当前位置可以写入数据，利用 CAS 尝试写入，失败则自旋保证成功。

如果当前位置的 hashcode == MOVED == -1,则需要进行扩容。

如果都不满足，则利用 synchronized 锁写入数据。

如果数量大于 TREEIFY_THRESHOLD 则要转换为红黑树。

ConcurrentHashMap的get操作大致步骤：  

根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值。

如果是红黑树那就按照树的方式获取值。

就不满足那就按照链表的方式遍历获取值。


CAS写入，是调用Unsafe类里的native方法实现的，四个参数，一个是node数组，一个是下标i，一个是期望值，一个是新值。  
CAS 是乐观锁的一种实现方式，是一种轻量级锁，CAS 操作的流程如下：线程在读取数据时不进行加锁，在准备写回数据时，比较原值是否修改，若未被其他线程修改则写回，若已被修改，则重新执行读取流程。  
这是一种乐观策略，认为并发操作并不总会发生。  

CAS存在的问题，ABA！（其他线程对其进行了修改但未被检测到）。解决方法：1、带一个版本号，比较值时还比较版本号，每次修改值时将版本号加1。2、时间戳！类似于版本号。  


### 安全失败（ fail — safe ）  

采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。

原理：由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发Concurrent Modification Exception。

缺点：基于拷贝内容的优点是避免了Concurrent Modification Exception，但同样地，迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的。

场景：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。

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

分别是：核心数量，最大数量，存活时间（超过核心线程数后多余的线程最多存活的时间），时间单位，阻塞队列（存储任务），线程工厂，抛弃策略（抛异常，丢新来的，丢等久了的，callerRun调用者执行）。  

ThreadPoolExecutor的实例执行任务的方法有：   
1、execute(Runnable ins) ，void方法。  
2、submit（ ） 可传入实现Callable或者Runnable的实例，此方法返回Future<?>，可得的线程返回值。   

Future的get方法在提交的任务完成前会阻塞调用的线程。  

## jvm->GC  
年轻代，老年代，元空间。  
年轻代：eden survivor1、2区（8：1：1），新对象会先放入eden，eden满时会minor gc，垃圾回收器都是复制清除算法。  
老年代：大对象会直接进入老年代，CMS垃圾回收器的流程：（标记清除算法，内存碎片的问题）  



## 语法  

### 泛型  

#### Java泛型的实现方法：类型擦除  

[参考链接](https://www.cnblogs.com/wuqinglong/p/9456193.html)  

Java的泛型是伪泛型,Java在编译期间，所有的泛型信息都会被擦掉。Java的泛型基本上都是在编译器这个层次上实现的，在生成的字节码中是不包含泛型中的类型信息的，使用泛型的时候加上类型参数，在编译器编译的时候会去掉，这个过程称为类型擦除。  

类型擦除后保留原始类型，原始类型就是擦除去了泛型信息，最后在字节码中的类型变量的真正类型，无论何时定义一个泛型，相应的原始类型都会被自动提供，类型变量擦除，并使用其限定类型（无限定的变量用Object）替换。  

Java编译器先检查代码中泛型的类型，然后在进行类型擦除，最后进行编译。   

泛型信息会被擦除掉，但会使用泛型的方法返回值以及泛型域会自动强制转换类型。  

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


1、synchronized get方法  
<details>
        <summary>点击查看</summary>
```
public class SingleTon {
    private static  SingleTon singleTon;
    private SingleTon(){
    }
    public static synchronized SingleTon getInstance(){
        if(singleTon==null){
            singleTon=new SingleTon();
        }
        return singleTon;
    }
}
```
</details>

2、将单例作为类的static属性，在类加载时单例会被赋值  

<details>
        <summary>点击查看</summary>
```
public class SingleTon {
    private static SingleTon singleTon=new SingleTon();
    private SingleTon(){
    }
    public static SingleTon getInstance(){
        return singleTon;
    }
}
```
</details>


3、double check 双所检测：在构造方法中对类加锁，加锁前后两次检测是否为null  

<details>
        <summary>点击查看</summary>
```
public class SingleTon {
    private static volatile SingleTon singleTon;
    private SingleTon(){
    }
    public static  SingleTon getInstance(){
        if(singleTon==null){
            synchronized (SingleTon.class){
                if(singleTon==null){
                    singleTon=new SingleTon();
                }
            }
        }
        return singleTon;
    }
}
注：此例子中volatile的主要是禁止重排序，初始化一个实例（SomeType st = new SomeType()）在java字节码中会有4个步骤，申请内存空间，初始化默认值（区别于构造器方法的初始化），执行构造器方法连接引用和实例。这4个步骤后两个有可能会重排序，1234  1243都有可能，造成未初始化完全的对象发布，其他线程使用时出现空指针异常等错误。volatile可以禁止指令重排序，从而避免这个问题。
参考：https://www.zhihu.com/question/56606703/answer/149894860
```

</details>

4、将单例作为类静态内部类InnerClass中的static属性，在get方法中返回InnerClass.instance，与2不同的是类加载时不会创建单例，调用get方法时才会创建。  

<details>
        <summary>点击查看</summary>
```
public class SingleTon {
    private static  SingleTon singleTon;
    private SingleTon(){
    }
    public static SingleTon getInstance(){
        return InnerClass.ins;
    }
    static class InnerClass{
        public static final SingleTon ins=new SingleTon();
    }
}
```
</details>

5、枚举类。  

<details>
        <summary>点击查看</summary>
```
public enum  SingleTon {
    INSTANCE;
    public Object manyMethod(){
        return new Object();
    }
}
```
</details>


1、如果单例由不同的类装载器装入，那便有可能存在多个单例类的实例。假定不是远端存取，例如一些servlet容器对每个servlet使用完全不同的类装载器，这样的话如果有两个servlet访问一个单例类，它们就都会有各自的实例。  
解决方法是指定类加载器来获得Class对象。  

2、如果Singleton实现了java.io.Serializable接口，那么这个类的实例就可能被序列化和复原。不管怎样，如果你序列化一个单例类的对象，接下来复原多个那个对象，那你就会有多个单例类的实例。  
解决方法是在类中有方法去处理反序列化的过程，并返回原来的单例实例。  

更详细的参考以下：

[参考链接](http://www.blogjava.net/kenzhh/archive/2013/03/15/357824.html)  


# 接口 interface  
接口中的方法都自动地被设置为public一样，接口中的域将被自动设为public static final。  

java中可通过接口的默认实现来继承多个实现，为接口方法提供一个默认实现，必须用default修饰符标记这样一个方法。  

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

建造者模式：将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。  
适配器模式：将一个类的接口转换成另外一个客户希望的接口。Adapter 模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。  
桥接模式：将抽象部分与它的实现部分分离，使它们都可以独立地变化。  
代理模式：为其他对象提供一种代理以控制对这个对象的访问。  


[参考](https://www.runoob.com/design-pattern/design-pattern-intro.html)


# 红黑树  
五条性质：  
1、节点黑或红；2、根节点黑色；3、叶节点黑色；4、红节点的子节点必须黑色（红色节点的父节点必为黑色）；5、从根到叶黑色节点数量相同。  

插入查找删除的时间复杂度时O(logn)。  

插入新节点的5种情况：  
1、新节点为根节点。新节点为黑色。  
2、新节点的父节点为黑色节点。新节点为红色。  
3、新节点的父节点与叔父节点均为红色（此时必有祖父节点且必为黑色）。  
  调整：新节点为红色，父与叔父节点均改为黑色，祖父节点改为红色，递归判断祖父节点。  
4、新节点是右子节点，其父节点为左子节点且为红色，叔父为黑色。  
  调整：新节点为红色，左旋父节点，进入情况5。  
5、新节点是左子节点，其父节点为左子节点且为红色，叔父为黑色。  
  调整：右旋祖父节点，调整颜色，原祖父变红，原父变黑，新节点为红。  

注：4，5情况下左右可对调。  


删除节点的情况：  
删除一个节点的情况可以转化为删除另一个只有一个子节点的节点。  
转化方法： 删除一个有两个非叶子节点的节点 X 时，找到其左子树种的最大子节点或右子树中的最小子节点 Y，将其值复制给 X ，然后删除 Y （由前述条件 Y 最多有一个子节点）。  

1、删的节点红色的，直接删。  
2、删的节点黑色的，其子节点红色的，子节点转黑。  
3、删的节点 X 是黑色的，其子节点 N 也黑色的。N 替代 X ，称 N 的兄弟节点 S，继续细分：  
A、N为根节点，结束。  
B、N为左子，S为红色，调整：左旋父节点，原父变红，原兄弟变黑。继续DEF的调整。  
C、N父为黑色，兄弟及其子也为黑，调整：将兄弟变红，父作为N从A步骤开始调整。  
D、N父为红，兄弟为黑，调整：父变黑，兄弟变红。结束。  
E、N为左子，兄弟S为黑，S左子为红，右子为黑，调整：右旋S，原S和原SL变色，进入F。  
F、N为左子，S为黑，S右子为红，调整：N父P左旋，SR变黑，S与P交换颜色。  

注：BEF情况下若B为右子，则步骤中左右对调。  

[参考链接](http://www.360doc.com/content/15/1214/14/14513665_520332407.shtml)


# 面向对象与面向过程  
面向对象的程序是由对象组成，程序中有的对象来自标准库，有的是自行实现的。  

面向过程的程序算法是第一位的，编程时首先要确定如何操作数据，然后再决定如何组织数据，以便于数据操作。  

针对规模较小的问题面向过程的开发更方便，面向对象更加适用于解决规模较大的问题。  


对象的三个基本特征：  
（1）封装，即内部的改动不会对外部产生影响。例如，访问数据的对象可以使用ADO或DAO对象模型，也可以直接使用ODBC API函数，但都不会影响其外部特性。  
（2）继承，通过派生来解决实现的重用问题。例如，从SalesOrder类派生出WebSalesOrder类，在WebSalesOrder中，可以重载父类的Confirm方法（发邮件而不是传真），也可以自动继承实现父类的Total方法，实现相同的行为。  
（3）多态（可替代性），不论何时创建了派生类对象，在使用基类对象的地方都可以使用此派生类对象。不同类型的对象就可以处理交互时使用的一组通用消息，并且以它们各自的方式进行。如前面的例子中，WebSalesOrder“is a”SalesOrder，也就是说，在任何使用SalesOrder的地方，都可以使用WebSalesOrder。