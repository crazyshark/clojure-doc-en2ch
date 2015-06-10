# 组织Clojure代码 #

> 命名空间是将Clojure代码分包在逻辑组中的一种组织形式，类似于java中的包（package）或其他语言中的模块（module）。几乎所有的Clojure源文件的开头都有一段使用 ns 宏的 命名空间描述 。下面是一个命名空间描述的示例：

```clj

(ns clojure.contrib.gen-html-docs

(:require [clojure.contrib.duck-streams :as duck-streams])

(:use (clojure.contrib seq-utils str-utils repl-utils def prxml))

(:import (java.lang Exception)

(java.util.regex Pattern)))
```

> 从根本上来说，一个命名空间就是一个Clojure映射。映射的键即是Clojure的符号，而映射的值则可以是Clojure的变量或java的类。Clojure的编译器使用这些映射来找出源代码中各个符号的意义。特殊的函数可以允许你添加、删除或更改这些命名空间映射中的条目。

## 命名空间基础 ##

**ns** 宏有一系列的选项以配置一个命名空间。因此在你应用ns宏之前应当理解它所基于的低层函数。

### 用 “in-ns” 切换命名空间 ###

> 当你在Clojure REPL环境下工作时，REPL将提示你正在一个特殊的命名空间里。Clojure总是在 user 命名空间中开始工作：

```clj

user=>
```

> 你所定义的任何符号都将在 **user** 命名空间中创建。你可以使用 “in-ns” 函数切换到另一个不同的命名空间里。

```clj

user=> (in-ns 'greetings)

#<Namespace greetings>

greetings=>
```

> “**in-ns**” 带一个参数，函数将切换到该参数命名的命名空间，并在该命名空间不存在是创建。请注意，在上例中，关键字greetings 前加了一个单引号，这是为了阻止Clojure在编译中尝试去解释它。


### 引用其他命名空间 ###

> 一个新建的命名空间中并不含有任何符号，甚至连Clojure最基本的核心函数都没有。如果尝试调用一个Clojure的内置函数，你将得到一个错误：

```clj

greetings=> (println "Hello, World!")

java.lang.Exception: Unable to resolve symbol: println in this context
```

> Clojure的内置函数是在Clojure.core 命名空间中被定义的，你可以在你的新命名空间中使用斜线以引用它们的命名空间：

```clj

greetings=> (clojure.core/println "Hello, World!")

Hello, World!

nil
```

> 为了避免在代码中的每一个符号前添加斜线，你可以使用refer函数引用其他的命名空间：

```clj

greetings=> (clojure.core/refer 'clojure.core)

nil
```

> 现在你可以直接调用Clojure.core中的函数而不必在使用斜线：

```clj

greetings=> (println "Hello, World!")

Hello, World!

nil
```

**refer**函数带一个符号参数并将所有的命名空间的公有符号与当前命名空间中的符号映射起来（我们将在下面的章节“公有与私有变量”中详细说明它们的区别）符号仍旧与它们原先的命名空间映射在一起。在上例中，通过调用**refer**函数，我们建立了一个从符号greetings/prinln 到变量#’Clojure.core/println的 命名空间映射 。

**refer**函数有一个作为特殊筛选器配置的可选参数。可选参数采用列表或符号映射关键字的格式。:exlude选项后面带一个不会被引用到当前命名空间的符号列表。例如下面的代码片段：
```clj

(refer 'clojure.core :exclude '(map set))
```

> 这段代码将引用除了map和set之外Clojure.core中的其他所有符号。接下来你可以定义自己的map和set，而不会与Clojure.core中原先定义的符号相冲突。

**:only**选项正好与**:exlude**相反，只有符号列表中列出的符号才会被引用到当前的命名空间中。例如：
```clj

(refer 'clojure.core :only '(println prn))
```

这段代码将只会引用Clojure.core中的println和prn符号，而其他在Clojure.core中的符号将仍旧保持着原先的命名空间限制。比如Clojure.core/def。

> 最后，refer同样允许你重命名引用的符号。通过包含 :rename关键字加上一个原命名空间与现命名空间的映射：

```clj

(refer 'clojure.core :rename {'map 'core-map, 'set 'core-set})
```

> 这段代码讲引用Clojure.core中的所有符号，但符号Clojure.core/map在当前命名空间中以core-map启用，而Clojure.core/set则是core-set。则在你希望定义新版本的内置函数的同时希望调用旧版的函数时非常有用。

> 作为从其他命名空间中复制映射的替代，你可以在当前命名空间中创建一个别名。在调用**alias**之后，你可以使用local-name引用其他命名空间中的符号。命名空间的别名使用**alias**函数创建。

(alias local-name namespace-name)

> 参数localname和namespace name都是（引用的）名字。alias在当前命名空间创建了一个别名。在调用**alias**，你可以使用local-name引用其他命名空间中的符号，而不是原命名空间中的完整名字。例如下面的代码：
```clj

greetings> (alias 'set 'clojure.set)

nil

greetings> (set/union #{1 3 5} #{2 3 4})

#{1 2 3 4 5}
```

## 载入其他命名空间 ##

**refer** 与 **alias** 允许你引用已存在的命名空间中的符号，但当命名空间是在其他文件中定义的该怎么办？包含其他的文件并不能加载这些命名空间。Clojure提供了一系列的函数以从其他文件中加载代码

### 从文件或流中加载 ###

> 最简单的加载函数是\*load-file

```clj

(load-file name）
```

**Load-file**带一个参数，并尝试去加载文件中所有的Clojure格式。文件名以一个字符串的形式给予。它包含所有的字典并由当前工作路径的上下文解释。在一个Unix类型的操作系统中，它看起来像：

```clj

(load-file "path/to/file.clj")
```

> 而在windows中，反斜杠必须被转义，因为路径是一个字符串：

```clj

(load-file "C:\\Documents\\file.clj")
```

> 如果你希望从其他来源加载代码，比如一个网络连接，你可以使用load-reader函数，它将使用java.io.Reader作为参数并加载解释从Reader中获得的代码。

### 从类路径中加载 ###

> Java虚拟机使用一个被称为类路径的特殊变量。作为加载可执行代码的一个路径列表。Clojure程序同样适用类余烬以查找资源代码。

> 类路径通常在Java命令行中作为一个路径和JAR文件的集合被指定。下面这个在Unix类型操作系统中的例子创建了一个由Clojure JAR文件与/code/sources目录组成的类路径：

```clj

java -cp clojure.jar:/code/sources clojure.main
```

> Java开发环境和编译管理工具通常有它们自己的方法以配置类路径。参阅你所使用的工具文档以获得更多信息。

### 命名空间名与文件名 ###

> Clojure命名空间与Java的包具有相似的命名惯例：它们根据小数点划分成有层次分明的组织。一个流行的惯例是在你使用你的域名来命名你的库时，你会反转域名的格式。因此当你为**www.example.com**工作时，你的命名空间可能被命名为 **com.example.one**，**com.example.two**，以此类推。

> 当要把命名空间翻译成文件名的时候，小数点(.)将变成路径的分隔符(/)而连接号(-)将变成下划线(`_`)。所以在一个Unix类型的系统，Clojure命名空间com.example.my-cool-library会被定义在文件 com/example/my`_`cool`_`library.clj中。为了加载这个命名空间，包括com的路径必须在类路径中。

### 从类路径中加载资源 ###

> load函数带任意个字符串参数，任何一个参数都将命名一个在类路径中的资源。一个资源名类似与一个文件名，但是不包含.clj后缀。如果资源名以一个斜杠(/)开头，它将被理解为在类路径中的某个文件夹中。例如下面的代码：

```clj

(load "/com/example/my_library")
```

> 这个调用将在类路径中的每个位置查找/com/example/my\_library.clj文件。（也会查找预编译类文件/com/example/my\_library.class。编译将在第十章中进行详细描述）
如果一个load的参数中并没有以斜杠开头，那么它将把理解为在当前的命名空间的文件夹中。

```clj

greetings=> (load "hello")
```

> 这个调用将会从greetings命名空间中加载，将会在类路径中查找greetings/hello.clj文件。

### 从类路径中加载命名空间 ###


> 在通常的编码中你将极少用到**load**函数，作为替代，Clojure提供了两个更高层的函数以加载命名空间，**require** 和 **use**。

```clj

(require 'com.example.lib)
```

> require函数带任意个参数，每一个参数都是一个符号，库指定向量，前缀列表或一个flag。参数以单引号开头以避免被解释。最简单的例子：一个符号，把符号转换成文件名，在类路径中查找这个文件然后加载它，验证所给的名字是一个命名空间，之后创建。

> 这个例子将会从类路径中加载com/example/lib.clj文件。在该文件加载完毕后，如果命名空间com.example.lib不存在，require函数将抛出一个异常。如果该命名空间已经存在，require函数将忽略它。

> require的指定库参数将允许你指定一个选项以加载命名空间。参数是向量的格式，以一个符号开始，之后是一个关键字选项。唯一的一个可接受的选项是 :as ，用来为命名空间创建一个本地化别名。

```clj

(require '[com.example.lib :as lib])
```

> 这个例子将会加载com.example.lib命名空间，并为它在当前命名空间中创造一个lib的别名。

> 通常很多命名空间享有一个共同的前缀。如果是这样，你可以使用前缀列表以加载一系列的命名空间。前缀列表是一个以所有命名空间公有的符号开始的列表，之后是每个命名空间名称的剩余部分。例如下面的写法：

```clj

(require 'com.example.one 'com.example.two 'com.example.three)
```

> 等同于这样写：
```clj

(require '(com.example one two three))
```

> 前缀列表和指定库可以连写，比如下面的例子：

```clj

(require '(com.example one [two :as t]))
```

> 这个例子将会加载命名空间com.example.one和com.example.two，并且为com.example.two创建一个别名 t。

> 最后，require函数接受任意个数的flag，在参数的任意位置作为关键字使用。:reload flag 使 require函数加载所有参数中的命名空间，即使是它们已经被加载。例如下面的代码：

```clj

(require 'com.example.one 'com.example.two :reload)
```

> 另一个flag，:reload-all，将重载所有列出的命名空间与所有从命名空间中require进来的附属命名空间。在你在REPL中尝试并且想要加载在代码中的改动时，:reload 和:reload-all将会很经常的用到。

:verbose flag将打印require函数创造的低级函数调用所返回的调试信息：

```clj

user=> (require '(clojure zip [set :as s]) :verbose)

(clojure.core/load "/clojure/zip")

(clojure.core/load "/clojure/set")

(clojure.core/in-ns 'user)

(clojure.core/alias 's 'clojure.set)

nil
```

### 一步加载与引用命名空间 ###

你通常会想要require一个命名空间并且同时 refer 里面可靠的符号。use函数提供了一个“一步到位”的做法。调用use函数等于在调用了require函数之后再调用refer函数。Use函数接受require函数的 :reload, :reload-all和 :verbose flag，与refer函数的:exclude, :only和 :rename选项，影响它们命名空间中被群组的向量。例如，考虑下面的代码：

```clj

(use '[com.example.library :only (a b c)] :reload-all :verbose)
```

这个例子将加载(重载)com.example.library命名空间并在当前命名空间中引用符号a ，b和c。注意，在这里你并不需要为列表（a b c）加上单引号因为整个向量已经加上了单引号。


---

> 警告：除了在REPL中实验之外，使用use来加载一个命名空间而不使用:only来限制引用是一个很不好的做法。调用use而不用 :only会让代码的读者不知道个别符号是从哪来的，并且在被use的命名空间更改后可能会引起一个预期外的冲突

---


## 导入Java类 ##

> 最后一个命名空间函数是用来处理java类的。你总是可以用一个java类的完全名来引用他，类似于java.util.Date。为了在不包括它的包的情况下引用它，你可以导入它。

```clj

user=> (import 'java.util.Date) nil

user=> (new Date)

#<Date Fri Oct 23 16:31:28 EDT 2009>
```

> 在Clojure1.0中，import是一个函数，因此你必须像例子中的一样把它的参数用括号括起来。而在Clojure1.1中，import是一个宏，所以它的参数不需要添加括号。与require和use类似，import同样和接受前缀列表。前缀必须是完整的java包名称，类名不允许包含点号。

```clj

(import '(java.util.regex Pattern Matcher))
```

> 作为一个特例，Java中的嵌套类（有时被称为内部类）必须使用它们的二进制类名，就像JVM使用的那样。内部类的二进制类名由外部类名组成，之后跟着内部类名。例如，类Wheel的内部类Truck的二进制名是Truck$Wheel。

在Clojure中，一个java的嵌套类除了它的包含类之外不能被命名。例如，要导入一个 java的嵌套类javax.swing.Box.filler，你必须这样做：

```clj

(import '(javax.swing Box$Filler))
```

之后你可以使用Box$Fille引用这个类了。

## 把它们综合起来：命名空间描述 ##

> 当在编写一个普通的Clojure代码时，你可能不会使用 **in-ns** ，**refer**，**alias**，**load**，**require**，**use**和**import**函数。作为替代，你也许会使用ns宏来添加一个命名空间描述作为你的Clojure代码的定义性开头，就像在本章开头的示例中的那样。

```clj

(ns name & references)
```

ns宏取一个符号作为它的第一个参数，它以这个名字创建一个新的命名空间并把它设定为当前的命名空间。ns是一个宏，所以它不会解释它的参数，所有不必在名字参数前加单引号。

剩下的参数与refer，load，require，use和import函数采用相同的格式，只有两处不同：

  1. 参数前不用加单引号。
  1. 函数名称以关键字的方式提供。

> 下面是一个示例：

```clj


(ns com.example.library

(:require [clojure.contrib.sql :as sql])

(:use (com.example one two))

(:import (java.util Date Calendar)

(java.io File FileInputStream)))
```

> 这段代码将新建一个新的命名空间com.example.library，并且自动引用clojure.core命名空间。加载Clojure.contrib.sql命名空间并创建一个sql的别名。加载com.example.one 和com.example.two并在当前命名空间中引用它们中的所有符号。最后，它导入java类Date，Calendar，File 和FileInputStream。

> 与 in-ns不同，ns宏自动引用Clojure.core命名空间作为前提。如果你想要控制是哪个核心符号将被引用到你的命名空间，在ns中使用:refer-clojure参数，就像这样：

```clj

(ns com.example.library

(:refer-clojure :exclude (map set)))
<code>

:refer-clojure和(refer ‘clojure.core)一起使用时取相同的参数。如果你不想引用任何来自于clojure.core中的任何符号，把:only所列出的列表留空，就像：

<code language="clj">
(:refer-clojure :only ())
```

## 符号和命名空间 ##

> 作为一个前提，命名空间本质上是变量到符号上的映射，但他们在独立属性上有一些小区别：符号可以有将它们绑定在特殊命名空间上的属性。

### 命名空间元数据 ###

> 与其他Clojure对象相类似，命名空间可以拥有附属于它们的元数据（详见第八章）。你可以通过在ns宏中放置读取时元数据来为命名空间添加元数据。就像这样：

```clj

(ns #^{:doc "This is my great library."
:author "Mr. Quux <quux@example.com>"
com.example.my-great-library)
```

> Clojure并没有为命名空间的元数据添加“官方的”特殊键（就像函数中:tag 和:arglist），很多Clojure库的开发者们达成了以下惯例：使用:doc元数据来描述命名空间的通常作用，使用:author元数据来表示作者的名字和email地址。

### 预描述 ###

> Clojure编译器要求符号在使用前必须被定义。通常这指导你这样组织你的代码：把简单的，低层级的函数放在代码前面部分，复杂的函数放在代码靠后部份。但是有些时候，你需要在一个符号被定义之前使用它。为了避免编译器抛出一个异常，你必须使用预描述。

```clj

(declare & symbols)
```

> 一个预描述通过declare宏创建，它将会简单的告诉编译器：“这个符号是存在的，它将在后面被定义。”以下是一个投机取巧但一无是处的例子：

```clj

(declare is-even? is-odd?)
(defn is-even? [n]

(if (= n 2) true

(is-odd? (dec n))))

(defn is-odd? [n]

(if (= n 3) true

(is-even? (dec n))))
```

### 符号与关键字的命名空间限制 ###

> 简单来说，与你看到的一样，符号将被命名空间限制起来。函数name 和namespace返回符号各个部分的描述字符串。

```clj

user=> (name 'com.example/thing)

"thing"

user=> (namespace 'com.example/thing)

"com.example"
```

> 注意，符号前加了引号以防止Clojure将它们转化为一个类或变量。

> namespace函数在遇到未被限制的符号时返回nil，因为它们没有命名空间。
```clj

user=> (namespace 'stuff)

nil
```

> 同样的，关键字也可以被命名空间限制。函数name 和namespace对于关键字起与符号同样的作用：
```clj

user=> (name :com.example/mykey)

"mykey"

user=> (namespace :com.example/mykey)

"com.example"

user=> (namespace :unqualified)

nil
```
> 作为一个语法上的方便，若要在当前命名空间中创建一个关键字，你可以在它们的名字前打两个冒号以代替一个。在”use”命名空间中，关键词::thing扩展到:user/thing.
```clj

user=> (namespace ::keyword)

"user"
```
> 效果还不够直接，反冒号 ` 宏可以被用来在当前命名空间中创建一个限制符号：
```clj

user=> `sym

user/sym
```

### 构造符号和关键字 ###

> 函数name 和namespace将符号和关键字转化成字符串，而symbol 和keyword则是另一种做法：给一个字符串作为名字（命名空间可选）然后构造一个符号或关键字。

```clj

user=> (symbol "hello")

hello

user=> (symbol "com.example" "hello")

com.example/hello

user=> (keyword "thing")

:thing

user=> (keyword "user" "goodbye")

:user/goodbye
```

> 注意，传给keyword函数的名字并不包括前面的冒号。

### 公有变量和私有变量 ###

> 默认的，在命名空间里的所有定义都是公开的，表示它们能够被其他命名空间自由引用，和被refer和use拷贝。但是很多命名空间被分为两部分，一部分是永远不允许被其他命名空间调用的内部函数，另一部分是可以被其他命名空间调用的公开函数。大概相当于面向对象语言中的公有和私有概念，例如在java中。

> 在Clojure中的私有变量永远不能被refer 和use所拷贝，也不能被一个命名空间限制的符号所引用。事实上，它们只能在定义它们的命名空间中被使用。

> 有两个创建私有变量的方法。一是defn-宏，用法与defn类似但会创建一个私有函数定义。第二种方法适用于任何定义，在定义符号时添加:private元数据。

```clj

(def #^{:private true} *my-private-value* 123)
```

> 注意：私有变量并不是真正隐藏。在任何代码中通过调用(deref (var namespace/name))即可获取该变量的值。但是私有变量保护你的变量不会在代码的其他不希望被调用的地方被轻易的调用。

## 命名空间进阶使用 ##

> 与java中包的简单命名手段不同，Clojure的命名空间是第一类对象，可供函数查询和控制。

### 查询命名空间 ###


> 特殊变量**ns**总是与当前命名空间绑定在一起。会被in-ns更改。

> 函数all-ns没有参数，返回当前定义的所有命名空间。


**注意：**命名空间的集合是全局的。你不能在同一个JVM中拥有多个Clojure实例以加载不同的命名空间。因为Clojure只是一个编译器而不是像Jython或JRuby那样的解释器，所以在这里谈论Clojure的实例化并没有意义。你可以使用Java classloader来创建一个独立的可执行上下文，但是这超出了本书的讨论范围。


> 两个函数可以帮助你从一个命名空间对象中取得命名空间的命名符号。Find-ns函数有一个符号参数，将返回以该参数命名的命名空间。当该命名空间不存在时返回nil。

> 你通常不会关心你到底是和一个命名空间对象本身或只是和一个命名它的符号打交道，如果是这样，函数the-ns将同时接受一个命名空间对象（直接返回该对象）或一个符号（这时它将调用find-ns函数）。与find-ns函数不同，the-ns函数在命名空间不存在时抛出一个异常。很多函数在这个部分都是在它们的参数中调用了the-ns函数，所以它们可以同时被一个命名空间对象和一个加了引号的符号所调用。

> ns-name 函数把命名空间的名称作为一个符号返回

> ns-aliases函数返回一个从符号到命名空间映射，描述所有的在命名空间中定义的命名空间别名。

> ns-map 函数返回一个从符号到对象（变量或类）的映射，描述在命名空间中的所有映射。通常这里有比你想要的还要多得多的信息，因此Clojure提供了一系列辅助函数以返回这些命名空间映射的子集。ns-public返回所有公有变量的映射；ns-interns返回包括公有和私有所有变量的映射；ns-refers返回所有从其他命名空间中引用的符号映射；ns-import返回从所有java类的映射。

> 例如，要获取命名空间clojure.core中所有公有符号的列表，你可以运行：

```clj

(keys (ns-publics 'clojure.core))
```

> 最后，也许你想知道一个符号当遇到特殊的上下文环境会后变成什么，ns-resolve函数取一个命名空间和一个符号作为参数，返回这个符号在命名空间中映射到的变量或类。

> 例如，命名空间clojure.core导入了java.math.BigDecimal类，你可以通过调用以下代码来寻找答案：

```clj

user> (ns-resolve 'clojure.core 'BigDecimal)

java.math.BigDecimal
```

> 作为一个快捷用法，函数resolve在当前命名空间中等同于ns-resolve。

### 控制命名空间 ###

> 函数in-ns和ns宏都创建了一个命名空间并把它设为当前的命名空间。

> 同样的，def和相关函数都对当前命名空间其影响。这里有特殊情况，比如在代码完善阶段，你希望建立一个命名空间并在里面定义对象但并不切换到里面。你也许会尝试着这样写：

```clj

;; Bad code!

(let [original (ns-name *ns*)]

(ns other)

(defn f [] (println "Function f")

(in-ns original)))
```

> 这并不会工作。因为Clojure在对ns表求值之前已经读取了当前命名空间中的符号f。所以你将在当前的命名空间而不是其他命名空间中结束对f的定义。

> 你可以使用create-ns函数作为替代，它带一个符号参数并返回一个以该参数命名的新的命名空间，若该名字的命名空间已存在则返回已存在的命名空间。之后你可以使用intern函数在该命名空间中定义变量。这是前面代码的可工作的版本：

```clj

(let [other-ns (create-ns 'other)]

(intern other-ns 'f

(fn [] (println "Function f"))))
```

> 在一个命名空间中创建并映射到一个符号的做法被称为interning，这恰好是intern函数的作用。

```clj

(intern namespace symbol value)
```

> value 参数是可选的，若忽略，则变量创建时并没有根值，类似于一个预描述。

> symbol参数必须是一个空符号并且没有命名空间限制前缀。namespace参数可以是一个符号或命名空间。

> ns-unmap函数跟intern函数作用相反，它删除命名空间中的一个映射。例如，每一个Clojure的命名空间，无论它是怎么创建的，命名空间的开始都有与java.lang中的所有类的映射。所以如果你想要完全清空一个命名空间，你可以这么做：

```clj

(let [empty-ns (create-ns 'empty)]

(doseq [sym (keys (ns-map empty-ns))]

(ns-unmap empty-ns sym))

empty-ns)
```

> 最后，remove-ns函数将完全删除一个命名空间，包括interned在里面的所有变量。注意：其他命名空间的代码将仍然在闭包中保留这些变量的引用，但是这些变量本身已经被清除，所以任何对它们的调用都会抛出一个“unbound Var”异常。

### 作为引用的命名空间 ###

> 正如我在本章开头时所描述的那样，一个命名空间实际上是一个从符号到变量或类的映射。我应该更精确的说，它是一个映射的引用，因为一个命名空间是可变的。命名空间上的所有作用是atomic的，就像Clojure Atoms一样。例如当你用defn重定义一个已存在的函数，Clojure 将确保新的和旧的定义永远不会重复。

> 不管怎样，Clojure 并没有提供让命名空间实施协同工作的方法，而你在Refs中可以这么做。如果你重定义许多函数，Clojure并不能保证所有“新”函数都同时被更新。新的定义和旧的定义将同时存在一小段时间。

> 一般说来，在正在运行的程序中热切换整个模块的问题是非常棘手的，需要最底层的语言支持。例如Erlang就是以支持模块的热切换为基础设计的。但Java本身并没有内置的热切换的支持，尽管一些java服务器正在尝试去实现。

## 总结 ##

> 你可以在命名空间的基础上做很多事，它们看起来是绝对应该放在第一位去做的。但是通常说来，在日复一日的编码中，你只需要一点特性和惯例。

> 首先，在你的每一个源文件开始处添加命名空间描述，用ns，:import，:use表达式描述类和它依赖于的命名空间。总在:use时添加:only选项以清晰的描述你需要从其他命名空间中引用的符号。这是一个完整的示例：

```clj

(ns com.example.apps.awesome

(:use [clojure.set :only (union intersection)]

[com.example.library :only (foo bar baz)]

[com.example.logger :only (log)])

(:import (java.io File InputStream OutputStream)

(java.util Date)))
```

> 不要只因为它们是clojure.core的一部分而惧怕重用那些好的名字，在需要时向ns中添加:refer-clojure表达式。

> 结构化你的源文件以避免预定义，这通常意味着把简单的，低层级的定义放在代码前面部分，而复杂的，基于前者的定义放在代码靠后部份。