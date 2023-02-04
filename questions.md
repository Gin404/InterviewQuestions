# 知识点
## 此文档旨在针对常问问题（知识点），每个问题总结一个300字左右的答案，不要长篇赘述。细节可以贴相关链接。
## java基础
### 1. 反射是什么，在哪里用到，怎么利用反射创建一个对象?
### 2. 对象加载的过程，属性先加载还是方法先加载?
### 3. 垃圾回收机制与jvm内存结构。
### 4. Hashmap/SparceArray/, ConcurrentHashMap实现。
1.SparceArray
SparseArray是Android中一种特有的数据结构,用来替代HashMap的.初始化时默认容量为10它里面有两个数组,一个是int[]数组存放key,一个是Object[]数组用来存放value.它的key只能为int.在put时会根据传入的key进行二分查找找到合适的插入位置,如果当前位置有值或者是DELETED节点,就直接覆盖,否则就需要拷贝该位置后面的数据全部后移一位,空出一个位置让其插入.如果数组满了但是还有DELETED节点,就需要调用gc方法,gc方法所做的就是把DELETED节点后面的数前移,压缩存储(把有数据的位置全部置顶).数组满了没有DELETED节点,就需要扩容.

调用remove时,并不会直接把key从int[]数组里面删掉,而是把当前key指向的value设置成DELETED节点,这样做是为了减少int[] 数组的结构调整,结构调整就意味着数据拷贝.但是当我们调用keyAt/valueAt获取索引时,如果有DELETED节点旧必须得调用gc,不然获得的index是不对的.延迟回收的好处适合频繁删除和插入来回执行的场景,性能很好.

get方法很简单,二分查找获取key对应的索引index,返回values[index]即可.

可以看到SparseArray比HashMap少了基本数据的自动装箱操作,而且不需要额外的结构体,单个元素存储成本低,在数据量小的情况下,随机访问的效率很高.但是缺点也显而易见,就是增删的效率比较低,在数据量比较大的时候,调用gc拷贝数组成本巨大.

除了SparseArray,Android还提供了SparseIntArray(int:int),SparseBooleanArray(int:boolean),SparseLongArray(int:long)等,其实就是把对应的value换成基本数据类型.


### 5. volatile的作用，在哪儿用到？
### 6. AtomicBoolean的实现原理？什么CAS？
### 7. 乐观锁，悲观锁？乐观锁CAS，ABA？
### 8. ReentrantLock synchronize volitle的区别？
### 9. 多线程协同？
### 10. 线程池？
### 11. sleep和wait的区别？

## 设计模式
### 1. 设计原则？
### 2. 分类？
### 3. android常用的设计模式？

## 计算机基础
### 1. http与https有什么区别?
1. http和https的概念：  
HTTP：是互联网上应用最为广泛的一种网络协议，是一个客户端和服务器端请求和应答的标准（基于TCP），用于从WWW服务器传输超文本到本地浏览器的传输协议，它可以使浏览器更加高效，使网络传输减少。  
HTTPS：是以安全为目标的HTTP通道，简单讲是HTTP的安全版，即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。  
**HTTPS协议的主要作用可以分为两种：一种是建立一个信息安全通道，来保证数据传输的安全；另一种就是确认网站的真实性。**  
2. https通信步骤  
- 首先服务器和客户端之间最终采用对称秘钥加密通信，但是认证过程涉及2组非对称加密。  
- 整个过程涉及两组秘钥，属于认证机构的非对称秘钥(K1, P1)，属于服务器的非对称秘钥(K2, P2)。K表示公钥，P表示私钥。  
- X.509证书: 由认证机构颁发，用于将加密秘钥和公司进行安全关联。包含：P1加密的服务端公钥K2，和P1加密的证书签名。
- 具体步骤：

	a. 服务器向权威机构申请证书。  
	b. 服务器把证书发给客户端。  
	c. 客户端用内置的K1，解密证书，验证签名，获取服务器公钥K2。  
	d. 客户端用K2加密自己的对称秘钥，发送给服务器。  
	e. 客户端和服务器用这一组对称秘钥进行通信。  

所以如果问到Charles抓包原理，关键在于证书上。把Charles证书安装在手机上后，手机把Charles当成服务器，完成一次https认证和通信；Charles作为客户端再和真正的服务器完成https认证和通信。

### 2. 浏览器输入一个域名发生了什么？  
1. 通过DNS解析IP地址  
	首先会找缓存，顺序：浏览器 -> 系统 -> 路由器 -> ISP；  
	没有的话，会向DNS服务器发送DNS查找请求。  
2. 浏览器查找本地是否存在缓存的Html，如果没有，则进行下一步TCP连接。  
3. 与 WEB 服务器建立 TCP 连接。  
	三次握手：  
	1. 客户端通过 SYN 报文段发送连接请求，确定服务端是否开启端口准备连接。状态设置为 SYN_SEND;  
	2. 服务器如果有开着的端口并且决定接受连接，就会返回一个 SYN+ACK 报文段给客户端，状态设置为 SYN_RECV；  
	3. 客户端收到服务器的 SYN+ACK 报文段，向服务器发送 ACK 报文段表示确认。此时客户端和服务器都设置为 ESTABLISHED 状态。连接建立，可以开始数据传输了。  

4. 如果是HTTPS的话，就进行加密认证。  
	
5. 浏览器发送请求获取页面html。  
6. 服务器响应html。  
7. 浏览器解析 HTML，渲染页面，执行js脚本等...  

## Android基础
### 1. Handler原理？

1. Looper 准备和开启循环

   - Looper#`prepare()` 初始化线程独有的 `Looper` 以及 `MessageQueue`
   - Looper#`loop()` 开启**死循环**读取 MessageQueue 中下一个满足执行时间的 Message
     - 尚无 Message 的话，调用 Native 侧的 `pollOnce()` 进入**无限等待**
     - 存在 Message，但执行时间 `when` 尚未满足的话，调用 pollOnce() 时传入剩余时长参数进入**有限等待**

2. Message 发送 、入队、出队

   Native 侧如果处于无限等待的话：任意线程向 `Handler` 发送 `Message` 或 `Runnable` 后，Message 将按照 when 条件的先后，被插入 Handler 持有的 Looper 实例所对应的 MessageQueue  中**适当的位置**。 MessageQueue 发现有合适的 Message 插入后将调用 Native 侧的 `wake()` 唤醒无限等待的线程。这将促使 MessageQueue 的读取继续**进入下一次循环**，此刻 Queue 中已有满足条件的 Message 则出队返回给 Looper

   Native 侧如果处于有限等待的话：在等待指定时长后 epoll_wait 将返回。线程继续读取 MessageQueue，此刻因为时长条件将满足将其出队

3. Looper 处理 Message 的实现

   Looper 得到 Message 后回调 Message 的 `callback` 属性即 Runnable，或依据 `target` 属性即 Handler，去执行 Handler 的回调。

   - 存在 `mCallback` 属性的话回调 `Handler$Callback`
   - 反之，回调 `handleMessage()`

   

   参考链接：https://juejin.cn/post/7054830013102686238#heading-0

   https://github.com/xfhy/Android-Notes/blob/master/Blogs/Android/%E7%B3%BB%E7%BB%9F%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/Handler%E6%9C%BA%E5%88%B6%E4%BD%A0%E9%9C%80%E8%A6%81%E7%9F%A5%E9%81%93%E7%9A%84%E4%B8%80%E5%88%87.md

### 2. 同步屏障？Choreographer？  
1. 首先Messsage有一个参数isAsynchronous，当为true的时候，消息为异步消息，为false时，为同步消息。  
2. 当一个Message的target为null的时候，那他是一个**同步屏障**。开发者无法设置target为null，MessageQueue有一个postSyncBarrier的方法，但是是private的。  
3. 当MQ获取下一个message的时候，如果碰到了同步屏障，**那么不会取出同步屏障，而是往后遍历，跳过所有同步消息，取出下一个异步消息并执行**。  
4. 同步屏障的移除也是由MessageQueue完成，同样是private方法。  
**综上，理论上，开发者只能添加异步消息，但是不能自行添加/移除同步屏障。**  
**那系统何时添加同步屏障呢？**  
结合屏幕刷新机制和view绘制原理，手机接收VSYNC信号的时候（60hz手机每16.6ms一次）会执行view的绘制（measure、layout、draw）。具体原因就在于ViewRootImpl.scheduleTraversals。  
**ViewRootImpl.scheduleTraversals：ui变化都会走到此方法。**  
他干了两件事：向主线程handler发送一个屏障消息 和 向Choreographer注册一个runnable。  
这个runnable会作为异步消息在下一个vsync信号到来的时候被执行，他也是干了两件事：**移除同步屏障 和 performTraversals执行绘制流程**。  
**什么时候使用异步消息呢？**  
同步Handler有一个特点是会遵循与绘制任务的顺序，设置同步屏障之后，会等待绘制任务完成，才会执行同步任务；而异步任务与绘制任务的先后顺序无法保证，在等待VSYNC的期间可能被执行，也有可能在绘制完成之后执行。因此，我的建议是：如果需要保证与绘制任务的顺序，使用同步Handler；其他，使用异步Handler。
### 3. IdleHandler？调用时机？用处？
### 4. HandlerThread？
一个自带handler机制的线程，用于串行执行耗时任务。  
关键点在于run方法中mLooper = Looper.myLooper赋值的过程会加锁。getLooper方法中也会加锁，如果mLooper为空，则一直wait，直到Looper.myLooper执行完notifyAll，才会返回。所以能保证looper不会有空指针。
### 5. invalidate/postInvalidate/requestLayout的区别？*
### 6. Fragment:replace和add的区别？show和hide？commit和commitAllowStateloss？
1. fragment容器为空的时候，replace和add没有区别。
   
2. 如果fragment容器有一个Fragment A。  
- 通过add添加Fragment B，生命周期变化如下：  
A: 无变化；  
B: onAttach 直到 onResume。  
此时，remove B，生命周期变化如下：  
A: 无变化；  
B: onPause 直到 onDetatch。  
（值得注意的是，如果同时调用了addToBackStack，B只会走到onDestroyView，此时通过findFragmentByTag依然可以找到FragmentB的实例。）   

- 通过replace添加Fragment B，生命周期变化如下：  
A: onPause 直到 onDetatch；  
B: onAttach 直到 onResume。  
在2处，通过findFragmentByTag可以找到A的实例。remove B，生命周期状态如下，  
A: 无变化；  
B: onPause 直到 onDetatch；  
(同样，如果同时调用了addToBackStack，B只会走到onDestroyView.)  

- **add和replace比较**  
1. 当Fragment不可见时，如果你要保留Fragment中的数据以及View的显示状态，那么可以使用add操作，后续中针对不同的状态隐藏和显示不同的Fragment。  
优点：快，知识Fragment中View的显示和隐藏。  
缺点：内存中保留的数据太多，容易导致造成OOM的风险。  
2. 当Fragment不可见时，你不需要保留Fragment中的数据以及View的显示状态，那么可以使用replace。  
优点：节省内存，不需要的数据能立即释放掉。  
缺点：频繁创建Fragment,也就是频繁走Fragment生命周期创建和销毁流程，造成性能开销。

- **commit和commitAlowStateloss**  
commit()操作是异步的，内部通过mManager.enqueueAction()加入处理队列。对应的同步方法为commitNow()，commit()内部会有checkStateLoss()操作，如果开发人员使用不当（比如commit()操作在onSaveInstanceState()之后），可能会抛出异常，而commitAllowingStateLoss()方法则是不会抛出异常版本的commit()方法，但是尽量使用commit()，而不要使用commitAllowingStateLoss()。  

- 参考文章  
  https://juejin.cn/post/6844903816240857095  
  https://juejin.cn/post/6943560702292557860
       
### 7. invalidate和requestlayout的区别？postInvalidate呢？
### 8. 事件分发机制？
分析滑动事件分发的4个重点：  
1. 滑动事件MotionEvent，一般需要关注3个事件种类：DOWN，UP，MOVE；  
2. dispatchTouchEvent  
作用是分发事件，如果一个事件分发到了一个view，则他的dispatchTouchEvent方法一定会被调用。返回结果取决于当前view的onTouchEvent和子View的dispatchTouchEvent，表示事件是否被消耗。
3. onInterceptTouchEvent。  
在dispatchTouchEvent方法内部调用，用于返回是否拦截这个事件。如果拦截，同一个事件序列不会再次调用此方法。  
4. onTouchEvent  
在dispatchTouchEvent方法中调用，用来处理点击事件。返回值代表是否消耗当前事件，如果消耗，则在当前事件序列中不会再调用此方法。  

下面伪代码可以表示事件分发的流程：  

	public boolean dispatchTouchEvent(MotionEvent ev) {
		boolean consume = false;
		if(onInterceptTouchEvent(ev)) {
			consume = onTouchEvent(ev);
		} else {
			consume = child.dispatchTouchEvent(ev);
		}
		return consume;
	}

一个MotionEvent的传递顺序：Activity->Window->View...  
**如果不做覆写，默认的事件传递流程：**  
Activity套MyViewGroup套MyView  
MainActivity: dispatchTouchEvent -> MyViewGroup: dispatchTouchEvent -> MyViewGroup: onInterceptTouchEvent -> MyView: dispatchTouchEvent -> MyView: onTouchEvent -> MyViewGroup: onTouchEvent -> MainActivity: TouchEvent  
其他一些要点：  
1. 事件序列以down为起点，中间有若干个move，最终以up为终点。  
2. 某个view一旦决定拦截，那整个事件序列一般只能由他来处理。  
3. 如果onTouchEvent处理down的时候返回false，则后续事件都不会再交给他处理。而是交由他的父view处理。  
**还有一些要点强烈建议看《Android开发艺术探索》。**

### 9. onTouchListener.onTouch、onTouchEvent、onClickListener.onClick优先级？
优先级其实是处理顺序：onTouchListener.onTouch > onTouchEvent > onClickListener.onClick。  
其中onTouchEvent是否被调用取决于onTouchListener.onTouch的返回值。onClickListener是在onTouchEvent里处理up事件的时候调用的。  
### 10. 滑动冲突如何解决？  
**两种方式**    
1.外部拦截法。   
父容器改动：

	public boolean onInterceptTouchEvent(MotionEvent event) {
		boolean intercepted = false;
		int x = (int) event.getX();
		int y = (int) event.getY();
		switch (event.getAction()) {
		    case MotionEvent.ACTION_DOWN:
			intercepted = false;
			break;
		    case MotionEvent.ACTION_MOVE:
			if (父容器需要当前点击事件) {
			    intercepted = true;
			} else {
			    intercepted = false;
			}
			break;
		    case MotionEvent.ACTION_UP:
			intercepted = false;
			break;
		    default:
			break;
		}
		mLastXIntercept = x;
		mLastYIntercept = y;
		return intercepted;
	}

注意，down事件返回false是因为一旦拦截，后续事件都会交给自己处理，返回false可以给子view处理事件的机会，毕竟我们只按需拦截move事件。  
一旦拦截了一个move事件，后续事件都会被自己处理。up也要返回false，否则子view没法触发onClick事件。  
2.内部拦截法。  
子容器的改动：

	public boolean dispatchTouchEvent(MotionEvent event) {
		int x = (int) event.getX();
		int y = (int) event.getY();
		switch (event.getAction()) {
		    case MotionEvent.ACTION_DOWN:
			parent.requestDisallowInterceptTouchEvent(true);
			break;
		    case MotionEvent.ACTION_MOVE:
			int deltaX = x - mLastX;
			int deltaY = y - mLastY;
			if (父容器需要此类点击事件)){
			parent.requestDisallowInterceptTouchEvent(false);
		    }
		    break;
		    case MotionEvent.ACTION_UP:
			break;
		    default:
			break;
		}
		mLastX = x;
		mLastY = y;
		return super.dispatchTouchEvent(event);
    	}

父容器的改动：  

	public boolean onInterceptTouchEvent(MotionEvent event) {
		int action = event.getAction();
		if(action == MotionEvent.ACTION_DOWN) {
			return false;
		} else {
			return true;
		}
	}

主要是子view通过requestDisallowInterceptTouchEvent来决定父view是否拦截事件，但是这个方法对于down事件没有用（因为父view会在down到来的时候重置disallow标记位），所以需要父view单独处理down事件。

### 11. Fragment通信方式？
### 12. MVC、MVP、MVVM？
### 13. Databinding原理？
### 14. LiveData原理？
LiveData **是一个可观察的数据持有者，并且能够感知组件的生命周期。** 也就是说，如果组件处于DESTROY状态，则它不会受到通知。   
先看一下LiveData的常规用法：

	public class MainActivity extends AppCompatActivity {
   	private static final String TAG="MainActivity";
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		MutableLiveData<String> mutableLiveData  = new MutableLiveData<>();
		mutableLiveData.observe(this, new Observer<String>() {//1
		    @Override
		    public void onChanged(@Nullable final String s) {
			Log.d(TAG, "onChanged:"+s);
		    }
		});
		mutableLiveData.postValue("Android进阶三部曲");//2
	    }
	}
	
1. oberve里会首先检查owner的状态，如果是DETROYED，则直接return。
2. 

### 15. ViewModel原理？

## 源码/三方库
### 1. SharedPreferences是不是进程安全的？
不是。但是有一个已经废弃的多进程模式 MODE_MULTI_PROCESS。 MODE_MULTI_PROCESS的时候，会在获取sp对象后进行一个判断：  
如果   
  mDiskWritesInFlight>0，也就是当前有未完成的写入磁盘的任务；  
  文件上次操作的时间和大小（可能是另一个进程）和缓存的本进程上次时间和大小对不上；  
就重新从磁盘读取并解析xml。但是这并不能保证实时性，进程A读取完，进程B仍然有可能更新文件。  

**SharedPreferences原理**  
知识点1： 获取SharedPreferences对象的方法--Context.getSharedPreferences(String name, int mode)。  
要点：  
1. 每个路径path对应的File会有缓存，file对应的SharedPreferencesImpl也有缓存。  
2. 通过path获取到file，最终再通过file获取到SharedPreferencesImpl对象，这个过程是线程安全的。  
3. 有一个多进程模式，但是已经被弃用，谷歌明确表示此模式不可靠。具体实现是通过获取文件当前的时间戳和大小与上次的比较来判断是否需要重新从磁盘加载file。  

知识点2：SharedPreferencesImpl。  
1. 构造函数：  
    a. 或根据当前file创建一个灾备文件。  
    b. mMap：用于存储从file解析出来的键值对。  
    c. 加载并解析xml键值对，此过程是在新的线程里异步进行的。  

2. 异步加载方法：loadFromDisk。  
    a. 如果灾备文件存在，就删除原文件，把灾备文件命名为原文件。  
    b. 加载/解析键值对，存入mMap。记录文件修改时间、文件大小。  
    c. 此过程加锁，load完会notifyAll。(对象锁A)  

知识点3：getString(String key, @Nullable String defValue)。  
1. 加锁，await等待load过程结束，线程安全。（对象锁A）  
2. 直接操作内存mMap中的值。  

知识点4：putString(String key, @Nullable String value)。  
1. 定义在内部类Editor里。  
2. 并不是直接操作mMap，Editor自带一个mModified，用于存储写操作的键值对。  
3. 加锁。(对象锁B)  

知识点5：commit()  
1. 首先把mModified合并到mMap里。  
    a. 有一个重要参数叫mDiskWritesInFlight，代表“此时需要将数据写入磁盘，但还未处理或未处理完成的次数”，唯独在合并之前会+1。  
2. 调用enqueueDiskWrite写入到磁盘上。此方法用一个countDownLatch同步等待写入过程。写入过程完成，才会conuntDown，在commit里往下走。  
    a. 写入磁盘加锁（对象锁C）。  
    b. 写入后，mDiskWritesInFlight会减1。  
    c. 重要：对于commit而言，如果只有一个commit请求，那就在当前线程处理写入过程；如果有多个，那就放在QueuedWork里处理。  
    d. 写入磁盘的时候，会对老文件进行灾备（重命名），然后一次性写入所有数据。如果成功就删除灾备文件，并记录时间、大小；否则删除这个半成品。  


知识点6：apply()  
1. 首先把mModified合并到mMap里。合并过程和commit没区别。  
2. 注意！这里countDownLatch.await放在了一个awaitCommit的runnable里。最终放在QueueWork里。  
3. 同时，enqueueDiskWrite里的磁盘写入工作，也是放在QueueWork里。  

**关键词**：  
1. 全量从磁盘读取xml键值对，放入内存。  
2. 读写分离，不共用一把锁。  
3. put操作先更新mModified，commit/apply后，合并mMap，更新磁盘。  
4. apply是异步更新，但是也会有anr，具体原因参考：https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484387&idx=1&sn=e3c8d6ef52520c51b5e07306d9750e70&scene=21#wechat_redirect  

参考文章：  
https://juejin.cn/post/6844903758355234824#comment  
https://juejin.cn/post/6884505736836022280  

## kotlin
### 1. 扩展函数和扩展属性？
扩展具体用法：  

	fun 类名.方法名([参数 1, 参数 2, ...]): 返回类型 {
	  方法体
	}

本质上是生成一个静态类（以扩展方法所在.kt文件命名），生成一个类的静态方法，参数1是被扩展类的实例，后续参数是自定义的参数。
扩展属性具体用法：

	val 类名.属性名: 类型
	    get() {
		方法体
	    }
	
	var 类名.属性名: 类型
	    get() {
		方法体
	    }
	    set(value) {
		方法体
	    }

扩展属性和扩展方法的原理一样，只不过扩展属性的方法名是自动生成的getter和setter。  
参考文章：https://juejin.cn/post/6844904170609180680
