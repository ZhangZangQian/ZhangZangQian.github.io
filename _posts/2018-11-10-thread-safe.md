---
title: 线程安全
author: zhangzangqian
date: 2018-11-10 21:00:00 +0800
categories: [技术]
tags: [Java, 并发]
math: true
mermaid: true
---

## 概述

《Java Concurrency In Practice》的作者 Brian Geotz 对”线程安全“有一个比较恰当的定义：”**当多个线程访问一个对象时，如果不考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象是线程安全的。**“

## Java 语言中的线程安全

我们这里讨论的线程安全，就限定于多个线程之间存在共享数据访问这个前提，因为如果一端代码根本不会与其他线程共享数据，那么从线程安全的角度来看，程序是串行执行还是多线程执行对它来说是完全没有区别的。

**按照线程安全的“安全程度”由强至弱来排序，我们可以将 Java 语言中各种操作共享的数据分为以下 5 类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。**

### 不可变

如要一个不可变的对象被正确的构建出来（没有发生 this 引用逃逸的情况），那其外部的可见状态永远也不会改变，永远也不会看到它在多个线程之后处于不一致的状态。**“不可变”带来的安全性是最简单和最纯粹的。**

**Java 语言中，如果共享数据是一个基本数据类型，那么只要定义时使用 final 关键字修饰它就可以保证它是不可变得。如果一个共享数据类型是一个对象，那就需要保证对象的行为不会对其状态发生影响才行。java.lang.String 类的对象，它是一个典型的不可变对象，我们调用他的 subString()、replace() 和 concat() 这些方法都不会影响它原来的值，只会返回一个新构造的字符串对象。**

**保证对象行为不影响自己状态的途径有很多种，其中最简单的就是把对象中带有状态的变量都声明为 final，这样在构造结束之后，它就是不可变的。**如下代码片段， java.lang.Integer 构造函数所示的，他通过将内部状态变量的 value 定义为 final 来保障状态不变。

```java
/**
* The value of the <code>Integer</code>
* @serial
*/
private final int value;

/**
* Constructs a newly allocated <code>Integer</code> object that
* represents the specified <code>int</code> value.
* 
* @param value the value to be represented by the
*              <code>Integer</code> object.
*/
public Integer(int value) {
    this.value = value;
}
```

### 绝对线程安全

绝对的线程安全完全满足 Brian Goetz 给出的线程安全的定义，这个定义其实是很严格的，**一个类要达到“不管运行时环境如何，调用者都不需要任何额外的同步措施”通常需要付出很大的，甚至有时候不切实际的代价。在 Java API 中标注自己是线程安全的类，大多数都不是绝对的线程安全。**我们可以通过 Java API 中一个不是“绝对的线程安全”的线程安全类来看看这里的“绝对”是什么意思。

如果说 java.util.Vector 是一个线程安全的容器，相信所有的 Java 程序员对此都不会有异议，因为它的 add()、get() 和 size() 这类方法都是被 synchronized 修饰的，尽管这样效率很低，但确实是安全的。但是，即使它所有的方法都被修饰成同步，也不意味着调用它的时候永远都不再需要同步手段了。

```java
/**
* 对 Vector 线程安全的测试
*/
private static Vector<Integer> vector = new Vector<>();

public static void main(String[] args) {
    while(true) {
        for (int i = 0; i < 10; i ++) {
            vector.add(i);
        }

        Thread removThread = new Thread(new Runnable() {
            @Overried
            public void run() {
                for (int i = 0; i < vector.size(); i ++) {
                    vector.remove(i);
                }
            } 
        });

        Thread.printThread = new Thread(new Runnable() {
            @Overried
            public void run() {
                for (int i = 0; i < vector.size(); i ++) {
                    System.out.println(vector.get(i));
                }
            }
        });
        removeThread.start();
        printThread.start();

        // 不要同时产生过多的线程，否则会导致操作系统假死
        while(Thread.activeCount() > 20);
    }
}
```

运行结果如下：

```console
Exception in thread "Thread-119" java.lang.ArrayIndexOutOfBoundsException: Array index out of range: 17
	at java.util.Vector.get(Vector.java:748)
	at com.ys.sakalaka.concurrent.VolatileTest.lambda$main$1(VolatileTest.java:30)
	at java.lang.Thread.run(Thread.java:748)
```

很明显，尽管使用到的 Vector 的 get()、remove() 和 size() 方法都是同步的，但是在多线程的环境中，如果不在方法调用端做额外的同步措施的话，使用这段代码仍然是不安全的，如果因为另一个线程恰好在错误的时间里删除了一个元素，导致序号 i 已经不再可用的话，再用 i 访问数组就会抛出一个 ArrayIndexOutOfBoundsException。如果要保证这段代码能正确执行下去，我们不得不把 removeThread 和 printThread 的定义改成代码清单所示的样子。

```console
Thread removeThread = new Thread(new Runnable() {
    @Overried
    public void run() {
        synchronzied(vector) {
            for (int i = 0; i < vector.size(); i ++){
                vector.remove(i);
            }
        }
    }
});

Thread printThread = new Thread(new Runnable() {
    @Overrid
    public void run() {
        synchronized(vector) {
            for (int i = 0; i < vector.size(); i ++) {
                System.out.println(vector.get(i));
            }
        }
    }
});
```

### 相对线程安全

**相对线程安全就是我们通常意义上所讲的线程安全，它需要保证这个对象单独的操作是线程安全的，我们在调用的时候不需要做额外的保障措施，但是对于一些顺序的连续调用，就可能需要在调用端同步使用额外的手端来保证调用的正确性。**上面两段代码就是相对线程安全的明显案例。

在 Java 语言中，大部分线程安全的类都属于这种类型，例如 Vector、HashTable、Collections 的 synchronizedCollection() 方法包装的集合等。

### 线程兼容

**线程兼容是指对象本身并不是线程安全的，但是我们可以通过调用端正确的使用同步手段来保证对象在并发环境中可以安全的使用。**我们平常说一个类不是线程安全的，绝大多数的时候指的是这一种情况。Java API 中大部分的类都是属于线程兼容的，如与前面的 Vector 和 HashTable 相对应的集合类 ArrayList 和 HashMap 等。

### 线程独立

**线程对立指的是无论调用端是否采取了同步措施，都无法在多线程环境中并发使用的代码。由于 Java 语言天生就具备多线程特性，线程对立这种排斥多线程的代码时很少出现的，而且同时都是有害的，应当尽量避免。**

一个线程对立的例子是 Thread 类的 suspend() 和 resume() 方法，如果有两个线程同时持有一个线程对象，一个尝试去中断线程，另外一个尝试去恢复线程，如果并发进行的话，无论调用时是否进行了同步，目标线程都是存在死锁风险的，如果 suspend() 中断的线程就是即将要执行 resume() 的那个线程，那就肯定要产生死锁了。也正是由于这个原因，suspend() 和 resume() 方法已经被 JDK 声明废弃 （@Deprecated）了。常见的线程对立的操作还有 System.setIn()、System.setOut() 和 System.runFinalizersOnExit() 等。

## 线程安全的实现方法

如何实现线程安全与代码编写有很大的关系，但虚拟机提供的**同步和所机制**也起到了非常重要的作用。

### 互斥同步

互斥同步（Mutual Excusion & Synchronzation）是常见的一种并发正确性的保障手段。**同步是指在多个线程并发访问共享数据时，保证共享数据在同一时刻只被一个（或者是一些，使用信号量的时候）线程使用。而互斥是实现同步的一种手段，临界区（Critical Section）、互斥量（Mutex）和信号量（Semaphore）都是主要的互斥实现方式。因此，在这 4 个组勉励，互斥是因，同步是果；互斥是方法，同步是目的。**

最基本的互斥同步手段就是 synchronized 关键字，synchronized 关键字经过编译之后，会在同步块的前后分别形成 monitorenter 和 monitorexit 这两个字节码指令，这两个字节码都需要一个 reference 类型的参数来指明锁定和解锁的对象。**如果 Java 程序中的 synchronized 明确指定了对象参数，那就是这个对象的 reference；如果没有明确指定，那就根据 synchronized 修饰的是实例方法还是类方法，去取对应的对象实例或 Class 对象来作为锁对象。**

根据虚拟机规范的要求，在执行 monitorenter 指令时，首先要尝试获取对象的锁，如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加 1，相应的，在执行 monitorexit 指令时会将计数器减 1，当计数器为 0 时，锁就被释放。如果获取对象锁失败，那当前线程就要阻塞等待，知道对象锁被另外一个线程释放为止。

在虚拟机规范中 monitorenter 和 monitorexit 的行为描述中，有两点是需要特别注意的。**首先 synchronized 同步块对同一条线程来说是可重入的，不会出现把自己锁死的问题。**其次，**同步块在已进入的线程执行完之前，会阻塞后面其他线程的进入。**之前我们说过，Java 的线程时映射到操作系统的原生线程之上的，如果需要阻塞或唤醒一个线程，都需要操作系统来帮忙完成，这就需要从用户态转换到核心态中，因此状态转换需要耗费很多的处理器时间。对于代码简单的同步块（如果 synchronized 修饰的 getter() 或 setter() 方法），状态转换消耗的时间有可能比用户代码执行的是还要长。所以 synchronized 是 Java 语言中一个重量级（Heavyweight）的操作，有经验的程序员都会在确实必要的情况下才使用这种操作。而虚拟机本身也会进行一些优化，譬如在通知操作系统阻塞线程之前加入一段自旋等待过程，避免频繁的切入到核心状态之中。

**除了 synchronized 之外，我们还可以使用 java.util.concurrent（下文简称 J.U.C）包中的重入锁（ReentrantLock）来实现同步**，在基本用法上，ReentrantLock 与 synchronized 很相似，他们都具备一样的线程重入特性，只是代码写法上有点区别，一个表现为 API 层面的互斥锁（lock() 和 unlock() 方法配合 try/finally 语句块来完成），另一个表现为原生语法层面的互斥锁。不过，**相比 synchronized，ReentrantLock 增加了一些高级功能，主要有以下 3 项：等待可中断、可实现公平锁，以及锁可以绑定多个条件。**

- **等待可中断是指当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情，可中断特性对处理执行时间非常长的同步块很有帮助。**
- **公平锁是指多个线程在等待同一个锁是，必须按照申请锁的时间顺序来一次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但可以公国带布尔值的构造函数要求使用公平锁。**
- **锁绑定多个条件是指一个 ReentrantLock 对象可以同时绑定多个 Condition 对象，而在 synchronized 中，锁对象的 wait() 或 notify() 或 notifyAll(） 方法可以实现一个隐含的条件，如果要和多于一个的条件关联的时候，就不得不额外的添加一个锁，而 ReentrantLock 则无须这样做，只需要多次调用 newCondition() 方法即可。**

**JDK 1.6 中加入了很多针对锁的优化措施，JDK 1.6 发布之后，人们就发现 synchronized 与 ReentrantLock 的性能基本上是完全持平了。因此，如果程序是使用 JDK 1.6 或以上部署的话，性能因素就不再是选择 ReentrantLock 的理由了，虚拟机在未来的性能改进中肯定也会更加偏向于原生的 synchronized，所以还是提倡在 synchronized 能实现需求的情况下，优先考虑使用 synchronized 来进行同步。**

### 非阻塞同步

**互斥同步最主要的问题就是进行线程同步时阻塞和唤醒带来的性能问题，因此这种同步也称为阻塞同步（Blocking Synchronization）。从处理问题的方式上来说，互斥同步属于一种悲观的并发策略，总是认为只要不去做正确的同步措施（例如加锁），那就肯定会出现问题。**随着硬件指令集的发展，我们有了另外一个选择：基于冲突检测的乐观并发策略，通俗的说，就是先进行操作，如果没有其他线程征用共享数据，那操作就成功了；如果共享数据有征用，产生了冲突，那就再采取其他的补偿措施（最常见的补偿措施就是不断的重试，知道成功为止），**这种乐观的并发策略的许多实现都不需要把线程挂起，因此这种同步操作被称为非阻塞同步（Non-Blocking Synchronization）。**

为什么说使用乐观并发策略需要“硬件指令集的发展”才能进行呢？因为我们**需要操作和冲突检测这两个步骤具备原子性，靠什么来保证呢？如果这里再使用互斥同步来保证就是去意义了，所以我们只能靠硬件来完成这件事情，硬件保证一个从语义上看起来需要多次操作的行为只通过一条处理器指令就能完成。**这样的指令常用的有：

- 测试并设置（Test-and-Set）
- 获取并增加（Fetch-Increment）
- 交换（Swap）
- 比较并交换（Compare-and-Swap）
- 加载连接/条件存储（Load-Linked/Store-Conditional，下文成 LL/SC）

前面的 3 条是 20 世纪就已经存在于大多数指令集之中的处理器指令，后面的两条是现代处理器新增的，而且这两条指令的目的和功能是类似的。在 IA64、x86 指令集中有 cmpxchg 指令来完成 CAS 功能，在 sparc-TSO 也有 casa 指令实现，而在 ARM 和 PowerPC 架构下，则需要使用一堆 ldrex/strex 指令来完成 LL/SC 的功能。

CAS 指令需要有 3 个操作数，分别是内存位置（在 Java 中可以简单理解为变量的内存地址，用 V 表示）、就得预期值（用 A 表示）和新值（用 B 表示）。CAS 指令执行时，当且仅当 V 符合预期值 A 时，处理器用新值 B 更新 V 的值，否则它就不执行更新，但是无论是否更新 V 的值，都会返回 V 的旧值，上述处理过程是一个原子操作。

在 JDK 1.5 之后，Java 程序中才可以使用 CAS 操作，该操作由 sun.music.Unsafe 类里面的 compareAndSwapInt() 和 compareAndSwapLong() 等几个方法包装提供，虚拟机在内部对这些方法做了特殊处理，即时编译出来的记过就是一条平台相关的处理器 CAS 指令，没有方法调用的过程，或者可以认为是无条件内联进去了（这种被虚拟机特殊处理的方法称为固有函数（Intrinsics），类似的固有函数还有 Math.sin() 等）。

**CAS 存在这样一个漏洞：如果一个变量 V 初次读取的时候是 A 值，并且在这段期间它的值曾经被修改成了 B，后来又被该回了 A，那 CAS 操作就会误认为它从来没有被改变过。这个漏洞被称为 CAS 操作的“ABA”问题。**J.U.C 包为了解决这个问题，提供了一个带有标记的原子引用类 “AtomicStampedReference”，它可以通过控制变量值得版本来保证 CAS 的正确性。不过目前来说这个类比较“鸡肋”，大部分情况下 ABA 问题都不回影响程序并发的正确性，如果需要解决 ABA 问题，改用传统的互斥同步可能会比原子类更高效。

### 无同步方案

要保证线程安全，并不是一定就要进行同步，两者没有因果关系。同步只是保证共享数据争用时的正确性的手段，如果一个方法本来就不涉及共享数据，那它自然就无需任何同步措施去保证正确性，因此会有一些代码天生就是线程安全的。

#### 可重入代码（Reentrant Code）

**相对线程安全来说，可重入性是更基本的特性，它可以保证线程安全，即所有的可重入代码都是线程安全的，但是并非所有的线程安全的代码都是可重入的。可重入的代码有一些共同的特征，例如不依赖存储在堆上的数据和公用的系统资源、用到的状态量都由参数中传入、不调用非可重入的方法等。**

#### 线程本地存储（Thread Local Storage）

如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行？如果能保证，我们就可以把共享数据的可见范围限制在同一线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。

其中最重要的一个应用实例就是经典 Web 交互模式中的“一个请求对应一个服务器线程”（Thread-per-Request）的处理方式，这种处理方式的广泛应用使得很多 Web 服务端应用都可以使用线程本地存储来解决线程安全问题。

Java 语言中，如果一个变量要被多线程访问，可以使用 volatile 关键字声明它为“易变得”；如果一个变量要被某个线程独享，Java 中就没有类似 C++ 中 __declspec(thread) 这样的关键字，**不过还是可以通过 java.lang.ThreadLocal 类来实现线程本地存储的功能。每一个线程的 Thread 对象中都有一个 ThreadLocalMap 对象，这个对象存储了一组以 ThreadLocal.threadLocalHashCode 为键，以本地线程变量的值为K-V的值对，ThreadLocal 对象就是当前线程的 ThreadLocalMap 的访问入口，每一个 ThreadLocal 对象都包含了一个独一无二的 threadLocalHashCode 值，使用这个值就可以在线程 K-V 值对中找回对应的本地线程变量。**