
本项目是对 [mal](https://github.com/kanaka/mal) 这个项目中 [指南部分](https://github.com/kanaka/mal/blob/master/process/guide.md) 的简体中文翻译。这份指南教你如何用某种编程语言实现一个 Lisp 解释器。请使用原项目 [mal](https://github.com/kanaka/mal) 进行开发和学习。译文的最新版本维护在仓库 [Windfarer/mal-zh](https://github.com/Windfarer/mal-zh)

本译文按照 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 进行授权。

This project is a Simplified Chinese translation of [kanaka/mal](https://github.com/kanaka/mal) project's guide. Take a look at 
 the origin repository to 
get the full version of source code.

---

# 实现Lisp解释器的步骤

听说你想自己写一个 Lisp 解释器？欢迎入坑！

Make-A-Lisp 这个项目的目标是让你更容易地实现你自己的 Lisp 解释器，使你在攀登 McCarthy (译注: 即[约翰·麦卡锡](https://zh.wikipedia.org/zh-hans/%E7%BA%A6%E7%BF%B0%C2%B7%E9%BA%A6%E5%8D%A1%E9%94%A1), Lisp语言之父) 之山的过程中，避开那些需要顿悟的难题("Aha!" moment)。当你登上山顶时，你将实现一个 mal Lisp 语言的解释器，它十分强大，可以做到自足执行(self-hosting)，这意味着你可以运行一个使用 mal 语言实现的 mal 语言的解释器。

那么，现在可以跳坑（呃... 可以开始攀登）了

- [选择一种语言](#pick-a-language)
- [开始](#getting-started)
- [通用的提示](#general-hints)
- [Make-A-Lisp 的流程](#the-make-a-lisp-process-1)
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
可能你心中已经有了使用哪种语言的打算。从技术角度来讲，mal 可以使用任何足够完备的编程语言来实现(即图灵完备)，然而，有一些语言的特性将使得任务变得 **更加** 简单。下面粗略按照重要程度列举了一些：

* 序列型数据结构（例如: 数组，列表，向量等）
* 关联型数据结构（例如：字典，哈希表，关联数组等）
* 函数引用（头等函数（first-class function)，函数指针等）
* 异常处理（try/catch, raise, throw 等）
* 变量参数函数（variadic, var args, splats, apply 等）
* 函数闭包
* PCRE 正则表达式

另外，如下的特性将让任务变得特别容易：

* 动态类型/boxed types（特别是，可以在序列型或关联型数据结构中存储不同类型数据，并且语言本身会帮你追踪数据的类型）
* 复合数据类型支持任意运行时“隐藏的”数据 (元数据，元表，动态字段属性) 

下列语言有上面提到的所有的特性：JavaScript, Ruby, Python, Lua, R, Clojure

Michael Fogus 写过关于一些有趣但小众的编程语言的博客，他的列表上的很多语言都还没有 mal 的实现:

* [http://blog.fogus.me/2011/08/14/perlis-languages/](http://blog.fogus.me/2011/08/14/perlis-languages/)
* [http://blog.fogus.me/2011/10/18/programming-language-development-the-past-5-years/](http://blog.fogus.me/2011/10/18/programming-language-development-the-past-5-years/)

流行的语言中大部分都已经有 mal 的实现了。但这并不应该打消你为一门已经有 mal 的实现的语言写一个你自己版本的实现的积极性。另外，如果你踏上这趟旅程，我建议你不要参考任何已有的实现（也就是“作弊”），这样才能最大化你的学习效果，而不是从我这里借鉴一个。从另一个角度来说，如果你的目标是尽快实现 mal，你 **应该** 去寻找最接近于目标语言的实现，并经常去查阅参考。

如果你想看一看编程语言流行程度的列表，可以看一下这张[编程语言流行程度图表](http://langpop.corger.nl/)。

<a name='getting-started'></a>
## 开始

* 安装你所选的语言的解释器/编译器，包管理器和构建工具（如果有的话）
* 在 GitHub 上 fork [mal](https://github.com/kanaka/mal) 这个仓库，并将你 fork 的仓库 clone 到本地:

```
git clone git@github.com:YOUR_NAME/mal.git
cd mal
```

* 创建一个你的实现的目录。例如，如果你的语言叫 "quux"：

```
mkdir quux
```

* 修改顶层的 Makefile，让测试用例能够测你的实现。例如，如果你的语言叫 "quux"，并且以 “qx” 作为文件名后缀，那么对 Makefile 做出下面三处改动：

```
IMPLS = ... quux ...
...
quux_STEP_TO_PROG = mylang/$($(1)).qx
```

* 在你的实现的目录下增加一个 "run" 脚本，来读取 "STEP" 这个环境变量，默认值为是 "stepA_mal"。确定这个 run 脚本已经设置了可执行的权限（否则测试将会失败，并且有 permission denied 的报错信息）下面分别是一种编译型语言和一种解释型语言（假设它的解释器叫 quux）的 "run" 脚本。

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

如果你的实现语言是编译型语言，那么你还需要在你的实现的目录的顶层增加一个 Makefile 文件。用这个 Makefile 来定义如何构建指向 quux\_STEP\_TO\_PROG 的文件。最顶层的 Makefile 将在运行测试之前尝试构建这些目标。如果你的语言是脚本语言/非编译型语言，那么就不需要这个 Makefile 了，因为 quux\_STEP\_TO\_PROG 将指向一个已经存在的无需编译/构建的源代码文件。

<a name='general-hints'></a>
## 通用的提示

StackOverflow 和 Google 是你的好伙伴。如今的多语言开发者不会记住那么多的编程语言，取而代之的是，他们会学习每种语言中特有的术语，用它们作为关键字，在搜索引擎中搜索答案。

下面是一些在多种编程语言之间进行比较的一些资料：

* [http://learnxinyminutes.com/](http://learnxinyminutes.com/)
* [http://hyperpolyglot.org/](http://hyperpolyglot.org/)
* [http://rosettacode.org/](http://rosettacode.org/)
* [http://rigaux.org/language-study/syntax-across-languages/](http://rigaux.org/language-study/syntax-across-languages/)

不要让你自己陷入特定的难题中。由于 make-a-lisp 过程是由一系列的步骤所构成的，实际上构建一个 Lisp 解释器的过程更像一棵有很多分叉的树。如果你卡在了尾调用优化，或者哈希表上时，就去做一些其他的部分。当你在做其他功能的时候，经常会突然得到解决问题的灵感。我力求清晰地组织这份指南和测试，以便某些问题的可以拖延一阵再解决。

对于 “可推迟/可选” 的说明: 当你运行某个步骤的测试的时候，最后的一些测试可能会有 "optional"（可选的任务）的标记。这表示这些测试的功能对于基础的 mal 实现不是必须的。这份指南中的很多步骤有个 “可推迟的任务”(deferrable) 小节，它们并不是相同的意思。这些小节中包括了被标记为 "optional" 的测试，但也包括了对于后面步骤中所必要的功能。换句话讲，这是“实现你自己的 Lisp 的一场冒险”。

使用测试驱动开发，make-a-lisp 过程的每个步骤都有一组与之相关的测试，在过程中也会有用来运行特定步骤所有测试的脚本。找出一个失败的测试，修复它，重新测试，直到那个步骤中所有测试都能通过为止。

## 参考代码

`process` 目录包含了一些过程中每个步骤的简略伪代码的和架构图。你可以用一个文本比较工具来比较前一个步骤和你当前步骤的伪代码之间的区别。每张架构图中对于上一步所做的变更都以红色高亮表示。这还有一份 [小抄 (cheatsheet)](http://kanaka.github.io/mal/process/cheatsheet.html)，简明扼要地描述了每个步骤的关键变化。

当你被某个步骤彻底卡住了，并且有了想放弃的念头，那么你应该“作个小弊”，参考一下已经存在的语言的实现对于当前步骤或功能的代码。因为你是来学习的，不是来考试的，所以不要有太强的负罪感。好吧，你最好还是能稍微认识到，这种行为不太好。

<a name='the-make-a-lisp-process-1'></a>
## Make-A-Lisp 实现一个 Lisp 解释器
以下步骤的目标语言叫作 "quux"，文件名后缀是 "qx"

<a name='step-0-the-repl'></a>
### 步骤 0: The REPL 读取求值打印循环(Read-Eval-Print Loop)
![step0_repl](images/step0_repl.png)

这个步骤基本上仅创建了你解释器的框架。

* 在 `quux/` 目录下创建 `step0_repl.qx` 文件。
* 添加 4 个函数 `READ`,`EVAL`,`PRINT` 以及 `rep`(read-eval-print)。`READ`,`EVAL` 和 `PRINT` 基本上是个假的实现，它只是返回它们的第一个参数（如果你的目标语言是静态类型的话，是一个字符串），`rep` 按顺序调用它们，将前一个的返回值传递给下一个。
* 添加一个主循环，让它循环打印提示符(提示符必须是"user> "才能通过测试)，读取一行用户输入，对于那一行输出调用rep函数，然后将结果打印出来。当你发送EOF的时候(通常是按 Ctrl-D)，它应该退出。
* 如果你使用一个编译型语言（静态编译「ahead-of-time」，而不是即时编译「just-in-time」），那么在你的目录下创建一个 Makefile（或者适当的项目定义文件）。

现在是时候运行你的第一部分测试了。它们将检查你的程序的输入和事实是否可以被测试所捕获。在目录顶层运行如下命令：

```
make "test^quux^step0"
```

将 `step0_repl.qx` 和 `Makefile` 添加并提交到 git 仓库中。

恭喜你！你已经完成了 make-a-lisp 的第一个步骤。

#### 可选的任务：
* 为你的解释器的 REPL 增加整行编辑和命令历史功能。许多语言已经提供了支持行编辑的库/模块。另外一个选项是，如果你用的语言支持用 FFI (foreign function interface 外来函数接口)来直接加载调用 GNU readline, editline 或 linenoise 库。将行编辑接口代码写在 `readline.qx` 文件中。

<a name='step-1-read-and-print'></a>
### 步骤 1: Read and Print 读取和打印
![step1_read_print](images/step1_read_print.png)

这个步骤中，你需要让你的解释器 “读取”(read) 用户输入的字符串，并把它解析为一种内部的树形数据结构（AST，抽象语法树），然后将这个数据结构 “打印”(print) 成字符串。

在非 Lisp 类语言中，这个步骤（叫作“词法分析和语法分析”）将会是编译器/解释器中最复杂的部分之一。而在 Lisp 中，你想要的这种数据的结构与程序员写的代码的结构基本上是一致的（Homoiconicity，同像性）。

举个例子，如果字符串是 "(+ 2 (* 3 4))" 那么读取 (read) 函数将把它解析为这样的树形结构：

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

每个左括号和与它匹配的右括号 (在 lisp 中叫 S-表达式"sexpr") 成为树中的一个节点，而其他的一切都成为了树中的叶子节点。

如果你能找到一份你目标语言的 JSON encoder/decoder 的实现代码，那么你就可以通过借鉴和修改它来搞定本步骤中 75% 的任务。

这一节余下的部分假设你没有从 JSON encoder/decoder 起步，而是使用了一个 Perl 兼容的正则表达式 (PCRE) 库/模块。的确，你可以采用简单的字符串操作来实现这个功能，但那样更复杂。`make` 和 `ps`(postscript) 和 Haskell 的实现中有一些不使用正则表达式实现 reader 的例子。

* 复制 `step0_repl.qx` 并重命名为 `step1_read_print.qx`。
* 新建一个 `reader.qx` 文件来保存与 reader 有关的函数。
* 如果你的目标语言有面向对象 (OOP) 的特性，那么下一步是在 `reader.qx` 中创建一个简单的有状态的 Reader 对象。这个对象用来保存 tokens 和 position。Reader 对象需要有两个方法：`next` 和 `peek`。`next` 返回当前位置 (position) 的 token，并且增大 position。而 `peek` 只是返回当前位置的 token。
* 在 `reader.qx` 中增加一个 `read_str` 函数。这个函数需要调用 `tokenizer` 获得 token 列表，然后使用这些 token 来创建一个新的 Reader 实例。然后它调用 `read_form` 来处理这个 Reader 实例。
* 在 `reader.qx` 中增加一个 `tokenizer` 函数。这个函数接受一个字符串参数，并且将返回一个数组 / 列表，里面包含了所有的 token(或者叫字符串，string）。下面的正则表达式 (PCRE) 能够匹配所有的 mal 的 token。

```
[\s,]*(~@|[\[\]{}()'`~^@]|"(?:\\.|[^\\"])*"|;.*|[^\s\[\]{}('"`,;)]*)
```

* 在括号中，每一个被下列六种正则表达式匹配到的字符串，将创建出一个新的 token

  * `[\s,]*`: 匹配任意个数的空格或逗号。它不是捕获对象，因此它会被忽略掉，不会标记化(not tokenized)
  
  * `~@`: 捕获两个特殊字符的组合 `~@`，会被标记化(tokenized)
  
  * ```[\[\]{}()'`~^@]```: 捕获 ```[]{}()'`~^@``` 这些字符中任意一个，会被标记化(tokenized)
  
  * `"(?:\\.|[^\\"])*"`: 捕获由双引号开头，并到下一个双引号结束之间的内容，如果中间出现双引号，且双引号前面有反斜杠，则将它们也包括在捕获的内容中，直到下一个双引号。会被标记化(tokenized)
  
  * `;.*`: 捕获由分号 `;` 开头的任意序列，会被标记化(tokenized)
  
  * ```[^\s\[\]{}('"`,;)]*```: 捕获一系列由零个或更多个非特殊字符组成的序列(如，symbol, 数字,"true","false"以及"nil")
  
* 在 `reader.qx` 文件中增加一个 `read_form` 函数，这个函数要读取(peek) Reader 对象的第一个 token，然后对 token 的第一个字符做条件判断。如果第一个字符是左括号，则使用 `read_list` 函数处理这个 Reader 对象。否则使用 `read_atom` 函数处理 Reader 对象。`read_form` 的返回值是一个 mal 数据类型。如果你的目标语言是静态类型语言，那么你要想办法让 `read_form` 函数能够返回出不同的类型或者子类型。举例来说，如果你用的是一门面向对象的语言，那么你可以在最顶层中定义 MalType(在 types.qx 中)，随后你的其他 mal 数据结构就可以继承它了。 MalList 类型（也是继承自 MalType）将由一个包含其他 MalType 对象的数组/列表构成。如果你用的语言是动态类型的，那么只需要返回一个包含其他 MalType 对象的数组/列表即可
* 在 `reader.qx` 中新增一个 `read_list` 函数。这个函数将对 Reader 对象反复调用 `read_form` 函数，直到遇到个 ')' 字符(如果在')' 之前遇到了 EOF，那就说明出错了)。它把调用结果收集到一个 List 类型中。如果你的语言中不存在能够存储多个 mal 数据类型的值的顺序数据类型，那么你需要自己实现一个（在 `types.qx` 中实现）。注意 `read_list` 函数反复调用的是 `read_form`，而不是 `read_atom` 函数。这种在 `read_list` 与 `read_form` 之间的递归定义可以能够让列表中包含列表。 
* 在 `reader.qx` 中新增一个 `read_atom` 函数。函数将会解析 token 的内容，并返回合适的纯（简单，非复合的）数据类型。最开始，你可以只实现数字类型（整型 integer）和 symbol。这能使你继续后面的一些步骤，在随后再继续实现其他的一些类型: nil, true, false 和 string。这些保留的 mal 类型: 关键字(keyword), 向量(vector), 哈希表(hash-map) 和 原子(atom) 在步骤 9 之前都不需要实现（但可以在本步骤到步骤 9 之间的任意时间点实现）。还有，符号(symbol) 类型只是一个由单独的字符串名字构成的对象（有些语言已经有 symbol 类型了）。
* 创建一个名为 `printer.qx` 的文件。这个文件包含一个叫 `pr_str` 的函数，它的功能与 `read_str` 正好相反：输入一个 mal 数据结构，返回出它的字符串形式。但是 `pr_str` 的功能很简单，它只是对于输入对象的一个 switch 条件语句：
  * symbol: 返回 symbol 的字符串名字
  * number: 将数字作为一个字符串返回
  * list: 对于列表中的每一个元素调用 `pr_str`，然后将结果使用空格分隔，把它们拼接在一起，最后在最外面加上括号
* 修改 `step1_read_print.qx` 中的 `READ` 函数，让它调用 `reader.read_str`，并且 `PRINT` 函数调用 `printer.pr_str`。`EVAL` 函数继续直接返回输入的东西，但是如今返回值的类型应当是 mal 数据类型了。

现在你已经实现了足够多的东西，可以开始测试你的代码了。你可以手工测试一些简单的输入：

* `123` -> `123`
* `123` -> `123`
* `abc` -> `abc`
* `abc` -> `abc`
* `(123 456)` -> `(123 456)`
* `(123 456 789)` -> `(123 456 789)`
* `(+ 2 (* 3 4) )` -> `(+ 2 (* 3 4))`

为了验证你的代码不只是去掉了多余的空格（并且没有失败），你可以 instrument 你的 `reader.qx` 中的函数。

当你已经通过了上述的简单手工测试，就可以运行步骤 1 的测试了。到最顶层的目录，执行下面的命令：

```
make "test^quux^step1"
```
修复所有与 symbol, number 和 list 有关的失败测试。

你现在已经完成了最困难的步骤之一。从这儿开始就是下山的路了，剩下的步骤可能更简单，并且每个步骤会逐渐让你有更多的收获。


#### 可推迟的任务:
* 为你的 reader 和 printer 函数增加其他的基础类型的支持: string, nil, true, 和 false. 这些类型在步骤 4 的时候就是必需的了。在读取一个字符串之后，要进行下列的转换：一个反斜杠后面跟着双引号的时候 `\"`，需要把它们翻译为一个普通的双引号 `"`，反斜杠后跟着 n 的时候 `\n` 需要翻译为换行，一个反斜杠后面跟着另一个双引号的时候 `\\`，需要把它们翻译为一个单引号 `\`。为了能正确的打印字符串（为了步骤 4 中的字符串函数），`pr_str` 函数需要另一个叫作 `print_readably` 的参数。当这个参数为 true 的时候，双引号、换行符和反斜杠会被翻译为它们被打印出来的表现形式（与 reader 的逻辑正好相反）。主程序中的 `PRINT` 函数应该在调用 `pr_str` 时将 print_readably 设置为 true。
* 为 reader 函数增加更多的错误检查，确保括号都能够正确匹配。在主循环中捕获并打印这些错误信息。如果你的语言中没有 try/catch 风格的冒泡式异常处理功能，那么你需要在代码中加上一个显式的异常处理，并且跳过错误，不要让程序崩溃掉。
* 为 reader 加上宏 (macros) 支持。这能够在读取阶段时将某些形式转换为其他形式。在 `tests/step1_read_print.mal` 中可以找到需要支持哪种宏的形式（它们只是对 token 流的简单转换）
* 支持其他几种 mal 数据类型: keyword, vector, hash-map.
  * 关键字 (keyword): keyword 是由冒号开头的 token。keyword 只能存储为有特殊 unicode 前缀的字符串，像 0x29E (或字符 0xff/127, 如果目标语言没有很好的 unicode 支持的话) printer 会把带这个前缀的字符串转换回关键字表示。这能够让在大多数语言中使用关键字作为哈希表的键变得很容易。你也可以将关键字存储为一种唯一数据类型，但你要确定它们可以作为哈希表的键使用（这可能需要一种类似的前缀）。
  * 向量 (vector): 向量可以使用与列表相同的底层类型实现，只要有一种能够记录它们之间区别的机制就行。你可以通过在开头和结尾的 token 上加上参数，从而做到使用同一个 reader 函数操作列表和向量的功能。
  * 哈希表 (hash-map): 哈希表是一种关系型数据结构，它将字符串映射到其他 mal 类型的值上。如果你将关键字实现为带前缀的的字符串，那么你只需要一种原生的关系数据结构，只要它支持以字符串作为键就可以了。Clojure 支持把任何值作为哈希表的键，但在 mal 的基础功能中只需要支持把字符串作为键即可。因为将哈希表表示为键和值的交替序列，你可能可以用读取列表和向量的 reader 函数来处理哈希表，只需要用参数来标示它的开头和结尾 token 即可。奇数位置的 token 作为键，而偶数位置的 token 作为值。
* 为你的 reader 增加对注释的支持。tokenizer 应该忽略由 ";" 开头的 token。你的 `reader_str` 函数需要正确的处理 tokenizer 不返回任何值的情况。最简单的办法是返回 `nil` 这个 mal 类型的值。一个更加简明的（在这种情况下不打印 nil）方式是抛出一个特殊的异常，使主循环直接在循环的开头跳过循环，从而不调用 rep。

<a name='step-2-eval'></a>
### 步骤 2: Eval 求值
![step2_eval](images/step2_eval.png)

在步骤 1 中，你的 mal 解释器基本上只有验证输入然后去除输出结果中多余空格的功能。在本步骤中，你将会为你的解释器增加 evaluator (EVAL) 的功能，从而把它改成一个简单的计算器。

比较步骤 1 和步骤 2 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step1_read_print.txt ../process/step2_eval.txt
```

* 将 `step1_read_print.qx` 复制为 `step2_eval.qx`
* 定义一个简单的初始化 REPL 环境。这个环境是一个关联数据结构，将符号 (或符号名 symbol names) 映射为数学运算函数。例如，在 python 中，这个环境应该看起来是这个样子的：

```
repl_env = {'+': lambda a,b: a+b,
            '-': lambda a,b: a-b,
            '*': lambda a,b: a*b,
            '/': lambda a,b: int(a/b)}
```

* 修改 `rep` 函数，将这个 REPL 环境作为调用 `EVAL` 函数时的第二个参数。
* 创建一个新函数 `eval_ast`，它将接受 `ast`(mal 数据类型)和一个关系数据结构（上文中的环境）。`eval_ast` 函数对 `ast` 的类型进行下列匹配，并做出相应处理：
  * symbol: 在环境结构中查找符号，返回对应的值，或在值不存在时报错
  * list: 返回对于列表中的每个元素 `EVAL` 调用得到的结果所组成的列表
  * 否则直接返回原 `ast` 值
* 修改 `EVAL` 函数，检查它的第一个参数 `ast` 是不是一个列表。
  * `ast` 不是列表: 返回对它调用 `eval_ast` 得到的结果
  * `ast` 是一个空的列表: 原封不动地返回 ast
  * `ast` 是一个列表: 调用 `eval_ast` 得到一个新的求值后的列表。取求值结果列表的第一项，将它作为函数调用，以求值结果列表的余下项作为参数传入。

如果你的目标语言不支持可变长度参数（例如，variadic, vararg, splats, apply），那么你需要将整个参数的列表作为一个单独的参数，然后在每个 mal 函数中再将它切分为一个个独立的值。这样做虽然比较闹心，但还是可以凑合用的。

调用或执行一个列表并返回某些新的东西，这样的过程在 Lisp 中被称为 "apply"(应用) 步骤。

用这些表达式来进行测试：

* `(+ 2 3)` -> `5`
* `(+ 2 (* 3 4))` -> `14`

你最可能遇到的挑战是，如何正确地以一个参数列表作为参数调用函数引用。

现在，回到顶层目录，执行步骤 2 的测试，并修复错误。

```
make "test^quux^step2"
```

现在你拥有了一个简单的前缀表达式计算器。

#### 可推迟的任务：

* `eval_ast` 应该对向量和哈希表中的元素进行求值。在 `eval_ast` 函数中加入下列条件判断：
  * 如果 `ast` 是一个向量: 返回对于向量中的每个元素 `EVAL` 调用得到的结果所组成的向量
  * 如果 `ast` 是一个哈希表: 返回一个新的哈希表，它的键是从原哈希表中来的键，值是对于原哈希表中的键所对应的值调用 `EVAL` 得到的结果。

<a name='step-3-environments'></a>
### 步骤 3: Environments 环境
![step3_env](images/step3_env.png)

在步骤 2 中我们已经实现了 REPL 环境 (`repl_env`)，在这个环境中可以存储和查找基本的算数运算函数。在本步骤中，你将会为解释器增加创建新环境 (`let*`) 和修改已存在的环境 (`def!`) 的功能。

Lisp 的环境是一个关系数据结构，它将符号 (即键) 映射到值上。但是 Lisp 环境还有一个很重要的额外功能：它们可以引用 (refer) 另一个环境 (外层的环境)。在环境中进行查找时，如果当前环境中没有要找的符号，那么将在外层环境中继续查找，持续进行这个过程，直到找到符号，或者外层的环境是 `nil`(在整个链中的最外层)为止。

比较步骤 2 和步骤 3 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step2_eval.txt ../process/step3_env.txt
```

* 将 `step2_eval.qx` 复制为 `step3_env.qx`
* 创建 `env.qx`，在里面写和环境有关的定义
* 定义一个 `Env` 对象，它有一个关系型数据结构的属性 `data`，并且在实例化的时候需要一个 `outer` 参数
* 为 Env 对象定义下列三个方法：
  * set: 接受一个符号作为键，一个
mal 类型对象作为值，并将它们装入 `data` 结构中 
  * find: 接受一个符号的键参数，如果当前环境中找到了这个键，那么返回环境。如果没有找到，并且外层环境不是 `nil`，那么在外层环境中（递归）调用 find
  * get: 接受一个符号的键参数，并且用 `find` 方法来找到这个键对应的环境，并且返回匹配到的值。如果没有在外层环境的链中没有找到这个键，则抛出一个 "not found"（未找到）错误
* 更新 `step3_env.qx`，使用新的 Env 类型来创建 repl_env (它的 outer 参数设置为 nil)，并且使用 `set` 方法将算数运算函数加入到环境中
* 修改 `eval_ast`，对 env 参数调用 `get` 方法
* 修改 `EVAL` 函数的 apply 的部分，对于列表的第一个元素进行条件判断:
  * symbol "def!": 调用当前环境（`EVAL` 的第二个，名为 `env` 的参数）的 set 方法，使用未求值的第一个参数（列表的第二个元素）作为符号键，并且将已求值的第二个参数作为值
  * symbol "let*": 以当前环境作为 outer，创建一个新的环境，并将第一个参数作为"let*"环境中新的 binding 列表。取 binding 列表的第二个元素，以新 "let*" 环境作为求值环境调用EVAL，然后在 "let*" 环境上调用set，以 binding 列表第一个元素作为键，以求值后的第二个元素作为值。对于 binding 列表中的每个奇/偶对重复进行上述过程。要特别注意的是，在列表前面的 binding，可以被后面的 binding 引用。最终，原始 let* 形式的第二个参数 (即第三个元素) 使用新的 "let*" 环境进行求值，结果作为新的 "let*" 的结果返回。 (新的 let 环境在结束后被丢弃)
  * 否则: 对于 list 调用 `eval_ast`，并像前面一样，将第一个元素应用到余下的元素上。

`def!` 和 `let*` 是 Lisp 中的 "special" (特例)（或叫“特殊 atom”），意思是它们是语言级别的特性，并且更特别，list 中余下的元素（参数）可能会被求值（或根本不被求值），与默认的应用情况——list中的所有元素都在第一个元素被调用到之前，已经求值完毕——不同。以 "special" 作为第一个元素的列表被称为 "special forms"。它们很特殊，因为它们遵守特殊的求值规则。

尝试一些简单的环境测试：

* `(def! a 6)` `->` `6`
* `a` `->` `6`
* `(def! b (+ a 2))` `->` `8`
* `(+ a b)` `->` `14`
* `(let* (c 2) c)` `->` `2`

回到目录的最顶层，运行步骤 3 的测试，并修复错误。

```
make "test^quux^step3"
```

<a name='step-4-if-fn-do'></a>
### 步骤 4: If Fn Do
![step4_if_fn_do](images/step4_if_fn_do.png)

在步骤 3 中，你为解释器增加了环境，和用来操作环境的特殊形式。在本步骤中，你将为 REPL 的默认环境增加三种新的特殊形式 (`if`, `fn*` 和 `do`) 以及几种核心函数。新的结构如下：

`fn*` 特殊形式是用户创建自定义函数的方式。在一些 Lisp 语言中，这个特殊的形式叫 "lambda"。

比较步骤 3 和步骤 4 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step3_env.txt ../process/step4_if_fn_do.txt
```

* 将 `step3_env.qx` 复制为 `step4_if_fn_do.qx`。
* 如果你还没有实现 reader 和 printer 对 nil, true 和 false 的支持，那么你需要在本步骤中实现。
* 修改环境的构建器 / 初始化器，让它接受两个新的参数: `binds` 和 `exprs`。将 `binds` 列表中每个元素 (符号) 绑定(`set`) 到 `exprs` 列表中对应的元素上。
* 在 `printer.px` 中增加对打印函数值的支持。像 `#` 之类的字符串字面值够用了。
* 为 `EVAL` 实现下列特殊形式：
  * `do`: 使用 `eval_ast` 对列表的所有元素进行求值，并返回最终被求值的元素。
  * `if`: 对第一个参数进行求值（即第二个元素），如果结果不是 `nil` 或 `false`，则求值第二个参数（即第三个元素），并返回它的结果。否则，求值第三个参数（第四个元素）并返回执行结果。如果条件是 false 并且没有第三个参数，则返回 `nil`。
  * `fn*`: 返回一个新的函数闭包。闭包的 body 中包含如下内容：    
    * 创建一个新的环境，以 env (外层作用域) 作为 `outer` 参数，第一个参数（即外层作用域的 `ast` 的第二个列表元素）作为 `binds` 参数，闭包的参数作为 `exprs` 的参数。
    * 对于第二个参数（即外层作用域的 `ast` 的第三个列表元素）调用 `EVAL`，使用新的环境。将结果作为闭包的返回值。

如果你的目标语言不支持闭包，那么你需要使用某种在关闭时可以保存值的结构或者对象保存如下的东西：`ast` 列表的第一个和第二个参数（函数参数列表和函数体），以及当前环境 `env`。在这种情况下，你的原生函数需要用相同的方式进行封装。可能你也需要在 `EVAL` 的 apply 部分有一个用来调用对象/结构的函数/方法。

测试你已经实现的基础部分：

* `(fn* [a] a)` -> `#<function>`
* `((fn* [a] a) 7)` -> `7`
* `((fn* [a] (+ a 1)) 10)` -> `11`
* `((fn* [a b] (+ a b)) 2 3`) -> `5`

* 增加一个新的文件 `core.qx`，并定义一个叫 `ns`(namespace)的关系数据结构，将 symbol 映射为函数。将算数运算函数定义放在这个函数里。
* 修改 `step4_if_fn_do.qx`，让它读取 `core.ns` 结构，并将每个 symbol / 函数加入 (`set`) 到 REPL 环境中(`repl_env`)。
* 将下列函数加入到 `core.ns` 中：
  * `prn`: 对于第一个参数调用 `pr_str`，`print_readably` 参数设置为 true，将结果打印到屏幕，并返回 `nil`。注意，完整版本的 `prn` 函数在下面 “可推迟的任务” 一节中。
  * `list`: 接受参数并将它们返回为一个列表。
  * `list?`: 如果参数是一个列表，返回 true，否则返回 false。
  * `empty?`: 将第一个参数当作一个列表处理，如果这个列表是空的，则返回 true，如果里面有元素，返回 false。
  * `count`: 将第一个参数当作一个列表处理，并返回列表包括元素的个数。
  * `=`: 对比前两个参数，如果它们是相同的类型并且值也相同，那么返回 true。在比较两个列表的时候，两个列表中的每两个对应元素都要进行比较，如果它们都相同，返回 true，否则返回 false。
  * `<`, `<=`, `>` 和 `>=`: 将前两个参数作为数字，并且进行数学比较，返回 true 或 false。

回到目录顶层，运行步骤 4 的测试。步骤 4 有大量的测试用例，但是所有的不涉及到字符串的非可选测试，都需要通过。

```
make "test^quux^step4"
```
你的 mal 实现已经开始像一门真正的语言了。你有了流程控制，判断，带词法作用域的用户定义函数，副作用（如果你实现了字符串函数）等。但是我们的小解释器还没有达到 Lisp-ness 的程度。后续的步骤将使你的小玩具改造成为一个全功能的语言。

#### 可推迟的步骤:
* 实现 Clojure 风格的可变函数参数。修改环境的构造器/初始化器，在 `binds` 列表中遇到 "&" 符号的时候，列表中这个 "&" 符号后面的元素将绑定到 "exprs" 列表中余下的还未绑定的部分上。
* 定义一个 `not` 函数，给 mal 自己用。在 `step4_if_fn_do.qx` 中以 "(def! not (fn* (a) (if a false true)))" 为参数调用 `rep` 函数。
* 在 `core.qx` 中实现字符串函数。你需要为 reader 和 printer 实现字符串支持（步骤 1 中的可推迟的步骤）。每一个字符串函数接受若干个 mal 类型的值，打印它们 (`pr_str`) 并将它们组装成一个新的字符串。
  * `pr-str`：对于每个参数调用 `pr_str`，将 `print_readably` 参数设置为 true，用 ""(空格字符)将结果连接，将字符串结果返回。
  * `str`: 对于每个参数调用 `pr_str`，将 `print_readably` 参数设置为 false，用 "" 将结果组装在一起，并返回新字符串。
  * `prn`: 对于每个参数调用 `pr_str`，将 `print_readably` 参数设置为 true，将结果用 " " 连接，把新字符串打印到屏幕上，并返回 `nil`。
  * `println`: 对于每个参数调用 `pr_str`，将 `print_readably` 参数设置为 false，将结果用 " " 连接，把新字符串打印到屏幕上，并返回 `nil`。

<a name='step-5-tail-call-optimization'></a>
### 步骤 5: 尾调用优化
![step5_tco](images/step5_tco.png)

在步骤 4 中，你增加了特殊的形式 `do`, `if` 和 `fn*` 并且定义了一些核心函数。在本步骤中你将实现一个 Lisp 的特性，叫作尾调用优化 (TCO)。也叫“尾递归” 或“尾调用”。

你在 `EVAL` 中已经定义的一些形式最终会回调进入 `EVAL` 中。对于这些以调用 `EVAL` 作为在返回之前做的最后一件事情（尾调用）的形式，你将直接回到过程的开头循环执行，而不是再一次调用它。这个做法的优点是能够避免在调用栈里增加更多的栈帧。这在 Lisp 语言中特别重要，因为它们倾向于用递归代替迭代作为控制结构。（虽然有些 Lisp 的方言中有迭代，例如 Common Lisp）有了尾调用优化，递归可以有像迭代一样的调用栈的效率。

比较步骤 4 和步骤 5 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step4_if_fn_do.txt ../process/step5_tco.txt
```

* 将 `step4_if_fn_do.qx` 复制为 `step5_tco.qx`
* 为 EVAL 的所有代码增加一个循环（如 while true）
* 修改下列的形式，增加对尾调用递归支持：
  * `let*`: 移除在 `ast` 第二个参数（即第三个列表元素）最后的 `EVAL` 调用，将 `env`（换句话说，即作为 `EVAL` 第二个参数传入的局部变量）设置为 `ast` 的第二个参数。回到循环开头继续执行（不返回）。
  * `do`: 修改 `eval_ast` 调用，让它求值所有的参数，除了最后一个参数（第二个列表元素以及后续的元素，但不包括最后一个元素）将 `ast` 设置为 `ast` 最后一个元素。回到循环开头继续执行（`env` 保持不变）
  * `if`: 继续求值条件，但不是求值 true 或 false 分支，而是将 `ast` 设置为被选中分支的未求值的量。回到循环开头继续执行（`env` 保持不变）
* 特殊形式 `fn*` 的返回值现在要变成一个对象 / 结构，它应该带有一个允调用 `EVAL` 的默认情况的树形，来对 mal 函数进行尾调用优化。这些参数是：
  * `ast`: `ast` 的第二个参数（即第三个列表元素），相当于函数的 body。
  * `params`: `ast` 的第一个参数（第二个列表元素），相当于函数的参数名。
  * `env`: 当前 `EVAL` 函数的 `env` 参数的值。
  * `fn`: 原始的函数值（换句话说，就是步骤 4 中 `fn*` 所返回的东西）。注意这是一个可推迟的任务，在步骤 9 的时候 `map` 和 `apply` 才需要用到它。如果在后面的步骤 6 中你选择不推迟 atoms/`swap!` 的实现的话，那么也需要先实现它。
* EVAL 默认的 "apply/invoke" 条件分支现在必须改为可以处理由 `fn*` 形式返回的新对象/结构体。继续对 `ast` 调用 `eval_ast`. 第一个元素是 `f`。 根据 `f` 不同的类型做对应的处理:
  * 常规函数 (不是使用 `fn*` 定义的): 像原来 (步骤 4) 一样apply/invoke。
  * `fn*` 值: 将 `ast` 设置为 `f` 的 `ast` 属性。生成一个新环境，以 `f` 的 `env` 和 `params` 属性作为 `outer` 和 `binds` 参数，以余下的 `ast` 参数 (列表第二个到最后一个元素)作为 `exprs` 参数。将 `env` 设置为新的环境，回到循环的开头继续执行。

执行一下上一步骤中的手工测试，确保你没有因为加入了尾调用优化而破坏了什么东西。

现在回到目录顶层，运行步骤 5 的测试。
```
make "test^quux^step5"
```
看一下步骤 5 的测试文件 `tests/step5_tco.mal`。函数 `sum-to` 不能被尾调用优化，因为它在递归调用之后又做了一些事情（`sum-to` 调用了它自身，之后执行了相加的操作）Lisp 用户说 `sum-to` 并没有在尾部调用。函数 `sum2` 在尾部调用了自己。换句话说，对 `sum2` 的递归调用是 `sum2` 最后一步做的事情。对于一个非常大的值调用 `sum-to` 将在大多数目标语言中导致栈溢出异常。（某些语言使用了非常特殊的技巧来避免栈溢出）

祝贺你，你的 mal 实现已经有了大多数主流语言所缺少的（尾调用优化）特性。

<a name='step-6-files-mutation-and-evil'></a>
步骤 6: Files, Mutation, and Evil
![step6_file](images/step6_file.png)

在步骤 5 中，你为解释器加入了尾调用优化。在本步骤中你将加入一些字符串和文件操作的功能，为你的实现增加一些 evil，呃 eval。只要你的语言支持函数闭包，那么本步骤将非常容易。然而，为了完成本步骤，你必须实现字符串类型的支持，所以如果你之前如果推迟了任务还没完成，你需要回去先把那个搞定。

比较步骤 5 和步骤 6 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step5_tco.txt ../process/step6_file.txt
```

* 将 `step5_tco.qx` 复制为 `step6_file.qx`
* 为核心命名空间增加两个新的字符串函数:
  * `read-string`: 这个函数将 reader 中的 `read_str` 函数暴露了出来。如果你的 mal 字符串类型与你目标语言不一样（例如静态类型语言），那么你的 `read-string` 函数需要将原始字符串通过调用 `read_str` 函数，而从 mal 字符串类型中解包出来。
  * `slurp`: 这个函数接受一个文件名（字符串）并并且将文件的内容作为字符串返回。和上面那个函数一样，如果你的 mal 字符串类型封装了目标语言的字符串，那么你需要将字符串参数解码来得到原始的文件名字符吃，并将结果编码 (封装) 回 mal 字符串类型。
* 在你的主程序中，为你的 REPL 环境增加一个新的符号 "eval"。这个符号对应的值是接受一个参数 `ast` 的函数。闭包调用你的 `EVAL` 函数，以 `ast` 作为第一个参数，REPL 环境作为第二个参数（外层的环境），将调用 `EVAL` 的结果返回。这个简单且强大新功能允许你将 mal 数据作为 mal 程序对待。例如，你现在可以这样做：

```
(def! mal-prog (list + 1 2))
(eval mal-prog)
```

* 使用 mal 语言自身，定义一个 `load-file` 函数，在你的主程序里调用 `rep` 函数，参数为 "(def! load-file (fn* (f) (eval (read-string (str"(do" (slurp f) ")")))))"

测试一下 `load-file`:

* `(load-file"../tests/incA.mal")` -> `9`
* `(inc4 3)` -> `7`

`load-file` 函数做了如下的事情：

* 调用 `slurp` 来通过文件名读取一个文件。将文件内容用 "(do ...)" 进行封装，这样整个文件就可以作为一个单程序的 AST（抽象语法树）。
* 以 `slurp` 的返回值作为参数调用 `read-string` 函数。它使用 reader 读取 / 转换文件的内容，使之成为 mal 数据 / AST.
* 使用 `eval`(在 REPL 环境中的那个)函数处理 `read-string` 函数返回的 AST，“运行”它。

除了增加文件和求值的支持以外，在本步骤中我们也要增加原子数据类型。原子是 mal 用来表示状态的方式，这个灵感来自于[Clojure 的原子](http://clojure.org/state)。原子保存了对一个任意类型 mal 值的引用。它支持读取一个 mal 值，修改它的引用，将它指向另一个 mal 值。注意这是唯一一种可变类型（但是它所引用的 mal 值仍是不可变的，在步骤 7 中有关于不可变特性的进一步解释）你需要在核心命名空间中增加 5 个函数来支持原子。

* `atom`: 输入一个 mal 值，并返回一个新的指向这个值的原子。
* `atom?`: 判断输入的参数是不是原子，如果是，返回 true。
* `deref`: 输入一个原子作为参数，返回这个原子所引用的值。
* `reset!`: 输入一个原子以及一个 mal 值，修改原子，让它指向这个 mal 值，并返回这个 mal 值。
* `swap!`: 输入一个原子，一个函数，以及零个或多个函数参数。将原子的值作为第一参数，并将余下的函数参数作为可选的参数传输函数中，将原子的值置为函数的求值结果。返回新的原子的值。(边注: Mal是单线程的，但在像Clojure之类的并发语言中，`swap!`将是一个原子操作，`(swap! myatom (fn* [x] (+ 1 x)))`总是会把`myatom`计数增加1，并且在原子被多个线程操作时不会导致结果出错)

你可以增加一个 reader 宏 `@`，它相当于一个 `deref` 的简略形式，因此 `@a` 相当于 `(deref a)`。为了达到这个目的，修改 `read_form` 函数的条件判断，增加一个规则，处理 `@`token: 如果 token 是 `@`(at 符号)，那么返回一个包括 `deref` 符号以及读取下一个形式 (`read_form`) 结果的新列表。

现在回到目录顶层，运行步骤 6 的测试。可选测试包括了对注释，向量，哈希表，和 `@` reader 宏的支持:

```
make "test^quux^step6"
```
恭喜你，你现在实现了一个完善的脚本语言，它可以运行其他的 mal 程序。`slurp` 函数把一个文件当作字符串读进来，`read-string` 函数调用 mal reader 将字符串转换为数据，然后 `eval` 函数读取数据，并把它当作一个普通的 mal 程序一样进行求值。然而，我们需要注意，`eval` 函数不是只能用来运行外部程序的。因为 mal 程序就是普通的 mal 数据结构，你可以在调用 `eval` 求值之前，动态生成或者操作这些数据结构。数据和程序之间的同形（形状相同），我们称之为同像性(homoiconicity)。Lisp 语言们的这种同像性将它们与其他大多数语言区分开来。

你的 mal 实现现在已经非常强大了，但在 `core.qx` 中的这组可用的功能还相当有限。你将在步骤 9 和步骤 A 中增加许多功能进去，但你从在接下来的步骤中支持 quoting (步骤 7) 和 macros (步骤 8) 开始，逐渐完善它。

可推迟的任务：
* 增加通过命令行运行其他 mal 程序的能力。在进入 REPL 循环之前，检查你的 mal 实现在被调用的时候有没有带参数，如果有参数的话，将第一个参数视为文件名，使用 `rep` 调用 `load-file` 将文件导入并执行，最后退出 / 终止执行。
* 将剩下的命令行参数传入 REPL 环境，让通过 `load-file` 函数执行的程序能够访问调用它们的环境。为你的 REPL 环境加入一个 "*ARGV*"(符号)。它的值是命令行余下的参数的一个列表。

<a name='step-7-quoting'></a>
### 步骤 7: Quoting

![step7_quote](images/step7_quote.png)

在步骤 7 中，你将为解释器加上 `quote` 和 `quasiquote` 这两个特殊形式，并且加入 `cons` 和 `concat` 这两个核心函数的支持。

特殊形式 `quote` 告诉求值器 (`EVAL`) 不要对参数进行求值。一开始，看起来这个功能没啥卵用，但有一个例子能够证明它有用，让 mal 程序能够有引用一个符号的自身，而不是它经过求值后的结果。比如列表。例如，考虑下列情况：

* `(prn abc)`: 这段程序将在当前求值环境中寻找符号 `abc`。并打印到屏幕上。如果 `abc` 没有被定义过的话，就会报错。
* `(prn (quote abc))`: 这段程序会打印 "abc"（打印符号本身）。它不会去管在当前环境中 `abc` 是否已被定义。
* `(prn (1 2 3))`: 这段程序会报错，因为 `1` 不是函数，不能被应用于参数 `(2 3)` 上。
* `(prn (quote (1 2 3)))`: 这段程序将打印 "(1 2 3)"。
* `(def! l (quote (1 2 3)))`: list quoting 允许我们在代码中直接定义 list(字面列表). 另一个做这件事的方法是使用 list 函数 `(def! l (list 1 2 3))`。

第二种特殊形式是 `quasiquote`。它允许一个 quoted 列表能够有一些临时 unquoted 的元素(正常求值)。有两种特殊形式`unquote` 和 `splice-unquote` 代表在 quasiquoted 列表里面的东西。最好来点例子解释一下:

* `(def! lst (quote (2 3)))` -> `(2 3)`
* `(quasiquote (1 (unquote lst)))` -> `(1 (2 3))`
* `(quasiquote (1 (splice-unquote lst)))` -> `(1 2 3)`

`unquote` 形式将将它的参数逆向求值，并将求值结果放入 quasiquoted 列表。形式 `splice-unquote` 也将它的参数逆向求值，但是被求值的值必须是列表，因为在后面它们会被切分 (splice) 到 quasiquoted 列表中。在将它于 macro 一起使用的时候，quasiquote 形式的真实力量才会展现出来(在下一步骤中)。

比较步骤 6 和步骤 7 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step6_file.txt ../process/step7_quote.txt
```

* 将 `step6_file.qx` 复制为 `step7_quote.qx`
* 在实现这些 quoting 形式时，你需要先在核心命名空间里实现一些支持函数:
  * `cons`: 这个函数将它的第一个参数连接到它的第二个参数 (一个列表) 前面，返回一个新列表。
  * `concat`: 这个函数接受零个或多个列表作为参数，并且返回由这些列表的所有参数组成的一个新列表。

关于不变可性: 注意 cons 和 concat 都没有修改它们的原始列表参数。所有对于它们的引用（换句话说，在其他列表中，它们可能作为其中的元素）将还指向原有的未变更的值。就像 Clojure 一样，mal 是一种使用不可变数据结构的语言。我建议你去学习一下 Clojure 语言中实现的不可变性的能力和重要性，mal 借用了它的大部分语法和特性。

* 添加 `quote` 特殊形式，这个特殊形式返回它的参数（`ast` 的第二个列表元素）

* 添加 `quasiquote` 特殊形式。实现实现一个 helper 函数 `is_pair`，它在参数是一个非空列表的时候返回 true。然后定义 `quasiquote` 函数。在 `EVAL` 中以 `ast` 的第一个参数（即第二个列表元素）为参数调用它，随后 `ast` 被设置为结果，并且回到循环的开头继续执行（TCO，尾调用优化）。`quasiquote` 函数输入 `ast` 参数后，进行如下的条件判断：
  1. 如果 `is_pair` 对于 `ast` 的判断结果是 false: 返回一个新列表，里面包括了一个名为 "quote" 的符号，以及 `ast`。
  2. 否则，如果 `ast` 的第一个元素是符号 "unquote": 返回 `ast` 的第二个元素。
  3. 如果 `is_pair` 对于 `ast` 的判断结果是 true，并且 `ast` 的第一个元素的第一个元素 (即 `ast[0][0]`) 是名为 "splice-unquote" 的符号：返回一个新的列表，其中包含：名为 "concat" 的符号，`ast` 的第一个元素的的第二个元素(即 `ast[0][1]`)，以及以 `ast` 的第二个元素到最后一个元素为参数调用 `quasiquote` 的结果。
  4. 否则: 返回一个新的列表，包括：名为 "cons" 的符号，以 `ast` 的第一个参数(即 `ast[0]`)，和以 `ast` 的第二个元素到最后一个元素为参数调用 `quasiquote` 的结果。

返回目录顶层，执行步骤 7 的测试。

```
make "test^quux^step7"
```

Quoting 是 mal 中许多无聊的函数中的一个，但别因此而灰心。你的 mal 实现已接近完工了，而 quoting 为接下来的接近收工的步骤: 宏 (macro)，做好了准备。

#### 可推迟的任务

* quoting 形式的全名相当罗嗦。大多数 Lisp 语言有一个简写的语法，mal 也不例外。这些简写语法被成为 reader macros 因为它们使我们能够在 reader 阶段中操作 mal 代码。在 eval 阶段中被执行的 macro 只是叫作 macro，我们将在下一节中介绍。扩展 reader 的 `read_form` 函数的条件判断，增加下列情况:
  * token 是 "'"(单引号): 返回一个新列表，包含符号 "quote"，以及对下一个 form 读取的结果(`read_form`)
  * token 是 "\`" (反引号): 返回一个新列表，包含符号 "quasiquote"，以及对下一个 form 读取的结果(`read_form`)
  * token 是 "~" (波浪号): 返回一个新列表，包含符号 "unquote"，以及对下一个 form 读取的结果(`read_form`)
  * token 是 "~@" (波浪号和 at 符号): 返回一个新列表，包含符号 "splice-unquote"，以及对下一个 form 读取的结果(`read_form`)
* 增加对 vector 的 quoting 的支持。`is_pair` 函数在参数是非空列表或 vector 时应该返回 true。`cons` 应该也能接受向量作为第二个参数。返回值是 list regardless。`concat` 应该支持列表，向量，或它们两者进行连接，结果永远是列表。

<a name='step-8-macros'></a>
### 步骤 8: Macros 宏
![step8_macros](images/step8_macros.png)

现在，你的 mal 实现已经为加入最 lisp 范的、最一颗赛艇的编程概念——macro 宏——做好了准备。在之前的步骤中，quoting 能实现一些简单的数据结构操作，以及我们 mal 代码的一些操作（因为在步骤 6 中，我们的 `eval` 函数能够将 mal 数据结构转换为代码）。在本步骤中，你将实现将一个 mal 函数标记为宏的功能，它可以在求值之前操作 mal 代码。换句话说，宏就是用户自定义的特殊形式。从另一角度看，宏允许 mal 程序重新定义 mal 语言本身。

比较步骤 7 和步骤 8 的伪代码，可以对本步骤中将要做的修改有简要的了解：
```
diff -urp ../process/step7_quote.txt ../process/step8_macros.txt
```

* 将 `step7_quote.qx` 复制为 `step8_macros.qx`
你可能认为，宏的无限力量可能需要实现某种复杂的机制。然而，其实实现起来非常简单。

* 为 mal 函数类型添加一个新的属性 `is_macro`，这个属性默认是 false
* 添加一个新形式 `defmacro!`。它与 `def!` 形式非常类似，但是在将 mal 函数加入到环境中之前，将 `is_macro` 属性设置为 true
* 添加一个 `is_macro_call` 函数：这个函数接受两个参数 `ast` 和 `env`。当 `ast` 列表第一个元素是个符号，并且这个符号指向 `env` 环境中的一个 `is_macro` 属性为 true 函数时，返回 true，否则返回 false。
* 添加一个 `macroexpand` 函数：这个函数接受两个参数 `ast` 和 `env`。它调用 `is_macro_call` 函数，传入参数 `ast` 和 `env`，并且在条件为 true 时进行循环。在循环中，`ast` 列表中的第一个元素 (一个符号) 在环境中查找 macro 函数。这个宏函数随后会以 `ast` 余下的元素（第二个到最后一个）作为参数被调用/应用。宏调用的的返回值将成为 `ast` 的新值。当 `ast` 不再是一个宏调用时，循环结束，当前 `ast` 的值会被返回。
* 在求值器 (`EVAL`) 中，在。将 `ast` 设置为调用的结果。如果 `ast` 的新值不再是一个列表 after macro expantion，那么任何对它调用 `eval_ast` 的结果，否则继续剩下的 apply 部分(特殊形式的条件判断)。
* 为 `macroexpand ` 添加一个新的特殊形式。以 `ast` 的第一个参数（第二个列表元素）和 `env` 作为参数调用 `macroexpand`。返回结果。这个特殊形式允许 mal 程序在不应用结果时进行显式的宏扩展（这在调试宏扩展时十分有用）

回到目录顶层，执行步骤 8 的测试：

```
make "test^quux^step8"
```
在一开始，宏测试极有可能无法通过。尽管宏的实现非常简单，但调试宏的运行时 bug 非常的困难。如果你遇到了很难搞定的问题，我在这里给你一些建议：

* 使用 macroexpand 特殊类型来排除 indirection 的一层（扩展但是跳过求值）。通常来说这能让我们找到问题的源头。
* 在 `eval` 函数的最上面（在 TCO 循环中）增加一个 debug print 的语句，打印当前 `ast` 的值（提示：使用 `pr_str` 来获得便于 debug 的输出）。在其他语言的实现中找到步骤 8 的代码，并解除注释它的 `eval` 函数（是的，我允许你违反一次规则）。将两者同步执行，并进行对比。第一处不同的输出可能指示着 bug 的所在。

恭喜你！你的 Lisp 解释器现在有了超能力，而其他非 Lisp 语言只有羡慕的份。（我非常确信，编程语言在你不使用它们的时候是会做梦的）如果你还不熟悉 Lisp 宏，我建议你做如下练习：写一个递归宏，处理后缀 mal 代码（也巨是说将函数作为最后一个，而不是第一个参数）或者不做这样的练习，因为事实上我自己也没尝试过，但我听说这是一个很有趣的练习。

在下一步骤中，你将为你的实现添加 try/catch 风格的异常处理，以及一些新的核心函数。经过步骤 9，你的实现将非常接近一个可以自举的 mal 语言实现。让我们继续吧！

可推迟的任务：
* 添加下列这些在宏函数中经常用到的核心函数：
  * `nth`: 这个函数接受一个列表（或向量）以及一个数字（序号）作为参数，返回列表中给定序号位置的元素。如果序号超出了返回，函数抛出一个异常。
  * `first`: 这个函数接受一个列表（或向量）作为参数，返回它的第一个元素，如果列表（或向量）是空的，或者参数本身是 `nil`，则返回 `nil`。
  * `rest`: 这个函数接受一个列表（或向量）作为参数，返回除第一个元素之外的所有元素。
* 在主程序中，使用 `rep` 函数定义两个新的控制结构宏，下面是调用 `rep` 定义这些宏时用到的字符串参数:

  * `cond`: "(defmacro! cond (fn* (& xs) (if (> (count xs) 0) (list'if (first xs) (if (> (count xs) 1) (nth xs 1) (throw \"odd number of forms to cond\")) (cons'cond (rest (rest xs)))))))"
    * 注意，`cond` 在 `cond` 为奇数个参数时调用了 `throw` 函数。 `throw` 函数将会在下一步中进行实现，但它仍要引发一个未定义符号错误来表明它的意图。
  * `or`: "(defmacro! or (fn* (& xs) (if (empty? xs) nil (if (= 1 (count xs)) (first xs) `(let* (or_FIXME ~(first xs)) (if or_FIXME or_FIXME (or ~@(rest xs))))))))"

<a name='step-9-try'></a>
### 步骤 9: Try
![step9_try](images/step9_try.png)

在本步骤中，你要实现 mal 的最后一种特殊形式，用来进行异常处理的：try*/catch*. 你也需要为你的实现添加一些核心函数。特别是，你将会为你的实现增加 apply 和 map 核心函数，来加强它的函数式编程的血统。


比较步骤 8 和步骤 9 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step8_macros.txt ../process/step9_try.txt
```

* 将 `step8_macros.qx` 复制为 `step9_try.qx`
* 为 `EVAL` 函数添加 `try*/catch*` 特殊形式。try catch 形式看起来是这样的: `(try* A (catch* B C))` 对 A 进行求值，如果抛出了异常，那么在符号 `B` 绑定到抛出异常的值的环境中，对 `C` 进行求值。
  * 如果你的目标语言有内建的 try/catch 风格的异常处理，那么你已经完成了 90% 的工作。增加一个(原生语言)try/catch 程序块，在 try 部分中对 `A` 进行求值，并捕获所有异常。如果捕获到异常，则将异常翻译为一个 mal 类型/值。对于原生的异常，这可以是一个消息字符串或一个 mal hash-map，其中包括了消息字符串和一些关于异常的其他异常。当一个通常的 mal 类型 / 值被当作异常，你可能需要将它报保存在原生的异常类型中，以便使用原生的 try/catch 机制对它进行处理。然后你要将 mal 类型/值从原生的异常中解出来。创建一个新的 mal 环境，在这个环境中，将 `B` 与异常的值绑定。最后，使用新的环境对 `C` 进行求值。
  * 如果你的目标语言没有内建的 try/catch 风格的异常处理，那么你就有一些额外的工作要做了。最直接的做法之一是创建一个全局的错误变量，保存被抛出的 mal 类型/值。但它的复杂之处在于，在许多地方你都必须检查这个全局错误状态是否已经被设置了，从函数中返回，中断执行。最佳的规则是，这个检查应该放在你 EVAL 函数的最开始，以及每一个对 EVAL 的调用之后（在后续调用链中的任何一个可能对 EVAL 进行调用的函数之后）是的，这样的做法非常不优雅，但在一开始选择目标语言的时候，我已经警告过你了。
* 添加 `throw` 核心函数
  * 如果你的目标语言支持 try/catch 风格的异常处理，那么本函数接受一个 mal 类型/值，并将它作为一个异常抛出。为了做到这一点，你可能需要创建封装了 mal 对象/类型的自定义异常对象。
  * 如果你的目标语言没有内建的 try/catch 风格的异常处理，则为这个 mal 对象/类型设置全局错误状态。
* 增加 `apply` 和 `map` 核心函数。在步骤 5 中，如果你没有将原始函数 `fn` 加入到 `fn*` 返回的结构中，那么要先把它搞定。
  * `apply`: 接受至少两个参数。第一个参数是一个函数，最后一个参数的是列表（或向量）。在函数和最后一个参数（如果有的话）之间的参数与最后一个参数连接起来，创建参数被用来调用函数。这个 apply 函数允许一个函数的参数被包含在一个列表（或）向量中。换句话说，`(apply F A B [C D])` 就相当于 `(F A B C D)`。  
  * `map`: 接受一个函数和一个列表，并对列表（或向量）中的每一个元素求值，并将结果返回为一个列表。
* 添加一些类型断言函数。在 Lisp 中，断言函数通常以以 "?" 或者 "p" 结尾，返回 true 或 false（或 true 值 / nil）。
  * `nil?`: 接受一个参数，如果参数是 nil(mal 中的 nil 值)的话返回 true (mal 中的 true 值)。
  * `true?`: 接受一个参数，如果参数是 true(mal 中的 true 值)的话，返回 true (mal 中的 true 值)。
  * `false?`: 接受一个参数，如果参数是 false(mal 中的 false 值)的话，返回 true (mal 中的 true 值)。
  * `symbol?`: 接受一个参数，如果参数是 symbol(mal 中的 symbol)的话，返回 true (mal 中的 true 值)。

现在回到顶层目录，运行步骤 9 的测试：

```
make "test^quux^step9"
```
你的 mal 实现现在大体上已经是一个功能完善的 Lisp 解释器了。但是如果你止步于此，就会错过创造一个 mal 实现中最激动人心和富有启发性的部分：自足执行 self-hosting。

#### 可推迟的任务
* 增加如下新的核心函数:
  * `symbol`: 接受一个字符串，返回以这个字符串作为名字的符号。
  * `keyword`: 接受一个字符串，返回有相同名字的关键字 (通常是有前缀的特殊unicode符号) 这个函数也被用来判断参数是否已经是一个关键字，并返回它。
  * `keyword?`: 接受一个参数，如果参数是关键字的话，返回 true(mal 中的 true 值)，否则返回 false(mal 中的 false 值)
  * `vector`: 接受若干个参数，返回包含这些参数的一个向量。
  * `vector?`: 接受一个参数，如果参数是向量的话，返回 true(mal 中的 true 值)，否则返回 false(mal 中的 false 值)
  * `hash-map`: 接受偶数数量的参数，返回一个新的 mal 哈希表，其中键为奇数位置的参数，它们的值分别为与之对应的偶数位置的参数，它基本上是 `{}`reader 字面语法的函数形式。
  * `map?`: 接受一个参数，如果参数是哈希表的话，返回 true(mal 中的 true 值)，否则返回 false(mal 中的 false 值)
  * `assoc`: 接受一个哈希表作为第一个参数，余下的参数为需要关联(合并)到哈希表里的奇/偶-键/值对。注意，原始的哈希表不会被修改(记住，mal 的值是不可变的)，旧哈希表中的键/值与参数中的键/值对合并而成的新的哈希表作为结果返回。
  * `dissoc`:接受一个哈希表作为第一个参数，余下的参数为需要从哈希表中删除的键。与前面一样，注意原始的哈希表是不变的，只是把删除了参数中指定的键的新哈希表返回出来。参数列表中在原哈希表不存在的键会被忽略。
  * `get`: 接受一个哈希表和一个键，返回哈希表中与这个键对应的值，如果哈希表中不存在这个键，则返回 nil。
  * `contains?`: 接受一个哈希表和一个键，如果哈希表中包含这个键，则返回 true(mal 中的 true 值)，否则返回 false(mal 中的 false 值)。
  * `keys`: 接受一个哈希表，并返回一个列表(mal 中的 列表值)，其中包含了哈希表中的所有的键。
  * `vals`: 接受一个哈希表，并返回一个列表(mal 中的 列表值)，其中包含了哈希表中的所有的值。
  * `sequential?`: 接受一个参数，如果参数是列表或者向量的话，返回 true(mal 中的 true 值)，否则返回 false(mal 中的 false 值)

<a name='step-a-metadata-self-hosting-and-interop'></a>
### 步骤 A: Metadata, Self-hosting and Interop

![stepA_mal](images/stepA_mal.png)

现在你来到了实现 mal 的最后一个步骤。本步骤是对一些无法放入到其他步骤中的任务的集合。更重要的是，在本步骤你做的事情，将解锁名为“自足执行”的神秘力量。你可能已经注意到，我们的诸多 mal 实现中其中一个，是用 mal 语言实现的。任何足够完善的 mal 实现都可以运行由 mal 语言实现的 mal。如果你之前从未构建过一个编译器或是解释器的话，你可能需要些时间思考一会。查看 mal 语言实现的 mal 的源码文件（因为你已经到了步骤 A，所以这不算是作弊了）。

如果你推迟了关键字，向量和哈希表的实现，那么如果你想实现自足执行的话，现在就需要先回去把这些任务完成。

比较步骤 8 和步骤 9 的伪代码，可以对本步骤中将要做的修改有简要的了解：

```
diff -urp ../process/step9_try.txt ../process/stepA_mal.txt
```

* 将 `step9_try.qx` 复制为 `stepA_mal.qx`
* 添加 `readline` 核心函数。这个函数接受一个字符串，用来提示用户输入。用户输入的文本作为一个字符串返回。如果用户输入了 EOF(通常是 Ctrl-D)，则返回 nil.
* 通过在mal函数上增加一个 metadata 属性，来为 mal 函数增加 meta-data 支持。这个属性引用了另一个 val 值/类型 (默认是nil)。添加下列与 metadata 相关的核心函数：
  * `meta`: 它以一个 mal 函数作为参数，返回 metadata 属性的值。
  * `with-meta`: 这个函数接受两个参数，第一个参数是一个 mal 函数，第二个参数为要设置为 metadata 的mal 值/类型。本函数将返回第一个参数函数的拷贝，且它的 `meta` 属性设置为第二个参数。注意环境和宏属性在拷贝的时候要同样保留下来。
* 为你的 REPL 环境添加一个新的 "\*host-language\*"(symbol)入口。这个入口的值包含了当前实现的名字。
* 当 REPL 启动时（区别于使用脚本和参数调用启动时），调用 `rep` 函数，打印下列字符串启动信息: "(println (str"Mal ["\*host-language*"]"))"

现在，回到目录顶层，运行步骤 A 的测试:

```
make "test^quux^stepA"
```

当你通过了步骤 A 的所有非可选测试，现在是尝试自足执行的时候了。正常启动你步骤 A 的实现，但是使用你在步骤 6 中实现的文件参数模式来运行 mal 语言版本的实现。

```
./stepA_mal.qx ../mal/step1_read_print.mal
./stepA_mal.qx ../mal/step2_eval.mal
...
./stepA_mal.qx ../mal/step9_try.mal
./stepA_mal.qx ../mal/stepA_mal.mal
```

这是一个很好的机会，在你运行 mal 语言实现的 mal 时找到一些错误。在自足执行时进行 debug 会更加困难，并且非常虐心。我自己找到的一个最好的方式是在 mal 实现的出错的步骤中添加 prn 语句（不是你自己的 mal 实现中）

另一个我经常用的方式是，将 mal 实现中导致问题的代码拿出来，一步步简化这些代码，直到将它们简化为可以重现问题的最简代码片段。当这段代码足够精简时，你可能就知道你 mal 实现的问题出在哪了。请将你的精简复现代码加入到测试用例中，将来的实现者就可以在它们进行自足执行之前就能避免问题，因为在这一步进行问题的定位和修复太困难了。

一旦你可以手工运行所有的自足执行步骤时，现在是在自足执行模式下运行所有测试的时候了。

```
make MAL_IMPL=quux "test^mal"
```

当你遇到问题时（你几乎肯定要遇到），使用上面说过的方式来进行 debug。

恭喜你！！！当所有测试通过后，你可以停下来并想想你已经完成了什么。你已经实现了一个 Lisp 解释器，它非常强大，并足够完整——可以运行一个大型的 mal 程序，而这个程序是 mal 语言实现的 mal 解释器。你甚至可能会问可否继续使用你的 mal 实现来运行 mal 实现，而它自己也是一个 mal 实现，如同盗梦空间一样。

#### 可选的任务: gensym

我们在步骤 8 中加入的 `or` 宏有一个 bug。它定义了一个名为 `or_FIXME` 的变量，它覆盖了用户代码中使用了这个宏。如果用户有一个变量叫 `or_FIXME`，它就无法作为 `or`宏的参数。为了修复这个问题，我们引入了 `gensym`: 这个函数返回一个在此前的程序中从未用到的符号。这也是一个使用 mal 原子来维护状态 (这里的状态是目前由 `gensym` 产生的符号的数量) 的例子。

之前，你使用 `rep` 来定义 `or` 宏。移除这个定义，使用 `rep` 定义一个新的，反向的 `gensym` 函数和清真的 `or` 宏。下面是你需要传给 `rep` 的字符串参数:

```
"(def! *gensym-counter* (atom 0))"

"(def! gensym (fn* [] (symbol (str \"G__\"(swap! *gensym-counter* (fn* [x] (+ 1 x)))))))"

"(defmacro! or (fn* (& xs) (if (empty? xs) nil (if (= 1 (count xs)) (first xs) (let* (condvar (gensym)) `(let* (~condvar ~(first xs)) (if ~condvar ~condvar (or ~@(rest xs)))))))))"
```
如果你想了解更多，请阅读这一篇[Peter Seibel's thorough discussion about gensym and leaking macros in Common Lisp](http://www.gigamonkeys.com/book/macros-defining-your-own.html#plugging-the-leaks)

#### 可选的任务
* 为其他的复合数据类型 (list, vector和hash-map) 以及原生函数添加 metadata 支持
* 添加如下新的核心函数：
  * `time-ms`: 不需要参数，返回从 epoch(1970 年 1 月 1 日 00:00:00 UTC)到当前时间之间的毫秒数。如果不能的话，就返回从某一特定时间点到当前时间之间的毫秒数。(`time-ms` 通常被用来进行比较，以衡量持续时间）。在实现 `time-ms` 之后，你可以运行 `make perf^quux` 来对你的 mal 实现进行性能 benchmark。
  * `conj`: 接受一个或更多元素的集合作为参数，返回包含原有集合中的元素以及新元素的集合。如果集合是一个列表，则新元素以逆序插入到列表的前面并返回新列表；如果集合是向量，则新元素被加入到给定的向量的尾部，并返回新向量。
  * `string?`: 如果参数是一个字符串，返回 true
  * `seq`: 接受一个 列表, 向量, 字符串或者 nil。如果传入的是空列表，空向量，或空字符串("")，则返回 nil，否则，如果为列表，则原样返回，如果是向量，则转换为列表并返回；如果是字符串，则将字符串切分为单个字符的字符串列表并返回。
* 为了实现对目标语言的互操作(interop)，添加如下核心函数:
  * `quux-eval`: 接受一个字符串，在目标语言中进行求值，并将返回值转换为相应的 mal 类型并返回。你也可以添加一些其他你觉得合适的互操作函数；比如 Clojure，有一个名为 `.` 的函数，允许调用 Java 的方法。如果目标语言是静态类型语言，尝试使用 FFI 或者一些因语言而异的反射机制。`quux-eval` 和其他的互操作函数的测试应该添加到 `quux/tests/stepA_mal.mal` 中。（例子请见[`lua-eval` 的测试](https://github.com/kanaka/mal/blob/master/lua/tests/stepA_mal.mal)）

### 下一步
* 加入 #mal IRC 频道。那里其实挺安静的，但有时候会有与mal，Lisp或一些深奥的编程语言相关的讨论。
* 如果你搞了一个新目标语言的实现（或是现有实现的唯一且有趣的变种），可以考虑向 mal 项目提交 Pull Request。[FAQ](https://github.com/kanaka/mal/blob/master/docs/FAQ.md#will-you-add-my-new-implementation) 解释了一个实现能够被合并到仓库中的通常要求。
* 让你的解释器实现生成目标语言的源代码，而不是马上执行它。换句话说，做一个编译器。
* 选一个新的目标语言，用它实现一个 mal。选一个与你所掌握的语言差异较大的语言。
* 用你的 mal 实现去做一个现实世界的项目。可以考虑实现如下的项目：
  * Web server （以 mal 作为 CGI 语言实现功能扩展）
  * IRC/Slack 聊天机器人
  * 编辑器（用 GUI 或者 curses），以 mal 作为脚本/扩展语言
  * 象棋或者围棋的AI
* 实现一些本指南中未提到的功能，一些参考：
  * 命名空间 (Namespaces)
  * 多线程支持
  * 带有行号或堆栈信息的报错
  * 惰性序列
  * Clojure-style 的协议
  * 完整的 call/cc (call-with-current-continuation) 支持
  * 显式的 TCO (例如`recur`) 带有尾部错误检查
