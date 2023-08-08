# Overview
WALA 提供了一系列用于程序分析的工具库。

典型的 WALA 框架将通过以下顺序使用库来执行过程间分析：
- 构建 ClassHierarchy：将程序（字节码）读入内存，并解析程序的基本信息。一个 ClassHierarchy 对象表示了待分析代码的全部。
- 构建 CallGraph：WALA 使用 on-the-fly 的方式构建调用图，并在图上进行指针分析，以解动态 dispatch 的调用，并构建由 CGNode 构成的调用图，图上反映的是可能存在的调用结构。
- 对调用图进行一些结果的分析。

WALA 可以进行多种类型的分析。几乎所有的客户都希望浏览 WALA IR，它对特定方法的指令和控制流进行编码。具体来说，IR 是 SSA 形式的指令，由 IR 构成不可变的控制流图。

# WALA 结构
## com.ibm.wala.core
WALA 的核心代码分析库
## com.ibm.wala.core.tests
WALA 测试集
## com.ibm.wala.shrike
WALA IR 名字为：Rob O'Callahan's  Shrike
Shrike 主要分为两种格式：
- ShrikeCT：直接操作类文件的工具
- ShrikeBT：JVM 字节码的一种优雅形式，去除了大部分不相关的信息

WALA 依靠 Shrike 去理解字节码，Shrike 也适用于对代码进行插桩。
