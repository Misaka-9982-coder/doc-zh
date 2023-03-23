# 树遍历解释器

在这一部分，我们开始 jlox，这是我们两个解释器中的第一个。编程语言是一个巨大的话题，有成堆的概念和术语会一下子塞进你的大脑。编程语言理论需要一定程度的思维严谨性，自上次微积分期末考试以来，您可能还没有要求过。（幸运的是，本书中没有太多的理论。）

实现解释器使用了一些在其他类型的应用程序中不常见的架构技巧和设计模式，因此我们也将习惯工程方面的事情。考虑到所有这些，我们将让我们必须编写的代码尽可能简单明了。

在不到 2000 行干净的 Java 代码中，我们将为 Lox 构建一个完整的解释器，它完全按照我们指定的方式实现该语言的每一个功能。前几章从前到后贯穿解释器的各个阶段——[扫描](http://craftinginterpreters.com/scanning.html)、[解析](http://craftinginterpreters.com/parsing-expressions.html)和[评估代码](http://craftinginterpreters.com/evaluating-expressions.html)。之后，我们一次添加一个语言功能，将一个简单的计算器发展成为一种成熟的脚本语言。