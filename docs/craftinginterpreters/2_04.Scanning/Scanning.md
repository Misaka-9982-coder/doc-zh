# 扫描

> 大咬一口。任何值得做的事情都值得过度努力。
> 
> -- Robert A. Heinlein,*Time Enough for Love*

任何编译器或解释器的第一步都是扫描。扫描器将原始源代码作为一系列字符，并将其分组为一系列称为**tokens**的块。这些是构成语言语法的有意义的“单词”和“标点符号”。

> 多年来，这项任务被称为“扫描(scanning)”和“分词(lexing)”（“词法分析”的缩写）。回到过去，当计算机和 Winnebagos 一样大但内存比手表还少时，有些人使用“扫描器(scanner)”只是指处理从磁盘读取原始源代码字符并将其缓冲在内存中的代码段。然后“lexing”是对字符做有用的事情的下一个阶段。
> 
> 如今，将源文件读入内存是微不足道的，因此它(scanner)很少成为编译器中的一个独立阶段。因此，这两个术语基本上可以互换。

扫描对我们来说也是一个很好的起点，因为代码不是很难，基本结构是一个庞大的`switch`语句。在我们稍后处理一些更有趣的材料之前，它会帮助我们热身。到本章结束时，我们将拥有一个功能齐全、快速的扫描器(scanner)，它读取任意 Lox 源代码字符串并生成将在下一章中输入解析器的tokens。

## 4.1 解释器框架

由于这是第一个真正的章节，在开始实际扫描一些代码之前，我们需要勾勒出我们的解释器 jlox 的基本框架。一切都从 Java 中的类开始。

```java
package com.craftinginterpreters.lox;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.nio.charset.Charset;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;

public class Lox {
  public static void main(String[] args) throws IOException {
    if (args.length > 1) {
      System.out.println("Usage: jlox [script]");
      System.exit(64); 
    } else if (args.length == 1) {
      runFile(args[0]);
    } else {
      runPrompt();
    }
  }
}

// lox/Lox.java, create new file
```

> 对于退出代码，我使用了 UNIX [“sysexits.h”](https://www.freebsd.org/cgi/man.cgi?query=sysexits&apropos=0&sektion=0&manpath=FreeBSD+4.3-RELEASE&format=html)头文件中定义的约定。这是我能找到的最接近标准的东西。

将其粘贴到文本文件中，然后设置好您的 IDE 或 Makefile 文件。当你准备好时，我会在这里。好吗？好的！

Lox 是一种脚本语言，这意味着它直接从源代码执行。我们的解释器支持两种运行代码的方式。如果你从命令行启动 jlox 并给它一个文件路径，它会读取文件并执行它。

```java
private static void runFile(String path) throws IOException {
   byte[] bytes = Files.readAllBytes(Paths.get(path));
   run(new String(bytes, Charset.defaultCharset()));
}

// lox/Lox.java, add after main()
```

如果想与解释器进行更亲密的交流对话，还可以以交互方式运行它。在没有任何参数的情况下启动 jlox，它会启动一个命令提示符中，可以在其中一次输入一行代码并执行。

> 交互式提示也称为“REPL”（发音像“rebel”但带有“p”）。这个名字来自 Lisp，其中实现一个就像围绕一些内置函数包装一个循环一样简单：
> 
> (print (eval (read)))
> 
> 从最内存嵌套的调用向外工作，**R**ead一行输入，**E**valuate它，**P**rint结果，然后**L**oop再来一遍。

```java
  private static void runPrompt() throws IOException {
    InputStreamReader input = new InputStreamReader(System.in);
    BufferedReader reader = new BufferedReader(input);

    for (;;) { 
      System.out.print("> ");
      String line = reader.readLine();
      if (line == null) break;
      run(line);
    }
  }
// lox/Lox.java, add after runFile()
```

`readLine()`顾名思义，该函数从命令行读取用户的一行输入作为结果返回。要终止交互式命令行应用程序，通常键入 Control-D。这样做会向程序发出“end-of-file”信号。当这种情况发生时`readLine()`返回`null`，所以用它检查是否退出循环。

提示符和文件运行器都是这个核心功能的简单封装：

```java
private static void run(String source) {
    Scanner scanner = new Scanner(source);
    List<Token> tokens = scanner.scanTokens();

    // For now, just print the tokens.
    for (Token token : tokens) {
      System.out.println(token);
    }
  }
// lox/Lox.java, add after runPrompt() 
```

它不是非常有用，因为还没有编写解释器，但是你知道吗？现在，它会打印出接下来编写的扫描器(scanner)产出的tokens，以便可以查看我们是否取得了进展。

### 4.1.1 错误处理

在我们进行设置时，基础设施的另一个关键部分是*错误处理*。教科书有时会掩饰这一点，因为它更像是一个实际问题，而不是正式的计算机科学问题。但是，如果您关心创建一种实际*可用*的语言，那么优雅地处理错误是至关重要的。

语言提供的用于处理错误的工具构成了其用户界面的很大一部分。当用户的代码正常工作时，他们根本不会考虑我们的语言，他们的脑子里全是关于*他们的程序*。通常只有当出现问题时，他们才会注意到我们的错误处理。

当发生这种情况时，我们有责任向用户提供他们需要的所有信息，以了解出了什么问题，并温和地引导他们回到他们试图去的地方。做好这件事意味着从现在开始，在解释器的整个实现过程中考虑错误处理。

> 说了这么多，对于*这个*解释器，我们将构建的是非常简单的框架。我很想谈论交互式调试器、静态分析器和其他有趣的东西，但是笔里只有这么多墨水。

```java
  static void error(int line, String message) {
    report(line, "", message);
  }

  private static void report(int line, String where,
                             String message) {
    System.err.println(
        "[line " + line + "] Error" + where + ": " + message);
    hadError = true;
  }
// lox/Lox.java, add after run()
```

这个`error()`函数和`report()`辅助函数告诉用户在确定的行上发生了一些语法错误。这确实是能够声称您甚至*有*错误报告的最低限度。想象一下，如果你不小心在某个函数调用中留下了一个悬空的逗号，并且解释器打印出来：

```java
Error: Unexpected "," somewhere in your code. Good luck finding it!
```

那不是很有帮助。我们至少需要将它们指向正确的行号。更好的是开始和结束列，这样用户就知道*该*行的位置。*比这*更好的是向用户*显示*违规行，例如：

```java
Error: Unexpected "," in argument list.

    15 | function(first, second,);
                               ^-- Here.
```

我很想在本书中实现类似的东西，但老实说它有很多蹩脚的字符串操作代码。对用户非常有用，但读起来不是很有趣，技术上也不是很有趣。所以我们只使用行号。在你自己的解释器中，请照我说的做，而不是像我做的那样。

我们在主 Lox 类中保留此错误报告功能的主要原因是因为该`hadError`字段在这里定义：

```java
public class Lox {
  static boolean hadError = false;
// lox/Lox.java, in class Lox
```

我们将使用它来确保我们不会尝试执行具有已知错误的代码。此外，它还允许我们以非零代码退出，就像一个好的命令行公民应该做的那样。

```java
 run(new String(bytes, Charset.defaultCharset()));

    // Indicate an error in the exit code.
    if (hadError) System.exit(65);
  }
// lox/Lox.java, in runFile()
```

我们需要在交互式循环中重置此标志。如果用户犯了错误，它不应该终止他们的整个会话。

```java
   run(line);
      hadError = false;
    }
// lox/Lox.java, in runPrompt()
```

我在此处提取错误报告而不是将其塞入扫描器和其他可能发生错误的阶段的另一个原因是提醒您，将*生成*错误的代码与*报告*错误的代码分开是一种很好的工程实践。

前端的各个阶段会检测错误，但知道如何将错误呈现给用户并不是他们的工作。在功能齐全的语言实现中，您可能会通过多种方式显示错误：在 stderr 上、在 IDE 的错误窗口中、记录到文件中等。您不希望代码在扫描器和解析器中到处都是。

理想情况下，我们会有一个实际的抽象，某种传递给扫描器和解析器的“ErrorReporter”接口，这样我们就可以替换不同的报告策略。对于这里的简单解释器，我没有这样做，但我至少将错误报告代码移到了另一个类中。

> 当我第一次实现jlox 时，我就是这样。我最终把它去掉了，因为它感觉对于本书中的最小解释器来说是过度设计的。

有了一些基本的错误处理，我们的应用程序外壳就准备好了。一旦有了一个带有`scanTokens()`方法的 Scanner 类，就可以开始运行它了。在开始之前，让更进一步地了解tokens是什么。

## 4.2  Lexemes 和Tokens

这是一行 Lox 代码：

```c
var language = "lox";
```

这里的`var`是声明变量的关键字。三个字符序列“v-a-r”有确定的意义。但是如果我们从`language` 的中间抽出三个字母，比如“gua”，它们本身就没有任何意义。

这就是词法分析的意义所在。我们的工作是扫描字符列表并将它们组合成仍然代表某些东西的最小序列。这些字符块中的每一个都称为**词素(lexeme)**。在该示例代码行中，词素(lexeme)是：

![](./assets/1bf290fc6c584d858c62344809f64ae0934a5a1b.png)

词素(lexeme)只是**源代码的原始子串**。然而，在将字符序列分组为词素(lexeme)的过程中，我们也偶然发现了一些其他有用的信息。当获取词素(lexeme)并将其与其他数据捆绑在一起时，结果就是一个标记(token)。它包括有用的东西，例如：

### 4.2.1 token类型

关键字是语言语法结构的一部分，所以解析器通常有这样的代码，“如果下一个token是`while`那么做...” , 这意味着解析器不仅想知道它有某个标识符的词素，而且想知道它有一个*保留*字，以及它是*哪个*关键字。

解析器可以通过比较字符串对原始词素中的标记进行分类，但这很慢而且有点难看。相反，当我们识别出一个词素时，我们也会记住它代表的是哪种词素*。*对于每个关键字、运算符、标点符号和文字类型，我们都有不同的类型。

> 毕竟，字符串比较最终会查看单个字符，这不是扫描器的工作吗？

```java
package com.craftinginterpreters.lox;

enum TokenType {
  // Single-character tokens.
  LEFT_PAREN, RIGHT_PAREN, LEFT_BRACE, RIGHT_BRACE,
  COMMA, DOT, MINUS, PLUS, SEMICOLON, SLASH, STAR,

  // One or two character tokens.
  BANG, BANG_EQUAL,
  EQUAL, EQUAL_EQUAL,
  GREATER, GREATER_EQUAL,
  LESS, LESS_EQUAL,

  // Literals.
  IDENTIFIER, STRING, NUMBER,

  // Keywords.
  AND, CLASS, ELSE, FALSE, FUN, FOR, IF, NIL, OR,
  PRINT, RETURN, SUPER, THIS, TRUE, VAR, WHILE,

  EOF
}
// lox/TokenType.java, create new file
```

### 4.2.2 字面值

字面值有词素——数字和字符串等。由于扫描器必须遍历文字中的每个字符才能正确识别它，它还可以将值的文本表示转换为稍后将由解释器使用的实时运行时对象。

### 4.2.3 定位信息

在之前我在宣扬错误处理的好处时，我们看到需要告诉用户*哪里*发生了错误。并从这里开始跟踪。在简单的解释器中，我们只注意token出现在哪一行，但更复杂的实现也包括列和长度。

> 一些token实现将位置存储为两个数字：从源文件开头到词素开头的偏移量，以及词素的长度。扫描器无论如何都需要知道这些，所以没有额外的开销来计算它们。
> 
> 通过回头查看源文件并计算前面的换行符，可以稍后将偏移量转换为行和列位置。这听起来很慢，而且确实如此。*但是，只有在需要实际向用户显示一行和一列时才*需要这样做。大多数标记永远不会出现在错误消息中。对于那些，提前计算位置信息的时间越少越好。

我们获取所有这些数据并将其包装在一个类中。

```java
package com.craftinginterpreters.lox;

class Token {
  final TokenType type;
  final String lexeme;
  final Object literal;
  final int line; 

  Token(TokenType type, String lexeme, Object literal, int line) {
    this.type = type;
    this.lexeme = lexeme;
    this.literal = literal;
    this.line = line;
  }

  public String toString() {
    return type + " " + lexeme + " " + literal;
  }
}
// lox/Token.java, create new file
```

现在我们有一个具有足够结构的对象，可用于解释器的所有后续阶段。

## 4.3 正则和表达式

既然我们知道要产出什么，那么让我们生产它。扫描器的核心是一个循环。从源代码的第一个字符开始，扫描器找出该字符属于哪个词素(lexeme)，并使用它和属于该词素(lexeme)的任何后续字符。当它到达该词素(lexeme)的末尾时，它会产出一个token。

然后它循环并再次执行，从源代码中的下一个字符开始。它一直这样做，吃字符，偶尔，呃，产出标记，直到它到达输入的末尾。

![](./assets/59656726271a04ab7f4a6bf03c59838860062ce6.png)

> 词法分析器。

在循环中，我们通过查看一些字符来确定它“匹配”了哪种词素(lexeme)，这部分听起来很熟悉。如果您知道正则表达式，可能会考虑为每种词素(lexeme)定义一个正则表达式并使用它们来匹配字符。例如，对于标识符（变量名等），Lox 具有与 C 相同的规则。用下边这个正则表达式匹配：

```
[a-zA-Z_][a-zA-Z_0-9]*
```

如果您确实想到了正则表达式，那么您的直觉是很深刻的。确定特定语言如何将字符分组为词素(lexeme)的规则称为其**词法**。在 Lox 中，与在大多数编程语言中一样，词法的规则非常简单，足以将语言归类为**[regular language](https://en.wikipedia.org/wiki/Regular_language)**。这与正则表达式中的“正则”相同。

> 如此掩饰这个理论让我很痛苦，尤其是当它像我认为的[乔姆斯基层次结构](https://en.wikipedia.org/wiki/Chomsky_hierarchy)和[有限状态机](https://en.wikipedia.org/wiki/Finite-state_machine)一样有趣时 。但老实说，其他书籍比我能更好地介绍这一点。[*Compilers: Principles, Techniques, and Tools*](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)（俗称“龙书”）是规范参考。

如果愿意，可以使用正则表达式非常精确地*识别*Lox 的所有不同词素(lexeme)，并且有一堆有趣的理论支持为什么会这样以及它意味着什么。像[Lex](http://dinosaur.compilertools.net/lex/)或[Flex](https://github.com/westes/flex)这样的工具就是专门为让你这样做而设计的——向它们扔一些正则表达式，它们就会给你一个完整的扫描器。

> Lex 由 Mike Lesk 和 Eric Schmidt 创建。是的，就是谷歌执行主席埃里克施密特。我并不是说编程语言是通往财富和名望的必经之路，但我们至少*可以算出一位超级亿万富翁。*

由于我们的目标是了解扫描器如何执行其操作，因此我们不会委托该任务。我们是关于手工制品的。

## 4.4 Scanner类

事不宜迟，让我们自己完成一个扫描器。

```java
package com.craftinginterpreters.lox;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static com.craftinginterpreters.lox.TokenType.*; 

class Scanner {
  private final String source;
  private final List<Token> tokens = new ArrayList<>();

  Scanner(String source) {
    this.source = source;
  }
}
// lox/Scanner.java, create new file
```

> 我知道有些人认为静态导入是糟糕的风格，但它们使我不必在扫描器和解析器上到处乱扔`TokenType.`。原谅我，但书中的每个字符都很重要。

我们将原始源代码存储为一个简单的字符串，并且准备好一个列表来填充将要生成的token。前面提到的循环看起来像这样：

```java
  List<Token> scanTokens() {
    while (!isAtEnd()) {
      // We are at the beginning of the next lexeme.
      start = current;
      scanToken();
    }

    tokens.add(new Token(EOF, "", null, line));
    return tokens;
  }
// lox/Scanner.java, add after Scanner() 
```

扫描器通过遍历源代码，添加token，直到用完所有字符。然后它附加一个最后的“文件结束”标记。这不是严格需要的，但它使解析器更干净一些。

这个循环依赖于几个字段来跟踪扫描器在源代码中的位置。

```java
  private final List<Token> tokens = new ArrayList<>();
  private int start = 0;
  private int current = 0;
  private int line = 1;

  Scanner(String source) {
// lox/Scanner.java, in class Scanner  
```

`start`和`current`字段是索引到字符串中的偏移量。该`start`字段指向正在扫描的词素(lexeme)中的第一个字符，且`current`指向当前正在考虑的字符。`line`字段跟踪`current`所在的源码行号位置，因此可以生成知道其位置的标记。

然后我们有一个小辅助函数，它告诉我们是否已经消耗了所有字符。

```java
 private boolean isAtEnd() {
    return current >= source.length();
  }
// lox/Scanner.java, add after scanTokens() 
```

## 4.5 识别lexemes

在每一轮循环中，扫描一个token。这是扫描器的真正核心。我们将从简单的开始。想象一下，如果每个词素(lexeme)只有一个字符长。您需要做的就是使用下一个字符并为其选择一个token类型。某些词素(lexeme)在 Lox*中*只是一个字符，所以让我们从这些开始。

```java
  private void scanToken() {
    char c = advance();
    switch (c) {
      case '(': addToken(LEFT_PAREN); break;
      case ')': addToken(RIGHT_PAREN); break;
      case '{': addToken(LEFT_BRACE); break;
      case '}': addToken(RIGHT_BRACE); break;
      case ',': addToken(COMMA); break;
      case '.': addToken(DOT); break;
      case '-': addToken(MINUS); break;
      case '+': addToken(PLUS); break;
      case ';': addToken(SEMICOLON); break;
      case '*': addToken(STAR); break; 
    }
  }
// lox/Scanner.java, add after scanTokens() 
```

> 想知道为什么`/`不在这里？别担心，我们会解决的。

同样，我们需要一些辅助方法。

```java
  private char advance() {
    return source.charAt(current++);
  }

  private void addToken(TokenType type) {
    addToken(type, null);
  }

  private void addToken(TokenType type, Object literal) {
    String text = source.substring(start, current);
    tokens.add(new Token(type, text, literal, line));
  }
// lox/Scanner.java, add after isAtEnd() 
```

该`advance()`方法使用源文件中的下一个字符并将其返回。哪里`advance()`是输入，哪里是`addToken()`输出。它获取当前词素的文本并为其创建一个新的标记。我们将很快使用其他重载来处理具有文字值的标记。

### 4.5.1  词法错误

在深入之前，让我们花点时间考虑一下词法层面的错误。如果用户向解释器抛出一个包含 Lox 不使用的某些字符的源文件，比如 `@#^`, 会发生什么？目前，这些字符被默默地丢弃了。Lox 语言不使用它们，但这并不意味着解释器可以假装它们不存在。相反，我们报告一个错误。

```java
      case '*': addToken(STAR); break; 

      default:
        Lox.error(line, "Unexpected character.");
        break;
    }
// lox/Scanner.java, in scanToken()
```

请注意，错误字符仍会被之前对`advance()` 的调用所*消耗*。这很重要，这样我们就不会陷入无限循环。

请注意，我们将接着*继续扫描*。程序后面可能还有其他错误。如果我们一次检测到尽可能多的这些，它会给用户带来更好的体验。否则，他们看到一个小错误并修复它，又会出现下一个错误，依此类推。语法错误 Whac-A-Mole 一点都不好玩。

（别担心。因为`hadError`设置了，我们永远不会尝试*执行*任何代码，即使我们继续扫描它的其余部分。）

> 该代码分别报告每个无效字符，因此如果用户不小心粘贴了一大块奇怪的文本，这会给用户带来大量错误。将一系列无效字符合并为一个错误会提供更好的用户体验。

### 4.5.2 操作符 operators

现在单字符词素(lexeme)已经处理，但这并没有涵盖 Lox 的所有运算符。`!`怎么处理？`!`是单个字符，对吧？有时，但如果下一个字符是等号，那么我们应该创建一个`!=`词素。请注意，`!`和`=` 不是两个独立的运算符。您不能在 Lox 中编写`! =`并使其表现得像不等式运算符。这就是为什么我们需要将其扫描为单个词素(lexeme)。同样，`<,>`和`=`后面都可以跟上`=`用于创建其他相等和比较运算符，比如`>= ,<=, == `。

对于所有这些，我们需要看第二个字符。

```java
      case '*': addToken(STAR); break; 
      case '!':
        addToken(match('=') ? BANG_EQUAL : BANG);
        break;
      case '=':
        addToken(match('=') ? EQUAL_EQUAL : EQUAL);
        break;
      case '<':
        addToken(match('=') ? LESS_EQUAL : LESS);
        break;
      case '>':
        addToken(match('=') ? GREATER_EQUAL : GREATER);
        break;

      default:
// lox/Scanner.java, in scanToken()
```

这些case 使用这个新方法：

```java
 private boolean match(char expected) {
    if (isAtEnd()) return false;
    if (source.charAt(current) != expected) return false;

    current++;
    return true;
  }
// lox/Scanner.java, add after scanToken()
```

这就像一个条件`advance()`.如果它是我们正在寻找的，只会消耗当前字符。

使用`match()`，分两个阶段识别这些词素。例如，当我们到达时`!`，我们跳转到它的 switch case。这意味着我们知道词位*以*?`!`然后查看下一个字符以确定是在  `!=`上还是仅仅在 `!`上。

## 4.6 更长lexemes

我们仍然缺少一个运算符：`/`用于除法。该字符需要一些特殊处理，因为注释也以斜杠开头。

```java
        break;
      case '/':
        if (match('/')) {
          // A comment goes until the end of the line.
          while (peek() != '\n' && !isAtEnd()) advance();
        } else {
          addToken(SLASH);
        }
        break;

      default:
// lox/Scanner.java, in scanToken()
```

这类似于其他双字符运算符，只是当找到第二个时`/`，还没有结束token。相反，会一直消耗字符，直到到达行尾。

这是我们处理较长词素的一般策略。在检测到一个开始之后，分流到一些特定于词素的代码，这些代码一直在消耗字符直到它看到结束。

我们还需要另一个辅助函数：

```java
 private char peek() {
    if (isAtEnd()) return '\0';
    return source.charAt(current);
  }
// lox/Scanner.java, add after match() 
```

有点像`advance()`，但不消耗字符。这称为**前瞻lookahead**。由于它只查看当前未使用的字符，因此我们有*一个 lookahead 字符*。通常，这个数字越小，扫描器运行得越快。词法规则决定了需要多少前瞻性。幸运的是，大多数广泛使用的语言只能看到前面的一两个字符。

> 从技术上讲，`match()`也在做前瞻。`advance()`并且`peek()`是基本运算符并将`match()`它们组合在一起。

注释是词素(lexeme)，但它们没有意义，解析器不想处理它们。所以当我们到达注释的末尾时，我们*不会*调用`addToken()`.当我们循环回去开始下一个词素时，`start`它被重置并且注释的词素消失在一阵烟雾中。

当这样做的时候，现在是跳过其他无意义字符的好时机：换行符和空格。

```java
         break;

      case ' ':
      case '\r':
      case '\t':
        // Ignore whitespace.
        break;

      case '\n':
        line++;
        break;

      default:
        Lox.error(line, "Unexpected character.");
// lox/Scanner.java, in scanToken()
```

当遇到空格时，简单地回到扫描循环的开头。在空白字符*之后*开始一个新的词素。对于换行，做同样的事情，但我们也增加行计数器。（这就是为什么我们过去常常`peek()`找到注释结尾的换行符而不是`match()`。我们希望换行符将我们带到这里，以便我们可以更新`line`。）

我们的扫描器变得越来越智能。它可以处理相当自由格式的代码，例如：

```
// this is a comment
(( )){} // grouping stuff
!*+-/=<> <= == // operators
```

### 4.6.1  字符串字面量

现在我们已经习惯了更长的词素，我们准备好处理文字了。将首先处理字符串，因为它们总是以特定字符开头`"`。

```java
        break;

      case '"': string(); break;

      default:
// lox/Scanner.java, in scanToken()
```

它调用：

```java
   private void string() {
    while (peek() != '"' && !isAtEnd()) {
      if (peek() == '\n') line++;
      advance();
    }

    if (isAtEnd()) {
      Lox.error(line, "Unterminated string.");
      return;
    }

    // The closing ".
    advance();

    // Trim the surrounding quotes.
    String value = source.substring(start + 1, current - 1);
    addToken(STRING, value);
  }
// lox/Scanner.java, add after scanToken()  
```

与注释一样，消耗字符，直到我们碰到`"`结束字符串。我们还优雅地处理了在字符串关闭之前用完输入并报告错误的情况。

没有特别的原因，Lox 支持多行字符串。这有利也有弊，但禁止它们比允许它们复杂一点，所以我保留了它们。这确实意味着我们还需要`line`在字符串中遇到换行符时进行更新。

最后一点有趣的是，当我们创建token时，还会生成实际的字符串*值*，稍后将由解释器使用。在这里，该转换只需要 `substring()`去除周围的引号。如果 Lox 支持转义序列，如`\n`，我们将在此处取消转义。

### 4.6.2 数字字面量

Lox 中的所有数字在运行时都是浮点数，但支持整数和小数。数字字面量是一系列数字，可选地后跟一个`.`和一个或多个尾随数字。

```c
1234 
12.34
```

> 由于我们只寻找一个数字来开始一个数字，这意味着`-123`不是数字字面量。相反，`-123`, 是一个应用于数字字面量的*表达式*。在实践中，结果是相同的，尽管如果我们要添加对数字的方法调用，它有一个有趣的边缘情况。考虑：`-``123`
> 
> ```
> print -123.abs();
> ```

> 之所以打印`-123`，是因为否定的优先级低于方法调用。我们可以通过使`-`数字的一部分成为文字来解决这个问题。但请考虑：
> 
> ```
> var n = 123;
> print -n.abs();
> ```

> 这仍然会产生`-123`，所以现在语言似乎不一致。不管你做什么，有些例子最终会变得很奇怪。

我们不允许前导或尾随小数点，因此这些都是无效的：

```c
.1234
1234.
```

可以很容易地支持前者，但为了简单起见，我将其省略了。如果我们想要允许像 . 这样的数字上的方法，后者会变得很奇怪`123.sqrt()`。

为了识别数字词素的开头，我们寻找任何数字。为每个十进制数字添加大小写有点乏味，因此我们将其填充为默认大小写。

```java
  default:
        if (isDigit(c)) {
          number();
        } else {
          Lox.error(line, "Unexpected character.");
        }
        break;
// lox/Scanner.java, in scanToken(), replace 1 line
```

这依赖于这个辅助函数：

```java
  private boolean isDigit(char c) {
    return c >= '0' && c <= '9';
  } 
// lox/Scanner.java, add after peek()
```

> Java 标准库提供了[`Character.isDigit()`](http://docs.oracle.com/javase/7/docs/api/java/lang/Character.html#isDigit(char))，这看起来很合适。然而，该方法允许诸如 Devanagari 数字、全角数字和其他我们不想要的有趣内容。

一旦知道在一个数字中，就会分支到一个单独的方法来使用其余的文字，就像我们处理字符串一样。

```java
  private void number() {
    while (isDigit(peek())) advance();

    // Look for a fractional part.
    if (peek() == '.' && isDigit(peekNext())) {
      // Consume the "."
      advance();

      while (isDigit(peek())) advance();
    }

    addToken(NUMBER,
        Double.parseDouble(source.substring(start, current)));
  }
// lox/Scanner.java, add after scanToken()
```

对于文字的整数部分，消耗了尽可能多的数字。然后我们寻找小数部分，即小数点 (`.`) 后跟至少一位数字。如果我们确实有小数部分，同样，会消耗尽可能多的数字。

越过小数点需要第二个字符 lookahead，因为我们不想消耗`.`直到我们确定它*后面*有一个数字。所以我们添加：

```java
  private char peekNext() {
    if (current + 1 >= source.length()) return '\0';
    return source.charAt(current + 1);
  } 
// lox/Scanner.java, add after peek()
```

> 我本可以`peek()`为要查看的字符数设置为一个参数，而不是定义两个函数，但这将允许*任意*远的前瞻。提供这两个函数可以让代码的读者更清楚我们的扫描器最多向前看两个字符。

最后，将词素转换成它的数值。我们的解释器使用 Java 的`Double`类型来表示数字，因此我们生成该类型的值。我们正在使用 Java 自己的解析方法将词素转换为真正的 Java 双精度数。我们可以自己实现它，但是老实说，除非你想为即将到来的编程面试补习，否则不值得你花时间。

其余的文字是布尔值和`nil`，但我们将它们作为关键字来处理，下边轮到了...

## 4.7 保留关键字和标识符

扫描器差不多完成了。唯一剩下的要实现的词法部分是标识符和它们的近亲，保留字。您可能认为可以 匹配关键字 比如`or`可以像处理多字符运算符比如如`<=`一样.

```java
 case 'o':
  if (match('r')) {
    addToken(OR);
  }
  break;
```

考虑一下如果用户命名一个变量会发生什么`orchid`。扫描器会看到前两个字母`or`，并立即发出`or`关键字token。这让我们得到了一个重要的原则，称为**Maximal munch 最大咀嚼**。当两个词法规则都可以匹配扫描器正在查看的一段代码时，*匹配最多字符的那个将获胜*。

该规则规定，如果我们可以`orchid`作为标识符和`or`关键字进行匹配，则前者获胜。这也是我们之前默认的原因，即`<=`应该作为单个`<=`标记进行扫描，而不是`<`后跟`=`.

> 考虑这段令人讨厌的 C 代码：
> 
> ---a;
> 
> 有效吗？这取决于扫描器如何拆分词位。如果扫描仪看到它是这样的怎么办：
> 
> ```
> - --a;
> ```

> 然后就可以解析了。但这需要扫描器了解周围代码的语法结构，这比我们想要的更复杂。相反，最大咀嚼规则说它*总是*像这样扫描：
> 
> ```
> -- -a;
> ```

> 它以这种方式扫描它，即使这样做会导致稍后在解析器中出现语法错误。

Maximal munch 意味着我们无法轻易检测到保留字，直到我们到达可能是标识符的结尾。毕竟，保留字是一种标识符，它只是一种被语言声明为自己使用的标识符。这就是术语**保留字**的来源。

因此，我们首先假设任何以字母或下划线开头的词素都是标识符

```java
      default:
        if (isDigit(c)) {
          number();
        } else if (isAlpha(c)) {
          identifier();
        } else {
          Lox.error(line, "Unexpected character.");
        }
// lox/Scanner.java, in scanToken()
```

其余代码都在这里：

```java
  private void identifier() {
    while (isAlphaNumeric(peek())) advance();

    addToken(IDENTIFIER);
  }
// lox/Scanner.java, add after scanToken()
```

我们根据这些辅助函数来定义：

```java
  private boolean isAlpha(char c) {
    return (c >= 'a' && c <= 'z') ||
           (c >= 'A' && c <= 'Z') ||
            c == '_';
  }

  private boolean isAlphaNumeric(char c) {
    return isAlpha(c) || isDigit(c);
  }
// lox/Scanner.java, add after peekNext()
```

这使标识符可以工作。为了处理关键字，我们查看标识符的词素是否是保留字之一。如果是，将使用特定于该关键字的token类型。在映射中定义了一组保留字。

```java
  private static final Map<String, TokenType> keywords;

  static {
    keywords = new HashMap<>();
    keywords.put("and",    AND);
    keywords.put("class",  CLASS);
    keywords.put("else",   ELSE);
    keywords.put("false",  FALSE);
    keywords.put("for",    FOR);
    keywords.put("fun",    FUN);
    keywords.put("if",     IF);
    keywords.put("nil",    NIL);
    keywords.put("or",     OR);
    keywords.put("print",  PRINT);
    keywords.put("return", RETURN);
    keywords.put("super",  SUPER);
    keywords.put("this",   THIS);
    keywords.put("true",   TRUE);
    keywords.put("var",    VAR);
    keywords.put("while",  WHILE);
  }
// lox/Scanner.java, in class Scanner
```

然后，在扫描一个标识符之后，检查它是否与Map中的任何东西相匹配。

```java
while (isAlphaNumeric(peek())) advance();

    String text = source.substring(start, current);
    TokenType type = keywords.get(text);
    if (type == null) type = IDENTIFIER;
    addToken(type);
  }
// lox/Scanner.java, in identifier(), replace 1 line
```

如果是这样，将使用该关键字的token类型。否则，它是一个常规的用户定义标识符。

这样，我们现在就有了一个完整的 Lox 词法扫描器。启动 REPL 并输入一些有效和无效的代码。它会产生您期望的token吗？尝试提出一些有趣的边缘情况，看看它是否能按应有的方式处理它们。

---

## [挑战](http://craftinginterpreters.com/scanning.html#challenges)

1. Python 和 Haskell 的词法语法是*不规则*的。那是什么意思，为什么不是呢？

2. 除了分隔token——区别于`print foo`和`printfoo`之外，空格在大多数语言中用处不大。然而，在一些黑暗的角落，空格*确实*会影响代码在 CoffeeScript、Ruby 和 C 预处理器中的解析方式。它在每种语言中的位置和影响是什么？

3. 与大多数扫描器一样，我们这里的扫描器会丢弃注释和空格，因为解析器不需要它们。你为什么要写一个*不*丢弃那些的扫描器？它有什么用？

4. 为 Lox 的扫描器添加对 C 样式`/* ... */`块注释的支持。确保处理其中的换行符。考虑让它们嵌套。添加对嵌套的支持是否比您预期的要多？为什么？

## [设计说明：隐式分号](http://craftinginterpreters.com/scanning.html#design-note)

今天的程序员被语言的选择宠坏了，并且对语法变得挑剔。他们希望语言看起来干净和现代。几乎每一种新语言（以及一些古老的语言，如 BASIC 从未有过的）都被刮掉的一点句法地衣是`;`作为显式语句终止符。

相反，他们将换行符视为语句终止符，这样做是有意义的。“有意义的地方”部分是具有挑战性的部分。虽然*大多数*语句都在自己的行中，但有时您需要将单个语句分散在几行中。那些混合的换行符不应被视为终止符。

大多数应该忽略换行符的明显情况很容易检测到，但也有一些令人讨厌的情况：

- 下一行的返回值：
  
  ```c
  if (condition) return
  "value"
  ```
  
  “value”是被返回的值，还是我们有一个`return`没有值的语句后跟一个包含字符串文字的表达式语句？

- 下一行带括号的表达式：
  
  ```c
  func
  (parenthesized)
  ```
  
  这是对 的调用`func(parenthesized)`，还是两个表达式语句，一个用于`func`括号内的表达式，一个用于括号内的表达式？

- 在下一行有一个`-`：
  
  ```c
  first
  -second
  ```
  
  这是一个`first - second`中缀减法，还是两个表达式语句，一个用于，一个用于`first`求反`second`？

在所有这些中，无论是否将换行符视为分隔符都会产生有效代码，但可能不是用户想要的代码。在各种语言中，用于决定哪些换行符是分隔符的规则种类繁多，令人不安。这里有一对：

- [Lua](https://www.lua.org/pil/1.1.html)完全忽略换行符，但仔细控制其语法，以便在大多数情况下根本不需要语句之间的分隔符。这是完全合法的：
  
  ```
  a = 1 b = 2
  ```
  
  Lua`return`通过要求`return`语句是块中的最后一条语句来避免这个问题。`return`如果关键字之前有值`end`，则它*必须*是`return`。对于其他两种情况，它们允许显式`;`并期望用户使用它。实际上，这几乎永远不会发生，因为带括号的或一元否定表达式语句没有意义。

- [Go](https://golang.org/ref/spec#Semicolons)在扫描器中处理换行符。如果换行符出现在已知可能结束语句的少数标记类型之一之后，则换行符将被视为分号。否则将被忽略。Go 团队提供了一个规范的代码格式化程序[gofmt](https://golang.org/cmd/gofmt/)，并且生态系统对它的使用非常热衷，这确保了惯用风格的代码可以很好地适应这个简单的规则。

- [Python](https://docs.python.org/3.5/reference/lexical_analysis.html#implicit-line-joining)将所有换行符都视为重要的，除非在一行的末尾使用显式反斜杠将其继续到下一行。但是，忽略一对方括号（ 、 或 ）`()`内`[]`任何位置的换行符。`{}`惯用风格强烈倾向于后者。
  
  这条规则适用于 Python，因为它是一种高度面向语句的语言。特别是，Python 的语法确保语句永远不会出现在表达式中。C 做同样的事情，但许多其他具有“lambda”或函数文字语法的语言却没有。
  
  JavaScript 中的一个示例：
  
  ```js
  console.log(function() {
    statement();
  });
  ```
  
  这里，`console.log()`*表达式*包含一个函数字面量，而函数字面量又包含*语句*`statement();`。
  
  如果您可以返回*到*换行符应该变得有意义但仍嵌套在括号内的语句，Python 将需要一组不同的规则来隐式连接行。

现在你知道为什么 Python`lambda`只允许一个表达式主体了。

- JavaScript 的“[自动分号插入](https://www.ecma-international.org/ecma-262/5.1/#sec-7.9)”规则是真正的奇葩。其他语言假设大多数换行符*是*有意义的，而在多行语句中只有少数应该被忽略，而 JS 假设相反。*除非*遇到解析错误，否则它将所有换行符视为无意义的空格。如果是这样，它会返回并尝试将之前的换行符转换为分号以获得语法上有效的内容。
  
  *如果我详细介绍它是如何工作*的，那么这个设计说明将变成设计诽谤，更不用说 JavaScript 的“解决方案”是一个坏主意的所有各种方式。一团糟。JavaScript 是我所知道的唯一一种语言，许多风格指南要求在每个语句后明确分号，即使该语言理论上允许您省略它们。

如果您正在设计一种新语言，您几乎肯定*应该*避免使用显式语句终止符。程序员和其他人一样都是追求时尚的生物，分号和全大写关键字一样已经过时了。只需确保您选择了一组对您的语言的特定语法和习语有意义的规则。并且不要做 JavaScript 做过的事情。