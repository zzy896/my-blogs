---
title: volatile详解
date: 2020-02-10 12:08:44
tags:
---
**一、volatile简介**
在单线程环境中，我们几乎用不到这个关键词，但是多线程环境中，这个关键词随处可见。总的来说，volatile有以下三个特性：

保证可见性；
不保证原子性；
禁止指令重排。
下面就来详细的说说这三个特性。
**二、保证可见性**
**1、什么是可见性？**
在说volatile保证可见性之前，先来说说什么叫可见性。谈到可见性，又不得不说JMM(java memory model)内存模型。JMM内存模型是逻辑上的划分，及并不是真实存在。Java线程之间的通信就由JMM控制。

我们定义的共享变量，是存储在主内存中的，也就是计算机的内存条中。线程A去操作共享变量的时候，并不能直接操作主内存中的值，而是将主内存中的值拷贝回自己的工作内存中，在工作内存中做修改。修改好后，在将值刷回到主内存中。

假设现在new 一个 student , age为 18，这个18是存储在主内存中的。现在两个线程先将18拷贝回自己的工作内存中。这时，A线程将18改为了20，刷回到主内存中。也就是说，现在主内存中的值变为了20。可是，B线程并不知道现在主内存中的值变了，因为A线程所做的操作对B是不可见的。我们需要一种机制，即一旦主内存中的值发生改变，就及时地通知所有的线程，保证他们对这个变化可见。这就是可见性。我们通常用happen - before(先行发生原则)，来阐述操作之间内存的可见性。也就是前一个的操作结果对后一个操作可见，那么这两个操作就存在 happen - before 规则。

**2、为什么volatile能保证可见性？**
先来说一说内存屏障(memory barrier)，这是一条CPU指令，可以影响数据的可见性。当变量用volatile修饰时，将会在写操作的后面加一条屏障指令，在读操作的前面加一条屏障指令。这样的话，一旦你写入完成，可以保证其他线程读到最新值，也就保证了可见性。

**3、验证volatile保证可见性。**
验证volatile可见性和不保证原子性的代码：

```bash
// 验证可见性
class MyData {
    //int number = 0; // 没加volatile关键字
    volatile int number = 0;
    int changeNumber() {
        return this.number = 60;
    }
}

public class VolatileTest {
    // 验证可见性
    public static void main(String[] args) {
        MyData myData = new MyData();
        new Thread("AAA") {
            public void run() {
                try {
                    Thread.sleep(3000);
                    // 睡3秒后调用changeNumber方法将number改为60
                    System.err.println(Thread.currentThread().getName() 
                                        +  " update number to " + myData.changeNumber());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            };
        }.start();
        // 主线程
        while (myData.number == 0) {
        }
        // 如果主线程读取到的一直都是最开始的0，
        //将造成死循环，这句话将无法输出
        System.err.println(Thread.currentThread().getName() 
                          + " get number value is " + myData.number);
    }
}
```
上面这段代码很简单，定义了一个MyData类，初始一个number，值为0。然后在main方法中创建另一个线程，将其值改为60。但是，这个线程对number所作的操作对main线程是不可见的，所以main线程以为number还是0，因此，将会造成死循环。如果number加了volatile修饰，main线程就可以获取到主内存中的最新值，就不会死循环。这就验证了volatile可以保证可见性。

**三、不保证原子性**
**1、什么叫原子性？**
所谓原子性，就是说一个操作不可被分割或加塞，要么全部执行，要么全不执行。

**2、volatile不保证原子性解析**
java程序在运行时，JVM将java文件编译成了class文件。我们使用javap命令对class文件进行反汇编，就可以查看到java编译器生成的字节码。最常见的 i++ 问题，其实 反汇编后是分三步进行的。

第一步：将i的初始值装载进工作内存；
第二步：在自己的工资内存中进行自增操作；
第三步：将自己工作内存的值刷回到主内存。
我们知道线程的执行具有随机性，假设现在i的初始值为0，有A和B两个线程对其进行++操作。首先两个线程将0拷贝到自己工作内存，当线程A在自己工作内存中进行了自增变成了1，还没来得及把1刷回到主内存，这是B线程抢到CPU执行权了。B将自己工作内存中的0进行自增，也变成了1。然后线程A将1刷回主内存，主内存此时变成了1，然后B也将1刷回主内存，主内存中的值还是1。本来A和B都对i进行了一次自增，此时主内存中的值应该是2，而结果是1，出现了写丢失的情况。这是因为i++本应该是一个原子操作，但是却被加塞了其他操作。所以说volatile不保证原子性。

**3、volatile不保证原子性验证**

```bash
 // 验证volatile不保证原子性
    void  addPlusPlus() {
        this.number++;
    }
    // 验证volatile不保证原子性
    public static void main(String[] args) {
        MyData mydata2 = new MyData();
        for(int i = 0; i < 20; i ++ ) { // 创建20个线程
            new Thread("线程" + i) {
                public void run() {
                    try {
                        for(int j = 0; j < 1000; j++) {
                            mydata2.addPlusPlus();// 每个线程执行1000次number++
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                };
            }.start();
        }
        // 保证上面的线程执行完main线程再输出结果。 大于2，因为默认有main线程和gc线程
        while(Thread.activeCount() > 2) {
            Thread.yield();
        }
        System.err.println(Thread.currentThread().getName() + " obtain the number is " + mydata2.number);
    }
```
同样是上面的MyData类，有一个volatile修饰的number变量初始值为0。现在有20个线程，每个线程对其执行1000次++操作。理论上执行完后，main线程输出的结果是20000，但是运行之后会发现，每次的运行结果都会小于20000，这就是因为出现了写丢失的情况。
解决办法：

第一种：可以在addPlusPlus方法中加synchronized；
第二种：可以使用原子包装类AtomicInteger。
第一种办法不太好，因为synchronized太重量级了，整个操作都加锁了。第二种办法更好。但是为什么AtomicInteger就可以保证原子性呢？因为它使用了CAS算法。什么是CAS？后续我再专门写一篇介绍CAS的文章。

**三、禁止指令重排**
**1、什么叫指令重排？**
上面说了，使用javap命令可以对class文件进行反汇编，查看到程序底层到底是如何执行的。像 i++ 这样一个简单的操作，底层就分三步执行。在多线程情况下，计算机为了提高执行效率，就会对这些步骤进行重排序，这就叫指令重排。比如现有如下代码：

```bash
int x = 1;
int y = 2;
x = x + 3;
y = x - 4;
```
这四条语句，正常的执行顺序是从上往下1234这样执行，x的结果应该是4，y的结果应该是0。但是在多线程环境中，编译器指令重排后，执行顺序可能就变成了1243，这样得出的x就是4，y就是-3，这结果显然就不正确了。不过编译器在重排的时候也会考虑数据的依赖性，比如执行顺序不可能为2413，因为第4条语句的执行是依赖x的。使用volatile修饰，就可以禁止指令重排。

**四、你在哪些地方使用过volatile？**

最经典的就是单例模式。

最简版单例模式：
```bash
public class SingletonDemo {
    private static SingletonDemo  singletonDemo = null;
    private SingletonDemo(){
        System.err.println("构造方法被执行");
    }
    public static SingletonDemo getInstance(){
        if (singletonDemo == null){
            singletonDemo = new SingletonDemo();
        }
        return singletonDemo;
    }
}
```
这是我们最开始学的时候写的单例模式。看似很完美。其实多线程环境中就会出问题。测试一下：
```bash
public static void main(String[] args){
        for (int i = 0; i <= 10; i++){
           new Thread(() -> SingletonDemo.getInstance()).start();
        }
}
```
10个线程去执行这个单例，看结果：
```bash
构造方法被执行
构造方法被执行

Process finished with exit code 0
```

构造方法被执行这句话打印了两次，说明创建了两次对象。所以在多线程环境中这个单例模式是有问题的。可以在getInstance方法上加synchronized，但是，这样就把一整个方法都锁了，这样不太好。下面介绍另一种方式。

**DCL版单例模式：**
DCL，是double check lock 的缩写，中文名叫双端检索机制。所谓双端检索，就是在加锁前和加锁后都用进行一次判断。代码如下：
```bash
public class SingletonDemo {
    private static SingletonDemo  singletonDemo = null;
    private SingletonDemo(){
        System.err.println("构造方法被执行");
    }
    public static SingletonDemo getInstance(){
        if (singletonDemo == null){ // 第一次check
            synchronized (SingletonDemo.class){
                if (singletonDemo == null) // 第二次check
                    singletonDemo = new SingletonDemo();
            }
        }
        return singletonDemo;
    }
 }
 ```
 
 用synchronized只锁住创建实例那部分代码，而不是整个方法。在加锁前和加锁后都进行判断，这就叫双端检索机制。经测试，这样确实只创建了一个对象。但是，这也并非绝对安全。new 一个对象也是分三步的：
 
 1.分配对象内存空间；(这个房间有人订了)
 2.初始化对象；(打扫好房间)
 3.将对象指向分配的内存地址，此时这个对象不为null。(订房间的人入住)
 步骤二和步骤三不存在数据依赖，因此编译器优化时允许这两句颠倒顺序。当指令重拍后，多线程去访问也会出问题。所以便有了如下的最终版单例模式。
 
 **最终版单例模式：**
 既然说到DCL版可能会出现指令重排的现象，所以最终版就是加上volatile。
 ```bash
 public class SingletonDemo {
     private static volatile SingletonDemo  singletonDemo = null;
     private SingletonDemo(){
         System.err.println("构造方法被执行");
     }
     public static SingletonDemo getInstance(){
         if (singletonDemo == null){ // 第一次check
             synchronized (SingletonDemo.class){
                 if (singletonDemo == null) // 第二次check
                     singletonDemo = new SingletonDemo();
             }
         }
         return singletonDemo;
     }
 }
  ```
 **总结：**
 **1、volatile特性：**
  
  可见性
  不保证原子性
  禁止指令重排
 **2、volatile的应用：**
  最经典的就是单例模式。