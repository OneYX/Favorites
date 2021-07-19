[![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160721.png)](https://img2020.cnblogs.com/blog/686418/202006/686418-20200630151944391-1320178406.png)

思维导图svg: https://note.youdao.com/ynoteshare1/index.html?id=eb05fdceddd07759b8b82c5b9094021a&type=note

在多线程使用共享资源的时候， 我们可以使用synchronized来锁定共享资源，使得同一时刻，只有一个线程可以访问和修改它，修改完毕后，其他线程才可以使用。这种方式叫做互斥锁。

当一个共享数据被当前正在访问到线程添加了互斥锁之后，在同一时刻，其他线程只能等待，直到当前线程释放该锁。

synchronized可以添加互斥锁，并且保证被其他线程看到。

## synchronized的三种应用方式

synchronized关键字最主要有以下3种应用方式，下面分别介绍

- 修饰实例方法，作用于当前实例加锁，进入同步代码钱要获得当前实例的锁
- 修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁
- 修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码块前要获得给定对象的锁

### synchronized作用于实例方法

我们设置类变量`static`为共享资源， 然后多个线程去修改。修改的含义是： 先读取，计算，再写入。那么这个过程就不是原子的，多个线程操作就会出现共享资源争抢问题。

我们在实例方法上添加synchronized，那么，同一个实例执行本方法时，抢到锁到可以执行。



```
public class AccountingSync implements Runnable{
    //共享资源(临界资源)
    static int i=0;

    /**
     * synchronized 修饰实例方法
     */
    public synchronized void increase(){
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        AccountingSync instance=new AccountingSync();
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
    /**
     * 输出结果:
     * 2000000
     */
}
```

上述代码中，开启两个线程去操作共享变量，两个线程执行的是同一个实例对象。如果不添加synchronized，其中`i++`不是原子操作，该操作先读取值，然后再写入一个新值。如果两个线程都读取了i=5，然后线程1写入i=6.线程2后写入，但也是写入i=6, 并不是我们期望的i=7.

添加synchronized修饰后，线程安全，线程必须获取到这个实力到锁才能执行读取和写入。

注意，我们synchronized修饰到是类方法，锁的是实例，当多个线程操作不同实例时，会使用不同实例的锁，就无法保证修改static变量的有序性了。



```
public class AccountingSyncBad implements Runnable{
    static int i=0;
    public synchronized void increase(){
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        //new新实例
        Thread t1=new Thread(new AccountingSyncBad());
        //new新实例
        Thread t2=new Thread(new AccountingSyncBad());
        t1.start();
        t2.start();
        //join含义:当前线程A等待thread线程终止之后才能从thread.join()返回
        t1.join();
        t2.join();
        System.out.println(i);
    }
}
```

上述代码，两个线程持有不同的对象instance，也就是使用不同的锁， 也就不会互斥访问共享资源，就会出现线程安全问题。

### synchronized作用于静态方法

synchronized作用于静态方法时，锁就是当前类到class对象锁。由于静态成员变量不专属于任何一个实例对象，是类成员，因此通过class对象锁可以控制静态成员的并发操作。

### synchronized同步代码块

除了使用关键字修饰实例方法和静态方法外，还可以使用同步代码块。



```
public class AccountingSync implements Runnable{
    static AccountingSync instance=new AccountingSync();
    static int i=0;
    @Override
    public void run() {
        //省略其他耗时操作....
        //使用同步代码块对变量i进行同步操作,锁对象为instance
        synchronized(instance){
            for(int j=0;j<1000000;j++){
                    i++;
              }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

上述代码，将synchronized作用于一个给定的实例对象instance, 即当前实例对象就是锁对象，每次当线程进入synchronized包裹到代码块时，就会要求当前线程持有instance实例对象锁，如果当前有其他线程正持有该对象锁，那么新到到线程就必须等待，这样也就保证了每次只有一个线程执行`i++`操作。当然， 还可以使用this或者class



```
//this,当前实例对象锁
synchronized(this){
    for(int j=0;j<1000000;j++){
        i++;
    }
}

//class对象锁
synchronized(AccountingSync.class){
    for(int j=0;j<1000000;j++){
        i++;
    }
}
```

了解完synchronized到基本含义和使用方式后，我们进一步深入理解synchronized的底层实现原理。

## synchronized底层语义原理

Java虚拟机中的同步(Synchronization)基于进入和退出管程(Monitor)对象实现，无论是显示同步(有明确的monitorenter和monitorexit指令，即同步代码块)还是隐式同步都是如此。在Java语言中，同步用的最多到地方可能是被synchronized修饰的同步方法。同步方法并不是由monitorenter和monitorexit指令来实现同步到，而是由方法调用指令读取运行时常量池中方法到ACC_SYNCHRONIZED标志来隐式实现的，关于这点，稍后分析。下面先来了解一个概念：Java对象头，这对深入理解synchronized实现原理非常关键。

### 理解Java对象头与Monitor

在JVM中，对象在内存中到布局分为三块区域：对象头，实例数据和对齐填充。 如下：

[![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160742)](https://img-blog.csdn.net/20170603163237166?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- 实例变量： 存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。
- 填充数据：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。

而对于顶部，则是Java头对象，它是实现synchronized的锁对象的基础，这点我们重点分析它。 一般而言，synchronized使用的锁对象是存储在Java对象头里的，jvm采用2个字来存储对象头(如果对象是数组则会分配3个字，多出来到1个字记录的是数组长度)，其主要结构是由Mark Word和Class Metadata Address组成，其结构说明如下：

| 虚拟机位数 | 头对象结构             | 说明                                                         |
| :--------- | :--------------------- | :----------------------------------------------------------- |
| 32/64bit   | Mark Word              | 存储对象的hashcode, 锁信息或分代年龄或GC标志等信息           |
| 32/64bit   | Class Metadata Address | 类型指针指向对象的类元数据， JVM通过这个指针确定该对象是哪个类的实例 |

其中Mark Word在默认情况下存储着对象的HashCode, 分代年龄，锁标记等， 以下是32位JVM的Mark Word默认存储结构。

| 锁状态   | 25bit        | 4bit         | 1bit是否是偏向锁 | 2bit锁标志位 |
| :------- | :----------- | :----------- | :--------------- | :----------- |
| 无锁状态 | 对象HashCode | 对象分代年龄 | 0                | 01           |

由于对象头的信息是与对象自身定义的数据没有关系到额外存储成本，因此考虑到JVM的空间效率，Mark Word被设计成为一个非固定的数据结构，以便存储更多有效的数据，它会根据对象本身的状态复用自己的存储空间，如32位JVM下，除了上述列出的Mark Word默认存储结构外，还有如下可能变化的结构：

[![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160804)](https://img-blog.csdn.net/20170603172215966?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

其中，轻量级锁和偏向锁是Java 6对synchronized锁进行优化后新增加的，我们稍后简要分析。这里我们主要分析一下重量级锁也就是通常说的synchronized的对象锁，锁标识位10，其中指针指向的时monitor对象(也称为管程或监视器锁)的起始地址。每个对象都存在着一个monitor与之关联，对象与其monitor之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个monitor被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由ObjetMonitor实现的，其主要数据结构如下(位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的)



```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

ObjectMonitor中有两个队列， `_WaitSet`和`_EntryList`， 用来保存ObjectWaiter对象列表(每个等待锁的线程都会被封装成ObjectWaiter对象)， `_owner`指向持有ObjectMonitor对象的线程。

当多个线程同时访问一段同步代码时，首先会进入`_EntryList`集合， 当线程获取到对象的monitor后，进入`_owner`区域， 并把monitor中到onwer变量设置为当前线程， 同时monitor中的计数器count+1。

若线程调用wait()方法，将释放当前持有的monitor， owner=null， count-1, 同时该线程进入waitSet集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)，并复位变量的值，以便其他线程进入获取monitor(锁)。 如下图所示：

[![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160811)](https://img-blog.csdn.net/20170604114223462?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由此看来，monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因(关于这点稍后还会进行分析)，ok~，有了上述知识基础后，下面我们将进一步分析synchronized在字节码层面的具体语义实现。

## synchronized代码块底层原理

现在我们重新定义一个synchronized修饰的同步代码块， 在代码块中操作共享变量i。



```
public class SyncCodeBlock {

   public int i;

   public void syncTask(){
       //同步代码库
       synchronized (this){
           i++;
       }
   }
}
```

编译上述代码，并使用javap反编译得到字节码如下(这里我们省略一部分没有必要的信息)：



```
Classfile /Users/zejian/Downloads/Java8_Action/src/main/java/com/zejian/concurrencys/SyncCodeBlock.class
  Last modified 2017-6-2; size 426 bytes
  MD5 checksum c80bc322c87b312de760942820b4fed5
  Compiled from "SyncCodeBlock.java"
public class com.zejian.concurrencys.SyncCodeBlock
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
  //........省略常量池中数据
  //构造函数
  public com.zejian.concurrencys.SyncCodeBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
  //===========主要看看syncTask方法实现================
  public void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter  //注意此处，进入同步方法
         4: aload_0
         5: dup
         6: getfield      #2             // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2            // Field i:I
        14: aload_1
        15: monitorexit   //注意此处，退出同步方法
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit //注意此处，退出同步方法
        22: aload_2
        23: athrow
        24: return
      Exception table:
      //省略其他字节码.......
}
SourceFile: "SyncCodeBlock.java"
```

我们主要关注字节码中的如下代码：



```
3: monitorenter  //进入同步方法
//..........省略其他  
15: monitorexit   //退出同步方法
16: goto          24
//省略其他.......
21: monitorexit //退出同步方法
```

从字节码中可知同步语句块的实现使用的是monitorenter 和 monitorexit 指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor (关于重入性稍后会分析)，重入时计数器的值也会加 1。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出多了一个monitorexit指令，它就是异常结束时被执行的释放monitor 的指令。

## synchronized方法底层原理

方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法。当方法调用时，调用指令将会 检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词）， 然后再执行方法，最后再方法完成(无论是正常完成还是非正常完成)时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛 出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。下面我们看看字节码层面如何实现：



```
public class SyncMethod {

   public int i;

   public synchronized void syncTask(){
           i++;
   }
}
```

使用javap反编译后的字节码如下：



```
Classfile /Users/zejian/Downloads/Java8_Action/src/main/java/com/zejian/concurrencys/SyncMethod.class
  Last modified 2017-6-2; size 308 bytes
  MD5 checksum f34075a8c059ea65e4cc2fa610e0cd94
  Compiled from "SyncMethod.java"
public class com.zejian.concurrencys.SyncMethod
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool;

   //省略没必要的字节码
  //==================syncTask方法======================
  public synchronized void syncTask();
    descriptor: ()V
    //方法标识ACC_PUBLIC代表public修饰，ACC_SYNCHRONIZED指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
}
SourceFile: "SyncMethod.java"
```

从字节码中可以看出，synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。这便是synchronized锁在同步代码块和同步方法上实现的基本原理。同时我们还必须注意到的是在Java早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的Mutex Lock来实现的，而操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的synchronized效率低的原因。庆幸的是在Java 6之后Java官方对从JVM层面对synchronized较大优化，所以现在的synchronized锁效率也优化得很不错了，Java 6之后，为了减少获得锁和释放锁所带来的性能消耗，引入了轻量级锁和偏向锁，接下来我们将简单了解一下Java官方在JVM层面对synchronized锁的优化。

## Java虚拟机对synchronized的优化

锁的状态总共有四种，无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级到重量级锁。

但锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级。

关于重量级锁，前面我们已经详细分析过，下面我们将介绍偏向锁和轻量级锁以及JVM的其他优化手段，这里并不打算深入到每个锁的实现和转换过程，更多地是阐述Java虚拟机提供到每个锁的核心优化思想。

### 偏向锁

偏向锁是Java 6之后加入的新锁，它是一种针对加锁操作的优化手段，经过研究发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁(会涉及到一些CAS操作,耗时)的代价而引入偏向锁。偏向锁的核心思想是，如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word 的结构也变为偏向锁结构，当这个线程再次请求锁时，无需再做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而也就提供程序的性能。所以，对于没有锁竞争的场合，偏向锁有很好的优化效果，毕竟极有可能连续多次是同一个线程申请相同的锁。但是对于锁竞争比较激烈的场合，偏向锁就失效了，因为这样场合极有可能每次申请锁的线程都是不相同的，因此这种场合下不应该使用偏向锁，否则会得不偿失，需要注意的是，偏向锁失败后，并不会立即膨胀为重量级锁，而是先升级为轻量级锁。下面我们接着了解轻量级锁。

### 轻量级锁

倘若偏向锁失败，虚拟机并不会立即升级为重量级锁，它还会尝试使用一种称为轻量级锁的优化手段(1.6之后加入的)，此时Mark Word 的结构也变为轻量级锁的结构。轻量级锁能够提升程序性能的依据是“对绝大部分的锁，在整个同步周期内都不存在竞争”，注意这是经验数据。需要了解的是，轻量级锁所适应的场景是线程交替执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量级锁膨胀为重量级锁。

### 自旋锁

轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。这是基于在大多数情况下，线程持有锁的时间都不会太长，如果直接挂起操作系统层面的线程可能会得不偿失，毕竟操作系统实现线程之间的切换时需要从用户态转换到核心态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，因此自旋锁会假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)，一般不会太久，可能是50个循环或100循环，在经过若干次循环后，如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这就是自旋锁的优化方式，这种方式确实也是可以提升效率的。最后没办法也就只能升级为重量级锁了。

### 锁消除

消除锁是虚拟机另外一种锁的优化，这种优化更彻底，Java虚拟机在JIT编译时(可以简单理解为当某段代码即将第一次被执行时进行编译，又称即时编译)，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的请求锁时间，如下StringBuffer的append是一个同步方法，但是在add方法中的StringBuffer属于一个局部变量，并且不会被其他线程所使用，因此StringBuffer不可能存在共享资源竞争的情景，JVM会自动将其锁消除。



```
/**
 * Created by zejian on 2017/6/4.
 * Blog : http://blog.csdn.net/javazejian [原文地址,请尊重原创]
 * 消除StringBuffer同步锁
 */
public class StringBufferRemoveSync {

    public void add(String str1, String str2) {
        //StringBuffer是线程安全,由于sb只会在append方法中使用,不可能被其他线程引用
        //因此sb属于不可能共享的资源,JVM会自动消除内部的锁
        StringBuffer sb = new StringBuffer();
        sb.append(str1).append(str2);
    }

    public static void main(String[] args) {
        StringBufferRemoveSync rmsync = new StringBufferRemoveSync();
        for (int i = 0; i < 10000000; i++) {
            rmsync.add("abc", "123");
        }
    }

}
```

### 无锁->偏向锁

[![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160818)](https://mmbiz.qpic.cn/mmbiz_png/SoGf97KLurAicd1Y2Vmx8AMVibe6YvE8giahbLO6jDZkMbiaOpkCU63biaAyZ89MP91WNe4JzDxyI3HhFXm5vUaNY7g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 首先A 线程访问同步代码块，使用CAS 操作将 Thread ID 放到 Mark Word 当中；
2. 如果CAS 成功，此时线程A 就获取了锁
3. 如果线程CAS 失败，证明有别的线程持有锁，例如上图的线程B 来CAS 就失败的，这个时候启动偏向锁撤销 （revoke bias）；
4. 锁撤销流程：- 让 A线程在全局安全点阻塞（类似于GC前线程在安全点阻塞） - 遍历线程栈，查看是否有被锁对象的锁记录（ Lock Record），如果有Lock Record，需要修复锁记录和Markword，使其变成无锁状态。- 恢复A线程 - 将是否为偏向锁状态置为 0 ，开始进行轻量级加锁流程 （后面讲述）

### 偏向锁 -> 轻量级锁

1. 线程A在自己的栈桢中创建锁记录 LockRecord。
2. 线程A 将 Mark Word 拷贝到线程栈的 Lock Record中，这个位置叫 displayced hdr，如下图所示：
   [![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160829)](https://mmbiz.qpic.cn/mmbiz_png/SoGf97KLurAicd1Y2Vmx8AMVibe6YvE8giaLjST0XfiarbR6vlTRgVfrR9z91nLlhh0bGHekbQ3Libl6RrWcHml7rdA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
3. 将锁记录中的Owner指针指向加锁的对象（存放对象地址）。
4. 将锁对象的对象头的MarkWord替换为指向锁记录的指针。这二步如下图所示：
   [![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160836)](https://mmbiz.qpic.cn/mmbiz_png/SoGf97KLurAicd1Y2Vmx8AMVibe6YvE8giazHicpdHpADZMibb8N7nJexCxpMXL5076bTc0tibuC6xaJRdMVf9RSIA8Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
   这时锁标志位变成 00 ，表示轻量级锁

### 轻量级锁 -> 重量级锁

当锁升级为轻量级锁之后，如果依然有新线程过来竞争锁，首先新线程会自旋尝试获取锁，尝试到一定次数（默认10次）依然没有拿到，锁就会升级成重量级锁.

[![img](https://cdn.jsdelivr.net/gh/OneYX/Favorites@master/images/2021/07/19/20210719160930)](https://mmbiz.qpic.cn/mmbiz_png/SoGf97KLurAicd1Y2Vmx8AMVibe6YvE8gia5ExIwuCLlcVOLOcM1IFZYvWoJMhQUias2KUg4etbTJMdvicc8HXEcTpQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 将 MonitorObject 中的 _owner设置成 A线程；
2. 将 mark word 设置为 Monitor 对象地址，锁标志位改为10
3. 将B线程阻塞放到 ContentionList 队列；

JVM 每次从Waiting Queue 的尾部取出一个线程放到OnDeck作为候选者，但是如果并发比较高，Waiting Queue会被大量线程执行CAS操作，为了降低对尾部元素的竞争，将Waiting Queue 拆分成ContentionList 和 EntryList 二个队列, JVM将一部分线程移到EntryList 作为准备进OnDeck的预备线程。另外说明几点：

所有请求锁的线程首先被放在ContentionList这个竞争队列中;

Contention List 中那些有资格成为候选资源的线程被移动到 Entry List 中;

任意时刻，最多只有一个线程正在竞争锁资源，该线程被成为 OnDeck;

当前已经获取到所资源的线程被称为 Owner;

处于 ContentionList、EntryList、WaitSet 中的线程都处于阻塞状态，该阻塞是由操作系统来完成的(Linux 内核下采用 `pthread_mutex_lock` 内核函数实现的);

作为Owner 的A 线程执行过程中，可能调用wait 释放锁，这个时候A线程进入 Wait Set , 等待被唤醒。

这是 synchronized 在 JDK 6之前的实现原理。

## 关于synchronized 可能需要了解的关键点

### synchronized的可重入性

从互斥锁的设计上来说，当一个线程试图操作一个由其他线程持有的对象锁的临界资源时，将会处于阻塞状态，但当一个线程再次请求自己持有对象锁的临界资源时，这种情况属于重入锁，请求将会成功，在java中synchronized是基于原子性的内部锁机制，是可重入的，因此在一个线程调用synchronized方法的同时在其方法体内部调用该对象另一个synchronized方法，也就是说一个线程得到一个对象锁后再次请求该对象锁，是允许的，这就是synchronized的可重入性。如下：



```
public class AccountingSync implements Runnable{
    static AccountingSync instance=new AccountingSync();
    static int i=0;
    static int j=0;
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){

            //this,当前实例对象锁
            synchronized(this){
                i++;
                increase();//synchronized的可重入性
            }
        }
    }

    public synchronized void increase(){
        j++;
    }


    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

正如代码所演示的，在获取当前实例对象锁后进入synchronized代码块执行同步代码，并在代码块中调用了当前实例对象的另外一个synchronized方法，再次请求当前实例锁时，将被允许，进而执行方法体代码，这就是重入锁最直接的体现，需要特别注意另外一种情况，当子类继承父类时，子类也是可以通过可重入锁调用父类的同步方法。注意由于synchronized是基于monitor实现的，因此每次重入，monitor中的计数器仍会加1。

## 线程中断与synchronized

### 线程中断

正如中断二字所表达的意义，在线程运行(run方法)中间打断它，在Java中，提供了以下3个有关线程中断的方法



```
//中断线程（实例方法）
public void Thread.interrupt();

//判断线程是否被中断（实例方法）
public boolean Thread.isInterrupted();

//判断是否被中断并清除当前中断状态（静态方法）
public static boolean Thread.interrupted();
```

当一个线程处于被阻塞状态或者试图执行一个阻塞操作时，使用Thread.interrupt()方式中断该线程，注意此时将会抛出一个InterruptedException的异常，同时中断状态将会被复位(由中断状态改为非中断状态)，如下代码将演示该过程：



```
public class InterruputSleepThread3 {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread() {
            @Override
            public void run() {
                //while在try中，通过异常中断就可以退出run循环
                try {
                    while (true) {
                        //当前线程处于阻塞状态，异常必须捕捉处理，无法往外抛出
                        TimeUnit.SECONDS.sleep(2);
                    }
                } catch (InterruptedException e) {
                    System.out.println("Interruted When Sleep");
                    boolean interrupt = this.isInterrupted();
                    //中断状态被复位
                    System.out.println("interrupt:"+interrupt);
                }
            }
        };
        t1.start();
        TimeUnit.SECONDS.sleep(2);
        //中断处于阻塞状态的线程
        t1.interrupt();

        /**
         * 输出结果:
           Interruted When Sleep
           interrupt:false
         */
    }
}
```

如上述代码所示，我们创建一个线程，并在线程中调用了sleep方法从而使用线程进入阻塞状态，启动线程后，调用线程实例对象的interrupt方法中断阻塞异常，并抛出InterruptedException异常，此时中断状态也将被复位。这里有些人可能会诧异，为什么不用Thread.sleep(2000);而是用TimeUnit.SECONDS.sleep(2);其实原因很简单，前者使用时并没有明确的单位说明，而后者非常明确表达秒的单位，事实上后者的内部实现最终还是调用了Thread.sleep(2000);，但为了编写的代码语义更清晰，建议使用TimeUnit.SECONDS.sleep(2);的方式，注意TimeUnit是个枚举类型。ok~，除了阻塞中断的情景，我们还可能会遇到处于运行期且非阻塞的状态的线程，这种情况下，直接调用Thread.interrupt()中断线程是不会得到任响应的，如下代码，将无法中断非阻塞状态下的线程：

## 等待唤醒机制与synchronized

所谓等待唤醒机制本篇主要指的是notify/notifyAll和wait方法，在使用这3个方法时，必须处于synchronized代码块或者synchronized方法中，否则就会抛出IllegalMonitorStateException异常，这是因为调用这几个方法前必须拿到当前对象的监视器monitor对象，也就是说notify/notifyAll和wait方法依赖于monitor对象，在前面的分析中，我们知道monitor 存在于对象头的Mark Word 中(存储monitor引用指针)，而synchronized关键字可以获取 monitor ，这也就是为什么notify/notifyAll和wait方法必须在synchronized代码块或者synchronized方法调用的原因。



```
synchronized (obj) {
       obj.wait();
       obj.notify();
       obj.notifyAll();         
 }
```

需要特别理解的一点是，与sleep方法不同的是wait方法调用完成后，线程将被暂停，但wait方法将会释放当前持有的监视器锁(monitor)，直到有线程调用notify/notifyAll方法后方能继续执行，而sleep方法只让线程休眠并不释放锁。同时notify/notifyAll方法调用后，并不会马上释放监视器锁，而是在相应的synchronized(){}/synchronized方法执行结束后才自动释放锁。

## 来源

- https://blog.csdn.net/javazejian/article/details/72828483
- https://mp.weixin.qq.com/s/ts2Pjz3VpWm50kY-Ru7iTA
- https://www.cnblogs.com/woshimrf/p/java-synchronized.html