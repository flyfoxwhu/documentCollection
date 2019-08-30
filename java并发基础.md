# Java并发基础
## 一、Java并发的常见概念
### 1、原子性、可见性与有序性
Java内存模型是围绕着在并发过程中如何处理原子性、可见性和有序性这3个特征来建立的。
#### 原子性
原子性（Atomicity）：由Java内存模型来直接保证的原子性变量操作包括 read、load、assign、use、store 和write，我们大致可以认为基本数据类型的访问读写是具备原子性的（例外就是 long 和 double 的非原子性协定，读者只要知道这件事就可以了，无须太过在意这些几乎不会发生的例外情况）。

如果应用场景需要一个更大范围的原子性保证（经常会遇到），Java 内存模型还提供了lock和unlock操作来满足这种需求，尽管虚拟机未把lock和unlock操作直接开放给用户使用，但是却提供了更高层次的字节码指令 monitorenter和monitorexit来隐式地使用这两个操作，这两个字节码指令反映到 Java 代码中就是同步块——synchronized关键字，因此在 synchronized 块之间的操作也具备原子性。
#### 可见性
可见性（Visibility）：可见性是指当一个线程修改了共享变量的值，其他线程能够立即得知这个修改。上文在讲解 volatile 变量的时候我们已详细讨论过这一点。Java 内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值这种依赖主内存作为传递媒介的方式来实现可见性的，无论是普通变量还是 volatile 变量都是如此，普通变量与 volatile 变量的区别是，volatile 的特殊规则保证了新值能立即同步到主内存，以及每次使用前立即从主内存刷新。因此，可以说 volatile 保证了多线程操作时变量的可见性，而普通变量则不能保证这一点。

除了 volatile 之外，Java还有两个关键字能实现可见性，即 synchronized和final。同步块的可见性是由 “对一个变量执行 unlock操作之前，必须先把此变量同步会主内存中（执行 store、write 操作）” 这条规则获得的，而 final 关键字的可见性是指：被 final 修饰的字段在构造器中一旦初始化完成，并且构造器没有把 “this” 的引用传递出去（this 引用逃逸是一件很危险的事情，其他线程有可能通过这个引用访问到 “初始化了一半” 的对象），那在其他线程中就能看见 final 字段的值。
#### 有序性
有序性（Ordering）：Java内存模型的有序性在前面讲解 volatile 时也详细地讨论过了，Java 程序中天然的有序性可以总结为一句话：如果在本线程内观察，所有的操作都是有序的；如果在一个线程中观察另一个线程，所有的操作都是无序的。前半句是指 “线程内表现为串行的语义” （Within-Thread As-If-Serial Semantics），后半句是指 “指令重排序” 现象和 “工作内存与主内存同步延迟” 现象。

### 2、Java内存模型happens-before原则
Java内存模型下存在一些 “天然的” 先行发生关系，即，“happens-before原则”。这些先行发生关系无须任何同步器协助就已经存在，可以在编码中直接使用。如果两个操作之间的关系不在此列，并且无法从下列规则推导出来的话，它们就没有顺序性保障，虚拟机可以对它们随意地进行重排序。
JSR-133内存模型使用happens-before的概念来阐述操作之间的内存可见性。在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系。这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。

**happens-before规则**：
* 程序顺序规则（Program Order Rule）：在一个线程内，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结果。
* 管程锁定规则（Monitor Lock Rule）：一个 unlock 操作先行发生于后面对同一个锁的 lock 操作。这里必须强调的是同一个锁，而 “后面” 是指时间上的先后顺序。
* volatile变量规则（Volatile Variable Rule）：对一个 volatile 变量的写操作先行发生于后面对这个变量的读操作，这里的 “后面” 同样是指时间上的先后顺序。
* 线程启动规则（Thread Start Rule）：Thread 对象的 start() 方法先行发生于此线程的每一个动作。
* 线程终止规则（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过 Thread.join() 方法结束、Thread.isAlive() 的返回值等手段检测到线程已经终止执行。
* 线程中断规则（Thread Interruption Rule）：对线程 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过 Thread.interrupted() 方法检测到是否有中断发生。
* 对象终结规则（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。
* 传递性（Transitivity）：如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那就可以得出操作 A 先行发生于操作 C 的结论。

其中与程序员密切相关的happens-before规则有：程序顺序规则、管程锁定规则、volatile变量规则、传递性

注意，两个操作之间具有 happens-before 关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before 仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前（the first is visible to and ordered before the second）。
其中有监视器锁规则：对一个监视器的解锁，happens-before 于随后对这个监视器的加锁。这一条，仅仅只是针对synchronized的set方法，而对于get并没有这方面的说明。

其实在这种上下文下面一个synchronized的set方法，一个普通的get方法，a线程调用set方法，b线程并不一定能对这个改变可见！

### 3、volatile
**volatile可见性**

前面happens-before原则就提到：volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。volatile从而保证了多线程下的可见性！！！

**volatile禁止内存重排序**

下面是JMM针对编译器制定的volatile重排序规则表：
![](https://i.imgur.com/k6Yim4I.jpg)

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

下面是基于保守策略的JMM内存屏障插入策略：
* 在每个volatile写操作的前面插入一个StoreStore屏障。
* 在每个volatile写操作的后面插入一个StoreLoad屏障。
* 在每个volatile读操作的后面插入一个LoadLoad屏障。
* 在每个volatile读操作的后面插入一个LoadStore屏障。

不同屏障的作用：
* StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作都已经刷新到主内存中
* StoreLoad屏障的作用是避免volatile写与后面可能有的volatile读/写操作重排序
* LoadLoad屏障用来禁止处理器把上面的volatile读与下面的普通读重排序
* LoadStore屏障用来禁止处理器把上面的volatile读与下面的普通写重排序

下面是保守策略下，volatile写操作插入内存屏障后生成的指令序列示意图：
![](https://i.imgur.com/opt6ZNA.jpg)

下面是在保守策略下，volatile读操作插入内存屏障后生成的指令序列示意图：
![](https://i.imgur.com/9vw751f.jpg)

上述volatile写操作和volatile读操作的内存屏障插入策略非常保守。在实际执行时，只要不改变volatile写-读的内存语义，编译器可以根据具体情况省略不必要的屏障。双重检查锁实现单例中就需要用到这个特性！！！

## 二、AQS的实现原理
### AQS介绍
AQS，即AbstractQueuedSynchronizer, 队列同步器，它是Java并发用来构建锁和其他同步组件的基础框架。AQS是一个抽象类，主是是以继承的方式使用。AQS本身是没有实现任何同步接口的，它仅仅只是定义了同步状态的获取和释放的方法来供自定义的同步组件的使用。
AQS的实现是基于一个FIFO队列的，每一个等待的线程被封装成Node存放在等待队列中，头结点是空的，不存储信息，等待队列中的节点都是阻塞的，并且在每次被唤醒后都会检测自己的前一个节点是否为头结点，如果是头节点证明在这个线程之前没有在等待的线程，就尝试着去获取共享资源。

### AQS的继承关系
AQS继承了AbstractOwnableSynchronizer，我们先分析一下这个父类。父类非常简单，持有一个独占模式下的线程，然后就只剩下对这个线程的get和set方法。

### AQS的内部类
AQS最主要的三个成员变量
```
    private transient volatile Node head;
    
    private transient volatile Node tail;

    private volatile int state;
```
AQS是用链表队列来实现线程等待的，那么队列肯定要有节点。Node类，每一个等待的线程都会被封装成Node类
**Node的域**
* waitStatus：等待状态
* prev：前驱节点
* next：后继节点
* thread：持有的线程
* nextWaiter：condiction 队列中的后继节点

**Node的status**：
* CANCELLED，值为1，表示当前的线程被取消，被打断或者获取超时了
* SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是 unpark；
* CONDITION，值为-2，表示当前节点在等待condition，也就是在 condition 队列中；
* PROPAGATE，值为-3，表示当前场景下后续的acquireShared 能够得以执行；

取消状态的值是唯一的正数，也是唯一当排队排到它了也不要资源而是直接轮到下个线程来获取资源的

### 获取和释放同步状态的过程：

**获取同步状态**

假设线程A要获取同步状态（这里想象成锁，方便理解），初始状态下state=0,所以线程A可以顺利获取锁，A获取锁后将state置为1。在A没有释放锁期间，线程B也来获取锁，此时因为state=1，表示锁被占用，所以将B的线程信息和等待状态等信息构成出一个Node节点对象，放入同步队列，head和tail分别指向队列的头部和尾部（此时队列中有一个空的Node节点作为头点，head指向这个空节点，空Node的后继节点是B对应的Node节点，tail指向它），同时阻塞线程B(这里的阻塞使用的是LockSupport.park()方法)。后续如果再有线程要获取锁，都会加入队列尾部并阻塞。

首先获取AQS的同步状态(state)，在锁中就是锁的状态，如果状态为0，则尝试设置状态为arg(这里为1), 若设置成功则表示当前线程获取锁，返回true。这个操作外部方法lock()就做过一次，这里再做只是为了再尝试一次，尽量以最简单的方式获取锁。

如果状态不为0，再判断当前线程是否是锁的owner(即当前线程在之前已经获取锁，这里又来获取)，如果是owner, 则尝试将状态值增加acquires，如果这个状态值越界，抛出异常；如果没有越界，则设置后返回true。这里可以看非公平锁的涵义，即获取锁并不会严格根据争用锁的先后顺序决定。这里的实现逻辑类似synchroized关键字的偏向锁的做法，即可重入而不用进一步进行锁的竞争，也解释了ReentrantLock中Reentrant的意义。
如果状态不为0，且当前线程不是owner，则返回false。

**获取同步状态的过程**：
* 使用我们实现的tryAcquire来尝试获得锁，如果获得了锁则退出
* 如果没有获得锁就将线程添加到等待队列中
* 看到上面的tryAcquire返回false后就会调用addWaiter新建节点加入等待队列中。参数EXCLUSIVE是独占模式。
* 在addWaiter方法创建完节点后，调用enq方法，在循环中用CAS操作将新的节点入队。

因为可能会有多个线程同时设置尾节点，所以需要放在循环中不断的设置尾节点。在这里，节点入队就结束了。


**释放同步状态**

当线程A释放锁时，即将state置为0，此时A会唤醒头节点的后继节点（所谓唤醒，其实是调用LockSupport.unpark(B)方法），即B线程从LockSupport.park()方法返回，此时B发现state已经为0，所以B线程可以顺利获取锁，B获取锁后B的Node节点随之出队。

总结一下过程
![](https://i.imgur.com/x9E8E0o.jpg)

## 三、Java中的锁
### 1、锁的基础知识
如果想要透彻的理解java锁的来龙去脉，需要先了解以下基础知识。

#### 基础知识之一：锁的类型
锁从宏观上分类，分为悲观锁与乐观锁。

**乐观锁**

乐观锁是一种乐观思想，即认为读多写少，遇到并发写的可能性低，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，采取在写时先读出当前版本号，然后加锁操作（比较跟上一次的版本号，如果一样则更新），如果失败则要重复读-比较-写的操作。

java中的乐观锁基本都是通过CAS操作实现的，CAS是一种更新的原子操作，比较当前值跟传入值是否一样，一样则更新，否则失败。

**悲观锁**

悲观锁是就是悲观思想，即认为写多，遇到并发写的可能性高，每次去拿数据的时候都认为别人会修改，所以每次在读写数据的时候都会上锁，这样别人想读写这个数据就会block直到拿到锁。java中的悲观锁就是Synchronized,AQS框架下的锁则是先尝试cas乐观锁去获取锁，获取不到，才会转换为悲观锁，如RetreenLock。

#### 基础知识之二：java线程阻塞的代价
java的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就需要操作系统介入，需要在户态与核心态之间切换，这种切换会消耗大量的系统资源，因为用户态与内核态都有各自专用的内存空间，专用的寄存器等，用户态切换至内核态需要传递给许多变量、参数给内核，内核也需要保护好用户态在切换时的一些寄存器值、变量等，以便内核态调用结束后切换回用户态继续工作。

如果线程状态切换是一个高频操作时，这将会消耗很多CPU处理时间；
如果对于那些需要同步的简单的代码块，获取锁挂起操作消耗的时间比用户代码执行的时间还要长，这种同步策略显然非常糟糕的。
synchronized会导致争用不到锁的线程进入阻塞状态，所以说它是java语言中一个重量级的同步操纵，被称为重量级锁，为了缓解上述性能问题，JVM从1.5开始，引入了轻量锁与偏向锁，默认启用了自旋锁，他们都属于乐观锁。

明确java线程切换的代价，是理解java中各种锁的优缺点的基础之一。

#### 基础知识之三：markword
在介绍java锁之前，先说下什么是markword，markword是java对象数据结构中的一部分，要详细了解java对象的结构可以点击这里,这里只做markword的详细介绍，因为对象的markword和java各种类型的锁密切相关；

markword数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32bit和64bit，它的最后2bit是锁状态标志位，用来标记当前对象的状态，对象的所处的状态，决定了markword存储的内容，如下表所示:

| 状态 | 标志位 | 存储内容 |
| -------- | -------- | -------- |		
|未锁定|01	|对象哈希码、对象分代年龄|
|轻量级锁定|	00	|指向锁记录的指针|
|膨胀(重量级锁定)	|10|	执行重量级锁定的指针|
|GC标记 |11	|空(不需要记录信息)|
|可偏向 |01	|偏向线程ID、偏向时间戳、对象分代年龄|

32位虚拟机在不同状态下markword结构如下图所示：
![](https://i.imgur.com/OlCTsDH.png)


#### 小结
前面提到了java的4种锁，他们分别是重量级锁、自旋锁、轻量级锁和偏向锁， 
不同的锁有不同特点，每种锁只有在其特定的场景下，才会有出色的表现，java中没有哪种锁能够在所有情况下都能有出色的效率，引入这么多锁的原因就是为了应对不同的情况；

前面讲到了重量级锁是悲观锁的一种，自旋锁、轻量级锁与偏向锁属于乐观锁，所以现在你就能够大致理解了他们的适用范围，但是具体如何使用这几种锁呢，就要看后面的具体分析他们的特性；

### 2、自旋锁
自旋锁原理非常简单，如果持有锁的线程能在很短时间内释放锁资源，那么那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞挂起状态，它们只需要等一等（自旋），等持有锁的线程释放锁后即可立即获取锁，这样就避免用户线程和内核的切换的消耗。

但是线程自旋是需要消耗cup的，说白了就是让cup在做无用功，如果一直获取不到锁，那线程也不能一直占用cup自旋做无用功，所以需要设定一个自旋等待的最大时间。

如果持有锁的线程执行的时间超过自旋等待的最大时间扔没有释放锁，就会导致其它争用锁的线程在最大等待时间内还是获取不到锁，这时争用线程会停止自旋进入阻塞状态。

#### 自旋锁的优缺点
自旋锁尽可能的减少线程的阻塞，这对于锁的竞争不激烈，且占用锁时间非常短的代码块来说性能能大幅度的提升，因为自旋的消耗会小于线程阻塞挂起再唤醒的操作的消耗，这些操作会导致线程发生两次上下文切换！

但是如果锁的竞争激烈，或者持有锁的线程需要长时间占用锁执行同步块，这时候就不适合使用自旋锁了，因为自旋锁在获取锁前一直都是占用cpu做无用功，占着XX不XX，同时有大量线程在竞争一个锁，会导致获取锁的时间很长，线程自旋的消耗大于线程阻塞挂起操作的消耗，其它需要cup的线程又不能获取到cpu，造成cpu的浪费。所以这种情况下我们要关闭自旋锁；

#### 自旋锁时间阈值
自旋锁的目的是为了占着CPU的资源不释放，等到获取到锁立即进行处理。但是如何去选择自旋的执行时间呢？如果自旋执行时间太长，会有大量的线程处于自旋状态占用CPU资源，进而会影响整体系统的性能。因此自旋的周期选的额外重要！

JVM对于自旋周期的选择，jdk1.5这个限度是一定的写死的，在1.6引入了适应性自旋锁，适应性自旋锁意味着自旋的时间不在是固定的了，而是由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定，基本认为一个线程上下文切换的时间是最佳的一个时间，同时JVM还针对当前CPU的负荷情况做了较多的优化

如果平均负载小于CPUs则一直自旋
如果有超过(CPUs/2)个线程正在自旋，则后来线程直接阻塞
如果正在自旋的线程发现Owner发生了变化则延迟自旋时间（自旋计数）或进入阻塞
如果CPU处于节电模式则停止自旋

自旋时间的最坏情况是CPU的存储延迟（CPU A存储了一个数据，到CPU B得知这个数据直接的时间差）

自旋时会适当放弃线程优先级之间的差异

自旋锁的开启
JDK1.6中-XX:+UseSpinning开启； 
-XX:PreBlockSpin=10 为自旋次数； 
JDK1.7后，去掉此参数，由jvm控制；

### 3、重量级锁Synchronized
#### Synchronized的作用
它可以把任意一个非NULL的对象当作锁。
synchronized作用于一个对象实例时，锁住的是所有以该对象为锁的代码块。
指定加锁对象:对给定对象加锁，进入同步代码前需要获得给定对象的锁。
直接作用于实例方法:相当于对当前实例加锁，进入同步代码前要获得当前实例的锁。
直接作用于静态方法:相当于对当前类加锁，进入同步代码前要获得当前类的锁。
synchronized它的工作就是对需要同步的代码加锁，使得每一次只有一个线程可以进入同步块（其实是一种悲观策略）从而保证线程之间得安全性。
从这里我们可以知道，我们需要分析的属于第二类情况，也就是说多个线程如果同时进行set方法的时候，由于存在锁，所以会一个一个进行set操作，并且是线程安全的，但是get方法并没有加锁，表示假如A线程在进行set的同时B线程可以进行get操作。并且可以多个线程同时进行get操作，但是同一时间最多只能有一个set操作。
#### Synchronized的实现
实现如下图所示；
![](https://i.imgur.com/0HUdoP6.png)

它有多个队列，当多个线程一起访问某个对象监视器的时候，对象监视器会将这些线程存储在不同的容器中。
* Contention List：竞争队列，所有请求锁的线程首先被放在这个竞争队列中；
* Entry List：Contention List中那些有资格成为候选资源的线程被移动到Entry List中；
* Wait Set：哪些调用wait方法被阻塞的线程被放置在这里；
* OnDeck：任意时刻，最多只有一个线程正在竞争锁资源，该线程被成为OnDeck；
* Owner：当前已经获取到所资源的线程被称为Owner；
* !Owner：当前释放锁的线程。

JVM每次从队列的尾部取出一个数据用于锁竞争候选者（OnDeck），但是并发情况下，ContentionList会被大量的并发线程进行CAS访问，为了降低对尾部元素的竞争，JVM会将一部分线程移动到EntryList中作为候选竞争线程。Owner线程会在unlock时，将ContentionList中的部分线程迁移到EntryList中，并指定EntryList中的某个线程为OnDeck线程（一般是最先进去的那个线程）。Owner线程并不直接把锁传递给OnDeck线程，而是把锁竞争的权利交给OnDeck，OnDeck需要重新竞争锁。这样虽然牺牲了一些公平性，但是能极大的提升系统的吞吐量，在JVM中，也把这种选择行为称之为“竞争切换”。

OnDeck线程获取到锁资源后会变为Owner线程，而没有得到锁资源的仍然停留在EntryList中。如果Owner线程被wait方法阻塞，则转移到WaitSet队列中，直到某个时刻通过notify或者notifyAll唤醒，会重新进去EntryList中。

处于ContentionList、EntryList、WaitSet中的线程都处于阻塞状态，该阻塞是由操作系统来完成的（Linux内核下采用pthread_mutex_lock内核函数实现的）。

Synchronized是非公平锁。 Synchronized在线程进入ContentionList时，等待的线程会先尝试自旋获取锁，如果获取不到就进入ContentionList，这明显对于已经进入队列的线程是不公平的，还有一个不公平的事情就是自旋获取锁的线程还可能直接抢占OnDeck线程的锁资源。

#### synchronized的执行过程： 
* 检测Mark Word里面是不是当前线程的ID，如果是，表示当前线程处于偏向锁 
* 如果不是，则使用CAS将当前线程的ID替换Mard Word，如果成功则表示当前线程获得偏向锁，置偏向标志位1 
* 如果失败，则说明发生竞争，撤销偏向锁，进而升级为轻量级锁
* 当前线程使用CAS将对象头的Mark Word替换为锁记录指针，如果成功，当前线程获得锁 
* 如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁 
* 如果自旋成功则依然处于轻量级状态 
* 如果自旋失败，则升级为重量级锁

### 3、偏向锁
Java偏向锁(Biased Locking)是Java6引入的一项多线程优化。 
偏向锁，顾名思义，它会偏向于第一个访问锁的线程，如果在运行过程中，同步锁只有一个线程访问，不存在多线程争用的情况，则线程是不需要触发同步的，这种情况下，就会给线程加一个偏向锁。 
如果在运行过程中，遇到了其他线程抢占锁，则持有偏向锁的线程会被挂起，JVM会消除它身上的偏向锁，将锁恢复到标准的轻量级锁。

它通过消除资源无竞争情况下的同步原语，进一步提高了程序的运行性能。

#### 偏向锁的实现
**偏向锁获取过程**：
访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01，确认为可偏向状态。

如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤5，否则进入步骤3。

如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行5；如果竞争失败，执行4。

如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。（撤销偏向锁的时候会导致stop the word）

执行同步代码。

> 注意：第四步中到达安全点safepoint会导致stop the word，时间很短。

#### 偏向锁的释放：
偏向锁的撤销在上述第四步骤中有提到。偏向锁只有遇到其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁，线程不会主动去释放偏向锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，判断锁对象是否处于被锁定状态，撤销偏向锁后恢复到未锁定（标志位为“01”）或轻量级锁（标志位为“00”）的状态。

#### 偏向锁的适用场景
始终只有一个线程在执行同步块，在它没有执行完释放锁之前，没有其它线程去执行同步块，在锁无竞争的情况下使用，一旦有了竞争就升级为轻量级锁，升级为轻量级锁的时候需要撤销偏向锁，撤销偏向锁的时候会导致stop the word操作； 
在有锁的竞争时，偏向锁会多做很多额外操作，尤其是撤销偏向所的时候会导致进入安全点，安全点会导致stw，导致性能下降，这种情况下应当禁用；

查看停顿–安全点停顿日志
要查看安全点停顿，可以打开安全点日志，通过设置JVM参数 -XX:+PrintGCApplicationStoppedTime 会打出系统停止的时间，添加-XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1 这两个参数会打印出详细信息，可以查看到使用偏向锁导致的停顿，时间非常短暂，但是争用严重的情况下，停顿次数也会非常多；

注意：安全点日志不能一直打开： 
* 安全点日志默认输出到stdout，一是stdout日志的整洁性，二是stdout所重定向的文件如果不在/dev/shm，可能被锁。 
* 对于一些很短的停顿，比如取消偏向锁，打印的消耗比停顿本身还大。 
* 安全点日志是在安全点内打印的，本身加大了安全点的停顿时间。

所以安全日志应该只在问题排查时打开。 
如果在生产系统上要打开，再再增加下面四个参数：
```
-XX:+UnlockDiagnosticVMOptions 
-XX: -DisplayVMOutput -XX:+LogVMOutput 
-XX:LogFile=/dev/shm/vm.log
``` 
打开Diagnostic（只是开放了更多的flag可选，不会主动激活某个flag），关掉输出VM日志到stdout，输出到独立文件,/dev/shm目录（内存文件系统）。
![](https://i.imgur.com/7COGc0q.png)

此日志分三部分： 
* 第一部分是时间戳，VM Operation的类型 
* 第二部分是线程概况，被中括号括起来 
  > total: 安全点里的总线程数 
initially_running: 安全点开始时正在运行状态的线程数 
wait_to_block: 在VM Operation开始前需要等待其暂停的线程数
 
* 第三部分是到达安全点时的各个阶段以及执行操作所花的时间，其中最重要的是vmop
  > spin: 等待线程响应safepoint号召的时间；
block: 暂停所有线程所用的时间；
sync: 等于 spin+block，这是从开始到进入安全点所耗的时间，可用于判断进入安全点耗时；
cleanup: 清理所用时间；
vmop: 真正执行VM Operation的时间。

可见，那些很多但又很短的安全点，全都是RevokeBias， 高并发的应用会禁用掉偏向锁。

jvm开启/关闭偏向锁
```
开启偏向锁：-XX:+UseBiasedLocking 
-XX:BiasedLockingStartupDelay=0
关闭偏向锁：-XX:-UseBiasedLocking
```

### 4、 轻量级锁
轻量级锁是由偏向所升级来的，偏向锁运行在一个线程进入同步块的情况下，当第二个线程加入锁争用的时候，偏向锁就会升级为轻量级锁； 

#### 轻量级锁的加锁过程：
* 在代码进入同步块的时候，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，是否为偏向锁为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。这时候线程堆栈与对象头的状态如图： 
 ![](https://i.imgur.com/CR7U2CR.jpg)
* 拷贝对象头中的Mark Word复制到锁记录中；
* 拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，并将Lock record里的owner指针指向object mark word。如果更新成功，则执行步骤4，否则执行步骤5。
* 如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态，这时候线程堆栈与对象头的状态如图所示。 
 ![](https://i.imgur.com/0olV6rB.jpg)
* 如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。 而当前线程便尝试使用自旋来获取锁，自旋就是为了不让线程阻塞，而采用循环去获取锁的过程。

#### 轻量级锁的释放
释放锁线程视角：由轻量锁切换到重量锁，是发生在轻量锁释放锁的期间，之前在获取锁的时候它拷贝了锁对象头的markword，在释放锁的时候如果它发现在它持有锁的期间有其他线程来尝试获取锁了，并且该线程对markword做了修改，两者比对发现不一致，则切换到重量锁。

因为重量级锁被修改了，所有display mark word和原来的markword不一样了。

怎么补救，就是进入mutex前，compare一下obj的markword状态。确认该markword是否被其他线程持有。

此时如果线程已经释放了markword，那么通过CAS后就可以直接进入线程，无需进入mutex，就这个作用。

尝试获取锁线程视角：如果线程尝试获取锁的时候，轻量锁正被其他线程占有，那么它就会修改markword，修改重量级锁，表示该进入重量锁了。

还有一个注意点：等待轻量锁的线程不会阻塞，它会一直自旋等待锁，并如上所说修改markword。

这就是自旋锁，尝试获取锁的线程，在没有获得锁的时候，不被挂起，而转而去执行一个空循环，即自旋。在若干个自旋后，如果还没有获得锁，则才被挂起，获得锁，则执行代码。

### 5、总结
![](https://i.imgur.com/7jVyF04.jpg)

* 上面几种锁都是JVM自己内部实现，当我们执行synchronized同步块的时候jvm会根据启用的锁和当前线程的争用情况，决定如何执行同步操作；
* 在所有的锁都启用的情况下线程进入临界区时会先去获取偏向锁，如果已经存在偏向锁了，则会尝试获取轻量级锁，启用自旋锁，如果自旋也没有获取到锁，则使用重量级锁，没有获取到锁的线程阻塞挂起，直到持有锁的线程执行完同步块唤醒他们；
* 偏向锁是在无锁争用的情况下使用的，也就是同步开在当前线程没有执行完之前，没有其它线程会执行该同步块，一旦有了第二个线程的争用，偏向锁就会升级为轻量级锁，如果轻量级锁自旋到达阈值后，没有获取到锁，就会升级为重量级锁；
* 如果线程争用激烈，那么应该禁用偏向锁。

## 四、锁优化
以上介绍的锁不是我们代码中能够控制的，但是借鉴上面的思想，我们可以优化我们自己线程的加锁操作；

### 减少锁的时间
不需要同步执行的代码，能不放在同步快里面执行就不要放在同步快内，可以让锁尽快释放；

### 减少锁的粒度
它的思想是将物理上的一个锁，拆成逻辑上的多个锁，增加并行度，从而降低锁竞争。它的思想也是用空间来换时间；

java中很多数据结构都是采用这种方法提高并发操作的效率：

### ConcurrentHashMap
java中的ConcurrentHashMap在jdk1.8之前的版本，使用一个Segment 数组

`Segment< K,V >[] segments`

Segment继承自ReenTrantLock，所以每个Segment就是个可重入锁，每个Segment 有一个HashEntry< K,V >数组用来存放数据，put操作时，先确定往哪个Segment放数据，只需要锁定这个Segment，执行put，其它的Segment不会被锁定；所以数组中有多少个Segment就允许同一时刻多少个线程存放数据，这样增加了并发能力。

### LongAdder
LongAdder 实现思路也类似ConcurrentHashMap，LongAdder有一个根据当前并发状况动态改变的Cell数组，Cell对象里面有一个long类型的value用来存储值; 
开始没有并发争用的时候或者是cells数组正在初始化的时候，会使用cas来将值累加到成员变量的base上，在并发争用的情况下，LongAdder会初始化cells数组，在Cell数组中选定一个Cell加锁，数组有多少个cell，就允许同时有多少线程进行修改，最后将数组中每个Cell中的value相加，在加上base的值，就是最终的值；cell数组还能根据当前线程争用情况进行扩容，初始长度为2，每次扩容会增长一倍，直到扩容到大于等于cpu数量就不再扩容，这也就是为什么LongAdder比cas和AtomicInteger效率要高的原因，后面两者都是volatile+cas实现的，他们的竞争维度是1，LongAdder的竞争维度为“Cell个数+1”为什么要+1？因为它还有一个base，如果竞争不到锁还会尝试将数值加到base上；

### LinkedBlockingQueue
LinkedBlockingQueue也体现了这样的思想，在队列头入队，在队列尾出队，入队和出队使用不同的锁，相对于LinkedBlockingArray只有一个锁效率要高；

拆锁的粒度不能无限拆，最多可以将一个锁拆为当前cup数量个锁即可；

### 锁粗化
大部分情况下我们是要让锁的粒度最小化，锁的粗化则是要增大锁的粒度; 
在以下场景下需要粗化锁的粒度： 
假如有一个循环，循环内的操作需要加锁，我们应该把锁放到循环外面，否则每次进出循环，都进出一次临界区，效率是非常差的；

### 使用读写锁
ReentrantReadWriteLock 是一个读写锁，读操作加读锁，可以并发读，写操作使用写锁，只能单线程写；

### 读写分离
CopyOnWriteArrayList 、CopyOnWriteArraySet 
CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。 
　CopyOnWrite并发容器用于读多写少的并发场景，因为，读的时候没有锁，但是对其进行更改的时候是会加锁的，否则会导致多个线程同时复制出多个副本，各自修改各自的；

### 使用cas
如果需要同步的操作执行速度非常快，并且线程竞争并不激烈，这时候使用cas效率会更高，因为加锁会导致线程的上下文切换，如果上下文切换的耗时比同步操作本身更耗时，且线程对资源的竞争不激烈，使用volatiled+cas操作会是非常高效的选择；

### 消除缓存行的伪共享
除了我们在代码中使用的同步锁和jvm自己内置的同步锁外，还有一种隐藏的锁就是缓存行，它也被称为性能杀手。 
在多核cup的处理器中，每个cup都有自己独占的一级缓存、二级缓存，甚至还有一个共享的三级缓存，为了提高性能，cpu读写数据是以缓存行为最小单元读写的；32位的cpu缓存行为32字节，64位cup的缓存行为64字节，这就导致了一些问题。 
例如，多个不需要同步的变量因为存储在连续的32字节或64字节里面，当需要其中的一个变量时，就将它们作为一个缓存行一起加载到某个cup-1私有的缓存中（虽然只需要一个变量，但是cpu读取会以缓存行为最小单位，将其相邻的变量一起读入），被读入cpu缓存的变量相当于是对主内存变量的一个拷贝，也相当于变相的将在同一个缓存行中的几个变量加了一把锁，这个缓存行中任何一个变量发生了变化，当cup-2需要读取这个缓存行时，就需要先将cup-1中被改变了的整个缓存行更新回主存（即使其它变量没有更改），然后cup-2才能够读取，而cup-2可能需要更改这个缓存行的变量与cpu-1已经更改的缓存行中的变量是不一样的，所以这相当于给几个毫不相关的变量加了一把同步锁； 
为了防止伪共享，不同jdk版本实现方式是不一样的： 
* 在jdk1.7之前会 将需要独占缓存行的变量前后添加一组long类型的变量，依靠这些无意义的数组的填充做到一个变量自己独占一个缓存行； 
* 在jdk1.7因为jvm会将这些没有用到的变量优化掉，所以采用继承一个声明了好多long变量的类的方式来实现； 
* 在jdk1.8中通过添加sun.misc.Contended注解来解决这个问题，若要使该注解有效必须在jvm中添加以下参数：-XX:-RestrictContended

sun.misc.Contended注解会在变量前面添加128字节的padding将当前变量与其他变量进行隔离； 
关于什么是缓存行，jdk是如何避免缓存行的，网上有非常多的解释，在这里就不再深入讲解了；

## 五、Java里常用的并发集合
### 非阻塞式安全列表 - ConcurrentLinkedDeque
ConcurrentLinkedDeque可以在并发环境中直接使用，所谓的非阻塞，就是当列表为空的时候，我们还继续从列表中取数据的话，它会直接返回null或者抛出异常。下面列出来一些常用的方法。
* peekFirst() 、 peekLast() ：返回列表中首位跟末尾元素，如果列表为空则返回null。返回的元素不从列表中删除。
* getFirst() 、 getLast() ：返回列表中首位跟末尾元素，如果列表为空则抛出 NoSuchElementExceotion 异常。返回的元素不从列表中删除。
* removeFirst() 、 removeLast() ：返回列表中首位跟末尾元素，如果列表为空则抛出 NoSuchElementExceotion 异常。【返回的元素会从列表中删除】。

ConcurrentLinkedQueue使用链表作为其数据结构。 ConcurrentLinkedQueue应该算是在高并发环境中性能最好的队列了。 它之所有能有很好的性能，是因为其内部复杂的实现。
ConcurrentLinkedQueue 主要使用CAS非阻塞算法来实现线程安全。 适合在对性能要求相对较高，对个线程同时对队列进行读写的场景， 即如果对队列加锁的成本较高则适合使用无锁的ConcurrentLinkedQueue来替代。
### 阻塞式安全列表 - LinkedBlockingDeque
LinkedBlockingDeque是一个阻塞式的线程安全列表，它跟 ConcurrentLinkedDeque最大的区别就是，当列表中元素满了或者为空的时候，我们对该列表的操作不会立即返回，而是阻塞当前操作，直到该操作可以执行时才返回。我们对比着上面ConcurrentLinkedDeque的常用方法，来看下LinkedBlockingDeque会有哪些不一致的地方呢？
* put() ：插入元素至列表中，当表中元素已满的时候，该操作将会被阻塞，直到表中存在空余空间。
* take() : 从列表中获取元素，当列表为空，该操作会被阻塞，直到列表不为空。
* peekFirst() 、 peekLast() ：返回列表中首位跟末尾元素，如果列表为空则返回null。返回的元素不从列表中删除。
* getFirst() 、 getLast() ：返回列表中首位跟末尾元素，如果列表为空则抛出 NoSuchElementExceotion 异常。返回的元素不从列表中删除。
* addFirst() 、 addLast() ：将元素添加至首位跟末尾，如果列表已满，则会抛出 IllegalStateException

可以看出不管是从获取还是插入元素，都多了不少“花样”，其差别就在于是否阻塞，不满足条件是否返回null，不满足条件是否抛异常这几个方面来区分。

### 优先级排序阻塞式安全列表 - PriorityBlockingQueue
相信大家都写过把某个列表元素按照特定的规则来排序之类的代码，在PriorityBlockingQueue中，存放进去的元素必须要实现Comparable接口。在这个接口中，有一个compareTo()方法,当执行该方法的对象跟参数传入的对象进行比较的时候，这个方法会返回一个数字值，如果值小于0，则当前对象小于参数传入对象。大于0则相反，等于0就表示两个对象相等。

```
public class DemoObj implements Comparable<DemoObj> {
   
    private int priority;
    
    @Override
    public int compareTo(DemoObj do){
        if(this.getPriority() > do.getPriority()){
            return -1;
        }else if(this.getPriority() < do.getPriority()){
            return 1;
        }
        return 0;
    }
    
    //省略getset ...
}


//==== use ===================

PriorityBlockingQueue<DemoObj> queue = new PriorityBlockingQueue()<>;
queue.put(DemoObj);
queue.peek();
```
其常用方法跟上面提到的类基本都差不多大家可以动手实现一下，简单对比的话，可以说是LinkedBlockingDeque的增强版，多了元素排序功能。

### 延迟元素线程安全列表 - DelayQueue
DelayQueue里面存放着带有日期的元素，当我们从列表获取数据的时候，未到时间的元素将会被忽略。因此，存放进来的元素必须实现Delayed接口，使之成为一个延迟对象。
```
/**
 *  compareTo方法与getDelay方法需排序一致
 */
class Order implements Delayed{

    private String name ;
    private long start = System.currentTimeMillis();
    private long time ;

    public MyDelayedTask(String name,long time) {
        this.name = name;
        this.time = time;
    }

    /**
     * 需要实现的接口，获得延迟时间   用过期时间-当前时间
     * @param unit
     * @return
     */
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert((start+time) - System.currentTimeMillis(),TimeUnit.MILLISECONDS);
    }

    /**
     * 延迟队列内部排序   当前对象延迟时间 - 入参对象的延迟时间
     * @param o
     * @return
     */
    @Override
    public int compareTo(Delayed o) {
        Order o1 = (Order) o;
        return (int) (this.getDelay(TimeUnit.MILLISECONDS) - o1.getDelay(TimeUnit.MILLISECONDS));
    }
}
```
使用方式如下
```
private static DelayQueue delayQueue  = new DelayQueue();
    public static void main(String[] args) throws InterruptedException {

        new Thread(new Runnable() {
            @Override
            public void run() {

                delayQueue.offer(new Order("t3000",3000));
                delayQueue.offer(new Order("t4000",4000));
                delayQueue.offer(new Order("t2000",2000));
                delayQueue.offer(new Order("t6000",6000));
                delayQueue.offer(new Order("t1000",1000));

            }
        }).start();

        while (true) {
            Delayed take = delayQueue.take();
        }
    }
```
### 线程安全的哈希表 - ConcurrentHashMap
ConcurrentHashMap是由Segment数组结构和HashEntry数组结构组成。 其中Segment 实现了 ReentrantLock,所以Segment是一种可重入锁，扮演锁的角色。 HashEntry 用于存储键值对数据。
一个ConcurrentHashMap里包含一个Segment数组。 Segment结构和HashMap类似，是一种数组和链表结构， 一个Segment包含一个HashEntry数组，每个HashEntry是一个链表结构的元素， 每个Segment守护着一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须首先获得对应的Segment的锁。
JDK 1.8以后ConcurrentHashMap取消了Segment分段锁，采用CAS和synchronized来保证并发安全。 数据结构与HashMap1.8的结构类似，数组+链表/红黑二叉树(链表长度>8时，转换为红黑树)。synchronized只锁定当前链表或红黑二叉树的首节点，这样只要hash值不冲突，就不会产生并发。

#### ConcurrentHashMap和Hashtable的区别
**底层数据结构**：
* JDK1.7 的ConcurrentHashMap底层采用分段的数组+链表实现， JDK1.8 的ConcurrentHashMap底层采用的数据结构与JDK1.8 的HashMap的结构一样，数组+链表/红黑二叉树。
* Hashtable和JDK1.8 之前的HashMap的底层数据结构类似都是采用数组+链表的形式， 数组是 HashMap 的主体，链表则是主要为了解决哈希冲突而存在的

**实现线程安全的方式**
* JDK1.7的ConcurrentHashMap（分段锁）对整个桶数组进行了分割分段(Segment)， 每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。 JDK 1.8采用数组+链表/红黑二叉树的数据结构来实现，并发控制使用synchronized和CAS来操作。
* Hashtable:使用 synchronized 来保证线程安全，效率非常低下。 当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态， 如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈。
