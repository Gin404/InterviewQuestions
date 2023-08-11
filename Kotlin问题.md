## 函数式编程的要点
1. 函数是一等公民，可以像变量一样传递、返回，在函数中嵌套函数。
2. 引用透明性。韩式运行不依赖于外部变量或者状态，只依赖于输入的参数，任何时候只要参数相同，调用函数所得的返回值总是相同。
3. 权责让渡。将低层次的细节的控制权交给运行时，开发者只关注高层次的细节。

## 函数式解法相对于命令式编程
1. 避免可变状态。
2. 程序是表达式和可变关系，而不是命令。
3. 建立在数学的直觉上。
4. **低层次的细节控制交给运行时，开发者只关注高层次的细节。**

### 代替遍历  
1. 筛选filter
2. 映射map
3. 压缩reduce/fold

### 用闭包实现记忆

### 函数作为参数进行递归
tailrec关键字，尾递归优化防止栈溢出

### 柯里化
把一个多参函数，转化为一连串的单参函数。用lambda表达式的特征。

### 函数式编程体现了哪些设计模式
1. 单一职责原则，面向对象里是对象包含的职责越少越好，函数式编程里一个函数就可以完成单一职责。
2. 开闭原则，函数不能被修改，但是可以被扩展和组合。
3. 依赖倒置，kotlin高阶函数的实现方式就是接口的实现，所以参数相同返回值类型相同，就可以替换。
4. 里氏代换，函数式编程需要关注的就是参数和返回值。
5. 最少知道，状态的权责让渡，不需要关注细节。

## Kotlin如何支持Java中没有的特性

## 扩展函数和扩展属性
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


## 协程
协程可以定义为一个线程框架或者是一种并发设计模式**协程的核心竞争力是简化异步并发任务。**
### suspend关键字和挂起函数
被suspend关键字修饰的函数叫做挂起函数。

    val user = getUserInfo()
    val mScope = MainScope() //等于 ContextScope(SupervisorJob() + Dispatchers.Main)
    ...
    suspend fun getUserInfo(): String {
        mScope.launch {
            delay(1000L)
        }
        return "BoyCoder"
    }

上面第一行，等于号左边是主线程，右边是IO线程。

### suspend的本质
getUserInfo反编译为java代码如下：

    public static final Object getUserInfo(Continuation $completion) {
        ...
        return "BoyCoder";
    }

suspend的本质就是Callback，上面的Continuation就是Callback。Continuation的结构如下：

    public interface Continuation<in T> {
    public val context: CoroutineContext
        //      相当于 onSuccess     结果   
        //                 ↓         ↓
        public fun resumeWith(result: Result<T>)
    }

**Continuation就是一个带有泛型的Callback，CoroutineContext是协程的上下文。** 从挂起函数转换成CallBack 函数的过程，被称为：CPS 转换(Continuation-Passing-Style Transformation)。 

### 挂起函数不一定被挂起
suspend修饰的函数里面包含其他的挂起函数他才真正的被挂起，比如上面的withContext。否则suspend是多余的。  
挂起函数CPS转换后根据返回值判断是否被挂起，比如返回的是CoroutineSingletons.COROUTINE_SUSPENDED代表被挂起。
CPS转换后返回值是 Any? 是为了适配各种情况。

### Continuation
Continuation的含义就是接下来要执行的代码。**那CPS就是将程序接下来要执行的代码进行传递的一种模式。**


### CoroutineContext协程上下文
协程上下文包含了实现协程的各个元素。包括Job、CoroutineDispatcher、CoroutineName、CoroutineExceptionHandler。他们都继承自Element，以kv的形式维护在上下文中。  
1. Job: 控制协程的生命周期；
2. CoroutineDispatcher: 向合适的线程分发任务；
3. CoroutineName: 协程的名称，调试的时候很有用；
4. CoroutineExceptionHandler: 处理未被捕捉的异常。

比如上面构造ContextScope传的就是Job+CoroutineDispatcher，其中加号是CoroutineContext的运算符重载。  

#### Job
用于处理协程的任务，launch或async的返回值，是协程的唯一标识，如果不传，会生成默认的CompletableJob实例。  
同时也管理协程的生命周期。包含一系列状态：新创建 (New)、活跃 (Active)、完成中 (Completing)、已完成 (Completed)、取消中 (Cancelling) 和已取消 (Cancelled)。  
详见android官方注释。

#### CoroutineDispatcher
CoroutineDispatcher 定义了 Coroutine 执行的线程。Dispatchers 是一个标准库中帮我们封装了切换线程的帮助类，可以简单理解为一个线程池。  
**Dispatchers.Main**: Android上的主线程。用来更新ui和一些轻量任务。调用suspend函数、ui函数、更新LiveData。  
**Dispatchers.IO**: 非主线程，专为IO进行了优化。用于执行数据库、文件、网络等IO操作。  
**Dispatchers.Default**: 专为cpu密集型任务进行了优化。比如数组排序、JSON数据解析、处理差异判断。  

### CoroutineScope协程作用域
定义协程必须指定其 CoroutineScope 。CoroutineScope 可以对协程进行追踪，即使协程被挂起也是如此。  
相当于给协程圈定了一个运行范围。官方提供的作用域有：  
**GlobalScope**: 不推荐使用。由于 GlobalScope 对象没有和应用生命周期组件相关联，需要自己管理 GlobalScope 所创建的 Coroutine，且GlobalScope的生命周期是 process 级别的，所以一般而言我们不推荐使用 GlobalScope 来创建 Coroutine。  
**runBlocking{}**: 主要用于测试。运行一个新的协程并且阻塞当前可中断的线程直至协程执行完成，该函数不应从一个协程中使用，该函数被设计用于桥接普通阻塞代码到以挂起风格（suspending style）编写的库，以用于主函数与测试。该函数主要用于测试，不适用于日常开发，该协程会阻塞当前线程直到协程体执行完成。  
**MainScope()**: 可用于开发。该函数是一个顶层函数，用于返回一个上下文是SupervisorJob() + Dispatchers.Main的作用域，该作用域常被使用在Activity/Fragment，并且在界面销毁时要调用fun CoroutineScope.cancel(cause: CancellationException? = null)对协程进行取消。  
**LifecycleOwner.lifecycleScope**: 推荐使用。绑定Lifecycle生命周期，在destroy的时候自动销毁。常用语Activity/Fragment。  
**ViewModel.viewModelScope**: 推荐使用。它能够在此ViewModel销毁时自动取消，同样不会造成协程泄漏。  

### 协程的取消和异常
协程之间存在层级关系，一个Job会包含很多子Job。  
如果父Job是一般的Job，在子Job产生异常后，会将异常传播给父Job。父Job会取消所有的子Job。  
如果父Job是SupervisorJob，在子Job产生异常后，会自己捕获异常，不会传播给父Job。所以不会影响其他Job。  

参考：https://rengwuxian.com/kotlin-coroutines-2/  
https://juejin.cn/post/6950616789390721037#heading-29

