## 基础类型
byte：1字节  
short：2字节  
int：4字节  
long：8字节  
float：4字节  
double：8字节  
char：2字节  
boolean：数组情况下是1字节，单个boolean是4字节

## 字符串
### String概述
1. 不可变性，体现在用final修饰不能被集成，任何对string的操作都是返回新的实例。
2. 内部使用private、final的char数组存储字符串。

### String为啥要被设计成不可变的？
1. 使用习惯需要，一般就是使用字面量，当做基本数据类型来使用。
2. 计算需要，hashcode不需要每次使用计算。
3. 线程安全需要。本质上因为不可写。每次对string操作，比如拼接，都返回新的实例（相对于原字符串）。
4. 缓存需要。字符串池保存的字符串能放心的复用。

### 字符串池和String.intern
1. 字面量方式创建的String会在字符串池中；new方式创建的实例会在堆中和字符串池中（如果之前没有）各创建一个实例。
2. jdk1.7: 字符串池在永久代，如果字符串池存在和当前字符串内容相同的缓存，则返回缓存实例。否则在字符串池中创建相应实例。
3. jdk1.8: 字符串池在堆中，存储的是字符串堆引用，如果字符串池中存在与当前字符串内容相同的缓存，则返回缓存的引用。否则则在字符串池中存储当前字符串的引用。


## 泛型
### 数组为什么不支持泛型？（不准确）
1. 对于数组的引用的类型上的泛型来说（等号左边），能在编译期保证类型安全； //Basket<Apple>[] array = new Basket[];
2. Java不允许创建（new 关键字）泛型数组实例； //new后边不能带有泛型
3. Java中的数组，语法上规定必须具有协变性，无法在编译期彻底保证类型安全，而这也与泛型的设计理念相悖，泛型默认是不变的。
所以数组左边能加泛型能起到编译期校验类型的作用，但是由于协变，可以强转数组实例为Object[]类型，放入其他实例，编译期不会报错，runtime才会报错。  

### 集合类型语法上支持泛型
1. 泛型具有不变形，List<? extends Fruit>，所以不能根据泛型关系强转集合类型。List<Apple>不能强转为List<Fruit>，除非强转为List。

### 泛型擦除
1. 任何结合类型在字节码上new指令里，都不带泛型类型。所以运行时无法保证类型安全。

## 泛型擦除带来的不便
1. 方法重载，不同泛型类型，但却会导致方法无法重载。
2. 静态作用域无法使用类泛型。class Basket<T> { static Basket<T> create(){} }，create方法报错，因为Basket<具体type>这个类并不存在！
3. 不能通过未具体化的泛型类型创建实例。new T()不行。
4. 不能使用instanceof，instanceof ArrayList<String>不行
5. data.getClass() == T.class也不行。
6. 

### 在kotlin中的体现
1. kotlin不需要先前兼容，所以数组Array<Fruit>和集合的api设计一致，都默认不具备协变。

### 怎么在runtime获取到泛型信息？
**通过子类把泛型信息具体化！！！**

   public class AppleBasket extends Basket<Apple>
   
**字节码保留的superclass信息，是擦除后的Basket；但是属性里的signature会保留泛型类型！！！！**  
**所以就可以通过匿名内部类的方式，构造一个子类实例，通过getClass().getGenericSuperclass().getActualTypeArguments()获取！**  
**事实上，Gson的TypeToken就是这么做的！！！**


## 内部类的问题
### 匿名内部类为啥访问外部变量要是final的？
首先，只有访问外部的局部变量才要加final，因为内部类持有局部变量的引用，保证内部类和外部类的引用指向同一个对象。  
如果是访问实例变量就不需要加final，因为内部类持有外部类的引用，是通过外部类的引用来访问实例变量的。  
如果外部的实例变量是private的，那就会在字节码生成一个方法，把内部类的操作迁移到方法里（类似setter）。


## Hashmap/SparseArray/ConcurrentHashMap实现。
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

**为啥扩容要用2的n次幂**
1. 在计算数组下标时，能用位云端代替取余操作，提高性能；
2. 在扩容时，无需重新计算每个节点，只需检查hash中的一位。

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
