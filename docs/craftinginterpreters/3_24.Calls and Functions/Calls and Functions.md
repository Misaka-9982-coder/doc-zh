# 函数调用

> 计算机科学中的任何问题都可以通过一个间接层来解决。除了间接层太多的问题。
> 
> -- David Wheeler

这一章是一个大怪物。我尝试将特性分解成小的片段，但有时你必须吞下整顿饭。下一个任务是函数。我们可以只从函数声明开始，但是当不能调用它们时这就不是很有用了。我们可以调用，但没有什么可调用的。如果 VM 没有连接到您可以看到的任何内容，那么 VM 中支持这两者所需的所有运行时支持都不是很有价值。所以我们要全力以赴。任务很多，但完成后我们会感觉很好。

> 吃——消费——是对创造性行为的奇怪比喻。但是，大多数产生“输出”的生物过程都少了一点，咳咳，正派。

## 24.1 函数对象

VM 中最有趣的结构变化都是关于栈的。我们已经有了局部变量和临时变量的栈，所以已经完成了一半。但是我们没有调用栈(*call*?stack)的概念。在我们取得很大进展之前，必须解决这个问题。但首先，让我们编写一些代码。一旦我开始运动，我总是感觉好多了。如果没有某种东西表示函数，我们将无能为力，所以将从这里开始。从 VM 的角度来看，什么是函数？

一个函数有一个可以执行的主体，所以这意味着一些字节码。我们可以将整个程序及其所有函数声明编译成一个大的整体块。每个函数都有一个指针，指向块内其代码的第一条指令。

这大致就是编译为本机代码的工作方式，您最终会得到一个完整的机器代码块。但是对于字节码 VM，我们可以做一些更高层次的事情。我认为更简洁的模型是为每个函数赋予它自己的 Chunk。还需要一些其他元数据，所以让我们继续将其全部填充到一个结构中。

```c
  struct Obj* next;
};

typedef struct {
  Obj obj;
  int arity;
  Chunk chunk;
  ObjString* name;
} ObjFunction;

struct ObjString {
// object.h, add after struct Obj
```

函数在 Lox 中是一等公民的，因此它们必然是 Lox 对象。因此 ObjFunction 具有相同Obj 标头,这与所有其他对象类型是一样的。`arity`字段存储函数期望的参数数量。然后，除了块之外，还存储函数的名称。这对于报告可读的运行时错误非常方便。

> 人们似乎没有发现数字字节码偏移量在崩溃转储中特别有启发性。

这是“object”模块第一次需要引用 Chunk，所以需要 include。

```c
#include "common.h"
#include "chunk.h"
#include "value.h"
// object.h
```

就像对字符串所做的那样，我们定义了一些辅助函数以使 Lox 函数更容易在 C 中使用。有点像寒酸的面向对象。首先，我们将声明一个 C 函数来创建一个新的 Lox 函数。

```c
  uint32_t hash;
};

ObjFunction* newFunction();
ObjString* takeString(char* chars, int length);
// object.h, add after struct ObjString
```

实现在这里：

```c
ObjFunction* newFunction() {
  ObjFunction* function = ALLOCATE_OBJ(ObjFunction, OBJ_FUNCTION);
  function->arity = 0;
  function->name = NULL;
  initChunk(&function->chunk);
  return function;
}
// object.c, add after allocateObject()
```

使用`ALLOCATE_OBJ()`来分配内存并初始化对象的标头，以便 VM 知道它是什么类型的对象。我们没有像使用 ObjString 那样传入参数来初始化函数，而是将函数设置为一种空白状态—0 arity、无名称和无代码。稍后将在创建函数后填写。

因为我们有一种新的对象，所以需要在枚举中有一个新的对象类型。

```c
typedef enum {
  OBJ_FUNCTION,
  OBJ_STRING,
} ObjType;
// object.h, in enum ObjType
```

当用完一个函数对象时，必须将它占用的内存还给给操作系统。

```c
  switch (object->type) {
    case OBJ_FUNCTION: {
      ObjFunction* function = (ObjFunction*)object;
      freeChunk(&function->chunk);
      FREE(ObjFunction, object);
      break;
    }
    case OBJ_STRING: {
// memory.c, in freeObject()
```

这个 switch case负责释放 ObjFunction 本身以及它拥有的任何其他内存。函数拥有它们的块，所以需要调用块类的析构函数。

> 不需要显式释放函数的名称，因为它是一个 ObjString。这意味着我们可以让垃圾收集器为我们管理它的生命周期。或者，至少，一旦我们[实现了垃圾收集器](http://craftinginterpreters.com/garbage-collection.html)，我们就能够做到。

Lox 让你打印任何对象，而函数是一等公民的对象，所以也需要处理它们。

```c
  switch (OBJ_TYPE(value)) {
    case OBJ_FUNCTION:
      printFunction(AS_FUNCTION(value));
      break;
    case OBJ_STRING:
// object.c, in printObject()
```

这会调用：

```c
static void printFunction(ObjFunction* function) {
  printf("<fn %s>", function->name->chars);
}
// object.c, add after copyString()
```

既然一个函数知道它的名字，它也可以说出来。

最后，我们有几个宏用于将值转换为函数。首先，确保值实际上是一个函数。

```c
#define OBJ_TYPE(value)        (AS_OBJ(value)->type)

#define IS_FUNCTION(value)     isObjType(value, OBJ_FUNCTION)
#define IS_STRING(value)       isObjType(value, OBJ_STRING)
// object.h
```

假设计算结果为 true，然后您可以使用以下方法安全地将 Value 转换为 ObjFunction 指针：

```c
#define IS_STRING(value)       isObjType(value, OBJ_STRING)

#define AS_FUNCTION(value)     ((ObjFunction*)AS_OBJ(value))
#define AS_STRING(value)       ((ObjString*)AS_OBJ(value))
// object.h
```

这样，我们的对象模型就知道如何表示函数了。热身完毕。你准备好接受更难的挑战了吗？

## 24.2 编译函数对象

现在，我们的编译器假设它总是编译成一个单独的块。由于每个函数的代码都位于不同的块中，因此变得更加复杂。当编译器遇到函数声明时，它需要在编译函数体时将代码发射到函数块中。在函数体的末尾，编译器需要返回到它正在处理的前一个块。

这对于函数体内的代码来说很好，但是如果代码不是这样呢？Lox 程序的“顶层”也是命令式代码，也需要将其编译成一个块。我们也可以通过将顶级代码放在自动定义的函数中来简化编译器和 VM。这样，编译器总是在某种函数体中，VM 总是通过调用函数来运行代码。就好像整个程序都包裹在一个隐式`main()`函数中。

> 对于全局变量该类比失效。它们具有不同于局部变量的特殊作用域规则，因此在这种情况下，脚本的顶层不像函数体。

在使用用户定义的函数之前，让我们进行重构以支持隐式顶级函数。它从 Compiler 结构开始。它不是直接指向编译器写入的块，而是引用正在构建的函数对象。

```c
typedef struct {
  ObjFunction* function;
  FunctionType type;

  Local locals[UINT8_COUNT];
// compiler.c, in struct Compiler
```

还有一些 FunctionType 枚举。这让编译器知道它是在编译顶层代码还是在编译函数体。大多数编译器并不关心这一点——这就是为什么它是一个有用的抽象——但在一两个地方，这种区别是有意义的。我们稍后再讲。

```c
typedef enum {
  TYPE_FUNCTION,
  TYPE_SCRIPT
} FunctionType;
// compiler.c, add after struct Local
```

编译器中写入块的每个地方现在都需要通过`function`指针。幸运的是，在许多章节之前，我们将对块的访问封装在`currentChunk()`函数中。现在只需要修复它，其余的编译器就会很高兴。

> 这几乎就像我有一个可以预见未来的水晶球，并且知道我们以后需要更改代码。但是，实际上，这是因为我在写书之前就编写了本书的所有代码。

```c
Compiler* current = NULL;

static Chunk* currentChunk() {
  return & current->function->chunk;
}

static void errorAt(Token* token, const char* message) {
// compiler.c, add after variable current, replace 5 lines
```

当前块始终是正在编译的函数所拥有的块。接下来，我们需要实际创建该函数。以前，VM 将一个块传递给编译器，编译器用代码填充它。相反，编译器将创建并返回一个函数，该函数包含用户程序的已编译顶级代码——这是我们目前支持的所有代码。

### 24.2.1 在编译时创建函数

我们开始在`compile()`对其按顺序进行处理，这是编译器的主要入口点。

```c
  Compiler compiler;
  initCompiler(&compiler, TYPE_SCRIPT);

  parser.hadError = false;
// compiler.c, in compile(), replace 2 lines
```

编译器的初始化方式有很多变化。首先，我们初始化编译器新的字段。

```c
static void initCompiler(Compiler* compiler, FunctionType type) {
  compiler->function = NULL;
  compiler->type = type;
  compiler->localCount = 0;
// compiler.c, function initCompiler(), replace 1 line
```

然后我们分配一个新的函数对象来编译。

```c
  compiler->scopeDepth = 0;
  compiler->function = newFunction();
  current = compiler;
// compiler.c, in initCompiler()
```

> 我明白，将`function`字段先设为 null然后在几行之后立即为其赋值看起来很愚蠢。更多与垃圾收集相关的偏执狂。

在编译器中创建 ObjFunction 可能看起来有点奇怪。函数对象是函数的运行时表示，但这里是在编译时创建它。可以这样想，函数类似于字符串或数字字面量。它在编译时和运行时世界之间架起了一座桥梁。当我们谈到函数声明时，它们实际上是字面量——它们是一种产生内置类型值的符号。所以编译器在编译期间创建函数对象。然后，在运行时，它们被简单地调用。

> 我们可以在编译时创建函数，因为它们只包含编译时可用的数据。函数的代码、名称和元数(arity)都是固定的。当我们在[下一章](http://craftinginterpreters.com/closures.html)中添加在运行时捕获变量的闭包时，情况会变得更加复杂。

这是另一段奇怪的代码：

```c
  current = compiler;

  Local* local = & current->locals[current->localCount++];
  local->depth = 0;
  local->name.start = "";
  local->name.length = 0;
}
// compiler.c, in initCompiler()
```

请记住，编译器的`locals`数组会跟踪哪些堆栈槽与哪些局部变量或临时变量相关联。从现在开始，编译器隐式声明堆栈槽为 0 供 VM 自己内部使用。我们给它一个空名称，这样用户就不能写一个引用它的标识符。当它变得有用时，我会解释它是关于什么的。

那是初始化方面。当完成一些代码的编译时，我们还需要在另一端进行一些更改。

```c
static ObjFunction* endCompiler() {
  emitReturn();
// compiler.c, function endCompiler(), replace 1 line
```

以前，当`interpret()`调用编译器时，它会传入要写入的块。现在编译器自己创建了函数对象，返回那个函数。我们在这里从当前的编译器中获取它：

```c
  emitReturn();
  ObjFunction* function = current->function;

#ifdef DEBUG_PRINT_CODE
// compiler.c, in endCompiler()
```

然后像这样返回到`compile()`：

```c
#endif

  return function;
}
// compiler.c, in endCompiler()
```

现在是对该函数进行另一次调整的好时机。早些时候，我们添加了一些诊断代码让 VM 转储反汇编的字节码，以便我们可以调试编译器。我们应该修复它以继续工作，因为生成的块被包装在一个函数中。

```c
#ifdef DEBUG_PRINT_CODE
  if (!parser.hadError) {
    disassembleChunk(currentChunk(), function->name != NULL
        ? function->name->chars : "<script>");
  }
#endif
// compiler.c, in endCompiler(), replace 1 line
```

请注意此处的检查以查看函数的名称是否为`NULL`?用户定义的函数有名称，但我们为顶层代码创建的隐式函数没有，在我们自己的诊断代码中，也需要优雅地处理它。如下：

```c
static void printFunction(ObjFunction* function) {
  if (function->name == NULL) {
    printf("<script>");
    return;
  }
  printf("<fn %s>", function->name->chars);
// object.c, in printFunction()
```

用户无法获得对顶级函数的引用并尝试打印它，但我们`DEBUG_TRACE_EXECUTION`打印整个堆栈的诊断代码可以而且确实如此。

> 如果我们用来查找错误的诊断代码本身会导致 VM 发生段错误，那可就没意思了！

将级别提高到`compile()`，我们调整其函数签名。

```c
#include "vm.h"

ObjFunction* compile(const char* source);

#endif
// compiler.h, function compile(), replace 1 line
```

现在它返回一个函数，而不是返回一个块。实现如下：

```c
ObjFunction* compile(const char* source) {
  initScanner(source);
// compiler.c, function compile(), replace 1 line
```

最后我们得到一些实际的代码。我们将函数的最后更改为：

```c
  while (!match(TOKEN_EOF)) {
    declaration();
  }

  ObjFunction* function = endCompiler();
  return parser.hadError ? NULL : function;
}
// compiler.c, in compile(), replace 2 lines
```

我们从编译器中获取函数对象。如果没有编译错误，就返回它。否则，通过返回来发出错误信号`NULL`。这样，VM 就不会尝试执行可能包含无效字节码的函数。

最终，我们将更新`interpret()`以处理`compile()`的新声明，但首先我们要进行一些其他更改。

## 24.3 调用栈帧

是时候进行概念上的重大飞跃了。在我们可以实现函数声明和调用之前，需要让 VM 准备好处理它们。需要关心的主要问题有两个：

### 24.3.1 分配局部变量

编译器为局部变量分配栈槽。当程序中的局部变量集分布在多个函数中时，它应该如何工作？

一种选择是将它们完全分开。每个函数都将在 VM 栈中获得自己的一组专用插槽，它将永远拥有这些插槽，即使未调用该函数也是如此。整个程序中的每个局部变量在 VM 中都会有一些内存供自己使用。

> 这基于如果你在 C 程序中使用`static` 声明每个局部变量.

信不信由你，早期的编程语言实现就是这样工作的。第一个 Fortran 编译器为每个变量静态分配内存。明显的问题是它真的很低效。大多数函数在任何时间点都不在被调用的中间，因此为它们占用未使用的内存是一种浪费。

然而，更根本的问题是递归。通过递归，您可以同时“参与”对同一函数的多次调用。每个都需要自己的内存来存储其局部变量。在 jlox 中，我们通过在每次调用函数或输入块时为环境动态分配内存来解决这个问题。在 clox 中，我们不希望每次函数调用都有这种性能成本。

> Fortran 通过完全禁止递归避免了这个问题。递归在当时被认为是一种高级的、深奥的特性。

现在，我们的解决方案介于 Fortran 的静态分配和 jlox 的动态方法之间。VM 中的值堆栈基于观察局部变量和临时变量以后进先出的方式运行。对我们来说幸运的是，即使您将函数调用混合使用，情况仍然如此。这是一个例子：

```c
fun first() {
  var a = 1;
  second();
  var b = 2;
}

fun second() {
  var c = 3;
  var d = 4;
}

first();
```

逐步执行程序并查看每个时间点内存中有哪些变量：

![](./assets/9272b64430c0cf2f4f1e17b42104fefa1b663c15.png)

当执行流经这两个调用时，每个局部变量都遵循以下原则：后边声明的变量都在前边声明的变量之前丢弃(any variable declared after it will be discarded before the first variable needs to be)。即使在跨越多个函数调用也是如此，我们知道将丢弃`a`之前 会先丢弃`c`和`d`。看上去我们应该能在 VM 的栈上分配局部变量。

理想情况下，我们仍然在编译时确定每个变量将在堆栈中的位置。这使得用于处理变量的字节码指令简单快速。在上面的示例中，我们可以"想象以一种直接的方式这样做，但这并不总是可行。考虑：

> 我说“想象”是因为编译器实际上无法解决这个问题。因为函数在 Lox 中是一等公民的，所以我们无法在编译时确定哪些函数调用了哪些其他函数。

```c
fun first() {
  var a = 1;
  second();
  var b = 2;
  second();
}

fun second() {
  var c = 3;
  var d = 4;
}

first();
```

在对`second()` 的第一次调用中，c和d将进入槽 1 和 2。但是在第二次调用中，我们需要为 b 腾出空间,，因此c和d需要位于插槽 2 和 3 中。因此编译器无法为跨函数调用的每个局部变量确定一个确切的插槽。但是在一个给定的函数中，每个局部变量的相对位置是固定的。变量d总是在c之后的槽中。这是关键的地方。

当一个函数被调用时，我们不知道栈顶在哪里，因为它可以从不同的上下文中调用。但是，无论栈顶恰好在哪里，我们都知道函数的所有局部变量相对于该起点的位置。因此，与许多问题一样，我们通过间接级别解决分配问题。

在每个函数调用的开始，VM 记录该函数自己的第一个局部变量开始的槽的位置。使用局部变量的指令通过相对于局部变量的槽索引访问它们，而不是像今天那样相对于栈底部。在编译时，我们计算这些相对槽。在运行时，我们通过添加函数调用的起始槽将相对槽转换为绝对栈索引。

就好像函数在更大的堆栈中获得了一个“窗口”或“框架”，它可以在其中存储其局部变量。**调用帧(call frame)** 的位置是在运行时确定的，但在其内部使用相对位置，我们知道在哪里可以找到东西。

![](./assets/50de8229a829a1101ca7bde51a50659edc108a78.png)

历史上，函数局部变量开始的位置 被叫做 **帧指针**，因为它指向函数调用帧的开头。有时你会听到**base pointer**，因为它指向所有函数变量所在的基地址栈槽(base stack slot)。

这是我们需要跟踪的第一条数据。每次我们调用一个函数时，VM 都会确定该函数变量开始的第一个栈槽。

### 24.3.2 返回地址

现在，虚拟机通过递增`ip`字段来处理指令流。唯一有趣的行为是围绕控制流指令，这些指令对`ip`实施了更大的数量的偏移。调用函数非常简单——只需将其`ip`设置为指向该函数块中的第一条指令即可。但是当函数完成时呢？

VM 需要返回调用函数的块，并在调用后恢复执行后边跟着的指令。因此，对于每个函数调用，我们需要跟踪调用完成时跳回的位置。这称为**返回地址**，因为它是 VM 在调用后返回的指令地址。

同样，由于递归，单个函数可能有多个返回地址，因此这是每个*调用*的属性，而不是函数本身。

> 早期 Fortran 编译器的作者有一个实现返回地址的巧妙技巧。由于它们*不*支持递归，因此任何给定的函数在任何时间点都只需要一个返回地址。因此，当在运行时调用函数时，程序会*修改自己的代码*，将函数末尾的跳转指令更改为跳转回其调用者。有时天才与疯子之间的界限很短。

### 24.3.3 调用栈

因此，对于每个实时函数调用——每个尚未返回的调用——需要跟踪函数局部变量在栈中的何处开始，以及调用者应从何处恢复。我们将把它和其他一些东西一起放在一个新的结构中。

```c
#define STACK_MAX 256

typedef struct {
  ObjFunction* function;
  uint8_t* ip;
  Value* slots;
} CallFrame;

typedef struct {
// vm.h
```

CallFrame 表示单个正在进行的函数调用。该`slots`字段指向此函数可以使用的第一个插槽处的 VM 值堆栈。我给它取了一个复数名称，因为——多亏了 C语言怪诞 “指针是一种数组”——我们会把它当作数组来对待。

返回地址的实现与我上面描述的有点不同。调用者不是将返回地址存储在被调用者的帧中，而是存储调用者自己的`ip`.当我们从函数返回时，VM 将跳转到调用者 CallFrame 的 `ip`， 并从那里恢复。

我还在这里填充了一个指向被调用函数的指针。我们将使用它来查找常量和其他一些东西。

每次调用函数时，我们都会创建一个这样的结构。可以在堆上动态分配它们，但这很慢。函数调用是核心操作，因此需要尽可能快。幸运的是，观察结果和变量相同：函数调用具有栈语义。如果`first()`调用`second()`，对`second()`调用将在 `first()`调用之前完成。

> 许多 Lisp 实现动态分配栈帧，因为它简化了[延续(continuations)](https://en.wikipedia.org/wiki/Continuation)的实现。如果您的语言支持延续，那么函数调用*并不*总是具有堆栈语义。

因此，在 VM 中，我们预先创建了这些 CallFrame 结构的数组，并将其视为栈，就像我们处理值数组一样。

```c
typedef struct {
  CallFrame frames[FRAMES_MAX];
  int frameCount;

  Value stack[STACK_MAX];
// vm.h, in struct VM, replace 2 lines
```

这个数组取代了过去直接在 VM 中拥有的`chunk`和`ip`字段。现在每个 CallFrame 都有自己`ip`的指针，指向它正在执行的 ObjFunction。从那里，我们可以找到函数的块。

VM 中的新`frameCount`字段存储 CallFrame 堆栈的当前高度——正在进行的函数调用的数量。为了使 clox 简单，数组的容量是固定的。这意味着，与许多语言实现一样，我们可以处理最大调用深度。对于 clox，它在此处定义：

```c
#include "value.h"

#define FRAMES_MAX 64
#define STACK_MAX (FRAMES_MAX * UINT8_COUNT)

typedef struct {
// vm.h, replace 1 line
```

我们还根据它重新定义了值栈的大小，以确保即使在非常深的调用树中我们也有足够的栈槽。VM 启动时，CallFrame 堆栈为空。

> 如果有足够多的函数调用使用了足够多的临时变量而不是本地变量，仍然有可能溢出堆栈。一个健壮的实现可以防止这种情况发生，但我试图让事情变得简单。

```c
  vm.stackTop = vm.stack;
  vm.frameCount = 0;
}
// vm.c, in resetStack()
```

“vm.h”头文件需要访问 ObjFunction，因此添加了头文件。

```c
#define clox_vm_h

#include "object.h"
#include "table.h"
// vm.h, replace 1 line
```

现在我们准备好转到 VM 的实现文件。我们前面有一些繁重的工作。我们已经把`ip`从VM 结构移到了 CallFrame。我们需要修复 VM 中涉及`ip`处理该问题的每一行代码。此外，需要更新通过栈槽访问局部变量的指令，以相对于当前 CallFrame 的`slots`字段执行此操作。

我们将从头开始，逐步推进。

```c
static InterpretResult run() {
  CallFrame* frame = &vm.frames[vm.frameCount - 1];

#define READ_BYTE() (*frame->ip++)

#define READ_SHORT() \
    (frame->ip += 2, \
    (uint16_t)((frame->ip[-2] << 8) | frame->ip[-1]))

#define READ_CONSTANT() \
    (frame->function->chunk.constants.values[READ_BYTE()])

#define READ_STRING() AS_STRING(READ_CONSTANT())
// vm.c, in run(), replace 4 lines
```

首先，将当前最顶层的 CallFrame 存储在主函数内的局部变量中(即CallFrame* frame=...)。然后用通过frame访问`ip`的版本替换之前的字节码访问宏。

> 我们每次都可以通过 CallFrame 数组来访问当前帧，但这很冗长。更重要的是，**将帧存储在局部变量中会鼓励 C 编译器将该指针保存在寄存器中**。这加快了frame对 `ip`的访问.不能*保证*编译器会这样做，但很有可能会这样做。

现在开始仔细微调涉及到的每个指令。

```c
      case OP_GET_LOCAL: {
        uint8_t slot = READ_BYTE();
        push(frame->slots[slot]);
        break;
// vm.c, in run(), replace 1 line
```

以前，`OP_GET_LOCAL`直接从 VM 的栈数组中读取给定的本地槽，这意味着它从栈底部开始索引槽。现在，它访问当前帧的`slots`数组，这意味着它访问相对于该帧开头的给定编号槽。

设置局部变量的工作方式相同。

```c
      case OP_SET_LOCAL: {
        uint8_t slot = READ_BYTE();
        frame->slots[slot] = peek(0);
        break;
// vm.c, in run(), replace 1 line
```

用于修改VM`ip`字段的跳转指令。现在，对当前帧的`ip`也做同样的修改.

```c
      case OP_JUMP: {
        uint16_t offset = READ_SHORT();
        frame->ip += offset;
        break;
// vm.c, in run(), replace 1 line
```

条件跳转也相同：

```c
      case OP_JUMP_IF_FALSE: {
        uint16_t offset = READ_SHORT();
        if (isFalsey(peek(0))) frame->ip += offset;
        break;
// vm.c, in run(), replace 1 line
```

以及我们的向后跳转循环指令：

```c
      case OP_LOOP: {
        uint16_t offset = READ_SHORT();
        frame->ip -= offset;
        break;
// vm.c, in run(), replace 1 line
```

我们有一些诊断代码可以在执行时打印每条指令，以帮助我们调试 VM。这也需要与新结构一起工作。

```c
    printf("\n");
    disassembleInstruction(&frame->function->chunk,
        (int)(frame->ip - frame->function->chunk.code));
#endif
// vm.c, in run(), replace 2 lines
```

现在我们从当前的 CallFrame 中读取，而不是传递 VM的`chunk`和`ip`字段。

你知道，实际上那并不算太糟糕。大多数指令只使用宏，因此不需要修改。接下来，我们跳到调用`run()`.

```c
InterpretResult interpret(const char* source) {
  ObjFunction* function = compile(source);
  if (function == NULL) return INTERPRET_COMPILE_ERROR;

  push(OBJ_VAL(function));
  CallFrame* frame = &vm.frames[vm.frameCount++];
  frame->function = function;
  frame->ip = function->chunk.code;
  frame->slots = vm.stack;

  InterpretResult result = run();
// vm.c, in interpret(), replace 10 lines
```

我们终于可以将早期的编译器更改连接到刚刚所做的后端更改。首先，将源代码传递给编译器。它向我们返回一个新的 ObjFunction，其中包含已编译的顶级代码。如果返回`NULL`，则意味着编译器已经报告了一些编译时错误。在那种情况下，会退出，因为无法运行任何东西。

否则，将函数存储在堆栈上并准备一个初始 CallFrame 来执行其代码。现在你可以明白为什么编译器会留出堆栈槽零——它存储被调用的函数。在新的 CallFrame 中，我们指向函数，将其初始化`ip`为指向函数字节码的开头，并将其堆栈窗口设置为从 VM 值堆栈的最底部开始。

这使解释器准备好开始执行代码。完成后，VM 用于释放硬编码块。现在 ObjFunction 拥有该代码，不需要再这样做了，所以就是这样结束`interpret()`：

```c
  frame->slots = vm.stack;

  return run();
}
// vm.c, in interpret(), replace 4 lines
```

引用旧 VM 字段的最后一段代码是`runtimeError()`.我们将在本章后面重新讨论它，但现在将其更改为：

```c
  fputs("\n", stderr);

  CallFrame* frame = &vm.frames[vm.frameCount - 1];
  size_t instruction = frame->ip - frame->function->chunk.code - 1;
  int line = frame->function->chunk.lines[instruction];
  fprintf(stderr, "[line %d] in script\n", line);
// vm.c, in runtimeError(), replace 2 lines
```

它不是直接从 VM 读取块和ip，而是从栈上最顶层的 CallFrame 中读取。那应该使该功能再次运行并像以前一样。

假设我们正确地完成了所有这些，让 clox 回到了可运行状态。启动它，它就会启动...正是它之前所做的。我们还没有添加任何新功能，所以这有点令人失望。但是所有的基础设施都在那里，现在已经准备好了。让我们利用它。

## 24.4 函数声明

在可以完成调用表达式之前，需要有一些东西可以调用，所以先做函数声明。从fun关键字开始。

> `fun`是的，每次出现关键字时，我都会开一个愚蠢的玩笑。

```c
static void declaration() {
  if (match(TOKEN_FUN)) {
    funDeclaration();
  } else if (match(TOKEN_VAR)) {
    varDeclaration();
// compiler.c, in declaration(), replace 1 line
```

将控制权传递给这里：

```c
static void funDeclaration() {
  uint8_t global = parseVariable("Expect function name.");
  markInitialized();
  function(TYPE_FUNCTION);
  defineVariable(global);
}
// compiler.c, add after block()
```

函数是一等公民，函数声明只是创建一个值并将其存储在新声明的变量中。所以我们像解析任何其他变量声明一样解析名称。顶层的函数声明会将函数绑定到全局变量。在块或其他函数内部，函数声明创建一个局部变量。

在前面的章节中，我解释了如何[分两个阶段定义](http://craftinginterpreters.com/local-variables.html#another-scope-edge-case)变量。这确保您**无法在变量自己的初始化程序中访问自己的值**。那会很糟糕，因为变量还没有值。

函数不会遇到这个问题。函数在其主体内引用自己的名称是安全的。在函数完全定义之前，您不能调用函数并执行函数体，因此您永远不会看到处于未初始化状态的变量。实际上，为了支持递归本地函数，允许这样做很有用。

为了让它工作，在编译函数体之前，在编译名称时立即将函数声明的变量标记为“已初始化”。这样就可以在正文中引用名称而不会产生错误。

不过，我们确实需要检查以下。

```c
static void markInitialized() {
  if (current->scopeDepth == 0) return;
  current->locals[current->localCount - 1].depth =
// compiler.c, in markInitialized()
```

之前，我们只在本地作用域内时才调用`markInitialized()`。现在，顶级函数声明也将调用此函数。当发生这种情况时，没有局部变量可以标记为已初始化——该函数被绑定到一个全局变量。

接下来，我们编译函数本身——它的参数列表和函数体。为此，使用一个单独的辅助函数。该辅助函数生成的代码将生成的函数对象留在堆栈顶部。之后，我们调用`defineVariable()`将该函数存储回我们为它声明的变量中。

我拆分出编译参数和函数体的代码，因为稍后将重用它来解析类内的方法声明。让我们逐步构建它，从这个开始：

```c
static void function(FunctionType type) {
  Compiler compiler;
  initCompiler(&compiler, type);
  beginScope(); 

  consume(TOKEN_LEFT_PAREN, "Expect '(' after function name.");
  consume(TOKEN_RIGHT_PAREN, "Expect ')' after parameters.");
  consume(TOKEN_LEFT_BRACE, "Expect '{' before function body.");
  block();

  ObjFunction* function = endCompiler();
  emitBytes(OP_CONSTANT, makeConstant(OBJ_VAL(function)));
}
```

> 这`beginScope()`没有相应的`endScope()`调用。因为当我们到达函数体的末尾时我们完全结束了 Compiler，所以没有必要关闭继续存留的最外层作用域。

现在，先不用关心参数。我们解析一对空括号，然后是正文。主体以左花括号开始，我们在这里解析它。然后调用现有的`block()`函数，它知道如何编译一个块的其余部分，包括右括号。

### 24.4.1 编译器的栈

有趣的部分是编译器的顶部和底部。Compiler结构 存储数据，例如哪些槽由哪些局部变量拥有，当前有多少嵌套块，等等。所有这些都是特定于单个函数的。但是现在前端需要处理相互嵌套的多个函数的编译。

> 请记住，编译器将顶层代码视为隐式函数的主体，因此一旦我们添加*任何*函数声明，我们就进入了嵌套函数的世界。

管理它的技巧是为每个正在编译的函数创建一个单独的编译器。当开始编译函数声明时，在 C 堆栈上创建一个新的 Compiler 并对其进行初始化。`initCompiler()`将该编译器设置为当前编译器。然后，当编译函数体时，所有发出字节码的函数都会写入新编译器函数拥有的块。

到达函数块主体的末尾后，我们调用`endCompiler()`.这会产生新编译的函数对象，将其作为常量存储在*外围surrounding* 函数的常量表中。但是，等等，我们如何回到外围函数？我们在`initCompiler()`覆盖 当前编译器指针时丢失了它。

我们通过将一系列嵌套的 Compiler 结构视为栈来解决这个问题。与 VM 中的 Value 和 CallFrame 堆栈不同，我们不会使用数组。相反，我们使用链表。每个编译器都指向包含它的函数的编译器，一直指向顶级代码的根编译器。

```c
} FunctionType;

typedef struct Compiler {
  struct Compiler* enclosing;
  ObjFunction* function;
// compiler.c, add after enum FunctionType, replace 1 line
```

在 Compiler 结构中，我们无法引用 Compiler 的类型定义 *typedef*，因为该声明尚未完成。相反，我们为结构本身命名并将其用于enclosing字段的类型。C很奇怪。

当初始化一个新的编译器时，在该指针中捕获即将不再是当前的编译器。

```c
static void initCompiler(Compiler* compiler, FunctionType type) {
  compiler->enclosing = current;
  compiler->function = NULL;
// compiler.c, in initCompiler()
```

然后，当编译器完成时，它通过将以前的编译器恢复为新的当前编译器，将自己从堆栈中弹出。

```c
#endif

  current = current->enclosing;
  return function;
// compiler.c, in endCompiler()
```

请注意，我们甚至不需要动态分配Compiler结构。每个都作为局部变量存储在 C 栈中—在`compile()`或`function()`中。编译器链表穿过 C 堆栈。我们可以获得无限数量的它们的原因是因为编译器使用递归下降，所以`function()`当你有嵌套函数声明时最终会递归调用自身。

> 将本机堆栈用于编译器结构确实意味着编译器对嵌套函数声明的深度有实际限制。走得太远，可能会溢出 C 堆栈。如果我们希望编译器对病态代码甚至恶意代码更加健壮——这是 JavaScript VM 等工具真正关心的问题——最好让我们的编译器人为地限制它允许的函数嵌套数量。

### 24.4.2 函数参数

如果不能向函数传递参数，函数就不是很有用，所以接下来我们来处理参数。

```c
 consume(TOKEN_LEFT_PAREN, "Expect '(' after function name.");
  if (!check(TOKEN_RIGHT_PAREN)) {
    do {
      current->function->arity++;
      if (current->function->arity > 255) {
        errorAtCurrent("Can't have more than 255 parameters.");
      }
      uint8_t constant = parseVariable("Expect parameter name.");
      defineVariable(constant);
    } while (match(TOKEN_COMMA));
  }
  consume(TOKEN_RIGHT_PAREN, "Expect ')' after parameters.");
// compiler.c, in function()  
```

从语义上讲，参数只是在函数体的最外层词法范围内声明的局部变量。可以使用现有的编译器支持来声明命名局部变量来解析和编译参数。与具有初始值设定项的局部变量不同，此处没有用于初始化参数值的代码。当我们在函数调用中传递参数时，我们将看到它们是如何初始化的。

当我们这样做时，通过解析的参数数量来记录函数的arity。我们用函数存储的另一块元数据是它的名称。编译函数声明时，我们`initCompiler()`会在解析函数名称后立即调用。这意味着可以立即从先前的token中获取名称。

```c
  current = compiler;
  if (type != TYPE_SCRIPT) {
    current->function->name = copyString(parser.previous.start,
                                         parser.previous.length);
  }

  Local* local = & current->locals[current->localCount++];
// compiler.c, in initCompiler()
```

注意，我们小心地创建了名称字符串的副本。请记住，词素直接指向原始源代码字符串。一旦代码完成编译，该字符串可能会被释放。我们在编译器中创建的函数对象比编译器寿命更长，并且一直存在到运行时。所以它需要它自己的堆分配名称字符串，它可以一直保留。

好。现在我们可以像这样编译函数声明：

```c
fun areWeHavingItYet() {
  print "Yes we are!";
}

print areWeHavingItYet;
```

我们只是不能对他们做任何有用的事情。

> 我们可以打印它们！不过，我想这不是很有用。

## 24.5 函数调用

到本节结束时，我们将开始看到一些有趣的行为。下一步是调用函数。我们通常不会这样想，但是函数调用表达式是一种中缀`(`运算符。对于被调用的事物，左侧有一个高优先级表达式——通常只是一个标识符。然后是中间的`(` ，后面是用逗号分隔的参数表达式，最后是一个 `)`把它包起来。

那个奇怪的语法视角解释了如何将语法挂载到解析表中。

```c
ParseRule rules[] = {
  [TOKEN_LEFT_PAREN]    = {grouping, call,   PREC_CALL},
  [TOKEN_RIGHT_PAREN]   = {NULL,     NULL,   PREC_NONE},
// compiler.c, add after unary(), replace 1 line
```

当解析器遇到表达式后的左括号时，它会分派到一个新的解析器函数。

```c
static void call(bool canAssign) {
  uint8_t argCount = argumentList();
  emitBytes(OP_CALL, argCount);
}
// compiler.c, add after binary()
```

我们已经使用了`(`token，所以接下来使用单独的`argumentList()`帮助程序编译参数。该函数返回它编译的参数数量。每个参数表达式都会生成代码，将其值留在栈中以准备调用。之后，发出一条新`OP_CALL`指令来调用该函数，使用参数计数作为操作数。

我们使用这个辅助函数来编译参数：

```c
static uint8_t argumentList() {
  uint8_t argCount = 0;
  if (!check(TOKEN_RIGHT_PAREN)) {
    do {
      expression();
      argCount++;
    } while (match(TOKEN_COMMA));
  }
  consume(TOKEN_RIGHT_PAREN, "Expect ')' after arguments.");
  return argCount;
}
// compiler.c, add after defineVariable()
```

该代码在 jlox 中看起来应该很熟悉。只要我们在每个表达式后找到逗号，就会仔细研究参数。在结束时，会消耗最后的右括号，就完成了。

嗯，差不多。回到 jlox，我们添加了一个编译时检查，以确保您不会将超过 255 个参数传递给调用。当时，我说那是因为 clox 需要类似的限制。现在你明白为什么了——因为我们将参数计数作为单字节操作数填充到字节码中，所以最多只能达到 255。我们也需要在这个编译器中验证这一点。

```c
      expression();
      if (argCount == 255) {
        error("Can't have more than 255 arguments.");
      }
      argCount++;
// compiler.c, in argumentList()
```

那是前端。让我们跳到后端，在中间快速停下来声明新指令。

```c
  OP_LOOP,
  OP_CALL,
  OP_RETURN,
// chunk.h, in enum OpCode
```

### 24.5.1 将绑定参数  Binding arguments to parameters

在我们开始实现之前，应该考虑调用时堆栈的样子以及我们需要从那里做什么。当遇到 call 指令时，已经执行了被调用函数的表达式，后面是它的参数。假设我们的程序看起来像这样：

```c
fun sum(a, b, c) {
  return a + b + c;
}

print 4 + sum(5, 6, 7);
```

如果在`OP_CALL`调用 `sum()`的指令处暂停 VM，则栈如下所示：

![](./assets/eacc6621bb0beb2c0468b5030d6291ce45a1e364.png)

从`sum()`它本身的角度想象一下。编译器编译`sum()`时，自动分配槽零。然后，之后，它按顺序为参数`a`、`b`和`c`分配本地槽。要执行对 `sum()`的调用，需要   初始完成被调用函数的CallFrame 以及它可以使用的栈槽区域。然后需要收集传递给函数的参数并将它们放入参数的相应槽中。

当 VM 开始执行的主体时`sum()`，我们希望它的栈窗口看起来像这样：

![](./assets/e8a05347acdc6636a37a560953c99aca1f821dee.png)

您是否注意到调用者设置的参数槽和被调用者需要的参数槽是如何以完全正确的顺序排列的？多方便啊！这并非巧合。当我谈到每个 CallFrame 在栈中都有自己的窗口时，我从未说过这些窗口必须是*不相交*的。没有什么能阻止我们重叠它们，就像这样：

![](./assets/d174fbecef70881418565e8c9abeed5b6c8059a7.png)

调用者(caller)栈的顶部包含被调用的函数，后面依次是参数。我们知道调用者在使用的槽之上没有任何其他槽，因为现在已经丢弃了评估参数表达式时所需的任何临时对象。被调用者(callee’s)堆栈的底部重叠，因此参数槽与参数值已经存在的位置完全对齐。

> 不同的字节码VM和真实的CPU架构有不同的调用约定，这是它们用来传递参数、存储返回地址等的具体机制。我这里使用的机制是基于Lua的干净、快速的虚拟机。

这意味着不需要做*任何*工作来“将参数绑定到参数”。插槽之间或环境之间没有复制值。争论已经恰好在他们需要的地方。在性能方面很难超越它。

是时候执行调用指令了。

```c
      }
      case OP_CALL: {
        int argCount = READ_BYTE();
        if (!callValue(peek(argCount), argCount)) {
          return INTERPRET_RUNTIME_ERROR;
        }
        break;
      }
      case OP_RETURN: {
// vm.c, in run()
```

需要知道被调用的函数以及传递给它的参数数量。我们从指令的操作数中得到后者。这也告诉我们通过从栈顶部计算过去的参数槽来在堆栈中找到函数的位置。我们将该数据交给一个单独的`callValue()`函数。如果返回`false`，则表示该调用导致了某种运行时错误。当这种情况发生时，中止解释器。

如果`callValue()`成功，调用函数的 CallFrame 栈上将有一个新帧。该`run()`函数有自己的指向当前帧的缓存指针，因此我们需要更新它

```c
          return INTERPRET_RUNTIME_ERROR;
        }
        frame = &vm.frames[vm.frameCount - 1];
        break;
// vm.c, in run()
```

由于字节码调度循环从该`frame`变量中读取，因此当 VM 开始执行下一条指令时，它将`ip`从新调用的函数的 CallFrame 中读取并跳转到其代码。执行该调用的工作从这里开始：

```c
static bool callValue(Value callee, int argCount) {
  if (IS_OBJ(callee)) {
    switch (OBJ_TYPE(callee)) {
      case OBJ_FUNCTION: 
        return call(AS_FUNCTION(callee), argCount);
      default:
        break; // Non-callable object type.
    }
  }
  runtimeError("Can only call functions and classes.");
  return false;
}
// vm.c add after peek()
```

> 现在使用`switch`语句来检查单个类型有点过分了，但是当我们添加case来处理其他可调用类型时就有意义了。

除了初始化一个新的 CallFrame，这里还有更多的事情要做。因为 Lox 是动态类型的，所以没有什么可以阻止用户编写错误的代码，例如：

```c
var notAFunction = 123;
notAFunction();
```

如果发生这种情况，运行时需要安全地报告错误并停止。所以我们做的第一件事就是检查我们试图调用的值的类型。如果它不是一个函数，就会出错。否则，实际调用发生在这里：

```c
static bool call(ObjFunction* function, int argCount) {
  CallFrame* frame = &vm.frames[vm.frameCount++];
  frame->function = function;
  frame->ip = function->chunk.code;
  frame->slots = vm.stackTop - argCount - 1;
  return true;
}
// vm.c, add after peek()
```

这只是初始化栈上的下一个 CallFrame。它存储指向被调用函数的指针，并将帧的ip指向函数字节码的开头。最后，它设置`slots`指针以将框架的窗口放入栈。那里的算法确保栈上的参数与函数的参数对齐：

![](./assets/06d4a725228976bb3d9a905a4fcbb205886014ed.png)

有趣的是`- 1`是要考虑栈槽零，当我们稍后添加方法时，编译器会为它预留。参数从 slot 1 开始，所以让窗口提前一个 slot 开始，以使它们与参数对齐。

在继续之前，让我们将新指令添加到我们的反汇编程序中。

```c
      return jumpInstruction("OP_LOOP", -1, chunk, offset);
    case OP_CALL:
      return byteInstruction("OP_CALL", chunk, offset);
    case OP_RETURN:
// debug.c, in disassembleInstruction()
```

还有一次快速的旅行。现在我们有了启动 CallFrame 的便捷函数，我们不妨使用它来设置执行顶层代码的第一个框架。

```c
  push(OBJ_VAL(function));
  call(function, 0);

  return run();
// vm.c, in interpret(), replace 4 lines
```

好的，现在回到调用...

### 24.5.2 运行时错误检查

重叠堆栈窗口的工作原理是假设调用为函数的每个参数传递了一个参数。但是，同样，因为 Lox 不是静态类型的，愚蠢的用户可能会传递太多或太少的参数。在 Lox 中，我们将其定义为运行时错误，会像这样报告：

```c
static bool call(ObjFunction* function, int argCount) {
  if (argCount != function->arity) {
    runtimeError("Expected %d arguments but got %d.",
        function->arity, argCount);
    return false;
  }

  CallFrame* frame = &vm.frames[vm.frameCount++];
// vm.c, in call()
```

非常简单。这就是为什么将每个函数的元数(arity)存储在 ObjFunction 中。

还有一个需要报告的错误，它与用户的愚蠢关系不大，而与我们自己的愚蠢关系更大。因为 CallFrame 数组具有固定大小，所以我们需要确保深度调用链不会溢出它。

```c
  }

  if (vm.frameCount == FRAMES_MAX) {
    runtimeError("Stack overflow.");
    return false;
  }

  CallFrame* frame = &vm.frames[vm.frameCount++];
// vm.c, in call()
```

在实践中，如果程序接近这个限制，很可能是某些失控的递归代码中存在错误。

### 24.5.3 打印堆栈跟踪

虽然我们讨论的是运行时错误，但让我们花点时间让它们变得更有用。停止运行时错误对于防止 VM 以某种定义不明确的方式崩溃和损毁很重要。但是简单地中止并不能帮助用户修复*导致*该错误的代码。

帮助调试运行时故障的经典工具是**堆栈跟踪**——打印出程序终止时仍在执行的每个函数，以及程序终止时的执行位置。现在我们有了一个调用堆栈并且我们已经方便地存储了每个函数的名称，当运行时错误破坏了用户存在的和谐时，我们可以显示整个堆栈。它看起来像这样：

```c
  fputs("\n", stderr);

  for (int i = vm.frameCount - 1; i >= 0; i--) {
    CallFrame* frame = &vm.frames[i];
    ObjFunction* function = frame->function;
    size_t instruction = frame->ip - function->chunk.code - 1;
    fprintf(stderr, "[line %d] in ", 
            function->chunk.lines[instruction]);
    if (function->name == NULL) {
      fprintf(stderr, "script\n");
    } else {
      fprintf(stderr, "%s()\n", function->name->chars);
    }
  }

  resetStack();
}
// vm.c, in runtimeError(), replace 4 lines
```

> 这个`- 1`是因为 IP 已经位于下一条要执行的指令上，但我们希望堆栈跟踪指向上一条失败的指令。

打印错误消息本身后，我们将调用堆栈从顶部（最近调用的函数）移动到底部（顶级代码）。对于每一帧，我们找到对应于该帧函数内部`ip`当前的行号。然后打印该行号以及函数名称。

> 对于应该在跟踪中显示哪个顺序堆栈帧存在一些分歧。大多数将最内层的函数作为第一行，并朝着堆栈的底部前进。Python 以相反的顺序打印它们。所以从上到下阅读告诉你你的程序是如何到达它所在的位置的，最后一行是错误实际发生的地方。
> 
> 这种风格是有逻辑的。它确保您始终可以看到最里面的函数，即使堆栈跟踪太长而无法显示在一个屏幕上。另一方面，新闻学中的“[倒金字塔](https://en.wikipedia.org/wiki/Inverted_pyramid_(journalism))”告诉我们应该将最重要的信息放在文本块的*最前面*。在堆栈跟踪中，这是实际发生错误的函数。大多数其他语言实现都是这样做的。

例如，如果你运行这个损坏的程序：

```c
fun a() { b(); }
fun b() { c(); }
fun c() {
  c("too", "many");
}

a();
```

它打印出来：

```c
Expected 0 arguments but got 2.
[line 4] in c()
[line 2] in b()
[line 1] in a()
[line 7] in script
```

这看起来还不错，不是吗？

### 24.5.4 从函数返回

我们越来越近了。现在可以调用函数，VM 会执行它们。但还不能从他们那里返回。`OP_RETURN`存在很长时间了，但它总是有一些临时代码挂在其中只是为了让我们脱离字节码循环。真正实现的时机已经到来。

```c
      case OP_RETURN: {
        Value result = pop();
        vm.frameCount--;
        if (vm.frameCount == 0) {
          pop();
          return INTERPRET_OK;
        }

        vm.stackTop = frame->slots;
        push(result);
        frame = &vm.frames[vm.frameCount - 1];
        break;
      }
// vm.c, in run(), replace 2 lines
```

当函数返回一个值时，该值将位于栈顶部。我们即将丢弃被调用函数的整个栈窗口，因此我们弹出该返回值并保存到result。然后丢弃返回函数的 CallFrame。如果那是最后一个 CallFrame，则意味着我们已经完成了顶层代码的执行。整个程序已经完成，所以我们从栈中弹出主脚本函数，然后退出解释器。

否则，丢弃被调用者用于其参数和局部变量的所有插槽。这包括调用者用来传递参数的相同槽。现在调用已完成，调用者不再需要它们。这意味着堆栈的顶部恰好位于返回函数栈窗口的开头。

我们将返回值push到栈中那个新的较低位置。然后我们将`run()`函数的缓存指针更新为当前帧。就像开始调用时一样，在字节码调度循环的下一次迭代中，VM`ip`将从该帧读取，并且执行将跳回到调用者，就在它停止的地方，在`OP_CALL`指令之后。

![](./assets/b6d7b72f696991427c4ce19feee730d7980730a6.png)

注意，我们在这里假设该函数*确实*返回了一个值，但是函数可以执行到其主体的末尾来隐式返回：

```js
fun noReturn() {
  print "Do stuff";
  // No return here.
}

print noReturn(); // ???
```

我们也需要正确处理。在这种情况下，语言被指定为隐式返回`nil`。为了做到这一点，我们添加了这个：

```c
static void emitReturn() {
  emitByte(OP_NIL);
  emitByte(OP_RETURN);
}
// compiler.c, in emitReturn()
```

编译器调用在函数体的末尾`emitReturn()`写入`OP_RETURN`指令。现在，在此之前，它会发出一条压入`nil`堆栈的指令。这样，我们就有了工作函数调用！他们甚至可以带参数！看起来我们几乎知道我们在这里做什么。

## 24.6  return statements

如果你想要一个返回隐式以外的东西的函数`nil`，你需要一个`return`语句。让我们开始工作吧

```c
    ifStatement();
  } else if (match(TOKEN_RETURN)) {
    returnStatement();
  } else if (match(TOKEN_WHILE)) {
// compiler.c, in statement()
```

当编译器看到一个`return`关键字时，它会出现在这里：

```c
static void returnStatement() {
  if (match(TOKEN_SEMICOLON)) {
    emitReturn();
  } else {
    expression();
    consume(TOKEN_SEMICOLON, "Expect ';' after return value.");
    emitByte(OP_RETURN);
  }
}
// compiler.c, add after printStatement()
```

返回值表达式是可选的，因此解析器查找分号token以判断是否提供了值。如果没有返回值，则该语句隐式返回`nil`。我们通过调用 `emitReturn()`来实现它，它会发出一条`OP_NIL`指令。否则，我们编译返回值表达式并用`OP_RETURN`指令返回它。

这与已经实现的`OP_RETURN`指令相同——不需要任何新的运行时代码。这与 jlox 有很大的不同。在jlox中，我们必须在执行`return`语句时使用异常来展开堆栈。那是因为你可以从一些嵌套块的深处返回。由于 jlox 递归地遍历 AST，这意味着需要逃避大量 Java 方法调用。

字节码编译器将其全部扁平化。在解析期间进行递归下降，但在运行时，VM 的字节码调度循环是完全平坦的。C 级别根本没有递归。因此，即使是从某些嵌套块中返回，也与从函数体末尾返回一样简单。

不过，我们还没有完全完成。新`return`语句引入了一个新的编译错误。返回对于从函数返回很有用，但 Lox 程序的顶层也是命令式代码。应该不能从那里返回。

```c
return "What?!";
```

> 在最高级别允许`return`并不是世界上最糟糕的想法。它会为您提供一种提前终止脚本的自然方式。您甚至可以使用返回的数字来指示进程的退出代码。

我们已经指定`return`在任何函数之外有一个语句是一个编译错误，我们这样实现：

```c
static void returnStatement() {
  if (current->type == TYPE_SCRIPT) {
    error("Can't return from top-level code.");
  }

  if (match(TOKEN_SEMICOLON)) {
// compiler.c, in returnStatement()
```

这是我们将 FunctionType 枚举添加到编译器的原因之一。

## 24.7 本机函数

我们的 VM 变得越来越强大。我们有函数、调用、参数、返回。您可以定义许多不同的函数，这些函数可以以有趣的方式相互调用。但是，归根结底，他们真的*什么也做不*了。Lox 程序唯一可以做的用户可见的事情，不管其复杂性如何，就是打印。要添加更多功能，我们需要将它们公开给用户。

**编程语言实现通过本机函数**伸出并触及物质世界。如果您希望能够编写检查时间、读取用户输入或访问文件系统的程序，我们需要添加本机函数——可从 Lox 调用但在 C 中实现——以公开这些功能。

在语言层面，Lox 相当完整——它有闭包、类、继承和其他有趣的东西。它感觉像一种玩具语言的一个原因是因为它几乎没有原生功能。我们可以通过添加一长串它们将它变成一种真正的语言。

然而，研究一堆操作系统操作实际上并不是很有教育意义。一旦您了解了如何将一段 C 代码绑定到 Lox，您就明白了。但是你确实需要看到*一个*，即使是一个单一的本地函数也需要我们构建所有的机制来将 Lox 与 C 接口。所以我们将通过它并完成所有艰苦的工作。然后，完成后，我们将添加一个微小的本机函数来证明它可以工作。

我们需要新机制的原因是，从实现的角度来看，原生函数不同于 Lox 函数。当它们被调用时，它们不会push CallFrame，因为没有字节码指向该帧。他们没有字节码块。相反，它们以某种方式引用了一段本机 C 代码。

通过将本机函数定义为完全不同的对象类型来在 clox 中处理此问题。

```c
} ObjFunction;

typedef Value (*NativeFn)(int argCount, Value* args);

typedef struct {
  Obj obj;
  NativeFn function;
} ObjNative;

struct ObjString {
// object.h, add after struct ObjFunction
```

表示形式比 ObjFunction 更简单——仅仅是一个 Obj 标头和一个指向实现本机行为的 C 函数的指针。本机函数采用参数计数和指向堆栈上第一个参数的指针。它通过该指针访问参数。完成后，它会返回结果值。

与往常一样，新的对象类型带有一些附件。要创建 ObjNative，我们声明一个类似构造函数的函数。

```c
ObjFunction* newFunction();
ObjNative* newNative(NativeFn function);
ObjString* takeString(char* chars, int length);
// object.h, add after newFunction()
```

我们这样实现：

```c
ObjNative* newNative(NativeFn function) {
  ObjNative* native = ALLOCATE_OBJ(ObjNative, OBJ_NATIVE);
  native->function = function;
  return native;
}
// object.c, add after newFunction()
```

构造函数采用 C 函数指针包装在 ObjNative 中。它设置对象头并存储函数。对于对象头，我们需要一个新的对象类型。

```c
typedef enum {
  OBJ_FUNCTION,
  OBJ_NATIVE,
  OBJ_STRING,
} ObjType;
// object.h, in enum ObjType
```

VM 还需要知道如何释放本机函数对象。

```c
    }
    case OBJ_NATIVE:
      FREE(ObjNative, object);
      break;
    case OBJ_STRING: {
// memory.c, in freeObject()
```

这里没有太多内容，因为 ObjNative 不拥有任何额外的内存。所有 Lox 对象都支持的另一个功能是打印。

```c
      break;
    case OBJ_NATIVE:
      printf("<native fn>");
      break;
    case OBJ_STRING:
// object.c, in printObject()
```

为了支持动态类型，我们有一个宏来查看一个值是否是一个本地函数。

```c
#define IS_FUNCTION(value)     isObjType(value, OBJ_FUNCTION)
#define IS_NATIVE(value)       isObjType(value, OBJ_NATIVE)
#define IS_STRING(value)       isObjType(value, OBJ_STRING)
// object.h
```

假如返回 true，此宏从表示本机函数的值中提取 C 函数指针：

```c
#define AS_FUNCTION(value)     ((ObjFunction*)AS_OBJ(value))
#define AS_NATIVE(value) \
    (((ObjNative*)AS_OBJ(value))->function)
#define AS_STRING(value)       ((ObjString*)AS_OBJ(value))
object.h
```

所有这些包袱让 VM 像对待任何其他对象一样对待本机函数。您可以将它们存储在变量中、传递它们、举办生日派对等。当然，我们真正关心的操作是*调用*它们——使用一个作为调用表达式中的左侧操作数。

在`callValue()`我们添加另一种类型的case。

```c
      case OBJ_FUNCTION: 
        return call(AS_FUNCTION(callee), argCount);
      case OBJ_NATIVE: {
        NativeFn native = AS_NATIVE(callee);
        Value result = native(argCount, vm.stackTop - argCount);
        vm.stackTop -= argCount + 1;
        push(result);
        return true;
      }
      default:
// vm.c, in callValue()
```

如果被调用的对象是本地函数，我们会立即调用 C 函数。无需处理 CallFrames 或任何东西。我们只是交给 C，得到结果，然后把它塞回栈中。这使得本机函数尽可能快。

有了这个，用户应该能够调用本机函数，但没有任何可调用的函数。没有诸如外部函数接口之类的东西，用户就无法定义自己的本机函数。这是我们作为 VM 实现者的工作。我们将从一个辅助函数开始，定义一个暴露给 Lox 程序的新本机函数。

```c
static void defineNative(const char* name, NativeFn function) {
  push(OBJ_VAL(copyString(name, (int)strlen(name))));
  push(OBJ_VAL(newNative(function)));
  tableSet(&vm.globals, AS_STRING(vm.stack[0]), vm.stack[1]);
  pop();
  pop();
}
// vm.c, add after runtimeError()
```

它带有一个指向 C 函数的指针和它在 Lox 中的名称。我们将函数包装在 ObjNative 中，然后将其存储在具有给定名称的全局变量中。

您可能想知道为什么在堆栈上压入和弹出名称和函数。这看起来很奇怪，对吧？这是涉及垃圾收集时您必须担心的事情。`copyString()`和`newNative()`动态分配内存。这意味着一旦有了 GC，它们就有可能触发回收。如果发生这种情况，我们需要确保收集器知道我们还没有完成名称和 ObjFunction，这样它就不会将它们从我们下面释放出来。将它们存储在值堆栈上可以实现这一点。

> 如果您没有遵循所有这些，请不要担心。[一旦我们着手实施 GC](http://craftinginterpreters.com/garbage-collection.html)，它就会变得更有意义。

感觉很傻，但在所有这些工作之后，我们将只添加一个小的本机函数。

```c
static Value clockNative(int argCount, Value* args) {
  return NUMBER_VAL((double)clock() / CLOCKS_PER_SEC);
}
// vm.c, add after variable vm
```

这将返回自程序开始运行以来经过的时间，以秒为单位。它可以方便地对 Lox 程序进行基准测试。在 Lox 中，我们将其命名为`clock()`。

```c
  initTable(&vm.strings);

  defineNative("clock", clockNative);
}
// vm.c, in initVM() 
```

要获取 C 标准库`clock()`函数，“vm”模块需要包含。

```c
#include <string.h>
#include <time.h>

#include "common.h"
// vm.c
```

有很多资料需要处理，但我们做到了！输入并尝试一下：

```c
fun fib(n) {
  if (n < 2) return n;
  return fib(n - 2) + fib(n - 1);
}

var start = clock();
print fib(35);
print clock() - start;
```

我们可以编写一个非常低效的递归斐波那契函数。更好的是，我们可以衡量它的效率*有多*低。当然，这不是计算斐波那契数列的最聪明的方法。但这是对语言实现对函数调用的支持进行压力测试的好方法。在我的机器上，在 clox 中运行它比在 jlox 中快五倍。这是一个很大的进步。

> 它比在 Ruby 2.4.3p205 中运行的同类 Ruby 程序慢一点，但比在 Python 3.7.3 中运行的程序快约 3 倍。我们仍然可以在 VM 中进行许多简单的优化。

---

## [挑战](http://craftinginterpreters.com/calls-and-functions.html#challenges)

1. 读取和写入`ip`字段是字节码循环中最频繁的操作之一。现在，通过指向当前的CallFrame 指针访问它。这需要一个间接指针，这可能会迫使 CPU 绕过高速缓存并访问主内存。这可能是一个真正的性能下降。
   
   理想情况下，我们会将其`ip`保存在本机 CPU 寄存器中。C 不允许我们在不进入内联汇编的情况下指定它，但可以构建代码以鼓励编译器进行优化。如果我们直接将 `ip`存储在 C 局部变量中并标记它`register`，C 编译器很有可能会同意我们的礼貌请求。
   
   这确实意味着我们需要在开始和结束函数调用时小心地将本地`ip`加载并存储回正确的 CallFrame 中。实现此优化。编写几个基准测试，看看它如何影响性能。您认为额外的代码复杂性值得吗？

2. 本机函数调用速度很快，部分原因是我们没有验证调用传递的参数是否与函数预期的一样多。我们真的应该这样做，否则在没有足够参数的情况下错误地调用本机函数可能会导致函数读取未初始化的内存。添加 arity 检查。

3. 现在，本地函数无法发出运行时错误信号。在实际实现中，这是我们需要支持的，因为本机函数存在于 C 的静态类型世界中，但从动态类型的 Lox 领域调用。例如，如果用户试图将字符串传递给`sqrt()`，则该本机函数需要报告运行时错误。
   
   扩展本机功能系统以支持它。此功能如何影响本机调用的性能？

4. 添加更多本机功能来执行您认为有用的事情。使用这些编写一些程序。你加了什么？它们如何影响语言的感觉以及它的实用性？