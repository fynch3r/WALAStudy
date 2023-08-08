# Overview

WALA 提供了一个流不敏感的 Anderson 风格的指针分析框架，可以通过多种不同的方式实现上下文敏感的指针分析。

WALA 所有的指针分析实现都以构建 on-the-fly 函数调用图。

context-sensitivity 主要是体现在两个维度：
- HeapModel：指针分析模型是如何抽象指针和堆位置的？
- ContextSelector：Call Graph 的构建过程中如何跟据上下文克隆方法？

可以通过自定义堆模型和上下文选择器来定义自己的指针分析策略。 
此外，WALA 提供了许多内置策略。

# Heap Model 堆模型
一个 HeapModel 对象告诉 WALA 指针分析框架如何抽象指针和堆位置。关键类：
- PointerKey ：指针的抽象
- InstanceKey ：堆位置的抽象

举个例子，一个PointerKey 可以表示一个 local variable 、一个 static field、一个 instance field 等等。

一个PointerKey 是具体程序的若干指针的等价类的名称，这些指针在抽象中被整合到一个抽象对象中，也就是 PointerKey。

InstanceKey 可以表示一个具体类型的所有实例化对象，或者由一个特定的 allocation-site 创建的所有对象，或者在一个特定的 context 中特定 allocation-site 创建的所有对象。

HeapModel 为指针分析提供回调（call-backs），用来在分析期间创建 PointerKeys 和 InstanceKeys，可以通过提供自己的 HeapModel 来自定义策略。

# Heap Graph 堆图
一个 HeapGraph 对象提供了一种方便的方式用来指导指针分析的结果。HeapGraph 由 PointerKey 和 InstanceKey 组成。

如果指针分析表示一个 P 可能指向 I ，那么 在 HeapGraph 存在一条边，从 PointerKeyP 指向 InstanceKey I。

从  InstanceKey I  指向  PointerKeyP  ，当且仅当：
- P 代表一个具体对象的具体属性，这个具体对象由 P 定义；
- P 代表数组内容，这个数组实例由 I 代表

举个例子，给你一个 HeapGraph 对象 h，假设你想知道所有和PointerKey  p 构成别名关系的PointerKey 指针对象们。

你需要首先用h.getSuccNodes(p) 找出所有 p 可能指向的全部 InstanceKey I 。对于每个 InstanceKey I ，使用h.getPredNodes(i) 找到所有i的前驱指针，这些前驱指针可能就是 p 的别名指针。

# Context Selector 上下文选择器
在 Call Graph 中的每个 node 都表示特定 context 的一个方法。ContextSelector 控制调用图构建 context 的策略。

最简单的 context 就是Everywhere.EVERYWHERE ，它表示一个方法的单个全局上下文。

其他 context 策略可以表示 call-string contexts，命名返回对象的上下文以实现对象敏感性或其他变体。

你可以通过提供自定义的 ContextSelector 对象来自定义上下文敏感策略。


回想一下，每个调用图节点都代表特定上下文中的一个方法。 ContextSelector 对象控制调用图构建上下文的策略。

最简单的 Context 是 Everywhere.EVERYWHERE，它表示方法的单个全局上下文。

其他上下文策略可以表示调用字符串上下文、命名接收者对象的上下文以实现对象敏感性或其他变体。

您可以通过提供自定义 ContextSelector 对象来自定义上下文敏感策略。

# Entry Points 入口点
例如我们有一个方法作为公共入口点：
```java
public static void foo(Object x,Object y)
```
指针分析将如何处理入口点参数？具体来说，指针分析会将哪些 InstanceKey 抽象实例化作为参数 x 和 y 的初始内容？

WALA 使用 Entrypoint 接口来指导入口点策略，这个接口中的关键方法：
```
public abstract TypeReference[] getParameterTypes(int i);
```
这方法会返回一组类型，是当前入口点的第 i 个参数的声明类型结果。

举个例子，假设我们给 foo 方法构建一个 EntryPoint 对象：
```java
getParameterType(0) = { java.lang.Object, java.lang.String }
getParameterType(1) = { java.lang.Integer }
```
然后从逻辑上来看，指针分析对 foo 方法的调用建模如下：
```java
Object o1 = new java.lang.Object();
Object o2 = new java.lang.String();
Object o3 = non-determinstic-choice ? o1:o2;
Object o4 = new java.lang.Integer();
foo(o3,o4);
```
WALA 会造出一个 FakeRootMethod 用来模拟调用所有的入口点方法。

WALA 会在 FakeRootMethod 中生成代表上述代码的“合成 IR 指令”。

基于此 IR ，指针分析将根据管理 HeapModel 来创建 InstanceKey 的抽象，就好像程序在 FakeRootMethod 中遇到指定类型的分配一样。

默认情况下，大多数 WALA 指针分析实现示例使用 Entrypoint 的 DefaultEntrypoint 实现。在这个实现中，getParameterTypes(i) 返回一个具有单个元素的数组，它是第 i 个参数的声明类型。 WALA 的大多数生产级客户端都提供了 Entrypoint 的自定义实现，专门针对正在分析的框架和客户端的类型。

# Built-in Policies 内置策略

下面介绍几个 WALA 指针分析的内置策略：

## ZeroCFA

最简单、资源消耗最小的 context-insentive 的指针分析，可以直接使用 Util.makeZeroCFABuilder() 来生成 Call Graph。

- HeapModel 为每个具体类型（concrete type）分配了一个 InstanceKey ，也就是说，特定类型的所有对象都由单个抽象对象表示；
- ContextSelector 为每个方法使用单个全局上下文，除了一些处理反射的特殊情况（后面会提到）；


## ZeroOneCFA

标准 Anderson 风格的指针分析的近似值，使用 allocation-site 来命名抽象对象。

HeapModel 为每个 allocation-site 分配了一个InstanceKey。

但是，ZeroOneCFA 有一些变体，这取决于 ZeroXInstanceKeys 类提供的对 heap model 的一些优化。有关更多详细信息，请参阅源代码。简单来说：
- VanillaZeroOneCFA 关闭所有的优化功能，每个 allocation 都单独处理；（Util.makeVanillaZeroOneCFA）
- ZeroOneCFA 开启以下优化：（Util.makeZeroOneCFABuilder）
  - SMUSH_STRINGS：单独的 String 或 StringBuilder 的 allocation-site 不消除歧义。一个InstanceKey 可以表示所有的 String / StringBuilder 对象。
  - SMUSH_THROWABLES：单个 Exception 对象根据类型区分，而不是 allcoation-site。
  - SMUSH_PRIMITIVE_HOLDERS：如果一个类没有引用类型的字段，那么只用一个 InstanceKey 来表示该类型的所有对象。
  - SMUSH_MANY：如果单个方法中有超过 k 个（当前 k = 25）特定类型的 allocation-site，则所有这些站点都有一个 InstanceKey 来表示。例如，一个库类初始化方法可能会分配 10000 个 Font 对象，这种优化不会区分。

## ZeroOneContainerCFA
详见 Util.makeZeroOneContainerCFABuilder() 

ZeroOneContainerCFA 拓展了 ZeroOneCFA 策略，对集合对象施加 object-sensitivity。

对于任意一个集合对象的 allocation-site，allocation-site 由元组命名，该元组扩展到最外层的封闭集合分配。该策略可能相对昂贵，但可以有效地消除标准集合类的内容的歧义。为了使其生效，需要注意的是 ZeroOneContainer 修改了 ContextSelector ，以根据返回值对象的 object-sensitive 的名称来克隆方法。ContainerContextSelector 类管理此逻辑，ContainerUtil.isContainer() 决定了哪些类可被视作容器。

ZeroOneContainer 还为某些工厂方法或已知需要此类精度的方法使用一种 call-string 敏感策略，例如System.arraycopy() 方法。ContainerContextSelector.isWellKnownStaticFactory() 确定哪些方法需要这种处理。

## Call-site Sensitivity/ K-CFA
对于 call-site 敏感，每个 CGNode 的 context 由调用链的最后 k 个call sites 构成。WALA 选择调用 CallString 来记录这些 call-site。你可以使用Util.makeNCFABuilder() 来基于 call-site sensitivity 来创建一个 CallGraph builder。对于堆抽象来说，WALA 使用 CGNode 的CallStringContext 作为堆的上下文来限定它，并使用分配站点（AllocationSiteInNode）来表示分配。

## Object Sensitivity/ K-Obj
Object-Sensitivity 被认为是面向对象语言（如 Java）中指针分析的最佳选择。 与 K-CFA 不同，K-Obj 分析使用 k 个对象的 allocation-site 作为上下文元素。 WALA 使用 AllocationString 来记录这些 allocation-site。 使用 Util.makeNObjBuilder() 或 Util.makeVanillaNObjBuilder() 根据 obj-sensitivity 创建调用图构建器。 相关的 Context 信息如下：
- 对于 vitural method invoke，Context 是一个AllocationStringContext 对象，其中包含一个 AllocationString。
- 对于 static method invoke，标准库中一些著名的静态工厂方法，它们的 Context 是 CallerSiteContext ；另外的 case，直接复制最后一个非静态方法的上下文作为被调用方法的 Context。
- 对于一个对象（在分配时固定）：堆的上下文与 CGNode 的上下文相同，allocation-site 的数量相同。

## Contexts for Reflection 关于反射的上下文
为了处理反射这种 case ，WALA 使用了 context-sensitive 策略，即使使用像 ZeroCFA 这样的 context-insensitive 的基本策略。

目前所有内置的指针分析策略其实都敏感地处理了对 Object.clone() 的上下文的调用，使用返回值的具体类型作为上下文。（详见 CloneContextSelector ）同时，我们认为 clone() 在每个 Context 的 IR 都是不同的，用来通过适当的建模相应的返回值类型的方法语义。（详见 CloneInterpreter）。

另外，还有对其他反射构造的重要支持，如 Class.forName()、Class.newInstance()、Method.invoke() 等。有关管理此反射处理的代码，请参阅 ReflectionContextSelector 和 ReflectionContextInterpreter，以及 AnalysisOptions.setReflectionOptions() 调整处理。

## Improving Scalability 拓展性提高
反射的使用和现代库/框架的大小使得将 flow-insensitive 的 point-to 指针分析扩展到现代 Java 程序分析非常困难。例如，在默认设置下，WALA 的指针分析无法处理与 Java 6 标准库链接的任何程序，因为库中存在大量反射。为了提高可扩展性（在某些情况下以一些健全为代价）尝试以下操作：
- 使用 AnalysisScope 排除项来排除您知道与应用程序无关的类/包。排除在传递给 AnalysisScopeReader 的适当方法的排除文件中指定。有关示例，请参阅 Java60RegressionExclusions.txt。
- 分析旧版本的标准库，例如从 Java 1.4 开始。
- 降低指针分析的上下文敏感性（见上文）。
- 通过在用于指针分析的对象 AnalysisOptions 上调用 setReflectionOptions() 来减少 WALA 的反射处理。您可能需要修改以查看哪些设置可以为您的应用程序提供可拓展性。























































