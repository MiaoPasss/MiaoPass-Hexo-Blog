---
title: synchronized和lock(CAS)的区别
date: 2026-03-10 17:28:17
categories: Programming
tags: 
- Operating System
---

synchronized：在需要同步的对象中加入此控制，synchronized可以加在方法上，也可以加在特定代码块中，括号中表示需要锁的对象。
lock：一般使用ReentrantLock类做为锁。在加锁和解锁处需要通过lock()和unlock()显示指出。所以一般会在finally块中写unlock()以防死锁。

1. ReentrantLock支持非阻塞的方式获取锁，能够响应中断，而synchronized不行
2. ReentrantLock必须手动获取和释放锁，而synchronized不需要
3. ReentrantLock可以是公平锁或者非公平锁，而synchronized只能是非公平锁
    * 公平锁 (Fair Lock)：线程申请锁的顺序与获取锁的顺序完全一致，FIFO (First Input First Output)。先排队的线程先获得锁。new ReentrantLock(true)。其内部不仅看锁状态，还会判断 hasQueuedPredecessors()，即等待队列中是否有前驱线程，有则不抢。
    * 非公平锁 (Non-Fair Lock)：不保证顺序。多个线程尝试获取锁时，可能新到的线程直接加塞抢到锁，而不按排队顺序。new ReentrantLock(false)（默认）。调用 lock() 时直接通过 CAS 尝试获取锁，抢不到再进入队列。
4. ReentrantLock在发生异常时，如果没有通过unlock去释放锁，很有可能造成死锁，因此需要在finally块中释放锁
    * 而synchronized在发生异常的时候，会自动释放线程占有的锁
5. synchronized和ReentrantLock都是可重入锁（某个线程已经获得某个锁，可以再次获取锁而不会出现死锁，即re-entrant）

为什么要使用非公平锁？
* 公平锁维护等待队列增加上下文切换，性能较慢；非公平锁利用 CAS 抢锁，在高并发场景下提升性能效率高。
    * 当获取公平锁时，先将线程自己添加到等待队列的队尾并休眠，当某线程用完锁之后，会去唤醒等待队列中队首的线程尝试去获取锁，锁的使用顺序也就是队列中的先后顺序，在整个过程中，线程会从运行状态切换到休眠状态，再从休眠状态恢复成运行状态，但线程每次休眠和恢复都需要从用户态转换成内核态，而这个状态的转换是比较慢的，所以公平锁的执行速度会比较慢。
    * 当获取非公平锁时，会先通过 CAS 尝试获取锁，如果获取成功就直接拥有锁，如果获取锁失败才会进入等待队列，等待下次尝试获取锁。这样做的好处是，获取锁不用遵循先到先得的规则，从而避免了线程休眠和恢复的操作
* 减少线程切换：被唤醒的线程还需要排队等待，不如让当前运行的线程（持有 CPU）直接抢，能更快完成任务。
* 提高吞吐量：即使有先来后到，非公平锁利用空闲时间直接抢锁，能让整体效率更高。
* 公平锁适用于需要精确控制执行顺序的业务，如金融转账、需要严格时间戳的日志记录。

---

# Synchronized

synchronized能够保证多个线程在同时执行被synchronized包裹的同一代码块时，有且仅有一个线程能执行相应的代码操作，而其他的线程会被阻塞等待。

```java synchronized 关键字包裹不同的目标
public class SyncTest {
    private Integer p = 0;

    // synchronized 修饰整个方法
    // 当多个线程同时调用该方法，只有一个能打印出 “准备获得属性p” 并返回p，之后别的线程再去竞争方法的使用权。
    private synchronized int getP(){
        System.out.println("准备获得属性p");
        return p;
    }

    // synchronized 包裹属性p
    // 当多个线程同时调用该方法时，会一起显示 ”准备属性p加一“ ，但同一时间段只有一个线程能够执行该操作。
    private void pAndOne(){
        System.out.println("准备属性p加一");
        synchronized (this.p){
            this.p +=1;
        }
    }

    // synchronized 包裹对象本身
    // 只要有一个线程执行根据对象本身同步的代码块，那么这部分代码块以及pAndOne的同步代码块，别的线程都是不能执行的。
    private void reset(){
        System.out.println("重置属性p");
        synchronized (this){
            this.p = 0;
        }
    }
}
```

## 实现原理
JAVA 虚拟机类加载机制和字节码执行引擎会在类和方法上添加访问标志这一块内容，用来标记类是否是静态是否是public，方法是否是public等等。

对于方法的同步，是通过方法的访问标志 *ACC_SYNCHRONIZED* 来控制的，即执行指定方法前会通过访问标志来判断是否需要和其他线程同步。
而对于针对对象的同步，则是通过字节指令来实现的，即先引入对象引用到当前栈，使用 *monitorenter* 字节指令告诉虚拟机该引用需要同步， *monitorexist* 字节指令表示退出。

# lock（CAS）
synchronized会一直获取执行权限直到执行完毕，即每个线程在执行相关代码块时都要与其他线程同步确认是否可以执行代码，容易影响性能。

lock可以帮我们实现尝试立刻获取锁，在指定时间内尝试获取锁，一直获取锁等操作，而semaphore信号量可以帮我们实现允许最多指定数量的线程获取锁。

## Unsafe
unsafe中的方法能够帮我们获取到一个对象的属性位于对象开始位置的相对距离，也就是说对象属性所在的地址与对象起始地址的差值。同时，还能获取一个对象指定相对距离后的数据，例如long，int，byte等等。最重要的是可以给一个特定的地址设置上数据。

## Compare And Swap
在unsafe类中，支持这样一个方法，compareAndSwapInt。它的含义是给一个对象var1（开始位置+指定长度var2）的地址写入一个int值var4，如果这个地方原来的值是var5。成功返回true，不成功返回false。

这个操作是本地方法调用，而具体一点，这个方法会直接调用cpu的compare_and_swap指令，这个指令是原子性的，即操作内存中一个地址上的值不会被中断。而且多核cpu间都是可见的。

借由这样的一个本地方法调用，jdk实现了一系列轻量级的非阻塞锁以及相关应用，例如ReentrantLock，Semaphore，ConcurrentHashMap，AtomicInteger等等。

## 性能问题
* 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题。
* 一个线程持有锁会导致其它所有需要此锁的线程挂起。
* 如果一个优先级高的线程等待一个优先级低的线程释放锁会导致优先级倒置，引起性能风险。
* volatile是不错的机制，但是volatile不能保证原子性。因此对于同步最终还是要回到锁机制上来。
    * Java 的 volatile 关键字是轻量级的同步机制，当一个线程修改了 volatile 修饰的变量，新值会立即刷新到主内存，当其他线程读取该变量时，会强制从主内存重新获取，确保读到最新值。
    * volatile 无法保证复合操作的原子性。例如 i++ 实际上包含“读取-修改-写入”三个步骤，使用 volatile 仍可能在多线程环境下产生冲突

## 实现原理
CAS是解决多线程并行情况下使用锁造成性能损耗的一种机制。

* CAS操作包含三个操作数——内存位置（V）、预期原值（A）、新值(B)。
* 如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。
* 无论哪种情况，它都会在CAS指令之前返回该位置的值。
* CAS有效地说明了“我认为位置V应该包含值A；如果包含该值，则将B放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。

假设内存中的原数据V，旧的预期值A，需要修改的新值B

* 比较 A 与 V 是否相等
* 如果比较相等，将 B 写入 V
* 返回操作是否成功

## 具体算法
* 在对变量进行计算之前(如 ++ 操作)，首先读取原变量值，称为 旧的预期值 A
* 然后在更新之前再获取当前内存中的值，称为 当前内存值 V
    * 如果 A==V 则说明变量从未被其他线程修改过，此时将会写入新值 B
    * 如果 A!=V 则说明变量已经被其他线程修改过，当前线程应当什么也不做。

## Java CAS使用示例
```java 使用 AtomicInteger 实现无锁计数器
import java.util.concurrent.atomic.AtomicInteger;

public class CasExample {
    // 1. 初始化原子整型变量
    private static AtomicInteger counter = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                // 自旋CAS操作，直到更新成功
                while (true) {
                    int oldValue = counter.get(); // 预期值E
                    int newValue = oldValue + 1; // 新值N
                    
                    // 2. 比较并交换：如果当前值等于oldValue，则赋值为newValue
                    if (counter.compareAndSet(oldValue, newValue)) {
                        break; // 成功则退出循环
                    }
                    // 失败说明被其他线程修改，自动下一次循环获取最新值再试
                }
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("最终计数器结果: " + counter.get()); // 预期为 2000
    }
}
```

## 使用 CAS 需要注意的问题
* ABA问题
因为CAS需要在操作值的时候，检查值有没有发生变化，没有发生变化才去更新。
但是如果一个值原来是A变成了B，又变成了A，CAS检查会判断该值未发生变化，实际却变化了。
解决思路：增加版本号，每次变量更新时把版本号+1，A-B-A就变成了1A-2B-3A。JDK5之后的atomic包提供了AtomicStampedReference来解决ABA问题，它的compareAndSet方法会首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志。全部相等，才会以原子方式将该引用、该标志的值设置为更新值。

* 时间长、开销大
自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。

* 只能保证一个共享变量的原子操作
对一个共享变量执行操作时，可以循环CAS方式确保原子操作。
但是对多个共享变量，就不灵了。
这里可以使用锁，或把多个共享变量合并为1个共享变量，如i=2,j=a,合并为ij=2a。然后用CAS操作ij。在JDK5后，提供了AtomicReference类来保证对象间的原子性，可以把多个共享变量放在一个对象里进行CAS操作。

# synchronized性能优化
synchronized是托管给 JVM 执行的， 而lock是java写的控制锁的代码。

在Java1.5中，synchronize是性能低效的。因为这是一个重量级操作，需要调用操作接口，导致有可能加锁消耗的系统时间比加锁以外的操作还多。相比之下使用Java提供的Lock对象，性能更高一些。

但从Java1.6开始。synchronize在语义上很清晰，可以进行很多优化，有适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等。导致在Java1.6上synchronize的性能并不比Lock差。官方也表示，他们也更支持synchronize，在未来的版本中还有优化余地。

# 机制区别
synchronized原始采用的是CPU悲观锁机制，即线程获得的是独占锁。独占锁意味着其他线程只能依靠阻塞来等待线程释放锁。而在CPU转换线程阻塞时会引起线程上下文切换，当有很多线程竞争锁的时候，会引起CPU频繁的上下文切换导致效率很低。

而Lock用的是乐观锁方式。所谓乐观锁就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。乐观锁实现的机制就是CAS操作（Compare and Swap ）。我们可以进一步研究ReentrantLock的源代码，会发现其中比较重要的获得锁的一个方法是compareAndSetState。这里其实就是调用的CPU提供的特殊指令。

现代的CPU提供了指令，可以自动更新共享数据，而且能够检测到其他线程的干扰，而 compareAndSet() 就用这些代替了锁定。这个算法称作非阻塞算法，意思是一个线程的失败或者挂起不应该影响其他线程的失败或挂起的算法。

## 悲观锁 synchronized
* 假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作。
* 悲观锁的实现，往往依靠底层提供的锁机制。
* 悲观锁会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。

## 乐观锁
* 假设不会发生并发冲突，每次不加锁而是假设没有冲突而去完成某项操作，只在提交操作时检查是否违反数据完整性。
* 如果因为冲突失败就重试，直到成功为止。
* 乐观锁大多是基于数据版本记录机制实现。
* 为数据增加一个版本标识，比如在基于数据库表的版本解决方案中，一般是通过为数据库表增加一个 “version” 字段来实现。读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一。
* 此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据。
* 乐观锁的缺点是不能解决脏读的问题。
* 在实际生产环境里边,如果并发量不大且不允许脏读，可以使用悲观锁解决并发问题。
* 如果系统的并发非常大的话,悲观锁定会带来非常大的性能问题,所以我们就要选择乐观锁定的方法。

# 用途区别
在非常复杂的同步应用中，请考虑使用ReentrantLock，特别是遇到下面几种种需求的时候。

1.某个线程在等待一个锁的控制权的这段时间需要中断 
2.需要分开处理一些wait-notify，ReentrantLock里面的Condition应用，能够控制notify哪个线程 
3.具有公平锁功能，每个到来的线程都将排队等候

---

# Credits

https://cloud.tencent.com/developer/article/1622173
https://blog.csdn.net/qq_27828675/article/details/115372519
https://cloud.tencent.com/developer/article/1997322

---