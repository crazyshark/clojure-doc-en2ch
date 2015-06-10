# CHAPTER 13 数据类型和协议 #


> Clojure是基于如下抽象类型：序列，引用，宏以及其他。然而，这些抽象类型大多来源于java 中的类或者接口，想要绕过java 中的这些类型往语言中添加新的抽象类型（比如说数据队列结构）是非常困难的。

> Clojure1.2中提供了很多新特性用于较简单地直接实现新抽象类型，同时保障了这些类型能充分利用java 平台的性能优化。

> 数据类型和协议大致等同于java 中的类和接口，不过更加灵活。

<pre>
注意：当这本书编写时，Clojure1.2还没发布。尽管概念上会保持一致，但实际发布的版本在名称和语法上面可能会同本书有细微的不同。<br>
</pre>

## 协议 ##

---

> 一个协议是数个方法的集合。协议拥有一个名称和一段可选的介绍性文字。其中的每个方法拥有一个名称，一个或者多个参数，和一段可选的介绍性文字。这就是协议！它没有具体的实现，没有实际的代码。

> 采用defprotocol来创建协议：
```
(defprotocol MyProtocol 
  "This is my new protocol" 
  (method-one [x] "This is the first method.") 
  (method-two ([x] [x y]) "The second method."))
```
> 如果你在my.code命名空间中执行这个例子，将创建如下的变量：
    * `my.code/MyProtocol:`  一个协议对象.
    * `my.code/method-one:`  一个单参数函数.
    * `my.code/method-two:`  一个拥有一个或者两个参数的函数.

> method-one 和 method-two 是多态函数，意即它们可以拥有多个不同对象类型的实现。你可以在协议定义后马上调用它们，但是会得到一个异常，因为它们还没有任何具体实现被定义。

> 什么是一个协议？它是一个约定，一系列功能的集合。一个支持某特定协议的对象或者数据类型（下节会解释）代表它拥有协议中方法的具体实现。


### 如同接口的协议 ###

---

> 在概念上，协议同java 中的接口很类似。实际上，defprotocol 创建了一个包含协议中方法声明的java 接口。你可以采用AOT-compile的方式编译包含协议定义的Clojure源文件，然后在java代码中把它做为接口使用。而这个接口的包会对应协议中的命名空间。包、接口、方法的名称都会遵从java 命名规则，比如会把“-”替换为`“_”`。如果协议中的方法有一个或者多个参数，接口中对应的方法都只会拥有一个参数：一个指针参数。之前的例子会创建一个接口对应下面的java 代码：
```
package my.code;  
public interface MyProtocol{ 
    public Object method_one(); 
    public Object method_two(Object y); 
}
```
> 在协议和接口之间有一个最重要的差别：协议不能继承。你不能象在java 中创建子接口那样创建“子协议”。

> 协议同Ruby中提供的“mix-in”功能也很类似，区别在于：协议不能继承。因此，协议之间永远不会冲突，不像“mix-in”那样。


## 数据类型 ##

---


> 尽管Clojure，严格来说，并不是一门面向对象语言，但有时候处理真实业务情况时，也可以试着用面向对象的方式来进行思考。大部分的应用拥有很多相同“类型”、类似“字段”的“记录”。

> 在Clojure1.2之前，处理很多记录的标准方式是采用map。这是有效的，但是不允许在多个map中重用同样的关键字（key）来做任何性能优化。

> `StructMaps`也曾经是一种解决方案，但同样拥有很多问题。`StructMaps`拥有一个关键字（key）的预定义集合，却没有运行时需要的实际“类型”。`StructMaps`不能被打印，也不能重复，也不能拥有基本类型的字段，最后，还比不上POJO中实例字段的性能。

> Clojure1.2 中引入了数据类型做为`StructMaps`的替代方案。一个数据类型代表一条有名字的记录类型，拥有一个已命名字段的集合，可以是协议或者接口的实现。采用defrecord来创建数据类型：
```
(defrecord name [fields...])
```
> 举个例子，一个数据类型能够存储一条包含两个字段，name 和 room number，的employee记录：
```
user> (defrecord Employee [name room])
```
> 在这个例子中，defrecord创建了一个新的类名叫Employee，它拥有一个默认构造函数，这个构造函数的参数对应创建时参数的类型和字段以及顺序。你可以通过在数据类型名后面加.的方式来创建对应的实例。
```
user> (def emp (Employee. "John Smith" 304))
```
> 数据类型实例的操作就象Clojure map一样。你能够象使用存取器函数一样使用关键字在数据类型对象中检索字段：
```
user> (:name emp) 
"John Smith" 
user> (:room emp) 
304
```
> 这可比在map中查找或者采用`StructMap`的存取器函数要快得多了。数据类型实例同样支持assoc 和 dissoc函数。
```
user=> (defrecord Scientist [name iq]) 
user.Scientist 
user=> (def x (Scientist. "Albert Einstein" 190)) 
#'user/x 
user=> (assoc x :name "Stephen Hawking") 
#:user.Scientist{:name "Stephen Hawking", :iq 190}
```
> 你甚至可以在不改变对象类型的情况下，为对象添加一个原本数据类型没有的字段。
```
user=> (assoc x :field "physics") 
#:user.Scientist{:name "Albert Einstein", :iq 190, :field "physics"}
```
> 然而，如果你采用dissoc函数来清除原本数据类型的关键字，你会得到一个普通的map结果。
```
user=> (dissoc x :iq) 
{:name "Albert Einstein"}
```

## 实现协议和接口 ##

---

> 一个数据类型，单独来看，只是用来存储数据的。一个协议，单独来看，完全没有任何操作。它们两者联合组成了一个强力的抽象概念。一旦定义了一个协议，它能够扩展到支持任何数据类型，我们说数据类型实现了协议。在这个阶段，协议定义的方法在数据类型的实例上就可以被调用了。

### 内联方法 ###

---

> 在使用defrecord创建一个数据类型时，你能够应用方法来实现任意数量的协议。语法如下：
```
(defrecord name [fields...] 
  SomeProtocol 
    (method-one [args] ... method body ...) 
    (method-two [args] ... method body ...) 
  AnotherProtocol 
    (method-three [args] ... method body ...))
```
> 你可以在字段vector后面挂接任意数量的协议和方法。每个方法实现拥有跟对应协议中一样的参数数量。而数据类型的实例字段在方法内部跟方法的局部变量同样有效，直接使用名字就可以调用。
```
(defrecord name [x y z] 
  SomeProtocol 
  (method-one [args] 
    ...do stuff with x, y, and z...))
```
> 有一些仅在方法内部生效的局部变量：defrecord并没有遮蔽语法的作用范围，比如fn，proxy，或者reify，这些将在“具体化匿名数据类型”（Reifying Anonymous Datatypes）章节中讲到。


### 继承java 接口 ###

---

> 数据类型同样也可以实现java 接口的方法。举个例子，你可以实现java.lang.Comparable接口，这样就允许你的新数据类型能够支持Clojure比较函数：
```
user> (defrecord Pair [x y] 
        java.lang.Comparable 
          (compareTo [this other] 
             (let [result (compare x (:x other))] 
               (if (zero? result) 
                 (compare y (:y other)) 
                 result)))) 
#'user/Pair 
user> (compare (Pair 1 2) (Pair 1 2)) 
0 
user> (compare (Pair 1 3) (Pair 1 100)) 
-1
```
> 注意“this“参数，它代表方法被调用时所属的对象，这个对象必须被明确地引入。这代表着java 方法的Clojure实现将比java 方法声明中的参数要多一个。

> 由于大部分Clojure的核心函数都被定义为接口操作，所以它们能够被多种数据类型扩展。Clojure定义了太多接口以至于无法在此一一列出，但是你可以在Clojure的源代码中找到它们。比如seq和rseq函数的源码各自在clojure.lang.Seqable 和 clojure.lang.Reversible下面。在未来的发布版本中（2.0或者更高），这些接口很可能会被重新定义为协议。

> defrecord函数不支持java 类继承，所以它不能覆写java 类的方法，包括抽象类。然而，它允许你覆写java.lang.Object的方法比如hashCode, equals, 和 toString。就像在接口中一样在defrecord中简单引入java.lang.Object，Clojure会基于值自动生成hashCode和equals的方法实现，所以几乎没必要自己去实现这些方法。

> java 接口有时会定义重载方法－－拥有不同参数类型相同名字的系列方法。如果这些方法拥有不同的参数个数（参数数量），只需要挨个定义参数就像是在不同名字的方法里面一样。（不要采用fn的多重参数数量语法。）如果这些方法拥有不同类型的参数，添加类型标签（Chapter 8）去声明它们。


### 如同类的数据类型 ###

---

> 一个数据类型等同于一个拥有公共常量实例字段并实现了任意数量接口的java 类。不过它并没有继承任何类除了java.lang.Object。

> 不同于java 类的是，一个数据类型并不需要实现每个协议或者接口的所有方法。在数据类型中调用并未实现的方法会抛出一个`AbstractMethodError`。

> 当AOT-编译后，defrecord将产生一个java 类，这个类的名字同数据类型的名字一样，包名对应当前的命名空间（服从java 的命名规则，跟协议一样）。生成的java 类将拥有两个构造函数：一个拥有定义时给的所有字段，而另一个除了这些字段外还拥有两个额外的参数；一个元数据map和一个附加字段map，它们中任意一个都有可能是nil。

> 你不能给数据类型添加额外的构造函数，同样不能添加协议或者接口中所没有定义的方法。

> 为了优化数据类型的内存占用，你能够为字段添加简单的类型提示。你同样可以在类名上添加类型提示字段；这不会影响内存使用（所有的指针都是一样的大小），但能够避免反射警告。
```
user> (defrecord Point [#^double x #^double y]) 
#'user/Point 
user> (Point. 1 5) 
#:Point{:x 1.0, :y 5.0}
```

## 扩展协议为已存在类型 ##

---

> 有时你或者需要创建一个新的协议来操作一个已存在的数据类型。假设现在你无法修改defrecord定义的源代码。你仍然可以采用extend函数扩展协议来支持那个数据类型：
```
(extend DatatypeName 
  SomeProtocol   
    {:method-one (fn [x y] ...) 	
     :method-two existing-function} 	 
  AnotherProtocol   
    {...})
```
> extend接收一个数据类型名，后面紧跟任意数量的成对协议/方法map。方法map就是个普通map，方法名是key，关联到方法实现。方法实现可以是fn创建的匿名函数或者已存在函数的标识名。

> 因为extend是一个普通函数，它的所有参数都被评估过。这意味着你可以将方法map存储在一个Var中，然后重用它去实现多个数据类型，这个功能非常类似mix-ins。
```
(def defaults 
     {:method-one (fn [x y] ...) 
      :method-two (fn [] ...)}) 	  
(extend DefaultType 
  SomeProtocol   
    defaults) 	
(extend AnotherType 
  SomeProtocol   
    (assoc defaults :method-two (fn ...)))
```
> 有两个便利的宏能够简化扩展语法，extend-type和extend-protocol。当你想要使用同一数据类型实现多个协议时，采用extend-type；当你想要使用多个数据类型实现同一协议时，采用extend-protocol。
```
(extend-type DatatypeName 
  SomeProtocol   
    (method-one [x] ... method body ...) 	
    (method-two [x] ...) 	
  AnotherProtocol   
    (method-three [x] ...))  
(extend-protocol SomeProtocol 
  SomeDatatype   
     (method-one [x] ...) 	 
     (method-two [x y] ...) 	 
  AnotherType   
     (method-one [x] ...) 	 
     (method-two [x y] ...))
```
> > 采用extend和它关联的宏添加的方法附属于协议，而不是数据类型本身。这使得它们更加灵活（它们作用于标准的java 类，在接下来的章节中会讲到），但比起直接内嵌在defrecord中的方法来要稍微低效一些。


### 继承java 类和接口 ###

---


> 数据类型和协议是一种强力的抽象类型，但是你经常不得不涉及那些你没有源代码的java类。java 并没有提供一种为已经存在的类添加新接口的方法（被称为接口注入），但Clojure协议可以被扩展为支持已存在的java 类。

> extend, extend-type, 和 extend-protocol都接受java 类做为”类型“。这同样可以作用于接口上。你能够编写`(extend-type SomeInterface...)`来为所有实现了`SomeInterface`的类添加一个协议。这开创了多重继承或者实现的可能性，因为一个类可以实现不止一个接口；结果是当前未定义的情况应该被避免。


## 具体化匿名数据类型 ##

---

> 有时你需要一个实现了某一协议或者接口的对象，但是你确实不想创建一个命名的数据类型。Clojure1.2通过reify宏来支持这点：
```
(reify 
  SomeProtocol   
    (method-one [] ...) 	
    (method-two [y] ...) 	
  AnotherProtocol   
    (method-three [] ...)) 
```
> reify的语法和defrecord非常相似，除了字段vector。同样地，象defrecord一样，reify可以继承java 接口或者java.lang.Object的方法。

> 不同于defrecord，reify的方法体是闭包语法的，就像fn创建匿名函数一样，所以它能获取本地变量：
```
user> (def thing (let [s "Capture me!"] 
                    (reify java.lang.Object 
                       (toString [] s)))) 
#'user/thing 
user> (str thing) 
"Capture me!"
```
> 原来需要使用代理来处理的很多情况现在都可以使用reify来处理。在这些场合下，reify将比代理更快更简单。然而，reify仅限于实现接口，它不能象代理那样覆写基类的方法。

> 在概念上，reify扮演了java 中匿名内部类同样的角色。


## 致力于使用数据类型和协议 ##

---

> 数据类型和协议是Clojure中显著的新特性，它们将对大部分Clojure程序的编写产生重大影响。标准和最佳实践仍在发展中，但很少出现指导方针：

  * 比起代理来，最好是使用reify，除非你需要去覆写基类的方法。
  * 比起gen-class来，最好是使用defrecord，除非你需要gen-class的特性去做java 交互
  * 在任何情况下，最好是使用defrecord，而不是defstruct。
  * 采用协议定义你的抽象，而不是接口。
  * 在单参数，基于类型来区分多个方法的情况下，最好是采用协议而不是multimethods。
  * 仅在消除歧义或者性能需求（Chapter 14）的情况下添加类型提示；大部分情况下，类型会被自动识别。

> 数据类型和协议并没有移除已存在的任何特性：defstruct, gen-class, proxy, 和multimethods这些都还在。只是defstruct已经有可能被废弃掉。

> 协议/数据类型和java 类的最大不同就在于没有继承。协议的扩展机制被设计为允许在不进行具体继承的情况下重用方法，因为具体继承总是关联着问题。


### 一个完整的示例 ###

---

> 这儿有个采用了协议和数据类型的经典“payroll”示例版本，这个payroll系统将拥有一个方法来基于雇员的工作时间来计算他们的薪水：
```
(defprotocol Payroll 
  (paycheck [emp hrs])) 
```
> 然后有两种类型的雇员：钟点工（“hourly” employees）根据小时拿钱，而正式工（“salaried” employees）每月领取固定的一份薪水，而不管他们工作了多少个小时：
```
(defrecord HourlyEmployee [name rate] 
  Payroll   
  (paycheck [hrs] (* rate hrs)))  
(defrecord SalariedEmployee [name salary] 
  Payroll   
  (paycheck [hrs] (/ salary 12.0)))
```
> 注意你并没有定义一个“是什么”（IS-A）关系。这儿并没有“雇员”（“Employee”）基本类型；也不需要。上面的定义只是表明了：有两个存在的类型，都支持Payroll的paycheck方法。

> 现在你能够定义一些雇员并计算他们的薪水了：
```
user=> (def emp1 (HourlyEmployee. "Devin" 12)) 
user=> (def emp2 (SalariedEmployee. "Casey" 30000)) 
user=> (paycheck emp1 105) 
1260 
user=> (paycheck emp2 120) 
2500.0
```
> 可能你同样需要派发薪水给承包商：在这种情况下，承包商的薪水往往在他们已经开始工作前就定下了。这应当是另外一个数据类型了，不过你还是能够采用reify来实现它：
```
(defn contract [amount] 
  (reify Payroll (paycheck [hrs] amount)))
```
> > 就像下面例子展示的那样:
```
user=> (def con1 (contract 5000)) 
user=> (paycheck con1 80) 
5000
```

## 高级数据类型 ##

---


> defrecord定义的数据类型在存储结构化数据上面非常有用，但是根本上而言，它们总是表现得象map。如果你希望定义一个彻底全新的类型，表现得完全不像map，那么就采用deftype来代替defrecord。deftype是个defrecord的“低层次”版本。
```
(deftype name [fields...] 
  SomeProtocol   
    (some-method [this x y] ...) 	
  SomeInterface   
    (aMethod [this] ...))
```
> deftype的语法基本和defrecord一样，但是deftype不会为你创建任何默认的方法实现。你必须自己实现所有的方法，包括标准对象方法比如equals 和 hashCode。deftype创建一个“空”的java 类；这个类旨在允许以Clojure的方式重定义核心数据结构，比如vectors 或者 maps。


## 摘要 ##

---

> 数据类型和协议是Clojure1.2中两个最令人激动的新特性。它们提供了一种强力的解决方案来处理面向对象设计能解决的问题，但并不采用面向对象中过时的继承实现概念。实际上，数据类型和协议同面向对象设计的早期研究具有显著的相似。它们优美地处理了往已存在的类型上添加新函数的问题，这有时候被称为“表达式问题”。因为它们基于java 平台高度优化的方法分发设计，所以能提供良好的性能。

术语对照：

数据类型－－datatype

协议－－－－protocol

方法－－－－method

函数－－－－function

类－－－－－class

对象－－－－object

实现－－－－implement

抽象－－－－abstraction

扩展－－－－extend

继承－－－－inheritance

面向对象－－object-oriented