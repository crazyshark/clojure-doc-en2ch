组合类型888

Clojure的组合数据类型是用来高效地满足操纵各种聚合数据结构的需要。这些数据类型经过优化之后效率更高，并且与Clojure的其它部分以及Java更加兼容，并且坚持了Clojure的原则：不变性。如果这些数据类型中的任何一种都不足以表示某种数据结构，那么我们可以通过任何方式来组合它们。

这些数据类型都具有如下性质：

都不可变。一旦被创建，它们就不可改变，因此对于任何时间的任何线程来讲，访问它们都是安全的。那些被认为是“改变了“它们的操作实际上是返回了一个全新的依旧不可变的对象。

都是持久的。这些数据类型会快速地与它们的之前版本共享数据结构来持久化内存和运行时间。因此，它们都十分的快速和高效，在某种程度上来讲，跟那些在其它语言中的可变的类似概念相比，更加高效。

适当地支持判断是否相等的语义。这意味着若两个对象的数据类型相同且包含相同引用，它们总是被认为是相同的，而不管其实例化和实现的细节。因此，两个组合类型的数据，即使创建于不同的时间或不同的地点，也依然可以用来比较。

在Clojure中使用起来十分简单。每种组合数据类型都有一个方便的字面表示和许多相关函数，确保使用这些数据类型顺利无碍。

支持与Java的互操作。这些数据类型都很好地支持了标准java.util.Collection框架的只读部分。在很多情况下，这表示它们可以不用更改地传递给那些需要组合数据类型的Java对象和方法。Lists实现了java.util.List，Maps实现了java.util.Map，Sets实现了java.util.Set。注意，因为它们都是不可变的，所以如果你使用了可能改变它们的方法，就会抛出一个UnsupportedOperationException异常。这与文档中规定的java.util.Connections接口标准一致，因为组合数据类型不支持“破坏性的“改变。

基于函数编程的范式，这些数据类型都支持通过简单而强大的操作来操作序列。这些功能在第五章有详细讨论。

列表

对Clojure来说列表十分重要，因为实际上Clojure程序本身就是由很多嵌套着的组成的。在最基本的层面上来讲，一个列表就是一些元素的有序集合。

列表可以通过使用括号来直接输入，这也是为什么Clojure代码本身就使用了如此多的列表。例如，正常地调用一个函数：
(println "Hello World!")
它是一串可执行的代码，同时也是一个列表。首先，Clojure读取程序将它作为一个列表来解析，然后将其第一个元素（在这里是println）作为函数来对它求值，然后将剩余的部分（"Hello World!"）作为参数传递给它。

如果只是作为数据结构而不是可执行代码来使用列表，只需要给列表加一个单引号作为前缀即可。这告诉Clojure将其作为数据结构来对待，而不是将其当作Clojure形式对其求值。例如，定义一个由1到5组成的列表，并将其绑定到一个符号，你可以这样做：
(def nums '(1 2 3 4 5))

-----
注意：这里的单引号实际上是另一种形式，叫做quote。'(1 2 3)和(quoto (1 2 3))只是表示相同事物的不同方法而已。quote（或者单引号）可以在任何地方使用，来阻止Clojure立即对一个表达式求值。实际上，它的作用远不止于声明一个列表，当涉及到元编程的时候，单引号十分必须。请阅读12章里在宏里使用quote来实现复杂的元编程的详细讨论。

列表是以单向链接列表的形式来实现的，在这一点上有利有弊。读取列表的第一个元素或者在列表头添加一个元素的操作都可以在常量时间内完成，然而访问列表的第N个元素却需要N次操作。因为这个原因，在很多情况下，向量是个更好地选择。不过列表在很多情况下依然十分有用，特别是在即使构建Clojure代码的时候。

list
list函数接收任意数量的参数并将它们的值组成列表。
(list 1 2 3)
--> (1 2 3)

peek
peek函数操纵一个单一的列表作为参数并返回列表中的第一个值。
(peek '(1 2 3))
--> 1

pop
pop函数操纵一个单一的列表作为参数并且返回一个去掉了首个元素的新列表。
(pop '(1 2 3))
--> (2 3)

list?
如果其参数是一个列表，那么列表测试函数list？返回true，否则返回false。
(list? '(1 2 3))
--> true

向量

向量跟列表很相似，它们都存储一串有序的元素。但是，它们有一个很重要的地方有所不同：向量支持高效地、近乎常量时间地根据元素的索引来访问。从这一点来看，相比于列表，向量更像是数组。总的来说，对于很多应用来讲向量更好，因为跟列表相比向量毫无劣势而且更快。

向量在Clojure程序中的字面表示是使用方括号。例如，一个由1到5组成的向量可以通过如下代码定义并绑定到一个符号上：
(def nums [1 2 3 4 5])

向量的它们的索引的函数。这不仅仅是一个数学上的描述——它们都是实现了的函数，并且可以通过函数调用来取得元素的值。通过索引来取得值的最简单的方法是：像函数一样调用这个向量，然后将你想要的索引传递给它。索引从0开始，所以，为了取得之前定义好的一个向量的第一个元素，你可以这样做：
user=> (nums 0)
1

尝试访问超出向量长度的索引会引发一个错误，具体来说是java.lang.IndexOutOfBounds异常。

vector
构建向量函数vector接收任意数量的参数并将它们的值组成一个向量。
(vector 1 2 3)
--> [1 2 3]

vec
向量转换函数vec接收一个单独的参数，可能是任何Clojure或Java的组合数据类型，然后将其元素的值作为参数组成一个新的向量。
(vec '(1 2 3))
--> [1 2 3]

get
get函数接收两个参数来操作向量。第一个参数是一个向量，第二个参数是一个整数索引。它返回给定索引处的值，若在索引处没有值，则返回nil。
(get ["first" "second" "third"] 1)
--> "second"

peek函数接收一个单独的向量作为参数，并且返回向量的最后一个值。考虑到列表和向量的不同实现方式，这跟列表的peek函数有所不同：向量总是访问最方便的那个元素。
(peek [1 2 3])
--> 3

vector?
向量测试函数vector接收一个单独的参数，若参数是一个向量则返回true，否则返回nil。
(vector? [1 2 3])
--> true

conj
连接函数conj接收一个组合数据类型（例如向量）作为其第一个参数和任意数量的其它参数。它返回一个新的向量，这个向量由将所有的其它参数连接到原来那个向量尾部组成。conj函数也对映射和集合适用。
(conj [1 2 3] 4 5)
--> [1 2 3 4 5]

assoc
向量组合函数assoc接收三个参数：第一个是向量，第二个是整数索引，第三个是一个值。它返回一个新的向量，这个向量是原来那个向量在给定的索引处插入那个值的结果。如果索引超过了向量的长度，那么会引发一个错误。
(assoc [1 2 3] 1 "new value")
--> [1 "new value" 2 3]

pop
pop函数接收一个单独的向量作为参数，并且返回一个去掉尾部元素的新向量。考虑到列表和向量的不同实现方式，这跟列表的peek函数有所不同：向量总是访问最方便的那个元素。
(pop [1 2 3])
--> [2 3]

subvec
子向量函数subvec接收两个或三个参数。第一个是一个向量，第二个和第三个（如果有的话）是索引。它返回一个新向量，这个向量由原来那个向量的介于两个索引之间或者第一个索引到向量末尾（如果没有第二个索引）的部分组成。
(subvec [1 2 3 4 5] 2)
--> [3 4 5]
(subvec [1 2 3 4 5] 2 4)
--> [3 4]

Maps

映射可能是Clojure里最有用最强大的内建组合数据类型了。实际上，映射十分简单。它存储一个键-值对的集合。键和值都可以是任何数据类型的对象，无论是基本数据类型还是其它映射。然而，使用关键字来作为映射的键非常合适，因此它们经常在应用映射的场合被使用。

映射的字面表示是使用花括号包围着的偶数个元素。这些元素都被看作键/值对。例如这个映射：
(def my-map {:a 1 :b 2 :c 3})

这个映射定义定义了一个有三个键的映射，关键字:a，:b和:c。键:a绑定到1，:b绑定到2，:c绑定到3。由于逗号在Clojure中和空格的作用是一样的，它经常被用来清晰地表示键/值对，而丝毫不改变映射定义的实际意义。下面这行代码跟之前的那行完全相同：
(def my-map {:a 1, :b 2, :c 3})

虽然关键字作为映射的键十分合适，但是并没有规则说你必须要使用它们：任何值，甚至是另一个组合数据类型，都可以作为键。关键字、字符串和数字都经常被用作映射的键。

与向量类似，映射是它们的键的函数（不过如果给定的键不存在，它们不会抛出异常）。要得到一个特定键对应的值，只要使用该映射最为函数，并将键作为参数传递给它。例如，为了得到上面的例子里:b对应的值，只需要这样做：
user=> (my-map :b)
2

普通的映射可能有三种不同的实现方式：数组映射、哈希映射和有序映射。它们分别使用数组、哈希表和二叉树来作为底层实现。数组映射最适用于较小的映射，而对哈希映射和有序映射的比较则要基于特定应用场合的情况。

默认地，根据字面定义的映射如果很小则被实例化为数组映射，若很大则为哈希映射。若要显式地创建特定类型的映射，可以使用hash-map或者sorted-map函数：
user=> (hash-map :a 1, :b 2, :c 3)
{:a 1, :b 2, :c 3}

user=> (sorted-map :a 1, :b 2, :c 3)
{:a 1, :b 2, :c 3}

注意，哈希映射并不储存键的顺序，而有序映射则会根据键来对值进行排序然后存储。默认地，sorted-map非常自然地对键进行比较：根据数字或者字母表里可用的那一种。

Struct Maps

使用映射时，很多时候有这种情况：我们需要产生一组有相同键组合的映射。因为一个普通的映射对它键和值都会分配内存，所以在产生大量的类似映射的时候这会导致内存的浪费。

不过，创建大量映射很多时候十分有用，所以Clojure提供了结构映射。结构映射允许你首先定一个键组成的结构，然后用它来实例化多个映射，并通过共享键和查找的信息来节省内存。它们在语义上跟普通映射相同：唯一不同的是实现方式。

要定义一个结构，使用defstruct：它接收一个名字和一些键作为参数。例如如下代码：
(defstruct person :first-name :last-name)

这定义一个名为person的结构，键为:first-name和:last-name。使用struct-map函数来创建person的实例：
(def person1 (struct-map person :first-name "Luke" :last-name "VanderHart"))
(def person2 (struct-map person :first-name "John" :last-name "Smith"))

现在，person1和person2是两个不同的映射并且十分节省地共享键的信息。但是他们依然是映射，因此从各方面来说，你都可以使用相同的方法来取得一个值甚至是添加新的键。当然，新添加的键不会像在结构里定义的键一样有节省内存的优势。跟普通映射相比，结构映射的唯一限制是，你不能删除一个结构映射里的某个在结构定义里定义了的键。这样错会引发一个错误。

结构映射同时允许你创建十分高效的函数来访问键的值。普通映射的查找速度绝不慢，但使用结构访问函数，你将可以大大缩短普通键查找过程所花的时间，以适用于那些极端性能敏感场合的应用。

要创建一个结构映射的高性能访问函数，使用accessor函数。它接收一个结构定义和一个键作为参数，并返回一个一等888函数作为返回值。这个函数接收一个结构映射作为参数，并返回一个值。
(def get-first-name (accessor person :first-name))

你可以使用新定义的get-first-name函数十分快速地取得一个函数映射里的:first-name键对应的值。下面的两个表达式是等价的，但是使用访问函数的版本更快。
(get-first-name person1)
(person1 :first-name)

总的来讲，除非是考虑到性能的因素，你无须担心要去使用结构映射。对很多应用来讲，普通映射就足够快了，而且结构映射只是稍微增加了复杂度，而除了些许的性能提升外并没有特别的好处。你应该知道它们，因为它们使得一些程序的性能更好，但实际上最好是你现实用普通映射，之后优化的时候再使用结构映射来重构你的程序。

Maps as Objects

很明显，在很多场景下映射都十分有用。编程时，连接键和值是一个很常见的操作。然而，映射的可用性远远不止于我们所认为它只是一个数据结构的那样。

一个很重要的例子是，结构可以做到面向对象编程中的对象90%能做的事。那么对象中命名的属性和映射里的键/值对到底有什么不同之处呢？像Javascript这种语言（对象是用映射实现的）表示，没有什么不同。

好的Clojure程序大量使用这种映射即是对象的观点。虽然Clojure在总体上不接受面向对象的理念，对面向对象设计的数十年的研究确实发现了一些关于数据包装和组织的好的规则。这样使用Clojure的映射的话，那么从面向对象的数据组织里获得某些技巧和教训并且规避它的缺点就变得可能了。在一个Clojure程序的上下文里，使用映射十分不错，因为可以通过普通的方式来操作它们，而不必为不同的类的对象创建操作的方法。

assoc
映射结合函数assoc接收一个映射和一些键/值对作为参数。它返回一个新的映射，此映射含有参数里提供的键，或者替换原映射里任何已有的键。
(assoc {:a 1 :b 2} :c 3)
-> {:c 3, :a 1, :b 2}

(assoc {:a 1 :b 2} :c 3 :d 4)
-> {:d 4, :c 3, :a 1, :b 2}

dissoc
映射分离函数dissoc接收一个映射和一些键作为参数。它返回一个新的映射，此映射去掉了参数了提供的这些键。
(dissoc {:a 1 :b 2 :c 3} :c)
-> {:a 1, :b 2}

(dissoc {:a 1 :b 2 :c 3 :d 4} :a :c)
-> {:b 2, :d 4}

conj
连接函数conj对映射的作用跟对向量的作用一样，不过连接的不是一个单独的元素，而是一个键/值对。
(conj {:a 1 :b 2 :c 3} {:d 4})
-> {:a 1, :b 2, :c 3, :d 4}

连接一个键/值对组成的向量也可以，例如如下代码所示：
(conj {:a 1 :b 2 :c 3} [:d 4])
-> {:d 4, :a 1, :b 2, :c 3}

merge
映射归并函数merge接收任意数量的参数，其中每个参数都是一个映射。它返回一个新的映射，该映射有参数里所有映射的键/值对组成。若某一个键出现在了多个映射里，最终其值会是最后包含此键的映射里对应的值。
(merge {:a 1 :b 2} {:c 3 :d 4})
-> {:d 4, :c 3, :a 1, :b 2}

merge-with
merge-with函数接收一个一等函数作为其第一个参数，然后是任意数量的映射作为其它参数。它返回一个新的映射，该映射由参数里的所有映射的键和值所组成。若一个键在多个映射里出现，那么最后的值是参数里给定的函数作用于所有这些冲突键的值的返回值。
(merge-with + {:a 1 :b 2} {:b 2 :c 4})
-> {:c 4, :a 1, :b 4}

get
get函数接收一个映射作为其第一个参数，一个键作为其第二个参数。第三个参数是可选的，是一个值，若没有找到参数里指定的键，则返回该值。它返回映射里指定键对应的值，若未找到并且第三个参数没有被指定，则返回nil。
(get {:a 1 :b 2 :c 3} :a)
-> 1

(get {:a 1 :b 2 :c 3} :d 0)
-> 0

contains?
contains?函数接收一个映射和一个键作为参数。若映射里存在该键，则返回true，否则返回false。除了映射，它也适用于向量和集合。
(contains? {:a 1 :b 2 :c 3} :a)
-> true

map?
映射测试函数map?接收一个单独的参数，若它是映射则返回true，否则返回false。
(map? {:a 1 :b 2 :c 3})
-> true

keys
keys函数接收一个单独的参数，一个向量。它返回一个由该向量里所有键组成的列表。
(keys {:a 1 :b 2 :c 3})
-> (:a :b :c)

vals
vals函数接收一个单独的参数，一个向量。它返回一个由该向量里所有值组成的列表。
(vals {:a 1 :b 2 :c 3})
-> (1 2 3)

Sets
Clojure里的集合的概念跟数学紧密相关：它们是不同的数据的集合，而且支持验证是否是集合的成员及其一般的集合运算，例如并、交和差。

集合的字面语法是一个井号后面跟着包围在花括号里的集合成员。例如如下的代码：
(def languages #{:java :lisp :c++})

跟映射一样，它们支持任何类型的对象作为其成员。例如一个使用字符串的类似集合：
(def languages-names #{"Java" "Lisp" "C++"})

集合的实现方式跟映射十分类似。它们都可以使用哈希表或二叉树来实现，使用hash-set或者sorted-set函数：
(def set1 (hash-set :a :b :c))
(def set2 (sorted-set :a :b :c))

跟映射相似，集合是它们的成员的函数。将一个集合调用为函数，并将一个值传递给它，若该值是集合的成员则会返回这个值，否则返回nil。
(set1 :a) ;return :a
(set1 :z) ;return nil

一般集合函数
注意，集合的关系函数并不在默认的clojure.core命名空间里，而是位于clojure.set命名空间。你要么显示地引用，要么使用ns形式的:use子句将其包含到你的命名空间里。请查阅第二章。

clojure.set/union
集合的并函数union接收任意数量的参数，每个参数都是一个集合。它返回一个新的集合，该集合由参数给定的集合的成员的并集组成。
(clojure.set/union #{:a :b} #{:c :d})
-> #{:a, :b, :c, :d}

clojure.set/intersection
集合的交函数intersection接收任意数量的参数，每个参数都是一个集合。它返回一个新的集合，该集合由参数给定的集合的成员的交集组成。
(clojure.set/intersection #{:a :b :c :d} #{:c :d :f :g})
-> #{:c, :d}

clojure.set/difference
集合的交函数intersection接收任意数量的参数，每个参数都是一个集合。它返回一个新的集合，该集合由参数中第一个集合里的元素里不在之后的其它集合里的元素组成。
(clojure.set/difference #{:a :b :c :d} #{:c :d})
-> #{:a, :b}

小结
Clojure提供了一组完整的强大的数据类型，使用它们可以满足任何程序的需求。它的基本类型提供了构建程序的基本组成部分，包括丰富简便的数字和字符串支持。
然而，Clojure的类型系统的真正威力在于它的集合数据类型库。组合数据类型不仅使用方面，更加补充了Clojure对于数据和不可变性的哲学。它严格遵守的原则有不可变性，意味着数据不可改变，持久性，意味着它们最大限度地高效共享其结构。依靠Clojure的内建数据结构并且熟悉可以操作它们的方法会十分有助于你构建高效、清晰和符合惯例的程序。
