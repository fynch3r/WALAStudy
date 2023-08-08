关于一些 Shrike 与源码的映射 API
## 获取源码文件名
IClass 类有一个 getSourceFileName 方法，同时，ClassLoaderImpl 在类加载的时候会尝试在 IClass 和 源代码的映射关系。在类加载结束之后，类加载器会为每一个 .java 内容寻找到每一个 IClass 对象。

## 获取 value number 对应的变量名称
如果信息可用，可以通过 IR.getLocalNames() 检索它。 请注意，并非每个 value number 都对应于 Java 源代码中的局部变量。 由于 IR 构造期间的复制传播，一些 value number 可能代表多个局部变量。

## 从 IR 指令获取源码行数
给定从字节码生成的 IR ir 中某些 SSAInstruction 的索引 i，您可以获得相应的字节码索引和源行号，如下所示：
```java
IBytecodeMethod method = (IBytecodeMethod)ir.getMethod();
int bytecodeIndex = method.getBytecodeIndex(i);
int sourceLineNum = method.getLineNumber(bytecodeIndex);
```
但事实上，代码行号信息并不是一直有效的。

## 从切片获取源码行数
给定来自 Java 字节码文件（通过 Shrike）的 Statement，您可以使用以下内容获取源代码中的行号：
```java
if (s.getKind() == Statement.Kind.NORMAL) { // ignore special kinds of statements
  int bcIndex, instructionIndex = ((NormalStatement) s).getInstructionIndex();
  try {
    bcIndex = ((ShrikeBTMethod) s.getNode().getMethod()).getBytecodeIndex(instructionIndex);
    try {
      int src_line_number = s.getNode().getMethod().getLineNumber(bcIndex);
      System.err.println ( "Source line number = " + src_line_number );
    } catch (Exception e) {
      System.err.println("Bytecode index no good");
      System.err.println(e.getMessage());
    }
  } catch (Exception e ) {
    System.err.println("it's probably not a BT method (e.g. it's a fakeroot method)");
    System.err.println(e.getMessage());
  }
}
```
但是，对于通过强制转换来自 Java 源文件的语句，没有字节码索引，因此（尽管 javadoc 可能会让您相信）getLineNumber() 的参数只是一个指令索引：
```java
if (s.getKind() == Statement.Kind.NORMAL) {
  int instructionIndex = ((NormalStatement) s).getInstructionIndex();
  int lineNum = ((ConcreteJavaMethod) s.getNode().getMethod()).getLineNumber(instructionIndex);
  System.out.println("Source line number = " + lineNum );
}
```
这在研究切片的输出时很有用。

## 根据 IClass 获取 .jar 文件
根据 IClass 对象获取 .jar 文件名：
```java
Shrike shrikeKlass = (ShrikeClass) klass;
JarFileEntry moduleEntry = (JarFileEntry) shrikeKlass.getModuleEntry();
String jarFile = moduleEntry.getJarFile();
```
需要注意的是，如果它不是 .jar 中的 .class 文件，则强制转换可能会失败。



