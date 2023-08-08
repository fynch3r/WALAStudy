# Overview
WALA IR 是表示特定方法指令的中央数据结构。
IR用一种接近JVM字节码的语言表示方法的指令，但用一种基于SSA的寄存器传输语言表示，该语言消除了堆栈抽象，而依赖于一组符号寄存器。
IR将指令组织在基本块的控制流图中，这点也是目前编译器教科书中的典型形式。
IR是不可变的；它不支持程序转换，WALA 也不支持从 IR 生成源代码。IR 的设计哲学是可以在生成分析信息的过程中使用，用于支持其他工具，如IDE或编译器。
我们可以用 WALA 中的 shrike 模块在字节码层面进行程序转换，但不能直接从 IR 层面进行转换。
通常，一次程序分析将会建立从 IR 到相关分析信息（例如抽象和数据流）的各种结构和映射。

# IR 基础
WALA IR 类提供特定方法的基于SSA的中间表示。
IR由基本块的控制流图以及一组指令组成。
由于空间效率的历史遗留问题，当前的IR实现有些复杂（未来可能会发生变化）。与其处理底层实现，不如尝试通过其公共接口访问IR。例如，下面是最有用的几种方式：
- IR.iterateAllInstructions(): 以未定义的顺序返回所有指令，用于流不敏感分析。
- IR.getControlFlowGraph(): 返回基于基本块（IBasicBlock）的控制流图 CFG。
- IR.getControlFlowGraph().iterateNodes(): 返回控制流图中节点（基本块）的迭代。
- IBasicBlock.iterateAllInstructions(): 返回特定基本块中的所有指令。

如果关心迭代指令的顺序，我们应该在基本块上循环，然后在每个基本块上迭代指令。
通常来说，我们构造 CallGraph，然后我们可以获取一个方法的特定IR：
- CGNode.getIR()：获取某个 CGNode 对应的 IR；
- AnalysisCache.getIR(IMethod)：构造某个特定 IMethod 的 IR；
但是， WALA 对于同一个 IMethod 生成的 IR 根据上下文不同而有些许区别，毕竟上下文敏感。

# Value Number
IR 中的每个变量都有一个特殊的 id，称为 value number：v1、v2、v3。

按照惯例，一个方法的数值从1开始。所以对于一个非静态方法，v1 代表 this 参数。接下来是该方法的参数（v2，v3，…）。对于一个静态方法，v1代表第一个参数。

大多数 SSAInstructions 指令最多定义一个变量，并使用一定数量的变量。每条指令都带有它所定义和使用的变量的编号，可以通过 getDef() 和 getUses() 访问。实际上，SSAInvokeInstructions 可以额外定义第二个变量，代表异常的返回值。

反过来说，每个变量（value number）将有确切的一个 def 和若干的 use。使用 DefUse 类来找到def 和 use 某个特定变量的指令。如果 DefUse 返回 null 作为一个变量的 def ，那么这个变量代表着一个参数或一个常量。关于常量的信息请见下文。请注意，定义可能是一个 phi statement，更多细节见下文。

在 SSA 翻译过程中，堆栈位置和 locals 变量都被翻译成符号化的虚拟寄存器。对于一个给定的 SSA 变量和指令索引，你可以使用 IR.getLocalNames() 来找出相应的 local 变量的名字（如果有的话）。可以看一下 SSABuilder.SSA2LocalMap，看看 IR 是如何在内部跟踪这些信息的。


# Phi statements
作为 SSA 形式的标准，WALA IR 包含 phi statements，以处理多个定义可能达到某种用途的情况。由于历史原因，这些 phi 语句（SSAPhiInstructions）没有作为正常指令存储在IR中。相反，它们被存储在 IR 的控制流图中的 BasicBlocks 中。要查看一个 IR 中的所有 phi statements，请调用IR.iteratePhis()。要查看某一特定基本块上的 phi statements，请调用BasicBlock.iteratePhis()。从逻辑上讲，基本块中的 phi statements 是在基本块的开头执行的，也就是说，在基本块的任何正常指令之前就执行了。

# Type Inference
类型推断

可以使用 TypeInference 类在程序中为IR中的 value number 探索过程内类型。例如：
```txt
IR ir = ...;
boolean doPrimitives = ...; // infer types for primitive vars?
TypeInference ti = TypeInference.make(ir, doPrimitives);
TypeAbstraction type = ti.getType(vn);
```

# Constant Values
基本类型和字符串常量

每个常量都有一个相应的变量，但在 IR 中没有明确的语句将常量分配给变量。要发现常量，可以使用IR 的 SymbolTable，通过调用 IR.getSymbolTable() 获得。SymbolTable.isConstant() 将返回一个特定的变量是否代表一个常数（传入变量的 value number 作为参数），而 SymbolTable.getConstantValue() 将返回代表的常量值。

# Pi Node(advanced)
pi 赋值（在WALA代码中被称为 "pi节点"）是在Bodik等人的ABCD工作中引入的，是一种直接以SSA形式捕获局部路径条件的技术。例如，如果我们有代码：
```txt
S1: c = v1 != null 
S2: if (c) { return v1; }
```
我们可以把它改写成：
```txt
S1: c = v1 != null 
S2: if (c) { v2 = PI(v1, S1); return v2; }
```
有了新的表示方法，一个对路径不敏感（path-insensitive）的分析可以很容易地推断出：v2永远不会是空的。
要用 pi 节点构建IR，请在相应的 SSAOptions 对象上调用 setPiNodePolicy() ，该对象规定了pi 节点的构建策略。WALA 有两个内置的 pi 节点策略：
- NullTestPiPolicy，它为空值检查添加 pi 节点（如上面的例子）
- InstanceOfPiPolicy，用于类型测试的实例。

和 phi 指令一样，pi 指令存储在 BasicBlocks 上，不是普通的 IR 指令。要遍历一个 IR 中的所有 pi 指令，可以调用 IR.iteratePis()。更有效的是，你可以调用 BasicBlock.iteratePis() 来查看存储在该块中的 pi 指令。请注意，pi 指令在逻辑上是在其所存储的基本块的末端执行的；这与 phi 指令相反。更确切地说，每个 SSAPiInstruction 都与 Control Flow Graph 中的某个边 e 相关联，并被存储在 *e$ 的源块上；该指令只有流经 e 时才会（逻辑上）执行。

对于一个给定的 SSAPiInstruction ，getVal() 告诉我们哪个 value number 被重新命名，而getDef() 告诉我们新的 value number。你如何知道新名称的条件是什么？ getSuccessorBlock() 告诉你将使用新名称的后续块号。假设你有存储 SSAPiInstruction 的块 b，你可以在控制流图中查看 b 的继任者 successor ，找出与继任块编号相对应的 BasicBlock  的 succ。现在，你可以调用 com.ibm.wala.cfg.Util.getTakenSuccessor() 或 com.ibm.wala.cfg.Util.getNotTakenSuccessor() 来计算当 b 结尾的条件为真或假时，是否达到了 succ。要想知道什么条件成立，你需要查看 b 结尾的 SSAConditionalBranchInstruction 和 SSAPiInstruction.getCause() 返回的 "cause "指令；具体的条件是针对 pi 节点策略的。

