## 打包流程？
1. 根据AIDL生成java文件；生成BuildConfig.java。
2. 合并Resources、assets、manifest、so等资源文件。  
   使用AAPT2，编译资源文件。除了assets和raw文件。产物是resources.arsc和R.java。R.java里保存了资源id。
3. java编译器编译.java文件。kotlin编译器编译kotlin文件。产物是.class字节码。注解处理器APT,KAPT也是在这个阶段运行，处理CLASS生命周期的注解。
4. Class文件打包成DEX。用R8工具。
5. apkbuilder或者zipflinger将以上资源、dex打包成apk。
6. zipalign对齐处理，提高资源访问效率。
7. 签名

## Gradle是干什么的？
Gradle是一个基于Apache Ant和Apache Maven概念的项目自动化构建开源工具。  
Gradle由不同的task组成不同的构建任务。
### Gradle的三个阶段
1. 准备阶段：  
   执行GradleMain.main();
   创建和注册GlobalScopeService;
   注册META-INF下的服务；
   执行DefaultGradleLauncher.executeTask开始构建。
2. 启动阶段；
3. 构建阶段；
   a. loadsetting阶段：解析settings文件和properties文件；
   b. configure阶段：创建project实例；解析project参数；配置project；**编译执行build.gradle**；
   c. 给root project apply默认插件，注册默认任务；
   d. 处理过滤Task，构建TaskGraph；
   e. 执行task；
   f. finish
## Gradle Task是什么？
Task会构成有向无环图TaskGraph.
## Gradle插件是干什么的？
## Debug和Release状态的不同？
## 5. 