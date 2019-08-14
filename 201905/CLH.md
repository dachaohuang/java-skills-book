# 线程安全实现与CLH队列

## 阻塞同步

作者：beanlam

链接：https://www.jianshu.com/p/0f6d3530d46b

来源：简书



在 Java 中，我们经常使用 synchronized 关键字来做到互斥同步以解决多线程并发访问共享数据的问题。synchronzied 关键字在编译后，会在 synchronized 所包含的同步代码块前后分别加入 monitorenter 和 monitorexit 这两个字节码指令。synchronized 关键字需要指定一个对象来进行加锁和解锁。例如：

```java
public class Main {

    private static final Object LOCK = new Object();
    
    public static void fun1() {
        synchronized (LOCK) {
            // do something
        }
    }
    
    public static void fun2() {
        synchronized (LOCK) {
            // do something
        }
    }
}
```

在没有明确指定该对象时，根据 synchonized 修饰的是实例方法还是静态方法，从而决定是采用对象实例或者类的class实例作为所对象。例如：

```java
public class SynchronizedTest {
    public synchronized void doSomething() {
        //采用实例对象作为锁对象
    }
  
  	public static synchronized void doSomething() {
        //采用SynchronizedTest.class 实例作为锁对象
    }
}
```

由于基于 synchronized 实现的阻塞互斥，除了需要阻塞操作线程，而且唤醒或者阻塞操作系统级别的原生线程，需要从用户态转换到内核态中，这个状态的转换消耗的时间可能比用户代码执行的时间还要长，因此我们经常说 synchronized 是 Java 语言中的 “重量级锁”。

## 非阻塞同步

### 乐观锁与悲观锁

使用 synchronized 关键字的同步方式最主要的问题就是进行线程阻塞和唤醒时所带来的的性能消耗问题。阻塞同步属于悲观的并发策略，只要有可能出现竞争，它都认为一定要加锁。

 然而同步策略还有另外一种乐观的策略，乐观并发策略先进性对数据的操作，如果没有发现其它线程也操作了数据，那么就认为这个操作是成功的。如果发生了其它线程也操作了数据，那么一般采取不断重试的手段，直到成功为止，这种乐观锁的策略，不需要把线程阻塞，属于非阻塞同步的一种手段。

### CAS

乐观并发策略主要有两个重要的阶段，一个是对数据进行操作，另外一个是进行冲突的检测，即检测其它线程有无同时也对该数据进行了操作。这里的数据操作和冲突检测需要具备**原子性**，否则就容易出现类似于 i++ 的问题。
 CAS 的含义为 compare and swap，目前绝大多数 CPU 都原生支持 CAS 原子指令，例如在 IA64、x86的指令集中，就有 cmpxchg 这样的指令来完成 CAS 功能，它的原子性要求是在硬件层面上得到保证的。
 CAS 指令一般需要有三个参数，分别是值的内存地址、期望中的旧值和新值。CAS 指令执行时，如果该内存地址上的值符合期望中的旧值，处理器会用新值更新该内存地址上的值，否则就不更新。这个操作在 CPU 内部保证了是原子性的。

 在 Java 中有许多 CAS 相关的 API，我们常见的有 `java.util.concurrent` 包下的各种原子类，例如`AtomicInteger`，`AtomicReference`等等。 这些类都支持 CAS 操作，其内部实际上也依赖于 `sun.misc.Unsafe` 这个类里的 compareAndSwapInt() 和 compareAndSwapLong() 方法。

 CAS 并非是完美无缺的，尽管它能保证原子性，但它存在一个著名的 **ABA** 问题。一个变量初次读取的时候值为 A，再一次读取的时候也为 A，那么我们是否能说明这个变量在两次读取中间没有发生过变化？不能。在这期间，变量可能由 A 变为 B，再由 B 变为 A，第二次读取的时候看到的是 A，但实际上这个变量发生了变化。一般的代码逻辑不会在意这个 ABA 问题，因为根据代码逻辑它不会影响并发的安全性，但如果在意的话，可能考虑采用阻塞同步的方式而不是 CAS。实际上 JDK 本身也对这个 ABA 问题解决方案，提供了 `AtomicStampedReference` 这个类，为变量加上版本来解决 ABA 问题。

### 自旋锁

以 synchronized 为代表的阻塞同步，因为阻塞线程会恢复线程的操作都需要涉及到操作系统层面的用户态和内核态之间的切换，这对系统的性能影响很大。自旋锁的策略是当线程去获取一个锁时，如果发现该锁已经被其它线程占有，那么它不马上放弃 CPU 的执行时间片，而是进入一个“无意义”的循环，查看该线程是否已经放弃了锁。
 但自旋锁适用于**临界区**比较小的情况，如果锁持有的时间过长，那么自旋操作本身就会白白耗掉系统的性能。

以下为一个简单的自旋锁实现：

```java
import java.util.concurrent.atomic.AtomicReference;
public class SpinLock {
   private AtomicReference<Thread> owner = new AtomicReference<Thread>();
   public void lock() {
       Thread currentThread = Thread.currentThread();
        // 如果锁未被占用，则设置当前线程为锁的拥有者
       while (!owner.compareAndSet(null, currentThread)) {}
   }

   public void unlock() {
       Thread currentThread = Thread.currentThread();
        // 只有锁的拥有者才能释放锁
       owner.compareAndSet(currentThread, null);
   }
}
```

上述的代码中， owner 变量保存获得了锁的线程。这里的自旋锁有一些缺点，第一个是没有保证公平性，等待获取锁的线程之间，无法按先后顺序分别获得锁；另一个，由于多个线程会去操作同一个变量 owner，在 CPU 的系统中，存在着各个 CPU 之间的缓存数据需要同步，保证一致性，这会带来性能问题。

### 公平的自旋

为了解决公平性问题，可以让每个锁拥有一个服务号，表示正在服务的线程，而每个线程尝试获取锁之前需要先获取一个排队号，然后不断轮询当前锁的服务号是否是自己的服务号，如果是，则表示获得了锁，否则就继续轮询。下面是一个简单的实现：

```
import java.util.concurrent.atomic.AtomicInteger;

public class TicketLock {
   private AtomicInteger serviceNum = new AtomicInteger(); // 服务号
   private AtomicInteger ticketNum = new AtomicInteger(); // 排队号

   public int lock() {
       // 首先原子性地获得一个排队号
       int myTicketNum = ticketNum.getAndIncrement();
       // 只要当前服务号不是自己的就不断轮询
       while (serviceNum.get() != myTicketNum) {
       }
       return myTicketNum;
    }

    public void unlock(int myTicket) {
        // 只有当前线程拥有者才能释放锁
        int next = myTicket + 1;
        serviceNum.compareAndSet(myTicket, next);
    }
}
```

虽然解决了公平性的问题，但依然存在前面说的多 CPU 缓存的同步问题，因为每个线程占用的 CPU 都在同时读写同一个变量 serviceNum，这会导致繁重的系统总线流量和内存操作次数，从而降低了系统整体的性能。

### MCS 自旋锁

MCS 的名称来自其发明人的名字：John Mellor-Crummey和Michael Scott。MCS 的实现是基于链表的，每个申请锁的线程都是链表上的一个节点，这些线程会一直轮询自己的本地变量，来知道它自己是否获得了锁。已经获得了锁的线程在释放锁的时候，负责通知其它线程，这样 CPU 之间缓存的同步操作就减少了很多，仅在线程通知另外一个线程的时候发生，降低了系统总线和内存的开销。实现如下所示：

```java
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;
public class MCSLock {
    public static class MCSNode {
        volatile MCSNode next;
        volatile boolean isWaiting = true; // 默认是在等待锁
    }
    volatile MCSNode queue;// 指向最后一个申请锁的MCSNode
    private static final AtomicReferenceFieldUpdater<MCSLock, MCSNode> UPDATER = AtomicReferenceFieldUpdater
            .newUpdater(MCSLock.class, MCSNode.class, "queue");

    public void lock(MCSNode currentThread) {
        MCSNode predecessor = UPDATER.getAndSet(this, currentThread);// step 1
        if (predecessor != null) {
            predecessor.next = currentThread;// step 2
            while (currentThread.isWaiting) {// step 3
            }
        } else { // 只有一个线程在使用锁，没有前驱来通知它，所以得自己标记自己已获得锁
            currentThread.isWaiting = false;
        }
    }

    public void unlock(MCSNode currentThread) {
        if (currentThread.isWaiting) {// 锁拥有者进行释放锁才有意义
            return;
        }

        if (currentThread.next == null) {// 检查是否有人排在自己后面
            if (UPDATER.compareAndSet(this, currentThread, null)) {// step 4
                // compareAndSet返回true表示确实没有人排在自己后面
                return;
            } else {
                // 突然有人排在自己后面了，可能还不知道是谁，下面是等待后续者
                // 这里之所以要忙等是因为：step 1执行完后，step 2可能还没执行完
                while (currentThread.next == null) { // step 5
                }
            }
        }
        currentThread.next.isWaiting = false;
        currentThread.next = null;// for GC
    }
}
```

MCS 的能够保证较高的效率，降低不必要的性能消耗，并且它是公平的自旋锁。

### CLH 自旋锁

CLH 锁与 MCS 锁的原理大致相同，都是各个线程轮询各自关注的变量，来避免多个线程对同一个变量的轮询，从而从 CPU 缓存一致性的角度上减少了系统的消耗。CLH 锁的名字也与他们的发明人的名字相关：Craig，Landin and Hagersten。
CLH 锁与 MCS 锁最大的不同是，MCS 轮询的是当前队列节点的变量，而 CLH 轮询的是当前节点的前驱节点的变量，来判断前一个线程是否释放了锁。
 实现如下所示：

```java
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;
public class CLHLock {
    public static class CLHNode {
        private volatile boolean isWaiting = true; // 默认是在等待锁
    }
    private volatile CLHNode tail ;
    private static final AtomicReferenceFieldUpdater<CLHLock, CLHNode> UPDATER = AtomicReferenceFieldUpdater
            . newUpdater(CLHLock.class, CLHNode .class , "tail" );
    public void lock(CLHNode currentThread) {
        CLHNode preNode = UPDATER.getAndSet( this, currentThread);
        if(preNode != null) {//已有线程占用了锁，进入自旋
            while(preNode.isWaiting ) {
            }
        }
    }

    public void unlock(CLHNode currentThread) {
        // 如果队列里只有当前线程，则释放对当前线程的引用（for GC）。
        if (!UPDATER .compareAndSet(this, currentThread, null)) {
            // 还有后续线程
            currentThread.isWaiting = false ;// 改变状态，让后续线程结束自旋
        }
    }
}
```

从上面可以看到，MCS 和 CLH 相比，CLH 的代码比 MCS 要少得多；CLH是在前驱节点的属性上自旋，而MCS是在本地属性变量上自旋；CLH的队列是隐式的，通过轮询关注上一个节点的某个变量，隐式地形成了链式的关系，但CLHNode并不实际持有下一个节点，MCS的队列是物理存在的，而 CLH 的队列是逻辑上存在的；此外，CLH 锁释放时只需要改变自己的属性，MCS 锁释放则需要改变后继节点的属性。

CLH 队列是 J.U.C 中 AQS 框架实现的核心原理。

