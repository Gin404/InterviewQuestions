## java基础
### 基础类型
byte：1字节  
short：2字节  
int：4字节  
long：8字节  
float：4字节  
double：8字节  
char：2字节  
boolean：数组情况下是1字节，单个boolean是4字节  




### Hashmap/SparseArray/ConcurrentHashMap实现。
**1.SparseArray**  
SparseArray是Android中一种特有的数据结构,用来替代HashMap的.初始化时默认容量为10它里面有两个数组,一个是int[]数组存放key,一个是Object[]数组用来存放value.  
它的key只能为int.在put时会根据传入的key进行二分查找找到合适的插入位置,如果当前位置有值或者是DELETED节点,就直接覆盖,否则就需要拷贝该位置后面的数据全部后移一位,空出一个位置让其插入.如果数组满了但是还有DELETED节点,就需要调用gc方法,gc方法所做的就是把DELETED节点后面的数前移,压缩存储(把有数据的位置全部置顶).数组满了没有DELETED节点,就需要扩容.
调用remove时,并不会直接把key从int[]数组里面删掉,而是把当前key指向的value设置成DELETED节点,这样做是为了减少int[] 数组的结构调整,结构调整就意味着数据拷贝.但是当我们调用keyAt/valueAt获取索引时,如果有DELETED节点旧必须得调用gc,不然获得的index是不对的.延迟回收的好处适合频繁删除和插入来回执行的场景,性能很好.
get方法很简单,二分查找获取key对应的索引index,返回values[index]即可.  
可以看到SparseArray比HashMap少了基本数据的自动装箱操作,而且不需要额外的结构体,单个元素存储成本低,在数据量小的情况下,随机访问的效率很高.但是缺点也显而易见,就是增删的效率比较低,在数据量比较大的时候,调用gc拷贝数组成本巨大.
除了SparseArray,Android还提供了SparseIntArray(int:int),SparseBooleanArray(int:boolean),SparseLongArray(int:long)等,其实就是把对应的value换成基本数据类型.n  
**2. HashMap**  
**基本结构**：  
jdk1.7 数组+链表。数组下标为根据key值算出的哈希值，数组元素为链表头，链表元素为KV键值对。  
jdk1.8 数组+链表+红黑树。和jdk1.7的区别是，链表长度**超过8**的时候，链表结构会被转为红黑树。查询时间复杂度是O(log(n))，优于链表的O(n)。

**jdk1.7流程：**  
**初始化**：  
数组初始长度：16，数组扩容必须乘2^n。  
默认负载因子：0.75，也就是数组75%的位置满了之后，会进行扩容。  
**put**：
1. 判断数组table是否为null或者长度n是0，如果是，则执行resize进行初始化。
2. 根据hash%n得到index，如果当前index上没有节点，则新建节点。
3. 如果存在节点，那就是hash冲突，  
   a. 遍历各节点，有相同的key值，则返回旧值，更新新值。  
   b. 没有相同的节点，新建节点插入到对应链表尾部。  
   c. 在遍历链表的过程中，如果链表长度大于8 (jdk1.8)，则转换链表为红黑树。
4. ++size，如果size大于扩容阈值，则resize。
5. 返回为null，那就是插入，不是更新。  

**get**：  
   比较简单，就是遍历链表和红黑树找到对应key的节点并返回value，否则返回null。  
   **为啥长度一定要是2^n?**  
   计算index的方式为hash&(len-1)，位运算计算效率高。换成二进制，len-1永远是1111...111，所以相当于直接取hash最后几位作为Index，分布比较均匀。而位运算下的十进制会有分布不均匀的情况，某种index会永远用不到。  
   **线程安全问题！！！**  
   首先了解一下resize的过程。  
   当数组长度超过Capacity*LoadFactor的时候，会进行扩容。  
   a. 先创建新的数组。  
   b. 然后遍历原数组，把Entry重新哈希到新数组。  
   **多个线程，同时扩容的过程中，有可能产生环形链表！！！** https://juejin.cn/post/7139320304920166430#heading-3  
   **此后，进行get操作，会造成死循环！！！**

**jdk1.8**：  
循环链表的情况，已经解决了。jdk有别的问题。  
**put方法会导致元素丢失**  
多线程put，在p.next链表下一个元素新建赋值的时候，有覆盖的情况。  
**put中要rehash扩容的时候，get可能返回null**
线程1新建table并赋值给成员参数的时候，线程2get就会使用这个新的table，返回的就是null了。

**3. ConcurrentHashMap**  
**jdk1.7版本**  
分段锁机制。每个ConcurrentHashMap维护一组Segment，每个Segment类似一个HashMap。每次put操作给自己的Segment加锁就行。put和rehash都是安全的。  
**jdk1.8版本**  
和1.7不同，没用Segment，而是和HashMap类似，用了一个Node数组来存储元素，也是数组+链表结构。  
大量运用了Unsafe类保证线程安全。  
**put**  
如果对应的key的index上没有元素，则用Unsafe的CAS操作，新建节点。  
如果发生了hash冲突，则以链表头为锁，synchronized。  
**扩容**  
新增一个成员变量nextTable，扩容后新的空的Node数组先赋值给这个成员变量，rehash都是针对这个nextTable完全不影响原来的table。  
rehash的过程会用链表头结点作为锁加锁，此时不能对这个链表进行put。  
在整个过程中，共享变量的存储和读取全部通过volatile或CAS的方式，保证了线程安全。  
完成扩容后，再将nextTable赋值给table。  

## JVM
### 反射是什么，在哪里用到，怎么利用反射创建一个对象?
**概念**：在程序运行时，程序有能力获取一个类的所有方法和属性；并且对于任意一个对象，可以调用它的任意方法或者获取其属性。  
java文件需要编译成.class文件才能被jvm加载使用,对象的.class数据在jvm里就是Class<T>；我们如果能拿到这个Class<T>对象，就能获取该Class<T>对应的对象类型，及在该类型声明的方法和属性值；还可以根据Class<T>创建相应的类型对象，通过Field,Method反过来操作对象。  
**应用**：
1. **动态**获取和使用类的**所有**变量和方法。包括private的！
2. 配合使用runtime的自定义注解。
3. 动态加载第三方jar包。
4. 按需加载类，节省编译和初始化时间。
5. Java中一个重要应用场景是动态代理。  
   **反射创建一个对象**  
   主要是如何获取到Class对象。
1. 通过实例获取：object.getClass()
2. 通过已知类型：Object.class
3. 通过Class.forName获取全路径指定类名的class，最终会调用到native方法。
### 2. 类加载的过程?
类加载的过程其实就是类的生命周期的一部分。一个.class文件从被加载到虚拟机内存里到从内存中卸载的过程。  
类的生命周期包括：**加载，链接，初始化，使用，卸载**。其中链接包括**验证**，**准备**和**解析**。  
加载：查找并加载class文件。  
链接-验证：确保被导入类型的正确性。  
链接-准备：**为类的静态字段分配字段，并用默认值初始化这些字段**。  
链接-解析：虚拟机常量池内的符号引用替换为直接引用。  
初始化：**将类变量初始化为正确初始值**。  
其中，加载阶段是有Java虚拟机外部的**类加载子系统**完成的。 类加载子系统通过两类类加载器查找和加载Class文件：**系统加载器**和**自定义加载器**。  
**系统加载器**  
Bootstrap ClassLoader（引导类加载器）：用C/C++代码实现的类加载器，用来加载JDK的核心库，比如java.lang、java.ui。加载路径：$JAVA_HOME/jre/lib目录或者-Xbootclasspath指定的目录。  
Extensions ClassLoader（拓展类加载器）：用于加载java拓展类，提供除系统类之外的额外功能。加载目录：$JAVA_HOME/jre/lib/ext或者系统属性java.ext.dir所指定的目录。  
Application ClassLoader（应用程序类加载器）：又称System ClassLoader，可以通过ClassLoader的getSystemClassLoader方法获取到。加载路径：当前应用程序classpath目录或系统属性java.class.path指定目录。  
**自定义加载器**  
通过继承java.lang.ClassLoader的方式实现自己的类加载器。

### 对象的创建？
对象的创建，也就是虚拟机执行new Object()的时候有哪些过程。
1. 虚拟机首先会检查常量池中是否有这个类的符号引用，并且检查这个符号引用代表的类是否已经被加载、链接、初始化。
2. 类加载完成后，会在java堆分配一块内存给对象。内存的分配方式根据所采用的垃圾收集器是否带有压缩整理功能有关。
   a. 指针碰撞：如果堆内存是完整的，也就是用过的内存放一边，空闲内存放一边，则在位于二者中间的指针指示器向空闲一侧移动一段与对象大小一样的内存，完成分配。  
   b. 空闲列表：如果堆内存是不完整的，则需要虚拟机维护一个列表记录哪些内存可用。寻找可用内存给对象，然后更新列表。
3. 处理并发安全问题。  
   两种解决方式：  
   a. 对**分配内存空间的动作**做同步处理。比如在虚拟机采用CAS算法并配上失败重试的方式保证原子性。  
   b. 每个现在Java堆中预先分配一小块内存，这款内存被称为本地线程分配缓冲（TLAB），线程需要分配内存时，就在对应的TLAB上分配，一块TLAB用完后需要新的TAB时，才需要同步锁定。
4. 初始化分配的内存空间。  
   将分配的内存除了对象头之外都初始化为0.
5. 设置对象的对象头。  
   将对象所属的类、hashcode、GC分代年龄等数据存储在对象的对象头中。
6. 执行init方法，初始化对象的成员变量，调用类的构造方法。
### jvm内存结构。（运行时数据区域）
1. 程序计数器：也就是PC寄存器，**线程私有**。用来保存下一条需要执行的字节码的地址。如果不是Native方法，那存储的就是正在执行的字节码的地址；如果是Native方法，那就是Undefined。唯一没有规定任何OutOfMemoryErro情况的数据区域。
2. Java虚拟机栈：**线程私有**。用来存储线程中Java方法调用的状态，包括局部变量、参数、返回值以及运算中间结果等。当中有多个栈帧，调用一个方法就会压入一个栈帧。线程请求的栈容量超过最大容量会抛出StackOverflow；虚拟机栈可以动态扩展，但是如果扩展没有足够内存会抛出OutOfMemoryError。
3. 本地方法栈：与Java虚拟机栈类似，只不过是支持Native语言的，比如C/C++。
4. Java堆：**线程共享**。用来存放对象实例。对象由垃圾收集器管理，无法显示的销毁。如果实例分配没有足够内存或者无法进行扩展时，会抛出OutOfMemoryError。
5. 方法区：**线程共享**。用来存储已经被虚拟机加载的类结构信息，包括运行时常量池、字段、方法信息、静态变量等数据。同样内存不够时会抛出OutOfMemoryError异常。
6. 运行时常量池：方法区的一部分。Class文件里除了类的版本、接口、字段和方法等信息，还包含常量池，用来存放编译时期生产的字面量和符号引用。这些内容会在类加载后存放在方法区的运行时程立池中，也就是将这些变为具体的直接引用。
### 垃圾回收机制。
GC的两个工作：内存的划分和分配；垃圾回收。  
内存分配：粗一点儿分为新生代、老年代。新生代细分为Eden、From Survivor、To Survivor。  
垃圾回收的对象：不被引用的对象，GC需要**标记**这些对象。  
**标记算法**  
引用计数法（主流不使用）：每个对象都有个引用计数器，如果被引用就+1，为0时就被回收。缺点：不能解决循环依赖的问题，比如A和B互相引用，再无其他引用，实则应该被回收。  
根搜索算法（主流在使用）：选定一些对象作为GC roots。以这些对象为起点，构成一个引用树，不在这里的对象说明没有被引用（不可达），可以被回收。作为GC root的对象包括：Java栈中引用的对象，本地方法栈中引用的对象、方法区运行时常量池引用的对象、方法区静态属性引用的对象、运行中的线程、Boorstrap ClassLoader记载的对象、GC控制的对象。  
**标记后的对象会被立即回收吗？**  
不会，如果对象不可达，并且GC准备好重新分配空间后，会先执行对象的**finalize**方法。如果finalize后仍然不可达，则会真正被回收。  
**垃圾收集算法**
1. 标记-清除算法：标记被回收的对象，然后清除被标记对象。最基础的算法，缺点是容易产生大量不连续的内存碎片。
2. 复制算法：将内存分为两块，每次对一块进行回收，将存活对象复制到另一块。代价是可用内存少一半，适合存活对象少的情况。被广泛用于新生代（因为绝大多数对象生命周期短，且存活于新生代中）。
3. 标记-压缩算法：老年代对象存活时间久，复制算法效率较低。标记压缩算法在标记后，将可存活对象复制压缩到内存的一端，然后再清除边界外的内存。
4. 分代收集算法：基于前面对新生代老年代的划分，分为Minor GC和Full GC。Minor GC针对新生代，是将Eden 和 From Survivor中存活的对象，复制到To Survivor中。有两种情况对象会晋升到老年代：存活对象分代年龄超过阈值（经历过对此Minor GC）和 To Survivor空间满了。复制完后，Eden和From就都是可回收的对象。然后From和To的两块空间会交换角色。Full GC就是用前面说的标记压缩法进行回收，耗时较长，收集频率比较低。


### volatile的作用，在哪儿用到？
volatile关键字为实例域的同步访问提供了免锁机制。volitale保证可见性（一个线程对值的修改立刻对其他线程可见）和有序性（禁止指令重排序，操作volitale变量之前的操作肯定在之后的操作前完成了），但是不保证原子性（比如自增操作）。  
volitale某些情况下能替代syncronized，但是涉及到自增、自减，或者变量包含在其他变量的不变式中就不能用这个关键字（setUpper setLower的例子）。  
**用处**
1. 状态标志：多线程共享的标志位。（体现可见性）
2. 单例的DCL双重校验锁。（体现有序性，如果不加，对象的引用关联、构造函数可能重排序）
### AtomicBoolean的实现原理？什么CAS？ABA问题？
CAS是Compare and Swap的缩写。一种乐观锁机制，假如要更新对变量a=0加1，则线程从主存中拷贝一个预期值a=0到自己的工作内存，然后+1得到新值1，再用预期值和主存中的当前值比较，如果想到，说明在此期间主存值没有发生变化，可以将新值覆盖写入。如果发生变化，则放弃此次更新，自旋重复CAS操作。  
Atomic系列都借助了jdk的工具类Unsafe实现了CAS操作，保证变量更新操作的原子性。  
CAS有个ABA的问题，那就是比较的时候，如果预期值和当前值相等，不代表当前值没有被更新，而是有可能更新了后等于原值。解决方法是给变量加上版本号：A-B-A  ->  A1-B2-A3。
### 乐观锁，悲观锁？乐观锁？
乐观锁主要就是指CAS，乐观锁时常抱有乐观的想法，即默认读多写少，且遇到并发写入的可能性低。所以不会直接上锁，而是在每次更新的时候，比较版本号，如果版本号一致，则更新，如果不一致，则失败进行重读。  
悲观锁和乐观锁正好相反。悲观锁默认写入操作多，而且会经常遇见并发写入的操作，所以每次读写数据时都会上锁。每次修改和读数据时，都需要拿到锁才可以。常见的悲观锁实现有Synchronized 和 ReentrentLock。

### ReentrantLock synchronized volitle的区别？
volatile前面说过了，某些情况可以代替锁。  
ReentrantLock和synchronized都是悲观锁。二者区别如下：
1. ReentrentLock显示的获得，释放锁，而Sychronized隐式的获得，释放锁。
2. ReentrentLock可响应中断锁，sychronized不可以。ReentrentLock加锁有一个lockInterruptly方法，在同步块里可以catch中断。sychronized需要自己判断中断标志位.
3. ReentrentLock是API级别的，Sychronized是JVM级别的。
4. ReentrentLock可以是公平也可以是不公平的，默认不公平，syncrhonized是不公平的。
5. ReentrentLock可以通过Condition绑定条件。通过condition来await和signall，和syncronized的await和notifyAll一样。
6. ReentrentLock发生异常，如果没有unlock，很可能出现死锁，所以一定要由finally模块，进行对锁的释放，Sychronized发生异常会自动释放线程占用的锁。

### sleep和wait的区别？  
1. sleep是Thread的静态方法，wait是Object的实例方法。
2. wait涉及到锁机制，必需在synchronized代码块里执行；sleep可以用在任何地方，不会释放锁，只是使当前线程休眠，会让出cpu资源。
3. wait在没有设置时限的情况下，需要被notify/notifyAll唤醒。sleep到指定时间自动唤醒。
### 线程池？  
线程池用到阻塞队列。阻塞队列常用于生产者和消费者的场景。    
队列中没有数据的情况下，消费者的所有线程会被挂起。  
队列数据满的情况下，生产者的所有线程会被挂起。  
**常见阻塞队列种类**    
ArrayBlockingQueue: 由数组结构组成的有界阻塞队列，按照FIFO的原则对元素进行排列。默认不保证线程公平的访问。也就是先阻塞的不一定先访问。  
LinkedBlockingQueue: 由链表结构组成的有界阻塞队列，同样是FIFO。除了数据结构实现不同，LinkedBlockingQueue的生产和消费使用两把锁来进行的，而ArrayBlockingQueue是一把锁。  
PriorityBlockingQueue: 支持优先级的无界阻塞队列。默认情况下按照自然顺序升序排列，可以自己实现compareTo或者Comparator。  
DelayQueue: 支持延时获取元素的无界阻塞队列。队列使用PriorityQueue实现，队列中的元素必须实现Delayed接口指定元素到期时间。只有元素到期后才能取走。  
SynchronousQueue: 一个不支持存储元素的阻塞队列。每个插入操作必须等待另一个线程的移出操作。反之亦然。严格来说不是一种容器。  
LinkedTransferQueue: 一个链表结构组成的无界阻塞队列。实现了一个接口TransferQueue。一言概之，transfer的作用就是如果有等待的消费者线程，那就把元素直接给消费者线程，否则放入队列尾部并且进入等待阻塞状态。  
LinkedBlockingDeque: 一个链表结构的双向阻塞队列。支持双端插入和取出元素。
**线程池**  
创建线程池用ThreadPoolExecutor。它的构造函数参数如下：  
**corePoolSize**: 核心线程数。默认线程池是空的，如果当前线程数小于corePoolSize，则创建新的线程；如果大于corePoolSize，则不创建新线程。  
**maximumPoolSize**: 线程池允许创建的最大线程数。如果任务队列满了，并且线程数小于maximumPoolSize，则仍会创建新的线程。  
**keepAliveTime**: 非核心线程超时时间。超过这个时间，非核心线程会被回收。  
**TIMEUnit**: keepAliveTime的单位。  
**ThreadFactory**: 线程工厂。可以用线程工厂给每个创建的线程设置名字。一般不需要设置此参数。  
**RejectExecutionHandler**: 饱和策略。当前任务队列和线程池都满的时候应对策略。默认是AbortPolicy，无法处理新任务，并且抛出异常。除此之外还有CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy。  
**线程池处理逻辑**    
提交任务->没有达到核心线程数就创建线程执行任务->队列没满就放在队列里->没有达到最大线程数就创建线程执行任务->饱和策略  
core pool > queue > not core pool
**常见线程池种类**    
**FixedThreadPool**: 只有核心线程，且数量固定。采用LinkedBlockingQueue，容量设置为Integer.MAX_VALUE。  
**CachedThreadPool**: 没有核心线程，最大线程数位MAX_VALUE，keepAliveTime位60s。采用SynchronousQueue。也就是每次提交新任务，都立即会有线程去处理。适合大量需要立即执行且耗时少的任务。  
**SingleThreadExecutor**: 只有一个核心线程，没有非核心线程。采用LinkedBlockingQueue。确保任务在单个线程中按顺序执行。  
**ScheduleThreadPool**: 自定义核心线程数，最大线程数为MAX_VALUE。使用DelayedWorkQueue。用于延时或者周期执行任务。