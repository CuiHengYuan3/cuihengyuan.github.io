##                         volatile,synchronized实现原理以及几种锁的介绍

Java代码在编译后会变成Java字节码，字节码被类加载器加载到JVM里，JVM执行字节码，最终需要转化为汇编指令在CPU上执行，Java中所使用的并发机制依赖于JVM的实现和CPU的指令。

volatile是轻量级的synchronized，它在多处理器开发中保证了共享变量的“可见性”。可见性的意思是当一个线程修改一个共享变量时，另外一个线程能读到这个修改的值。如果volatile变量修饰符使用恰当的话，它比synchronized的使用和执行成本更低，因为它不会引起线程上下文的切换和调度。如果一个字段被声明成volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的

```
0x01a3de1d: movb $0×0,0×1104800(%esi);0x01a3de24: lock addl $0×0,(%esp);
```

有volatile变量修饰的共享变量进行写操作的时候会多出第二行汇编代码,有两个作用 

1.将当前处理器缓存行的数据写回到系统内存。

2.这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。

利用synchronized实现同步的基础：Java中的每一个对象都可以作为锁。具体表现为以下3种形式。

❑ 对于普通同步方法，锁是当前实例对象。

❑ 对于静态同步方法，锁是当前类的Class对象。

❑ 对于同步方法块，锁是Synchonized括号里配置的对象。

当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁

synchronized关键字最主要有以下3种应用方式

修饰实例方法，作用于当前实例加锁，进入同步代码前要获得当前实例的锁



修饰静态方法，作用于当前类对象加锁，进入同步代码前要获得当前类对象的锁



修饰代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁



JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，但两者的实现细节不一样。



synchronized可以使修饰的代码块，方法线程安全的调用。看一个简单的例子

```Java
public class SynchronizedDemo implements Runnable {

    private static int sharedVar = 1;

    private void increase() {
        sharedVar++;
    }

    @Override
    public void run() {

        for (int i = 0; i < 5000; i++) {
            increase();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        SynchronizedDemo demo = new SynchronizedDemo();

        Thread thread1 = new Thread(demo);
        Thread thread2 = new Thread(demo);
        Thread thread3 = new Thread(demo);
        Thread thread4 = new Thread(demo);

        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();

        thread1.join();
        thread2.join();
        thread3.join();
        thread4.join();
        System.out.println(sharedVar);
    }

}
```



类中有一个静态变量sharedVar，类中有一个让这个变量自增的方法increase，由于实现了runnable接口，在run方法里面（线程中）调用5000次increase方法。

然后在main中以当前的一个对象创建了4个线程并开始启动，然后分别调用join方法，线程的join方法指的是调用线程等待该线程完成后，才能继续用下运行。意思就是main线程停止，等4个线程执行完后在走，保证打印的sharedVar是最终的值。

​                                                                  run一下看看答案

![1568880368618](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568880368618.png)

意料之中，由于多线程造成的并发，sharedVar++并非原子操作，是先读取值，在加上1成为一个新的值再写入。

如果在一个线程读的时候另一个线程也在读，最后会得到一样的值写入，那么就相当于浪费了一次循环。

![1568881047962](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568881047962.png)

```
//加上synchronized
private synchronized void  increase() {
    sharedVar++;
}
```

​                                                                                 再来一次

![1568881124886](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568881124886.png)

**这就是线程安全的答案**



当一个线程正在访问一个对象的 synchronized 实例方法，那么其他线程不能访问该对象的其他 synchronized 方法（因为线程访问synchronized 方法需要获得相应的锁），毕竟一个对象只有一把锁，当一个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，所以无法访问该对象的其他synchronized实例方法，但是其他线程还是可以访问该实例对象的其他非synchronized方法。

## synchronized作用于静态方法

```java
public class SynchronizedDemo implements Runnable {

    private static int sharedVar = 1;

    private synchronized void  increase() {
        sharedVar++;
    }

    //静态
    private  static synchronized void  StaticIncrease() {
        sharedVar++;
    }

    @Override
    public void run() {

        for (int i = 0; i < 5000; i++) {
            increase();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        SynchronizedDemo demo = new SynchronizedDemo();

        Thread thread1 = new Thread(demo);

        Thread thread2 = new Thread(() -> {
            for (int i = 0; i <5000 ; i++) {
                StaticIncrease();
            }
        });


        thread1.start();
        thread2.start();

        thread1.join();
        thread2.join();

        System.out.println(sharedVar);
    }

}
```

![1568882078530](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568882078530.png)

显然又发生的并发，很多人可能会问，不是加了synchronized吗？注意，前面说过，当synchronized作用于静态方法的时候，对应的锁是当前类的Class对象，而作用于成员方法的时候锁是当前的对象。所以此时的这两个线程分别执行这两个方法是可以的，也就是又同时操作了sharedVar,导致了并发



## synchronized同步代码块

除了使用关键字修饰实例方法和静态方法外，还可以使用同步代码块，在某些情况下，我们编写的方法体可能比较大，同时存在一些比较耗时的操作，而需要同步的代码又只有一小部分，如果直接对整个方法进行同步操作，可能会得不偿失，此时我们可以使用同步代码块的方式对需要同步的代码进行包裹，这样就无需对整个方法进行同步操作了
这就不演示了



## synchronized原理

#### java对象头与Monitor：

ava 虚拟机中的同步(Synchronization)基于进入和退出管程(Monitor)对象实现， 无论是显式同步(有明确的 monitorenter 和 monitorexit 指令,即同步代码块)还是隐式同步都是如此。在 Java 语言中，同步用的最多的地方可能是被 synchronized 修饰的同步方法。



![1568883196523](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568883196523.png)



synchronized用的锁是存在Java对象头里的。如果对象是数组类型，则虚拟机用3个字宽（Word）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，1字宽等于4字节，即32bit

![1568883413969](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568883413969.png)

Java对象头里的Mark Word里默认存储对象的HashCode、分代年龄和锁标记位。32位JVM的Mark Word的默认存储结构如表

![1568987661754](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568987661754.png)

在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化。Mark Word可能变化为存储以下4种数据

![1568987846085](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568987846085.png)

在Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级

![1568990450279](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1568990450279.png)



每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式

当一个 monitor 被某个线程持有后，它便处于锁定状态

monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）

```c++
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

ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1

**在某种程度上monitor对象就是“锁”**

同步语句块的实现使用的是monitorenter 和 monitorexit 指令，其中monitorenter指令指向同步代码块的开始位置，monitorexit指令则指明同步代码块的结束位置，当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。
倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。