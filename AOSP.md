# Android Open Source Project 知识点

## Dalvik和ART


## Android 系统启动过程

第一步，启动 bootloader 

第二部，启动 init 进程。init 进程是用户空间的第一个进程，

init 进程会 fork 出 zygote进程，zygote 进程会在 native 中启动 Java 虚拟机，注册 jni 函数，预加载系统资源，进入 Java 世界，

zygote 进出会启动 SystemServer，进入 socket loop 循环。

SystemServer 



## 谈谈对 Zygote 的理解

1.zygote 的作用：启动 SystemServer ，孵化应用进程

2.zygote 的启动流程：init 进程解析 init.rc 文件，zygote 就在 init.rc 文件中定义，系统会加载 /system/bin/app_process 二进制文件，执行 zygote 的启动。

3.zygote 的工作原理：如何孵化进程？通过 linux 的 fork 函数去创建进程，fork 函数会返回两次，pid 等于0 是子进程，可以出来子进程的逻辑。pid 大于 0，是子进程的 pid，可以处理父进程的逻辑。

怎么和其他进程通信？zygote 启动后，会进入 loop 循环，并开启 socket，等待其他进程和他通行。zygote 接收到 socket 请求后，会执行 runOnce 方法，先读取请求的参数列表，调用 fork 函数，fork 子进程。

## 应用进程如何启动的？
应用进程启动不代表应用的UI已经展示出来了。在启动Activity之前，要首先保证应用进程的启动。Zygote在Java框架层有一个Server端的Socket，这个Socket就是用来等待AMS请求Zygote来创建新的应用进程的。  
Zygote启动后创建了虚拟机实例、binder线程池和消息循环，所以Zygote fork出的应用进程，也就自带这些机制了。可以方便的使用binder和处理消息。  


## Activity启动流程？
1. Launcher通过start
2. 
## Service启动流程？
## ContentProvider启动流程？
##  