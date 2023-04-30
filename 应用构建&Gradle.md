### 1. 打包流程？
1. 根据AIDL生成java文件；生成BuildConfig.java。
2. 合并Resources、assets、manifest、so等资源文件。  
   使用AAPT2，编译资源文件。除了assets和raw文件。产物是resources.arsc和R.java。R.java里保存了资源id。
3. java编译器编译.java文件。kotlin编译器编译kotlin文件。产物是.class字节码。注解处理器APT,KAPT也是在这个阶段运行，处理CLASS生命周期的注解。
4. Class文件打包成DEX。用R8工具。
5. apkbuilder或者zipflinger将以上资源、dex打包成apk。
6. zipalign对齐处理，提高资源访问效率。
7. 签名

### 2. Gradle是干什么的？
### 3. Gradle插件是干什么的？
### 4. Debug和Release状态的不同？
### 5. 