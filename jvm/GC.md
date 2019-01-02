#### GC垃圾搜集器总结
##### GC的目的：
1. 分配内存：垃圾回收算法的设计往往制约了内存分配的方式；
2. 确保存活对象不会被回收
3. 回收垃圾对象

##### GC经验法则

1. 对于大部分对象来说，它会在“年轻”的时候死去；
2. 很少引用存在于“年老的”和“年轻的”对象之间；

##### GC设计因素
1. 串行（Serial）和并行（Parallel）(多个垃圾搜集器一起工作)
2. 并发（Concurrent）和暂停式（Stop the world） （GC线程和应用线程同时运行）
3. 压缩（Compacting）和非压缩（Non-Compacting）（存不存在碎片的问题）

##### 度量
* 暂停时间（pause time）：是指在一次垃圾回收中，Stop-the-world状态下占用的时间。暂停时间主要受到算法和堆大小的影响。相同条件下，堆越小，暂停时间就越短。但是堆越小，那么回收频率就越高。
* 吞吐量（throughput）：一般而言，堆越大，吞吐量越高，回收频率越低。

1. Serial Collector
* 年轻代搜集
标记阶段：从根（root）出发，沿着引用链标记存活对象；
复制：将存活对象复制到特定的区域；
* 老年代收集集
标记：比较存活的对象
清扫：在该阶段，会识别出垃圾对象。“清扫”这个概念带有一些误导的色彩，在算法的实现中，并不需要真的对垃圾对象占据的内存进行清扫，仅仅标记一下就够了。（注：在新对象被分配到该被清扫的区域的时候，会执行一次JVM层面上的初始化过程，该过程会把该内存区域重置，因而有了Java语言中的不同数据类型的默认值一说）
整理：将所有的存活对象都挪到一端
* 应用场景
Serial Collector主要适用于对pause time要求不高，可用内存和CPU数量都比较小的应用上。在新的HotSpot中，如果虚拟机运行的平台是client-style类型的，那么就会采用Serial Collector。

>因此在分配内存空间的时候都可以使用bump-the-pointer的技术。无论是年轻代还是老年代，都是单线程的，也是是暂停应用进程，单个GC运行。

2. Parallel Collector
Parallel Collector是 Serial Collector的多线程版本。Parallel Collector只有对年轻代的回收（minor collection）将会采用多线程，对老年代的回收还是采用单线程的形式。
>Paraller Collector采用了TLABs（thread-local allocation buffers）它将老年代划分成一个个固定大小的Buffer，给每一个回收线程分配一个，回收线程就只在自己的Buffer内分配空间.但是这会带来另外一个问题，就是每一个线程并不能恰好用完一个Buffer，可能出现的情况是，一个线程检测到Buffer里面的空闲空间已经不够了，于是只能申请另外一个Buffer。而原来的Buffer的那部分不足的空间就会被浪费掉。这就是所谓的float garbage。
* 应用场景
Parallel Collector又被叫做Throughput Collector。所以很显然，它适用于要求吞吐量高的场景。

3. CMS Collector
* 在CMS Collector里，年轻代的回收是和Parallel Collector一样的，也就是说，年轻代的回收是stop-the-world式的
* 只有在老年代，相应的major collection里面才会使用CMS算法。

分成以下步骤：
* initial mark：initial mark是stop-the-world式的，也就是说在这个阶段是需要暂停应用的执行。initial mark只是识别出来标记的根。（Reference from thread stacks, Reference from young ）
* concurrent mark：并发标记阶段。在该阶段，应用可以继续运行；
* remark：在concurrent mark阶段，因为应用依旧在运行，所以可能原本标记为垃圾的对象又“复活”了，也可能分配了新的对象。所以会引入找一个remark阶段。该阶段也是stop-the-world的；
* concurrent sweep：并发清扫。该阶段应用会继续运行；（card table简单理解是并发回收器的工作列表。CMS使用该技术会在concurrent mark阶段，将改变了引用关系的对象标记为“dirty”，在remark阶段中重新扫描）

>1. 在concurrent mark阶段，可能触发minor collection. 2. 因为在回收阶段还有可能分配对象，所以垃圾回收不能等内存满了才开始，必须要提前开始.这就会造成一个问题，就是在垃圾回收阶段，空闲空间不足了. 3. 该回收的垃圾没有被回收。这也被称为floating garbage。这主要是出现在，原本一个对象被标记为存活，但是在concurrent阶段，应用修改了指向该对象的引用，使得它称为了垃圾。但是CMS Collector无法将其检测出来。因此它能够躲过这一轮的垃圾回收，直到下一次的回收周期；4. CMS Collector不是compacting的，这意味着垃圾回收之后得到的空闲空间并不是连续的。CMS采用了新的分配方式：空闲链表分配方式，该概念和操作系统中内存管理中的空闲链表是一样的；空闲空间不连续会导致空间有效利用率下降,更频繁的触发GC。

* 应用场景
适用于要求pause time尽可能短，并且拥有多个CPU的应用。CMS Collector的别名是Latency Collector。


4. G1 Collector

G1 Collector（Garbage-First Collector）可以被看做是CMS Collector的升级加强版。G1 Collector的算法流程和CMS类似，所不同的有：
1. G1 Collector采用的是标记-整理算法。这意味着每次算法结束得到的都是连续空间；
2. G1 Collector虽然还采用分代的方式，但是它的内存模型有了巨大的变化。它的内存基本结构被分成了一个个Region。G1 Collector维护了一个Region的列表，每次判断哪个Region的回收价值最大，便回收该Region。也就是说，G1 Collector回收并不是回收整个区域，而是分区域收集的；
3. G1 Collector有一个很重要的特性，就是“软实时”。G1 Collector可以让使用者指定在一个长度为M毫秒的时间段内，消耗在垃圾收集上的时间不得超过N毫秒。这已经是期望达到一种类似于实时垃圾回收的效果了。
4. 其具体流程是：
initial marking phase：标记根，该阶段是stop-the-world式的；
root region scanning phase：该阶段标记Survivor Region中的指向老年代的引用，及其引用对象；
Concurrent marking phase：
Remark phase：
Cleanup phase：


* Remember Set和Card Table
RS(Remember Set)是一种抽象概念，用于记录从非收集部分指向收集部分的指针的集合。Card Table是RS的一种实现。主要用来记录并发标记过程中，引用发生发生改变的Region。这样在Remark的时候只需要扫描CardTable就可以。
* Remember Set的写屏障
写屏障是指，在改变特定内存的值（实际上也就是写入内存）的时候额外执行的一些动作。写屏障通常用于在运行时探测并记录回收相关指针，例如在运行过程中，当老年代的引用指向年轻代，那么都会被写屏障捕获，并且记录下来。因此在年轻代回收的时候，就可以避免扫描整个老年代来查找根。实际是一小代码块。
* Collect Set
Collect Set(CSet)是指，在Evacuation阶段，由G1垃圾回收器选择的待回收的Region集合。G1垃圾回收器的软实时的特性就是通过CSet的选择来实现的。对应于算法的两种模式fully-young generational mode和partially-young mode
1. 在fully-young generational mode下：顾名思义，该模式下CSet将只包含young的Region。G1将调整young的Region的数量来匹配软实时的目标；
2. 在partially-young mode下：该模式会选择所有的young region，并且选择一部分的old region。old region的选择将依据在Marking cycle phase中对存活对象的计数。G1选择存活对象最少的Region进行回收。
* SATB(snapshot-at-the-beginning)G1垃圾回收器使用该技术在标记阶段记录一个存活对象的快照
* Marking bitmaps和TAMS
G1使用了两个bitmap，一个叫做previous bitmap，另外一个叫做next bitmap。previous bitmap记录的是上一次的标记阶段完成之后的构造的bitmap；next bitmap则是当前正在标记阶段正在构造的bitmap。在当前标记阶段结束之后，当前标记的next bitmap就变成了下一次标记阶段的previous bitmap。
TAMS(top at mark start)变量，是一对用于区分在标记阶段新分配对象的变量，分别被称为previous TAMS和next TAMS。在previous TAMS和next TAMS之间的对象则是本次标记阶段时候新分配的对象

##### G1算法步骤：
整个算法可以分成两大部分：
* Marking cycle phase：标记阶段，该阶段是不断循环进行的；
* Evacuation phase：该阶段是负责把一部分region的活对象拷贝到空Region里面去，然后回收原本的Region空间，该阶段是STW(stop-the-world)的；
而算法也可以分成两种模式：
* fully-young generational mode：有时候也会被称为young GC，该模式只会回收young region，算法是通过调整young region的数量来达到软实时目标的；
* partially-young mode：也被称为Mixed GC，该阶段会回收young region和old region，算法通过调整old region的数量来达到软实时目标；

##### Marking Cycle Phase
算法的Marking cycle phase大概可以分成五个阶段：

1. Initial marking phase：G1收集器扫描所有的根。该过程是和young GC的暂停过程一起的；
Root 
2. region scanning phase：扫描Survivor Regions中指向老年代的被initial mark phase标记的引用及引用的对象，这一个过程是并发进行的。但是该过程要在下一个young GC开始之前结束；
3. Concurrent marking phase：并发标记阶段，标记整个堆的存活对象。该过程可以被young GC所打断。并发阶段产生的新的引用（或者引用的更新）会被SATB的write barrier记录下来；
4. Remark phase：也叫final marking phase。该阶段只需要扫描SATB(Snapshot At The Beginning)的buffer，处理在并发阶段产生的新的存活对象的引用。作为对比，CMS的remark需要扫描整个mod union table的标记为dirty的entry以及全部根；
5. Cleanup phase：清理阶段。该阶段会计算每一个region里面存活的对象，并把完全没有存活对象的Region直接放到空闲列表中。在该阶段还会重置Remember Set。该阶段在计算Region中存活对象的时候，是STW(Stop-the-world)的，而在重置Remember Set的时候，却是可以并行的；


##### Evacuation
Evacuation阶段STW的，大概可以分成两个步骤：第一个步骤是从Region中选出若干个Region进行回收，这些被选中的Region称为Collect Set（简称CSet）；而第二个步骤则是把这些Region中存活的对象复制到空闲的Region中去，同时把这些已经被回收的Region放到空闲Region列表中。

Evacuation的触发时机在不同的模式下会有一些不同。在不同的模式下都相同的是，只要堆的使用率达到了某个阈值，就必然会触发Evacuation。这是为了确保在Evacuation的时候有足够的空闲Region来容纳存活对象。

在young GC的情况下，G1会选择N个region作为CSet，该CSet首先需要满足软实时的要求，而一旦已经有N个region已经被分配了，那么就会执行一次Evacuation。

G1会尽可能的执行mixed GC。唯一的限制就是mix GC也需要满足软实时的要求。