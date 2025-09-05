使用解析器组合进行解释和编译

Rust 是一种系统编程语言。系统编程的典型任务之一是处理*形式语言*。形式语言是由良好定义的逻辑规则指定的语言，在计算机技术的各个方面都得到广泛应用。它们可以广泛地分为命令、编程和标记语言。

要处理形式语言，第一步是解析。**解析**意味着分析一段代码的语法结构，以检查它是否遵守其应使用的语法规则，然后，如果语法得到遵守，生成一个描述解析代码片段结构的数据库结构，以便进一步处理该代码。

在本章中，我们将看到如何处理用形式语言编写的文本，从解析步骤开始，并继续进行几个可能的输出——简单地检查语法、解释程序以及将程序翻译成 Rust 语言。

为了展示这些特性，将定义一种极其简单的编程语言，并围绕它构建四个工具（语法检查器、语义检查器、解释器和翻译器）。

在本章中，你将了解以下主题：

+   使用形式语法定义编程语言

+   将编程语言分为三类

+   学习构建解析器的两种流行技术——编译器编译器和解析器组合器

+   使用名为**Nom**的 Rust 解析器组合库

+   使用 Nom 库根据**上下文无关语法**处理源代码以检查其语法（`calc_parser`）

+   验证变量声明及其在某些源代码中的使用的一致性，同时为代码的最佳执行准备所需的结构（`calc_analyzer`）

+   执行预处理代码，在名为**解释**（`calc_interpreter`）的过程中

+   将预处理代码翻译成另一种编程语言，在名为**编译**（`calc_compiler`）的过程中；例如，将翻译成 Rust 代码

阅读本章后，你将能够编写简单形式语言的语法或理解现有形式语言的语法。你还将能够通过遵循其语法编写任何编程语言的解释器。此外，你还将能够编写将一种形式语言翻译成另一种形式语言的翻译器，遵循它们的语法。

# 技术要求

阅读本章内容，不需要了解前几章的知识。对形式语言理论和技术的了解有帮助但不是必需的，因为所需的知识将在本章中解释。将使用 Nom 库来构建此类工具，因此它将在本章中描述。

本章的完整源代码位于 GitHub 仓库的`Chapter08`文件夹中，网址为[`github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers`](https://github.com/PacktPublishing/Creative-Projects-for-Rust-Programmers)。

# 项目概述

在本章中，我们将构建四个递增复杂性的项目，如下所示：

+   第一个项目（`calc_parser`）将仅是`Calc`语言的语法检查器。实际上，它只是一个解析器，随后是对解析结果的格式化调试打印。

+   第二个项目（`calc_analyzer`）使用第一个项目的解析结果来验证变量声明的完整性和使用的一致性，然后以格式化的调试打印分析结果。

+   第三个项目（`calc_interpreter`）使用分析结果在交互式解释器中执行预处理后的代码。

+   第四个项目（`calc_compiler`）再次使用分析结果将预处理后的代码翻译成等效的 Rust 代码。

# 介绍 Calc

为了使以下解释更加清晰，我们首先定义一个*玩具*编程语言，我们将称之为`Calc`（来自计算器）。玩具编程语言是一种用于演示或证明某事的编程语言，不是用于开发现实世界软件的。以下是一个用`Calc`编写的简单程序示例：

```rs
@first
@second
> first
> second
@sum
sum := first + second
< sum
< first * second
```

以下程序要求用户输入两个数字，然后在控制台上打印这些数字的和与积。让我们逐个语句进行分析，如下所示：

+   前两个语句（`@first`和`@second`）声明了两个变量。`Calc`中的任何变量都代表一个 64 位浮点数。

+   第三和第四个语句（`> first`和`> second`）是输入语句。每个语句都会打印一个问号，等待用户输入一个数字并按*Enter*键。如果按*Enter*键之前没有输入数字或输入了无效的数字，则将值`0`赋给变量。

+   第五个语句声明了`sum`变量。

+   第六个语句（`sum := first + second`）是一个帕斯卡风格的赋值语句。它计算`first`和`second`变量的和，并将结果赋值给`sum`变量。

+   第七和第八个语句执行输出。第七个语句（`< sum`）在控制台上打印`sum`变量的当前值。第八个语句（`< first * second`）计算`first`和`second`变量的当前值的乘积，然后在控制台上打印这种乘积的结果。

`Calc`语言有两个其他运算符——`-`（减号）和`/`（除号）——分别用于指定减法和除法。此外，以下代码显示操作可以在表达式中组合，因此这些是有效的赋值语句：

```rs
y := m * x + q
a := a + b - c / d
```

操作是从左到右执行的，但乘法和除法的优先级高于加法和减法。

除了变量之外，还允许使用数值字面量。因此，你可以编写以下代码：

```rs
a := 2.1 + 4 * 5
```

这个语句将`22.1`赋值给`a`，因为在执行加法之前会先执行乘法。为了强制不同的优先级，允许使用括号，如下面的代码片段所示：

```rs
a := (2.1 + 4) * 5
```

前面的代码片段将`30.5`赋值给`a`。

在前面的代码片段中，除了换行符之外，没有字符将一个语句与下一个语句分开。实际上，`Calc`语言没有用于分隔语句的符号，而且它也不需要这些符号。因此，第一个程序应该等同于以下内容：

```rs
@first@second>first>second@sum sum:=first+second<sum<first*second
```

在前面的代码片段中，没有歧义，因为`@`字符标记了声明的开始，`>`字符标记了输入操作的开始，`<`字符标记了输出操作的开始，而在当前语句不允许变量的位置上的变量标记了赋值的开始。

要理解这种语法，必须解释一些语法术语，如下所述：

+   整个文本是一个**程序**。

+   任何程序都是一系列**语句**。在第一个示例程序中，每一行恰好有一个语句。

+   在某些语句中，可能有一个可以评估的算术公式，例如`a * 3 + 2`。这个公式是一个**表达式**。

+   任何表达式都可以包含更简单表达式的和或差。不包含和或差的简单表达式被称为**项**。因此，任何表达式都可以是一个项（如果它不包含和或差），或者它可以是表达式和项的和，或者它可以是表达式和项的差。

+   任何项都可以包含更简单表达式的乘法或除法。不包含乘法或除法的简单表达式被称为**因子**。因此，任何项都可以是一个因子（如果它不包含乘法或除法），或者它可以是项和因子的乘积，或者它可以是项和因子的除法。以下列出了三种可能的因子类型：

    +   变量的名称，称为**标识符**

    +   数值常数，由数字序列表示，称为**字面量**

    +   括号内的完整表达式，以强制它们的优先级

在`Calc`语言中，为了简单起见，并且与大多数编程语言不同，标识符中不允许使用数字和下划线。因此，任何标识符都是一个非空字母序列。或者换句话说，任何标识符可以是一个字母，或者是一个字母后跟一个标识符。

形式语言的语法可以通过称为**巴科斯-诺尔**形式的符号来指定。使用这种符号，我们的`Calc`语言可以通过以下规则来指定：

```rs
<program> ::= "" | <program> <statement>
<statement> ::= "@" <identifier> | ">" <identifier> | "<" <expr> | <identifier> ":=" <expr>
<expr> ::= <term> | <expr> "+" <term> | <expr> "-" <term>
<term> ::= <factor> | <term> "*" <factor> | <term> "/" <factor>
<factor> ::= <identifier> | <literal> | "(" <expr> ")"
<identifier> := <letter> | <identifier> <letter>
```

上述代码片段中使用的所有规则的说明如下：

+   第一条规则规定，一个程序是一个空字符串或一个程序后跟一个语句。这相当于说，一个程序是一系列零个或多个语句。

+   第二条规则规定，一个语句要么是一个 `@` 字符后跟一个标识符，要么是一个 `>` 字符后跟一个标识符，要么是一个 `<` 字符后跟一个表达式，或者是一个标识符后跟 `:=` 字符对，然后是一个表达式。

+   第三条规则规定，一个表达式要么是一个项，要么是一个项后跟一个 `+` 字符和一个项，或者是一个表达式后跟一个 `-` 字符和一个项。这相当于说，一个表达式是一个项后跟零个或多个项项，其中项项是一个 `+` 或 `-` 操作符后跟一个项。

+   类似地，第四条规则规定，一个项要么是一个因子，要么是一个项后跟一个 `*` 字符和一个因子，或者是一个项后跟一个 `/` 字符和一个因子。这相当于说，一个项是一个因子后跟零个或多个因子项，其中因子项是一个乘法或除法操作符后跟一个因子。

+   第五条规则规定，一个因子要么是一个标识符或一个文字面量，要么是一个括号内的表达式。只有当括号在代码中正确配对时，此规则才成立。

+   第六条规则规定，一个标识符是一个字母或一个标识符后跟一个字母。这相当于说，一个标识符是一系列一个或多个字母。此语法没有指定如何处理大小写敏感性，但我们将假设标识符是大小写敏感的。

此语法未定义 `<letter>` 符号和 `<literal>` 符号的意思，因此在这里进行解释：

+   `<letter>` 符号表示任何 `is_alphabetic` Rust 标准库函数返回 `true` 的字符。

+   `<literal>` 符号表示任何浮点数。实际上，由于我们将使用 Rust 代码来解析、存储和处理它，`Calc` 对 `literal` 的定义与 Rust 对 `f64` 文字面量的定义相同。例如 `-4.56e300` 将被允许，但 `1_000` 和 `3f64` 将不被允许。

关于空白符，已经进行了简化。空白符、制表符和换行符在代码的所有位置都是允许的，除了在标识符内、在文字面量内和在 `:=` 符号内。它们是可选的，但唯一需要空白符的位置是在语句的结束标识符和赋值的开始标识符之间，因为否则这两个标识符将合并成一个。

在本节中，我们定义了`Calc`语言的语法。这样的形式定义被称为**语法**。这是一个非常简单的语法，但它与现实世界编程语言的语法相似。为一种语言拥有形式语法对于构建处理用这种语言编写的代码的软件是有用的。

现在我们已经看到了我们的玩具语言，我们准备好处理用这种语言编写的代码了。第一个任务是构建一个语法检查器，用于验证该语言中任何程序的结构的有效性。

# 理解形式语言及其解析器

正如我们所见，系统编程的典型任务之一是处理**形式语言**。在这样的一些形式语言中，通常执行几种操作。这里列出了其中最典型的：

+   检查一行或文件的语法有效性

+   根据格式规则格式化文件

+   执行用命令语言编写的命令

+   解释用编程语言编写的文件——即立即执行它

+   将用编程语言编写的文件编译成另一种编程语言——即将其翻译成汇编语言或机器语言

+   将标记文件转换为另一种标记语言

+   在浏览器中渲染标记文件

所有这些操作都有共同的第一步——解析。检查一个字符串以根据语法提取其结构的过程称为**解析**。根据我们想要解析的形式语言的类别，至少有三种可能的解析技术。我们将在本节中看到这些类别：**正则语言**、**上下文无关语言**和**上下文相关语言**。

## 正则语言

最简单语言的类别是正则语言，它可以使用正则表达式来定义。

以最简单的方式，正则表达式是使用以下运算符在子字符串之间创建的模式：

+   **连接** **（或序列）**：这意味着一个子字符串必须跟随另一个子字符串；例如，`ab`意味着`b`必须跟随`a`。

+   **交替** **（或选择）**：这意味着一个子字符串可以用另一个子字符串代替；例如，`a|b`意味着`a`或`b`可以交替使用。

+   **Kleene 星号** **（或重复）**：这意味着一个子字符串可以使用零次或多次；例如，`a*`意味着`a`可以使用零次、一次、两次或更多次。

使用这样的运算符时，可以使用括号。因此，以下是一个正则表达式：

*a(bcd|(ef)*)g*

这意味着一个有效的字符串必须以一个*a*开头，后面跟着两个可能的子字符串——一个是字符串*bcd*，另一个是空字符串或字符串*ef*，或者字符串*ef*的任何多次重复，然后必须有*g*。以下是一些属于这种正则语言的字符串：

+   *abcdg*

+   *ag*

+   *aefg*

+   *aefefg*

+   *aefefefg*

+   *aefefefefg*

正则语言的一个优点是它们的解析所需的内存量仅取决于语法，而不取决于正在解析的文本；因此，通常它们即使解析巨大的文本也只需要很少的内存。

正则表达式 crate 是使用正则表达式解析正则语言的最受欢迎的方法。如果您有正则语言要解析，那么建议使用这样的库。例如，检测一个有效的标识符或一个有效的浮点数是正则语言解析器的任务。

## 上下文无关语言

由于编程语言不是简单的正则语言，因此不能使用正则表达式来解析它们。不属于正则语言的典型语言特性是括号的使用。大多数编程语言允许使用`((5))`字符串，但不允许使用`((5)`字符串，因为任何开括号都必须由一个闭括号匹配。这样的规则不能用正则表达式表示。

更一般（因此更强大）的语言类别是上下文无关语言。这些语言由语法定义，就像在前一节关于`Calc`语言的章节中看到的那样，包括某些元素必须匹配的事实（例如括号、方括号、花括号和引号）。

与正则语言不同，上下文无关语言所需的内存量取决于解析的文本。每次遇到一个开括号时，它必须被存储起来以匹配相应的闭括号。尽管这种内存使用通常相当小，并且以**后进先出**（**LIFO**）的方式访问（就像在堆栈数据结构中一样），但它非常高效，因为不需要堆分配。

然而，即使是上下文无关语言也足以用于现实世界的应用，因为现实世界的语言需要是上下文相关的，如下一节所述。

## 上下文相关语言

不幸的是，即使是上下文无关文法（CFGs）也不足以表示现实世界的编程语言。问题在于标识符的使用。

在许多编程语言中，在使用变量之前必须声明它。在任何代码位置，只能使用到该点定义的变量。这样一组可用的标识符被视为解析下一个语句的上下文。在许多编程语言中，这样的上下文不仅包含变量的名称，还包括其类型，以及它肯定已经接收了一个值或它可能仍然未初始化的事实。

为了捕捉这样的约束，可以定义上下文相关的语言，尽管这种形式主义相当难以操作，并且生成的语法解析效率不高。

因此，解析编程语言文本的通常方法是将解析分为几个概念上的遍历，如下所示：

+   **第 1 步**：在可能的情况下使用正则表达式——即解析标识符、字面量、运算符和分隔符。这一步生成一个*标记流*，其中每个标记代表一个解析项。例如，任何标识符都是不同的标记，而空白和注释则被跳过。这一步通常被称为**词法分析**或**lexing**。

+   **第 2 步**：使用上下文无关解析器，即应用语法规则到标记流中。这一步创建了一个表示程序的树形结构，这个结构被称为**语法树**。标记被存储为树的叶子（即终端节点）。这个树仍然可能包含上下文相关的错误，例如未声明的标识符的使用。这一步通常被称为**语法分析**。

+   **第 3 步**：处理语法树，将任何变量使用与该变量的声明关联起来，并可能检查其类型。这一步创建了一个新的数据结构，称为**符号表**，它描述了语法树中找到的所有标识符，并且用对这样的符号表的引用装饰了语法树。这一步通常被称为**语义分析**，因为它通常也涉及类型检查。

当我们有一个装饰的语法树及其相关的符号表时，解析操作就完成了。现在，开发者可以使用这些数据结构执行以下操作：

+   获取语法错误，如果代码无效

+   获取有关如何改进代码的建议

+   获取有关代码的一些度量

+   解释代码（如果语言是编程语言）

+   将代码翻译成另一种语言

在本章中，将执行以下操作：

+   词法分析步骤和语法分析步骤将被组合成一个单一的步骤，该步骤将处理源代码并生成语法树（在`calc_parser`项目中）。

+   语义分析步骤将使用解析器生成的语法树来创建符号表和装饰的语法树（在`calc_analyser`项目中）。

+   符号表和装饰的语法树将被用来执行用`Calc`语言编写的程序（在`calc_interpreter`项目中）。

+   符号表和装饰的语法树还将被用来将程序翻译成 Rust 语言（在`calc_complier`项目中）。

在本节中，我们看到了编程语言的有用分类。即使每种编程语言都属于上下文相关类别，其他类别仍然有用，因为解释器和编译器将正则语法和 CFG 作为它们操作的一部分。

但在看到完整的项目之前，让我们看看构建解析器所使用的技巧，特别是 Nom 库所使用的技巧。

# 使用 Nom 构建解析器

在开始编写`Calc`语言的解析器之前，让我们先看看用于构建解释器和编译器的最流行的解析技术。这是为了理解 Nom 库，它使用这些技术之一。

## 学习关于编译器-编译器和解析器组合器

要获得一个极其快速和灵活的解析器，你需要从头开始构建它。但是，几十年来，一种更简单的方法被用来构建解析器，即使用名为**编译器-编译器**或**编译器生成器**的工具：生成编译器的程序。这些程序接收输入为一个装饰过的语法规范，并为这种语法生成解析器的源代码。然后，这些生成的源代码必须与其他源文件一起编译，以获得可执行的编译器。

这种传统方法现在已有些过时，另一种名为**解析器组合器**的方法已经出现。解析器组合器是一组函数，允许将多个解析器组合起来以获得另一个解析器。

我们已经看到，任何`Calc`程序只是`Calc`语句的序列。如果我们有一个单行`Calc`语句的解析器，并且能够按顺序应用这样的解析器，那么我们就可以解析任何`Calc`程序。

我们应该知道，任何`Calc`语句要么是`Calc`声明，要么是`Calc`赋值，要么是`Calc`输入操作，要么是`Calc`输出操作。如果我们为每个这样的语句都有一个解析器，并且能够交替地应用这些解析器，我们就可以解析任何`Calc`语句。我们可以继续进行，直到我们得到单个字符（或者如果我们使用词法分析器的输出，则是到标记）。因此，一个程序的解析器可以通过组合其组成部分的解析器来获得。

但是，用 Rust 编写的解析器是什么？它是一个接收源代码字符串作为输入并返回结果的函数。结果可以是`Err`（如果该字符串无法解析）或`Ok`（包含表示解析项的数据结构）。

因此，虽然正常函数接收数据作为输入并返回数据作为输出，我们的解析器组合器接收一个或多个具有函数作为输入的解析器，并返回一个解析器作为输出。接收函数作为输入并返回函数作为输出的函数被称为**二阶**函数，因为它们处理函数而不是数据。在计算机科学中，二阶函数的概念起源于函数式语言，解析器组合器的概念也来自这样的语言。

在 Rust 的 2018 版本之前，二阶函数是不可行的，因为 Rust 函数不能返回函数而不分配闭包。因此，Nom 库（直到版本 4）使用宏而不是函数作为组合器以保持高性能。当 Rust 引入了`impl Trait`特性（包含在 2018 版本中）时，使用函数而不是宏实现解析器组合器的有效方法成为可能。因此，Nom 的第 5 版完全基于函数，仅保留宏以保持向后兼容性。

在下一节中，我们将看到 Nom 库的基本功能，我们将使用这些功能来构建解释器和编译器。

## 学习 Nom 的基础知识

Nom crate 实质上是一个函数集合。其中大部分是解析器组合器——也就是说，它们将一个或多个解析器作为参数，并以解析器作为返回值。你可以把它们看作是输入一个或多个解析器并输出组合解析器的机器。

一些 Nom 函数是解析器——也就是说，它们将一个`char`值的序列作为参数，如果解析失败则返回错误，或者在成功的情况下返回表示解析文本的数据结构。

现在，我们将通过非常简单的程序来查看 Nom 的最基本功能。特别是，我们将看到以下内容：

+   `char` 解析器：解析单个固定字符

+   `alt` 解析器组合器：接受替代解析器

+   `tuple` 解析器组合器：接受一组固定的解析器

+   `tag` 解析器：解析固定字符的字符串

+   `map` 解析器组合器：转换解析器的输出值

+   `Result::map` 函数：在解析器的输出上应用更复杂的转换

+   `preceded`、`terminated`和`delimited`解析器组合器：用于接受一组固定的解析器并从输出中丢弃其中的一些

+   `take` 解析器：接受定义数量的字符

+   `many1` 解析器组合器：接受一个或多个解析器的重复序列

### 解析字符的替代方案

作为解析器的例子，让我们看看如何解析固定字符的替代方案。我们想要解析一个非常简单的语言，这种语言只有三个单词——*a*、*b*和*c*。这样的解析器只有在它的输入是字符串*a*、字符串*b*或字符串*c*时才会成功。

如果解析成功，我们想要的结果是一对东西——剩余的输入（即在有效部分被处理后）和已处理文本的表示。由于我们的单词由单个字符组成，我们想要（作为这种表示）一个`char`类型的值，其中只包含解析的字符。

以下是我们使用 Nom 的第一个代码片段：

```rs
extern crate nom;
use nom::{branch::alt, character::complete::char, IResult};

fn parse_abc(input: &str) -> IResult<&str, char> {
    alt((char('a'), char('b'), char('c')))(input)
}

fn main() {
    println!("a: {:?}", parse_abc("a"));
    println!("x: {:?}", parse_abc("x"));
    println!("bjk: {:?}", parse_abc("bjk"));
}
```

如果你编译这个程序，包括 Nom crate 的依赖项，并运行它，它应该打印以下输出：

```rs
a: Ok(("", 'a'))
x: Err(Error(("x", Char)))
bjk: Ok(("jk", 'b'))
```

我们将我们的解析器命名为 `parse_abc`。它接受一个字符串切片作为输入，并返回 `IResult<&str, char>` 类型的值。这种返回值类型是一种 `Result`。这种 `Result` 类型的 `Ok` 情况是两个值的元组——包含剩余输入的字符串切片和一个字符，即通过解析文本获得的信息。这种 `Result` 类型的 `Err` 情况是由 Nom 包内部定义的。

如输出所示，`parse_abc("a")` 表达式返回 `Ok(("", 'a'))`。这意味着当解析 `a` 字符串时，解析成功；没有剩余的输入需要处理，并且提取的字符是 `'a'`。

相反，`parse_abc("x")` 表达式返回 `Err(Error(("x", Char)))`。这意味着当解析 `x` 字符串时，解析失败；`x` 字符串仍然需要处理，并且错误类型是 `Char`，意味着期望一个 `Char` 项目。请注意，`Char` 是由 Nom 定义的一种类型。

最后，`parse_abc("bjk")` 表达式返回 `Ok(("jk", 'b'))`。这意味着当解析字符串 `bjk` 时，解析成功；`jk` 输入仍然需要处理，并且提取的字符是 `'b'`。

现在，让我们看看我们的解析器是如何实现的。为 Nom 构建的任何解析器的签名都必须相似，并且它们的主体必须是一个函数调用，该函数的参数作为其参数（在这种情况下，`(input)`）。

有趣的部分是 `alt((char('a'), char('b'), char('c')))`. 这个表达式意味着我们想要通过组合三个解析器，`char('a')`、`char('b')` 和 `char('c')` 来构建一个解析器。`char` 函数（不要与具有相同名称的 Rust 类型混淆）是一个内置的 Nom 解析器，它识别指定的字符并返回一个包含该字符的 `char` 类型的值。`alt` 函数（简称 alternative，即“选择”）是一个解析器组合器。它只有一个参数，即由几个解析器组成的元组。`alt` 解析器会选择一个与输入匹配的指定解析器。

确保对于任何给定的输入，最多只有一个解析器接受输入，这是你的责任。否则，语法是模糊的。以下是一个模糊解析器的示例——`alt((char('a'), char('b'), char('a')))`。`char('a')` 子解析器被重复，但 Rust 编译器不会注意到这一点。

在下一节中，我们将看到如何解析字符序列。

### 解析字符序列

现在，让我们看看另一个解析器，如下所示：

```rs
extern crate nom;
use nom::{character::complete::char, sequence::tuple, IResult};

fn parse_abc_sequence(input: &str)
    -> IResult<&str, (char, char, char)> {
    tuple((char('a'), char('b'), char('c')))(input)
}

fn main() {
    println!("abc: {:?}", parse_abc_sequence("abc"));
    println!("bca: {:?}", parse_abc_sequence("bca"));
    println!("abcjk: {:?}", parse_abc_sequence("abcjk"));
}
```

运行后，它应该打印以下内容：

```rs
abc: Ok(("", ('a', 'b', 'c')))
bca: Err(Error(("bca", Char)))
abcjk: Ok(("jk", ('a', 'b', 'c')))
```

这次，字母 `a`、`b` 和 `c` 必须按照这个确切的顺序出现，并且 `parse_abc_sequence` 函数返回包含这些字符的元组。对于 `abc` 输入，没有剩余的输入，并且返回 `('a', 'b', 'c')` 元组。`bca` 输入不被接受，因为它以 `b` 字符开头而不是 `a`。`abcjk` 输入被接受，就像第一种情况一样，但这次有一个剩余的输入。

解析器的组合是`tuple((char('a'), char('b'), char('c')))`。这与前面的程序类似，但通过使用`tuple`解析器组合器，获得了一个需要按顺序满足所有指定解析器的解析器。

在下一节中，我们将看到如何解析固定文本字符串。

### 解析固定字符串

在之前讨论的`parse_abc_sequence`函数中，为了识别`abc`序列，必须指定三次`char`解析器，结果是`char`值的元组。

对于较长的字符串（如语言的关键字），这不太方便，因为它们更容易被视为字符串而不是字符序列。Nom 库还包含一个用于固定字符串的解析器，名为`tag`。前面的程序可以使用此内置解析器重写，如下面的代码块所示：

```rs
extern crate nom;
use nom::{bytes::complete::tag, IResult};

fn parse_abc_string(input: &str) -> IResult<&str, &str> {
    tag("abc")(input)
}

fn main() {
    println!("abc: {:?}", parse_abc_string("abc"));
    println!("bca: {:?}", parse_abc_string("bca"));
    println!("abcjk: {:?}", parse_abc_string("abcjk"));
}
```

它将打印以下输出：

```rs
abc: Ok(("", "abc"))
bca: Err(Error(("bca", Tag)))
abcjk: Ok(("jk", "abc"))
```

而不是`tuple((char('a'), char('b'), char('c')))`表达式，现在有一个简单的调用`tag("abc")`，并且解析器返回一个字符串切片，而不是`char`值的元组。

在下一节中，我们将看到如何将解析器产生的值转换成另一个值，可能是另一种类型。

### 将解析后的项映射到其他对象

到目前为止，我们得到的结果只是我们在输入中找到的内容。但通常，我们在返回结果之前想要转换解析后的输入。

假设我们想要解析三个字母（`a`、`b`或`c`），但我们想要在解析的结果中，对于字母`a`返回数字`5`，对于字母`b`返回数字`16`，对于字母`c`返回数字`8`。

因此，我们想要一个解析器，它可以解析一个字母，但，如果解析成功，它返回一个数字，而不是返回那个字母。我们还想将字符`a`映射到数字`5`，字符`b`映射到数字`16`，字符`c`映射到数字`8`。原始的结果类型是`char`，而映射的结果类型是`u8`。以下代码块显示了执行这种转换的程序：

```rs
extern crate nom;
use nom::{branch::alt, character::complete::char, combinator::map, IResult};

fn parse_abc_as_numbers(input: &str)
    -> IResult<&str, u8> {
    alt((
        map(char('a'), |_| 5),
        map(char('b'), |_| 16),
        map(char('c'), |_| 8),
    ))(input)
}

fn main() {
    println!("a: {:?}", parse_abc_as_numbers("a"));
    println!("x: {:?}", parse_abc_as_numbers("x"));
    println!("bjk: {:?}", parse_abc_as_numbers("bjk"));
}
```

当它运行时，它应该打印以下输出：

```rs
a: Ok(("", 5))
x: Err(Error(("x", Char)))
bjk: Ok(("jk", 16))
```

对于`a`输入，提取出`5`。对于`x`输入，得到一个解析错误。对于`bjk`输入，提取出`16`，并且`jk`字符串作为待解析的输入保留。

对于这三个字符中的每一个，实现中包含类似`map(char('a'), |_| 5)`的内容。`map`函数是另一个解析器组合器，它接受一个解析器和闭包。如果解析器匹配，则生成一个值。闭包在这样一个值上被调用，并返回一个转换后的值。在这种情况下，闭包的参数是不需要的。

同一个解析器的另一种等效实现如下：

```rs
fn parse_abc_as_numbers(input: &str) -> IResult<&str, u8> {
    fn transform_letter(ch: char) -> u8 {
        match ch {
            'a' => 5,
            'b' => 16,
            'c' => 8,
            _ => 0,
        }
    }
    alt((
        map(char('a'), transform_letter),
        map(char('b'), transform_letter),
        map(char('c'), transform_letter),
    ))(input)
}
```

它定义了一个`transform_letter`内部函数，该函数应用转换并将该函数仅作为`map`组合器的第二个参数传递。

在下一节中，我们将看到如何以更复杂的方式操作解析器的输出，因为我们将会省略或交换结果元组中的某些字段。

### 创建自定义解析结果

到目前为止，解析的结果是由其中使用的解析器和组合器决定的——如果一个解析器使用三个项目的`tuple`组合器，结果就是一个包含三个项目的元组。这很少是我们想要的。例如，我们可能想要省略结果元组中的某些项目，或者添加一个固定项目，或者交换项目。

假设我们想要解析`abc`字符串，但在结果中我们想要省略`b`，只保留`ac`。为此，我们必须以下述方式后处理解析结果：

```rs
extern crate nom;
use nom::{character::complete::char, sequence::tuple, IResult};

fn parse_abc_to_ac(input: &str) -> IResult<&str, (char, char)> {
    tuple((char('a'), char('b'), char('c')))(input)
        .map(|(rest, result)| (rest, (result.0, result.2)))
}

fn main() {
    println!("abc: {:?}", parse_abc_to_ac("abc"));
}
```

它将打印以下输出：

```rs
abc: Ok(("", ('a', 'c')))
```

当然，我们解析器的结果现在只包含一对——`(char, char)`。后处理在代码体的第二行中体现。它使用了一个`map`函数，这个函数与前面的例子中看到的不同；它属于`Result`类型。这种方法接收一个闭包，该闭包获取结果的`Ok`变体，并返回一个新的`Ok`变体，具有适当的数据类型。如果类型已经明确指定，那么代码将如下所示：

```rs
.map(|(rest, result): (&str, (char, char, char))|
    -> (&str, (char, char)) {
    (rest, (result.0, result.2))
}
```

从前面的代码中，`tuple`的调用返回一个结果，其`Ok`变体具有`(&str, (char, char, char))`类型。第一个元素是剩余的输入，分配给`rest`变量，第二个元素是解析的`char`值序列，分配给`result`变量。

然后，我们必须构造一个包含两个项目的对——即我们想要的*剩余输入*，以及我们想要的*结果*的字符对。作为剩余输入，我们指定由`tuple`提供的相同对，而作为结果，我们指定`(result.0, result.2)`——即第一个和第三个解析的字符，它们将是`'a'`和`'c'`。

以下的一些情况相当典型：

+   两个解析器的序列，需要保留第一个解析器的结果并丢弃第二个解析器的结果。

+   两个解析器的序列，需要丢弃第一个解析器的结果并保留第二个解析器的结果。

+   三个解析器的序列，需要保留第二个解析器的结果并丢弃第一个和第三个解析器的结果。这在括号表达式或引号文本中很典型。

对于这些先前的情况，映射技术同样适用，但 Nom 包含一些特定的组合器，具体如下：

+   `preceded(a, b)`：这仅保留`b`的结果。

+   `terminated(a, b)`：这仅保留`a`的结果。

+   `delimited(a, b, c)`：这仅保留`b`的结果。

在下一节中，我们将看到如何解析指定数量的字符并返回解析的字符。

### 解析可变文本

我们迄今为止所进行的解析非常有局限性，因为我们只是检查了输入是否遵循了一种语言，而没有接受任意文本或数字的可能性。

假设我们想要解析一个以`n`字符开头，后跟两个其他任意字符的文本，并且我们只想处理后面的两个字符。这可以通过`take`内置解析器来完成，如下面的代码片段所示：

```rs
extern crate nom;
use nom::{bytes::complete::take, character::complete::char, sequence::tuple, IResult};

fn parse_variable_text(input: &str)
    -> IResult<&str, (char, &str)> {
    tuple((char('n'), take(2usize)))(input)
}

fn main() {
    println!("nghj: {:?}", parse_variable_text("nghj"));
    println!("xghj: {:?}", parse_variable_text("xghj"));
    println!("ng: {:?}", parse_variable_text("ng"));
}
```

它将打印以下输出：

```rs
nghj: Ok(("j", ('n', "gh")))
xghj: Err(Error(("xghj", Char)))
ng: Err(Error(("g", Eof)))
```

第一次调用是成功的一次。`n`字符被`char('n')`跳过，然后通过`take(2usize)`读取另外两个字符。这个解析器读取由其参数指定的字符数（必须是无符号数），并将这个字节序列作为字符串切片返回。要读取单个字符，只需调用`take(1usize)`，它将返回一个字符串切片。

第二次调用失败，因为缺少起始的`n`。第三次调用失败，因为起始的`n`之后字符少于两个，因此生成了`Eof`（代表**文件结束**）错误。

在下一节中，我们将看到如何通过重复应用给定的解析器来解析一个或多个模式序列。

### 重复解析器

需要解析由解析器识别的重复表达式序列的情况相当常见。因此，必须多次应用该解析器，直到它失败。这种重复是通过几个组合器完成的——即`many0`和`many1`。

前者即使没有解析到表达式的任何实例也会成功——即它是零或更多组合器。后者只有在至少解析到表达式的至少一个实例时才会成功——即它是一或更多组合器。让我们看看如何识别一个或多个`abc`字符串序列，如下所示：

```rs
extern crate nom;
use nom::{bytes::complete::take, multi::many1, IResult};

fn repeated_text(input: &str) -> IResult<&str, Vec<&str>> {
    many1(take(3usize))(input)
}

fn main() {
    println!(": {:?}", repeated_text(""));
    println!("ab: {:?}", repeated_text("abc"));
    println!("abcabcabc: {:?}", repeated_text("abcabcabc"));
}
```

它将打印以下输出：

```rs
: Err(Error(("", Eof)))
abc: Ok(("", ["abc"]))
abcabcabcx: Ok(("x", ["abc", "abc", "abc"]))
```

第一次调用失败，因为空字符串不包含任何`abc`的实例。如果使用了`many0`组合器，这次调用将成功。

另外两次调用仍然成功，并返回找到的实例的`Vec`。

在本节中，我们介绍了最流行的解析技术：编译器编译器和解析器组合器。它们在构建解释器和编译器时都很有用。然后，我们介绍了将在本章剩余部分以及下一章部分使用的 Nom 解析器组合器库。

现在，我们已经对 Nom 有了足够的了解，可以开始看到本章的第一个项目。

# calc_parser 项目

这个项目是`Calc`语言的解析器。它是一个程序，可以检查一个字符串并检测它是否遵循`Calc`语言的语法，使用上下文无关解析器，并在这种情况下，根据语言的语法提取字符串的逻辑结构。这样的结构通常被称为**语法树**，因为它具有树形结构，并且它代表了解析文本的语法。

语法树是一个内部数据结构，因此通常用户看不到，也不需要导出。但是，出于调试目的，这个程序将把这个数据结构格式化打印到控制台。

由本项目构建的程序期望一个 `Calc` 语言文件作为命令行参数。在项目的 `data` 文件夹中，有两个示例程序——即 `sum.calc` 和 `bad_sum.calc`。

第一个是 `sum.calc`，如下所示：

```rs
@a
@b
>a
>b
<a+b
```

它声明了两个变量 `a` 和 `b`，然后要求用户为它们输入值，并打印它们的和。

另一个程序，`bad_sum.calc`，与前者相同，除了第二行——即 `@d`——表示一个错误，因为后来使用了未声明的 `b` 变量。

要在第一个示例 `Calc` 程序上运行项目，请进入 `calc_parser` 文件夹，并输入以下内容：

```rs
cargo run data/sum.calc
```

这样的命令应该在控制台上打印以下文本：

```rs
Parsed program: [
    Declaration(
        "a",
    ),
    Declaration(
        "b",
    ),
    InputOperation(
        "a",
    ),
    InputOperation(
        "b",
    ),
    OutputOperation(
        (
            (
                Identifier(
                    "a",
                ),
                [],
            ),
            [
                (
                    Add,
                    (
                        Identifier(
                            "b",
                        ),
                        [],
                    ),
                ),
            ],
        ),
    ),
]
```

从前面的代码中，首先声明了 `"a"` 标识符，然后是 `"b"` 标识符，接着是对名为 `"a"` 的变量的输入操作，然后是对名为 `"b"` 的变量的输入操作，最后是一个带有许多括号的输出操作。

在 `OutputOperation` 下的第一个开括号代表表达式项的开始，根据之前提出的语法，它必须出现在任何输出操作语句中。这样的表达式包含两个项——一个项和一个操作符-项对列表。

第一个项包含两个项——一个因子和一个操作符-因子对列表。因子是 `"a"` 标识符，操作符-因子对列表为空。然后，让我们将这个传递给操作符-项对列表。它只包含一个项，其中操作符是 `Add`，项是一个因子后跟一个操作符-因子对列表。因子是 `"b"` 标识符，列表为空。

如果运行 `cargo run data/bad_sum.calc` 命令，不会检测到错误，因为这个程序只执行语法分析而不检查语义上下文。输出与之前相同，除了第六行——即 `"d"` 而不是 `"b"`。

现在，让我们检查 Rust 程序的源代码。唯一的第三方库是 **Nom**，这是一个仅用于词法和语法分析阶段的库（因此被本章的所有项目使用，因为它们都需要解析）。

有两个源文件——`main.rs` 和 `parser.rs`。让我们首先看看 `main.rs` 源文件。

## 理解 main.rs 源文件

`main.rs` 源文件只包含 `main` 函数和 `process_file` 函数。`main` 函数只是检查命令行是否包含参数，并将其传递给 `process_file` 函数，同时附带可执行 Rust 程序的路径。

`process_file` 函数检查命令行参数是否以 `.calc` 结尾——即唯一期望的文件类型，然后它将文件的 内容读取到 `source_code` 字符串中，并通过调用 `parser::parse_program(&source_code)`（位于 `parser.rs` 源文件中）来解析该字符串。

当然，这样一个文件是一个整个程序的解析器，因此它返回一个`Result`值。这种返回值的`Ok`变体是由剩余代码和语法树组成的对。然后，通过以下给出的语句将语法树格式化输出：

```rs
println!("Parsed program: {:#?}", parsed_program);
```

当处理只有五行和 17 个字符的小型`sum.calc`文件时，这个单独的`println!`语句会输出之前显示的长输出，共有 35 行和 604 字节。当然，对于更长的程序，输出会更长。

接下来，让我们看看`parser.rs`源文件。

## 了解`parser.rs`源文件

`parser.rs`源文件包含一个针对语言语法中每个语法元素的解析函数。这些函数的详细说明如下：

| **函数** | **描述** |
| --- | --- |
| `parse_program` | 这解析整个`Calc`程序。 |
| `parse_declaration` | 这解析一个`Calc`声明语句，例如`@total`。 |
| `parse_input_statement` | 这解析一个`Calc`输入语句，例如`>addend`。 |
| `parse_output_statement` | 这解析一个`Calc`输出语句，例如`<total`。 |
| `parse_assignment` | 这解析一个`Calc`赋值语句，例如`total := addend * 2`。 |
| `parse_expr` | 这解析一个`Calc`表达式，例如`addend * 2 + val / (incr + 1)`。 |
| `parse_term` | 这解析一个`Calc`项，例如`val / (incr + 1)`。 |
| `parse_factor` | 这解析一个`Calc`因子，例如`incr`，或`4.56e12`，或`(incr + 1)`。 |
| `parse_subexpr` | 这解析一个`Calc`括号表达式，例如`(incr + 1)`。 |
| `parse_identifier` | 这解析一个`Calc`标识符，例如`addend`。 |
| `skip_spaces` | 这解析零个或多个空白字符序列。 |

关于先前声明的语法，有一些解释是必要的——`parse_subexpr`解析器的任务是解析`( <expr> )`序列，丢弃括号，并使用`parse_expr`解析`<expr>`初始表达式。`skip_spaces`函数是一个解析器，其任务是解析零个或多个空白字符（空格、制表符、换行符），目的是忽略它们。

在成功的情况下，所有其他前面的函数都返回一个表示解析代码的数据结构。

由于内置的`double`解析器将用于解析浮点数，因此没有为文字数字提供解析器。

### 理解解析器所需类型

在这个文件中，除了解析器之外，还定义了几个类型来表示解析程序的结构。最重要的类型定义如下：

```rs
type ParsedProgram<'a> = Vec<ParsedStatement<'a>>;
```

前面的代码片段只是说明解析后的程序是一个解析语句的向量。

注意生命周期指定。为了保持最佳性能，最小化内存分配。当然，会分配向量，但解析的字符串不会分配；它们是引用输入字符串的字符串切片。因此，语法树依赖于输入字符串，其生命周期必须短于输入字符串。

前面的声明使用了 `ParsedStatement` 类型，其声明方式如下：

```rs
enum ParsedStatement<'a> {
    Declaration(&'a str),
    InputOperation(&'a str),
    OutputOperation(ParsedExpr<'a>),
    Assignment(&'a str, ParsedExpr<'a>),
}
```

前面的代码片段说明一个解析语句可以是以下之一：

+   一个封装正在声明的变量名称的声明

+   一个封装将要接收输入值的变量名称的输入语句

+   一个封装将要打印的解析表达式值的输出操作

+   一个封装将要接收计算值的变量名称和解析表达式的赋值语句，其值将被分配给该变量

这个声明使用了 `ParsedExpr` 类型，它声明如下：

```rs
type ParsedExpr<'a> = (ParsedTerm<'a>, Vec<(ExprOperator, ParsedTerm<'a>)>);
```

从前面的代码片段中，一个解析表达式是一个由一个解析项和零个或多个对组成的对，其中每个对由一个表达式运算符和一个解析项组成。

表达式运算符定义为 `enum ExprOperator { Add, Subtract }`，而解析项定义为以下内容：

```rs
type ParsedTerm<'a> = (ParsedFactor<'a>, Vec<(TermOperator, ParsedFactor<'a>)>);
```

我们可以看到，一个解析项是一个由一个解析因子和零个或多个对组成的对，其中每个对由一个项运算符和一个解析因子组成。项运算符定义为 `enum TermOperator { Multiply, Divide }`，而解析因子定义为以下内容：

```rs
enum ParsedFactor<'a> {
    Literal(f64),
    Identifier(&'a str),
    SubExpression(Box<ParsedExpr<'a>>),
}
```

这个声明说明一个解析因子可以是一个封装数字的文本，或者是一个封装变量名称的标识符，或者是一个封装解析表达式的子表达式。

注意到 `Box` 的使用。这是必需的，因为任何解析表达式都包含一个包含一个能够包含解析表达式的 `enum` 解析因子的解析项。因此，我们有一个包含的无限递归。如果我们使用一个 `Box`，我们就在主结构之外分配内存。

因此，我们已经看到了将要被解析器代码使用的所有类型的定义。现在，让我们以自顶向下的方式查看代码。

### 查看解析器代码

我们现在可以查看用于解析整个程序的代码。以下代码片段显示了解析器的入口点：

```rs
pub fn parse_program(input: &str) -> IResult<&str, ParsedProgram> {
    many0(preceded(
        skip_spaces,
        alt((
            parse_declaration,
            parse_input_statement,
            parse_output_statement,
            parse_assignment,
        )),
    ))(input)
}
```

注意到其结果类型是 `ParsedProgram`，它是一个解析语句的向量。

主体使用 `many0` 解析组合子来接受零个或多个语句（一个空程序被认为是有效的）。实际上，为了解析一个语句，使用 `preceded` 组合子来组合两个解析器并丢弃第一个解析器的输出。它的第一个参数是 `skip_spaces` 解析器，因此它简单地跳过语句之间的可能空格。第二个参数是 `alt` 组合子，用于接受四种可能语句中的任何一个。

`many0` 组合子生成一个对象向量，这些对象由组合子的参数生成。这些参数生成解析语句，因此我们有了所需的解析语句向量。

因此，总结来说，这个函数接受零个或多个语句，可能由空白字符分隔。在成功的情况下，函数返回的值是一个向量，其元素是解析语句的表示。

`Calc`声明的解析器如下所示：

```rs
fn parse_declaration(input: &str) -> IResult<&str, ParsedStatement> {
    tuple((char('@'), skip_spaces, parse_identifier))(input)
        .map(|(input, output)| (input, ParsedStatement::Declaration(output.2)))
}
```

从前面的代码片段中，一个声明必须是一个由`@`字符、可选的空白字符和一个标识符组成的序列；因此，使用`tuple`组合器来链式调用这样的解析器。然而，我们并不关心那个初始字符也不关心那些空白字符。我们只想得到标识符的文本，封装在`ParsedStatement`中。

因此，在应用`tuple`之后，结果是映射到一个`Declaration`对象，其参数是由`tuple`生成的第三个项。

以下代码片段显示了`Calc`输入语句的解析器：

```rs
fn parse_input_statement(input: &str) -> IResult<&str, ParsedStatement> {
    tuple((char('>'), skip_spaces, parse_identifier))(input)
        .map(|(input, output)| (input, ParsedStatement::InputOperation(output.2)))
}
```

`Calc`输入语句的解析器与前面的解析器非常相似。它只是查找`>`字符，并返回一个封装由`parse_identifier`返回的字符串的`InputOperation`变体。

以下代码片段显示了`Calc`输出语句的解析器：

```rs
fn parse_output_statement(input: &str) -> IResult<&str, ParsedStatement> {
    tuple((char('<'), skip_spaces, parse_expr))(input)
        .map(|(input, output)| (input, ParsedStatement::OutputOperation(output.2)))
}
```

此外，从前面代码的解析器与前面两个解析器相似。它只是查找`<`字符，解析一个表达式而不是标识符，并返回一个封装由`parse_expr`返回的解析表达式的`OutputOperation`。

最后一种`Calc`语句是赋值。它的解析器如下所示：

```rs
fn parse_assignment(input: &str) -> IResult<&str, ParsedStatement> {
    tuple((
        parse_identifier,
        skip_spaces,
        tag(":="),
        skip_spaces,
        parse_expr,
    ))(input)
    .map(|(input, output)| (input, ParsedStatement::Assignment(output.0, output.4)))
}
```

这与前面的语句解析器有所不同。它链式调用五个解析器——一个标识符、一些可能的空白字符、字符串`:=`、一些可能的空白字符和一个表达式。结果是封装了元组中解析的第一个和最后一个项的`Assignment`变体——即标识符字符串和解析的表达式。

我们已经遇到了表达式解析器的使用，它定义如下：

```rs
fn parse_expr(input: &str) -> IResult<&str, ParsedExpr> {
    tuple((
        parse_term,
        many0(tuple((
            preceded(
                skip_spaces,
                alt((
                    map(char('+'), |_| ExprOperator::Add),
                    map(char('-'), |_| ExprOperator::Subtract),
                )),
            ),
            parse_term,
        ))),
    ))(input)
}
```

从前面的代码中，要解析一个表达式，首先必须解析一个项（`parse_term`），然后是零个或多个（`many0`）由一个运算符和一个项（`parse_term`）组成的对（`tuple`）。运算符可以由空白字符（`skip_spaces`）前导，这些空白字符必须被丢弃（`preceded`），并且它可以是加号字符（`char('+'`）或减号字符（`char('-'`）。但我们想用`ExprOperator`值替换这些字符。结果对象已经具有预期的类型，因此不需要其他`map`转换。

项的解析器与表达式的解析器类似。如下所示：

```rs
fn parse_term(input: &str) -> IResult<&str, ParsedTerm> {
    tuple((
        parse_factor,
        many0(tuple((
            preceded(
                skip_spaces,
                alt((
                    map(char('*'), |_| TermOperator::Multiply),
                    map(char('/'), |_| TermOperator::Divide),
                )),
            ),
            parse_factor,
        ))),
    ))(input)
}
```

`parse_expr`和`parse_term`之间的唯一区别如下：

+   当`parse_expr`调用`parse_term`时，`parse_term`调用`parse_factor`。

+   其中`parse_expr`将`'+'`字符映射到`ExprOperator::Add`值，将`'-'`字符映射到`ExprOperator::Subtract`值，`parse_term`将`'*'`字符映射到`TermOperator::Multiply`值，将`'/'`字符映射到`TermOperator::Divide`值。

+   其中`parse_expr`在返回值类型中有`ParsedExpr`类型，`parse_term`有`ParsedTerm`类型。

因子的解析器再次遵循相对语法规则，其返回类型的定义`ParsedFactor`如下代码片段所示：

```rs
fn parse_factor(input: &str) -> IResult<&str, ParsedFactor> {
    preceded(
        skip_spaces,
        alt((
            map(parse_identifier, ParsedFactor::Identifier),
            map(double, ParsedFactor::Literal),
            map(parse_subexpr, |expr|
                ParsedFactor::SubExpression(Box::new(expr))
            ),
        )),
    )(input)
}
```

此解析器会丢弃可能的初始空格，然后交替解析一个标识符、一个数字或一个子表达式。数字解析器是`double`，这是一个 Nom 内置函数，它根据 Rust `f64`字面量的语法解析数字。

所有这些解析的返回类型都是错误的，因此，使用`map`组合器来生成它们的返回值。对于标识符，只需要引用将自动使用`parse_identifier`函数返回的值作为参数构建的`Identifier`变体。一个等效但更冗长的代码将是`map(parse_identifier, |id| ParsedFactor::Identifier(id))`。

类似地，字面量从`f64`类型转换为`ParsedFactor::Literal(f64)`类型，子表达式被装箱并封装在`SubExpression`变体中。

子表达式的解析必须仅匹配并丢弃空格和括号，如下代码片段所示：

```rs
fn parse_subexpr(input: &str) -> IResult<&str, ParsedExpr> {
    delimited(
        preceded(skip_spaces, char('(')),
        parse_expr,
        preceded(skip_spaces, char(')')),
    )(input)
```

内部的`parse_expr`解析器是唯一一个将其输出传递给结果的。为了解析一个标识符，使用了一个内置解析器，如下所示：

```rs
fn parse_identifier(input: &str) -> IResult<&str, &str> {
    alpha1(input)
}
```

`alpha1`解析器返回一个由一个或多个字母组成的字符串。不允许数字和其他字符。通常，这不会命名为*解析器*，而是词法分析器、词法分析器、扫描器或标记器，但 Nom 没有区分。

最后，处理空格的小解析器（或词法分析器）如下所示：

```rs
fn skip_spaces(input: &str) -> IResult<&str, &str> {
    let chars = " \t\r\n";
    take_while(move |ch| chars.contains(ch))(input)
}
```

它使用我们尚未见过的组合器`take_while`。它接收一个返回布尔值（即谓词）的闭包作为参数。这样的闭包会在任何输入字符上被调用。如果闭包返回`true`，解析器将继续，否则停止。因此，它返回输入字符序列的最大序列，对于该序列，谓词值为*true*。

在我们的情况下，谓词检查字符是否包含在四个字符的切片中。

因此，我们已经看到了`Calc`语言的全部解析器。当然，现实世界的解析器要复杂得多，但概念是相同的。

在本节中，我们看到了如何使用 Nom 库通过 CFG 解析用`Calc`语言编写的程序。这是应用**上下文相关语法**（**CSG**）以及随后解释器或编译器的先决条件。

注意，这个程序解析器将任何字符序列都视为有效的标识符，而不检查变量在使用前是否已定义，或者变量是否被多次定义。对于此类检查，必须进行进一步的处理。这将在下一个项目中看到。

# calc_analyzer 项目

前一个项目遵循 CFG 来构建解析器。这很好，但有一个大问题：`Calc`语言不是上下文无关的。事实上，有两个问题，如下所示：

+   在输入语句、输出语句和赋值语句中，任何变量的使用都必须先声明该变量。

+   任何变量都不应被声明多次。

这样的要求无法在上下文无关语言中表达。

此外，`Calc`只有一个数据类型——即浮点数——但考虑一下如果它也有字符串类型会怎样。你可以加减两个数字，但不能减去两个字符串。如果声明了一个名为`a`的变量为`number`类型，而另一个名为`b`的变量为`string`类型，你不能将`a`赋值给`b`，反之亦然。

通常情况下，对一个变量的操作取决于声明该变量时使用的类型。此外，这种约束无法在 CFG 中表达。

为了避免定义一个难以指定和解析的正式**上下文相关语法**（**CDG**），通常的做法是以非正式的方式定义这样的规则，称为**语义**规则，然后对语法树进行后处理以检查这些规则的有效性。

在这里，我们将执行此类语义检查的模块称为`analyzer`（使用一个语义检查器来验证变量的一些约束，例如变量在使用前必须被定义，以及变量不能被定义多次的事实），而`calc_analyzer`项目则是将此模块添加到解析器中的项目。

在下一节中，我们将看到`analyzer`模块的架构。

## 检查解析程序的变量

分析器从解析器结束的地方开始——一个包含标识符字符串、文字值和运算符的语法树。因此，它不再需要源代码。为了完成其任务，它遍历这样的树，每次遇到变量声明时，都必须确保它尚未被声明，而每次遇到变量使用时，都必须确保它已经被声明。

为了在不遍历语法树的情况下执行此类任务，还需要另一个数据结构。这样的数据结构是在遍历语法树时收集已声明的变量集合，当遇到变量声明时，分析器会在此集合中查找该变量的先前声明；如果找到，则是一个重复声明的错误；否则，将向集合中添加一个条目。

此外，当遇到变量使用时，分析器会在这样的集合中查找相同变量的先前声明，尽管这次，如果没有找到，则是一个缺少声明的错误。对于我们的简单语言，这样的集合只包含变量，但在更复杂的语言中，它将包含任何类型的标识符——常量、函数、命名空间等等。标识符的另一个名称是**符号**；因此，通常，这个集合被称为**符号表**。

为了对`Calc`程序进行变量检查，我们的符号表只需要存储变量的名称，尽管我们希望我们的分析器执行一些其他任务，这些任务在构建解释器时将非常有用。解释器在运行程序时必须在某处存储标识符的*值*，而不仅仅是它们的名称，因为我们已经有一个存储每个变量名称的变量集合，我们可以在变量的条目中为每个变量的值预留空间。当我们为`Calc`构建解释器时，这将非常有用。

但这超出了分析器在为解释器做准备时能做的事情。解释器必须扫描一种语法树来执行语句，当它遇到变量时，它必须查找其值。解析器生成的语法树包含变量的标识符，而不是它们的值，所以每当解释器找到一个变量时，它都应该在符号表中搜索那个字符串。

但我们想要一个快速的解释器，而字符串查找并不像使用指针或数组索引那样快。所以，为了准备快速解释，当分析器访问语法树时，它会将每个标识符替换为其在符号表中的位置索引。嗯，在 Rust 中，字符串不能被数字替换，所以一种可能的技术是在语法树中预留一个索引字段，并在变量在符号表中找到时填充该索引。

在这里，选择了另一种技术。分析器在访问语法树的同时，构建了一个并行分析的树，结构非常相似，但使用符号表中的索引而不是标识符。这样的树，加上为变量值预留空间的符号表，将非常适合解释程序。

因此，首先，让我们看看这个项目做了什么。打开`calc_analyzer`文件夹，并输入以下命令：`cargo run data/sum.calc`。

以下输出应出现在控制台上：

```rs
Symbol table: SymbolTable {
 entries: [
 (
 "a",
 0.0,
 ),
 (
 "b",
 0.0,
 ),
 ],
}
Analyzed program: [
 Declaration(
 0,
 ),
 Declaration(
 1,
 ),
 InputOperation(
 0,
 ),
 InputOperation(
 1,
 ),
 OutputOperation(
 (
 (
 Identifier(
 0,
 ),
 [],
 ),
 [
 (
 Add,
 (
 Identifier(
 1,
 ),
 [],
 ),
 ),
 ],
 ),
 ),
]
```

之前的代码程序，就像之前的那个一样，没有为用户输出。它将源代码解析成语法树，然后分析该语法树，构建符号表和分析程序。输出只是这些数据结构的格式化打印。

首先输出的结构是符号表。它有两个条目——`a`变量，其初始值为`0.0`，以及`b`变量，其初始值也为`0.0`。

然后，是分析后的程序，它与上一个项目中打印的解析程序非常相似。唯一的不同之处在于，所有`"a"`的出现都被替换为`0`，所有`"b"`的出现都被替换为`1`。这些数字是这些变量在符号表中的位置。

该项目扩展了前面的项目。`parser.rs`源文件是相同的，另外增加了两个文件——`symbol_table.rs`和`analyzer.rs`。但让我们首先从`main.rs`文件开始。

## 理解`main.rs`文件

此文件执行了前面项目中执行的所有操作，除了最后的格式化打印，它被以下行替换：

```rs
    let analyzed_program;
    let mut variables = symbol_table::SymbolTable::new();
    match analyzer::analyze_program(&mut variables, &parsed_program) {
        Ok(analyzed_tree) => {
            analyzed_program = analyzed_tree;
        }
        Err(err) => {
            eprintln!("Invalid code in '{}': {}", source_path, err);
            return;
        }
    }

    println!("Symbol table: {:#?}", variables);
    println!("Analyzed program: {:#?}", analyzed_program);
```

从前面的代码片段中，分析器构建的两个数据结构首先声明——`analyzed_program`是带有变量索引的语法树，而`variables`是符号表。

所有的分析都是由`analyze_program`函数执行的。如果成功，它将返回分析后的程序，最后打印出这两个结构。

现在，让我们来检查符号表（`symbol_table.rs`）的实现。

## 查看 symbol_table.rs 文件

在`symbol_table.rs`文件中，有一个`SymbolTable`类型的实现，这是一个包含源代码中找到的标识符的集合。符号表的每个条目描述一个变量。这样的条目必须至少包含变量的名称。在类型语言中，它还必须包含该变量的数据类型的表示，尽管`Calc`不需要那个，因为它只有一个数据类型。

如果语言支持在块、函数、类或更大的结构（编译单元、模块、命名空间或包）中进行作用域，那么必须有多个符号表或一个指定这种作用域的符号表，尽管`Calc`不需要那个，因为它只有一个作用域。

符号表主要用于检查标识符和将代码翻译成另一种语言，尽管它也可以用于解释代码。当解释器评估一个表达式时，它需要获取该表达式中使用的变量的当前值。符号表可以用来存储任何变量的当前值，并提供这些值给解释器。因此，如果你想支持解释器，你的符号表也应该为定义的变量的当前值预留空间。

在下一个项目中，我们将创建一个解释器，因此，为了支持它，在这里，我们在符号表的每个条目中添加了一个字段，用于存储变量的当前值。我们符号表的每个条目的类型将是`(String, f64)`，其中第一个字段是变量的名称，第二个字段是变量的当前值。这个值字段将在解释程序时被访问。

我们如何访问符号表中的条目？在分析程序时，我们必须搜索一个字符串，因此哈希表将提供最佳性能。然而，在解释代码时，我们可以用索引替换标识符，因此使用向量中的索引将提供最佳性能。在这里，为了简单起见，选择了一个向量，如果没有很多变量，这已经足够好了。因此，我们的定义如下所示：

```rs
struct SymbolTable {
    entries: Vec<(String, f64)>,
}
```

对于 `SymbolTable` 类型，实现了三种方法，如下面的代码片段所示：

```rs
fn new() -> SymbolTable
fn insert_symbol(&mut self, identifier: &str) -> Result<usize, String>
fn find_symbol(&self, identifier: &str) -> Result<usize, String>
```

`new` 方法简单地创建一个空的符号表。

`insert_symbol` 方法试图将指定的标识符插入到符号表中。如果没有具有该名称的标识符，则为此名称添加一个条目，默认值为零，并且 `Ok` 结果是新条目的索引。否则，在 `Err` 结果中返回错误消息 `Error: Identifier '{}' declared several times.`。

`find_symbol` 方法试图在符号表中找到指定的标识符。如果找到，则 `Ok` 结果是找到条目的索引。否则，在 `Err` 结果中返回错误消息 `Error: Identifier '{}' used before having been declared.`。

现在，让我们看看分析器的源文件。

## 简单地查看 analyzer.rs 文件

如前所述，分析阶段读取解析阶段创建的分层结构，并构建另一个具有 `AnalyzedProgram` 类型的分层结构。因此，此模块必须声明此类以及它需要的所有类型，与 `ParsedProgram` 类型并行，如下所示：

```rs
type AnalyzedProgram = Vec<AnalyzedStatement>;
```

任何分析程序是一系列分析语句，如下面的代码片段所示：

```rs
enum AnalyzedStatement {
    Declaration(usize),
    InputOperation(usize),
    OutputOperation(AnalyzedExpr),
    Assignment(usize, AnalyzedExpr),
}

```

任何分析语句都是以下之一：

+   通过索引引用变量的声明

+   通过索引引用变量的输入操作

+   包含分析表达式的输出操作

+   通过索引引用变量并包含分析表达式的赋值操作

任何分析表达式是一个分析项和一个表达式运算符与一个分析项的零个或多个对的序列，如下面的代码片段所示：

```rs
type AnalyzedExpr = (AnalyzedTerm, Vec<(ExprOperator, AnalyzedTerm)>);
```

任何分析项是一个分析因子和一个项运算符与一个分析因子的零个或多个对的序列，如下面的代码片段所示：

```rs
type AnalyzedTerm = (AnalyzedFactor, Vec<(TermOperator, AnalyzedFactor)>);
```

任何分析因子是一个包含 64 位浮点数的字面量，或通过索引引用变量的标识符，或包含对堆分配的分析表达式的引用的子表达式，如下面的代码片段所示：

```rs
pub enum AnalyzedFactor {
Literal(f64),
Identifier(usize),
SubExpression(Box<AnalyzedExpr>),
}
```

分析器的入口点如下面的代码片段所示：

```rs
fn analyze_program(variables: &mut SymbolTable, parsed_program: &ParsedProgram)
    -> Result<AnalyzedProgram, String> {
    let mut analyzed_program = AnalyzedProgram::new();
    for statement in parsed_program {
        analyzed_program.push(analyze_statement(variables, statement)?);
    }
    Ok(analyzed_program)
}
```

`analyze_program`函数，如同此模块的所有函数一样，获取符号表的可变引用，因为它们都必须直接或间接地读取和写入符号。此外，它获取一个解析程序的引用。如果函数成功，它将更新符号表并返回一个分析程序；否则，它可能留下部分更新的符号表并返回一个错误消息。

主体简单地创建一个空的已分析程序并处理所有解析语句，通过调用`analyze_statement`。任何解析语句都会被分析，分析后的语句被添加到已分析程序中。对于任何语句分析失败，将立即返回此函数的错误。

因此，我们需要了解如何分析语句，如下所示：

```rs
fn analyze_statement(
    variables: &mut SymbolTable,
    parsed_statement: &ParsedStatement,
) -> Result<AnalyzedStatement, String> {
    match parsed_statement {
        ParsedStatement::Assignment(identifier, expr) => {
            let handle = variables.find_symbol(identifier)?;
            let analyzed_expr = analyze_expr(variables, expr)?;
            Ok(AnalyzedStatement::Assignment(handle, analyzed_expr))
        }
        ParsedStatement::Declaration(identifier) => {
            let handle = variables.insert_symbol(identifier)?;
            Ok(AnalyzedStatement::Declaration(handle))
        }
        ParsedStatement::InputOperation(identifier) => {
            let handle = variables.find_symbol(identifier)?;
            Ok(AnalyzedStatement::InputOperation(handle))
        }
        ParsedStatement::OutputOperation(expr) => {
            let analyzed_expr = analyze_expr(variables, expr)?;
            Ok(AnalyzedStatement::OutputOperation(analyzed_expr))
        }
    }
}
```

`analyze_statement`函数将接收到的解析语句与四种语句类型进行匹配，提取相应变体的成员。

在声明中包含的标识符从未被定义过，因此它应该不存在于符号表中。因此，在处理这类语句时，此标识符将通过使用`let handle = variables.insert_symbol(identifier)?`Rust 语句插入到符号表中。如果插入失败，错误将传递出此函数。如果插入成功，符号的位置将存储在一个局部变量中。

在赋值和输入操作中包含的标识符应该已经被定义，因此它应该包含在符号表中。因此，在处理这类语句时，标识符将通过使用`let handle = variables.find_symbol(identifier)?`Rust 语句在符号表中查找。

在赋值和输出操作中包含的表达式通过`let analyzed_expr = analyze_expr(variables, expr)?`Rust 语句进行分析。如果分析失败，错误将传递出此函数。如果分析成功，分析后的结果表达式将存储在一个局部变量中。

对于四种`Calc`语句类型中的任何一种，如果没有遇到错误，函数将返回一个包含相应分析语句变体的成功结果。

因此，我们需要了解如何分析表达式，如下所示：

```rs
fn analyze_expr(
    variables: &mut SymbolTable,
    parsed_expr: &ParsedExpr,
) -> Result<AnalyzedExpr, String> {
    let first_term = analyze_term(variables, &parsed_expr.0)?;
    let mut other_terms = Vec::<(ExprOperator, AnalyzedTerm)>::new();
    for term in &parsed_expr.1 {
        other_terms.push((term.0, analyze_term(variables, &term.1)?));
    }
    Ok((first_term, other_terms))
}
```

接收到的解析表达式是一个对——`&parsed_expr.0`是一个解析项，`&parsed_expr.1`是一个包含表达式运算符和分析项对的向量。我们想要构建一个具有相同结构的分析表达式。

因此，首先分析第一个项。然后，创建一个空的包含表达式运算符和分析项对的列表；这就是分析向量。然后，对于解析向量的每个项目，构建一个项目并添加到分析向量中。最后，返回第一个分析项和其他分析项的向量对。

因此，我们需要了解如何通过以下代码分析项：

```rs
fn analyze_term(
    variables: &mut SymbolTable,
    parsed_term: &ParsedTerm,
) -> Result<AnalyzedTerm, String> {
    let first_factor = analyze_factor(variables, &parsed_term.0)?;
    let mut other_factors = Vec::<(TermOperator, AnalyzedFactor)>::new();
    for factor in &parsed_term.1 {
        other_factors.push((factor.0, analyze_factor(variables, 
         &factor.1)?));
    }
    Ok((first_factor, other_factors))
}
```

上述程序与之前的程序非常相似。首先解析的因子被分析以获得第一个分析因子，其他解析因子被分析以获得其他分析因子。

因此，我们需要了解如何分析因子。如下所示：

```rs
fn analyze_factor(
    variables: &mut SymbolTable,
    parsed_factor: &ParsedFactor,
) -> Result<AnalyzedFactor, String> {
    match parsed_factor {
        ParsedFactor::Literal(value) => 
         Ok(AnalyzedFactor::Literal(*value)),
        ParsedFactor::Identifier(name) => {
            Ok(AnalyzedFactor::Identifier(variables.find_symbol(name)?))
        }
        ParsedFactor::SubExpression(expr) => 
         Ok(AnalyzedFactor::SubExpression(
            Box::<AnalyzedExpr>::new(analyze_expr(variables, expr)?),
        )),
    }
}
```

`analyze_factor` 函数的逻辑如下：

+   如果要分析解析的因子是一个字面量，则返回一个分析后的字面量，其中包含相同的值。

+   如果它是一个标识符，则返回一个分析后的标识符，其中包含解析到的标识符在符号表中的索引。如果未找到，则返回错误。

+   如果要分析解析的因子是一个子表达式，则返回一个子表达式，其中包含一个通过分析解析表达式获得的包装分析表达式，如果成功。如果失败，则返回错误。

因此，我们已经完成了分析器模块的审查。

在本节中，我们看到了如何分析上一节创建的解析器的结果，应用了 CSG（这是构建解释器或编译器所需的）。下一个项目将展示我们如何使用和执行分析程序。

# calc_interpreter 项目

最后，我们到达了可以实际运行我们的 `Calc` 程序的项目。

要运行它，进入 `calc_interpreter` 文件夹，并输入 `cargo run`。编译后，以下文本将出现在控制台上：

```rs
* Calc interactive interpreter *
> 
```

第一行是介绍信息，第二行是提示符。现在，我们输入以下内容作为示例：

```rs
@a >a @b b := a + 2 <b
```

在您按下 *Enter* 后，此 `Calc` 程序将被执行。声明了 `a` 变量，当执行输入语句时，控制台将出现一个问号。输入 `5` 并按 *Enter*。

程序继续声明 `b` 变量，将其赋值为 `a + 2` 表达式的值，然后打印 `b` 的值为 `7`。然后，程序结束，提示符重新出现。

因此，在屏幕上，将有以下内容：

```rs
* Calc interactive interpreter *
> @a >a @b b := a + 2 <b
? 5
7
> 
```

解释器此外还有一些特定命令，以便能够运行 `Calc` 程序。如果您输入 `v`（代表 *变量*）然后按 *Enter*，您将看到以下内容：

```rs
> v
Variables:
 a: 5
 b: 7
> 
```

此命令已转储符号表的内容，显示迄今为止声明的所有变量及其当前值。现在，您可以使用这些变量及其当前值输入其他 `Calc` 命令。

另一个解释器命令是 `c`（代表清除变量），它清空符号表。最后一个命令是 `q`（代表退出），它终止解释器。

那么 `Calc` 命令是如何执行的？如果您有一个分析程序树，以及包含变量值空间的关联符号表，这相当简单。只需将语义（即行为）应用于任何分析元素，程序就会自行运行。

此外，此项目扩展了先前的项目。`parser.rs`和`analyzer.rs`源文件是相同的；向`symbol_table.rs`文件中添加了一些行，并添加了一个其他文件——`executor.rs`。但让我们从`main.rs`文件开始。

## 了解 main.rs 文件

此文件包含两个函数，除了`main`函数外，还有`run_interpreter`和`input_command`。

`main`函数仅调用`run_interpreter`，因为这是解释器的目的。此函数具有以下结构：

```rs
fn run_interpreter() {
    eprintln!("* Calc interactive interpreter *");
    let mut variables = symbol_table::SymbolTable::new();
    loop {
        let command = input_command();
        if command.len() == 0 {
            break;
        }
 *<<process interpreter commands>>*
 *<<parse, analyze, and execute the commands>>*
    }
}
```

在打印介绍消息并创建符号表后，函数进入一个无限循环。

循环的第一个语句是调用`input_command`函数，该函数从控制台（或从文件或管道，如果标准输入被重定向）读取命令。然后，如果到达 EOF，则退出循环，整个程序也随之退出。

否则，处理解释器特定的命令，如果在输入文本中没有这样的命令，它将被解析并分析为一个`Calc`程序，然后执行分析后的程序。

以下代码块显示了解释器命令的实现方式：

```rs
match command.trim() {
    "q" => break,
    "c" => {
        variables = symbol_table::SymbolTable::new();
        eprintln!("Cleared variables.");
    }
    "v" => {
        eprintln!("Variables:");
        for v in variables.iter() {
            eprintln!(" {}: {}", v.0, v.1);
        }
    }
```

一个`q`（退出）命令简单地跳出循环。一个`c`（清除）命令用一个新的符号表替换当前的符号表。一个`v`（变量）命令遍历符号表条目，并打印每个条目的名称和当前值。

如果输入文本不是这样的单字母命令之一，它将被以下代码处理：

```rs
    trimmed_command => match parser::parse_program(&trimmed_command) {
        Ok((rest, parsed_program)) => {
            if rest.len() > 0 {
                eprintln!("Unparsed input: `{}`.", rest)
            } else {
                match analyzer::analyze_program(&mut variables, 
                 &parsed_program) {
                    Ok(analyzed_program) => {
                        executor::execute_program(&mut variables, 
                          &analyzed_program)
                    }
                    Err(err) => eprintln!("Error: {}", err),
                }
            }
        }
        Err(err) => eprintln!("Error: {:?}", err),
    },
```

`parser::parse_program`函数，如果成功，创建一个解析程序对象。在出错或输入中仍有待解析的内容的情况下，打印错误消息并丢弃命令。

否则，`analyzer::analyze_program`使用这样的解析程序创建（如果成功）一个分析程序对象。在出错的情况下，打印错误消息并丢弃命令。

最后，通过调用`executor::execute_program`执行分析程序。现在，让我们看看`symbol_table.rs`文件中有什么变化。

## 快速查看 symbol_table.rs 文件

已向`SymbolTable`类型的实现中添加了以下签名的三个函数：

```rs
pub fn get_value(&self, handle: usize) -> f64
pub fn set_value(&mut self, handle: usize, value: f64)
pub fn iter(&self) -> std::slice::Iter<(String, f64)>
```

`get_value`函数根据变量的索引获取变量的值。`set_value`函数根据索引和要分配的值设置变量的值。`iter`函数返回一个只读迭代器，用于遍历符号表中存储的变量。对于每个变量，返回一个包含名称和值的对。

接下来，我们看到实现解释器核心的模块。

## 理解 executor.rs 文件

此模块没有声明类型，因为它只使用其他模块中声明的类型。入口点是能够执行整个程序的功能，如下所示：

```rs
pub fn execute_program(variables: &mut SymbolTable, program: &AnalyzedProgram) {
    for statement in program {
        execute_statement(variables, statement);
    }
}
```

它接收一个可变引用到符号表和一个引用到已分析程序，并简单地对该程序中的任何语句调用`execute_statement`函数。

以下代码块显示了最后一个函数（这更复杂）：

```rs
fn execute_statement(variables: &mut SymbolTable, statement: &AnalyzedStatement) {
    match statement {
        AnalyzedStatement::Assignment(handle, expr) => {
            variables.set_value(*handle, evaluate_expr(variables, expr));
        }
        AnalyzedStatement::Declaration(handle) => {}
        AnalyzedStatement::InputOperation(handle) => {
            let mut text = String::new();
            eprint!("? ");
            std::io::stdin()
                .read_line(&mut text)
                .expect("Cannot read line.");
            let value = text.trim().parse::<f64>().unwrap_or(0.);
            variables.set_value(*handle, value);
        }
        AnalyzedStatement::OutputOperation(expr) => {
            println!("{}", evaluate_expr(variables, expr));
        }
    }
}
```

根据所使用的语句类型，执行不同的操作。对于赋值，它调用`evaluate_expr`函数以获取相关表达式的值，并使用`set_value`将该值赋给相关变量。

对于声明，不需要做任何事情，因为变量插入符号表及其初始化已经由分析器完成。

对于输入操作，打印一个问号作为提示，然后读取一行并将其解析为`f64`数字。如果转换失败，则使用零。然后将该值存储到符号表中作为变量的新值。

对于输出操作，评估表达式并打印结果值。以下代码显示了如何评估`Calc`表达式：

```rs
fn evaluate_expr(variables: &SymbolTable, expr: &AnalyzedExpr) -> f64 {
    let mut result = evaluate_term(variables, &expr.0);
    for term in &expr.1 {
        match term.0 {
            ExprOperator::Add => result += evaluate_term(variables, 
             &term.1),
            ExprOperator::Subtract => result -= evaluate_term(variables, 
             &term.1),
        }
    }
    result
}
```

首先，通过调用`evaluate_term`函数来评估第一个项，并将其值存储为一个临时结果。

然后，对于其他每个项，根据所使用的表达式运算符的类型，评估该项并获得的值加到或减去临时结果。

以下代码块显示了如何评估`Calc`项：

```rs
fn evaluate_term(variables: &SymbolTable, term: &AnalyzedTerm) -> f64 {
    let mut result = evaluate_factor(variables, &term.0);
    for factor in &term.1 {
        match factor.0 {
            TermOperator::Multiply => result *= evaluate_factor(
            variables, &factor.1),
            TermOperator::Divide => result /= evaluate_factor(
            variables, &factor.1),
        }
    }
    result
}
```

上述代码块显示了与之前类似的功能。它使用`evaluate_factor`函数评估当前项的所有因子，如下面的代码片段所示：

```rs
fn evaluate_factor(variables: &SymbolTable, factor: &AnalyzedFactor) -> f64 {
    match factor {
        AnalyzedFactor::Literal(value) => *value,
        AnalyzedFactor::Identifier(handle) => variables.get_value(*handle),
        AnalyzedFactor::SubExpression(expr) => evaluate_expr(variables, expr),
    }
}
```

要评估一个因子，需要考虑因子的类型。`literal`的值只是包含的值。`identifier`的值通过调用`get_value`从符号表中获得。

通过评估它包含的表达式来获得`SubExpression`的值。因此，我们已经看到了执行一个`Calc`程序所需的所有内容。

在本节中，我们看到了如何使用上下文相关分析的结果来解释`Calc`程序。这种解释可以是交互式的，通过**读取-评估-打印循环**（REPL）或通过处理用`Calc`语言编写的文件。

在下一个项目中，我们将看到如何将`Calc`程序翻译成 Rust 程序。

# calc_compiler 项目

拥有一个已分析程序（及其匹配的符号表），也容易创建一个将其翻译成另一种语言的程序。为了避免引入一种新语言，这里使用了 Rust 语言作为目标语言，但翻译到其他高级语言也不会更困难。

要运行它，进入`calc_compiler`文件夹并输入`cargo run data/sum.calc`。在编译项目后，程序将打印以下内容：

```rs
Compiled data/sum.calc to data/sum.rs
```

如果你进入`data`子文件夹，你会找到一个名为`sum.rs`的新文件，其中包含以下代码：

```rs
use std::io::Write;

#[allow(dead_code)]
fn input() -> f64 {
    let mut text = String::new();
    eprint!("? ");
    std::io::stderr().flush().unwrap();
    std::io::stdin()
        .read_line(&mut text)
        .expect("Cannot read line.");
    text.trim().parse::<f64>().unwrap_or(0.)
}

fn main() {
    let mut _a = 0.0;
    let mut _b = 0.0;
    _a = input();
    _b = input();
    println!("{}", _a + _b);
}
```

如果你喜欢，你可以使用`rustc sum.rs`命令来编译它，然后你可以运行生成的可执行文件。

对于任何编译的`Calc`程序，此文件始终相同，直到包含`fn main() {`的行。`input`例程是`Calc`运行时库。

Rust 生成的代码的其余部分对应于`Calc`语句。注意，所有变量都是可变的，并初始化为`0.0`，因此它们的类型是`f64`。变量的名称以前缀下划线开头，以防止与 Rust 关键字冲突。

实际上，这个项目还包含了前面项目中看到的解释器。如果你不带命令行参数运行项目，将启动一个交互式解释器。

让我们看看下一部分的源代码。此外，这个项目扩展了前面的项目。`parser.rs`、`analyzer.rs`和`executor.rs`源文件相同；向`symbol_table.rs`文件添加了一些行，并添加了一个其他文件——`compiler.rs`。

只向`symbol_table.rs`文件添加了一个小的函数。其签名如下所示：

```rs
pub fn get_name(&self, handle: usize) -> String
```

它允许根据索引获取标识符的名称。

但让我们从`main.rs`文件开始。

## 简要查看`main.rs`文件

`main`函数首先检查命令行参数。如果没有参数，则调用`run_interpreter`函数，与`calc_interpreter`项目中使用的相同。

相反，如果有单个参数，则会在其上调用`process_file`函数。这与`calc_analyzer`项目中使用的类似。只有两个区别。一个是插入的语句，如下面的代码片段所示：

```rs
let target_path = source_path[0..source_path.len() - CALC_SUFFIX.len()].to_string() + ".rs";
```

这生成了结果 Rust 文件的路径。另一个是替换两个结束语句，这些语句打印分析的结果，如下所示：

```rs
match std::fs::write(
    &target_path,
    compiler::translate_to_rust_program(&variables, &analyzed_program),
) {
    Ok(_) => eprintln!("Compiled {} to {}.", source_path, target_path),
    Err(err) => eprintln!("Failed to write to file {}: ({})", target_path, err),
}
```

这将翻译成 Rust 代码，获取一个多行字符串，并将该字符串写入目标文件。

因此，我们需要检查定义在`compiler.rs`源文件中的`compiler`模块。

## 理解`compiler.rs`文件

此模块不定义类型，因为它使用其他模块中定义的类型。与解析器、分析器和解释器一样，它为每种语言构造有一个函数，并通过遍历已分析程序树来执行翻译。

入口点从以下代码开始：

```rs
pub fn translate_to_rust_program(
    variables: &SymbolTable,
    analyzed_program: &AnalyzedProgram,
) -> String {
    let mut rust_program = String::new();
    rust_program += "use std::io::Write;\n";
    ...
```

这个函数，就像这个模块中的所有其他函数一样，获取对符号表和已分析程序的不可变引用。它返回一个包含 Rust 代码的`String`。首先创建一个空字符串，然后将所需的行附加到它。

此函数的最后部分如下所示：

```rs
    ...
    for statement in analyzed_program {
        rust_program += " ";
        rust_program += &translate_to_rust_statement(&variables, 
         statement);
        rust_program += ";\n";
    }
    rust_program += "}\n";
    rust_program
}
```

对于任何`Calc`语句，都会调用`translate_to_rust_statement`函数，并将它返回的 Rust 代码附加到字符串上。

将`Calc`语句翻译成 Rust 代码的函数体如下所示：

```rs
match analyzed_statement {
    AnalyzedStatement::Assignment(handle, expr) => format!(
        "_{} = {}",
        variables.get_name(*handle),
        translate_to_rust_expr(&variables, expr)
    ),
    AnalyzedStatement::Declaration(handle) => {
        format!("let mut _{} = 0.0", variables.get_name(*handle))
    }
    AnalyzedStatement::InputOperation(handle) => {
        format!("_{} = input()", variables.get_name(*handle))
    }
    AnalyzedStatement::OutputOperation(expr) => format!(
        "println!(\"{}\", {})",
        "{}",
        translate_to_rust_expr(&variables, expr)
    ),
}
```

为了翻译一个赋值，通过调用`get_name`函数从符号表中获取变量的名称，并通过调用`translate_to_rust_expr`函数获取对应表达式的代码。对于其他语句也执行同样的操作。

为了翻译一个表达式，使用了以下函数：

```rs
fn translate_to_rust_expr(variables: &SymbolTable, analyzed_expr: &AnalyzedExpr) -> String {
    let mut result = translate_to_rust_term(variables, &analyzed_expr.0);
    for term in &analyzed_expr.1 {
        match term.0 {
            ExprOperator::Add => {
                result += " + ";
                result += &translate_to_rust_term(variables, &term.1);
            }
            ExprOperator::Subtract => {
                result += " - ";
                result += &translate_to_rust_term(variables, &term.1);
            }
        }
    }
    result
}
```

这些术语通过调用`translate_to_rust_term`函数进行翻译。加法和减法使用 Rust 字符串字面量`" + "`和`" - "`进行翻译。

项的翻译与表达式的翻译非常相似，但使用项运算符和调用`translate_to_rust_factor`函数。

此函数体定义如下：

```rs
match analyzed_factor {
    AnalyzedFactor::Literal(value) => value.to_string() + "f64",
    AnalyzedFactor::Identifier(handle) => "_".to_string()
    + &variables.get_name(*handle),
    AnalyzedFactor::SubExpression(expr) => {
        "(".to_string() + &translate_to_rust_expr(variables, expr) + ")"
    }
}
```

对于字面量的翻译，它被转换成字符串，并附加`"f64"`来强制其类型。对于标识符的翻译，其名称从符号表中获取。对于子表达式的翻译，内部表达式被翻译，结果被括号包围。

在本节中，我们了解了如何在 Rust 中构建一个程序，该程序读取`Calc`程序并生成等效的 Rust 程序。这样的程序然后可以使用`rustc`命令进行编译。

# 摘要

在本章中，我们了解了一些编程语言的理论以及处理它们的算法。

尤其是我们可以看到，编程语言的语法可以用形式文法来表示。形式文法有一个有用的分类——正则语言、上下文无关语言和上下文相关语言。

编程语言属于第三类，但通常它们首先被词法分析器解析为正则语言。结果是作为上下文无关语言由解析器解析，然后分析以考虑上下文相关特性。

我们了解了处理形式语言（如编程语言或标记语言）文本的最流行技术——编译器-编译器和解析器组合器。特别是，我们看到了如何使用 Nom crate，这是一个解析器组合器库。

我们看到了许多内置的 Nom 解析器和解析器组合器，以及如何使用它们来创建我们自己的解析器，编写了许多使用 Nom 解析简单模式的 Rust 程序。我们定义了一个极其简单的编程语言的语法，我们称之为`Calc`，并使用它构建了一些小程序。我们为`Calc`构建了一个上下文无关解析器，该解析器在控制台上输出解析后的数据结构（`calc_parser`）。

我们为`Calc`构建了一个上下文相关分析器，该分析器在控制台上输出分析后的数据结构（`calc_analyzer`）。我们使用前面项目中描述的解析器和分析器为`Calc`构建了一个解释器（`calc_interpreter`）。我们还构建了一个编译器，可以将`Calc`程序翻译成等效的 Rust 程序（`calc_compiler`）。

在下一章中，我们将看到 Nom 和解析技术处理二进制数据的另一种用途。

# 问题

1.  正则语言、上下文无关语言和上下文相关语言是什么？

1.  巴科斯-诺尔范式是用来指定语言语法的什么？

1.  编译器-编译器是什么？

1.  解析器组合器是什么？

1.  为什么 Nom 在 2018 年 Rust 版本之前必须只使用宏？

1.  Nom 库中的`tuple`、`alt`和`map`函数做什么？

1.  不通过中间语言，编程语言的解释器可能有哪些阶段？

1.  编译器可能有哪些阶段？

1.  当分析变量使用时，符号表有什么作用？

1.  当解释程序时，符号表有什么作用？

# 进一步阅读

可以从[`github.com/Geal/nom`](https://github.com/Geal/nom)下载 Nom 项目。此存储库还包含一些示例。

关于形式语言及其处理软件有许多教科书。特别是，你可以在维基百科上搜索以下术语：编译器-编译器、解析器组合器、巴科斯-诺尔范式、语法驱动翻译。
