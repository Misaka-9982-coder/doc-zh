# 字节码虚拟机

我们的 Java 解释器 jlox 教会了我们许多编程语言的基础知识，但我们还有很多东西要学。首先，如果您在 jlox 中运行任何有趣的 Lox 程序，您会发现它非常慢。它使用的解释风格——直接遍历 AST——对于*一些现实世界的用途*来说已经足够好了，但对于通用脚本语言来说还有很多不足之处。

此外，我们隐含地依赖 JVM 本身的运行时特性。我们理所当然地认为`instanceof`Java中的东西*以某种方式*工作。而且我们再也不用担心内存管理，因为 JVM 的垃圾收集器会替我们处理它。

当我们专注于高级概念时，掩盖这些是很好的。但是现在我们已经了解了解释器的方法，是时候深入到那些较低的层并只使用 C 标准库从头开始构建我们自己的虚拟机了...