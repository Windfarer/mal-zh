本文的最新版本维护在 [Windfarer/mal-cn](https://github.com/kanaka/mal-cn)

本项目是对[kanaka/mal](https://github.com/kanaka/mal)这个项目中[指南部分](https://github.com/kanaka/mal/blob/master/process/guide.md)的简体中文翻译。这份指南教你如何用某种编程语言实现一个Lisp解释器。请使用原项目 [kanaka/mal](https://github.com/kanaka/mal) 进行你的开发和学习等工作。

This project is a Simplified Chinese translation of [kanaka/mal](https://github.com/kanaka/mal) project's the guide. Take a look at 
 the origin repository [kanaka/mal](https://github.com/kanaka/mal) to 
get the full version of source code.

---

#实现Lisp的过程

听说你想自己写一个Lisp解释器？欢迎入坑！

Make-A-Lisp 这个项目的目标是让你更容易地实现你自己的Lisp解释器，#tbd在攀登McCarthy之山中的Aha!时刻。当你登上山顶时，你将实现一个mal Lisp语言的解释器，它十分强大，足以自给自足，这意味着你可以运行一个使用mal语言实现的mal语言的解释器。

那么，现在可以跳坑（呃...可以开始攀登了）

- [选择一种语言](#pick-a-language)
- [开始](#getting-started)
- [通用的提示](#general-hints)
- [Make-A-Lisp的流程](#the-make-a-lisp-process-1)
  - [步骤 0: The REPL](#step-0-the-repl)
  - [步骤 1: Read and Print](#step-1-read-and-print)
  - [步骤 2: 执行](#step-2-eval)
  - [步骤 3: 环境](#step-3-environments)
  - [步骤 4: If Fn Do](#step-4-if-fn-do)
  - [步骤 5: 尾调用优化](#step-5-tail-call-optimization)
  - [步骤 6: Files, Mutation, and Evil](#step-6-files-mutation-and-evil)
  - [步骤 7: Quoting](#step-7-quoting)
  - [步骤 8: Macros](#step-8-macros)
  - [步骤 9: Try](#step-9-try)
  - [步骤 A: Metadata, Self-hosting and Interop](#step-a-metadata-self-hosting-and-interop)

<a name='pick-a-language'></a>
## 选择一种语言
可能你心中已经有了使用哪种语言的打算。从技术角度来讲，mal可以使用任何足够完备的编程语言来实现（即：图灵完备），然而，有一些语言的特性将使得任务变得**更加**简单。下面粗略按照重要程度列举了一些：

* 序列型数据结构（例如: 数组，列表，向量等）
* 关联型数据结构（例如：字典，哈希表，关联数组等）
* 函数引用（first class functions，函数指针等）
* 异常处理（try/catch, raise, throw等）
* 变量参数函数（variadic, var args, splats, apply等）
* 函数闭包
* PCRE正则表达式

另外，如下的特性将让任务变得特别容易：

* 动态类型/boxed types（特别是，可以在序列型或关联型数据结构中存储不同类型数据，并且语言本身会帮你追踪数据的类型）
* Compound data types support arbitrary runtime "hidden" data (metadata, metatables, dynamic fields attributes)

这些语言有上面提到的所有的特性：JavaScript, Ruby, Python, Lua, R, Clojure

Michael Fogus写过关于一些有趣但小众的编程语言的博客，在他列表上的很多语言都还没有mal的实现:

* [http://blog.fogus.me/2011/08/14/perlis-languages/](http://blog.fogus.me/2011/08/14/perlis-languages/)
* [http://blog.fogus.me/2011/10/18/programming-language-development-the-past-5-years/](http://blog.fogus.me/2011/10/18/programming-language-development-the-past-5-years/)

流行的语言中大部分都已经有mal的实现了。但这并不应该打消你为一门已经有mal的实现的语言写一个你自己版本的实现的积极性。另外，如果你踏上这趟旅途，我建议你避免参考任何已有的实现（也就是“作弊”），这样才能最大化你的学习效果，而不是borrow mine.从另一个角度来说，如果你的目标是尽快实现mal，你**应该**去寻找最接近于目标语言的实现，并经常去查阅参考。

如果你想看一看编程语言流行程度的列表，可以看一下这张[编程语言流行程度图表](http://langpop.corger.nl/)。

<a name='getting-started'></a>
## 开始

* 安装你所选的语言的解释器/编译器，包管理器和构建工具（如果有的话）
* 在GitHub上fork [mal](https://github.com/kanaka/mal) 这个仓库，并将你的fork仓库clone到本地:

```
git clone git@github.com:YOUR_NAME/mal.git
cd mal
```

* 创建一个你的实现的目录。例如，如果你的语言叫"quux"：

```
mkdir quux
```

* 修改顶层的Makefile，让测试用例能够测你的实现。例如，如果你的语言叫"quux"，并且以“qx”作为文件名后缀，那么对Makefile做出下面三处改动：

```
IMPLS = ... quux ...
...
quux_STEP_TO_PROG = mylang/$($(1)).qx
```

* 在你的实现的目录下增加一个"run"脚本，来读取“STEP”这个环境变量，默认值为是"stepA_mal"。确定这个run脚本已经设置了可执行的权限（否则测试将会失败，并且有permission denied的报错信息）下面分别是一种编译型语言和一种解释型语言（假设它的解释器叫quux）的"run"脚本。

```
#!/bin/bash
exec $(dirname $0)/${STEP:-stepA_mal} "${@}"
```

```
#!/bin/bash
exec quux $(dirname $0)/${STEP:-stepA_mal}.qx "${@}"
```

这个脚本让你可以像这样来测试你的实现：

```
make "test^quux^stepX"
```

如果你的实现语言是编译型语言，那么你还需要在你的实现的目录的顶层增加一个Makefile文件。用这个Makefile来定义如何构建指向quux\_STEP\_TO\_PROG macro的文件。最顶层的Makefile将在运行测试之前尝试构建这些目标。如果你的语言是脚本语言/非编译型语言，那么就不需要这个Makefile了，因为quux\_STEP\_TO\_PROG将指向一个已经存在的无需编译/构建的源代码文件。

<a name='general-hints'></a>
## 通用的提示

StackOverflow和Google是你的好伙伴。如今的多语言开发者不会记住那么多的编程语言，取而代之的是，他们会学习每种语言中特有的术语，用这些作为关键字，在搜索引擎中搜索答案。

下面是一些在多种编程语言之间进行比较的一些资料：

* [http://learnxinyminutes.com/](http://learnxinyminutes.com/)
* [http://hyperpolyglot.org/](http://hyperpolyglot.org/)
* [http://rosettacode.org/](http://rosettacode.org/)
* [http://rigaux.org/language-study/syntax-across-languages/](http://rigaux.org/language-study/syntax-across-languages/)

不要让你自己陷入特定的难题中。由于make-a-lisp过程是由一系列的步骤所构成的，实际上构建一个lisp解释器的过程更像一棵有很多分叉的树。如果你卡在了尾调用优化，或者哈希表上时，就去做一些其他的部分。当你在做其他功能的时候，会经常突然得到解决问题的灵感。我力求清晰地组织这份指南和测试，以便某些问题的可以拖延一阵再解决。

对于“可推迟/可选”的说明: 当你运行某个步骤的测试的时候，最后的一些测试可能会有"optional"（可选的任务）的标记。这表示这些测试的功能对于基础的mal实现不是必须的。这份指南中的很多步骤有个“可推迟的ren w”(deferrable)小节，它们并不是相同的意思。这些小节中包括了被标记为"optional"的测试，但也包括了对于后面步骤中所必要的功能。换句话讲，这是“实现你自己的Lisp的一场冒险”。

使用测试驱动开发，make-a-lisp过程的每个步骤都有一组与之相关的测试，在过程中也会有用来运行特定步骤所有测试的脚本。找出一个失败的测试，修复它，重新测试，直到那个步骤中所有测试都能通过为止。

## 参考代码

`process`目录包含了一些过程中每个步骤的简略伪代码的和架构图。你可以用一个文本比较工具来比较前一个步骤和你当前步骤的伪代码之间的区别。每张架构图中对于上一步所做的变更都以红色高亮表示。这还有一份[cheatsheet](http://kanaka.github.io/mal/process/cheatsheet.html)，简明扼要地描述了每个步骤的关键变化。

当你彻底被某个步骤卡住了，并且有了想放弃的念头，那么你应该“作个小弊”，参考一下已经存在的语言的实现对于当前步骤或功能的代码。因为你是来学习的，不是来考试的，所以不要有太强的负罪感。好吧，你最好还是能稍微认识到，这种行为不太好。

<a name='the-make-a-lisp-process-1'></a>
## Make-A-Lisp的流程
以下步骤的目标语言叫作"quux"，文件名后缀是"qx"

<a name='step-0-the-repl'></a>
### 步骤0: The REPL 读取求值打印循环(Read-Eval-Print Loop)
![step0_repl](/content/images/2017/03/step0_repl.png)

这个步骤基本上只是创建了你解释器的框架。

* 在`quux/`目录下创建`step0_repl.qx`文件。
* 添加4个函数`READ`,`EVAL`,`PRINT`以及`rep`(read-eval-print)。`READ`,`EVAL`和`PRINT`基本上是#tbd返回它们的第一个参数（如果你的目标语言是静态类型的话，是一个字符串），`rep`按顺序调用它们，将前一个的返回值传递给下一个。
* 如果你使用一个编译型语言（静态编译「ahead-of-time」，而不是即时编译「just-in-time」），那么在你的目录下创建一个Makefile（或者适当的项目定义文件）。

现在是时候运行你的第一部分测试了。它们将检查你的程序的输入和事实是否可以被测试所捕获。在目录顶层运行如下命令：

```
make "test^quux^step0"
```

将`step0_repl.qx`和`Makefile`添加并提交到git仓库中。

恭喜你！你已经完成了make-a-lisp的第一个步骤。

#### 可选的任务：
* 为你的解释器的REPL增加整行编辑和命令历史功能。许多语言已经提供了支持行编辑的库/模块。另外一个选项是，如果你用的语言支持用FFI (foreign function interface外来函数接口)来直接加载调用GNU readline, editline或linenoise库。将行编辑接口代码写在`readline.qx`文件中。

<a name='step-1-read-and-print'></a>
### 步骤 1: Read and Print 读取和打印
![step1_read_print](/content/images/2017/03/step1_read_print.png)

这个步骤中，你需要让你的解释器“读取”(read)用户输入的字符串，并把它解析为一种内部的树形数据结构（AST，抽象语法树），然后将这个数据结构“打印”(print)成字符串。

在非lisp类语言中，这个步骤（叫作“词法分析和语法分析”）将会是编译器/解释器中最复杂的部分之一。而在Lisp中，你想要的这种数据的结构与程序员写的代码的结构基本上是一致的（Homoiconicity，同像性）。

举个例子，如果字符串是"(+ 2 (* 3 4))"那么读取(read)函数将把它解析为这样的树形结构：

```
          List
         / |  \
        /  |   \
       /   |    \
  Sym:+  Int:2  List
               / |  \
              /  |   \
             /   |    \
         Sym:*  Int:3  Int:4
```

每个左括号和与它匹配的右括号(在lisp中叫S-表达式 "sexpr")成为树中的一个节点，而其他的一切都成为了树中的叶子节点。

如果你能找到一份你目标语言的JSON encoder/decoder的实现代码，那么你就可以通过借鉴和修改它来搞定本步骤中75%的任务。

这一节余下的部分假设你没有从JSON encoder/decoder起步，而是使用了一个Perl兼容的正则表达式(PCRE)库/模块。的确，你可以采用简单的字符串操作来实现这个功能，#tbd。`make`和`ps`(postscript)和Haskell的实现中有一些不使用正则表达式实现reader的例子。

* 复制`step0_repl.qx`并重命名为`step1_read_print.qx`。
* 新建一个`reader.qx`文件来保存与reader有关的函数。
* 如果你的目标语言有面向对象(OOP)的特性，那么下一步是在`reader.qx`中创建一个简单的有状态的Reader对象。这个对象用来保存tokens和position。Reader对象需要有两个方法：`next`和`peek`。`next`返回当前位置(position)的token，并且增大position。而`peek`只是返回当前位置的token。
* 在`reader.qx`中增加一个`read_str`函数。这个函数需要调用`tokenizer`，使用tokens来创建一个新的Reader实例。然后它调用`read_form`来处理这个Reader实例。
* 在`reader.qx`中增加一个`tokenizer`函数。这个函数接受一个字符串参数，并且将返回一个数组/列表，里面包含了所有的token(或者叫字符串，string）。下面的正则表达式(PCRE)能够匹配所有的mal的token。

```
[\s,]*(~@|[\[\]{}()'`~^@]|"(?:\\.|[^\\"])*"|;.*|[^\s\[\]{}('"`,;)]*)
```

* 对于每一个#tbd，将被解析为一个新的token
  * `[\s,]*`: 匹配任意个数的空格或逗号。它不是捕获对象，因此它会被忽略掉，不会被解析(not tokenized)
  * `~@`: 捕获两个特殊字符的组合`~@`，会被解析(tokenized)
  * `[\[\]{}()'`~^@]`: 捕获`[]{}'`~^@`这些字符中任意一个，会被解析(tokenized)
  * `"(?:\\.|[^\\"])*"`: 捕获由双引号开头，并到下一个双引号结束之间的内容，如果中间出现双引号，且双引号前面有反斜杠，则将它们也包括在捕获的内容中，直到下一个双引号。会被解析(tokenized)
  * `;.*`: 捕获由分号`;`开头的任意序列，会被解析(tokenized)
  * ```[^\s\[\]{}('"`,;)]*```: 捕获一系列由零个或更多个非特殊字符组成的序列(如，symbol, 数字, "true", "false" 以及 "nil")
* 在`reader.qx`文件中增加一个`read_form`函数，这个函数要读取Reader对象的第一个token，然后#tbd第一个字符。如果第一个字符是左括号，则使用`read_list`函数处理这个Reader对象。否则使用`read_atom`函数处理Reader对象。`read_form`的返回值是一个mal数据类型。如果你的目标语言是静态类型语言，那么你要想办法让`read_form`函数能够返回出不同的类型或者子类型。举例来说，如果你用的是一门面向对象的语言，那么你可以在最顶层中定义MalType(在types.qx中)，随后你的其他mal数据结构就可以继承它了。 MalList类型（也是继承自MalType）将由一个包含其他MalType对象的数组/列表构成。如果你用的语言是动态类型的，那么只需要返回一个包含其他MalType对象的数组/列表即可
* 在`reader.qx`中新增一个`read_list`函数。这个函数将对Reader对象反复调用`read_form`函数，直到遇到个')'字符(如果在')'之前遇到了EOF，那就说明出错了)。它把调用结果收集到一个List类型中。如果你的语言中不存在能够存储多个mal数据类型的值的顺序数据类型，那么你需要自己实现一个（在`types.qx`中实现）。注意`read_list`函数反复调用的是`read_form`，而不是`read_atom`函数。这种在`read_list`与`read_form`之间的递归定义可以能够让列表中包含列表。 
* 在`reader.qx`中新增一个`read_atom`函数。函数将会解析token的内容，并返回合适的纯（简单，非复合的）数据类型。最开始，你可以只实现数字类型（整型integer）和symbol。这能使你继续后面的一些步骤，在随后再继续实现其他的一些类型: nil, true, false和string。这些保留的mal类型:keyword, vector, hash-map和atom在步骤9之前都不需要实现（但可以在本步骤到步骤9之间的任意时间点实现）。还有，symbol类型只是一个由单独的字符串名字构成的对象（有些语言已经有symbol类型了）。
* 创建一个名为`printer.qx`的文件。这个文件包含一个叫`pr_str`的函数，它的功能与`read_str`正好相反：输入一个mal数据结构，返回出它的字符串形式。但是`pr_str`的功能很简单，它只是对于输入对象的一个switch语句：
  * symbol: 返回symbol的字符串名字
  * number: 将数字作为一个字符串返回
  * list: 对于列表中的每一个元素调用`pr_str`，然后将结果使用空格分隔，把它们拼接在一起，最后在最外面加上括号
* 修改`step1_read_print.qx`中的`READ`函数，让它调用`reader.read_str`，并且`PRINT`函数调用`printer.pr_str`。`EVAL`函数继续直接返回输入的东西，但是如今返回值的类型应当是mal数据类型了。

现在你#tbd可以开始测试你的代码了。你可以手工测试一些简单的输入：

* `123` -> `123`
* `123` -> `123`
* `abc` -> `abc`
* `abc` -> `abc`
* `(123 456)` -> `(123 456)`
* `( 123 456 789 )` -> `(123 456 789)`
* `( + 2 (* 3 4) )` -> `(+ 2 (* 3 4))`

为了验证你的代码不只是去掉了多余的空格（并且没有失败），你可以#tbd你的`reader.qx`中的函数。

当你已经通过了上述的简单手工测试，就可以运行步骤1的测试了。到最顶层的目录，执行下面的命令：

```
make "test^quux^step1"
```
修复所有与symbol, number和list有关的失败测试。

你现在已经完成了最困难的步骤之一。#tbd 剩下的步骤可能更简单，并且每个步骤会逐渐让你有更多的收获。


#### 可推迟的任务:
* 为你的reader和printer函数增加其他的基础类型的支持: string, nil, true, and false. 这些类型在步骤4的时候就是必需的了。在读取一个字符串之后，要进行下列的转换：一个反斜杠后面跟着双引号的时候`\"`，需要把它们翻译为一个普通的双引号`"`，反斜杠后跟着n的时候`\n`需要翻译为换行，一个反斜杠后面跟着另一个双引号的时候`\\`，需要把它们翻译为一个单引号`\`。为了能正确的打印字符串（#tbd），`pr_str`函数需要另一个叫作`print_readably`的参数。当这个参数为true的时候，双引号、换行符和反斜杠会被翻译为它们被打印出来的表现形式（与reader的逻辑正好相反）。主程序中的`PRINT`函数应该在调用`pr_str`时将print_readably设置为true。
* 为reader函数增加更多的错误检查，确保括号都能够正确匹配。在主循环中捕获并打印这些错误信息。如果你的语言中没有try/catch风格的冒泡式异常处理功能，那么你需要在代码中加上一个显式的异常处理，并且跳过错误，不要让程序崩溃掉。
* 为reader加上macros支持。这能够在读取阶段时将某些形式转换为其他形式。在`tests/step1_read_print.mal`中可以找到需要支持哪种macros的形式（它们只是对token流的简单转换）
* 支持其他几种mal数据类型: keyword, vector, hash-map.
  * keyword(关键字): keyword是由冒号开头的token。keyword只能存储为有特殊unicode前缀的字符串，像0x29E (或字符 0xff/127, 如果目标语言没有很好的unicode支持的话) printer会把带这个前缀的字符串转换回keyword表示。这能够让在大多数语言中使用keyword作为哈希表的key变得很容易。你也可以将keyword存储为一种唯一数据类型，但你要确定它们可以作为哈希表的key使用（#tbd）。
  * vector(向量): vector可以被实现为#tbd。你可以通过在开头和结尾的token上加上参数，从而做到使用同一个reader函数操作list和vector的功能。
  * hash-map(哈希表): 哈希表是一种关系型数据结构，它将字符串映射到其他mal类型的值上。如果你将keyword实现为带前缀的的字符串，那么你只需要一种原生的关系数据结构，只要它支持以字符串作为key就可以了。Clojure支持把任何值作为哈希表的key，但在mal的基础功能中只需要支持把字符串作为key即可。因为将hash-map表示为key和value的交替序列，你可能可以用读取list和vector的reader函数来处理hash-map，只需要用参数来标示它的开头和结尾token即可。奇数位置的token作为key，而偶数位置的token作为value。
* 为你的reader增加对注释的支持。tokenizer应该忽略由";"开头的token。你的`reader_str`函数需要正确的处理tokenizer不返回任何值的情况。最简单的办法是返回`nil`这个mal类型的值。一个更加简明的（在这种情况下不打印nil）方式是抛出一个特殊的异常，使主循环直接在循环的开头跳过循环，从而不调用rep。

### 步骤 2: Eval 求值
![step2_eval](/content/images/2017/03/step2_eval.png)

在步骤1中，你的mal解释器基本上只有验证输入然后去除输出结果中多余空格的功能。在本步骤中，你将会为你的解释器增加evaluator (EVAL)的功能，从而把它改成一个简单的计算器。

比较步骤1和步骤2的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step1_read_print.txt ../process/step2_eval.txt
```

* 将`step1_read_print.qx`复制为`step2_eval.qx`
* 定义一个简单的初始化REPL环境。这个环境是一个关联数据结构，将symbol (或symbol names) 映射为数学运算函数。例如，在python中，这个环境应该看起来是这个样子的：

```
repl_env = {'+': lambda a,b: a+b,
            '-': lambda a,b: a-b,
            '*': lambda a,b: a*b,
            '/': lambda a,b: int(a/b)}
```

* 修改`rep`函数，将这个REPL环境作为调用`EVAL`函数时的第二个参数。
* 创建一个新函数`eval_ast`，它将接受`ast`(mal数据类型)和一个关系数据结构（上文中的环境）`eval_ast`对于`ast`类型进行下列情况匹配并做相应处理：
  * symbol: 在环境结构中查找symbol，返回对应的value，或在value不存在时报错
  * list: 返回对于list中的每个元素`EVAL`调用得到的结果所组成的list
  * 否则直接返回原`ast`值
* 修改`EVAL`函数，检查它的第一个参数`ast`是不是一个list。
  * `ast`不是list: 返回对它调用`eval_ast`得到的结果
  * `ast`是一个空的list: 原封不动地返回ast
  * `ast`是一个list: 调用`eval_ast`得到一个新的执行过的list #tbd 

如果你的目标语言不支持可变长度参数（例如，variadic, vararg, splats, apply），那么你需要将整个参数的列表作为一个单独的参数，然后在每个mal函数中再将它切分为一个个独立的值。这样做虽然比较闹心，但还是可以凑合用的。

对于一个list#tbd

用这些表达式来进行测试：

* `(+ 2 3)` -> `5`
* `(+ 2 (* 3 4))` -> `14`

你最可能遇到的挑战是，如何使用一个参数列表#tbd

现在，回到顶层目录，执行步骤2的测试，并修复错误。

```
make "test^quux^step2"
```

现在你有了一个简单的前缀表达式计算器了。

可推迟的任务：

* `eval_ast`应该#tbd vector和hash-map。在`eval_ast`函数中加入下列情况：
  * 如果`ast`是一个vector: 返回对于vector中的每个元素`EVAL`调用得到的结果所组成的vector
  * 如果`ast`是一个hash-map: 返回一个新的hash-map，它的key是从原hash-map中来的key，value是对于原hash中的value调用`EVAL`得到的结果。

### 步骤 3: Environments 环境
![step3_env](/content/images/2017/03/step3_env.png)

在步骤2中我们已经实现了REPL环境(`repl_env`)，在这个环境中可以存储和查找基本的算数运算函数。在本步骤中，你将会为解释器增加创建新环境(`let*`)和修改已存在的环境(`def!`)的功能。

Lisp的环境是一个关系数据结构，它将symbol(即key)映射到value上。但是Lisp环境还有一个很重要的额外功能：它们可以引用refer另一个环境(外层的环境)。在环境中进行查找时，如果当前环境中没有要找的symbol，那么将在外层环境中继续查找，持续进行这个过程，直到找到symbol，或者外层的环境是`nil`(在整个链中的最外层)

比较步骤2和步骤3的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step2_eval.txt ../process/step3_env.txt
```

* 将`step2_eval.qx`复制为`step3_env.qx`
* 创建`env.qx`，在里面写和环境有关的定义
* 定义一个`Env`对象，它有一个关系型数据结构的属性`data`，并且在实例化的时候需要一个`outer`参数
* 为Env对象定义下列三个方法：
  * set: 接受一个symbol作为key，一个
mal类型对象作为value，并将它们装入`data`结构中 
  * find: 接受一个symbol的key参数，如果当前环境中找到了这个key，那么返回环境。如果没有找到，并且外层环境不是`nil`，那么在外层环境中（递归）调用find
  * get: 接受一个symbol的key参数，并且用`find`方法来找到这个key对应的环境，并且返回匹配到的value。如果没有在外层环境的链中没有找到这个key，则抛出一个"not found"（未找到）错误
* 更新`step3_env.qx`，使用新的Env类型来创建repl_env(它的outer参数设置为nil)，并且使用`set`方法将算数运算函数加入到环境中
* 修改`eval_ast`，对env参数调用`get`方法
* 修改`EVAL`函数的apply的部分，对于list的第一个元素#tbd:
  * symbol "def!": 调用当前环境（`EVAL`的第二个，名为`env`的参数）的set方法，使用未求值的第一个参数（list的第二个元素）作为symbol key，并且将已求值的第二个参数作为value
  * symbol "let*": 以当前环境作为outer，创建一个新的环境，并将第一个参数作为列表#tbd。
  * 否则: 对于list调用`eval_ast`，并将第一个元素应用到后面的#tbd

`def!`和`let*`是Lisp中的特例（或叫“特殊atom原子”），意思是它们是语言级别的特性，比list剩下的元素（参数）更特别，#tbd（太长了= =）

尝试一些简单的环境测试：

* `(def! a 6)` `->` `6`
* `a` `->` `6`
* `(def! b (+ a 2))` `->` `8`
* `(+ a b)` `->` `14`
* `(let* (c 2) c)` `->` `2`

回到目录的最顶层，运行步骤3的测试，并修复错误。

```
make "test^quux^step3"
```

### 步骤 4: If Fn Do
![step4_if_fn_do](/content/images/2017/04/step4_if_fn_do.png)

在步骤3中，你为解释器增加了环境，和用来操作环境的特殊形式。在本步骤中，你将为REPL的默认环境增加三种新的特殊形式(`if`, `fn*`和`do`)以及几种核心函数。新的结构如下：

`fn*`特殊形式是用户创建自定义函数的方式。在一些Lisp语言中，这个特殊的形式叫"lambda"。

比较步骤3和步骤4的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step3_env.txt ../process/step4_if_fn_do.txt
```

* 将`step3_env.qx`复制为`step4_if_fn_do.qx`。
* 如果你还没有实现reader和printer对nil, true和false的支持，那么你需要在本步骤中实现。
* 修改环境的构建器/初始化器，让它接受两个新的参数: `binds`和`exprs`。将(`set`)#tbd
* 在`printer.px`中增加对打印函数值的支持。像`#`之类的字符串就足够了#tbd
* 为`EVAL`实现下列特殊形式：
  * `do`: 使用`eval_ast`对list的所有元素进行求值，并返回最终被求值的元素。
  * `if`: 对第一个参数进行求值（即第二个元素），如果结果不是`nil`或`false`，则求值第二个参数（即第三个元素），并返回它的结果。否则，求值第三个参数（第四个元素）并返回执行结果。如果条件是false并且没有第三个参数，则返回`nil`。
  * `fn*`: 返回一个新的函数闭包。闭包的body中包含如下内容：    
    * 创建一个新的环境，以env(#tbd)作为`outer`参数，第一个参数（即外层作用域的`ast`的第二个list元素）作为`binds`参数，闭包的参数作为`exprs`的参数。
    * 对于第二个参数（即外层作用域的`ast`的第三个list元素）调用`EVAL`，使用新的环境。将结果作为闭包的返回值。

如果你的目标语言不支持闭包，那么你需要使用某种在关闭时可以保存值的结构或者对象保存如下的东西：`ast`list的第一个和第二个参数（函数参数列表和函数体），以及当前环境`env`。在这种情况下，你的原生函数需要用相同的方式被包裹。你可能也需要在`EVAL`的apply部分有一个用来调用对象/结构的函数/方法。

测试你已经实现的基础部分：

* `(fn* [a] a)` -> `#<function>`
* `( (fn* [a] a) 7)` -> `7`
* `( (fn* [a] (+ a 1)) 10)` -> `11`
* `( (fn* [a b] (+ a b)) 2 3`) -> `5`

* 增加一个新的文件`core.qx`，并定义一个叫`ns`(namespace)的关系数据结构，将symbol映射为函数。将算数运算函数定义放在这个函数里。
* 修改`step4_if_fn_do.qx`，让它读取`core.ns`结构，并将每个symbol/函数加入(`set`)到REPL环境中(`repl_env`)。
* 将下列函数加入到`core.ns`中：
  * `prn`: 对于第一个参数调用`pr_str`，`print_readably`参数设置为true，将结果打印到屏幕，并返回`nil`。注意，完整版本的`prn`函数在下面“可推迟的任务”一节中。
  * `list`: 接受参数并将它们返回为一个list。
  * `list?`: 如果参数是一个list，返回true，否则返回false。
  * `empty?`: 将第一个参数当作一个列表处理，如果这个list是空的，则返回true，如果里面有元素，返回false。
  * `count`: 将第一个参数当作一个列表处理，并返回列表包括元素的个数。
  * `=`: 对比前两个参数，如果它们是相同的类型并且值也相同，那么返回true。在比较两个列表的时候，两个列表中的每两个对应元素都要进行比较，如果它们都相同，返回true，否则返回false。
  * `<`, `<=`, `>`和`>=`: 将前两个参数作为数字，并且进行数学比较，返回true或false。

回到目录顶层，运行步骤4的测试。步骤4有大量的测试用例，但是所有的不涉及到字符串的非可选测试，都需要通过。

```
make "test^quux^step4"
```
你的mal实现已经开始像一门真正的语言了。你有了流程控制，判断，带词法作用域的用户定义函数，副作用（如果你实现了字符串函数）等。但是我们的小解释器还没有达到Lisp-ness的程度。后续的步骤将使你的小玩具改造成为一个全功能的语言。

#### 可推迟的步骤:
* 实现Clojure风格的可变函数参数。修改环境的构造器/初始化器，在`binds`列表中遇到"&"符号的时候，在这个符号后面的元素#tbd
* 定义一个`not`函数，给mal自己用。在`step4_if_fn_do.qx`中以"(def! not (fn* (a) (if a false true)))"为参数调用`rep`函数。
* 在`core.qx`中实现字符串函数。你需要为reader和printer实现字符串支持（步骤1中的可推迟的步骤）。每一个字符串函数接受若干个mal类型的值，打印它们(`pr_str`)并将它们组装成一个新的字符串。
  * `pr-str`：对于每个参数调用`pr_str`，将`print_readably`参数设置为true，用" "(空格字符)将结果连接，将字符串结果返回。
  * `str`: 对于每个参数调用`pr_str`，将`print_readably`参数设置为false，用""将结果组装在一起，并返回新字符串。
  * `prn`: 对于每个参数调用`pr_str`，将`print_readably`参数设置为true，将结果用" "连接，把新字符串打印到屏幕上，并返回`nil`。
  * `println`: 对于每个参数调用`pr_str`，将`print_readably`参数设置为false，将结果用" "连接，把新字符串打印到屏幕上，并返回`nil`。

### 步骤 5: 尾调用优化
![step5_tco](/content/images/2017/04/step5_tco.png)

在步骤4中，你增加了特殊的形式`do`, `if`和`fn*`并且定义了一些核心函数。在本步骤中你将实现一个Lisp的特性，叫作尾调用优化(TCO)。也叫“尾递归”或“尾调用”。

你在`EVAL`中已经定义的一些形式最终会回调进入`EVAL`中。对于这些以调用`EVAL`作为在返回之前做的最后一件事情（尾调用）的形式，你将直接回到过程的开头循环执行，而不是再一次调用它。这个做法的优点是能够避免在调用栈里增加更多的栈帧。这在Lisp语言中特别重要，因为它们倾向于使用用递归代替迭代#tbd。（虽然有些Lisp的方言中有迭代，例如Common Lisp）有了尾调用优化，递归可以有像迭代一样的调用栈的效率。

比较步骤4和步骤5的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step4_if_fn_do.txt ../process/step5_tco.txt
```

* 将`step4_if_fn_do.qx`复制为`step5_tco.qx`
* 为EVAL的所有代码增加一个循环（如while true）
* 修改下列的形式，增加对尾调用递归支持：
  * `let*`: 移除在`ast`第二个参数（即第三个列表元素）最后的`EVAL`调用，将`env`（换句话说，即作为`EVAL`第二个参数传入的局部变量）设置为`ast`的第二个参数。回到循环开头继续执行（不返回）。
  * `do`: 修改`eval_ast`调用，让它求值所有的参数，除了最后一个参数（#tbd）将`ast`设置为`ast`最后一个元素。回到循环开头继续执行（`env`保持不变）
  * `if`: 继续求值条件，但不是求值true或false分支，而是将`ast`设置为被选中分支的未求值的量。回到循环开头继续执行（`env`保持不变）
* 特殊形式`fn*`的返回值现在要变成一个对象/结构，它应该带有一个允调用`EVAL`的默认情况的树形，来对mal函数进行尾调用优化。这些参数是：
  * `ast`: `ast`的第二个参数（即第三个列表元素），相当于函数的body。
  * `params`: `ast`的第一个参数（第二个列表元素），相当于函数的参数名。
  * `env`: 当前`EVAL`函数的`env`参数的值。
  * `fn`: 原始的函数值（换句话说，就是步骤4中`fn*`所返回的东西）。注意这是一个可推迟的任务，在步骤9的时候`map`和`apply`才需要用到它。如果在后面的步骤6中你选择不推迟 atoms/`swap!`的实现的话，那么也需要先实现它。
* 默认的"apply"/invoke #TBD

执行一下上一步骤中的手工测试，确保你没有因为加入了尾调用优化而破坏了什么东西。

现在回到目录顶层，运行步骤5的测试。
```
make "test^quux^step5"
```
看一下步骤5的测试文件`tests/step5_tco.mal`。函数`sum-to`不能被尾调用优化，因为它在递归调用之后又做了一些事情（`sum-to`调用了它自身，之后执行了相加的操作）Lisp用户说`sum-to`并没有在尾部调用。函数`sum2`在尾部调用了自己。换句话说，对`sum2`的递归调用是`sum2`最后一步做的事情。对于一个非常大的值调用`sum-to`将在大多数目标语言中导致栈溢出异常。（某些语言使用了非常特殊的技巧来避免栈溢出）

祝贺你，你的mal实现已经有了大多数主流语言所缺少的（尾调用优化）特性。

步骤6: Files, Mutation, and Evil
![step6_file](/content/images/2017/04/step6_file.png)

在步骤5中，你为解释器加入了尾调用优化。在本步骤中你将加入一些字符串和文件操作，为你的实现增加一些evil，呃eval（#tbd）。只要你的语言支持函数闭包，那么本步骤将非常容器。然而，为了完成本步骤，你必须实现字符串类型的支持，所以如果你之前如果推迟了任务还没完成，你需要回去先把那个搞定。

比较步骤5和步骤6的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step5_tco.txt ../process/step6_file.txt
```

* 将`step5_tco.qx`复制为`step6_file.qx`
* 为核心命名空间增加两个新的字符串函数:
  * `read-string`: 这个函数将reader中的`read_str`函数暴露了出来。如果你的mal字符串类型与你目标语言不一样（例如静态类型语言），那么你的`read-string`函数需要将原始字符串通过调用`read_str`函数，而从mal字符串类型中解包出来。
  * `slurp`: 这个函数接受一个文件名（字符串）并并且将文件的内容作为字符串返回。和上面那个函数一样，如果你的mal字符串类型封装了目标语言的字符串，那么你需要将字符串参数解码来得到原始的文件名字符吃，并将结果编码(封装)回mal字符串类型。
* 在你的主程序中，为你的REPL环境增加一个新的符号"eval"。这个符号对应的值是接受一个参数`ast`的函数。闭包，调用你的`EVAL`函数，以`ast`作为第一个参数，EVAL环境作为第二个参数（#TBD）。将调用`EVAL`的结果返回。这个简单且强大新功能允许你将mal数据作为mal程序对待。例如，你现在可以这样做：

```
(def! mal-prog (list + 1 2))
(eval mal-prog)
```

* 使用mal语言自身，定义一个`load-file`函数，在你的主程序里调用`rep`函数，参数为"(def! load-file (fn* (f) (eval (read-string (str "(do " (slurp f) ")")))))"

测试一下`load-file`:

* `(load-file "../tests/incA.mal")` -> `9`
* `(inc4 3)` -> `7`

`load-file`函数做了如下的事情：

* 调用`slurp`来通过文件名读取一个文件。将文件内容用"(do ...)"进行包裹，这样整个文件就可以作为一个单程序的AST（抽象语法树）。
* 以`slurp`的返回值作为参数调用`read-string`函数。它使用reader读取/转换文件的内容，使之成为mal数据/AST.
* 使用`eval`(在REPL环境中的那个)函数处理`read-string`函数返回的AST，“运行”它。

除了增加文件和求值的支持以外，在本步骤中我们也要增加原子数据类型。原子是mal用来表示状态的方式，这个灵感来自于[Clojure的原子](http://clojure.org/state)。原子保存了对一个任意类型mal值的引用。它支持读取一个mal值，修改它的引用，将它指向另一个mal值。注意这是唯一一种可变类型（但是它所引用的mal值仍是不可变的，在步骤7中有关于不可变特性的进一步解释）你需要在核心命名空间中增加5个函数来支持原子。

* `atom`: 输入一个mal值，并返回一个新的指向这个值的原子。
* `atom?`: 判断输入的参数是不是原子，如果是，返回true。
* `deref`: 输入一个原子作为参数，返回这个原子所引用的值。
* `reset!`: 输入一个原子以及一个mal值，修改原子，让它指向这个mal值，并返回这个mal值。
* `swap!`: 输入一个原子，一个函数，以及零个或多个函数参数。原子的值被修改为#tbd返回新的原子的值。#tbd

你可以增加一个reader macro`@`，它相当于一个`deref`的简略形式，因此`@a`相当于`(deref a)`。为了达到这个目的，修改`read_form`函数的条件判断，增加一个规则，处理`@`token: 如果token是`@`(#tbd)，那么返回一个包括`deref`符号以及读取下一个形式(`read_form`)结果的新列表。

现在回到目录顶层，运行步骤6的测试。可选测试包括了对注释，向量，哈希表，和`@` reader macro的支持:

```
make "test^quux^step6"
```
恭喜你，你现在实现了一个完善的脚本语言，它可以运行其他的mal程序。`slurp`函数把一个文件当作字符串读进来，`read-string`函数调用mal reader将字符串转换为数据，然后`eval`函数读取数据，并把它当作一个普通的mal程序一样进行求值。然而，我们需要注意，`eval`函数不是只能用来运行外部程序的。因为mal程序就是普通的mal数据结构，你可以在调用`eval`求值之前，动态生成或者操作这些数据结构。数据和程序之间的同形（形状相同），我们称之为同像性(homoiconicity)。Lisp语言们的这种同像性将它们与其他大多数语言区分开来。

你的mal实现现在已经非常强大了，但在`core.qx`中的这组可用的函数还相当有限。你将在步骤9和步骤A中增加许多函数进去。#tbd

可推迟的任务：
* 增加通过命令行运行其他mal程序的能力。在进入REPL循环之前，检查你的mal实现在被调用的时候有没有带参数，如果有参数的话，将第一个参数视为文件名，使用`rep`调用`load-file`将文件导入并执行，最后退出/终止执行。
* 将剩下的命令行参数传入REPL环境，让通过`load-file`函数执行的程序能够访问调用它们的环境。为你的REPL环境加入一个"*ARGV*"(符号)。它是剩下的命令行参数作为一个mal列表的值。#tbd

### 步骤 7: Quoting

![step7_quote](/content/images/2017/04/step7_quote.png)

在步骤7中，你将为解释器加上`quote`和`quasiquote`这两个特殊形式，并且加入`cons`和`concat`这两个核心函数的支持。

特殊形式`quote`告诉求值器(`EVAL`)不要对参数进行求值。一开始，看起来这个功能没啥卵用，但能证明它有用的一个例子是，让mal程序能够有引用一个符号的自身，而不是它经过求值后的结果。比如列表。例如，考虑下列情况：

* `(prn abc)`: 这段程序将在当前求值环境中寻找符号`abc`。并打印到屏幕上。如果`abc`没有被定义过的话，就会报错。
* `(prn (quote abc))`: 这段程序会打印"abc"（打印符号本身）。它不会去管在当前环境中`abc`是否已被定义。
* `(prn (1 2 3))`: 这段程序会报错，因为`1`不是函数，不能被应用于参数`(2 3)`上。
* `(prn (quote (1 2 3)))`: 这段程序将打印"(1 2 3)"。
* `(def! l (quote (1 2 3)))`: list quoting允许我们在代码中直接定义list(#tbd).另一个做这件事的方法是使用list函数`(def! l (list 1 2 3))`。

第二种特殊形式是`quasiquote`。它允许一个quoted list能够有一些临时unquoted的元素(正常求值)。两种#tbd`unquote`和`splice-unquote`。#tbd最好的例子:

* `(def! lst (quote (2 3)))` -> `(2 3)`
* `(quasiquote (1 (unquote lst)))` -> `(1 (2 3))`
* `(quasiquote (1 (splice-unquote lst)))` -> `(1 2 3)`

`unquote`形式对它的参数进行#tbd，并将求值结果放入quasiquoted list。形式`splice-unquote`也将它的参数#tbd，但是被求值的值必须是列表，因为在后面它们会被切分(splice)到quasiquoted list中。在将它于macro一起使用的时候，quasiquote形式的真实力量才会展现出来(在下一步骤中)。

比较步骤6和步骤7的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step6_file.txt ../process/step7_quote.txt
```

* 将`step6_file.qx`复制为`step7_quote.qx`
* 在实现这些quoting form时，你需要先在核心命名空间里实现一些支持函数:
  * `cons`: 这个函数将它的第一个参数连接到它的第二个参数(一个列表)前面，返回一个新列表。
  * `concat`: 这个函数接受零个或多个列表作为参数，并且返回由这些列表的所有参数组成的一个新列表。

关于不变可性: 注意cons和concat都没有修改它们的原始列表参数。所有对于它们的引用（换句话说，在其他列表中，它们可能作为其中的元素）将还指向原有的未变更的值。就像Clojure一样，mal是一种使用不可变数据结构的语言。我建议你去学习一下Clojure语言中实现的不可变性的能力和重要性，mal借用了它的大部分语法和特性。

* 添加`quote`特殊形式，这个特殊形式返回它的参数（`ast`的第二个列表元素）

* 添加`quasiquote`特殊形式。实现实现一个helper函数`is_pair`，它在参数是一个非空列表的时候返回true。然后定义`quasiquote`函数。#tbd`ast`参数的第一个参数（第二个列表元素），随后`ast`被设置为结果，并且回到循环的开头继续执行（TCO，尾调用优化）。`quasiquote`函数输入`ast`参数后，进行如下的条件判断：
  1. 如果`is_pair`对于`ast`的判断结果是false: 返回一个新列表，里面包括了一个名为"quote"的符号，以及`ast`。
  2. 否则，如果`ast`的第一个元素是符号"unquote": 返回`ast`的第二个元素。
  3. 如果`is_pair`对于`ast`的判断结果是true，并且`ast`的第一个元素的第一个元素(即`ast[0][0]`)是名为"splice-unquote"的符号：返回一个新的列表，其中包含：名为"concat"的符号，`ast`的第一个元素的的第二个元素(即`ast[0][1]`)，以及以`ast`的第二个元素到最后一个元素为参数调用`quasiquote`的结果。
  4. 否则: 返回一个新的列表，包括：名为"cons"的符号，以`ast`的第一个参数(即`ast[0]`)，和以`ast`的第二个元素到最后一个元素为参数调用`quasiquote`的结果。

返回目录顶层，执行步骤7的测试。

```
make "test^quux^step7"
```

Quoting是mal中许多无聊的函数中的一个，但别因此而灰心。你的mal实现已接近完工了，而quoting为接下来的接近收工的步骤: macro，做好了准备。

#### 可推迟的任务

* quoting 形式的全名相当罗嗦。大多数Lisp语言有一个简写的语法，mal也不例外。这些简写语法被成为reader macros因为它们使我们能够在reader阶段中操作mal代码。在eval阶段中被执行的macro只是叫作macro，我们将在下一节中介绍。扩展reader的`read_form`函数的条件判断，增加下列情况:
  * token是"'" (单引号): 返回一个新列表，包含符号"quote"，以及对下一个form读取的结果(`read_form`)
  * token是"\`" (反引号): 返回一个新列表，包含符号"quasiquote"，以及对下一个form读取的结果(`read_form`)
  * token是"~" (波浪号): 返回一个新列表，包含符号"unquote"，以及对下一个form读取的结果(`read_form`)
  * token是"~@" (波浪号和at符号): 返回一个新列表，包含符号"splice-unquote"，以及对下一个form读取的结果(`read_form`)
* 增加对vector的quoting的支持。`is_pair`函数在参数是非空列表或vector时应该返回true。`cons`应该也能接受vector作为第二个参数。返回值是list regardless。`concat`应该支持list，vector，或它们两者进行连接，结果永远是list。


### Step 8: Macros 宏
![step8_macros](/content/images/2017/04/step8_macros.png)
