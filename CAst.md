WALA 为多种语言设计了通用抽象语法树系统（Common Abstract Syntax Tree，CAst），目前支持 Java 、 JavaScript 两种语言。

有几个构建步骤需要让 CAst 为你工作：
- com.ibm.wala.cast：核心 CAst 系统功能实现；
- com.ibm.wala.cast.test：对 CAst 的测试支持；


# Java 前端
Java 前端目前有两种方式实现：
- Polyglot: 解析 Java 源代码然后生成 AST；
- Eclipse JDT：由 Evan Battaglia @Berkeley 提供支持

# CAst CallGraph Details
## Lexical Scoping
CAst 允许具有嵌套函数的语言以及从嵌套函数访问在闭包函数中声明的变量， 我们称这种访问为词法访问，我们称被嵌套函数访问的变量被暴露。 在这里，我们讨论在 WALA 的指针分析期间如何处理词法访问。 请注意，虽然来自匿名类的词法访问被转换为 Java 字节码，但在使用 CAst 分析 Java 源代码时直接处理此类访问。

## Original Approach
最初，即使对于暴露的变量，WALA 也不遗余力地尝试保持某种 SSA 表示，目的是通过一定程度的流敏感度来提高精度。在这种方法中，对嵌套函数的调用的词法读取必须访问公开变量的正确 SSA 名称，并且来自此类调用的词法写入必须被视为“def”并创建一个新的 SSA 名称。不幸的是，这种方法变得相当复杂，因为分析必须计算调用图信息以确定何时调用嵌套函数。因此，SSA 重命名和调用图构建必须同时执行，这是一个棘手的命题。 JavaScript 中的问题更加复杂，因为嵌套函数可能作为闭包返回，在这种情况下，词法写入操作的是闭包而不是调用堆栈上的局部变量。虽然实现这种技术的代码仍然存在（例如，参见 SSAConversion），但我们已经过渡到一种更简单的方法，我们认为这种方法在实践中应该几乎没有精度损失。

## New Approach
我们的新方法本质上是通过堆分配任何暴露的变量并适当地处理词法访问来工作的。 具体来说：
- 词法访问是通过在从特定语言的 AST 生成 CAst 时构建符号表来确定的。相关代码在 AstTranslator 中，例如 doLexicallyScopedRead 和 doLexicallyScopedWrite 方法。请注意，使用此方案，我们希望 AstTranslator.useLocalValuesForLexicalVars() 的实现返回 false。
- 从概念上讲，指针分析为表示声明函数的每个 CGNode 维护一个公开变量的副本（请参阅 AstSSAPropagationCallGraphBuilder 中的 UpwardFunargPointerKey 内部类）。
- 给定从 CGNode n 在函数 f 中声明的 v 的词法访问，AstSSAPropagationCallGraphBuilder.getLexicalDefiners() 方法发现 f 的相关 CGNode。首先，它为 n 计算 v1 的当前指向集，其中包含 n 的“接收者”。 （在 JavaScript 中，v1 保存被调用函数的抽象位置，而不是 this 参数。）接收器/函数被假定为 ScopeMappingInstanceKey 类型，一个 InstanceKey 维护有关分配函数的方法的词法父级的信息。对于每个 ScopeMappingInstanceKey，我们调用 getFunargNodes 来为 f 找到合适的 CGNodes。注意：对于 JavaScript，getFunargNodes 依赖于（合成）构造函数分配要使用调用者或调用字符串上下文分析的函数，以确定哪个函数调用了构造函数。
- 一旦我们为 f 提供了合适的 CGNode，我们为 v 创建相应的 UpwardFunargPointerKey 对象，然后根据访问是读取还是写入，在键上添加适当的约束。
