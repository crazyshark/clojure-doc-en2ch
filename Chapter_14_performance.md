# Chapter 14 性能 #

> 在原理上，Clojure能够跟java一样快:两者都是编译成java字节码，运行在java虚拟机上面。Clojure的设计小心地避免了某些特性－－比如连等式，或者类似Common Lisp的条件系统－－这些在jvm上面会严重地危害性能。但Clojure始终是一门年青的语言，并没有耗费成千上万小时去优化编译器。这样的结果是，Clojure的代码通常比等价的java代码运行得要慢些。然而，通过某些细
微的调整，Clojure的性能表现能被带到接近java的水平。不要忘记java从代码层面来说，在性能临界上的表现总是可靠的。

## JVM上概要分析 ##

---

> 评估任何程序语言或者算法性能的首要原则就是：测试！

> 不要假设一项技术必然更快因为它似乎拥有更少的步骤或者采用更少的变量。但这在现代的JVM比如Hotspot上面是对的，JVM在程序运行的过程中不断地估算代码的性能，动态地重新编译临界区。

> 所谓的微基准，即在隔离条件下测量某个单独操作，在当前的环境下是没有意义的。同样，以JVM初始化启动时间做为标准的测量也是没有意义的（这个错误通常发生在对比java和C++时）。现代的JVM通常针对吞吐量进行优化，最大化了操作的总量，哪怕是有可能要花很长一段时间来执行的操作。


### JAVA性能的一般技巧 ###

---

> java虚拟机有一堆可以影响性能的选项。首先，JVM要区分“客户端”和“服务端”模式，“服务端”模式的性能全面超越“客户端”（代价是更长的启动时间）。

> 其次，Java堆空间的大小和选择不同的垃圾收集器策略同样影响性能。这对于Clojure来说尤为真实，因为比起Java来，Clojure对常量的频繁使用会占用更多的堆空间，也会给垃圾收集造成更大的压力。

> 在现代JVM中有更多可调整的参数会影响性能。你要确保很熟悉虚拟机提供给你的“性能开关”，并且尝试着观察它们如何影响你实际应用的性能。


### 采用Time进行简单性能测试 ###

---

> Clojure拥有一个内建的简单性能测试工具，time宏。time接受一个简单表达式，分析它，然后打印出执行表达式将耗费多少毫秒：
```Clojure

user=> (time (reduce + (range 100)))
"Elapsed time: 1.005 msecs"
4950
```
> 就像我们之前提到的，这个测试宏在JVM上下文中几乎没什么意义。来个稍微好点的测试，测试一下在一个闭合循环中重复执行上万次计算：
```

user=> (time (dotimes [i 100000]
(reduce + (range 100))))
"Elapsed time: 252.594 msecs"
nil
```
> 然而，这依然不能统观全貌，因为JVM可能在执行循环的过程中就对计算进行了优化。多次重复循环就能得到一个更加精确的结果：
```

user=> (dotimes [j 5]
(time (dotimes [i 100000]
(reduce + (range 100)))))
"Elapsed time: 355.759 msecs"
"Elapsed time: 239.404 msecs"
"Elapsed time: 217.362 msecs"
"Elapsed time: 221.168 msecs"
"Elapsed time: 217.753 msecs"
```
> 如同你看到的那样，在这个例子中，数次迭代的执行时间基本上在220毫秒上下波动。这种模式就是JVM的特点。

> 然而，即使拥有了这些信息，你仍然无法准确预期在一个大的应用上下文中，(reduce + (range 100))这个计算的性能如何。只能执行更多的测试来验证了。

> 同样地，我们也得意识到延迟序列（lazy sequences）对于性能的影响。如果你用来测试的表达式采用了延迟序列（比如：map），time宏可能就只会给出初始化序列的时间。为了计算出获得整个序列的时间，你必须采用doall。但doall在复杂的数据结构中很难执行，而且获得的结果并不代表数据结构实际使用的性能。


### 采用java性能工具 ###

---

> 因为Clojure被编译成java字节码，java的性能工具同样可以在Clojure中使用，但是关于这些工具的讨论已经超出了这本书的讨论范围。

> 最佳的经验法则如下：采用最直接、最简单的方式编写你的代码，然后测试看看是否达到预期性能。如果没有，采用性能工具去确定影响性能的关键点，然后迁移或者重写这些点上的代码直到满足你的性能需求。接下来的几页会描述一些分析性能关键点的技术。

## 备忘 ##

---

> 有一项能够加速大型复杂函数的简单技术叫做备忘（memoization），这是某种形式的缓存。当每次调用函数时，一个备忘函数会在table中储存被调函数的输入参数以及返回值。如果这个函数以同样的输入参数被再次调用时，它返回的就是储存在table中的值而不需要再次进行计算。

> Clojure内建支持备忘的函数是memoize，它接收一个函数做为参数，返回这个函数的备忘版本:
```

(defn really-slow-function [x y z] ...)
(def faster-function (memoize really-slow-function))
```
> 备忘是一个牺牲内存加速执行的经典例子。如果一个函数需要花费比哈希表寻址更多的时间去计算结果，而且它又经常以同样的输入参数被调用，那么采用备忘就是一种好的候选方案。只有纯函数－－意即针对特定输入总是返回同样结果的函数－－能够进行备忘。


## 反射和类型提示 ##

---

> 就像你所知的那样，java是种静态类型语言：它是在编译时确定所有对象的类型。而Clojure是动态类型，意即某些对象的类型一直要到运行时才会确定。

> 要在静态类型的java上面实现动态类型的函数调用，Clojure采用了java的反射特性。反射允许代码在运行时查阅java的类并根据名字来调用方法。然而，通过反射调用方法比直接调用编译后的方法要慢上很多。

> Clojure允许你为符号和表达式加上类型提示以避免编译器通过反射调用方法。类型提示采用:tag关键字通过读取时元数据来进行声明。一个类型提示的symbol可以写成`#^{:tag hint} symbol，通常简写为`#^hint symbol。类型提示是一个java的类名。（这个类名会同样是个字符串，很少被需要去处理java费解的互用性问题）

> 为了找出一个方法是反射调用还是不是，设置编译器属性`*warn-on-reflection*`为真，然后，执行代码，如果遇到反射调用，Clojure的编译器就会打印出一条警告信息。
```

user=> (set! *warn-on-reflection* true)
true
user=> (defn nth-char [s n]
(.charAt s n))
Reflection warning - call to charAt can't be resolved.
```
> 只要在调用的方法中对符号添加类型提示，这个警告通常就被消除掉了。对函数的参数或者采用let的本地绑定都有效果。
```

;; No reflection warnings:
user=> (defn nth-char [#^String s n]
(.charAt s n))
#'user/nth-char
user=> (defn nth-char [s n]
(let [#^String st s]
(.charAt st n)))
#'user/nth-char
```
> 在java方法采取不同类型参数重载的情况下，更增加了类型提示的必要性。举个例子，String.replace方法可以接收char 或者 `CharSequence`参数。你不得不在三个参数上都加类型提示来避免反射。
```

user=> (defn str-replace [#^String s a b]
(.replace s a b))
Reflection warning - call to replace can't be resolved.
#'user/str-replace
user=> (defn str-replace [#^String s
#^CharSequence a
#^CharSequence b]
(.replace s a b))
#'user/str-replace
```
> 注意类型提示并非强制类型转换，它并不能把一种数据类型转换成另一种。采用错误的参数类型调用一个类型提示方法会抛出一个运行时错误：
```

user=> (str-replace "Hello" \H \J)
java.lang.ClassCastException: java.lang.Character cannot be cast to java.lang.CharSequence 
```
> 同样，注意一个不正确的提示类型会导致一个反射警告：
```

user=> (defn str-replace [#^String s #^Integer a #^Integer b]
(.replace s a b))
Reflection warning - call to replace can't be resolved.
#'user/str-replace
```
> 你可以在函数定义时为Var加一个类型标记，为函数的返回值做类型提示。这在任何Var上都起作用，包括那些做为全局值使用的。
```

user=> (defn greeting [] "Hello, World!")  ;; no type hint
#'user/greeting
user=> (.length (greeting))
Reflection warning - reference to field length can't be resolved.
13
user=> (defn #^String greeting [] "Hello, World!")
#'user/greeting
user=> (.length (greeting))  ;; no reflection warning
13
user=> (defn greeting {:tag String} [] "Hello, World!") ;; same as above
```
> 在极少的情况下，符号的类型提示都无法有效地避免反射。在这样的情况下，你可以为整个表达式加上类型提示：
```

user=> (.length (identity "Hello, World!"))
Reflection warning - reference to field length can't be resolved.
13
user=> (.length #^String (identity "Hello, World!"))
13
```
> Clojure的编译器在跟踪对象类型上面非常聪明。比如，java方法的返回类型总是能够被获知而无须类型提示。只要给予少少的提示，编译器就能够推演出大部分所需的其他类型信息。一般而言，你应该先写出没有任何类型提示的代码，然后打开`*warn-on-reflection*`参数，仅仅在需要提升性能的地方加上类型提示。


## 使用基本类型 ##

---

> java的类型系统并非百分百地面向对象；它支持以下这些基本类型：boolean，char，byte,short，int，long，float和double。这些基本类型并不适合java的类体系，当方法中需要把基本类型做为对象使用时，就必须把它们装箱成 Boolean, Character, Byte, Short, Integer, Long, Float, 和Double类。从java1.5开始，java编译器就能够在需要时对基本类型进行自动装箱和拆箱。

> 在Clojure中，一切都是对象，所以数字总是被装箱过的。这个可以在检查简单算术的结果时看到：
```

user=> (class (+ 1 1))
java.lang.Integer
```
> 然而，在JVM中，操作装箱的数字比不装箱的基本类型要来得慢。所以，在数学计算密集的应用中，采用装箱数字的Clojure代码要比直接采用基本类型的java代码要慢一些。

### 循环中的基本类型 ###

---

> Clojure在循环体内支持基本类型。在loop或者let绑定的vector中，你可以采用boolean,char,byte,short,int,float和double这些函数来强制使用基本类型。这儿有个例子，是采用欧几里德的算法来计算两个整数的最大公分母：
```

(defn gcd [a b]
(loop [a (int a), b (int b)]
(cond (zero? a) b
(zero? b) a
(> a b) (recur (- a b) b)
:else (recur a (- b a)))))
```
> 在初始化loop的vector时，参数被强制转换成int。这个版本比不采用基本类型的版本大约要快12倍。但是，请注意！假设你采用mod（模计算）函数来实现同样的算法，采用基本类型的代码会更慢。因为Clojure函数（除了运算函数）的参数总是会自动装箱。因此，当使用循环中基本类型时，你应该仅调用以下的基本类型识别函数：
  * 运算函数：+，－，`*`，/
  * 比较函数： ==, <, >, <=, 和 >=
  * 断定函数：pos?, neg?, 和 zero?
  * 参数和返回值都是基本类型的java方法
  * 没有类型检查的算法函数（将在以下章节中谈到）

> 注意，通用的＝函数不在这个列表中。与之替代的，采用了仅能作用于数字的＝＝函数。在未来的发布版本中，已经计划在全部的Clojure函数中，包括用户自定义函数，支持所有的基本类型。

> 在采用基本类型操作时，数值字面量同样也会被强制转换为基本类型。举个例子，下面的代码计算了1到100整数的和：
```

(let [max (int 100)]
(loop [sum (int 0)
i (int 1)]
(if (> i max)
sum
(recur (+ sum i) (inc i)))))
```
> 循环变量sum和i的初始值必须采用int函数强制转换为基本类型。对max的强制基本类型转换在循环外进行的，因为它只需要赋值一次。如果你采用字面数字100替代本地变量max，代码依然有效，但是不会象之前那么快了。

### 无类型检查的整数运算 ###

---

> Clojure的基本运算被定义为类型安全的，意即如果操作的结果大于返回值类型的长度，就会抛出一个错误。举个例子，下面这个循环计算2的64次方，就会抛出一个异常：
```

user=> (let [max (int 64)
two (int 2)]
(loop [total (int 1), n (int 0)] 
(if (== n max) 
total 
(recur (* total two) (inc n)))))
java.lang.ArithmeticException: integer overflow 
```
> 然而，某些特定的算法（比如：散列法）在整数运算中是允许悄然溢出的。Clojure提供了一系列函数用于象java算术操作一样准确执行整数运算。它们都接收Integer或者Long参数，并且支持基本类型。

> 以下这些函数都有整型溢出的倾向：unchecked-add, unchecked-subtract,unchecked-multiply, unchecked-negate, unchecked-inc, 和 unchecked-dec。
```

user=> (unchecked-inc Integer/MAX_VALUE)
-2147483648
user=> (unchecked-negate Integer/MIN_VALUE)
-2147483648
```
> > unchecked-divide 和 unchecked-remainder函数会把结果截断为整型。
```

user=> (unchecked-divide 403 100)
4
```
> > 无类型检查的操作在使用循环基本类型时，比标准的算术函数要稍快一些。但是，在你使用无类型检查的函数前，要先想好能否接受没有安全检查所导致的结果。

### 基本数组 ###

---


> 从Clojure1.1开始，你能够为数组声明基本类型提示：`Java的boolean[], char[], byte[], short[], int[], long[], float[], 和 double[]数组能被分别标识为#^booleans, #^chars, #^bytes, #^shorts, #^ints, #^longs, #^floats, and #^doubles。（同样可以采用#^objects 标识 Object[]）`

> 有类型提示的数组可以采用aget 和 aset进行基本类型操作，而没必要采用类型强制的注入函数比如aset-int和aset-double；实际上，这些类型强制的函数比aset在操作类型提示的数组时要更慢。对于aset来说，新添加的值必须总是正确的基本类型。对于aset和aget而言，数组的下标必须是基本类型int。在保留函数式编程方式的情况下操作基本类型数组，amap和areduce宏（Chapter10中有描述）无疑是更好更快的选择。

## 瞬态（Transients） ##

---

> 就像你已经了解到的那样，Clojure中的数据结构是不可变且持久化的，这样就保障了在多线程中的并发安全性。但是如果你有一个明确知道只会应用于单线程中的线程又怎么样呢？是否还是需要付出不可变/持久化的性能代价呢？Clojure的答案是：不！

> Clojure1.1引入了transients，这是一个临时的、可变的且性能最优化的数据结构。当你通过一系列步骤构建一个巨大的数据结构时，它显得尤为有用。

> transients的一个关键特性是它不用你改变代码中任何函数式编程风格。最重要的是，它并非真正给你一个可变的数据结构（如同java的collection实现类那样）让你可以肆意妄为。transients的可变特性主要在实现细节上。

> 可以通过以下的例子对transients来作出最好的解释，下面的代码创建了一个map，将ASCII字符转化为对应的十进制值：
```

(loop [m {}, i 0]
(if (> i 127)
m
(recur (assoc m (char i) i) (inc i)))) 
```
> 以下是采用transients循环的同样实现：
```

(loop [m (transient {}), i 0]
(if (> i 127)
(persistent! m)
(recur (assoc! m (char i) i) (inc i)))) 
```
> 注意那些非常细小的改动。这个例子展示了采用transients所需的三处代码改动：
  * 采用transient函数初始化一个transient版本的数据结构。transient函数支持Vectors, hash maps, 和 hash sets。
  * 将conj, assoc, dissoc, disj, 和 pop替换成它们对应的transient版本：conj!, assoc!, dissoc!, disj!, 和 pop! 。
  * 在结果调用persistent!来返回一个普通的持久化数据结构。

> transients的一个重要特性是，不管输入的大小，transient和persistent!函数总是在固定的时间运行。因此，采用transient函数初始化一个大的数据结构，然后采用transient特有的函数去操作它，最后在返回前调用persistent!，这样是非常高效的。

> 记住，transients并非如同java collection般的可变数据结构。就像持久化数据结构一样，你必须采用任何函数的返回值来做为改变了的数据结构。下面这个指令式的代码是不会生效的：
```

;; bad code!
(let [m (transient {})]
 (dotimes [i 127]
(assoc! m (char i) i))
(persistent! m)) 
```
> > dotimes宏创建一个指令式的循环，在每次迭代中，assoc!的返回值被丢弃了。这段代码的实际结果是无法预测的，但通常都是错的。


> 通常而言，transients被用于单一函数内部或者loop/recur代码块中。它们可以被传递给其他函数，但必须严格遵守以下限制：
  * 必须执行线程隔离。访问另一个线程的transient结构会抛出一个异常。
  * 在调用了persistent!之后，数据结构的transient版本就已经被废弃了，如果再次尝试访问它，会抛出一个异常。
  * transient结构的中间版本不能被存储或者使用，只有最后一个版本是有效的（这不同于其他持久化的数据结构）。

> transients的优势在于进行数据结构改变操作时比其他持久化的数据结构要快很多。通常，你在任何地方递归创建一个大的数据结构，transients都能提供相当的性能优化。但是，transients的使用总是要限制在单一函数体中，而不能传播到其他代码层面去。

## Var查找 ##

---

> 每次你使用一个Var时，Clojure就会去找到这个Var的值。如果你在循环中重复使用同一个Var，寻值的过程就会大大拖慢代码。为了避免这种性能问题，采用let在循环的范围中把Var的值绑定到本地，如同以下例子：
```

(def *var* 100)
(let [value *var*]
  ... loop using value instead of *var* ...) 
```
> 采用这个技术需要注意的是：它并非总能带来性能提升，而且在未来的发布版本中可能变得不必要。

## 代码嵌入（Inlining） ##

---

> 代码嵌入－－采用编译过的函数体来代替函数调用－－是在编译器中一个常见的优化。Hotspot类型的JVM广泛地将关键性能的代码进行嵌入。

> Clojure对基本类型进行自动嵌入。然而，能够接受数个参数的算术函数，比如+，仅在接受两个参数的情况下可以被嵌入。这意味着以下代码：
```

(+ a b c d)
```
> > 如果写成这样将会更快：
```

(+ a (+ b (+ c d)))
```
> > 尤其是参数的值是基本类型时，速度差异更加明显。这个在未来的发布版本中可能变得不必要。

### 宏和定义嵌入 ###

---


> 从某种意义来说，宏也是代码嵌入的一种，因为它会在编译时被展开。然而，宏不能被高阶函数（比如map和reduce）做为参数使用。

> 做为替代，Clojure提供了definline。definline看起来功能跟defmacro一样：它的主体会返回一个代表代码已经被编译过的数据结构。不同于defmacro的是，它会创建一个可以象普通函数那样到处被使用的真正函数。下面是例子：
```

;; a normal function:
(defn square [x] (* x x))
;; an inlined function:
(definline square2 [x] (* ~x ~x)) 
```
> > 在Clojure中，definline被标记为”实验性“的。它的存在主要是为了解决函数无法接受基本类型参数或者无法返回基本类型值的问题。当函数对基本类型的支持特性被添加后，JVM会自动内嵌你的代码，而definline将变得不必要。

## 摘要 ##

---


> 就像著名的Donald Knuth说的那样，”过早进行性能优化是一切错误的根源“。关键字在于”过早“。在你做过测试之前就进行优化是毫无意义的。在你没有确认性能关键点之前进行优化比不做更糟糕。在一个实时编译、自动优化的运行时环境（比如JVM）中，情况变得更加不确定，因为监控某块代码并预测它的运行速度是很困难的。

> 最佳的途径是从两个不同的角度去确认性能：第一，高层次的性能考虑，比如避免不必要的I/O瓶颈，在应用的设计阶段就应当考虑到。而低层次的性能考虑应该延期到代码设计完成，通过性能工具确认了性能关键点后去考虑。

> Clojure提供了很多工具去优化代码性能而不用牺牲函数式风格。当这些工具都不够用时，你就需要退回到编程技术上，或者在Clojure采用数组，或者是在java中实现功能，再由Clojure来调用。