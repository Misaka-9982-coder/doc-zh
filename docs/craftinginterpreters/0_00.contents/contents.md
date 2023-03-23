![](./assets/6f7f36299a48f19840b95efbe9ecd1ccaf67956d.png)

# 本书在线地址 [http://craftinginterpreters.com/](http://craftinginterpreters.com/)

# 感谢作者



作者用近 4 年的时间持续创作和改进本书，并把其 Web 版本公开在网上。这本纸质书于今年 7 月出版，立刻在 Hacker News 等网络媒介上引起关注和讨论。

书中作者首先定义了一个动态类型的语言 Lox。Lox 并不是一个“简单”的 toy 语言，它支持了 function、closure、class、inheritance、gabage collection 等高级特性。然后作者用两种不同的实现方式（Tree-walk Interpreter 和 Bytecode VM）给我们展示了如何 Parsing 以及实现这些高级特性。

1. Tree-walk Interpreter. 作者认为上来直接用 C 语言实现 Bytecode VM 对于读者来说有点不友好，所以先用 Java 实现了一个 Tree-walk Interpreter。这部分主要 focus 在概念上，实现了一个简单易理解，但不是那么高效的 Interpreter。

2. Bytecode VM. 在本书的第二部分，做着用 C 语言实现了一个基于 Bytecode VM 的解释器。包含了Pratt Parser、编译 Lox 代码到 VM 字节码、VM、gabage collection、optimization 等。

很佩服作者把一个复杂的 topic 分解成一步一步可以独立运行的模块的能力。本书共有 30 章，几乎每一章结尾时都有一个可以运行的解释器，只是后面的章节会在前面的章节上加一些 feature。这个说起来简单，但是如何分析 dependency、分解 feature、重构等还是比较挑战的。

最后，书中的每个插图都是作者亲手画在纸上，然后扫描的电脑上的。



# 目录

## 0.前言

### 奉献 Dedication

### [致谢 Acknowledgements](../../craftinginterpreters/0_01.acknowledgements/acknowledgements.md)

## 一、欢迎

### 1、[介绍 Introduction](../../craftinginterpreters/1_01.Introduction/Introduction.md)

 设计说明：名称中包含什么？ Design Note: What’s in a Name?

### 2、[概览 A Map of the Territory](../../craftinginterpreters/1_02.A%20Map%20of%20the%20Territory/A%20Map%20of%20the%20Territory.md)

### 3、[Lox 语言 The Lox Language](../../craftinginterpreters/1_03.The%20Lox%20Language/The%20Lox%20Language.md)

 设计说明：表达式和语句 Design Note: Expressions and Statements

## 二、[基于树遍历的解释器 A TREE-WALK INTERPRETER](../../craftinginterpreters/2_00.A%20TREE-WALK%20INTERPRETER/A%20TREE-WALK%20INTERPRETER.md)

### 4、[扫描 Scanning](../../craftinginterpreters/2_04.Scanning/Scanning.md)

 设计说明：隐式分号 Design Note: Implicit Semicolons

### 5、[代码表示 Representing Code](../../craftinginterpreters/2_05.Representing%20Code/Representing%20Code.md)

### 6、[解析表达式 Parsing Expressions](../../craftinginterpreters/2_06.Parsing%20Expressions/Parsing%20Expressions.md)

 设计说明：逻辑与历史  Design Note: Logic Versus History

### 7、[表达式求值 Evaluating Expressions](../../craftinginterpreters/2_07.Evaluating%20Expressions/Evaluating%20Expressions.md)

 设计说明：静态和动态类型 Design Note: Static and Dynamic Typing

### 8、[语句和状态 Statements and State](../../craftinginterpreters/2_08.Statements%20and%20State/Statements%20and%20State.md)

 设计说明：隐式变量声明 Design Note: Implicit Variable Declaration

### 9、[控制流 Control Flow](../../craftinginterpreters/2_09.Control%20Flow/Control%20Flow.md)

 设计说明：一勺语法糖  Design Note: Spoonfuls of Syntactic Sugar

### 10、[函数 Functions](../../craftinginterpreters/2_10.Functions/Functions.md)

### 11、[解析和绑定 Resolving and Binding](../../craftinginterpreters/2_11.Resolving%20and%20Binding/Resolving%20and%20Binding.md)

### 12、[类 Classes](../../craftinginterpreters/2_12.Classes/Classes.md)

 设计说明：原型和电源  Design Note: Prototypes and Power

### 13、[继承 Inheritance](../../craftinginterpreters/2_13.Inheritance/Inheritance.md)

## 三、[字节码虚拟机 A BYTECODE VIRTUAL MACHINE](../../craftinginterpreters/3_00.A%20BYTECODE%20VIRTUAL%20MACHINE/A%20BYTECODE%20VIRTUAL%20MACHINE.md)

### 14、[字节码块 Chunks of Bytecode](../../craftinginterpreters/3_14.Chunks%20of%20Bytecode/Chunks%20of%20Bytecode.md)

 设计笔记：测试你的语言   Design Note: Test Your Language

### 15、[虚拟机 A Virtual Machine](../../craftinginterpreters/3_15.A%20Virtual%20Machine/A%20Virtual%20Machine.md)

 设计说明：基于寄存器的字节码  Design Note: Register-Based Bytecode

### 16、[按需扫描 Scanning on Demand](../../craftinginterpreters/3_16.Scanning%20on%20Demand/Scanning%20on%20Demand.md)

### 17、[编译表达式 Compiling Expressions](../../craftinginterpreters/3_17.Compiling%20Expressions/Compiling%20Expressions.md)

 设计说明：只是解析    Design Note: It’s Just Parsing

### 18、[值的类型 Types of Values](../../craftinginterpreters/3_18.Types%20of%20Values/Types%20of%20Values.md)

### 19、[字符串 Strings](../../craftinginterpreters/3_19.Strings/Strings.md)

 设计说明：字符串编码  Design Note: String Encoding

### 20、[哈希表 Hash Tables](../../craftinginterpreters/3_20.Hash%20Tables/Hash%20Tables.md)

### 21、[全局变量 Global Variables](../../craftinginterpreters/3_21.Global%20Variables/Global%20Variables.md)

### 22、[局部变量 Local Variables](../../craftinginterpreters/3_22.Local%20Variables/Local%20Variables.md)

### 23、[跳转控制 Jumping Back and Forth](../../craftinginterpreters/3_23.Jumping%20Back%20and%20Forth/Jumping%20Back%20and%20Forth.md)

 设计说明：考虑到有害的 Goto   Design Note: Considering Goto Harmful

### 24、[函数调用   Calls and Functions](../../craftinginterpreters/3_24.Calls%20and%20Functions/Calls%20and%20Functions.md)

### 25、[闭包 Closures](../../craftinginterpreters/3_25.Closures/Closures.md)

 设计说明：关闭循环变量  Design Note: Closing Over the Loop Variable

### 26、[垃圾收集 Garbage Collection](../../craftinginterpreters/3_26.Garbage%20Collection/Garbage%20Collection.md)

 设计说明：分代垃圾回收 Design Note: Generational Collectors

### 27、[类和实例  Classes and Instances ](../../craftinginterpreters/3_27.Classes%20and%20Instances/Classes%20and%20Instances.md)

### 28、[方法和初始化器  Methods and Initializers ](../../craftinginterpreters/3_28.Methods%20and%20Initializers/Methods%20and%20Initializers.md)

 设计说明：新奇预算  Design Note: Novelty Budget

### 29. [超类  Superclasses ](../../craftinginterpreters/3_29.Superclasses/Superclasses.md)

### 30. [优化  Optimization ](../../craftinginterpreters/3_30.Optimization/Optimization.md)

## 四、后记

### [A1.附录1：Lox 语法    Lox Grammar  ](../../craftinginterpreters/4_01.A1.Appendix%20I%20Lox%20Grammar/A1.Appendix%20I%20Lox%20Grammar.md)

### [A2.附录2：生成的语法树类   Generated Syntax Tree Classes ](../../craftinginterpreters/4_02.A2.Appendix%20II%20Generated%20Syntax%20Tree%20Classes/A2.Appendix%20II%20Generated%20Syntax%20Tree%20Classes.md)