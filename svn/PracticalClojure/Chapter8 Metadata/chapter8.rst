==============
元数据(metadata)
==============


在代码中描述你的数据
=========================

程序员经常谈论metadata,即数据的数据。metadata在不同的语境中定义是不一样的。clojure提供给对象附加metadata的机制。
但是他有个特殊的定义：元数据是附加到对象的数据的映射，不影响对象的值。

两个具有相同值和不同元数据(metadata)的对象被认为是相等的（具有相同的hash值）。但是，元数据(metadata)和其他clojure 
的数据结构一样，具有不变的数据类型(immutable semantics)。当改变对象的元数据(metadata)会生成一个新的对象，新生成的
对象与原始的对象有相同的值（相同的hash值）。
 
当改变一个值时，有些操作会改变metadata,有些不会，这章会重点讨论。

读写metadata(Reading and Writing Metadata)
==================================================
 
默认环境中，REPL是不会打印了metadata的。需要将*\*print-meta\**设置成true，为后边示例:: 
	
	(set! \*print-meta\* true)

你可以用*with-meta*函数给clojure里的符号(symbol)或内建的数据结构附加元数据（metadata）,用meta函数获取元数据(metadata)。::
     
    (with-meta obj meta-map)
    (meta obj)

with-meta函数会生成新对象，对象的值是*obj*,元数据(metadata)是*meta-map*,*meta* 函数返回对象obj的元数据map(不知怎么翻译好metadata map)。
例如下面的代码 :: 
    
    user=> (with-meta [1 2] {:about "A vector"})
    #^{:about "A vector"} [1 2]

你也可以用*vary-meta*对象的元数据map(metadata map):: 
    
    (vary-meta obj function & args)

vary-meta 接收一个带有任意参数的函数并应用到对象当前的元数据map(metadata map)。它会返回一个更新了元数据（metadata）的新对象。
例如：下面的代码 ::

    user=> (def x (with-meta [3 4] {:help "Small vector"}))
    user=> x
    #^{:help "Small vector"} [3 4]
    user=> (vary-meta x assoc :help "Tiny vector")
    #^{:help "Tiny vector"} [3 4]

注意两个具有相同值不同元数据的对象的相等的。（可以用clojure里的"="函数测试，但是，在内存中不是同一对象（可以clojure里的identical?函数测试）::

    user=> (def v [1 2 3])
    user=> (= v (with-meta v {:x 1}))
    true
    user=> (identical? v (with-meta v {:x 1}))
    false

记住，你只能在clojure里的一些特殊类型添加元数据(metadata),比如列表(list),向量(vector),符号(symbol) (functions in Clojure 1.2)。
java类，像String和Number,是不支持的。  
    


metadata保存操作(Metadata-Preserving Operations)
==================================================
一些操作"改变”不变的数据结构时会存储它的元数据（metadata），有些则不会。
例如，*conj*函数操作列表时会存储它元数据（metadata），但是是*cons*函数则不会 ::

    user=> (def x (with-meta (list 1 2) {:m 1}))
    user=> x
    #^{:m 1} (1 2)
    user=> (conj x 3)
    #^{:m 1} (3 1 2)
    user=> (cons 3 x)
    (3 1 2) ;; 没有元数据

一般情况下，collection functions (conj,assoc,dissoc,and so on)支持保存元数据(metadata)，序列函数(cons,take,drop,etc.)是不支持的。
但是也有一些例外，在clojure 1.0,conj函数对向量(vector)处理时是不保存元数据(metadata)(这是个bug),但是在clojure1.1 时，又支持了。
(The moral is this) ,总之，小心处理带有元数据的数据结构。不要假想元数据会被存储。

注1
---------------------------
为什么不？试想，元数据(metadata) 可以存储在全局哈希表里，允许元数据附加到任意的java对象。然而，这种设计对于性能和内存使用有很多的
不足。所以没有被支持。

警告
-----------------------
元数据是clojure与从不同的特性。其他编程语言很少有类似的特性。当你考虑使用元数据（metadata)，要小心它代表的语义：元数据（metadata）
不是对象的值的一部分。一般来说，任何与你用户应用程序相关连的数据，不应该存储在元数据中。


(Read-Time Metadata)
==================================================
clojure的reader(第二章中有描述）允许你用*#^*reader宏给forms附加元数据(metadata)。*#^*后是要附加到后边form read的元数据map 。  
如果\*print-meta\*是trure的话，clojure会用相同的语法打印输出。例如，你可以像这样给向量（literal vector）附加元数据。:: 

    user=> #^{:m 1} [1 2]
    #^{:m 1} [1 2]

注意：*#^* 不是with-meta函数的替代者。*#^* 给literal froms附加元数据(metadata),考虑下面的代码:: 

    user=> #^{:m 1} (list 1 2)
    (1 2) ;; 没有元数据

在这个例子中，*#^* reader macro给literal form (list 1 2) 附加元数据map{:m 1},
当这个form被执行(evaluated)时 ,返回列表(1 2) 是没有元数据的。


#^ reader macro 通常用来给符号(symbols)附加元数据(metadata),而不是数据结构。
Special forms such as def can make use of this read-time metadata.


注意 clojure 1.0提供^作为meta的简写。这种写法不是很有用。在clojure1.1和1.2时被弃用。


变量里的metadata
==================================================
在clojre里最常见的使用元数据的地方是给变量添加描述信息。函数def,defn和defmacro 会给每个变量添加默认的元数据。
例如，下面的代码::
    
    user=> (meta (var or))
    {:ns #<Namespace clojure.core>
    :name or
    :file "clojure/core.clj"
    :line 504
    :doc "Evaluates exprs one at a time..."
     ([] [x] [x & next])
    :macro true}

另外，*def* 以及同类的函数会


    user=> (def #^{:doc "My cool thing"} \*thing\*)
    #'user/\*thing\*
    user=> (:doc (meta (var \*thing\*)))
    "My cool thing"



    => (doc \*thing\*)
    -------------------------
    user/\*thing\*
    nil
        My cool thing


    (defn name doc-string meta-map [params] ...)
    (defmacro name doc-string meta-map [params] ...)


    (def #^{:private true} \*my-private-var\*) ;; for Vars
    (defmacro my-private-macro {:private true} [args] ...)
    ;; for macros



引用类型里的metadata
==================================================

    (alter-meta! iref f & args)


    (alter-meta! (var for) assoc :note "Not a loop!")
    {:note "Not a loop!" :macro true, ...
    user=> (:note (meta (var for)))
    "Not a loop!"

    user=> (def r (ref nil :meta {:about "This is my ref"}))
    user=> (meta r)
    {:about "This is my ref"}



总结
===================================================

元数据(metadata)是种与种不同的特性。
