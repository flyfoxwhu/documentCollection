# JVM
## 一、JVM内存模型
### 1、JVM内存模型的基本概念
 Java虚拟机规范中试图定义一种Java内存模型（Java Memory Model，JMM）来屏蔽掉各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的内存访问效果。在此之前，主流程序语言（如 C/C++ 等）直接使用物理硬件和操作系统的内存模型，因此，会由于不同平台上内存模型的差异，有可能导致程序在一套平台上并发完全正常，而在另外一套平台上并发访问却经常出错，因此在某些场景就必须针对不同的平台来编写程序。

定义 Java 内存模型并非一件容易的事情，这个模型必须定义得足够严谨，才能让 Java 的并发内存访问操作不会产生歧义；但是，也必须定义得足够宽松，使得虚拟机的实现有足够的自由空间去利用硬件的各种特性（寄存器、高速缓存和指令集中某些特有的指令）来获取更好的执行速度。经过长时间的验证和修补，在 JDK 1.5（实现了 JSR-133）发布后，Java 内存模型已经成熟和完善起来了。

### 2、主内存与工作内存
Java 内存模型的主要目标是定义程序中各个变量的访问规则，即在虚拟机中将变量存储到内存和从内存中取出变量这样的底层细节。此处的变量（Variables）与 Java 编程中所说的变量有所区别，它包括了实例字段、静态字段和构成数组对象的元素，但不包括局部变量与方法参数，因为后者是线程私有的，不会被共享，自然就不会存在竞争问题。为了获得较好的执行效能，Java 内存模型并没有限制执行引擎使用处理器的特定寄存器或缓存和主内存进行交互，也没有限制即时编译器进行调整代码执行顺序这类优化措施。

Java 内存模型规定了所有的变量都存储在主内存（Main Memory）中（此处的主内存与介绍物理硬件时的主内存名字一样，两者也可以互相类比，但此处仅是虚拟机内存的一部分）。每条线程还有自己的工作内存（Working Memory，可与前面讲的处理器高速缓存类比），线程的工作内存中保存了被该线程使用到的变量的主内存副本拷贝，线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，而不能直接读写主内存中的变量。不同的线程之间也无法直接访问对方工作内存中的变量，线程间变量值的传递均需要通过主内存来完成，线程、主内存、工作内存三者的交互关系如图所示。
![](https://i.imgur.com/yC4gPr1.png)
这里所讲的主内存、工作内存与前面所讲的 Java 内存区域的 Java 堆、栈、方法区等并不是同一个层次的内存划分，这两者基本上是没有关系的，如果两者一定要勉强对应起来，那从变量、主内存、工作内存的定义来看，主内存主要对应于 Java 堆中的对象实例数据部分，而工作内存则对应于虚拟机栈中的部分区域。从更低层次上说，主内存就直接对应于物理硬件的内存，而为了获取更高的运行速度，虚拟机（甚至是硬件系统本身的优化措施）可能会让工作内存优先存储于寄存器和高速缓存中，因为程序运行时主要访问读写的是工作内存。

### 3、内存间交互操作
关于主内存与工作内存之间具体的交互协议，即一个变量如何从主内存拷贝到工作内存、如何从工作内存同步会主内存之类的实现细节，Java 内存模型中定义了以下 8 种操作来完成，虚拟机实现时必须保证下面提及的每一种操作都是原子的、不可再分的（对于 double 和 long 类型的变量来说，load、store、read 和 write 操作在某些平台上允许有例外）。
* lock（锁定）：作用于主内存的变量，它把一个变量标识为一条线程独占的状态。
* unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定。
* read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的 load 动作使用。
* load（载入）：作用于工作内存的变量，它把 read 操作从主内存中得到的变量值放入工作内存的变量副本中。
* use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作。
* assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
* store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的 write 操作使用。
* write（写入）：作用于主内存的变量，它把 store 操作从工作内存中得到的变量的值放入主内存的变量中。

如果要把一个变量从主内存复制到工作内存，那就要顺序地执行 read 和 load 操作，如果要把变量从工作内存同步回主内存，就要顺序地执行 store 和 write 操作。注意，Java 内存模型只要求上述两个操作必须按顺序执行，而没有保证是连续执行。也就是说，read 与 load 之间、store 与 write 之间是可插入其他指令的，如对主内存中的变量 a、b 进行访问时，一种可能出现顺序是 read a、read b、load b、load a。除此之外，Java 内存模型还规定了在执行上述8种基本操作时必须满足如下规则：

* a.不允许 read 和 load、store 和 write 操作之一单独出现，即不允许一个变量从主内存读取了但工作内存不接受，或者从工作内存发起回写了但主内存不接受的情况出现。
* b.不允许一个线程丢弃它的最近的 assign 操作，即变量在工作内存中改变了之后必须把该变化同步会主内存。
* c.不允许一个线程无原因地（没有发生过任何 assign 操作）把数据从线程的工作内存同步回主内存中。
* d.一个新的变量只能在主内存中 “诞生”，不允许在工作内存中直接使用一个未被初始化（load 或 assign）的变量，换句话说，就是对一个变量实施 use、store 操作之前，必须先执行过了 assign 和 load 操作。
* e.一个变量在同一个时刻只允许一条线程对其进行 lock 操作，但 lock 操作可以被同一条线程重复执行多次，多次执行 lock 后，只有执行相同次数的 unlock 操作，变量才会被解锁。
* f.如果对一个变量执行 lock 操作，那将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行 load 或 assign 操纵初始化变量的值。
* g.如果一个变量事先没有被 lock 操作锁定，那就不允许对它执行 unlock 操作，也不允许去 unlock 一个被其他线程锁定住的变量。
* h.对一个变量执行 unlock 操作之前，必须先把此变量同步回主内存中（执行 store、write 操作）。

这8种内存访问操作以及上述规则限定，再加上稍后介绍的对volatile的一些特殊规定，就已经完全确定了Java程序中哪些内存访问操作在并发下是安全的。由于这种定义相当严谨但又十分烦琐，实践起来很麻烦，所以在后面笔者将介绍这种定义的一个等效判断原则——先行发生原则，用来确定一个访问在并发环境下是否安全。

### 4、对于volatile型变量的特殊规则
关键字volatile可以说是Java虚拟机提供的最轻量级的同步机制。

当一个变量定义为volatile之后，它将具备两种特性，第一是保证此变量对所有线程的可见性，这里的 “可见性” 是指当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。而普通变量不能做到这一点，普通变量的值在线程间传递均需要通过主内存来完成，例如，线程A修改一个普通变量的值，然后向主内存进行回写，另外一条线程B在线程A回写完成了之后再从主内存进行读取操作，新变量值才会对线程B可见。

关于volatile变量的可见性，经常会被开发人员误解，认为以下描述成立：“volatile变量对所有线程是立即可见的，对volatile变量所有的写操作都能立刻反应到其他线程之中，换句话说，volatile变量在各个线程中是一致的，所以基于volatile变量的运算在并发下是安全的”。这句话的论据部分并没有错，但是其论据并不能得出 “基于volatile变量的运算在并发下是安全的” 这个结论。volatile变量在各个线程的工作内存中不存在一致性问题（在各个线程的工作内存中，volatile变量也可以存在不一致的情况，但由于每次使用之前都要先刷新，执行引擎看不到不一致的情况，因此可以认为不存在不一致性问题），但是Java里面的运算并非原子操作，导致volatile变量的运算在并发下一样是不安全的。

由于volatile变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然需要通过加锁（使用synchronized或 java.util.concurrent中的原子类）来保证原子性。
* a.运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。
* b.变量不需要与其他状态变量共同参与不变约束。

使用volatile变量的第二个语义是禁止指令重排序优化，普通的变量仅仅会保证在该方法的执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致。因为在一个线程的方法执行过程中无法感知到这点，这也就是 Java 内存模型中描述的所谓的 “线程内表现为串行的语义”（Within-Thread As-If-Serial Semantics）。

volatile的内存屏障策略非常严格保守，非常悲观且毫无安全感的心态：在每个volatile写操作前插入StoreStore屏障，在写操作后插入StoreLoad屏障；在每个volatile读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障；由于内存屏障的作用，避免了volatile变量和其它指令重排序、线程之间实现了通信，使得volatile表现出了锁的特性。

那为何说它禁止指令重排序呢？从硬件架构上讲，指令重排序是指 CPU采用了允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理。但并不是说指令任意重排，CPU需要能正确处理指令依赖情况以保障程序能得出正确的执行结果。譬如指令1把地址A中的值加10，指令2把地址A中的值乘以2，指令3把地址B中的值减去3，这时指令1和指令2是有依赖的，它们之间的顺序不能重排——（A + 10）2与A 2 + 10显然不相等，但指令3可以重排到指令 1、2 之前或者中间，只要保证CPU执行后面依赖到A、B值的操作是能获取到正确的A和B值即可。所以在本内CPU中，重排序看起来依然是有序的。因此lock addl $0x0, (%esp) 指令把修改同步到内存时，意味着所有之前的操作都已经执行完成，这样便形成了“指令重排序无法越过内存屏障” 的效果。

假定T表示一个线程，V和W分别表示两个volatile型变量，那么在进行read、load、use、assign、store和write操作时需要满足如下规则：
* a.只有当线程T对变量V执行的前一个动作是load的时候，线程 T 才能对变量V执行use动作；并且，只有当线程T对变量V执行的后一个动作是use的时候，线程T才能对变量V执行load动作。线程 T 对变量V的use动作可以认为是和线程T对变量V的load、read动作相关联，必须连续一起出现（这套规则要求在工作内存中，每次使用 V前都必须先从主内存刷新最新的值，用于保证能看见其他线程对变量V所做的修改后的值）。
* b只有当线程T对变量的前一个动作是assign的时候，线程T才能对变量V执行store动作；并且，只有当线程T对变量V执行的后一个动作是store的时候，线程T才能对变量V执行assign动作。线程T对变量V的assign动作可以认为是和线程T对变量V的store、write动作相关联，必须连续一起出现（这条规则要求在工作内存中，每次修改V后都必须立刻同步回主内存中，用于保证其他线程可以看到自己对变量V所做的修改）。
* c.假定动作A是线程T对变量V实施的use或assign动作，假定动作F是和动作A相关联的load或store动作，假定动作P是和动作F相应的对变量V的read或write动作；类似的，假定动作B 是线程 T 对变量W实施的use或assign动作，假定动作G 是和动作B相关联的load或store动作，假定动作Q是和动作G相应的对变量W的read或write动作。如果A先于B，那么P先于Q（这条规则要求volatile修饰的变量不会被指令重排序优化，保证代码的执行顺序与程序的顺序相同）。

### 5、对于long和double型变量的特殊规则
Java 内存模型要求 lock、unlock、read、assign、use、store、write 这 8 个操作都具有原子性，但是对于 64 位的数据类型（long 和 double），在模型中特别定义了一条相对宽松的规定：允许虚拟机将没有被 volatile 修饰的 64 位数据的读写操作划分为两次 32 位的操作来进行，即允许虚拟机实现选择可以不保证 64 位数据类型的 load、store、read 和write这 4 个操作的原子性，这点就是所谓的 long 和 double 的非原子性协定（Nonatomic Treatment of double and long Variables）。

 如果有多个线程共享一个并未声明为volatile的long或double类型的变量，并且同时对它们进行读取和修改操作，那么某些线程可能会读取到一个既非原值，也不是其他线程修改的值的代表了 “半个变量” 的数值。
  不过这种读取到 “半个变量” 的情况非常罕见（在目前商用Java虚拟机中不会出现），因为Java内存模型虽然允许虚拟机不把long和double变量的读写实现成原子操作，但允许虚拟机选择把这些操作实现为具有原子性的操作，而且还 “强烈建议” 虚拟机这样实现。在实际开发中，目前各种平台下的商用虚拟机几乎都选择把64位的数据的读写操作作为原子操作来对待，因此我们在编写代码时一般不需要把用到的long和double变量专门声明为volatile。

## 二、JVM性能调优
### 1、性能定义
* 吞吐量 - 指不考虑 GC 引起的停顿时间或内存消耗，垃圾收集器能支撑应用达到的最高性能指标。
* 延迟 - 其度量标准是缩短由于垃圾啊收集引起的停顿时间或者完全消除因垃圾收集所引起的停顿，避免应用运行时发生抖动。
* 内存占用 - 垃圾收集器流畅运行所需要的内存数量。

### 2、调优原则
**GC优化的两个目标**：
* 将进入老年代的对象数量降到最低
* 减少Full GC的执行时间

**GC优化的基本原则**：将不同的GC参数应用到两个及以上的服务器上然后比较它们的性能，然后将那些被证明可以提高性能或减少GC 执行时间的参数应用于最终的工作服务器上。

#### （1）将进入老年代的对象数量降到最低
除了可以在JDK7及更高版本中使用的G1收集器以外，其他分代GC都是由Oracle JVM提供的。关于分代GC，就是对象在Eden区被创建，随后被转移到Survivor区，在此之后剩余的对象会被转入老年代。也有一些对象由于占用内存过大，在Eden区被创建后会直接被传入老年代。老年代GC相对来说会比新生代GC更耗时，因此，减少进入老年代的对象数量可以显著降低Full GC的频率。你可能会以为减少进入老年代的对象数量意味着把它们留在新生代，事实正好相反，新生代内存的大小是可以调节的。
#### （2）降低Full GC的时间
Full GC的执行时间比Minor GC要长很多，因此，如果在Full GC上花费过多的时间（超过 1s），将可能出现超时错误。如果通过减小老年代内存来减少Full GC时间，可能会引起 OutOfMemoryError或者导致Full GC的频率升高。
另外，如果通过增加老年代内存来降低Full GC的频率，Full GC的时间可能因此增加。
因此，你需要把老年代的大小设置成一个“合适”的值。

### 3、GC优化需要考虑的JVM参数
类型参数描述堆内存大小
* -Xms启动JVM 时堆内存的大小
* -Xmx堆内存最大限制新生代空间大小
* -XX:NewRatio新生代和老年代的内存比
* -XX:NewSize新生代内存大小
* -XX:SurvivorRatioEden区和Survivor区的内存比

GC优化时最常用的参数是-Xms,-Xmx和-XX:NewRatio。-Xms和-Xmx参数通常是必须的，所以NewRatio的值将对 GC 性能产生重要的影响。
有些人可能会问如何设置永久代内存大小，你可以用-XX:PermSize和-XX:MaxPermSize参数来进行设置，但是要记住，只有当出现OutOfMemoryError错误时你才需要去设置永久代内存。

### 4、GC优化的过程
GC 优化的过程和大多数常见的提升性能的过程相似，下面是笔者使用的流程：
#### （1）、监控GC状态
你需要监控GC从而检查系统中运行的GC的各种状态。
#### （2）、分析监控结果后决定是否需要优化GC
在检查GC状态后，你需要分析监控结构并决定是否需要进行GC优化。如果分析结果显示运行GC的时间只有0.1-0.3秒，那么就不需要把时间浪费在GC优化上，但如果运行GC的时间达到1-3秒，甚至大于10秒，那么GC优化将是很有必要的。但是，如果你已经分配了大约10GB内存给Java，并且这些内存无法省下，那么就无法进行GC优化了。在进行GC优化之前，你需要考虑为什么你需要分配这么大的内存空间，如果你分配了1GB或2GB大小的内存并且出现了OutOfMemoryError，那你就应该执行堆快照heap dump来消除导致异常的原因。
注意：**堆快照（heap dump）** 是一个用来检查Java内存中的对象和数据的内存文件。该文件可以通过执行JDK中的jmap命令来创建。在创建文件的过程中，所有Java程序都将暂停，因此，不要在系统执行过程中创建该文件。

#### （3）、设置GC类型/内存大小
如果你决定要进行GC优化，那么你需要选择一个GC类型并且为它设置内存大小。此时如果你有多个服务器，请如上文提到的那样，在每台机器上设置不同的GC参数并分析它们的区别。
#### （4）、分析结果
在设置完GC参数后就可以开始收集数据，请在收集至少24小时后再进行结果分析。如果你足够幸运，你可能会找到系统的最佳GC参数。如若不然，你还需要分析输出日志并检查分配的内存，然后需要通过不断调整GC类型/内存大小来找到系统的最佳参数。
#### （5）、将参数应用到所有服务器上
如果GC优化的结果令人满意，就可以将参数应用到所有服务器上，并停止GC优化。

## 二、常用JDK命令
### 1、jmap
jmap 即JVM Memory Map。
jmap 用于生成 heap dump 文件。
如果不使用这个命令，还可以使用 -XX:+HeapDumpOnOutOfMemoryError 参数来让虚拟机出现 OOM 的时候，自动生成 dump 文件。
jmap 不仅能生成 dump 文件，还可以查询 finalize 执行队列、Java 堆和永久代的详细信息，如当前使用率、当前使用的是哪种收集器等。
命令格式：
jmap [option] LVMID
option 参数：
* dump - 生成堆转储快照
* finalizerinfo - 显示在 F-Queue 队列等待 Finalizer 线程执行 finalizer 方法的对象
* heap - 显示 Java 堆详细信息
* histo - 显示堆中对象的统计信息
* permstat - to print permanent generation statistics
* F - 当-dump 没有响应时，强制生成 dump 快照
示例：jmap -dump PID 生成堆快照

dump 堆到文件，format 指定输出格式，live 指明是活着的对象，file 指定文件名
```
$ jmap -dump:live,format=b,file=dump.hprof 28920
Dumping heap to /home/xxx/dump.hprof ...
Heap dump file created
```
dump.hprof 这个后缀是为了后续可以直接用 MAT(Memory Anlysis Tool)打开。
示例：jmap -heap 查看指定进程的堆信息
注意：使用 CMS GC 情况下，jmap -heap 的执行有可能会导致 java 进程挂起。
```
jmap -heap PID
[root@chances bin]# ./jmap -heap 12379
Attaching to process ID 12379, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 17.0-b16

using thread-local object allocation.
Parallel GC with 6 thread(s)

Heap Configuration:
   MinHeapFreeRatio = 40
   MaxHeapFreeRatio = 70
   MaxHeapSize      = 83886080 (80.0MB)
   NewSize          = 1310720 (1.25MB)
   MaxNewSize       = 17592186044415 MB
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2
   SurvivorRatio    = 8
   PermSize         = 20971520 (20.0MB)
   MaxPermSize      = 88080384 (84.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 9306112 (8.875MB)
   used     = 5375360 (5.1263427734375MB)
   free     = 3930752 (3.7486572265625MB)
   57.761608714788736% used
From Space:
   capacity = 9306112 (8.875MB)
   used     = 3425240 (3.2665634155273438MB)
   free     = 5880872 (5.608436584472656MB)
   36.80634834397007% used
To Space:
   capacity = 9306112 (8.875MB)
   used     = 0 (0.0MB)
   free     = 9306112 (8.875MB)
   0.0% used
PS Old Generation
   capacity = 55967744 (53.375MB)
   used     = 48354640 (46.11457824707031MB)
   free     = 7613104 (7.2604217529296875MB)
   86.39733629427693% used
PS Perm Generation
   capacity = 62062592 (59.1875MB)
   used     = 60243112 (57.452308654785156MB)
   free     = 1819480 (1.7351913452148438MB)
   97.06831451706046% used
```
### 2、jstack
jstack用于生成java虚拟机当前时刻的线程快照。
线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。
线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得 core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。另外，jstack 工具还可以附属到正在运行的 java程序中，看到当时运行的java程序的java stack和native stack的信息, 如果现在运行的java程序呈现hung的状态，jstack是非常有用的。
命令格式：
jstack [option] LVMID
option 参数：
* -F - 当正常输出请求不被响应时，强制输出线程堆栈
* -l - 除堆栈外，显示关于锁的附加信息
* -m - 如果调用到本地方法的话，可以显示 C/C++的堆栈

### 3、jps
jps(JVM Process Status Tool)，显示指定系统内所有的 HotSpot 虚拟机进程。
命令格式：
jps [options] [hostid]
option 参数：
* -l - 输出主类全名或jar路径
* -q - 只输出LVMID
* -m - 输出JVM启动时传递给main()的参数
* -v - 输出JVM启动时显示指定的JVM参数

其中[option]、[hostid]参数也可以不写。
```
$ jps -l -m
28920 org.apache.catalina.startup.Bootstrap start
11589 org.apache.catalina.startup.Bootstrap start
25816 sun.tools.jps.Jps -l -m
```

### 4、jstat
jstat(JVM statistics Monitoring)，是用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT 编译等运行数据。
命令格式：
jstat [option] LVMID [interval] [count]
参数：
* [option] - 操作参数
* LVMID - 本地虚拟机进程 ID
* [interval] - 连续输出的时间间隔
* [count] - 连续输出的次数

### 5、jhat
jhat(JVM Heap Analysis Tool)，是与 jmap 搭配使用，用来分析jmap生成的dump，jhat内置了一个微型的HTTP/HTML服务器，生成dump的分析结果后，可以在浏览器中查看。
注意：一般不会直接在服务器上进行分析，因为jhat是一个耗时并且耗费硬件资源的过程，一般把服务器生成的dump文件复制到本地或其他机器上进行分析。
命令格式：
jhat [dumpfile]

### 6、jinfo
jinfo(JVM Configuration info)，用于实时查看和调整虚拟机运行参数。
之前的jps -v口令只能查看到显示指定的参数，如果想要查看未被显示指定的参数的值就要使用jinfo口令
命令格式：
jinfo [option] [args] LVMID
**option参数：**
* -flag: 输出指定args参数的值
* -flags: 不需要args参数，输出所有JVM参数的值
* -sysprops:输出系统属性，等同于System.getProperties()

## 三、HotSpot VM参数
详细参数说明请参考官方文档：Java HotSpot VM Options，这里仅列举常用参数。
### 1、JVM内存配置
配置描述
* -Xms堆空间初始值
* -Xmx堆空间最大值
* -XX:NewSize新生代空间初始值
* -XX:MaxNewSize新生代空间最大值
* -Xmn新生代空间大小
* -XX:PermSize永久代空间的初始值
* -XX:MaxPermSize永久代空间的最大值。

JVM 中最大堆大小有三方面限制：
* 相关操作系统的数据模型（32-bt还是64-bit）限制；
* 系统的可用虚拟内存限制；
* 系统的可用物理内存限制。

整个堆大小=年轻代大小 + 年老代大小 + 持久代大小

### 2、GC类型配置
配置描述
* -XX:+UseSerialGC串行垃圾回收器
* -XX:+UseParallelGC并行垃圾回收器
* -XX:+UseParNewGC使用ParNew + Serial Old 垃圾回收器组合
* -XX:+UseConcMarkSweepGC并发标记扫描垃圾回收器
* -XX:ParallelCMSThreads=并发标记扫描垃圾回收器 = 为使用的线程数量
* -XX:+UseG1GCG1 垃圾回收器

### 3、辅助配置
配置描述
* -XX:+PrintGCDetails 打印GC日志
* -Xloggc:<filename> 指定GC日志文件名
* -XX:+HeapDumpOnOutOfMemoryError内存溢出时输出堆快照文件


## 四、GC日志
### 1、获取GC日志有两种方式：
* 使用命令动态查看
* 在容器中设置相关参数打印GC日志

jstat -gc 统计垃圾回收堆的行为：
```
jstat -gc 1262
 S0C    S1C     S0U     S1U   EC       EU        OC         OU        PC       PU         YGC    YGCT    FGC    FGCT     GCT
26112.0 24064.0 6562.5  0.0   564224.0 76274.5   434176.0   388518.3  524288.0 42724.7    320    6.417   1      0.398    6.815
```
也可以设置间隔固定时间来打印：
`$ jstat -gc 1262 2000 20`
这个命令意思就是每隔 2000ms 输出 1262 的 gc 情况，一共输出 20 次
Tomcat 设置示例：
```
JAVA_OPTS="-server -Xms2000m -Xmx2000m -Xmn800m -XX:PermSize=64m -XX:MaxPermSize=256m -XX:SurvivorRatio=4
-verbose:gc -Xloggc:$CATALINA_HOME/logs/gc.log
-Djava.awt.headless=true
-XX:+PrintGCTimeStamps -XX:+PrintGCDetails
-Dsun.rmi.dgc.server.gcInterval=600000 -Dsun.rmi.dgc.client.gcInterval=600000
-XX:+UseConcMarkSweepGC -XX:MaxTenuringThreshold=15"
-Xms2000m -Xmx2000m -Xmn800m -XX:PermSize=64m -XX:MaxPermSize=256m
``` 
* Xms即为jvm启动时得 JVM 初始堆大小,
* Xmx为jvm的最大堆大小，
* xmn为新生代的大小，
* permsize为永久代的初始大小，
* MaxPermSize为永久代的最大空间。
* -XX:SurvivorRatio=4 SurvivorRatio为新生代空间中的 Eden区和救助空间Survivor区的大小比值，默认是8，则两个Survivor区与一个Eden区的比值为2:8，一个Survivor区占整个年轻代的1/10。调小这个参数将增大survivor区，让对象尽量在survito区呆长一点，减少进入年老代的对象。去掉救助空间的想法是让大部分不能马上回收的数据尽快进入年老代，加快年老代的回收频率，减少年老代暴涨的可能性，这个是通过将-XX:SurvivorRatio设置成比较大的值（比如 65536)来做到。
* -verbose:gc -Xloggc:$CATALINA_HOME/logs/gc.log 将虚拟机每次垃圾回收的信息写到日志文件中，文件名由 file 指定，文件格式是平文件，内容和-verbose:gc 输出内容相同。
* -Djava.awt.headless=true Headless 模式是系统的一种配置模式。在该模式下，系统缺少了显示设备、键盘或鼠标。
* -XX:+PrintGCTimeStamps -XX:+PrintGCDetails设置gc日志的格式
* -Dsun.rmi.dgc.server.gcInterval=600000指定rmi调用时 gc的时间间隔
* -XX:+UseConcMarkSweepGC -XX:MaxTenuringThreshold=15采用并发gc方式，经过 15 次 minor gc后进入年老代

### 2、GC日志解读
![](https://i.imgur.com/VLbhxpw.jpg)

### 3、分析 GC 日志
**Young GC 回收日志:**
```
2016-07-05T10:43:18.093+0800: 25.395: [GC [PSYoungGen: 274931K->10738K(274944K)] 371093K->147186K(450048K), 0.0668480 secs] [Times: user=0.17 sys=0.08, real=0.07 secs]
```

**Full GC 回收日志:**
```
2016-07-05T10:43:18.160+0800: 25.462: [Full GC [PSYoungGen: 10738K->0K(274944K)] [ParOldGen: 136447K->140379K(302592K)] 147186K->140379K(577536K) [PSPermGen: 85411K->85376K(171008K)], 0.6763541 secs] [Times: user=1.75 sys=0.02, real=0.68 secs]
```

通过上面日志分析得出，PSYoungGen、ParOldGen、PSPermGen 属于Parallel收集器。其中PSYoungGen表示gc回收前后年轻代的内存变化；ParOldGen表示gc回收前后老年代的内存变化；PSPermGen表示gc回收前后永久区的内存变化。young gc主要是针对年轻代进行内存回收比较频繁，耗时短；full gc会对整个堆内存进行回城，耗时长，因此一般尽量减少full gc的次数

## 五、垃圾收集器
垃圾收集器就是内存回收操作的具体实现，HotSpot 里足足有 7 种，为啥要弄这么多，因为它们各有各的适用场景。有的属于新生代收集器，有的属于老年代收集器，所以一般是搭配使用的（除了万能的 G1）。关于它们的简单介绍以及分类请见下图。
![](https://i.imgur.com/mmY8EMq.png)

### Serial / ParNew 搭配 Serial Old 收集器
![](https://i.imgur.com/RpT3L9a.png)

Serial 收集器是虚拟机在 Client 模式下的默认新生代收集器，它的优势是简单高效，在单 CPU 模式下很牛。

ParNew 收集器就是 Serial 收集器的多线程版本，虽然除此之外没什么创新之处，但它却是许多运行在 Server 模式下的虚拟机中的首选新生代收集器，因为除了 Serial 收集器外，只有它能和 CMS 收集器搭配使用。

### Parallel 搭配 Parallel Scavenge 收集器
首先，这俩货肯定是要搭配使用的，不仅仅如此，它俩还贼特别，它们的关注点与其他收集器不同，其他收集器关注于尽可能缩短垃圾收集时用户线程的停顿时间，而 Parallel Scavenge 收集器的目的是达到一个可控的吞吐量。

> 吞吐量 = 运行用户代码时间 / ( 运行用户代码时间 + 垃圾收集时间 )

因此，Parallel Scavenge 收集器不管是新生代还是老年代都是多个线程同时进行垃圾收集，十分适合于应用在注重吞吐量以及 CPU 资源敏感的场合。
可调节的虚拟机参数：
* -XX:MaxGCPauseMillis：最大 GC 停顿的秒数；
* -XX:GCTimeRatio：吞吐量大小，一个 0 ~ 100 的数，最大 GC 时间占总时间的比率 = 1 / (GCTimeRatio + 1)；
* -XX:+UseAdaptiveSizePolicy：一个开关参数，打开后就无需手工指定 -Xmn，-XX:SurvivorRatio 等参数了，虚拟机会根据当前系统的运行情况收集性能监控信息，自行调整。

### CMS 收集器
![](https://i.imgur.com/wKsEJAs.png)
![](https://i.imgur.com/ZoocAkx.png)

**参数设置**：
* -XX:+UseCMSCompactAtFullCollection：在 CMS 要进行 Full GC 时进行内存碎片整理（默认开启）
* -XX:CMSFullGCsBeforeCompaction：在多少次 Full GC 后进行一次空间整理（默认是 0，即每一次 Full GC 后都进行一次空间整理）

**关于CMS使用标记-清除算法的一点思考**：
之前对于CMS为什么要采用 标记-清除算法十分的不理解，既然已经有了看起来更高级的 标记-整理算法，那CMS为什么不用呢？最近想了想，感觉可能是这个原因，不过也不是很确定，只是个人的一种猜测。

标记-整理会将所有存活对象向一端移动，然后直接清理掉边界以外的内存。这就意味着需要一个指针来维护这个分隔存活对象和无用空间的点，而我们知道 CMS 是并发清理的，虽然我们启动了多个线程进行垃圾回收，不过如果使用 标记 - 整理 算法，为了保证线程安全，在整理时要对那个分隔指针加锁，保证同一时刻只有一个线程能修改它，加锁的这一过程相当于将并行的清理过程变成了串行的，也就失去了并行清理的意义了。
所以，CMS采用了标记-清除算法。

### G1 收集器
![](https://i.imgur.com/XaHfQoA.png)
![](https://i.imgur.com/NMtmti9.png)


## 六、常见问题分析
### 1、OutOfMemory(OOM)分析
OutOfMemory，即内存溢出，是一个常见的JVM问题。那么分析 OOM的思路是什么呢？
OutOfMemoryError的三种类型：
* OutOfMemoryError:PermGen space - 方法区和运行时常量池溢出
* OutOfMemoryError:Java heap space - 堆空间溢出
* OutOfMemoryError: GC overhead limit exceeded - GC为释放很小空间占用大量时间
* OutOfMemoryError:unable to create new native thread - 线程过多

#### （1）OutOfMemoryError:PermGen space
OutOfMemoryError:PermGen space 表示方法区和运行时常量池溢出
**原因**： Perm区主要用于存放Class和Meta信息的，Class在被Loader时就会被放到 PermGen space，这个区域称为年老代。GC在主程序运行期间不会对年老区进行清理，默认是64M大小。
当程序程序中使用了大量的 jar 或 class，使java虚拟机装载类的空间不够，超过64M就会报这部分内存溢出了，需要加大内存分配，一般28m足够。

**解决方案**：
* 扩大永久代空间
JDK7 以前使用 -XX:PermSize 和 -XX:MaxPermSize 来控制永久代大小。JDK8 以后把原本放在永久代的字符串常量池移出, 放在 Java 堆中(元空间 Metaspace)中，元数据并不在虚拟机中，使用的是本地的内存。使用 -XX:MetaspaceSize 和 -XX:MaxMetaspaceSize 控制元空间大小。
注意：-XX:PermSize 一般设为 64M
* 清理应用程序中 WEB-INF/lib 下的 jar，用不上的 jar 删除掉，多个应用公共的 jar 移动到 Tomcat 的 lib 目录，减少重复加载。

#### （2）OutOfMemoryError:Java heap space
OutOfMemoryError:Java heap space 表示堆空间溢出。
**原因**：JVM分配给堆内存的空间已经用满了。

**问题定位**：
* 使用jmap或-XX:+HeapDumpOnOutOfMemoryError获取堆快照。 
* 使用内存分析工具visualvm、mat、jProfile等对堆快照文件进行分析。 
* 根据分析图，重点是确认内存中的对象是否是必要的，分清究竟是是内存泄漏（Memory Leak）还是内存溢出（Memory Overflow）

#### （3）OutOfMemoryError: GC overhead limit exceeded
**原因**：JDK6新增错误类型，当GC为释放很小空间占用大量时间时抛出；一般是因为堆太小，导致异常的原因，没有足够的内存。

**解决方案**：查看系统是否有使用大内存的代码或死循环； 通过添加JVM配置，来限制使用内存：

```
-XX:-UseGCOverheadLimit
```

#### （4）OutOfMemoryError：unable to create new native thread
**原因：** 线程过多
那么能创建多少线程呢？这里有一个公式：
```
(MaxProcessMemory - JVMMemory - ReservedOsMemory) / (ThreadStackSize) = Number of threads

MaxProcessMemory 指的是一个进程的最大内存  
JVMMemory         JVM内存  
ReservedOsMemory  保留的操作系统内存  
ThreadStackSize      线程栈的大小
```

当发起一个线程的创建时，虚拟机会在JVM内存创建一个Thread对象同时创建一个操作系统线程，而这个系统线程的内存用的不是 JVMMemory，而是系统中剩下的内存：(MaxProcessMemory - JVMMemory - ReservedOsMemory) 结论：你给JVM内存越多，那么你能用来创建的系统线程的内存就会越少，越容易发生 java.lang.OutOfMemoryError: unable to create new native thread。

### 2、内存泄露
####（1）内存泄漏的定义
内存泄漏是指由于疏忽或错误造成程序未能释放已经不再使用的内存的情况。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，失去了对该段内存的控制，因而造成了内存的浪费。内存泄漏随着被执行的次数越多-最终会导致内存溢出。
而因程序死循环导致的不断创建对象-只要被执行到就会产生内存溢出。

#### （2）内存泄漏常见几个情况：
* 静态集合类：声明为静态（static）的HashMap、Vector 等集合。通俗来讲A中有B，当前只把B设置为空，A没有设置为空，回收时B无法回收-因被A引用。
* 监听器：监听器被注册后释放对象时没有删除监听器
* 物理连接：DataSource.getConnection()建立链接，必须通过 close()关闭链接
* 内部类和外部模块等的引用：发现它的方式同内存溢出，可再加个实时观察jstat -gcutil 7362 2500 70

#### （3）重点关注：
* FGC — 从应用程序启动到采样时发生Full GC的次数。
* FGCT — 从应用程序启动到采样时Full GC所用的时间（单位秒）。
* FGC 次数越多，FGCT 所需时间越多-可非常有可能存在内存泄漏。
 
#### （4）解决方案
**方案一**：检查程序，看是否有死循环或不必要地重复创建大量对象。有则改之。
下面是一个重复创建内存的示例：
```
public class OOM {
    public static void main(String[] args) {
        Integer sum1=300000;
        Integer sum2=400000;
        OOM oom = new OOM();
        System.out.println("往ArrayList中加入30w内容");
        oom.javaHeapSpace(sum1);
        oom.memoryTotal();
        System.out.println("往ArrayList中加入40w内容");
        oom.javaHeapSpace(sum2);
        oom.memoryTotal();
    }
    public void javaHeapSpace(Integer sum){
        Random random = new Random();  
        ArrayList openList = new ArrayList();
        int i = 0;
        while(i++!=sum;){
            String charOrNum = String.valueOf(random.nextInt(10));
            openList.add(charOrNum);
        }  
    }
    public void memoryTotal(){
        Runtime run = Runtime.getRuntime();
        long max = run.maxMemory();
        long total = run.totalMemory();
        long free = run.freeMemory();
        long usable = max - total + free;
        System.out.println("最大内存 = " + max);
        System.out.println("已分配内存 = " + total);
        System.out.println("已分配内存中的剩余空间 = " + free);
        System.out.println("最大可用内存 = " + usable);
    }
}
```

执行结果：
```
往ArrayList中加入30w内容
最大内存 = 20447232
已分配内存 = 20447232
已分配内存中的剩余空间 = 4032576
最大可用内存 = 4032576
往ArrayList中加入40w内容

Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:2245)
    at java.util.Arrays.copyOf(Arrays.java:2219)
    at java.util.ArrayList.grow(ArrayList.java:242)
    at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:216)
    at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:208)
    at java.util.ArrayList.add(ArrayList.java:440)
    at pers.qingqian.study.seven.OOM.javaHeapSpace(OOM.java:36)
    at pers.qingqian.study.seven.OOM.main(OOM.java:26)
```
**方案二**：扩大堆内存空间，使用-Xms 和-Xmx来控制堆内存空间大小。



### 3、CPU 过高
**定位步骤：**
（1）执行 top -c命令，找到cpu最高的进程的id
（2）jstack PID 导出Java应用程序的线程堆栈信息。

示例：

```
jstack 6795

"Low Memory Detector" daemon prio=10 tid=0x081465f8 nid=0x7 runnable [0x00000000..0x00000000]  
        "CompilerThread0" daemon prio=10 tid=0x08143c58 nid=0x6 waiting on condition [0x00000000..0xfb5fd798]
        "Signal Dispatcher" daemon prio=10 tid=0x08142f08 nid=0x5 waiting on condition [0x00000000..0x00000000]
        "Finalizer" daemon prio=10 tid=0x08137ca0 nid=0x4 in Object.wait() [0xfbeed000..0xfbeeddb8]

        at java.lang.Object.wait(Native Method)  

        - waiting on <0xef600848> (a java.lang.ref.ReferenceQueue$Lock)  

        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:116)  

        - locked <0xef600848> (a java.lang.ref.ReferenceQueue$Lock)  

        at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:132)  

        at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:159)  

        "Reference Handler" daemon prio=10 tid=0x081370f0 nid=0x3 in Object.wait() [0xfbf4a000..0xfbf4aa38]  

        at java.lang.Object.wait(Native Method)  

        - waiting on <0xef600758> (a java.lang.ref.Reference$Lock)  

        at java.lang.Object.wait(Object.java:474)  

        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:116)  

        - locked <0xef600758> (a java.lang.ref.Reference$Lock)  

        "VM Thread" prio=10 tid=0x08134878 nid=0x2 runnable  

        "VM Periodic Task Thread" prio=10 tid=0x08147768 nid=0x8 waiting on condition
```
在打印的堆栈日志文件中，tid和nid的含义：
* nid : 对应的 Linux 操作系统下的tid线程号，也就是前面转化的16进制数字
* tid: 这个应该是jvm的jmm内存规范中的唯一地址定位
* 在CPU过高的情况下，查找响应的线程，一般定位都是用nid来定位的。而如果发生死锁之类的问题，一般用tid来定位。

（3）定位CPU高的线程打印其nid
查看线程下具体进程信息的命令如下：
```
top -H -p 6735
top - 14:20:09 up 611 days,  2:56,  1 user,  load average: 13.19, 7.76, 7.82
Threads: 6991 total,  17 running, 6974 sleeping,   0 stopped,   0 zombie
%Cpu(s): 90.4 us,  2.1 sy,  0.0 ni,  7.0 id,  0.0 wa,  0.0 hi,  0.4 si,  0.0 st
KiB Mem:  32783044 total, 32505008 used,   278036 free,   120304 buffers
KiB Swap:        0 total,        0 used,        0 free.  4497428 cached Mem

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
 6800 root      20   0 27.299g 0.021t   7172 S 54.7 70.1 187:55.61 java
 6803 root      20   0 27.299g 0.021t   7172 S 54.4 70.1 187:52.59 java
 6798 root      20   0 27.299g 0.021t   7172 S 53.7 70.1 187:55.08 java
 6801 root      20   0 27.299g 0.021t   7172 S 53.7 70.1 187:55.25 java
 6797 root      20   0 27.299g 0.021t   7172 S 53.1 70.1 187:52.78 java
 6804 root      20   0 27.299g 0.021t   7172 S 53.1 70.1 187:55.76 java
 6802 root      20   0 27.299g 0.021t   7172 S 52.1 70.1 187:54.79 java
 6799 root      20   0 27.299g 0.021t   7172 S 51.8 70.1 187:53.36 java
 6807 root      20   0 27.299g 0.021t   7172 S 13.6 70.1  48:58.60 java
11014 root      20   0 27.299g 0.021t   7172 R  8.4 70.1   8:00.32 java
10642 root      20   0 27.299g 0.021t   7172 R  6.5 70.1   6:32.06 java
 6808 root      20   0 27.299g 0.021t   7172 S  6.1 70.1 159:08.40 java
11315 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   5:54.10 java
12545 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   6:55.48 java
23353 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   2:20.55 java
24868 root      20   0 27.299g 0.021t   7172 S  3.9 70.1   2:12.46 java
 9146 root      20   0 27.299g 0.021t   7172 S  3.6 70.1   7:42.72 java
```
由此可以看出占用CPU较高的线程，但是这些还不高，无法直接定位到具体的类。nid是16进制的，所以我们要获取线程的16进制ID：
`printf "%x\n" 6800`
输出结果:45cd
（4）然后根据nid到jstack打印的堆栈日志中查定位：
```
"catalina-exec-5692" daemon prio=10 tid=0x00007f3b05013800 nid=0x45cd waiting on condition [0x00007f3ae08e3000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000006a7800598> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:226)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2082)
        at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
        at org.apache.tomcat.util.threads.TaskQueue.poll(TaskQueue.java:86)
        at org.apache.tomcat.util.threads.TaskQueue.poll(TaskQueue.java:32)
        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1068)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1130)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
        at java.lang.Thread.run(Thread.java:745)
```