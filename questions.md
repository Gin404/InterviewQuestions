# 知识点
## java基础
### 1. 反射是什么，在哪里用到，怎么利用反射创建一个对象?
### 2. 对象加载的过程，属性先加载还是方法先加载?
### 3. 垃圾回收机制与jvm内存结构。
### 4. Hashmap/SparceArray/, ConcurrentHashMap实现。
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
### 4. http与https有什么区别?
### 5. 浏览器输入一个域名发生了什么？

## Android基础
### 1. Handler原理？
### 2. 同步屏障？Choreographer？
### 3. IdleHandler？调用时机？用处？
### 4. HandlerThread？
### 5. invalidate/postInvalidate/requestLayout的区别？*
### 6. Fragment:replace和add的区别？show和hide？commit和commitAllowStateloss？
- fragment容器为空的时候，replace和add没有区别。
        
- 如果fragment容器有一个Fragment A。
        通过add添加Fragment B，生命周期变化如下：
        A: 无变化；
        B: onAttach 直到 onResume。
        此时，remove B，生命周期变化如下：
        A: 无变化；
        B: onPause 直到 onDetatch。
        （值得注意的是，如果同时调用了addToBackStack，B只会走到onDestroyView，此时通过findFragmentByTag依然可以找到FragmentB的实例。）

        通过replace添加Fragment B，生命周期变化如下：
        A: onPause 直到 onDetatch；
        B: onAttach 直到 onResume。
        在2处，通过findFragmentByTag可以找到A的实例。remove B，生命周期状态如下，
        A: 无变化；
        B: onPause 直到 onDetatch；
        (同样，如果同时调用了addToBackStack，B只会走到onDestroyView.)
        
       
### 7. Fragment通信方式？
### 8. MVC、MVP、MVVM？
### 9. Databinding原理？ViewModel原理？

## 源码/三方库
### 1. SharedPreferences是不是进程安全的？
    答：不是。但是有一个已经废弃的多进程模式 MODE_MULTI_PROCESS。 MODE_MULTI_PROCESS的时候，会在获取sp对象后进行一个判断：
        如果 
          mDiskWritesInFlight>0，也就是当前有未完成的写入磁盘的任务；
          文件上次操作的时间和大小（可能是另一个进程）和缓存的本进程上次时间和大小对不上；
        就重新从磁盘读取并解析xml。但是这并不能保证实时性，进程A读取完，进程B仍然有可能更新文件。
    
    SharedPreferences原理
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
    
    关键词：
    1. 全量从磁盘读取xml键值对，放入内存。
    2. 读写分离，不共用一把锁。
    3. put操作先更新mModified，commit/apply后，合并mMap，更新磁盘。
    4. apply是异步更新，但是也会有anr，具体原因参考：https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247484387&idx=1&sn=e3c8d6ef52520c51b5e07306d9750e70&scene=21#wechat_redirect

    参考文章：
    https://juejin.cn/post/6844903758355234824#comment
    https://juejin.cn/post/6884505736836022280

      
