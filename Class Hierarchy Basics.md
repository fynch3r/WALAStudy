ClassHierachy 对象是定义分析范围（正在分析的程序）的IClass对象的的集合。

# ClassLoaders
Java 语言为类设定了命名空间：classloaders，WALA 模仿了这种形式进行实现：
每个 IClass 属于一个 IClassLoader，同时每个 TypeReference 指定了 ClassLoaderReference
除非在分析 J2EE 或特殊目标，其余情况下， WALA 定义一个类层次会用到以下两种类加载器：
- Application: 负责加载应用自身代码
- Primordial: 负责加载 J2SE 代码，例如包名符合 java.* 格式的类
WALA 中的 classloader 同样遵循委派制，如果 Application classloader 找不到特定类，会委派给 Primordial classloader 加载。
如果你想知道某个类是由哪个类加载负责的，可以通过 `ClassHierarchy.lookupClass` 获取类名对应的 IClass 对象：
```java
cha.lookupClass(TypeReference.findOrCreate(ClassLoaderReference.Application,"className");
```
查询结果：`<Application,Lcom/fynch3r/walatest/Behavior>`
当前的 ClassHierarchy 实现是故意可变的。在对合成类建模时，我们偶尔会根据分析结果动态创建新类，特别是针对J2EE。

例如求解方法：
IMethod iMethod = cha.resolveMethod(MethodReference.findOrCreate(ClassLoaderReference.Application,"Lcom/fynch3r/walatest/Father","drink","()VLcom.fynch3r.walatest.Father;"));

# 生成 ClassHierarchy 

ClassHierarchy 生成方法由工厂类 ClassHierarchyFactory 提供，以分析域对象为构建原料，这里推荐使用 makeWithRoot 方法构建类层次对象：
```java
ClassHierarchy cha = ClassHierarchyFactory.makeWithRoot(scope);
```
在缺失了某些分析所需的类时， makeWithRoot 方法会尽可能地为依赖这些缺失类的方法添加“Root”， 即认为 java.lang.Object 为这些类的父类。
