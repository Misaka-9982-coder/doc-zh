# 解析表达式

> 连国王都知道如何控制的语法。--Molière

本章标志着本书的第一个重要里程碑。我们中的许多人都将正则表达式和子字符串操作拼凑在一起，以便从一堆文本中提取一些意义。该代码可能充满了错误和需要维护的野兽。编写一个*真正的*解析器——一个具有良好的错误处理能力、连贯的内部结构以及强大地咀嚼复杂语法的能力的解析器——被认为是一种罕见的、令人印象深刻的技能。在本章中，您将实现它。

> “Parse”来自古法语“pars”的英语，意思是“词性”。这意味着获取文本并将每个单词映射到该语言的语法。我们在这里以同样的意义使用它，除了我们的语言比古法语更现代一点。
> 
> 就像许多成年仪式一样，你可能会发现它在你身后时看起来比在你面前隐约可见时看起来小一些，也不那么令人生畏。

[这比您想象的要容易，部分原因是我们在上一章](http://craftinginterpreters.com/representing-code.html)中提前完成了很多艰苦的工作。您已经了解了正式语法。您熟悉语法树，我们有一些 Java 类来表示它们。唯一剩下的部分是解析——将一系列token转换为其中一个语法树。

一些 CS 教科书非常重视解析器。在 60 年代，计算机科学家——可以理解地厌倦了汇编语言编程——开始设计更复杂、更人性化的语言，如 Fortran 和 ALGOL。确实是，对于当时的原始计算机来说，它们并不是很友好。

> 想象一下，在他们认为*Fortran*是一种改进的情况下，在那些旧机器上进行汇编编程是多么痛苦。

这些先驱设计了他们甚至不确定如何为其编写编译器的语言，然后进行了开创性的工作，发明了可以在那些旧的微型机器上处理这些新的大型语言的解析和编译技术。

经典的编译器书籍读起来就像是对这些英雄及其工具的奉承传记。*Compilers: Principles, Techniques, and Tools*的封面字面上有一条标有“编译器设计的复杂性”的龙，被一名手持剑和盾牌的骑士杀死，剑和盾牌上标有“LALR 解析器生成器”和“语法定向翻译”。他们把它放在厚厚的地方。

有点自我祝贺是当之无愧的，但事实是您不需要了解大部分内容就可以为现代机器开发出高质量的解析器。一如既往，我鼓励你扩大你的教育范围并在以后接受它，但本书省略了奖杯。

## 6.1 歧义和解析游戏

在上一章中，我说过您可以像玩游戏一样“玩”上下文无关语法来*生成*字符串。解析器以相反的方式玩这个游戏。给定一个字符串——一系列token——我们将这些token映射到语法中的终端，以确定哪些规则**可能**生成该字符串。

“可能”的部分很有趣。完全有可能创建一个有*歧义*的语法，其中不同的产生式选择可能导致相同的字符串。当您使用语法*生成*字符串时，这并不重要。但一旦你有了字符串，谁在乎你是如何得到它的呢？

解析时，歧义意味着解析器可能会误解用户的代码。在解析时，不仅要确定字符串是否是有效的 Lox 代码，还要跟踪哪些规则匹配它的哪些部分，以便知道每个标记属于语言的哪个部分。这是在上一章中整理的 Lox 表达式语法：

```
expression     → literal
               | unary
               | binary
               | grouping ;

literal        → NUMBER | STRING | "true" | "false" | "nil" ;
grouping       → "(" expression ")" ;
unary          → ( "-" | "!" ) expression ;
binary         → expression operator expression ;
operator       → "==" | "!=" | "<" | "<=" | ">" | ">="
               | "+"  | "-"  | "*" | "/" ;
```

这是该语法中的有效的字符串：

![](./assets/cb9147328e2426866592f86e4809a87684f0a013.png)

但是我们可以通过两种方式生成它。一种方法是：

1. 从 开始`expression`，选择`binary`。
2. 对于左边`expression`，选择`NUMBER`并使用`6`。
3. 对于操作符，选择`"/"`.
4. 对于右边`expression`，`binary`再次选择。
5. 在该嵌套`binary`表达式中，选择`3 - 1`.

另一个是：

1. 从 开始`expression`，选择`binary`。
2. 对于左边`expression`，`binary`再次选择。
3. 在该嵌套`binary`表达式中，选择`6 / 3`.
4. 回到外面`binary`，对于操作符，选择`"-"`。
5. 对于右边`expression`，选择`NUMBER`并使用`1`。

那些产生相同的*字符串*，但不相同的*语法树*：

![](./assets/ab420f7808d5ae00e820d2138a1ae5217ab7d948.png)

换句话说，语法允许将表达式视为`(6 / 3) - 1` 或 `6 / (3 - 1)`。该`binary`规则允许操作数以您想要的任何方式嵌套。这反过来会影响评估解析树的结果。自黑板首次发明以来，数学家解决这种歧义的方法是定义优先级和结合性规则。

- **优先级(Precedence)** 确定在包含 不同运算符混合的表达式中首先评估哪个运算符。优先规则告诉我们，在上面的例子中`/`在`-`之前。优先级较高的运算符在优先级较低的运算符之前进行评估。等效地，更高优先级的运算符被称为“绑定得更紧”。

- **结合性(Associativity)** 决定了在一系列 相同的运算符中首先评估哪个运算符。当运算符是 左结合的（想想“从左到右”）时，左边的运算符先于右边的运算符求值。因为`-`是左结合的，所以这个表达式：
  
  ```
  5 - 3 - 1
  ```
  
  相当于：
  
  ```
  ( 5 - 3 ) - 1
  ```
  
  另一方面，赋值是**右结合的**。这个：
  
  ```
  a = b = c
  ```
  
  相当于：
  
  ```
  a = (b = c)
  ```

> 虽然现在不常见，但某些语言指定某些运算符对*没有*相对优先级。这使得在不使用显式分组的情况下在表达式中混合这些运算符成为语法错误。
> 
> 同样，一些运算符是**非结合**。这意味着在一个序列中多次使用该运算符是错误的。例如，Perl 的范围运算符不是结合的，因此`a .. b`是可以的，但`a .. b .. c`它是一个错误。

如果没有明确定义的优先级和结合性，使用**多个运算符**的表达式是不明确的——它可以被解析为不同的语法树，而这些语法树又可以求出不同的结果。我们将在 Lox 中通过应用与 C 相同的优先级规则（从最低到最高）来解决这个问题。

| 名称         | 运营商            | 同事    |
| ---------- | -------------- | ----- |
| Equality   | `==? !=`       | Left  |
| Comparison | `>``>=``<``<=` | Left  |
| Term       | `-``+`         | Left  |
| Factor     | `/``*`         | Left  |
| Unary      | `!``-`         | Right |

现在，语法将所有表达式类型填充到一个`expression`规则中。同样的规则被用作操作数的非终结符，它让语法接受任何类型的表达式作为子表达式，而不管优先规则是否允许它。

我们通过对语法进行分层来解决这个问题。我们为每个优先级定义一个单独的规则。

```
expression     → ...
equality       → ...
comparison     → ...
term           → ...
factor         → ...
unary          → ...
primary        → ...
```

> 一些解析器生成器不是将优先级直接纳入语法规则，而是让您保留相同的模棱两可但简单的语法，然后在旁边添加一些明确的运算符优先级元数据以消除歧义。

这里的每个规则只匹配其优先级别或更高级别的表达式。例如，`unary`匹配一元表达式（如）`!negated`或 primary表达式（如`1234`）。`term`且能`1 + 2`配也`3 * 4 / 5`。最终`primary`规则涵盖了最高优先级的形式——字面量和括号表达式。

只需要填写每条规则的产生式。先做简单的。顶级`expression`规则匹配任何优先级别的任何表达式。因为`equality`具有最低的优先级，如果我们匹配它，那么它涵盖了一切。

```
expression     → equality
```

> 我们可以去掉`expression`并简单地在包含表达式的其他规则中使用`equality`，但是使用`expression`会使其他规则更好读一些。
> 
> 此外，在后面的章节中，当我们扩展语法以包括赋值和逻辑运算符时，我们只需要更改`expression`产生式 而不是触及包含表达式的每个规则。

在优先级表的另一端，一个主表达式包含所有文字和分组表达式。

```
primary        → NUMBER | STRING | "true" | "false" | "nil"
               | "(" expression ")" ;
```

一元表达式以一元运算符开头，后跟操作数。由于一元运算符可以嵌套——`!!true`这是一个有效的奇怪表达式——操作数本身可以是一元运算符。递归规则可以很好地处理这个问题。

```
unary          → ( "!" | "-" ) unary ;
```

但是这个规则有一个问题。就是它永远不会终止。

请记住，每个规则都需要匹配该优先级*或更高*优先级的表达式，因此我们还需要让它匹配主表达式。

```
unary          → ( "!" | "-" ) unary
               | primary ;
```

这样就行了。

其余规则都是二元运算符。我们将从乘法和除法规则开始。这是第一次尝试：

```
factor         → factor ( "/" | "*" ) unary
               | unary ;
```

该规则递归以匹配左操作数。这使规则能够匹配一系列乘法和除法表达式，例如`1 * 2 / 3`.将递归产生式放在左侧和`unary`右侧，使规则左结合且明确。

> 原则上，将乘法视为左结合还是右结合并不重要——无论哪种方式都会得到相同的结果。然而，在精度有限的现实世界中，舍入和溢出意味着关联性会影响乘法序列的结果。考虑：
> 
> print 0.1 * (0.2 * 0.3);
> print (0.1 * 0.2) * 0.3;
> 
> 在像 Lox 这样使用[IEEE 754](https://en.wikipedia.org/wiki/Double-precision_floating-point_format)双精度浮点数的语言中，第一个计算结果为`0.006`，而第二个 计算结果为`0.006000000000000001`。有时这种微小的差异很重要。?[这](https://docs.oracle.com/cd/E19957-01/806-3568/ncg_goldberg.html)是一个了解更多信息的好地方。

所有这些都是正确的，但规则主体中的第一个符号与规则头部相同的事实意味着此产生式是直接**左递归的**。一些解析技术，包括我们将要使用的技术，在左递归方面存在问题。（其他地方的递归，就像我们在 `unary`中所做 的间接递归`primary`不是问题。）

您可以定义许多匹配相同语言的语法。如何为特定语言建模的选择部分是品味问题，部分是务实问题。这条规则是正确的，但对于我们打算如何解析它来说并不是最佳的。我们将使用不同的规则，而不是左递归规则。

```
factor         → unary ( ( "/" | "*" ) unary )* ;
```

我们将factor表达式定义为乘法和除法的展开*序列*。这与前面的规则匹配相同的语法，但更好地反映了我们将编写的用于解析 Lox 的代码。对所有其他二元运算符优先级使用相同的结构，从而为我们提供了完整的表达式语法：

```
expression     → equality ;
equality       → comparison ( ( "!=" | "==" ) comparison )* ;
comparison     → term ( ( ">" | ">=" | "<" | "<=" ) term )* ;
term           → factor ( ( "-" | "+" ) factor )* ;
factor         → unary ( ( "/" | "*" ) unary )* ;
unary          → ( "!" | "-" ) unary
               | primary ;
primary        → NUMBER | STRING | "true" | "false" | "nil"
               | "(" expression ")" ;
```

这个语法比之前的那个更复杂，但作为补偿，我们已经消除了前一个的歧义。这正是制作解析器所需要的。

## 6.2 递归下降解析

有一整套解析技术，其名称主要是“L”和“R”的组合——[LL(k)](https://en.wikipedia.org/wiki/LL_parser)、[LR(1)](https://en.wikipedia.org/wiki/LR_parser)、LALR——[以及](https://en.wikipedia.org/wiki/LALR_parser)更奇特的野兽，如[解析器组合器](https://en.wikipedia.org/wiki/Parser_combinator)、[Earley 解析器](https://en.wikipedia.org/wiki/Earley_parser)、[调车场算法](https://en.wikipedia.org/wiki/Shunting-yard_algorithm), 和[Packrat 解析](https://en.wikipedia.org/wiki/Parsing_expression_grammar)。对于我们的第一个解释器，一种技术就足够了：**递归下降**。

递归下降是构建解析器的最简单方法，不需要使用复杂的解析器生成工具，如 Yacc、Bison 或 ANTLR。所需要的只是简单的手写代码。不过，不要被它的简单性所迷惑。递归下降解析器快速、健壮，并且可以支持复杂的错误处理。事实上，GCC、V8（Chrome 中的 JavaScript VM）、Roslyn（用 C# 编写的 C# 编译器）和许多其他重量级生产语言实现都使用递归下降。它很震撼。

递归下降被认为是**自上而下的解析器**，因为它从顶部或最外层的语法规则（此处是`expression`）开始，并在最终到达语法树的叶子之前一直向下进入嵌套的子表达式。这与像 LR 这样的自下而上的解析器形成对比，后者从主要表达式开始并将它们组合成越来越大的语法块。

> 它被称为“递归*下降*”，因为它沿着语法向下走。令人困惑的是，我们在谈论“高”和“低”优先级时也使用方向来比喻，但方向是相反的。在自上而下的解析器中，您首先到达优先级最低的表达式，因为它们可能依次包含优先级较高的子表达式。

![](./assets/a27beb3d620b75c043acabcad45a83e8b13f3c45.png)

> CS 人员确实需要聚在一起理顺他们的隐喻。甚至不要让我开始了解堆栈的生长方向或为什么树的根在上面。

递归下降解析器是将语法规则直接翻译成命令式代码。每个规则成为一个函数。规则的主体转换为大致如下的代码：

| 语法符号     | 代码表示            |
| -------- | --------------- |
| 终结符      | 匹配和使用token的代码   |
| 非终结符     | 调用该规则的函数        |
| `\|`     | `if`或`switch`声明 |
| `*`或者`+` | `while`或`for`循环 |
| `?`      | `if`语句          |

下降被描述为“递归”，因为当语法规则直接或间接引用自身时，这将转化为递归函数调用。

### 6.2.1 解析器类

每个语法规则都成为这个新类中的一个方法：

```java
package com.craftinginterpreters.lox;

import java.util.List;

import static com.craftinginterpreters.lox.TokenType.*;

class Parser {
  private final List<Token> tokens;
  private int current = 0;

  Parser(List<Token> tokens) {
    this.tokens = tokens;
  }
}
// lox/Parser.java, create new file
```

与扫描器类一样，解析器消耗一个展开的输入序列，只是现在正在读取token而不是字符。我们存储token列表并使用`current`指向急需等待解析的下一个token。

现在将直接运行表达式语法并将每个规则转换为 Java 代码。第一条规则`expression`简单地扩展为`equality`规则，所以这很简单。

```java
  private Expr expression() {
    return equality();
  }
// lox/Parser.java, add after Parser() 
```

用于解析语法规则的每个方法都会为该规则生成语法树并将其返回给调用者。当规则的主体包含一个非终结符——即对另一个规则的引用——我们调用另一个规则的方法。

> 这就是为什么左递归对于递归下降是有问题的。左递归规则的函数立即调用自身，它再次调用自身，依此类推，直到解析器遇到堆栈溢出并死掉。

equality规则稍微复杂一些。

```
equality       → comparison ( ( "!=" | "==" ) comparison )* ;
```

在 Java 中，这变成：

```java
 private Expr equality() {
    Expr expr = comparison();

    while (match(BANG_EQUAL, EQUAL_EQUAL)) {
      Token operator = previous();
      Expr right = comparison();
      expr = new Expr.Binary(expr, operator, right);
    }

    return expr;
  }
// lox/Parser.java, add after expression()
```

让我们仔细分析一下。正文中的第一个`comparison`非终结符转换为`comparison()`方法中的第一个调用。获取该结果并将其存储在局部变量中expr。

然后，`( ... )*`规则中的循环映射到一个`while`循环。我们需要知道何时退出该循环。可以看到，在规则内部，我们必须首先找到 `!=`或`==`标记。所以，如果没有检查其中之一，我们必须完成相等运算符的序列。此处使用方便的`match()`方法来表示该检查逻辑。

```java
   private boolean match(TokenType... types) {
    for (TokenType type : types) {
      if (check(type)) {
        advance();
        return true;
      }
    }

    return false;
  }
// lox/Parser.java, add after equality()
```

match 方法将检查当前token是否是任何给定类型。如果是，它会消耗token并返回`true`。否则，它返回`false`并单独保留当前标记。该`match()`方法是根据两个更基本的操作定义的。

如果当前前token是给定类型，则该`check()`方法返回`true`不像`match()`，它从不消耗token，它只检查。

```java
  private boolean check(TokenType type) {
    if (isAtEnd()) return false;
    return peek().type == type;
  }
// lox/Parser.java, add after match()  
```

而`advance()`方法消耗当前token并返回它，类似于扫描器的相应方法沿着字符流前进。

```java
  private Token advance() {
    if (!isAtEnd()) current++;
    return previous();
  }
// lox/Parser.java, add after check() 
```

下边这些方法是一些原子的辅组方法。

```java
  private boolean isAtEnd() {
    return peek().type == EOF;
  }

  private Token peek() {
    return tokens.get(current);
  }

  private Token previous() {
    return tokens.get(current - 1);
  }
// lox/Parser.java, add after advance() 
```

`isAtEnd()`检查是否用完了要解析的token。`peek()`返回尚未消费的当前token，`previous()`返回最近消费的token。后者使使用`match()`和访问刚刚匹配的token变得更容易。

这就是我们需要的大部分解析基础设施。我们刚刚说到哪了？是的，如果我们在`while`循环内`equality()`，那么我们知道找到了一个`!=`或`==`运算符并且必须解析一个相等表达式。

我们获取匹配的运算符token，以便可以跟踪我们拥有哪种相等表达式。然后我们再次调用`comparison()`来解析右边的操作数。将运算符和它的两个操作数组合成一个新的`Expr.Binary`语法树节点，然后循环。对于每次迭代，我们将结果表达式存储回同一个`expr`局部变量中。当我们通过一系列等式表达式时，会创建一个二元运算符节点的左结合嵌套树。

![](./assets/d415e1f508ecdbdd3d113050caadd72a30acf59c.png)

> 解析`a == b == c == d == e`。对于每次迭代，我们使用前一个作为左操作数创建一个新的二进制表达式。

一旦遇到不是相等运算符的标记，解析器就会退出循环。最后，它返回表达式。请注意，如果解析器从未遇到相等运算符，那么它永远不会进入循环。在这种情况下，该`equality()`方法有效地调用并返回`comparison()`。这样，此方法匹配相等运算符*或任何更高优先级*的运算符。

继续下一个规则...

```
comparison     → term ( ( ">" | ">=" | "<" | "<=" ) term )* ;
```

翻译成Java：

```java
  private Expr comparison() {
    Expr expr = term();

    while (match(GREATER, GREATER_EQUAL, LESS, LESS_EQUAL)) {
      Token operator = previous();
      Expr right = term();
      expr = new Expr.Binary(expr, operator, right);
    }

    return expr;
  }
// lox/Parser.java, add after equality() 
```

语法规则几乎相同，`equality`相应的代码也是如此。唯一的区别是我们匹配的运算符的token类型，以及我们为操作数调用的方法——现在是`term()`而不是`comparison()`.其余两个二元运算符规则遵循相同的模式。

按照先后顺序，先加减法：

> 如果想使用 Java 8，你可以创建一个辅助方法来解析左结合系列的二元运算符，给定一个标记类型列表，以及一个操作数方法句柄来简化这个冗余代码。

```java
  private Expr term() {
    Expr expr = factor();

    while (match(MINUS, PLUS)) {
      Token operator = previous();
      Expr right = factor();
      expr = new Expr.Binary(expr, operator, right);
    }

    return expr;
  }
// lox/Parser.java, add after comparison()  
```

最后是，乘法和除法：

```java
  private Expr factor() {
    Expr expr = unary();

    while (match(SLASH, STAR)) {
      Token operator = previous();
      Expr right = unary();
      expr = new Expr.Binary(expr, operator, right);
    }

    return expr;
  }
// lox/Parser.java, add after term()
```

这就是所有二元运算符，使用正确的优先级和结合性进行解析。我们正在爬上优先级层次结构，现在已经到达一元运算符。

```
unary          → ( "!" | "-" ) unary
               | primary ;
```

这个代码有点不同。

```java
  private Expr unary() {
    if (match(BANG, MINUS)) {
      Token operator = previous();
      Expr right = unary();
      return new Expr.Unary(operator, right);
    }

    return primary();
  }
// lox/Parser.java, add after factor()
```

同样，我们查看当前token以查看如何解析。如果是`!`or`-`，我们必须有一个一元表达式。在这种情况下，我们获取token，然后再次递归调用`unary()`以解析操作数。将所有内容包装在一元表达式语法树中，就完成了。

> 解析器提前查看即将到来的标记以决定如何解析这一事实将递归下降归入**预测解析器**的范畴。

否则，一定已经达到了最高级别的优先级，即primary表达式。

```
primary        → NUMBER | STRING | "true" | "false" | "nil"
               | "(" expression ")" ;
```

该规则的大多数情况都是简单终端，因此解析很简单。

```java
  private Expr primary() {
    if (match(FALSE)) return new Expr.Literal(false);
    if (match(TRUE)) return new Expr.Literal(true);
    if (match(NIL)) return new Expr.Literal(null);

    if (match(NUMBER, STRING)) {
      return new Expr.Literal(previous().literal);
    }

    if (match(LEFT_PAREN)) {
      Expr expr = expression();
      consume(RIGHT_PAREN, "Expect ')' after expression.");
      return new Expr.Grouping(expr);
    }
  }
// lox/Parser.java, add after unary()  
```

有趣的分支是处理括号的分支。在我们匹配一个开始`(`并解析其中的表达式之后，我们*必须*找到一个`)`标记。如果我们不这样做，那就是一个错误。

## 6.3 语法错误

解析器实际上有两个工作：

1. 给定一个有效的token序列，生成一个相应的语法树。

2. 给定一个*无效*的token序列，检测任何错误并告诉用户他们的错误。

别小看第二份工作的重要性！在现代 IDE 和编辑器中，解析器不断地重新解析代码——通常是在用户仍在编辑代码的时候——以便语法高亮和支持自动完成等功能。这意味着它会一直遇到不完整、半错状态*的代码。*

当用户没有意识到语法错误时，解析器就会帮助引导他们回到正确的路径。它报告错误的方式是您语言的用户界面的很大一部分。良好的语法错误处理很难。根据定义，代码未处于明确定义的状态，因此没有万无一失的方法来了解用户的*意图*。解析器无法读懂你的想法。

至少现在还没有。随着机器学习如今的发展，谁知道未来会发生什么？

当解析器遇到语法错误时，有几个硬性要求。解析器必须：

- **检测并报告错误**。如果它没有检测到错误并将生成的格式错误的语法树传递给解释器，则可能会引发各种恐怖事件。
  
  从哲学上讲，如果没有检测到错误并且解释器运行代码，那么它*真的*是错误吗？

- **避免碰撞或悬挂**。语法错误是生活中的一个事实，面对它们，语言工具必须是健壮的。不允许出现段错误或陷入无限循环。虽然源代码可能不是有效?*代码*，但它仍然是解析器的有效输入，因为用户使用解析器来了解允许的语法。

如果您想完全参与解析器游戏，这些就是赌注，但您真的想提高赌注。一个体面的解析器应该：

- **快点**。计算机比解析器技术刚发明时快数千倍。需要优化解析器以便它可以在喝咖啡休息时间处理整个源文件的日子已经过去了。但是程序员的期望值已经上升得同样快，甚至更快。他们希望他们的编辑在每次击键后在几毫秒内重新解析文件。

- **报告尽可能多的不同错误**。在第一个错误后中止很容易实现，但是如果用户每次修复他们认为是文件中的一个错误时，都会出现一个新错误，这对用户来说会很烦人。他们想看到所有的错误。

- **最小化级联错误**。一旦发现一个错误，解析器就不再真正知道发生了什么。它试图让自己回到正轨并继续前进，但如果它感到困惑，它可能会报告大量的幽灵错误，这些错误并不表明代码中存在其他真正的问题。当第一个错误被修复时，那些幻影就消失了，因为它们只反映了解析器自己的困惑。级联错误很烦人，因为它们会吓到用户认为他们的代码处于比实际更糟糕的状态。

最后两点很关键。我们希望报告尽可能多的单独错误，但我们不想报告那些仅仅是早期错误的副作用。

解析器响应错误并继续查找后续错误的方式称为**错误恢复**。这是 60 年代的热门研究课题。那时，你会把一叠打孔卡交给秘书，第二天回来看编译器是否成功。对于如此缓慢的迭代循环，您*真的*希望一次性找出代码中的每一个错误。

今天，当解析器在您完成输入之前完成时，这已经不是什么问题了。简单、快速的错误恢复很好。

### 6.3.1  恐慌模式错误恢复

![](./assets/0b14a5b14a26878ab2be2514a2732bab969f6537.png)

在过去设计的所有恢复技术中，最经得起时间考验的一种被称为——有点令人担忧——**恐慌模式**。一旦解析器检测到错误，它就会进入恐慌模式。它知道至少有一个token没有意义，因为它的当前状态处于一些语法产生式的中间。

在它可以返回解析之前，它需要使其状态和即将到来的token序列对齐，以便下一个token与正在解析的规则相匹配。这个过程称为**同步**。

为此，我们在语法中选择一些规则来标记同步点。解析器通过跳出任何嵌套产生式来修复其解析状态，直到它返回到该规则。然后它通过丢弃token来同步token流，直到它到达可以出现在规则中的那个点的token。

隐藏在那些被丢弃的token中的任何额外的真实语法错误都不会被报告，但这也意味着作为初始错误副作用的任何错误的级联错误也不会被*错误地*报告，这是一个不错的权衡。

语法中传统的同步位置是在语句之间。我们还没有这些，所以我们不会在本章中实际同步，但会为以后准备好机制。

### 6.3.2 进入恐慌模式

回到我们围绕错误恢复进行这一边旅行之前，我们正在编写代码来解析带括号的表达式。解析表达式后，解析器通过调用`consume()`查找结束符`)` 。最后，这是该方法：

```java
  private Token consume(TokenType type, String message) {
    if (check(type)) return advance();

    throw error(peek(), message);
  }
// lox/Parser.java, add after match()
```

它类似于`match()`检查下一个token是否为预期类型。如果是，它会消耗token并且一切正常。如果那里有其他token，那么就遇到了错误。我们通过调用它来报告它：

```java
private ParseError error(Token token, String message) {
    Lox.error(token, message);
    return new ParseError();
  }
// lox/Parser.java, add after previous() 
```

首先，通过调用向用户显示错误：

```java
  static void error(Token token, String message) {
    if (token.type == TokenType.EOF) {
      report(token.line, " at end", message);
    } else {
      report(token.line, " at '" + token.lexeme + "'", message);
    }
  }
// lox/Lox.java, add after report()  
```

这会报告给定标记的错误。它显示token的位置和token本身。这将在稍后派上用场，因为我们在整个解释器中使用token来跟踪代码中的位置。

我们报告错误后，用户知道他们的错误，但是*解析器*接下来做什么呢？回到`error()`，我们创建并返回一个 ParseError，这个新类的一个实例：

```java
class Parser {
  private static class ParseError extends RuntimeException {}

  private final List<Token> tokens;
// lox/Parser.java, nest inside class Parser
```

这是我们用来展开解析器的简单哨兵类。该`error()`方法*返回*错误而不是*抛出*错误，因为我们想让解析器内部的调用方法决定是否展开。一些解析错误发生在解析器不太可能进入怪异状态并且我们不需要同步的地方。在那些地方，我们只是报告错误并继续前进。

例如，Lox 限制了可以传递给函数的参数数量。如果传递的参数太多，解析器需要报告该错误，但它可以而且应该继续解析额外的参数，而不是惊慌失措并进入恐慌模式。

> 处理常见语法错误的另一种方法是使用**错误产生**式。*您使用成功*匹配*错误*语法的规则来扩充语法。解析器安全地解析它，但随后将其报告为错误，而不是生成语法树。
> 
> 例如，某些语言具有一元运算`+`符，例如`+123`，但 Lox 没有。当解析器偶然`+`发现表达式开头的 a 时，我们可以扩展一元规则以允许它，而不是感到困惑。
> 
> ```java
> unary → ( "!" | "-" | "+" ) unary
>       | primary ;
> ```

> 这让解析器在`+`不进入恐慌模式或让解析器处于奇怪状态的情况下进行消费。
> 
> 错误产生式运作良好，因为您，解析器作者，知道代码是*如何*错误的以及用户可能试图做什么。这意味着您可以提供更有帮助的消息让用户回到正轨，例如“不支持一元 '+' 表达式。”?成熟的解析器往往会像藤壶一样积累错误产生式，因为它们可以帮助用户修复常见错误。

但是，在我们的例子中，语法错误非常严重，我们想要恐慌和同步。丢弃标记非常容易，但是我们如何同步解析器自身的状态呢？

### 6.3.3 同步递归下降解析器

通过递归下降，解析器的状态——规定它处于识别过程中的状态——不会明确存储在字段中。相反，我们使用 Java 自己的调用堆栈来跟踪解析器正在做什么。正在解析的每个规则都是堆栈上的一个调用帧。为了重置该状态，我们需要清除那些调用帧。

在 Java 中执行此操作的自然方法是异常。当我们想要同步时，我们*抛出*那个 ParseError 对象。在我们正在同步的语法规则的方法的更高层，我们将捕获它。因为我们在语句边界上同步，所以我们会在那里捕获异常。捕获异常后，解析器处于正确的状态。剩下的就是同步token。

我们想丢弃token，直到我们正好在下一条语句的开头。这个边界很容易发现——这是我们选择它的主要原因之一。*在*分号之后，我们可能已经完成了一条语句。大多数语句以关键字开头—`for`、`if`、`return`、`var`等。当下*一个*token是其中任何一个时，我们可能就要开始一个语句了。

> 我说“可能”是因为我们可以在`for`循环中使用分号分隔子句。我们的同步并不完美，但没关系。我们已经准确地报告了第一个错误，所以之后的一切都是“尽力而为”。

此方法封装了该逻辑：

```java
 private void synchronize() {
    advance();

    while (!isAtEnd()) {
      if (previous().type == SEMICOLON) return;

      switch (peek().type) {
        case CLASS:
        case FUN:
        case VAR:
        case FOR:
        case IF:
        case WHILE:
        case PRINT:
        case RETURN:
          return;
      }

      advance();
    }
  }
//lox/Parser.java, add after error()  
```

它会丢弃token，直到它认为它找到了语句边界。捕捉到 ParseError 后，我们将调用它，然后我们有望恢复同步。当它运行良好时，我们已经丢弃了无论如何都可能导致级联错误的token，现在我们可以从下一条语句开始解析文件的其余部分。

但是现在，我们还没法看到这个方法的实际效果，因为我们还没有语句。我们将[在几章中谈到](http://craftinginterpreters.com/statements-and-state.html)这一点。现在，如果发生错误，我们会惊慌失措并一路回绕到顶部并停止解析。因为无论如何我们只能解析一个表达式，所以这没什么大损失。

## 6.4 连接解析器

我们现在主要完成了解析表达式。还有一个地方我们需要添加一点错误处理。当解析器通过每个语法规则的解析方法下降时，它最终会命中`primary()`.如果那里的所有情况都不匹配，则意味着我们正坐在一个无法启动表达式的token上。我们也需要处理该错误。

```java
  if (match(LEFT_PAREN)) {
      Expr expr = expression();
      consume(RIGHT_PAREN, "Expect ')' after expression.");
      return new Expr.Grouping(expr);
    }

    throw error(peek(), "Expect expression.");
  }
// lox/Parser.java, in primary()
```

这样，解析器中剩下的就是定义一个初始方法来启动它。很自然地，该方法被称为`parse()`.

```java
  Expr parse() {
    try {
      return expression();
    } catch (ParseError error) {
      return null;
    }
  }
// lox/Parser.java, add after Parser() 
```

稍后我们将语句添加到语言中时，我们将重新访问此方法。现在，它解析单个表达式并返回它。我们还有一些临时代码可以退出恐慌模式。语法错误恢复是解析器的工作，所以我们不希望 ParseError 异常逃逸到解释器的其余部分。

当确实发生语法错误时，此方法返回`null`。没关系。解析器承诺不会因无效语法而崩溃或挂起，但它不承诺在发现错误时返回可用的语法树。一旦解析器报告错误，`hadError`就会设置并跳过后续阶段。

最后，我们可以将我们全新的解析器连接到主 Lox 类并进行尝试。我们仍然没有解释器，所以现在，我们将解析为语法树，然后使用上一章的 [AstPrinter](http://craftinginterpreters.com/representing-code.html#a-not-very-pretty-printer)来类显示它。

删除旧代码以打印扫描的令牌并将其替换为：

```java
  List<Token> tokens = scanner.scanTokens();
    Parser parser = new Parser(tokens);
    Expr expression = parser.parse();

    // Stop if there was a syntax error.
    if (hadError) return;

    System.out.println(new AstPrinter().print(expression));
  }
// lox/Lox.java, in run(), replace 5 lines
```

恭喜你，你已经跨过了门槛！这就是手写解析器的全部内容。我们将在后面的章节中用赋值、语句和其他东西来扩展语法，但这些都不会比我们在这里处理的二元运算符更复杂。

> 可以定义比 Lox 更复杂的语法，使用递归下降很难解析。当您可能需要向前看大量标记以弄清楚您所坐的是什么时，预测解析会变得很棘手。
> 
> 实际上，大多数语言都旨在避免这种情况。即使在它们不存在的情况下，您通常也可以毫不费力地绕过它。如果你可以使用递归下降来解析 C++——许多 C++ 编译器都这样做——你就可以解析任何东西。

启动解释器并输入一些表达式。看看它如何正确处理优先级和结合性？少于 200 行代码还不错。

---

## [挑战](http://craftinginterpreters.com/parsing-expressions.html#challenges)

1. 在 C 中，块是一种语句形式，它允许您在需要单个语句的地方打包一系列语句。[逗号运算符](https://en.wikipedia.org/wiki/Comma_operator)是表达式的类似语法。可以在需要单个表达式的地方给出逗号分隔的一系列表达式（函数调用的参数列表内除外）。在运行时，逗号运算符计算左操作数并丢弃结果。然后它评估并返回正确的操作数。
   
   添加对逗号表达式的支持。赋予它们与 C 中相同的优先级和结合性。编写语法，然后实现必要的解析代码。

2. 同样，添加对 C 风格条件或“三元”运算符的支持`?:`。在`?`和`:`之间允许什么优先级别，整个运算符是左结合的还是右结合的？

3. 添加错误产生式以处理没有左操作数出现的每个二元运算符。换句话说，检测出现在表达式开头的二元运算符。将其报告为错误，同时解析并丢弃具有适当优先级的右侧操作数。

## [设计说明：逻辑与历史](http://craftinginterpreters.com/parsing-expressions.html#design-note)

假设我们决定向 Lox添加按位运算符`&`和`|`运算符。我们应该把它们放在优先级的什么位置？C—以及大多数追随 C 脚步的语言—将它们放在`==`.这被广泛认为是一个错误，因为这意味着像测试标志这样的常见操作需要括号。

```java
if (flags & FLAG_MASK == SOME_FLAG) { ... } // Wrong.
if ((flags & FLAG_MASK) == SOME_FLAG) { ... } // Right.
```

我们是否应该为 Lox 修复此问题并将按位运算符置于优先级表中高于 C 的位置？我们可以采取两种策略。

您几乎不想将`==`表达式的结果用作按位运算符的操作数。通过使按位绑定更紧密，用户不需要经常加上括号。因此，如果我们这样做，并且用户假设优先级的选择是合乎逻辑的以最小化括号，那么他们很可能会正确地推断出来。

这种内部一致性使语言更容易学习，因为用户必须偶然发现然后纠正的边缘情况和异常更少。这很好，因为在用户可以使用我们的语言之前，他们必须将所有这些语法和语义加载到他们的头脑中。一种更简单、更理性的语言*是有道理的*。

但是，对于许多用户来说，将我们的语言的思想融入他们的湿件中有一个更快的捷径——*使用他们已经知道的概念*。我们语言的许多新来者将来自其他一种或多种语言。如果我们的语言使用一些与那些相同的语法或语义，那么用户学习（和*忘却*）的东西就会少得多。

这对语法特别有用。今天您可能不太记得它，但回想起来，当您学习第一门编程语言时，代码可能看起来很陌生且难以接近。经过艰苦的努力，你才学会阅读和接受它。如果你为你的新语言设计了一种新的语法，你就会迫使用户重新开始这个过程。

利用用户已知的知识是您可以用来简化语言采用的最强大的工具之一。几乎不可能高估它的价值。但它面临着一个棘手的问题：当用户都知道的东西很糟糕时会发生*什么*？C 的按位运算符优先级是一个没有意义的错误。但这是一个*熟悉*的错误，数百万人已经习惯并学会了忍受。

您是否忠于您的语言自身的内在逻辑而忽略历史？你是从一张白纸和首要原则开始的吗？或者您是否将您的语言编织到丰富的编程历史中，并通过从他们已经知道的东西开始来帮助您的用户？

这里没有完美的答案，只有取舍。你我都明显偏向于喜欢新奇的语言，所以我们自然而然地倾向于焚毁历史书，开始自己的故事。

实际上，最好充分利用用户已经知道的内容。让他们使用您的语言需要一个巨大的飞跃。你越能缩小鸿沟，就会有越多的人愿意跨越它。但是你不能*总是*坚持历史，否则你的语言不会有任何新的和令人信服的东西来给人们一个跳过的*理由。*