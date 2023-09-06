
## Android的类加载机制
### 类加载器和java不同
1. 类加载器 PathClassLoader：只能加载应用包内的dex
2. 类加载器 DexClassLoader：可以加载任意位置的 dex
3. 类加载器 BaseDexClassLoader：持有 DexPathList，结构如下, BaseDexClassLoader（DexPathList（Element数组（DexFile（多个Class））））

### DexPathList
DexPathList: 将 dex 文件转换成 element 存入数组（dexElements）,findClass()是遍历Elements并进行类名匹配。

### Android类加载过程
Dex 文件在类加载器中被包装成 Element，Element以数组形式被DexPathList 持有，加载类时通过遍历Element数组进行类名匹配查找，只要把新的Dex文件插入到 Element头部即可实现热修（反射）。

### 双亲委托
类加载器加载类时，将加载请求逐级向上委托，直到BootStrapClassloader，真正的加载从顶层开始，逐级向下查找。避免了类重复加载，以及安全，因为系统类总是由上层类加载器加载，无法通过自定义篡改

## 插件化

## 热修复

## Android版本差异