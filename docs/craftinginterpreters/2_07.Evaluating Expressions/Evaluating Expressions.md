# 评估表达式

> 你是我的创造者，但我是你的主人；遵守！
> 
> --Mary Shelley,*Frankenstein*

如果你想为这一章设定适当的基调，试着想象一场雷雨，一种喜欢在故事高潮时猛拉开百叶窗的漩涡风暴。也许扔几道闪电。在本章中，我们的解释器将深吸一口气，睁开眼睛，并执行一些代码。

![](./assets/f693555e20dd988f18e05ce74a34023f37b8e376.png)

> 破旧的维多利亚式豪宅是可选的，但增加了氛围。

语言实现可以通过多种方式使计算机执行用户源代码中的命令。他们可以将其编译为机器代码，将其翻译成另一种高级语言，或将其简化为某种字节码格式以供虚拟机运行。不过，对于我们的第一个解释器，我们将采用最简单、最短的路径并执行语法树本身。

现在，我们的解析器只支持表达式。因此，为了“执行”代码，我们将评估一个表达式并产生一个值。对于我们可以解析的每种表达式语法——字面量、运算符等——我们需要一个相应的代码块来知道如何评估该树并产生结果。这就提出了两个问题：

1. 我们产生什么样的值？

2. 我们如何组织这些代码块？

下边依次解决问题...

## 7.1 值的表示

在 Lox 中，值由字面量创建，由表达式计算，并存储在变量中。用户将这些视为*Lox*对象，但它们是用我们的解释器编写的底层语言实现的。这意味着在 Lox 的动态类型和 Java 的静态类型之间架起桥梁。Lox中的一个变量可以存储任意（Lox）类型的值，甚至可以存储不同时间点的不同类型的值。我们可以使用什么 Java 类型来表示它？

> 在这里，我几乎可以互换使用“值”和“对象”。
> 
> 稍后在 C 解释器中，我们将对它们进行细微区分，但这主要是针对实现的两个不同角落使用独特的术语——就地数据与堆分配数据。从用户的角度来看，这些术语是同义词。

给定一个具有该静态类型的 Java 变量，我们还必须能够确定它在运行时持有哪种类型的值。当解释器执行一个`+`运算符时，它需要判断它是在添加两个数字还是连接两个字符串。是否有一种 Java 类型可以容纳数字、字符串、布尔值等？有没有一个可以告诉我们它的运行时类型是什么？有！就是 java.lang.Object。

在解释器中需要存储 Lox 值的地方，我们可以使用 Object 作为类型。Java 有其原始类型的包装类型，它们都是 Object 的子类，因此我们可以将它们用于 Lox 的内置类型：

| Lox type      | Java representation |
| ------------- | ------------------- |
| Any Lox value | Object              |
| `nil`         | `null`              |
| Boolean       | Boolean             |
| number        | Double              |
| string        | String              |

给定一个静态类型 Object 的值，我们可以使用 Java 的内置`instanceof`运算符确定运行时值是数字还是字符串或其他任何内容。换句话说，JVM自己的对象表示方便地为我们提供了实现 Lox 内置类型所需的一切。稍后当我们添加 Lox 的函数、类和实例的概念时，我们将不得不做更多的工作，但是 Object 和装箱的原始类足以满足我们现在需要的类型。

> 我们需要对值做的另一件事是管理它们的内存，Java 也这样做。方便的对象表示和非常好的垃圾收集器是我们用 Java 编写第一个解释器的主要原因。

## 7.2 评估表达式

接下来，我们需要一堆代码来为解析的每种表达式实现求值逻辑。可以将该代码以类似`interpret()`方法的方式填充到语法树类中。实际上，我们可以告诉每个语法树节点，“解释你自己”。这就是四人帮的[解释器设计模式](https://en.wikipedia.org/wiki/Interpreter_pattern)。这是一个简洁的模式，但正如我之前提到的，如果我们将各种逻辑塞进树类中，它会变得混乱。

相反，我们将重用流行的[Visitor 模式](http://craftinginterpreters.com/representing-code.html#the-visitor-pattern)。在上一章中，我们创建了一个 AstPrinter 类。它接受语法树并递归遍历它，构建一个最终返回的字符串。这几乎正是真正的解释器所做的，除了它不是连接字符串，而是计算值。

我们从一个新class开始。

```java
package com.craftinginterpreters.lox;

class Interpreter implements Expr.Visitor<Object> {
}
// lox/Interpreter.java, create new file
```

该类声明它是一个访问者。访问方法的返回类型将是 Object，这是我们用来在 Java 代码中引用 Lox 值的根类。为了实现 Visitor 接口，我们需要为解析器生成的四个表达式树类中的每一个定义访问方法。将从最简单的开始...

### 7.2.1 评估字面值

表达式树的叶子——所有其他表达式组成的语法的原子——是字面值。字面值几乎已经是值，但区别很重要。字面值是产生值的一些语法。字面值总是出现在用户源代码的某处。许多值是由计算产生的，并不存在于代码本身的任何地方。那些不是字面值。字面值来自解析器的域。值是一个解释器概念，是运行时世界的一部分。

> 在[下一章](http://craftinginterpreters.com/statements-and-state.html)中，我们在实现变量时，会添加标识符表达式，它也是叶节点。

因此，就像我们在解析器中将字面值*token*转换为字面值*语法树节点*一样 ，现在我们将字面值树节点转换为运行时值。事实证明这很简单。

```java
  @Override
  public Object visitLiteralExpr(Expr.Literal expr) {
    return expr.value;
  }
// lox/Interpreter.java, in class Interpreter  
```

我们在扫描过程中急切地产生了运行时值并将其填充到token中。解析器获取该值并将其插入字面值树节点中，因此要评估字面值，我们只需将其直接返回即可。

### 7.2.2 计算括号

下一个要评估的最简单的节点是分组——由于在表达式中使用显式括号而得到的节点。

```java
  @Override
  public Object visitGroupingExpr(Expr.Grouping expr) {
    return evaluate(expr.expression);
  }
// lox/Interpreter.java, in class Interpreter  
```

分组节点具有对包含在括号内的表达式的内部节点的引用。为了评估分组表达式本身，我们递归地评估该子表达式并将其返回。

我们依赖这个辅助方法，它简单地将表达式发送回解释器的访问者实现：

一些解析器不为括号定义树节点。相反，在解析带括号的表达式时，它们只是返回内部表达式的节点。我们确实在 Lox 中为括号创建了一个节点，因为稍后我们需要它来正确处理赋值表达式的左侧。

```java
  private Object evaluate(Expr expr) {
    return expr.accept(this);
  }
// lox/Interpreter.java, in class Interpreter 
```

### 7.2.3 评估一元表达式

与分组一样，一元表达式有一个我们必须首先求值的子表达式。不同之处在于，一元表达式本身会在之后做一些工作。

```java
  @Override
  public Object visitUnaryExpr(Expr.Unary expr) {
    Object right = evaluate(expr.right);

    switch (expr.operator.type) {
      case MINUS:
        return -(double)right;
    }

    // Unreachable.
    return null;
  }
// lox/Interpreter.java, add after visitLiteralExpr()  
```

首先，我们评估操作数表达式。然后我们将一元运算符本身应用于其结果。有两种不同的一元表达式，由运算符token的类型标识。

这里显示的是`-`，它否定子表达式的结果。子表达式必须是数字。由于在 Java 中我们*静态*地不知道这一点，所以在执行操作之前对其进行转换。这种类型转换发生在运行时`-`评估 时。这是使语言动态输入的核心。

> 您可能想知道如果转换失败会发生什么。别担心，我们很快就会谈到这一点。

您可以开始了解求值如何递归地遍历树。在我们评估其操作数子表达式之前，我们无法评估一元运算符本身。这意味着我们的解释器正在进行**后序遍历**——每个节点在执行自己的工作之前评估其子节点。

另一个一元运算符是逻辑非。

```java
   switch (expr.operator.type) {
      case BANG:
        return !isTruthy(right);
      case MINUS:
// lox/Interpreter.java, in visitUnaryExpr()
```

实现很简单，但是这个“truthy”的东西是什么？我们需要稍微顺便谈谈西方哲学的一个重大问题：什么是真值?

### 7.2.4  真值与假值

好吧，也许我们不会真正进入普遍问题，但至少在 Lox 的世界里，我们需要决定会返回什么？当你在使用除`true`或`false`逻辑运算之外的东西比如逻辑运算`!`或任何其他期待是布尔值的地方。

我们*可以*说这是一个错误，因为我们不使用隐式转换，但大多数动态类型语言并不是那么苦行僧。相反，他们将所有类型的值的宇宙分为两组，其中一组定义为“真”或“真实”或（我最喜欢的）“真值”，其余为“假” ”或“虚假”。这种划分有些随意，并且在一些语言中变得很奇怪。

> 在 JavaScript 中，字符串是真实的，但空字符串不是。数组是真实的，但空数组是真实的?。.?.?也很真实。数字`0`是虚假的，但*字符串*?`"0"`是真实的。
> 
> 在 Python 中，空字符串和 JS 一样是假的，但其他空序列也是假的。
> 
> 在 PHP 中，数字`0`和字符串`"0"`都是假的。大多数其他非空字符串都是真实的。
> 
> 明白了吗？

Lox 遵循 Ruby 的简单规则：`false`并且`nil`是假的，其他一切都是真的。我们这样实现：

```java
 private boolean isTruthy(Object object) {
    if (object == null) return false;
    if (object instanceof Boolean) return (boolean)object;
    return true;
  }
// lox/Interpreter.java, add after visitUnaryExpr()
```

### 7.2.5 评估二元运算符

关于最后一个表达式树类，二元运算符。其中有一些，我们将从算术开始。

```java
@Override
  public Object visitBinaryExpr(Expr.Binary expr) {
    Object left = evaluate(expr.left);
    Object right = evaluate(expr.right); 

    switch (expr.operator.type) {
      case MINUS:
        return (double)left - (double)right;
      case SLASH:
        return (double)left / (double)right;
      case STAR:
        return (double)left * (double)right;
    }

    // Unreachable.
    return null;
  }
// lox/Interpreter.java, add after evaluate()  
```

> 您是否注意到我们在这里固定了语言语义的一个微妙角落？在二进制表达式中，我们按从左到右的顺序计算操作数。如果这些操作数有副作用，则该选择是用户可见的，因此这不仅仅是一个实现细节。
> 
> 如果我们希望我们的两个解释器保持一致（提示：我们这样做），我们需要确保 clox 做同样的事情。

我想你可以弄清楚这里发生了什么。与一元否定运算符的主要区别在于我们有两个要计算的操作数。

我省略了一个算术运算符，因为它有点特殊。

```java
    switch (expr.operator.type) {
      case MINUS:
        return (double)left - (double)right;
      case PLUS:
        if (left instanceof Double && right instanceof Double) {
          return (double)left + (double)right;
        } 

        if (left instanceof String && right instanceof String) {
          return (String)left + (String)right;
        }

        break;
      case SLASH:
// lox/Interpreter.java, in visitBinaryExpr()
```

该`+`运算符也可用于连接两个字符串。为了处理这个问题，我们不只是假设操作数是某种类型并*强制转换*它们，我们动态地*检查*类型并选择适当的操作。这就是为什么我们需要我们的对象表示来支持`instanceof`。

> 我们可以专门为字符串连接定义一个运算符。这就是 Perl (`.`)、Lua (`..`)、Smalltalk (`,`)、Haskell (`++`) 和其他语言所做的。
> 
> 我认为使用与 Java、JavaScript、Python 和其他语言相同的语法会使 Lox 更易于使用。这意味着`+`运算符被**重载**以支持添加数字和连接字符串。即使在不用`+`于字符串的语言中，它们仍然经常重载它以添加整数和浮点数。

接下来是比较运算符。

```java
    switch (expr.operator.type) {
      case GREATER:
        return (double)left > (double)right;
      case GREATER_EQUAL:
        return (double)left >= (double)right;
      case LESS:
        return (double)left < (double)right;
      case LESS_EQUAL:
        return (double)left <= (double)right;
      case MINUS:
// lox/Interpreter.java, in visitBinaryExpr()
```

它们与算术基本相同。唯一的区别是算术运算符产生的值与操作数（数字或字符串）的类型相同，而比较运算符总是产生布尔值。

最后一对运算符是相等的。

```java
      case BANG_EQUAL: return !isEqual(left, right);
      case EQUAL_EQUAL: return isEqual(left, right);
// lox/Interpreter.java, in visitBinaryExpr()
```

与需要数字的比较运算符不同，相等运算符支持任何类型的操作数，甚至是混合操作数。你不能问 Lox 3 是否*小于*，`"three"`?,但你可以问它是否*等于*它。

> 剧透警告：不是。

与真值测试一样，相等逻辑被提升到一个单独的方法中。

```java
private boolean isEqual(Object a, Object b) {
    if (a == null && b == null) return true;
    if (a == null) return false;

    return a.equals(b);
  }
// lox/Interpreter.java, add after isTruthy() 
```

这是我们如何用 Java 表示 Lox 对象的细节的角落之一。我们需要正确实现*Lox 的*相等性概念，这可能与 Java 不同。

幸运的是，两者非常相似。Lox 不进行相等性的隐式转换，而 Java 也不做。我们确实必须特别处理`nil`/`null`，以便在我们尝试在`null`上调用`equals()`时不会抛出 NullPointerException。还好，这对我们很友好。Java 的`equals()`Boolean、Double 和 String 方法具有我们想要的 Lox 行为。

> 您希望它评估什么：
> 
> ```java
> ( 0 / 0 ) == ( 0 / 0 )
> ```

> 根据指定双精度数行为的[IEEE 754](https://en.wikipedia.org/wiki/IEEE_754)，零除以零会得到特殊的**NaN**（“非数字”）值。奇怪的是，NaN*不*等于自身。

> 在 Java 中，`==`原始双打上的运算符保留了该行为，但?`equals()`Double 类上的方法没有。Lox 使用后者，因此不遵循 IEEE。这些微妙的不兼容性占据了语言实现者生活中令人沮丧的一小部分。

就这些！这就是我们正确解释有效 Lox 表达式所需的所有代码。但是*无效*的呢？特别是，当子表达式的计算结果为执行操作的错误类型的对象时会发生什么？

## 7.3 运行时错误

每当一个子表达式产生一个对象并且运算符要求它是一个数字或一个字符串时，我就很随意地干扰强制转换。这些操作可能会失败。虽然用户的代码是错误的，如果我们想要创建可用的语言，我们也有责任优雅地处理该错误。

> 我们根本无法检测或报告类型错误。如果您将指针转换为某种与实际指向的数据不匹配的类型，这就是 C 所做的。C 通过允许这样做获得了灵活性和速度，但也是出了名的危险。一旦你误解了内存中的位，所有的赌注都会落空。
> 
> 很少有现代语言接受这样的不安全操作。相反，大多数都是**内存安全**的，并通过静态和运行时检查的组合确保程序永远不会错误地解释存储在一块内存中的值。

现在是我们讨论**运行时错误**的时候了。在前面的章节中，我在谈论错误处理时泼了很多墨，但那些都是*语法*或*静态*错误。*在执行任何*代码之前检测并报告这些。运行时错误是语言语义要求我们在程序运行时检测和报告的故障（因此得名）。

现在，如果操作数对于正在执行的操作而言是错误的类型，则 Java 转换将失败并且 JVM 将抛出 ClassCastException。这将展开整个堆栈并退出应用程序，向用户吐出 Java 堆栈跟踪。这可能不是我们想要的。Lox 是用 Java 实现的这一事实应该是对用户隐藏的细节。相反，我们希望他们了解发生了*Lox*运行时错误，并向他们提供与我们的语言和他们的程序相关的错误消息。

不过，Java 行为确实有其优点。当错误发生时，它会正确地停止执行任何代码。假设用户输入了一些表达式，例如：

```java
2 * (3 / -"muffin")
```

您不能 否定muffin，因此我们需要在该内部`-`表达式中报告运行时错误。这反过来意味着我们无法评估`/`表达式，因为它没有有意义的右操作数。对于`*`也一样.所以当某个表达式深处发生运行时错误时，我们需要一路逃逸出去。

> 我不知道，伙计，你*能*拒绝松饼吗？

![](./assets/d98df7197f0bd09d25e9bc8b8832048fb0d06b48.png)

我们可以打印运行时错误，然后中止进程并完全退出应用程序。这具有一定的情节剧天赋。类似于麦克风下降的编程语言解释器。

尽管这很诱人，但我们或许应该做一些不那么灾难性的事情。虽然运行时错误需要停止计算*表达式*，但它不应该杀死*解释器*。如果用户正在运行 REPL 并且在一行代码中有拼写错误，他们仍然应该能够继续进行会话并在此之后输入更多代码。

### 7.3.1 检测运行时错误

我们的树遍历解释器使用递归方法调用评估嵌套表达式，我们需要跳出这些嵌套方法。在 Java 中抛出异常是实现这一点的好方法。但是，我们将定义一个特定的 Lox 的错误，而不是使用 Java 自己的转换错误，以便我们可以按照自己的意愿处理它。

在我们进行转换之前，我们自己检查对象的类型。因此，对于一元`-`，我们添加：

```java
      case MINUS:
        checkNumberOperand(expr.operator, right);
        return -(double)right;
// lox/Interpreter.java, in visitUnaryExpr()
```

检查操作数的代码是：

```java
private void checkNumberOperand(Token operator, Object operand) {
    if (operand instanceof Double) return;
    throw new RuntimeError(operator, "Operand must be a number.");
  }
// lox/Interpreter.java, add after visitUnaryExpr()
```

当检查失败时，它会抛出以下之一：

```java
package com.craftinginterpreters.lox;

class RuntimeError extends RuntimeException {
  final Token token;

  RuntimeError(Token token, String message) {
    super(message);
    this.token = token;
  }
}
// lox/RuntimeError.java, create new file
```

与 Java 转换异常不同，我们的类跟踪token，该token标识用户代码中运行时错误的来源。与静态错误一样，这有助于用户知道在哪里修复他们的代码。

> 我承认“RuntimeError”这个名字令人困惑，因为 Java 定义了一个 RuntimeException 类。构建解释器的一个烦人的事情是你的名字经常与实现语言已经采用的名字冲突。等到我们支持 Lox 类。

我们需要对二元运算符进行类似的检查。由于我向您保证过实现解释器所需的每一行代码，所以我将逐一检查它们。

大于：

```java
      case GREATER:
        checkNumberOperands(expr.operator, left, right);
        return (double)left > (double)right;
// lox/Interpreter.java, in visitBinaryExpr()
```

大于或等于：

```java
      case GREATER_EQUAL:
        checkNumberOperands(expr.operator, left, right);
        return (double)left >= (double)right;
// lox/Interpreter.java, in visitBinaryExpr()
```

小于：

```java
      case LESS:
        checkNumberOperands(expr.operator, left, right);
        return (double)left < (double)right;
// lox/Interpreter.java, in visitBinaryExpr()
```

小于或等于：

```java
      case LESS_EQUAL:
        checkNumberOperands(expr.operator, left, right);
        return (double)left <= (double)right;
// lox/Interpreter.java, in visitBinaryExpr()
```

减法：

```java
      case MINUS:
        checkNumberOperands(expr.operator, left, right);
        return (double)left - (double)right;
// lox/Interpreter.java, in visitBinaryExpr()
```

除法：

```java
      case SLASH:
        checkNumberOperands(expr.operator, left, right);
        return (double)left / (double)right;
// lox/Interpreter.java, in visitBinaryExpr()
```

乘法：

```java
      case STAR:
        checkNumberOperands(expr.operator, left, right);
        return (double)left * (double)right;
// lox/Interpreter.java, in visitBinaryExpr()
```

所有这些都依赖于这个验证器，它实际上与一元验证器相同：

```java
  private void checkNumberOperands(Token operator,
                                   Object left, Object right) {
    if (left instanceof Double && right instanceof Double) return;

    throw new RuntimeError(operator, "Operands must be numbers.");
  }
// lox/Interpreter.java, add after checkNumberOperand()  
```

> 另一个微妙的语义选择：我们在检查其中一个的类型之前评估*两个*操作*数*。想象一下，我们有一个函数打印它的参数然后返回它。使用它，我们写：`say()`

```java
say("left") - say("right");
```

> 我们的解释器在报告运行时错误之前打印“left”和“right”。我们本可以指定在评估右操作数之前先检查左操作数。

剩下的最后一个运算符，也是奇葩的运算符，是加法。由于`+`对数字和字符串进行了重载，因此它已经具有检查类型的代码。如果两个成功案例都不匹配，我们需要做的就是失败。

```java
          return (String)left + (String)right;
        }

        throw new RuntimeError(expr.operator,
            "Operands must be two numbers or two strings.");
      case SLASH:
// lox/Interpreter.java, in visitBinaryExpr(), replace 1 line
```

这让我们能够检测到评估器内部深处的运行时错误。错误正在被抛出。下一步是编写捕获它们的代码。为此，我们需要将 Interpreter 类连接到驱动它的主 Lox 类中。

## 7.4 接通解释器

visit 方法是 Interpreter 类的核心，真正的工作发生在这里。我们需要在它们周围包裹一层皮肤以与程序的其余部分进行交互。Interpreter 的公共 API 只是一种方法。

```java
  void interpret(Expr expression) { 
    try {
      Object value = evaluate(expression);
      System.out.println(stringify(value));
    } catch (RuntimeError error) {
      Lox.runtimeError(error);
    }
  }
// lox/Interpreter.java, in class Interpreter  
```

这接受表达式的语法树并对其求值。如果成功，`evaluate()`则返回结果值的对象。`interpret()`将其转换为字符串并将其显示给用户。要将 Lox 值转换为字符串，我们依赖于：

```java
  private String stringify(Object object) {
    if (object == null) return "nil";

    if (object instanceof Double) {
      String text = object.toString();
      if (text.endsWith(".0")) {
        text = text.substring(0, text.length() - 2);
      }
      return text;
    }

    return object.toString();
  }
// lox/Interpreter.java, add after isEqual()  
```

这是另一段类似`isTruthy()`的代码，它跨越 Lox 对象的用户视图和它们在 Java 中的内部表示之间的隔阂。

这很简单。由于 Lox 被设计为熟悉 Java 的人，因此布尔值之类的东西在两种语言中看起来是一样的。两种边缘情况是`nil`，我们使用 Java 表示`null`，和数字。

Lox 甚至对整数值也使用双精度数。在这种情况下，他们应该不带小数点打印。由于 Java 同时具有浮点数和整数类型，它希望您知道您使用的是哪一种。它通过将显式添加`.0`到整数值双打来告诉您。我们不关心这个，所以我们把它砍掉了。

> 再一次，我们用数字来处理这种边缘情况，以确保 jlox 和 clox 工作相同。像这样处理语言的奇怪角落会让你发疯，但这是工作的重要组成部分。
> 
> 用户有意或无意地依赖这些细节，如果实现不一致，他们的程序在不同的解释器上运行时就会崩溃。

### 7.4.1 报告运行时错误

如果在评估表达式时抛出运行时错误，则`interpret()`捕获它。这让我们可以向用户报告错误，然后优雅地继续。我们现有的所有错误报告代码都位于 Lox 类中，因此我们也将此方法放在那里：

```java
  static void runtimeError(RuntimeError error) {
    System.err.println(error.getMessage() +
        "\n[line " + error.token.line + "]");
    hadRuntimeError = true;
  }
// lox/Lox.java, add after error() 
```

我们使用与 RuntimeError 关联的标记来告诉用户发生错误时正在执行哪一行代码。更好的办法是为用户提供完整的调用堆栈，以显示他们如何*执行*该代码。但是我们还没有函数调用，所以我想我们不必担心。

显示错误后，`runtimeError()`设置此字段：

```java
  static boolean hadError = false;
  static boolean hadRuntimeError = false;

  public static void main(String[] args) throws IOException {
// lox/Lox.java, in class Lox  
```

该字段很小但起着很重要的作用。

```java
    run(new String(bytes, Charset.defaultCharset()));

    // Indicate an error in the exit code.
    if (hadError) System.exit(65);
    if (hadRuntimeError) System.exit(70);
  }
// lox/Lox.java, in runFile()
```

如果用户正在从文件运行 Lox 脚本并且发生运行时错误，我们会在进程退出时设置退出代码以告知调用进程。不是每个人都关心shell规则，但我们关心。

> 如果用户正在运行 REPL，我们不关心跟踪运行时错误。在他们被报告后，我们简单地循环并让他们输入新代码并继续进行。

### 7.4.2 运行解释器

现在我们有了解释器，Lox 类就可以开始使用它了。

```java
public class Lox {
  private static final Interpreter interpreter = new Interpreter();
  static boolean hadError = false;
// lox/Lox.java, in class Lox
```

我们将字段设为静态，以便对`run()`REPL 会话内部的连续调用重用同一个解释器。现在这没有什么区别，但是稍后当解释器存储全局变量时它会有所不同。这些变量应该在整个 REPL 会话中持续存在。

最后，我们删除了[上一章](http://craftinginterpreters.com/parsing-expressions.html)中用于打印语法树的临时代码行，并将其替换为：

```java
    // Stop if there was a syntax error.
    if (hadError) return;

    interpreter.interpret(expression);
  }
// lox/Lox.java, in run(), replace 1 line
```

我们现在有一个完整的语言管道：扫描、解析和执行。恭喜，您现在拥有了自己的算术计算器。

如您所见，解释器非常简单。但是我们今天设置的 Interpreter 类和 Visitor 模式构成了骨架，后面的章节将充满有趣的内容——变量、函数等。现在，解释器并没有做太多事情，但它有了生命！

![](./assets/f11df8c9bff3af55c57c8476211a1024cc4f93ad.png)

---

## [挑战](http://craftinginterpreters.com/evaluating-expressions.html#challenges)

1. 允许对数字以外的类型进行比较可能很有用。运算符可能对字符串有合理的解释。即使是混合类型之间的比较，`3 < "pancake"`也可以很方便地启用异构类型的有序集合之类的东西。或者它可能只会导致错误和混乱。
   
   你会扩展 Lox 以支持比较其他类型吗？如果是这样，您允许哪些类型对以及如何定义它们的顺序？证明你的选择是合理的，并将它们与其他语言进行比较。

2. 许多语言都这样定义`+`，如果*其中一个*操作数是字符串，则将另一个操作数转换为字符串，然后将结果连接起来。例如，`"scone" + 4`会产生`scone4`.扩展代码使`visitBinaryExpr()`支持它。

3. 如果你将一个数除以零，现在会发生什么？你认为应该发生什么？证明你的选择。您知道的其他语言如何处理被零除，以及它们为什么会做出这样的选择？
   
   更改中的实现`visitBinaryExpr()`以检测并报告此案例的运行时错误。

## [设计说明：静态和动态类型](http://craftinginterpreters.com/evaluating-expressions.html#design-note)

某些语言（如 Java）是静态类型的，这意味着在运行任何代码之前的编译时会检测并报告类型错误。其他的，如 Lox，是动态类型的，并且将类型错误检查推迟到运行时尝试操作之前。我们倾向于认为这是一个非黑即白的选择，但实际上它们之间存在连续统一体。

事实证明，即使是大多数静态类型语言也会在运行时进行*一些*类型检查。类型系统静态地检查大多数类型规则，但在生成的代码中为其他操作插入运行时检查。

例如，在 Java 中，*静态*类型系统假定强制转换表达式总是安全地成功。在转换一些值后，您可以将其静态地视为目标类型，而不会出现任何编译错误。但很明显，沮丧可能会失败。静态检查器可以假定转换总是成功而不违反语言的健全性保证的唯一原因是因为转换*在运行时*被检查并在失败时抛出异常。

一个更微妙的例子是Java 和 C# 中的[协变数](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)#Covariant_arrays_in_Java_and_C.23)组。数组的静态子类型规则允许不合理的操作。考虑：

```java
Object[] stuff = new Integer[1];
stuff[0] = "not an int!";
```

这段代码编译没有任何错误。第一行向上转换 Integer 数组并将其存储在 Object 数组类型的变量中。第二行在其中一个单元格中存储一个字符串。对象数组类型静态地允许——字符串*是*对象——但在运行时引用的实际整数数组`stuff`不应该包含字符串！为了避免这种灾难，当您将值存储在数组中时，JVM 会执行*运行时*检查以确保它是允许的类型。否则，它会抛出 ArrayStoreException。

Java 可以通过在第一行禁止转换来避免在运行时检查它。它可以使数组*不变*，使得整数数组*不是*对象数组。这在静态上是合理的，但它禁止仅从数组中读取的常见且安全的代码模式。如果您从不*写入*数组，则协方差是安全的。在 Java 1.0 支持泛型之前，这些模式对于可用性特别重要。James Gosling 和其他 Java 设计师在一些静态安全性和性能上进行了权衡——那些数组存储检查需要时间——以换取一些灵活性。

*很少有现代静态类型语言不会在某处*做出这种权衡?。甚至 Haskell 也会让你运行带有非穷尽匹配的代码。如果您发现自己在设计一种静态类型语言，请记住，您有时可以通过将某些类型检查推迟到运行时来为用户提供更大的灵活性，而不会牺牲*太多静态安全的好处。*

另一方面，用户选择静态类型语言的一个关键原因是因为这种语言让他们相信，在他们的程序运行时，某些类型的错误*永远不会发生。*将太多的类型检查推迟到运行时，你会削弱这种信心。