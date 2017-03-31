本文是对[kanaka/mal](https://github.com/kanaka/mal)这个项目中[指南部分](https://github.com/kanaka/mal/blob/master/process/guide.md)的中文翻译。这份指南教你如何用某种编程语言实现一个Lisp解释器，步骤清晰简明，十分适合新手学习。

This article is a Simplified Chinese translation of [kanaka/mal](https://github.com/kanaka/mal) project's the guide under [Mozilla Public License 2.0](https://github.com/kanaka/mal/blob/master/LICENSE)

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
  * set: 接受一个symbol作为key，一个mal类型对象作为value，并将它们装入`data`结构中 
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

