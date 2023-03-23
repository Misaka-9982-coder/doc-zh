# 类

> 如果一个人没有完全了解它的本质，那么他就没有权利去爱或恨它。伟大的爱源于对心爱对象的深入了解，如果你对它知之甚少，你将只能爱它一点点或根本不爱它。
> 
> -- Leonardo da Vinci

我们已经读了十一章，您机器上的解释器几乎是一种完整的脚本语言。它可以使用一些内置的数据结构，如 list 和 map，它当然需要一个用于文件 I/O、用户输入等的核心库。但语言本身就足够了。我们有一些与 BASIC、Tcl、Scheme（去掉宏）以及早期版本的 Python 和 Lua 相同的过程语言。

如果这是 80 年代，我们就会到此为止。但是今天，许多流行的语言都支持“面向对象编程”。将其添加到 Lox 将为用户提供一组熟悉的工具来编写更大的程序。即使您个人不喜欢OOP，本章和[下一章](http://craftinginterpreters.com/inheritance.html)也会帮助您了解其他人如何设计和构建对象系统。

>  但是，如果您*真的*讨厌class ，则可以跳过这两章。它们与本书的其余部分相当孤立。就我个人而言，我发现多了解一些我不喜欢的事情是件好事。事物在远处看起来很简单，但当我走近时，细节就会浮现，我会获得更细微的视角。

## 12.1 OOP 和类

实现面向对象编程的三大途径：类class 、[原型](http://gameprogrammingpatterns.com/prototype.html)和[多方法](https://en.wikipedia.org/wiki/Multiple_dispatch)。class 排在第一位，是最受欢迎的风格。随着 JavaScript（以及较小程度的[Lua](https://www.lua.org/pil/13.4.1.html)）的兴起，原型比以前更广为人知。[我稍后](http://craftinginterpreters.com/classes.html#design-note)会详细讨论这些。对于 Lox，我们采用的是经典方法。

> 多方法是您最不可能熟悉的方法。我很想多谈谈它们——我曾经围绕它们设计了[一种业余语言](http://magpie-lang.org/)，它们*非常棒*——但我只能写这么多页。如果你想了解更多，看看[CLOS](https://en.wikipedia.org/wiki/Common_Lisp_Object_System)（Common Lisp 中的对象系统）、[Dylan](https://opendylan.org/)、[Julia](https://julialang.org/)或[Raku](https://docs.raku.org/language/functions#Multi-dispatch)。

由于您已经和我一起编写了大约一千行 Java 代码，所以我假设您不需要详细介绍面向对象。主要目标是将数据与作用于它的代码捆绑在一起。用户通过声明一个*类*来做到这一点：

1. 公开*构造函数*以创建和初始化类的新*实例*

2. 提供一种在实例上存储和访问*字段的方法*

3. 定义一组由类的所有实例共享的*方法，这些方法对每个实例的状态进行操作。*

这几乎是最小的。大多数面向对象的语言，一直追溯到 Simula，也进行继承以跨类重用行为。我们将在[下一章](http://craftinginterpreters.com/inheritance.html)中添加它。即使把它踢出去，我们还有很多事情要做。这是一个重要的章节，在我们拥有上述所有内容之前，一切都不会完全融合在一起，所以集中你的耐力。

![](./assets/2f4760b7aed09b025d24f312b2e9dbd2ef6bec9f.png)

> 这就像生活的循环，*没有埃尔顿约翰*爵士。

## 12.2 类声明

像之前做的一样，我们将从语法开始。语句引入了一个`class`新名称，因此它存在于`declaration`语法规则中。

```
declaration    → classDecl
               | funDecl
               | varDecl
               | statement ;

classDecl      → "class" IDENTIFIER "{" function* "}" ;
```

新`classDecl`规则依赖于我们[之前](http://craftinginterpreters.com/functions.html#function-declarations)`function`定义的规则。刷新你的记忆：

```
function       → IDENTIFIER "(" parameters? ")" block ;
parameters     → IDENTIFIER ( "," IDENTIFIER )* ;
```

在英语中，类声明是`class`关键字，后跟类名，然后是花括号主体。该主体内部是一个方法声明列表。与函数声明不同，方法没有前导`fun`关键字。每个方法都是一个名称、参数列表和主体。这是一个例子：

> 注意，我尝试说明方法和函数不同

```js
class Breakfast {
  cook() {
    print "Eggs a-fryin'!";
  }

  serve(who) {
    print "Enjoy your breakfast, " + who + ".";
  }
}
```

与大多数动态类型语言一样，字段未明确列在类声明中。实例是松散的数据包，您可以使用普通命令式代码自由地向它们添加您认为合适的字段。

在我们的 AST 生成器中，`classDecl`语法规则有自己的语句节点。

```java
      "Block      : List<Stmt> statements",
      "Class      : Token name, List<Stmt.Function> methods",
      "Expression : Expr expression",
// tool/GenerateAst.java, in main()
```

> 为新节点生成的代码在[附录 II](http://craftinginterpreters.com/appendix-ii.html#class-statement)中。

它将类的名称和方法存储在其主体中。方法由用于函数声明 AST 节点的现有 Stmt.Function 类表示。这为我们提供了方法所需的所有状态：名称、参数列表和主体。

类可以出现在任何允许命名声明的地方，由前导`class`关键字触发。

```java
    try {
      if (match(CLASS)) return classDeclaration();
      if (match(FUN)) return function("function");
// lox/Parser.java, in declaration()
```

这调用：

```java
  private Stmt classDeclaration() {
    Token name = consume(IDENTIFIER, "Expect class name.");
    consume(LEFT_BRACE, "Expect '{' before class body.");

    List<Stmt.Function> methods = new ArrayList<>();
    while (!check(RIGHT_BRACE) && !isAtEnd()) {
      methods.add(function("method"));
    }

    consume(RIGHT_BRACE, "Expect '}' after class body.");

    return new Stmt.Class(name, methods);
  }
// lox/Parser.java, add after declaration() 
```

与大多数其他解析方法相比，它有更多内容，但它大致遵循语法。已经消耗了`class`关键字，所以接下来寻找预期的类名，然后是左花括号。一旦进入主体，将继续解析方法声明，直到遇到右大括号。每个方法声明都通过对 的调用进行解析`function()`，在[介绍函数的章节中](http://craftinginterpreters.com/functions.html)定义了它。

就像在解析器中的任何开放式循环中所做的那样，还检查是否到达文件末尾。这不会发生在正确的代码中，因为类的末尾应该有一个右大括号，但如果用户有语法错误而忘记正确结束类主体，它可以确保解析器不会陷入无限循环。

将名称和方法列表包装到 Stmt.Class 节点中，然后就完成了。以前，我们会直接跳转到解释器，但现在我们需要先通过解析器遍历节点。

```java
 @Override
  public Void visitClassStmt(Stmt.Class stmt) {
    declare(stmt.name);
    define(stmt.name);
    return null;
  }
// lox/Resolver.java, add after visitBlockStmt() 
```

我们还不用担心解析方法本身，所以现在我们需要做的就是使用类名声明类。将类声明为局部变量并不常见，但 Lox 允许这样做，因此我们需要正确处理。

现在我们解释类声明。

```java
  @Override
  public Void visitClassStmt(Stmt.Class stmt) {
    environment.define(stmt.name.lexeme, null);
    LoxClass klass = new LoxClass(stmt.name.lexeme);
    environment.assign(stmt.name, klass);
    return null;
  }
// lox/Interpreter.java, add after visitBlockStmt()
```

这看起来类似于我们执行函数声明的方式。我们在当前环境中声明类的名称。然后我们将类*语法节点*变成 LoxClass，类的*运行时*表示。我们回过头来将类对象存储在我们之前声明的变量中。该两阶段变量绑定过程允许在其自己的方法中引用该类。

我们将在整章中对其进行完善，但 LoxClass 的初稿如下所示：

```java
package com.craftinginterpreters.lox;

import java.util.List;
import java.util.Map;

class LoxClass {
  final String name;

  LoxClass(String name) {
    this.name = name;
  }

  @Override
  public String toString() {
    return name;
  }
}
// lox/LoxClass.java, create new file
```

从字面上看是一个名字的包装。我们甚至还没有存储方法。不是很有用，但它确实有一个`toString()`方法，因此我们可以编写一个简单的脚本并测试类对象是否真的被解析和执行。

```java
class DevonshireCream {
  serveOn() {
    return "Scones";
  }
}

print DevonshireCream; // Prints "DevonshireCream".
```

## 12.3 创建实例

我们有了class ，但他们还没有做任何事情。Lox 没有可以直接在类本身上调用的“静态”方法，因此如果没有实例，类就毫无用处。因此实例是下一步。

虽然某些语法和语义在 OOP 语言中是相当标准的，但创建新实例的方式却不是。Ruby 遵循 Smalltalk，通过调用类对象本身的方法来创建实例，这是一种优雅的递归方法。有些，如 C++ 和 Java，有一个`new`关键字专门用于生成一个新对象。Python 让您像调用函数一样“调用”类本身。（JavaScript，曾经很奇怪，有点两者兼而有之。）

> 在 Smalltalk 中，甚至*类*都是通过调用现有对象（通常是所需的超类）上的方法来创建的。这有点像乌龟一路倒下的事情。它最终会在一些神奇的类（如 Object 和 Metaclass）上触底，运行时会让人联想到它们是*ex nihilo*。

我对 Lox 采取了最小的方法。我们已经有了类对象，也有了函数调用，所以我们将在类对象上使用调用表达式来创建新实例。就好像一个类是一个生成自身实例的工厂函数。这对我来说感觉很优雅，也让我们无需引入像`new`.因此，我们可以跳过前端直接进入运行时。

现在，如果你试试这个：

```js
class Bagel {}
Bagel();
```

您收到运行时错误。`visitCallExpr()`检查被调用对象是否实现`LoxCallable`接口并报告错误，因为 LoxClass 没有。还*没有*，就是这样。

```java
import java.util.Map;

class LoxClass implements LoxCallable {
  final String name;
// lox/LoxClass.java, replace 1 line
```

实现该接口需要两个方法。

```java
@Override
  public Object call(Interpreter interpreter,
                     List<Object> arguments) {
    LoxInstance instance = new LoxInstance(this);
    return instance;
  }

  @Override
  public int arity() {
    return 0;
  }
// lox/LoxClass.java, add after toString() 
```

有趣的是`call()`。当您“调用”一个类时，它会为被调用的类实例化一个新的 LoxInstance 并返回它。该`arity()`方法是解释器如何验证您将正确数量的参数传递给可调用对象的方式。现在，我们会说你不能传递任何参数。当我们谈到用户定义的构造函数时，我们将重新讨论它。

这将我们引向 LoxInstance，它是 Lox 类实例的运行时表示。同样，我们的第一个实现从小处着手。

```java
package com.craftinginterpreters.lox;

import java.util.HashMap;
import java.util.Map;

class LoxInstance {
  private LoxClass klass;

  LoxInstance(LoxClass klass) {
    this.klass = klass;
  }

  @Override
  public String toString() {
    return klass.name + " instance";
  }
}
// lox/LoxInstance.java, create new file
```

与 LoxClass 一样，它非常简单，但我们才刚刚开始。如果您想尝试一下，可以运行以下脚本：

```js
class Bagel {}
var bagel = Bagel();
print bagel; // Prints "Bagel instance".
```

这个程序做的不多，但它开始做*一些事情了*。

## 12.4 实例属性

我们有实例，所以应该让它们有用。我们正处在岔路口。我们可以先添加行为——方法——或者我们可以从状态开始——属性。我们将采用后者，因为正如我们将看到的那样，两者以一种有趣的方式纠缠在一起，如果我们先让属性起作用，就会更容易理解它们。

Lox 在处理状态方面遵循 JavaScript 和 Python。每个实例都是命名值的开放集合。实例类的方法可以访问和修改属性，但外部代码也可以。使用`.`语法访问属性。

> 允许类外部的代码直接修改对象的字段违背了类*封装*状态的面向对象的信条。有些语言采取更有原则的立场。在 Smalltalk 中，使用简单的标识符访问字段—本质上，变量仅在类方法的范围内。Ruby 使用`@`后跟名称来访问对象中的字段。该语法仅在方法内部有意义，并且始终访问当前对象的状态。
> 
> 不管是好是坏，Lox 对其面向对象编程的信仰并不那么虔诚。

```js
someObject.someProperty
```

后跟`.`一个标识符的表达式从表达式求值的对象中读取具有该名称的属性。该点与函数调用表达式中的括号具有相同的优先级，因此我们通过将现有`call`规则替换为以下内容来将其放入语法中：

```
call           → primary ( "(" arguments? ")" | "." IDENTIFIER )* ;
```

在主要表达式之后，我们允许一系列括号调用和点属性访问的任意组合。“属性访问”很拗口，所以从现在开始，我们称这些为“get 表达式”。

### 12.4.1 Get表达式

语法树节点为:

```java
      "Call     : Expr callee, Token paren, List<Expr> arguments",
      "Get      : Expr object, Token name",
      "Grouping : Expr expression",
// tool/GenerateAst.java, in main()
```

为新节点生成的代码在[附录 II](http://craftinginterpreters.com/appendix-ii.html#get-expression)中。

按照语法，新的解析代码进入现有的`call()`方法

```java
    while (true) { 
      if (match(LEFT_PAREN)) {
        expr = finishCall(expr);
      } else if (match(DOT)) {
        Token name = consume(IDENTIFIER,
            "Expect property name after '.'.");
        expr = new Expr.Get(expr, name);
      } else {
        break;
      }
    }
// lox/Parser.java, in call()
```

外部`while`循环对应于语法规则中的`*`。我们沿着token构建一个调用链，并在找到括号和点时获取，如下所示：

![](./assets/4eb893ce64b55b66f25af5eb7bf6cf5e85daf5ab.png)

新 Expr.Get 节点的实例提供给解析器。

```java
 @Override
  public Void visitGetExpr(Expr.Get expr) {
    resolve(expr.object);
    return null;
  }
// lox/Resolver.java, add after visitCallExpr() 
```

好的，不多说了。由于属性是动态查找的，因此不会解析它们。在解析期间，我们仅递归到点左侧的表达式。实际的属性访问发生在解释器中。

> 您可以从字面上看到 Lox 中的属性分派是动态的，因为我们在静态解析过程中不处理属性名称。

```java
  @Override
  public Object visitGetExpr(Expr.Get expr) {
    Object object = evaluate(expr.object);
    if (object instanceof LoxInstance) {
      return ((LoxInstance) object).get(expr.name);
    }

    throw new RuntimeError(expr.name,
        "Only instances have properties.");
  }
// lox/Interpreter.java, add after visitCallExpr()
```

首先，我们评估其属性被访问的表达式。在 Lox 中，只有类的实例具有属性。如果对象是数字等其他类型，则对其调用 getter 是运行时错误。

如果对象是 LoxInstance，那么我们要求它查找属性。一定是时候给 LoxInstance 一些实际状态了。一个Map就可以了。

```java
  private LoxClass klass;
  private final Map<String, Object> fields = new HashMap<>();

  LoxInstance(LoxClass klass) {
// lox/LoxInstance.java, in class LoxInstance 
```

Map中的每个键都是一个属性名称，对应的值是属性的值。要查找实例的属性：

```java
  Object get(Token name) {
    if (fields.containsKey(name.lexeme)) {
      return fields.get(name.lexeme);
    }

    throw new RuntimeError(name, 
        "Undefined property '" + name.lexeme + "'.");
  }
// lox/LoxInstance.java, add after LoxInstance() 
```

> 对每个字段访问进行哈希表查找对于许多语言实现来说已经足够快，但并不理想。用于 JavaScript 等语言的高性能 VM 使用复杂的优化（如“[隐藏类](http://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html)”）来避免这种开销。
> 
> 矛盾的是，为使动态语言更快而发明的许多优化都基于这样的观察——即使在那些语言中——大多数代码在其使用的对象类型及其字段方面都是相当静态的。

我们需要处理的一个有趣的边缘情况是如果实例没有*具有*给定名称的属性会发生什么。我们可以默默地返回一些虚拟值，如`nil`，但我对 JavaScript 等语言的经验是，这种行为比它做任何有用的事情更经常地掩盖错误。相反，我们会将其设为运行时错误。

所以我们做的第一件事是查看实例是否真的有一个具有给定名称的字段。只有这样我们才返回它。否则，我们会引发错误。

请注意我是如何从谈论“属性”切换到谈论“字段”的。两者之间存在细微差别。字段是**直接**存储在实例中的命名状态。属性是get 表达式可能返回的命名的*东西*。每个字段都是一个属性，但正如我们稍后将看到的，并非每个属性都是一个字段。

> 呵呵，预示。幽灵般的！

理论上，现在可以读取对象的属性。但是由于无法将任何状态实际填充到实例中，因此没有可访问的字段。在测试读取之前，必须支持写入。

### 12.4.2 Set表达式

Setter 使用与 getter 相同的语法，只是它们出现在赋值的左侧。

```js
someObject.someProperty = value;
```

在语法领域，我们扩展赋值规则以允许在左侧使用点标识符。

```
assignment     → ( call "." )? IDENTIFIER "=" assignment
               | logic_or ;
```

与 getter 不同，setter 不能链式调用。但是，在`call`引用最后一个点之前允许任何高优先级表达式，包括任意数量的*getter*，如：

![](./assets/081c8ed28f8734759bd2ba3b93796a609eaddaac.png)

注意这里只有*最后*一部分`.meat`是*setter*。`.omelette`和`.filling`部分都是get*表达式*。

正如我们有两个单独的 AST 节点用于变量访问和变量赋值一样，我们需要第二个 setter 节点来补充我们的 getter 节点。

```java
      "Logical  : Expr left, Token operator, Expr right",
      "Set      : Expr object, Token name, Expr value",
      "Unary    : Token operator, Expr right",
// tool/GenerateAst.java, in main()
```

> 为新节点生成的代码在[附录 II](http://craftinginterpreters.com/appendix-ii.html#set-expression)中。

不知道你是否记得，我们在解析器中处理赋值的方式有点滑稽。在遇到`=` 之前，解析器都无法确定这一系列token是否是一个赋值表达式的左值。  现在`call`在赋值语法规则左侧，可以扩展到任意大的表达式，最后的`=`可能与正在解析赋值的点之间有很多token。

相反，我们使用的技巧是将左侧解析为普通表达式。然后，当偶然发现它后面的等号时，我们采用已经解析的表达式并将其转换为正确的语法树节点以进行赋值。

我们向该转换添加另一个子句，以处理将左侧的 Expr.Get 表达式转换为相应的 Expr.Set。

```java
        return new Expr.Assign(name, value);
      } else if (expr instanceof Expr.Get) {
        Expr.Get get = (Expr.Get)expr;
        return new Expr.Set(get.object, get.name, value);
      }
// lox/Parser.java, in assignment()
```

那就是解析我们的语法。我们将该节点推送到解析器中。

```java
  @Override
  public Void visitSetExpr(Expr.Set expr) {
    resolve(expr.value);
    resolve(expr.object);
    return null;
  }
// lox/Resolver.java, add after visitLogicalExpr() 
```

同样，与 Expr.Get 一样，属性本身是动态求值的，因此没有什么需要解析的。我们需要做的就是递归到 Expr.Set 的两个子表达式中，即要设置其属性的对象及其设置的值。

这将我们引向解释器。

```java
  @Override
  public Object visitSetExpr(Expr.Set expr) {
    Object object = evaluate(expr.object);

    if (!(object instanceof LoxInstance)) { 
      throw new RuntimeError(expr.name,
                             "Only instances have fields.");
    }

    Object value = evaluate(expr.value);
    ((LoxInstance)object).set(expr.name, value);
    return value;
  }
// lox/Interpreter.java, add after visitLogicalExpr() 
```

评估正在设置其属性的对象并检查它是否是 LoxInstance。如果不是，那就是运行时错误。否则，评估设置的值并将其存储在实例上。这依赖于 LoxInstance 中的一个新方法。

> 这是另一个语义边缘案例。共有三种不同的操作：
> 
> 1. 评估对象。
> 
> 2. 如果它不是类的实例，则引发运行时错误。
> 
> 3. 评估值。
>    
>    这些执行的顺序可能是用户可见的，这意味着我们需要仔细指定它并确保我们的实现以相同的顺序执行这些操作。

```java
 void set(Token name, Object value) {
    fields.put(name.lexeme, value);
  }
// lox/LoxInstance.java, add after get()
```

这里没有真正的魔法。我们将值直接填充到字段所在的 Java Map中。由于 Lox 允许在实例上自由创建新字段，因此无需查看key是否已经存在。

## 12.5 类方法

您可以创建类的实例并将数据填充到其中，但类本身实际上并没有*做*任何事情。实例只是Map，所有实例都或多或少相同。为了让它们感觉像类的实例，我们需要行为——方法。

解析器已经解析了方法声明，所以我们已经做得很好了。我们也不需要为方法*调用*添加任何新的解析器支持。我们已经有了`.`(getters) 和`()`(function calls)。“方法调用 (method call)”只是将它们链接在一起。

![](./assets/75a6cd73d981ee41dac1c7626c6bb4df0809eb77.png)

这就提出了一个有趣的问题。当这两个表达式分开时会发生什么？假设`method`在这个例子中是类上的方法?`object`而不是实例上的字段，下面这段代码应该做什么？

```js
var m = object.method;
m(argument);
```

该程序“查找”该方法(method)并将结果（无论结果是什么）存储在一个变量中，然后稍后调用该对象。这是允许的吗？你能像对待实例上的函数一样对待方法吗？

另一个方向呢？

```js
class Box {}

fun notMethod(argument) {
  print "called function with " + argument;
}

var box = Box();
box.function = notMethod;
box.function("argument");
```

该程序创建一个实例，然后在其上的一个字段中存储一个函数。然后它使用与方法调用相同的语法调用该函数。那样有用吗？

不同的语言对这些问题有不同的答案。人们可以写一篇关于它的论文。对于 Lox，我们会说这两个问题的答案都是肯定的，它确实有效。我们有几个理由来证明这一点。对于第二个示例——调用存储在字段中的函数——我们希望支持它，因为first-class函数很有用，将它们存储在字段中是一件非常正常的事情。

第一个例子比较晦涩。一个动机是用户通常希望能够在不改变程序含义的情况下将子表达式提升到局部变量中。你可以这样：

```js
breakfast(omelette.filledWith(cheese), sausage);
```

把它变成这样：

```js
var eggs = omelette.filledWith(cheese);
breakfast(eggs, sausage);
```

它做同样的事情。同样，由于方法调用中的`.`和 `()`是两个独立的表达式，因此您似乎应该能够将*查找*部分提升到一个变量中，然后再调用它。我们需要仔细考虑查找方法时得到的结果是什么，以及它的行为方式，即使是在奇怪的情况下，例如：

> 对此的一个激励性用途是回调。通常，您希望传递一个回调，其主体只是调用某个对象上的一个方法。能够查找方法并直接传递它可以省去手动声明函数来包装它的繁琐工作。比较一下：
> 
> ```js
> fun callback(a, b, c) {
>   object.method(a, b, c);
> }
> 
> takeCallback(callback);
> ```

> 有了这个：
> 
> ```js
> takeCallback(object.method);
> ```

```js
class Person {
  sayName() {
    print this.name;
  }
}

var jane = Person();
jane.name = "Jane";

var method = jane.sayName;
method(); // ?
```

如果您获取某个实例上某个方法的句柄并稍后调用它，它会“记住”它从中提取的实例吗？方法内部`this`是否仍然引用那个原始对象？

这是一个更病态的例子来扰乱你的大脑：

```js
class Person {
  sayName() {
    print this.name;
  }
}

var jane = Person();
jane.name = "Jane";

var bill = Person();
bill.name = "Bill";

bill.sayName = jane.sayName;
bill.sayName(); // ?
```

最后一行打印“Bill”是因为这是我们*调用*该方法的实例，还是打印“Jane”是因为它是我们首先获取该方法的实例？

Lua 和 JavaScript 中的等效代码将打印“Bill”。这些语言并没有真正的“方法”概念。一切都是字段内的函数（functions-in-fields），因此并不清楚 `jane`“拥有”`sayName`比` bill`多一些。

但是，Lox 具有真正的类语法，所以我们确实知道哪些可调用的东西是方法，哪些是函数。因此，与 Python、C# 和其他语言一样，当方法首次被抓取时，我们会将方法的`this`“绑定”到原始实例。Python 称这些叫**绑定方法**。

> 我知道，富有想象力的名字，对吧？

实际上，这通常就是您想要的。如果您引用某个对象上的方法以便稍后将其用作回调，您需要记住它所属的实例，即使该回调恰好存储在某个其他对象的字段中。

好吧，有很多语义要加载到您的脑海中。暂时忘记边缘情况。我们会回到那些。现在，让我们开始使用基本的方法调用。我们已经在类体内解析了方法声明，所以下一步是解析它们。

```java
    define(stmt.name);

    for (Stmt.Function method : stmt.methods) {
      FunctionType declaration = FunctionType.METHOD;
      resolveFunction(method, declaration); 
    }

    return null;
// lox/Resolver.java, in visitClassStmt()
```

> 将函数类型存储在局部变量中现在毫无意义，但我们很快就会扩展这段代码，它会更有意义。

我们遍历  class主体中的method 并调用`resolveFunction()`，这是我们已经编写的用于处理函数声明的方法。唯一的区别是传入了一个新的 FunctionType 枚举值。

```java
    NONE,
    FUNCTION,
    METHOD
  }
// lox/Resolver.java, in enum FunctionType, add “,” to previous line
```

当我们解析`this`表达式时，这将很重要。现在，不用担心。有趣的东西在解释器中

```java
    environment.define(stmt.name.lexeme, null);

    Map<String, LoxFunction> methods = new HashMap<>();
    for (Stmt.Function method : stmt.methods) {
      LoxFunction function = new LoxFunction(method, environment);
      methods.put(method.name.lexeme, function);
    }

    LoxClass klass = new LoxClass(stmt.name.lexeme, methods);
    environment.assign(stmt.name, klass);
// lox/Interpreter.java, in visitClassStmt(), replace 1 line
```

当解释一个类声明语句时，将类的语法表示——它的 AST 节点——转化为它的运行时表示。现在，也需要对类中包含的方法执行此操作。每个方法声明都会变成一个 LoxFunction 对象。

我们采用所有这些并将它们包装到一个Map中，以方法名称作为键。它存储在 LoxClass 中。

```java
  final String name;
  private final Map<String, LoxFunction> methods;

  LoxClass(String name, Map<String, LoxFunction> methods) {
    this.name = name;
    this.methods = methods;
  }

  @Override
  public String toString() {
// lox/LoxClass.java, in class LoxClass, replace 4 lines  
```

实例是存储状态的地方，而类存储行为。LoxInstance 有它的字段Map，而 LoxClass 有一个方法Map。即使方法由类拥有，它们仍然可以通过该类的实例访问。

```java
   Object get(Token name) {
    if (fields.containsKey(name.lexeme)) {
      return fields.get(name.lexeme);
    }

    LoxFunction method = klass.findMethod(name.lexeme);
    if (method != null) return method;

    throw new RuntimeError(name, 
        "Undefined property '" + name.lexeme + "'.");
// lox/LoxInstance.java, in get()
```

在实例上查找属性时，如果找不到匹配的字段，会在实例的类中查找具有该名称的方法。如果找到，将其返回。这就是“字段”和“属性”之间的区别变得有意思的地方。当访问一个属性时，您可能会得到一个字段——存储在实例上的一些状态——或者您可以访问实例类上定义的方法。

使用此方法查找该方法：

> 首先查找字段意味着字段遮蔽方法，这是一个微妙但重要的语义点。

```java
  LoxFunction findMethod(String name) {
    if (methods.containsKey(name)) {
      return methods.get(name);
    }

    return null;
  }
// lox/LoxClass.java, add after LoxClass()
```

您可能会猜到此方法稍后会变得更有趣。现在，对类的方法表进行简单的Map查找就足以让我们开始了。试一试：

```js
class Bacon {
  eat() {
    print "Crunch crunch crunch!";
  }
}

Bacon().eat(); // Prints "Crunch crunch crunch!".
```

> 如果您更喜欢有嚼劲的培根而不是松脆的培根，我们深表歉意。随意根据自己的喜好调整脚本。

## 12.6 This

我们可以在对象上同时定义行为和状态，但它们还没有绑定在一起。在方法内部，无法访问“当前”对象（调用该方法的实例）的字段，也无法在同一对象上调用其他方法。

要获取该实例，它需要一个名称。Smalltalk、Ruby 和 Swift 使用“self”。Simula、C++、Java 和其他语言使用“this”。Python 按照惯例使用“self”，但从技术上讲，您可以随意称呼它。

> “I”本来是一个不错的选择，但是将“i”用于循环变量早于 OOP 并一直追溯到 Fortran。我们是祖先偶然选择的受害者。

对于 Lox，由于我们通常遵循 Java 风格，所以将使用“this”。在方法体内，`this`表达式计算为调用该方法的实例。或者，更具体地说，由于方法的访问和调用分两步进行，因此它将引用从中*访问*该方法的对象。

这使我们的工作更加困难。如下代码：

```js
class Egotist {
  speak() {
    print this;
  }
}

var method = Egotist().speak;
method();
```

在倒数第二行，我们从类的实例中获取对`speak()`该方法的引用。它返回一个函数，并且该函数需要记住它关联的实例，以便*稍后*在最后一行，它仍然可以在调用该函数时找到它。

我们在方法被访问的那一刻需要`this`，并以某种方式将其附加到函数，以便它在需要的时候一直存在。嗯? 一种存储函数周围的一些额外数据的方法，呃？这听起来很像*闭包*，不是吗？

如果在查找方法时返回的函数周围的环境中定义`this`为一种隐藏变量，那么`this`稍后在主体中使用 将能够找到它。LoxFunction 已经具备控制周围环境的能力，因此我们拥有所需的机制。

通过一个例子来看看它是如何工作的：

```js
class Cake {
  taste() {
    var adjective = "delicious";
    print "The " + this.flavor + " cake is " + adjective + "!";
  }
}

var cake = Cake();
cake.flavor = "German chocolate";
cake.taste(); // Prints "The German chocolate cake is delicious!".
```

当第一次评估类定义时，我们为`taste()`创建一个 LoxFunction。它的闭包是类周围的环境，在本例中是全局环境。所以我们存储在类的方法Map中的 LoxFunction 看起来像这样：

![](./assets/47b096bb40ac998afc1ff60f688100e7aa7b1a41.png)

当我们评估`cake.taste`get 表达式时，我们创建了一个新环境，该环境绑定`this`到访问该方法的对象（此处为`cake`）。然后我们使用与原始代码相同的代码创建一个*新的 LoxFunction，但使用新环境作为其闭包。*

![](./assets/4832399ab25d00dd42e53fc7f00f3197b4dcad10.png)

这是在评估方法名称的 get 表达式时返回的 LoxFunction。当该函数稍后被`()`表达式调用时，我们像往常一样为方法体创建一个环境。

![](./assets/4775bab5f20116f2a89f7afb2458226cc59fc81b.png)

方法主体环境的父级是之前创建的绑定`this`到当前对象的环境。因此，在主体内部的任何使用都`this`成功解析为该实例。

重用环境代码来实现`this`还可以处理方法和函数交互的有趣情况，例如：

```js
class Thing {
  getCallback() {
    fun localFunction() {
      print this;
    }

    return localFunction;
  }
}

var callback = Thing().getCallback();
callback();
```

例如，在 JavaScript 中，从方法内部返回回调是很常见的。该回调可能想要挂起并保留对该方法关联的原始对象（`this`值）的 访问。现有的对闭包和环境链的支持应该可以正确地完成所有这些工作。

让我们开始写代码。第一步是为`this`.

```java
      "Set      : Expr object, Token name, Expr value",
      "This     : Token keyword",
      "Unary    : Token operator, Expr right",
// tool/GenerateAst.java, in main()
```

> 为新节点生成的代码在[附录 II](http://craftinginterpreters.com/appendix-ii.html#this-expression)中。

解析很简单，因为它是我们的词法分析器已经识别为保留字的单个token。

```java
      return new Expr.Literal(previous().literal);
    }

    if (match(THIS)) return new Expr.This(previous());

    if (match(IDENTIFIER)) {
// lox/Parser.java, in primary()
```

当我们到达解析器时，您可以开始了解`this`如何像变量一样工作。

```java
  @Override
  public Void visitThisExpr(Expr.This expr) {
    resolveLocal(expr, expr.keyword);
    return null;
  }

// lox/Resolver.java, add after visitSetExpr()  
```

我们把“this” 作为 普通局部变量名称来解析它。当然，这现在行不通，因为“this”*没有*在任何作用域内声明。让我们在`visitClassStmt()`方法中修正它.

```java
    define(stmt.name);

    beginScope();
    scopes.peek().put("this", true);

    for (Stmt.Function method : stmt.methods) {
// lox/Resolver.java, in visitClassStmt()
```

在我们介入并开始解析方法体之前，我们推送一个新的作用域并在其中定义“this”，就好像它是一个变量一样。然后，当完成后，丢弃周围的作用域。

```java
    }

    endScope();

    return null;
// lox/Resolver.java, in visitClassStmt()
```

现在，无论何时`this`遇到表达式（至少在方法内部），它都会解析为在方法主体块外部的隐式作用域中定义的“局部变量”。

解析器为`this`创建了新的*作用域*，因此解释器需要为其创建相应的*环境*。请记住，**我们始终必须保持解析器的作用域链和解释器的环境链彼此同步**。在运行时，在实例上找到方法后创建环境。将前一行简单地返回方法的 LoxFunction 的代码替换为：

```java
    LoxFunction method = klass.findMethod(name.lexeme);
    if (method != null) return method.bind(this);

    throw new RuntimeError(name, 
        "Undefined property '" + name.lexeme + "'.");
// lox/LoxInstance.java, in get(), replace 1 line
```

注意对 `bind()`的新调用。看起来像这样：

```java
  LoxFunction bind(LoxInstance instance) {
    Environment environment = new Environment(closure);
    environment.define("this", instance);
    return new LoxFunction(declaration, environment);
  }
// lox/LoxFunction.java, add after LoxFunction()
```

没什么可说的。我们在方法的原始闭包内创建了一个新环境。有点像闭包中的闭包。当方法被调用时，它将成为方法主体环境的父级。

我们将“this”声明为该环境中的一个变量，并将其绑定到给定的实例，即从中访问该方法的实例。*Et* voilà，返回的 LoxFunction 现在带有它自己的小持久世界，其中“this”绑定到对象。

剩下的任务是解释这些`this`表达式。类似于解析器，它与解释变量表达式相同。

```java
  @Override
  public Object visitThisExpr(Expr.This expr) {
    return lookUpVariable(expr.keyword, expr);
  }
// lox/Interpreter.java, add after visitSetExpr()
```

继续尝试使用之前的蛋糕示例。使用不到 20 行的代码，解释器可以处理`this`内部方法，即使它可以与嵌套类、方法内部函数、方法句柄等进行交互的所有奇怪方式。

### 12.6.1  this 的无效使用

等一下。如果您尝试在方法之外使用`this`会发生什么？关于：

```js
print this;
```

或者：

```js
fun notAMethod() {
  print this;
}
```

如果您不在方法中，则没有指向`this`的实例。可以给它一些默认值，`nil`或者让它成为一个运行时错误，但用户显然犯了一个错误。他们越早发现并改正错误，他们就会越开心。

解析变量的遍历过程(resolution pass)是静态检测此错误的好地方。它已经检测到`return`函数之外的语句。我们将为 `this`做类似的事情。按照我们现有的 FunctionType 枚举，我们定义了一个新的 ClassType 枚举。

```java
  }

  private enum ClassType {
    NONE,
    CLASS
  }

  private ClassType currentClass = ClassType.NONE;

  void resolve(List<Stmt> statements) {
// lox/Resolver.java, add after enum FunctionType 
```

是的，它可以是一个布尔值。当我们实现继承时，将加入第三个值，因此现在是枚举。我们还添加了一个相应的字段`currentClass`，它的值告诉我们在遍历语法树时当前是否在类声明中。初始值`NONE`，这意味着现在不在类声明中。

当我们开始解析一个类声明时，我们会改变它。

```java
  public Void visitClassStmt(Stmt.Class stmt) {
    ClassType enclosingClass = currentClass;
    currentClass = ClassType.CLASS;

    declare(stmt.name);
// lox/Resolver.java, in visitClassStmt()
```

与 `currentFunction`一样，我们将currentClass字段之前的值存储在局部变量中。这让我们可以借助 JVM 来保存`currentClass`到方法堆栈栈中。这样，如果一个类嵌套在另一个类中，就不会忘记之前的值。

一旦方法解析完成，我们就通过恢复旧值来“弹出”堆栈

```java
    endScope();

    currentClass = enclosingClass;
    return null;
// lox/Resolver.java, in visitClassStmt()
```

当我们解析一个`this`表达式时，如果表达式没有出现在方法体内，`currentClass`字段会为我们提供报告错误所需的数据。

```java
   public Void visitThisExpr(Expr.This expr) {
    if (currentClass == ClassType.NONE) {
      Lox.error(expr.keyword,
          "Can't use 'this' outside of a class.");
      return null;
    }

    resolveLocal(expr, expr.keyword);
// lox/Resolver.java, in visitThisExpr()
```

这应该可以帮助用户`this`正确使用，并且让我们不必在运行时在解释器中处理误用。

## 12.7 构造函数和初始化

现在几乎可以用类做任何事情，当接近本章的结尾时，我们发现自己奇怪地专注于开头。方法和字段将状态和行为封装在一起，为了使对象始终*保持*有效配置。但是我们如何确保一个全新的对象以良好的状态开始呢？

为此，需要构造函数。我发现它们是语言设计中最棘手的部分之一，如果你仔细观察大多数其他语言，你会发现 对象构造周围存在裂缝，设计的接缝不能完美地结合在一起。也许出生的那一刻本质上有些混乱。

> 举几个例子： 在 Java 中，即使 final 字段必须被初始化，仍然可以在它*之前*读取一个。异常——一个巨大而复杂的特性——被添加到 C++ 中主要是作为从构造函数中发出错误的一种方式。

“构造”一个对象实际上是一对操作：

1. 运行时为新实例*分配*所需的内存。在大多数语言中，此操作处于用户代码能够访问的基础级别。
   
   > C++ 的“[placement new](https://en.wikipedia.org/wiki/Placement_syntax)?”是一个罕见的例子，其中分配的内部结构是暴露给程序员的。

2. 然后，调用用户提供的代码块来*初始化*未成形的对象。

后者是我们在听到“构造函数”时倾向于想到的，但在我们到达那个点之前，语言本身通常已经为我们做了一些基础工作。事实上，Lox 解释器在创建新的 LoxInstance 对象时已经涵盖了这一点。

我们现在将完成剩下的部分——用户定义的初始化。对于为类设置新对象的代码块，语言有多种表示法。C++、Java 和 C# 使用名称与类名匹配的方法。Ruby 和 Python 称之为`init()`.后者很好而且很短，所以我们会这样做。

在 LoxClass 的 LoxCallable 实现中，我们添加了几行

```java
                      List<Object> arguments) {
    LoxInstance instance = new LoxInstance(this);
    LoxFunction initializer = findMethod("init");
    if (initializer != null) {
      initializer.bind(instance).call(interpreter, arguments);
    }

    return instance;
// lox/LoxClass.java, in call()
```

当一个类被调用时，在创建 LoxInstance 之后，寻找一个“init”方法。如果找到一个，会像普通方法调用一样立即绑定并调用它。参数列表一起转发。

该参数列表意味着还需要调整类声明其元数的方式。

```java
 public int arity() {
    LoxFunction initializer = findMethod("init");
    if (initializer == null) return 0;
    return initializer.arity();
  }
// lox/LoxClass.java, in arity(), replace 1 line
```

如果有初始化器，则该方法的元数(arity)决定了调用类本身时必须传递的参数数量。不过，为方便起见，我们*不需要*类来定义初始化程序。如果没有初始化器，arity 仍然是零。

基本上就是这样。由于我们在调用之前绑定`init()`方法，因此它可以访问`this`其主体内部。这与传递给类的参数一起，是您根据需要设置新实例所需的全部内容。

### 12.7.1 直接调用 init()

像往常一样，探索这个新的语义领域会激起一些奇怪的生物。考虑：

```js
class Foo {
  init() {
    print this;
  }
}

var foo = Foo();
print foo.init();
```

你能通过直接调用它的`init()`方法来“重新初始化”一个对象吗？如果这样做，它会返回什么？一个合理的答案是`nil`，因为这就是init函数返回值。

然而——我通常不喜欢为了满足实现而妥协——如果我们说`init()`方法总是返回`this`，即使直接调用，也会使 clox 的构造函数实现变得容易得多。为了保持 jlox 与之兼容，我们在 LoxFunction 中添加了一些特殊情况代码。

> 也许“不喜欢”这个说法太过强烈了。实现的约束和资源影响语言的设计是合理的。一天只有这么多小时，如果在这里或那里走捷径可以让您在更短的时间内为用户提供更多功能，这很可能是他们快乐和生产力的净赢。诀窍是弄清楚*哪些*角落不会让您的用户和未来的自己诅咒您的短视。

```java
      return returnValue.value;
    }

    if (isInitializer) return closure.getAt(0, "this");
    return null;
// lox/LoxFunction.java, in call()
```

如果函数是初始化器，我们会覆盖实际的返回值并强制返回`this`。那依赖于一个新的`isInitializer`字段。

```java
  private final Environment closure;

  private final boolean isInitializer;

  LoxFunction(Stmt.Function declaration, Environment closure,
              boolean isInitializer) {
    this.isInitializer = isInitializer;
    this.closure = closure;
    this.declaration = declaration;
// lox/LoxFunction.java, in class LoxFunction, replace 1 line 
```

我们不能简单地查看 LoxFunction 的名称是否为“init”，因为用户可能已经使用该名称定义了一个*函数*。那样的话，就没有`this`可以返回了。为了避免*这种*奇怪的边缘情况，我们将直接把 LoxFunction 是否表示初始化方法保存起来。这意味着需要返回并修复创建 LoxFunctions 的几个地方。

```java
  public Void visitFunctionStmt(Stmt.Function stmt) {
    LoxFunction function = new LoxFunction(stmt, environment,
                                           false);
    environment.define(stmt.name.lexeme, function);
// lox/Interpreter.java, in visitFunctionStmt(), replace 1 line
```

对于实际的函数声明，`isInitializer`始终为 false。对于方法，我们检查名称。

```java
    for (Stmt.Function method : stmt.methods) {
      LoxFunction function = new LoxFunction(method, environment,
          method.name.lexeme.equals("init"));
      methods.put(method.name.lexeme, function);
// lox/Interpreter.java, in visitClassStmt(), replace 1 line
```

然后在`bind()`中，即创建绑定`this`到方法的闭包的地方，传递原始方法的值。

```java
    environment.define("this", instance);
    return new LoxFunction(declaration, environment,
                           isInitializer);
  }
// lox/LoxFunction.java, in bind(), replace 1 line
```

### 12.7.2 从 init() 返回

我们还没有走出困境。我们一直假设用户编写的初始化程序不会显式返回值，因为大多数构造函数不会。如果用户尝试：

```js
class Foo {
  init() {
    return "something else";
  }
}
```

肯定不会如他们所愿，所以不妨让它成为一个静态错误。回到解析器，我们向 FunctionType 添加另一个 case。

```java
    FUNCTION,
    INITIALIZER,
    METHOD
// lox/Resolver.java, in enum FunctionType
```

使用被访问方法的名称来确定是否正在解析初始化程序。

```java
      FunctionType declaration = FunctionType.METHOD;
      if (method.name.lexeme.equals("init")) {
        declaration = FunctionType.INITIALIZER;
      }

      resolveFunction(method, declaration); 
// lox/Resolver.java, in visitClassStmt()
```

当稍后遍历`return`语句时，会检查该字段并使其从`init()`方法内部返回值成为错误。

```java
   if (stmt.value != null) {
      if (currentFunction == FunctionType.INITIALIZER) {
        Lox.error(stmt.keyword,
            "Can't return a value from an initializer.");
      }

      resolve(stmt.value);
// lox/Resolver.java, in visitReturnStmt()
```

但我们*还*没有完成。我们静态地不允许从初始化器返回一个*值*，但你仍然可以使用一个空的 `return`。

```js
class Foo {
  init() {
    return;
  }
}
```

有时这实际上很有用，所以我们不想完全禁止它。相反，它应该返回`this`而不是`nil`. 这在 LoxFunction 中很容易解决

```java
    } catch (Return returnValue) {
      if (isInitializer) return closure.getAt(0, "this");

      return returnValue.value;
// lox/LoxFunction.java, in call()
```

如果我们在初始化程序中执行`return`语句，而不是返回值（永远是`nil`），我们再次返回`this`。

哇！这是一个完整的任务列表，但我们的回报是我们的小解释器已经形成了一个完整的编程范式。类、方法、字段、`this`和构造函数。我们的婴儿语言看起来非常成熟。

--

## [挑战](http://craftinginterpreters.com/classes.html#challenges)

1. 我们在实例上有方法，但是没有办法定义可以直接在类对象本身上调用的“静态”方法。添加对他们的支持。在方法之前使用`class`关键字来指示类对象的静态方法。
   
   ```js
   class Math {
     class square(n) {
       return n * n;
     }
   }
   
   print Math.square(3); // Prints "9". 
   ```
   
   您可以随心所欲地解决这个问题，但是 Smalltalk 和 Ruby 使用的“[元类](https://en.wikipedia.org/wiki/Metaclass)”是一种特别优雅的方法。*提示：使 LoxClass 扩展 LoxInstance 并从那里开始。*

2. 大多数现代语言都支持“getters”和“setters”——类中的成员看起来像字段读写，但实际上执行用户定义的代码。扩展 Lox 以支持 getter 方法。这些是在没有参数列表的情况下声明的。当访问具有该名称的属性时，将执行 getter 的主体。
   
   ```js
   class Circle {
     init(radius) {
       this.radius = radius;
     }
   
     area {
       return 3.141592653 * this.radius * this.radius;
     }
   }
   
   var circle = Circle(4);
   print circle.area; // Prints roughly "50.2655".
   ```

3. Python 和 JavaScript 允许您从对象自身的方法之外自由访问对象的字段。Ruby 和 Smalltalk 封装了实例状态。只有类上的方法可以访问原始字段，并且由类决定公开哪个状态。大多数静态类型语言都提供修饰符，如`private`和`public`来控制类的哪些部分可以在每个成员的基础上从外部访问。
   
   这些方法之间的权衡是什么？为什么一种语言可能更喜欢其中一种？

## [设计说明：原型和能量](http://craftinginterpreters.com/classes.html#design-note)

在本章中，我们介绍了两个新的运行时实体 LoxClass 和 LoxInstance。**前者是对象行为所在，后者是状态所在**。如果您可以在 LoxInstance 内部的单个对象上直接定义方法会怎样？在那种情况下，我们根本不需要 LoxClass。LoxInstance 将是一个完整的包，用于定义对象的行为和状态。

我们仍然需要某种方式，无需类，即可跨多个实例重用行为。我们可以让一个 LoxInstance直接[*委托*](https://en.wikipedia.org/wiki/Prototype-based_programming#Delegation)给另一个 LoxInstance 来重用它的字段和方法，有点像继承。

用户会将他们的程序建模为一组对象，其中一些对象相互委托以反映共性。用作委托的对象表示其他人改进的“规范”或“原型”对象。结果是一个更简单的运行时，只有一个内部构造 LoxInstance。

这就是此范例名称**[原型](https://en.wikipedia.org/wiki/Prototype-based_programming)**的来源。它是由 David Ungar 和 Randall Smith 用一种叫做[Self](http://www.selflanguage.org/)的语言发明的。他们通过从 Smalltalk 开始并按照上述脑力练习想出了它，看看他们可以将它削减多少。

很长一段时间以来，原型都是一种学术上的好奇心，一种引人入胜的好奇心，它产生了有趣的研究，但并未对更大的编程世界产生影响。也就是说，直到 Brendan Eich 将原型塞进 JavaScript，然后它迅速接管了世界。关于 JavaScript 中的原型，已经写了很多（很多）词。这是否表明原型是出色的还是令人困惑的——或者两者兼而有之！-是一个悬而未决的问题。

包括你真正[的少数](http://gameprogrammingpatterns.com/prototype.html)。

我不会讨论我是否认为原型对于一门语言来说是个好主意。我制作了[原型](http://finch.stuffwithstuff.com/)语言和[基于类的](http://wren.io/)语言，我对这两种语言的看法都很复杂。我想讨论的是*简单性*在语言中的作用。

原型比类更简单——语言实现者编写的代码更少，用户学习和理解的概念也更少。这会让他们变得更好吗？我们语言书呆子有迷恋极简主义的倾向。就个人而言，我认为简单只是等式的一部分。我们真正想给用户的是*能量*，我定义为：

```
power = breadth × ease ÷ complexity
```

这些都不是精确的数字度量。我在这里使用数学作为类比，而不是实际的量化。

- **广度 Breadth** 是语言允许您表达的不同事物的范围。C 的应用范围很广——它被用于从操作系统到用户应用程序再到游戏的方方面面。AppleScript 和 Matlab 等领域特定语言的应用范围较小。

- **易用性 ease**就是让语言做你想做的事需要付出多少努力。“可用性”可能是另一个术语，尽管它承载的包袱比我想要的要多。“高级”语言往往比“低级”语言更容易。大多数语言都有一个“纹理”，即有些东西比其他东西更容易表达。

- **复杂性 complexity**是语言（包括其运行时、核心库、工具、生态系统等）的规模。人们谈论一种语言的规范有多少页，或者它有多少关键字。这是用户在系统中工作之前必须加载到他们的湿件中的量。它是简单的反义词。

降低复杂性*确实会*增加能量。分母越小，结果值就越大，所以我们认为简单就是好的直觉是正确的。但是，在降低复杂性时，我们必须注意不要牺牲过程中的广度或易用性，否则总功率可能会下降。如果Java 删除了字符串，它会是一种非常*简单*的语言，但它可能无法很好地处理文本操作任务，也不会那么容易地完成任务。

那么，艺术就是寻找可以忽略的*偶然复杂性*——语言特征和交互不会通过增加语言的广度或易用性来增加它们的重量。

如果用户想根据对象类别来表达他们的程序，那么将类烘焙到语言中会增加这样做的便利性，希望有足够大的余地来支付增加的复杂性。但是，如果这不是用户使用您的语言的方式，那么请务必将课程排除在外。