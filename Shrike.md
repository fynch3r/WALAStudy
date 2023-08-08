对用于读取、修改和编写 Java 字节码的 Shrike 工具包的基本技术概述。

# 关键类
## ShrikeCT
用于读取和编写字节码文件：
- ClassReader：提供字节码的读取功能
- ClassWriter：提供一个类文件的 JVM 表达形式

## ShrikeBT
用于操控、处理字节码：
- MethodData：表达方法的信息
- MethodEditor：修改字节码的核心工具类
- ClassInstrumenter：可以通过 ClassInstrumenter#getReader() 获取特定方法，然后调用 ClassInstrumenter#emitClass() 获取一个用于修改类文件的 ClassWriter
- CTCompiler：用于将一个 ShrikeBT 代码编译进特定方法。可以参考CTUtils.compileAndAddMethodToClassWriter() 和MethodData.makeWithDefaultHandlersAndInstToBytecodes()

