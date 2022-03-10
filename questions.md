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
