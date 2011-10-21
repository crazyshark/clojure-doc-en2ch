==============
元数据(metadata)
==============


在代码中描述你的数据
=========================

程序员经常谈论metadata,即数据的数据。metadata在不同的上下文环境中定义是不一样的。clojure提供机制metadata
    
两个具有相同值和不同元数据(metadata)的对象被认为是相等的（具有相同的hash值）。但是，元数据(metadata)和其他clojure 
的数据结构一样，具有不变的数据类型(immutable semantics)。当改变对象的元数据(metadata)会生成一个新的对象，新生成的
对象与原始的对象有相同的值（相同的hash值）。
 
当改变一个值时，有些操作会改变metadata,有些不会。这章会重点讨论。

读写metadata(Reading and Writing Metadata)
==================================================
 
默认环境中，REPL是不会打印了metadata的。需要将*\*print-meta\**设置成true，为后边示例:: 
	
	(set! \*print-meta\* true)

你可以用*with-meta*函数给clojure里的符号(symbol)或内建的数据结构附加元数据（metadata）,用meta函数获取元数据(metadata)。::
     
    (with-meta obj meta-map)
    (meta obj)

with-meta函数会生成新对象，对象的值是*obj*,元数据(metadata)是*meta-map*,*meta* 函数返回对象obj的元数据地图(不知怎么翻译好metadata map)。
例如下面的代码 :: 
    
    user=> (with-meta [1 2] {:about "A vector"})
    #^{:about "A vector"} [1 2]

你也可以用*vary-meta*对象的元数据地图(metadata map):: 
    
    (vary-meta obj function & args)

vary-meta 接收一个带有任意参数的函数并应用到对象当前的元数据地图(metadata map)。它会返回一个更新元数据（metadata）的新对象。
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
一些"改变”不变的数据结构时会存储它的元数据（metadata），有些则不会。
例如，*conj*函数操作列表时会存储它元数据（metadata），但是是*cons*函数则不会 ::

    user=> (def x (with-meta (list 1 2) {:m 1}))
    user=> x
    #^{:m 1} (1 2)
    user=> (conj x 3)
    #^{:m 1} (3 1 2)
    user=> (cons 3 x)
    (3 1 2) ;; no metadata!




(Read-Time Metadata)
==================================================
    
    user=> #^{:m 1} [1 2]
    #^{:m 1} [1 2]



    user=> #^{:m 1} (list 1 2)
    (1 2) ;; no metadata!


变量里的metadata
==================================================

    
    user=> (meta (var or))
    {:ns #<Namespace clojure.core>
    :name or
    :file "clojure/core.clj"
    :line 504
    :doc "Evaluates exprs one at a time..."
     ([] [x] [x & next])
    :macro true}


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
