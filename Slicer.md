WALA 包含一个切片器，它基于系统依赖图中的 context-sensitive的可达性拓扑搜索。

# Driver
PDFSlicer 提供了一个简单的驱动程序来计算切片并将其可视化为 PDF。

# API
如果要以编程方式访问 Slicer，您可能会使用 Slicer 类中的 computeForwardSlice 或 computeBackwardSlice  方法。这些方法需要切片的 Statement 对象作为参数。 Statement 对象代表系统依赖图 (SDG) 中的一个节点。 Statement 表示来自 SSA IR 的指令或插入到模型参数传递的额外节点。 要创建表示普通 SSA IR 指令的 Statement，请使用 NormalStatement 类； 这个类通过它在 IR getInstructions() 数组中的索引来识别一条指令。

# Statement Types
IR 中的语句类型：
- Normal statements 表示 IR 中的 SSA 类型语句；
- PARAM_CALLER 和 PARAM_CALLEE 是模型传递过程中的额外节点。当一个参数的值（value）从 caller 调用 callee 的过程发生了传递，那么会产生如下行为：
  - value 从 caller SSA IR 中的 variable → caller 端的 PARAM_CALLER 语句；
  - caller 端的 PARAM_CALLER 语句 → callee 端的 PARAM_CALLEE 语句；
  - callee 端的 PARAM_CALLEE 语句 → caller 端的 SSA IR 中的语句；
- RETURN_CALLER 和 RETURN_CALLEE 提供类似功能；
- HEAP_* statements 模型参数传递和返回值表示在方法调用和返回时堆的语句。

# Options

可以通过 DataDependenceOptions 和 ControlDependenceOptions 来自定义切片。
一些分析选项：
- FULL：跟踪所有数据依赖关系
- NO_BASE_PTRS：与 FULL 类似，忽略定义用于间接内存访问的基指针的数据依赖边。
- NO_BASE_NO_HEAP：与 NO_BASE_PTRS 类似，忽略所有数据依赖边 to/from 堆。
- NO_HEAP：与 FULL 类似，忽略所有数据依赖边 to/from 堆。
- NONE：忽略所有数据依赖。
- REFLECTION：与 NO_BASE_NO_HEAP 类似，但忽略源自 checkcast 语句的数据依赖边。 这是从 newInstance 到 casts 的依赖算法。

可用的 ControlDependence 选项包括：
- FULL：跟踪所有控制依赖；
- NONE：忽略所有控制依赖；

# Performance Tips
与指针分析一样，将切片器扩展到现代 Java 程序和库是很困难的（请参阅瘦切片论文中的进一步讨论）。 配置越精确，跟踪的依赖越多，可扩展性就越难； 

例如，在跟踪基于堆的数据依赖关系时进行切片就不能很好地扩展。 为了提高性能，使用所需的最小依赖性进行切片（例如，NO_HEAP 用于数据流，NONE 用于控制流）。 另请参阅指针分析可扩展性提示，因为它们也可以提供帮助（例如，更多排除项也有助于切片器可扩展性）。

# Thin Slicing
为了进行 context-sensitive 的 Thin Slicing，我们可以用：
DataDependenceOptions.NO_BASE_PTRS 和 ControlDependence.NONE 选项。
具体看 CISlicer 类

# Example 
为了计算切片，我们需要程序的 CallGraph 和一个可计算的 PointerAnalysis 。
我们还需要一些程序语句作为种子语句。
下面代码说明了这些必要步骤，这个例子中的种子语句是在public static void main(String args[]) 方法中调用 println() 方法的第一个对象实际例子。
一旦计算出切片，我们就可以对语句进行更多分析（包括确定原始源代码行号）。 

```java
public static void doSlicing(String appJarPath) throws Exception {
        AnalysisScope scope = AnalysisScopeReader.instance.makeJavaBinaryAnalysisScope(appJarPath, new File(CallGraphTestUtil.REGRESSION_EXCLUSIONS));
        ClassHierarchy cha = ClassHierarchyFactory.make(scope);

        Iterable<Entrypoint> entrypoints = Util.makeMainEntrypoints(scope, cha);
        AnalysisOptions options = new AnalysisOptions(scope, entrypoints);

        SSAPropagationCallGraphBuilder callGraphBuilder = Util.makeZeroCFABuilder(Language.JAVA, options, new AnalysisCacheImpl(), cha);
        CallGraph callGraph = callGraphBuilder.makeCallGraph(options);
        PointerAnalysis<InstanceKey> pointerAnalysis = callGraphBuilder.getPointerAnalysis();

        // find seed statement
        Statement statement = findCallTo(findMainMethod(callGraph), "println");
        Collection<Statement> slice;

        // context-sensitive traditional slice
        slice = Slicer.computeBackwardSlice(statement, callGraph, pointerAnalysis);
        dumpSlice(slice);
}

public static CGNode findMainMethod(CallGraph callGraph) {
        Descriptor descriptor = Descriptor.findOrCreateUTF8("([Ljava/lang/String;)V");
        Atom name = Atom.findOrCreateUnicodeAtom("main");
        Iterator<CGNode> succNodes = callGraph.getSuccNodes(callGraph.getFakeRootNode());
        while (succNodes.hasNext()) {
            CGNode node = succNodes.next();
            if (node.getMethod().getName().equals(name) && node.getMethod().getDescriptor().equals(descriptor)) {
                return node;
            }
        }
        Assertions.UNREACHABLE("Failed to find main() method.");
        return null;
}

public static Statement findCallTo(CGNode node, String methodName) {
        IR ir = node.getIR();
        Iterator<SSAInstruction> ssaInstructionIterator = ir.iterateAllInstructions();
        while (ssaInstructionIterator.hasNext()) {
            SSAInstruction ssa = ssaInstructionIterator.next();
            if (ssa instanceof SSAAbstractInvokeInstruction) {
                SSAAbstractInvokeInstruction call = (SSAAbstractInvokeInstruction) ssa;
                if (call.getCallSite().getDeclaredTarget().getName().toString().equals(methodName)) {
                    IntSet indices = ir.getCallInstructionIndices(call.getCallSite());
                    Assertions.productionAssertion(indices.size() == 1, "expected 1 but go " + indices.size());
                    return new NormalStatement(node, indices.intIterator().next());
                }
            }
        }
        Assertions.UNREACHABLE("Failed to find call to " + methodName + " in " + node);
        return null;
}


public static void dumpSlice(Collection<Statement> slice) {
        for (Statement statement : slice) {
            System.out.println(statement);
        }
}
```

## Warning: exclusion of copy statement from slice
由于 SSA IR 已经进行了一定程度的优化，因此一些语句（例如简单赋值 (x=y, y=z)）不会出现在 IR 中，这是由于 SSABuilder 类在 SSA 构建期间自动完成的复制传播优化。 事实上，没有SSA赋值指令； 此外，javac 编译器可以自由地进行这些优化，因此这些语句甚至可能不会出现在字节码中。 因此，这些 Java 语句将永远不会出现在切片中。

## Executable slices
WALA 生成的切片不一定是可执行的； WALA 的实现计算了一个闭包切片（不同之处请参见此处）。 计算可执行切片可能需要额外的工作。
