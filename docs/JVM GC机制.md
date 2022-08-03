
# 判断哪些对象需要被回收（死亡）
## 引用计数算法：
给对象中添加一个引用计数器，每当有一个地方引用时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的，即垃圾回收器可以回收的.。 但是JVM没有使用此方法，因为此方法无法解决2个对象相互循环引用的问题。

优点:引用计数法实现简单，但效率不高,大多数情况下都是个不错的算法,例如微软的COM(component Object Model)技术, 使用ActionScript3的FlashPlayer,Python语言和游戏脚本领域被广泛应用的Squirrel中都用了引用计数算法进行内存管理.

缺点:很难解决对象之间的相互循环引用的问题

例如如下代码:

```
public class Test {
	public Object instance = null;
	private static final int _1MB = 1024*1024;
	/**
	 * 此成员作用,占用内存,以便能在gc日志中看清是否被回收过
	 */
	private byte[] bigSize = new byte[2*_1MB];
	public static void testGC() {
		Test obj1 = new Test();
		Test obj2 = new Test();
		obj1.instance = obj2;
		obj2.instance = obj1;
		obj1 = null;
		obj2 = null;
		
		System.gc();
	}
}
```

实际上这2个对象已经不可能被访问到, 但他们互相引用对方,导致引用计数器值都不为0;于是这种算法就不能回收了

## 可达性分析算法：
在Java,C#,等语言中,都是通过可达性分析算法来判断对象是否存活。大多数主流的JVM都采用这样的算法来管理内存，它能够解决对象之间的循环引用的问题。对象与对象之间虽然有循环引用，当他们到GC Roots没有任何引用链，系统还是判定它们为可回收对象。

这个算法的基本思路就是通过一系列称为`"GC Roots"`的对象作为起始点,从这些节点开始向下搜索, 搜索走过的路径称为引用链,当一个对象到GC Roots没有任何引用链时,则证明此对象已经死亡(不可用).如下:
![1](https://i.loli.net/2020/07/07/2ABJPSjQCzRFbXH.jpg)
黄色为仍然存活的对象; 白色为判定可回收的对象

在Java语言中，可作为**GC Roots的对象**包括下面几种：

* 虚拟机栈（栈帧中的本地变量表）中引用的对象。
* 方法区中类静态属性引用的对象。
* 方法区中常量引用的对象。
* 本地方法栈中JNI（即一般说的Native方法）引用的对象。

# 垃圾收集算法

## 标记-清除算法（Mark-Sweep）
此方法分为“标记”和“清除”两个阶段：首先标记出所有不需要回收的对象，在标记完成后统一回收所有未被标记的对象。它是最基础的收集算法，后续的收集算法都是基于这种思路并对其不足进行改进而得到的。

主要两个不足：一个是效率问题，标记和清除两个过程的效率都不高(有两次扫描的，耗时严重；)；另一个是空间问题，标记清除之后会产生大量不连续内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

标记-清除算法标记的是可达的对象，回收的是未被标记的对象。清除阶段过后，所有存活的已标记的对象标记复原成未标记状态。

先从根对象开始标记存活对象（在对象的header打上标记），然后对堆内存进行一次从头到尾的线性遍历，没有标记的为不可达对象则回收

![1](https://i.loli.net/2020/07/07/K69qbFGSpwBXEIO.jpg)

## 复制算法（Copying）
&emsp;此方法将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。但是可用内存变成原来的一半，代价较大。
&emsp;此方法一般用在回收新生代，因为新生代的对象98%都是很快就会被回收，所以不用1:1划分，而是分为一块较大的Eden空间和2块较小的Survivor空间。每次使用Eden和其中一块Survivor。当回收时，将Eden和Survivor中还存活着的对象一次性地复制到另外一块Survivor空间上，最后清理掉Eden和刚才用过的Survivor空间。HotSpot虚拟机默认Eden和Survivor的大小比例是8:1:1，即新生代中可用内存为90%，只有10%被浪费。
&emsp;缺点：如果对象存活率高，要执行较多的复制操作，效率将会变低。

![2](https://i.loli.net/2020/07/07/Y8HqAceR39tdhbv.jpg)

## 标记-整理算法（Mark-Compact）
复制收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低。更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接选用这种算法。
根据老年代的特点，有人提出了另外一种“标记-整理”（Mark-Compact）算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

![3](https://i.loli.net/2020/07/07/JsiNZHGY4fvCDAX.jpg)

## 分代收集算法（Generational Collection）
当前商业虚拟机的垃圾收集都采用“分代收集”算法，这种算法并没有什么新的思想，只是根据对象存活周期的不同将内存划分为几块。一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。
在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记—清理”或者“标记—整理”算法来进行回收。



# 垃圾收集器

如果说收集算法是内存回收的方法论，垃圾收集器就是内存回收的具体实现

## 1.Serial 收集器

串行收集器是最古老，最稳定以及效率高的收集器
可能会产生较长的停顿，只使用一个线程去回收h-XX:+UseSerialGC

* 新生代、老年代使用串行回收
* 新生代复制算法
* 老年代标记-压缩

![1](https://i.loli.net/2020/07/31/IYoG3eh8yp6tDgM.jpg)

## 2. 并行收集器

### 2.1 ParNew

-XX:+UseParNewGC（new代表新生代，所以适用于新生代）

* 新生代并行
* 老年代串行

Serial收集器新生代的并行版本，在新生代回收时使用复制算法，多线程，需要多核支持

-XX:ParallelGCThreads 限制线程数量

![2](https://i.loli.net/2020/07/31/HXaprVTPe6iNCqG.jpg)

### 2.2 Parallel收集器

类似ParNew ，新生代复制算法，老年代标记-压缩，更加关注吞吐量 

* 使用Parallel收集器+ **老年代串行**
  -XX:+UseParallelGC  

* 使用Parallel收集器+ **老年代并行**
  -XX:+UseParallelOldGC 

![3](https://i.loli.net/2020/07/31/vx7EZ1heDuqcjIp.jpg)

### 2.3 其他GC参数

* -XX:MaxGCPauseMills
  - 最大停顿时间，单位毫秒
  - GC尽力保证回收时间不超过设定值

* -XX:GCTimeRatio 
  - 0-100的取值范围
  - 垃圾收集时间占总时间的比
  - 默认99，即最大允许1%时间做GC

这两个参数是矛盾的。因为停顿时间和吞吐量不可能同时调优

## 3. CMS收集器

* Concurrent Mark Sweep 并发标记清除（应用程序线程和GC线程交替执行）
* 使用标记-清除算法
* 并发阶段会降低吞吐量（停顿时间减少，吞吐量降低）
* 老年代收集器（新生代使用ParNew）
* -XX:+UseConcMarkSweepGC

CMS运行过程比较复杂，**着重实现了标记的过程**，可分为

* 1. 初始标记（会产生全局停顿）

  - 根可以直接关联到的对象
  - 速度快

* 2. 并发标记（和用户线程一起） 

  - 主要标记过程，标记全部对象

* 3. 重新标记 （会产生全局停顿） 

  - 由于并发标记时，用户线程依然运行，因此在正式清理前，再做修正

* 4. 并发清除（和用户线程一起） 

  - 基于标记结果，直接清理对象

![4](https://i.loli.net/2020/07/31/P8kn2jKuiOFpX4V.jpg)

这里就能很明显的看出，为什么CMS要使用标记清除而不是标记压缩，如果使用标记压缩，需要对对象的内存位置进行改变，这样程序就很难继续执行。但是标记清除会产生大量内存碎片，不利于内存分配。 

**CMS收集器特点：**

* 尽可能降低停顿
* 会影响系统整体吞吐量和性能
  - 比如，在用户线程运行过程中，分一半CPU去做GC，系统性能在GC阶段，反应速度就下降一半
* 清理不彻底 
  - 因为在清理阶段，用户线程还在运行，会产生新的垃圾，无法清理
* 因为和用户线程一起运行，不能在空间快满时再清理（因为也许在并发GC的期间，用户线程又申请了大量内存，导致内存不够） 
  - -XX:CMSInitiatingOccupancyFraction设置触发GC的阈值
  - 如果不幸内存预留空间不够，就会引起concurrent mode failure
    ，一旦 concurrent mode failure产生，将使用串行收集器作为后备。

**CMS也提供了整理碎片的参数：**

* -XX:+ UseCMSCompactAtFullCollection 

Full GC后，进行一次整理，整理过程是独占的，会引起停顿时间变长

* -XX:+CMSFullGCsBeforeCompaction  

设置进行几次Full GC后，进行一次碎片整理

* -XX:ParallelCMSThreads 

设定CMS的线程数量（一般情况约等于可用CPU数量）

CMS的提出是想改善GC的停顿时间，在GC过程中的确做到了减少GC时间，但是同样导致产生大量内存碎片，又需要消耗大量时间去整理碎片，从本质上并没有改善时间。

## 4. G1收集器

G1是目前技术发展的最前沿成果之一，HotSpot开发团队赋予它的使命是未来可以替换掉JDK1.5中发布的CMS收集器。

与CMS收集器相比G1收集器有以下特点：

(1) 空间整合，G1收集器采用标记整理算法，不会产生内存空间碎片。分配大对象时不会因为无法找到连续空间而提前触发下一次GC。

(2)可预测停顿，这是G1的另一大优势，降低停顿时间是G1和CMS的共同关注点，但G1除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为N毫秒的时间片段内，消耗在垃圾收集上的时间不得超过N毫秒，这几乎已经是实时Java（RTSJ）的垃圾收集器的特征了。

上面提到的垃圾收集器，收集的范围都是整个新生代或者老年代，而G1不再是这样。使用G1收集器时，Java堆的内存布局与其他收集器有很大差别，它将整个Java堆划分为多个大小相等的独立区域（Region），虽然还保留有新生代和老年代的概念，但新生代和老年代不再是物理隔阂了，它们都是一部分（可以不连续）Region的集合。==

G1的新生代收集跟ParNew类似，当新生代占用达到一定比例的时候，开始并行收集。

和CMS类似，G1收集器收集老年代对象会有短暂停顿。

**步骤**

(1)标记阶段，首先初始标记(Initial-Mark),这个阶段是停顿的(Stop the World Event)，并且会触发一次普通Mintor GC。对应GC log:GC pause (young) (inital-mark)

(2)Root Region Scanning，程序运行过程中会回收survivor区(存活到老年代)，这一过程必须在young GC之前完成。

(3)Concurrent Marking，在整个堆中进行并发标记(和应用程序并发执行)，此过程可能被young GC中断。在并发标记阶段，若发现区域对象中的所有对象都是垃圾，那个这个区域会被立即回收(图中打X)。同时，并发标记过程中，会计算每个区域的对象活性(区域中存活对象的比例)。

![5](https://i.loli.net/2020/07/31/Bpb4IMsxzCVu2GJ.jpg)

(4)Remark, 再标记，会有短暂停顿(STW)。再标记阶段是用来收集 并发标记阶段 产生新的垃圾(并发阶段和应用程序一同运行)；G1中采用了比CMS更快的初始快照算法:snapshot-at-the-beginning (SATB)。

(5)Copy/Clean up，多线程清除失活对象，会有STW。G1将回收区域的存活对象拷贝到新区域，清除Remember Sets，并发清空回收区域并把它返回到空闲区域链表中。

![6](https://i.loli.net/2020/07/31/CaKJji2w7FrvIzW.jpg)

(6)复制/清除过程后。回收区域的活性对象已经被集中回收到深蓝色和深绿色区域。

# 触发FULL GC的几种情况

GC又分为 minor GC 和 Full GC (也称为 Major GC )

Minor GC触发条件：

* 当Eden区满时，触发Minor GC。

Full GC触发情况：

  * a.调用System.gc时，系统建议执行Full GC，但是不必然执行
  * b.老年代空间不足
  * c.方法区空间不足
  * d.通过Minor GC后进入老年代的平均大小大于老年代的可用内存
  * e.由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小


## System.gc()方法的调用

此方法的调用是建议JVM进行Full GC,虽然只是建议而非一定,但很多情况下它会触发 Full GC,从而增加Full GC的频率,也即增加了间歇性停顿的次数。强烈建议能不使用此方法就别使用，让虚拟机自己去管理它的内存，可通过通过-XX:+ DisableExplicitGC来禁止RMI调用System.gc。

## 老年代空间不足

老年代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出如下错误：
`java.lang.OutOfMemoryError: Java heap space`
为避免以上两种状况引起的Full GC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。

## 永生区空间不足

JVM规范中运行时数据区域中的方法区，在HotSpot虚拟机中又被习惯称为永生代或者永生区，Permanet Generation中存放的为一些class的信息、常量、静态变量等数据**，**当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下也会执行Full GC。**如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息：
`java.lang.OutOfMemoryError: PermGen space`
为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

## CMS GC时出现promotion failed和concurrent mode failure

对于采用CMS进行老年代GC的程序而言，尤其要注意GC日志中是否有promotion failed和concurrent mode failure两种状况，当这两种状况出现时可能会触发Full GC。

promotion failed是在进行Minor GC时，survivor space放不下、对象只能放入老年代，而此时老年代也放不下造成的**；
**concurrent mode failure是在执行CMS GC的过程中同时有对象要放入老年代，而此时老年代空间不足造成的（有时候“空间不足”是CMS GC时当前的浮动垃圾过多导致暂时性的空间不足触发Full GC）。

对措施为：增大survivor space、老年代空间或调低触发并发GC的比率，但在JDK 5.0+、6.0+的版本中有可能会由于JDK的bug29导致CMS在remark完毕后很久才触发sweeping动作。对于这种状况，可通过设置-XX: CMSMaxAbortablePrecleanTime=5（单位为ms）来避免。

## 统计得到的Minor GC晋升到旧生代的平均大小大于老年代的剩余空间

这是一个较为复杂的触发情况，Hotspot为了避免由于新生代对象晋升到旧生代导致旧生代空间不足的现象，在进行Minor GC时，做了一个判断，如果之前统计所得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间，那么就直接触发Full GC。

例如程序第一次触发Minor GC后，有6MB的对象晋升到旧生代，那么当下一次Minor GC发生时，首先检查旧生代的剩余空间是否大于6MB，如果小于6MB，则执行Full GC。

当新生代采用PS GC时，方式稍有不同，PS GC是在Minor GC后也会检查，例如上面的例子中第一次Minor GC后，PS GC会检查此时旧生代的剩余空间是否大于6MB，如小于，则触发对旧生代的回收。

除了以上4种状况外，对于使用RMI来进行RPC或管理的Sun JDK应用而言，默认情况下会一小时执行一次Full GC。可通过在启动时通过- java - Dsun.rmi.dgc.client.gcInterval=3600000来设置Full GC执行的间隔时间或通过-XX:+ DisableExplicitGC来禁止RMI调用System.gc。

## 堆中分配很大的对象

所谓大对象，是指需要大量连续内存空间的java对象，例如很长的数组，此种对象会直接进入老年代，而老年代虽然有很大的剩余空间，但是无法找到足够大的连续空间来分配给当前对象，此种情况就会触发JVM进行Full GC。

为了解决这个问题，CMS垃圾收集器提供了一个可配置的参数，即-XX:+UseCMSCompactAtFullCollection开关参数，用于在“享受”完Full GC服务之后额外免费赠送一个碎片整理的过程，内存整理的过程无法并发的，空间碎片问题没有了，但提顿时间不得不变长了，JVM设计者们还提供了另外一个参数 -XX:CMSFullGCsBeforeCompaction,这个参数用于设置在执行多少次不压缩的Full GC后,跟着来一次带压缩的。

# Finalize()方法

## 1. finalize的作用

(1)finalize()是Object的protected方法，子类可以覆盖该方法以实现资源清理工作，GC在回收对象之前调用该方法。
(2)finalize()与C++中的析构函数不是对应的。C++中的析构函数调用的时机是确定的（对象离开作用域或delete掉），但Java中的finalize的调用具有不确定性
(3)不建议用finalize方法完成“非内存资源”的清理工作，但建议用于：① 清理本地对象(通过JNI创建的对象)；② 作为确保某些非内存资源(如Socket、文件等)释放的一个补充：在finalize方法中显式调用其他资源释放方法。其原因可见下文[finalize的问题]

## 2. finalize的问题

(1)一些与finalize相关的方法，由于一些致命的缺陷，已经被废弃了，如System.runFinalizersOnExit()方法、Runtime.runFinalizersOnExit()方法
(2)System.gc()与System.runFinalization()方法增加了finalize方法执行的机会，但不可盲目依赖它们
(3)Java语言规范并不保证finalize方法会被及时地执行、而且根本不会保证它们会被执行
(4)finalize方法可能会带来性能问题。因为JVM通常在单独的低优先级线程中完成finalize的执行
(5)对象再生问题：finalize方法中，可将待回收对象赋值给GC Roots可达的对象引用，从而达到对象再生的目的
(6)finalize方法至多由GC执行一次(用户当然可以手动调用对象的finalize方法，但并不影响GC对finalize的行为)

## 3. finalize的执行过程(生命周期)

(1) 首先，大致描述一下finalize流程：当对象变成(GC Roots)不可达时，GC会判断该对象是否覆盖了finalize方法，若未覆盖，则直接将其回收。否则，若对象未执行过finalize方法，将其放入F-Queue队列，由一低优先级线程执行该队列中对象的finalize方法。执行finalize方法完毕后，GC会再次判断该对象是否可达，若不可达，则进行回收，否则，对象“复活”。

(2) 具体的finalize流程：
  对象可由两种状态，涉及到两类状态空间，一是终结状态空间 F = {unfinalized, finalizable, finalized}；二是可达状态空间 R = {reachable, finalizer-reachable, unreachable}。各状态含义如下：

* unfinalized: 新建对象会先进入此状态，GC并未准备执行其finalize方法，因为该对象是可达的
* finalizable: 表示GC可对该对象执行finalize方法，GC已检测到该对象不可达。正如前面所述，GC通过F-Queue队列和一专用线程完成finalize的执行
* finalized: 表示GC已经对该对象执行过finalize方法
* reachable: 表示GC Roots引用可达
* finalizer-reachable(f-reachable)：表示不是reachable，但可通过某个finalizable对象可达
* unreachable：对象不可通过上面两种途径可达

状态变迁图：

![1](https://i.loli.net/2020/07/31/8eRTZjMYNz3wXiA.jpg)

变迁说明：

  (1)新建对象首先处于[reachable, unfinalized]状态(A)
  (2)随着程序的运行，一些引用关系会消失，导致状态变迁，从reachable状态变迁到f-reachable(B, C, D)或unreachable(E, F)状态
  (3)若JVM检测到处于unfinalized状态的对象变成f-reachable或unreachable，JVM会将其标记为finalizable状态(G,H)。若对象原处于[unreachable, unfinalized]状态，则同时将其标记为f-reachable(H)。
  (4)在某个时刻，JVM取出某个finalizable对象，将其标记为finalized并在某个线程中执行其finalize方法。由于是在活动线程中引用了该对象，该对象将变迁到(reachable, finalized)状态(K或J)。该动作将影响某些其他对象从f-reachable状态重新回到reachable状态(L, M, N)
  (5)处于finalizable状态的对象不能同时是unreahable的，由第4点可知，将对象finalizable对象标记为finalized时会由某个线程执行该对象的finalize方法，致使其变成reachable。这也是图中只有八个状态点的原因
  (6)程序员手动调用finalize方法并不会影响到上述内部标记的变化，因此JVM只会至多调用finalize一次，即使该对象“复活”也是如此。程序员手动调用多少次不影响JVM的行为
  (7)若JVM检测到finalized状态的对象变成unreachable，回收其内存(I)
  (8)若对象并未覆盖finalize方法，JVM会进行优化，直接回收对象（O）
  (9)注：System.runFinalizersOnExit()等方法可以使对象即使处于reachable状态，JVM仍对其执行finalize方法