# Chapter 11 并行编程 #

## Clojure 并行编程 ##
在第6章，我们花了很多时间来讨论Clojure如何在并发环境中安全地管理状态。状态管理的确是并发编程中最棘手的部分，Clojure在这方面也付出了很多努力。

然而，这些讨论并没有明确告诉我们如何着手把我们的程序变为并行计算，以及什么把程序划分成多线程执行的最佳策略。尽管这不是一个困扰我们的问题，但我们仍应该有相应的了解。知道多线程中分布运算何时以何种方式执行，你可以最大程度地把程序并行化，使其更快运行，并保证在多核电脑上的扩展性。

Clojure提供了对应不同抽象层次多种并发技术， 如对JVM原语进行多层代理 ，以及通过与Java互操作来完成。一些技术更适合于以数据为中心的并发，其他的更便于手动介入线程运行。

本章将阐述可以在一个Clojure程序中引入并发的各种方法和他们的优缺点。

## 代理 （Agents） ##
代理在第6章曾讨论过，他的主要用途是管理状态时进行身份区分。代理很有趣，他作为一座桥，连通了状态管理和执行管理，这两个过程都包括代理。回想在第6章，我们曾经详细的讨论过如何创建一个action并把它发送给代理。本节主要介绍他们的并发性和产生的影响。

### 代理线程池 （Agent Thread Pools ） ###
从执行层面看，代理运行在一个有Clojure运行时管理的线程池中。被发送到代理的action，按队列先后的顺序在两个线程池之一运行，线程池的选择取决于这个action是被send还是send-off函数分派来的。

被send函数使用的线程池，并行数量被调整到符合JVM支持的对应处理器个数。这样可以对运算密集型action进行优化：并行执行的action数，基本与cpu个数一致。当一个线程池处于饱和状态时，新来的action 将排在处理队列末尾，依次等候处理。

被send-off函数使用的线程池，不限制并行数量。因为在这里执行的是高延迟度的action，例如：访问远程资源时，大部分时间都用于等待。此时，让多个进程共享一个处理器的将更加高效。

如果你不按任务类型来使用send和send-off，也不会发生什么灾难。你的程序仍会正确运行，只是效率不是太高。如果一个高延迟度的action使用send来分派，它将会一直占用一个send-thread，直到执行完毕，而不能把线程让给其他action。相反，如果一个运算密集型的action使用send-off分派，虽然执行结果没有影响，但它的执行会经常被操作系统的线程调度打断。

### 代理的例子 ###
这个例子演示运算密集型代理。假设你有一个代理用来计算平均数，被计算的是一个链表（list），它包含着很多数字。代理的值由两个关键字决定：装满数字的链表，以及当前的平均值。

```

user=> (def my-average (agent {:nums [] :avg 0}))
#’user/my-average
```

现在，让我们定义一个函数作为这个代理的一个action。定义两个变量：代理的当前值，已经要被加上的数字。返回值是代理的新值。

```

(defn update-average [current n]
(let [new-nums (conj (:nums current) n)]
{:nums new-nums
:avg (/ (reduce + new-nums) (count new-nums))}))
```

这种情况下，action是直接执行的，没有IO操作，因此我们选择send而不是send-off。

```

user=> (send my-average update-average 10)
#<Agent @4cdac8 {:nums [], :avg 0}>

user=> (send my-average update-average 20)
#<Agent @4cdac8 {:nums [10], :avg 10}>

user=> (send my-average update-average 10)
#<Agent @4cdac8 {:nums [10 20], :avg 15}>

user=> (send my-average update-average 20)
#<Agent @4cdac8 {:nums [10 20 10], :avg 40/3}>
```

最后，查看结果：
```

user=> @my-average
{:nums [10 20 10 20], :avg 15}
```

看起来跟一般情况没有区别。但是，因为你使用了send，并且update-average只包括处理和IO等待，你可以认为代理全速处理了send发送的action。

### 并发代理性能 ###
代理的性能跟机器的cpu数成正比。如果一个算法或处理过程由抽象的“任务”组成（或这可以被分解成这种方式），使用代理是一个极佳的选择。理论上讲，基于代理的程序在不修改任何代码的情况下，其性能可以接近运行在数百个CPU上的普通程序。当然，这取决于代理在做哪种类型的工作。

## 并行函数 ##
Clojure标准库中包含对应的函数和宏来初始化一个并行处理过程。它们可以很方便的使用，不需要设置，并且可以快速插入到你的代码中。

包括三个内建函数pmap，pvalues和pcalls，它们的功能类似：实际上其它两个函数都是在pmap基础上构建的。以此为基础，我们可以自己构建丰富的并行工具箱。

为了演示并行的例子，我们要让我们的程序多跑一段时间，只有一个指令的执行过程是不需要用并行计算的。为了做到这一点，我们创建一个使用函数作为参数的函数，并且返回参数的“重量级”版本--要执行一秒钟才返回。转换函数定义如下：

```

(defn make-heavy [f]
(fn [& args]
(Thread/sleep 1000)
(apply f args)))
```

> 你可以看到，这个返回函数取代参数函数执行，并且使用内建的time宏来延迟1秒钟。例如：一个普通的 + 函数调用几乎不花费多少时间：

```

user=> (time (+ 5 5))
"Elapsed time: 2.0E-6 msecs"
10
```

而如我们所料，包装过的 + 函数 话费了1秒钟左右。

```

user=> (time ((make-heavy +) 5 5))
"Elapsed time: 1001.128155 msecs"
10
```

通过使用这个办法，你将看到使用并行函数带来的好处。

**pmap**<br />
pmap的函数原型和功能与普通的map函数别无二致。唯一的区别在于提供的函数会在一个并行队列中调用，在这里可以利用跟cpu数目相对应的系统线程并行执行。pmap是部分延迟解释的，整个结果只有在被请求时才生成，但是并行计算部分要比请求提前执行。
下面的例子，展示了pmap和map的相同功能：

```

user=> (pmap inc [1 2 3 4])
(2 3 4 5)
```

为了看出并行计算发挥的作用，我们使用“重量级”版本。首先看普通版本的执行时间，这里我们同样用doall函数来强制对结果求值：

```

user=> (time (doall (map (make-heavy inc) [1 2 3 4 5])))
"Elapsed time: 5002.96291 msecs"
(2 3 4 5 6)
```

看到，普通map使用“重量级”inc函数5次。因为在同一个线程中执行，每次调用消耗了1秒，合计5秒。

现在用pmap替代map：

```

user=> (time (doall (pmap (make-heavy inc) [1 2 3 4 5])))
"Elapsed time: 1031.941815 msecs"
(2 3 4 5 6)
```

结果只用了1秒多。因为虽然同样调用相同的函数5次，但是它们是并行执行的。多使用的30毫秒，是用于设置多余线程使用的时间。

**pvalues**<br />
pvalues接受任意多个表达式，并行执行，并为每个表达式返回一个延迟解释的序列。
```

user=> (pvalues (+ 5 5) (- 5 3) (* 2 4))
(10 2 8)
```

**pcalls**<br />
pcalls接受任意多个无参的函数，并行执行，并为每个函数返回一个延迟解释的序列。
```

user=> (pcalls #(+ 5 2) #(* 2 5))
(7 10)
```

### 开销和性能 ###
对于偏重于计算的操作，这些并发函数可以轻松的大幅提高执行速度。但是对于非计算为主的操作，他们也许并不合适。

当并行函数执行时，他们把参数分解成一个个工作单元并分派执行。这个过程需要对工作单元进行加载，如果计算过程比配置执行过程还要快，则并行反而会比非并行时运行更慢。

这意味着采用并行方式能否取得好的效果，取决于要执行什么样的操作。如果是轻量级计算的操作，比如基础数学运算（就像前面的例子），则避免并行方式，因为配置并行执行的开销要超过计算的开销。如果是偏重于计算的操作，它们的每个计算都要花费大量的时间，此时适合使用并行，并且会取得不错效果。而对于位于这两个极端中间类型的操作，需要通过实际验证来决定是否采用并行方式。你可以试着扩大每步并行运算的规模，例如：将所有的处理过程组织成一个过程，再拆分进行并行处理，而不是依次进行处理。

正如所见，比较对于轻量级操作使用pmap和map时的时间消耗：例如，轻量级的inc操作。

```

user=> (time (dorun (map inc (range 1 1000))))
"Elapsed time: 9.150946 msecs"

user=> (time (dorun (pmap inc (range 1 1000))))
"Elapsed time: 182.349073 msecs"
```

这清楚的说明，创建线程和分工协作所花费的成本，大大超过使用并行计算带来的好处。

## Futures 和 Promises ##

futures 和 promises是两个稍微更低级一点的线程构造，他们的 功能类似于Java 6 的并行api所提供的功能。它们易于理解和使用，并且可以使用Clojure语法非常直接的产生线程。

### Futures ###
Clojure的 future 表示一个在单独线程中执行的运算。当future创建后，一个新的线程被创建，并且开始执行运算。当运算结束时，线程被回收，运算结果的值在future解引用（dereferencing）时被提取。相反，如果future 解引用时运算还没有结束，则解引用的线程会阻塞，直到运算执行完成。

使用future宏可以创建一个future，通过传入任意个表达式，产生一个future。这个futrue执行所有的表达式并返回最后一个表达式的值。比如：

```

user=> (def my-future (future (* 100 100)))
#'user/my-future

user=> @my-future
10000
```

在这个例子中，(`*` 100 100)在一个独立的线程中执行。在一个实际项目中，这类简单表达式不值得使用future来执行。让我们使用Java的函数Thread.sleep()来模拟一个执行时间较长的运算。在Clojure中，我们通过Thread/sleep来调用这个函数。它可以使当前线程暂停一定的毫秒数。

```

user=> (def my-future (future (Thread/sleep 10000)))
'#user/my-future
```

这个future需要运行10秒。如果你在接下来的10秒内在REPL中输入下面的代码，你会看到解引用future的过程被阻塞了，因为future没有执行完毕。

```

user=> @my-future
nil
```

系统将会暂停，直到future执行完才返回值nil。

你还可以使用future-call函数来创建future。这跟使用future宏相似，只是它不把表达式作为参数处理，它接受一个无参的函数作为参数，并在一个独立的线程中执行这个函数。你可以使用同样的办法来解引用并检查future。

### 控制 Futures ###

Clojure 包含几个控制和检查future的函数。

**future-cancel**<br />
这个函数试图在future执行完毕之前取消future。只能在特定情况下使用，因为取消的过程使用的是Java的线程中断机制。要取消一个运算，需要不时地从内部检测线程中断状态，或者调用一个函数来完成（例如，Thread/sleep）。详细内容，参考Java线程相关文档。
> future-cancel 接受一个参数，就是要取消的future。如果future已经执行完毕，将不会被取消。如果一个future在执行完毕前被取消，在尝试解引用future时会引发CancellationException异常。

**future-cancelled?**<br />
future-cancelled? 接受一个future参数，并返回future是否被取消的检测。可以用来检测future-cancel是否执行成功，以便可以安全的解引用这个future。

**future-done?**<br />
future-done? 接受一个future参数，并返回future是否已完成检测。如果完成，返回true，否则返回false。这个函数用来检测future执行状态，并了解在解引用future时是否会引起阻塞

**future?**<br />
future? 接受一个参数，并返回这个参数是否为future的检测。

### Promises ###
promise可能未被赋值。当一个promise在没有被赋值时被解引用，解引用线程将会阻塞，直到promise被复制。与本章中描述的其他特点不同，promise实际上并不是并行执行，但它经常被用来进行执行流程的控制（特别是future的协调处理）。当一个promise被赋值后，所以等待这个promise的线程获得这个值，并释放。promise的值传递后会被解引用。
调用 promise函数来创建一个promise.

```

user=> (def my-promise (promise))
'#user/my-promise
```

使用deliver函数来传递promise的值。这个函数有2个参数，一个是promise，一个是值，返回这个promise。deliver对于每个promise只能执行一次，如果多次对一个promise调用，将会引发异常。

```

user=> (deliver my-promise 5)
#<AFn$IDeref&db53459f@1465272: 5>
```

promise的解引用方法：
```

user=> @my-promise
5
```


**注意事项：**<br />
要当心！使用promise可能会让你得程序死锁。保证promise会得到一个传入值，否则它们会永远阻塞。在前面的例子里，如果在deliver调用之前在REPL中解引用my-promise，REPL会被阻塞，而这时你也无法把值传递给promise。你不得不重启REPL。
在Clojure程序中，promise的作用被限制了：通常，更好的选择是使用更高级别的并行结构。但是对于需要手动控制线程等待或线程间切换的应用场景，promise可以提供一个简单的机制。

## 基于Java的线程 ##
如果Clojure提供的并行工具都无法满足你得需要，你还可以使用Java自身的线程能力。通过使用Clojure的Java互操作，这些工作可以用Java中同样的方法来完成。有时，调用实现java.lang.Runnable借口的 Clojure函数 甚至更容易一些，这样他们就可以直接被传递给线程。同时，Clojure的宏可以用于精简Java的模式化代码。
对Java并发的完全讨论超出了本章（本书）的讨论范围。尽管如此，本节将展示一个普遍性的任务：创建单线程。同样的办法可以应用到其他Java并行API上。Java并发API的信息和入门，请看http://java.sun.com/docs/books/tutorial/essential/concurrency/.

### 创建一个线程 ###
在Java中创建一个线程的基本方法是实例化一个java.lang.Thread对象，在结构中传入一个可执行的过程，并调用它的start()方法。同样的操作可以在Clojure中实现，看下面的代码：
```

user=> (def value (atom 0))
#’user/value
```

首先，创建一个atom来存储一个值。这个操作不是线程代码的一部分，但你需要一个办法来检测线程是否在运行，你可以通过更新这个atom的值来实现。
```

user=> (def my-thread (Thread. #(swap! value inc)))
#'user/my-thread
```

通过调用java.lang.Thread的构造函数创建一个线程，这个构造函数接受一个可执行过程(runnable)作为参数。这这里，你可以提供一个简单的内联函数（在Clojure里，所有函数都是可执行过程(runnable)）。然后调用start()方法，启动这个线程：
```

user=> (.start my-thread)
nil
```

并且检测线程是否在运行:
```

user=> @value
1
```

注意，在Java中这并不是一个最佳的创建线程的方式。通常，我们会使用executor框架（可以看前面给出的URL的介绍）。尽管如此，这样的方法还是可以创建线程，并把Clojure函数作为可执行过程(runnable)传递进去。

## 总结 ##
Clojure引入的并行机制很多，他们基本遵循下面的简述：
  * 在Clojure中，最低级的并行功能是调用Java内建的并行库。Java做的事它也能做，但是缺乏Clojure高级特性带来的易用性。
  * 使用Clojure的特性，可以轻松的控制线程创建。如果你想让线程强制等待，可以使用promise。
  * 对数据的多个部分并行执行同样的操作，很难充分利用Clojure的并行函数。如果一个算法使用map，那么他可以被轻松的用pmap改造成并行执行。
  * Clojure的代理技术，可以提供一个高级的，基于线程池的线程管理系统，管理线程的状态和执行。

要成功编写一个高度并发的Clojure程序，需要选择正确的线程模型，使用前面我们介绍的方法，并且安全的进行状态管理（在第10章讨论）。Clojure提供了简单易用的工具，可以让我们轻松完成这一系列任务。