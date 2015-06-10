## 章九 多路方法和层次结构 ##

### 没有类的运行时多态 ###
<p>
以传统的对类和方法的观点来看，Clojure不是一门面向对象的语言，虽然它构建在Java面向对象的基础设施之上。<br />
大多数主流的，像Java和C++这样的面向对象语言，用类定义树形的类型层次结构，并提供这些类型所包含的方法的实现。<br />
Clojure分离了类型层次结构和方法实现，极大地简化了多重继承带来的问题。另外，它允许你在同一个类型上建立多重的，独立的层级结构。这使得定义更贴近现实世界的IS-A关系模型成为可能 。<br>
</p>

### 多路方法 ###
<p>
Clojure 多路方法为运行时多态的分发提供了支持。它允许你给一个方法定义提供多个实现。在运行时，哪个实现被执行取决于方法被赋予的参数。<br />
很多面向对象语言的方法分发，是单变量的、基于类型的，哪个方法被执行，完全由第一个参数所属的类型或类来决定。方法在第一个参数上被调用。Java和C++都把第一个参数放到方法名前面来明确它的重要意义。<br />
Clojure 多路方法更为灵活。它支持多重分派，任何变量都能把实现挂到同一个方法上。同样地，分发将基于变量的任何特征，而不单单是类型。<br />
多路方法使用关键字<b>defmulti</b>创建，使用<b>defmethod</b>实现。<br>
<pre><code><br>
(defmulti name dispatch-fn)<br>
(defmethod multifn dispatch-value [args...] &amp; body)<br>
</code></pre>

多路方法的调用和普通方法是一样的。当你调用它的时候，分发函数被赋予和多路方法相同的参数立即被调用。分发函数返回的值叫做分发值。然后Clojure搜索分发值对应的方法（由defmethod定义）。<br />
假设你在写一个幻想类角色扮演游戏，里面有不同种类的生物：人类、精灵、兽人、等等。每个生物用一个map描述，如下：<br>
<pre><code><br>
(def a {:name "Arthur", :species ::human, :strength 8})<br>
(def b {:name "Balfor", :species ::elf, :strength 7})<br>
(def c {:name "Calis", :species ::elf, :strength 5})<br>
(def d {:name "Drung", :species ::orc, :strength 6})<br>
</code></pre>

将名称空间限定的关键字用于物种（::human代替:human），这个后面详述起重要性。（第七章有关于限定关键字的解释）<br />
现在你可以定义一个区分不同物种生物的多路方法。例如，你能够给每个物种不同的移动方式。<br>
<pre><code><br>
(defmulti move :species)<br>
<br>
(defmethod move ::elf [creature]<br>
(str (:name creature) " runs swiftly."))<br>
<br>
(defmethod move ::human [creature]<br>
(str (:name creature) " walks steadily."))<br>
<br>
(defmethod move ::orc [creature]<br>
(str (:name creature) " stomps heavily."))<br>
</code></pre>

当你调用<b>move</b>的时候，适当的方法会被调用。<br>
<pre><code><br>
user=&gt; (move a)<br>
"Arthur walks steadily."<br>
user=&gt; (move b)<br>
"Balfor runs swiftly."<br>
user=&gt; (move c)<br>
"Calis runs swiftly."<br>
</code></pre>

发生了什么？当你调用<b>（move a）</b>，首先Clojure为定义于关键字<b>:species</b>上的<b>move</b>多路方法调用分发方法。那个关键字，会被用于从map中找到一个对应值。所以<b>（move a）</b>调用<b>（:space a）</b>，返回<b>::human</b>。然后Clojure会为<b>move</b>找到一个分发值为<b>::human</b>的方法，调用它。<br />
相同的行为可以被有条件地实现。多路方法的优点是你能在任何时候添加新方法。当你增加一个新物种生物的时候，你可以简单的定义另一个<b>move</b>方法，而不用修改以后的代码。<br />
分发方法可不是只能使用简单的关键字；任何函数都是可以的。例如，你能让分发方法按照力量对生物分类。<br>
<pre><code><br>
(defmulti attack (fn [creature]<br>
(if (&gt; (:strength creature) 5)<br>
:strong<br>
:weak)))<br>
<br>
(defmethod attack :strong [creature]<br>
(str (:name creature) " attacks mightily."))<br>
<br>
(defmethod attack :weak [creature]<br>
(str (:name creature) " attacks feebly."))<br>
</code></pre>

当你调用<b>attack</b>多路方法的时候，会先调用匿名方法<b>fn</b>，它返回<b>:strong</b>或者<b>:weak</b>。这个关键词（分发值）决定了哪个<b>attack</b>方法被掉用。<br>
<pre><code><br>
user=&gt; (attack c)<br>
"Calis attacks feebly."<br>
user=&gt; (attack d)<br>
"Drung attacks mightily."<br>
</code></pre>
</p>

#### 多重分发 ####
<p>
就像我在这章最开始时候说的那样，多路方法提供了对多参数的分发的支持。为了做到这点，分发方法返回一个vecter。例如，在游戏中，定义一个多路方法描述两个不同物种碰面时候会发生什么。比方说精灵，它们和兽人敌对，和其他种族友好：<br>
<pre><code><br>
(defmulti encounter (fn [x y]<br>
[(:species x) (:species y)]))<br>
(defmethod encounter [::elf ::orc] [elf orc]<br>
(str "Brave elf " (:name elf)<br>
" attacks evil orc " (:name orc)))<br>
(defmethod encounter [::orc ::elf] [orc elf]<br>
(str "Evil orc " (:name orc)<br>
" attacks innocent elf " (:name elf)))<br>
(defmethod encounter [::elf ::elf] [orc1 orc2]<br>
(str "Two elves, " (:name orc1)<br>
" and " (:name orc2)<br>
", greet each other."))<br>
</code></pre>

注意方法和多路方法的参数名不必相同，但是分发方法和方法的参数数量必须相同。<br />
现在以两个生物为参数调用<b>encounter</b>多路放下，看会发生什么：<br>
<pre><code><br>
user=&gt; (encounter b c)<br>
"Two elves, Balfor and Calis, greet each other."<br>
user=&gt; (encounter d b)<br>
"Evil orc Drung attacks innocent elf Balfor"<br>
</code></pre>
</p>

#### 默认分派值 ####
<p>
注意你并没有为所有物种的组合都定义<b>encounter</b>方法。如果你试图用一个未知的组合调用<b>encounter</b>，将会报错：<br>
<pre><code><br>
user=&gt; (encounter a c)<br>
java.lang.IllegalArgumentException:<br>
No method in multimethod 'encounter'<br>
for dispatch value: [:user/human :user/elf]<br>
</code></pre>

或者你能保证定义对所有可能的组合都可用的方法，或者你以关键字<b>:default</b>为分发值提供一个默认实现。<br>
<pre><code><br>
(defmethod encounter :default [x y]<br>
(str (:name x) " and " (:name y)<br>
" ignore each other."))<br>
</code></pre>

默认方法将在没有其他方法匹配得上的时候被调用：<br>
<pre><code><br>
user=&gt; (encounter a c)<br>
"Arthur and Calis ignore each other."<br>
</code></pre>

你可以像下面这样，给<b>defmulti</b>增加一个<b>:default</b>选项来提供一个可选的默认分发值：<br>
<pre><code><br>
(defmulti talk :species :default "other")<br>
(defmethod talk ::orc [creature]<br>
(str (:name creature) " grunts."))<br>
(defmethod talk "other" [creature]<br>
(str (:name creature) " speaks."))<br>
</code></pre>
</p>

### 层次结构 ###
<p>
在很多面向对象的语言中，类型层次结构都是由类和子类间的继承关系隐式定义的。类也定义方法实现，这种关系显得紧凑但不灵活，尤其在像C++那样允许多继承的语言中。Java不允许多继承避开了这个问题，但是这又使得对现实世界建模变得很困难。<br />
在Clojure中，继承和方法实现是完全分离的，所以它比基于类继承结构更灵活。它支持几乎所有的组合关系，包括多继承和多根。<br />
Clojure定义一个叫做“global”的层次，我们把它描述为第一个。你也可以定义独立的层级结构，在本节最后会讨论这个问题。<br />
<pre><code><br>
(derive child parent)<br>
</code></pre>

derive在亲和子之间创建一个IS-A关系。亲和子以标签的形式被引用，因为它们被用于标识类型或者类别。标签可以是关键字或者符号，而且（在global层次）必须是名称空间限定（参考第7章）的。<br />
继续说幻想游戏，你可以定义生物的类型用于共享特定的属性。例如，人类和精灵都是好的而兽人是坏的：<br>
<pre><code><br>
user=&gt; (derive ::human ::good)<br>
user=&gt; (derive ::elf ::good)<br>
user=&gt; (derive ::orc ::evil)<br>
</code></pre>

精灵和兽人是魔法生物：<br>
<pre><code><br>
user=&gt; (derive ::elf ::magical)<br>
user=&gt; (derive ::orc ::magical)<br>
</code></pre>

为了添加点儿乐趣，再增加一种特殊的人类，英雄：<br>
<pre><code><br>
user=&gt; (derive ::hero ::human)<br>
</code></pre>

我们创建了如附表9-1所示的关系图。<br>
<b>figure-9-1.jpg here</b>
</p>

#### 查询层次结构 ####
<p>
一旦你定义好这些关系，你就可以用<b>isa?</b>方法来查询它们：<br>
<pre><code><br>
(isa? child parent)<br>
</code></pre>

当子派生于（直接或者间接地）亲的时候，<b>isa?</b>返回<b>true</b>。如一下代码所示：<br>
<pre><code><br>
user=&gt; (isa? ::orc ::good)<br>
false<br>
user=&gt; (isa? ::hero ::good)<br>
true<br>
user=&gt; (isa? ::hero ::magical)<br>
false<br>
</code></pre>

当子和亲相同（由Clojure的=函数定义）的时候，<b>isa?</b>同样会返回<b>true</b>：<br>
<pre><code><br>
user=&gt; (isa? ::human ::human)<br>
true<br>
</code></pre>
</p>

### 多路方法的层次结构 ###
<p>
当多路方法在查找正确的方法调用的时候，它会使用<b>isa?</b>方法比较分发值。也就是说多路方法不仅能够对显式的类型进行分发，对派生也同样有效。这是一个只作用于魔法生物的多路方法：<br>
<pre><code><br>
(defmulti cast-spell :species)<br>
<br>
(defmethod cast-spell ::magical [creature]<br>
(str (:name creature) " casts a spell."))<br>
<br>
(defmethod cast-spell :default [creature]<br>
(str "No, " (:name creature) " is not magical!"))<br>
<br>
user=&gt; (cast-spell c)<br>
"Calis casts a spell."<br>
user=&gt; (cast-spell a)<br>
"No, Arthur is not magical!"<br>
</code></pre>

当分发值是一个vector的时候，多路方法用<b>isa?</b>从左到右比较vector的每一个元素，多参数组合分发到层级结构。例如，你可以基于好、坏生物，重新定义你的multimethod多路方法。<br>
<pre><code><br>
(defmulti encounter (fn [x y]<br>
[(:species x) (:species y)]))<br>
<br>
(defmethod encounter [::good ::good] [x y]<br>
(str (:name x) " and " (:name y) " say hello."))<br>
<br>
(defmethod encounter [::good ::evil] [x y]<br>
(str (:name x) " is attacked by " (:name y)))<br>
<br>
(defmethod encounter [::evil ::good] [x y]<br>
(str (:name x) " attacks " (:name y)))<br>
<br>
(defmethod encounter :default [x y]<br>
(str (:name x) " and " (:name y)<br>
" ignore one another."))<br>
<br>
user=&gt; (encounter c a)<br>
"Calis and Arthur say hello."<br>
user=&gt; (encounter a d)<br>
"Arthur is attacked by Drung"<br>
</code></pre>
</p>

#### Java的层次结构 ####
<p>
Clojure的层级结构可以集成和扩展Java的类层级结构。除了符号和关键字之外，子也可以派生自Java类。在JDK中没有类能够跟幻想世界相匹配，但是你可以指定Java Date类是坏的：<br>
<pre><code><br>
user=&gt; (derive java.util.Date ::evil)<br>
</code></pre>

isa?函数能够同时理解层级结构和Java类关系：<br>
<pre><code><br>
user=&gt; (isa? java.util.Date ::evil)<br>
true<br>
user=&gt; (isa? Float Number)<br>
true<br>
</code></pre>

你可以定义多路方法，就像Java方法一样在类上进行分发。举个例子，invert多路方法，能同时工作在数字（取负数）和字符串（反转）上：<br>
<pre><code><br>
(defmulti invert class)<br>
(defmethod invert Number [x]<br>
(- x))<br>
(defmethod invert String [x]<br>
(apply str (reverse x)))<br>
user=&gt; (invert 3.14)<br>
-3.14<br>
user=&gt; (invert<br>
</code></pre>
</p>

#### 更多的层次结构查询 ####
<p>
3个方法展示出了关于层次结构的更多的信息。<br>
<pre><code><br>
(parents tag)<br>
(ancestors tag)<br>
(descendants tag)<br>
</code></pre>

个方法都返回几何。parents返回当前亲标签，ancestors返回所有当前的和间接的亲。descendants返回所有的当前的和间接地子标签<br>
parents和ancestors同样适用于Java类；descendants对Java类不适用（受限于Java的类型系统）。<br>
<pre><code><br>
user=&gt; (parents ::orc)<br>
#{:user/magical :user/evil}<br>
user=&gt; (descendants ::good)<br>
#{:user/elf :user/hero :user/human}<br>
user=&gt; (parents ::hero)<br>
#{:user/human}<br>
user=&gt; (ancestors ::hero)<br>
#{:user/good :user/human}<br>
user=&gt; (parents java.util.Date)<br>
#{java.lang.Object java.lang.Cloneable<br>
java.io.Serializable java.lang.Comparable<br>
:user/evil}<br>
</code></pre>
</p>

注意java.util.Date的祖先，包括Java类层次结构定义的和**derive**定义的。
#### 解决冲突 ####
<p>
因为Clojure的层级结构允许多继承，所以当多路方法有多个合法的选项的时候就会产生问题。Clojure不知道选择哪个，所以就会抛出异常。<br />
举个例子，考虑在你的幻想游戏中，一个多路方法同时有分发值<b>::good</b>和<b>::magical</b>：<br>
<pre><code><br>
(defmulti slay :species)<br>
<br>
(defmethod slay ::good [creature]<br>
(str "Oh no! A good creature was slain!"))<br>
<br>
(defmethod slay ::magical [creature]<br>
(str "A magical creature was slain!"))<br>
</code></pre>

当你干掉一个人类或者一个兽人的时候，你确定会发生什么：<br>
<pre><code><br>
user=&gt; (slay a) ;; human<br>
"Oh no! A good creature was slain!"<br>
user=&gt; (slay d) ;; orc<br>
"A magical creature was slain!"<br>
</code></pre>

但是如果你干掉一个精灵呢？<br>
<pre><code><br>
user=&gt; (slay b)<br>
java.lang.IllegalArgumentException:<br>
Multiple methods in multimethod 'slay' match<br>
dispatch value: :user/elf -&gt; :user/magical<br>
and :user/good, and neither is preferred<br>
</code></pre>

从这个异常中发现，::elf同时派生自::magical和::good，而它们都有slay方法。<br />
为了解决这个问题，我们必须制定分发值的匹配顺序。prefer-method方法带一个多路方法为参数，指定哪个分发值比别的占优：<br>
<pre><code><br>
(prefer-method multimethod preferred-value other-value)<br>
</code></pre>

<pre><code><br>
user=&gt; (prefer-method slay ::good ::magical)<br>
user=&gt; (slay b)<br>
"Oh no! A good creature was slain!"<br>
</code></pre>

第二种解决办法（其实不能算作一种解决方案），简单地移除其中一个冲突的方法。remove-method方法带一个多路方法为参数和一个分发值，它会删除该分发值对应的方法。<br>
<pre><code><br>
(remove-method multimethod dispatch-value)<br>
</code></pre>

<pre><code><br>
user=&gt; (remove-method slay ::magical)<br>
user=&gt; (slay b)<br>
"Oh no! A good creature was slain!"<br>
</code></pre>
</p>

#### 类型标记 ####
<p>
<b>type</b>方法是<b>class</b>的一个通用的版本。<b>type</b>查找并返回关于它的参数的<b>:type</b>元数据（参考第8章）。如果那个对象没有<b>:type</b>元数据，或者不支持元数据，那么会返回对象所属的类：<br>
<pre><code><br>
user=&gt; (type (with-meta {:name "Bob"} {:type ::person}))<br>
:user/person<br>
user=&gt; (type 42)<br>
java.lang.Integer<br>
user=&gt; (type {:name "Alice"})<br>
clojure.lang.PersistentArrayMap<br>
</code></pre>

如果你用<b>:type</b>元数据重新定义游戏中的生物种族：<br>
<pre><code><br>
(def a (with-meta {:name "Arthur", :strength 8}<br>
{:type ::human}))<br>
(def b (with-meta {:name "Balfor", :strength 7}<br>
{:type ::elf}))<br>
</code></pre>

你就可以重新定义<b>move</b>多路方法，以类型来分发：<br>
<pre><code><br>
(defmulti move type)<br>
<br>
(defmethod move ::elf [creature]<br>
(str (:name creature) " runs swiftly."))<br>
<br>
(defmethod move ::human [creature]<br>
(str (:name creature) " walks steadily."))<br>
</code></pre>

这就使得<b>move</b>多路方法能够同时处理元数据可用的Clojure的数据结构和原生的Java对象：<br>
<pre><code><br>
(defmethod move Number [n]<br>
(str "What?! Numbers don't move!"))<br>
<br>
user=&gt; (move a)<br>
"Arthur walks steadily."<br>
user=&gt; (move b)<br>
"Balfor runs swiftly."<br>
user=&gt; (move 6.022)<br>
"What?! Numbers don't move!"<br>
</code></pre>
</p>

### 用户定义的层次结构 ###
<p>
除全局层次结构之外，你还可以创建你自己的独立的层次构。<b>makehierarchy</b>方法返回一个新的曾次结构（也就是一个亲、子关系的map）。<b>derive</b>、<b>isa?</b>、<b>parents</b>、<b>ancestors</b>和<b>descendants</b>方法都接受一个附加的first参数，指定使用的层次机构。<br />
不同于全局层次机构，用户定义的层次机构允许使用非限定（没有名称空间）的关键字或者符号作为标签。<br />
注意使用<b>derive</b>创建用户自定义层次机构时，有些许的不同。使用两个参数调用时，<b>derive</b>修改全局层次结构。但是用户定义的层次机构是不可变的，就像Clojure其他的数据结构一样，所以3参数版本的<b>derive</b>返回修改之后的层次结构。如下例所示：<br>
<pre><code><br>
user=&gt; (def h (make-hierarchy))<br>
user=&gt; (derive h :child :parent)<br>
user=&gt; (isa? h :child :parent)<br>
false<br>
</code></pre>

因此构建用户自定义层次机构的过程，必须跟<b>derive</b>语句串起来，就像下面例子里那样：<br>
<pre><code><br>
user=&gt; (def h (-&gt; (make-hierarchy)<br>
(derive :one :base)<br>
(derive :two :base)<br>
(derive :three :two)))<br>
user=&gt; (isa? h :three :base)<br>
true<br>
</code></pre>

或者使用一个Clojure的可变类型的引用，像是<b>Var</b>：<br>
<pre><code><br>
user=&gt; (def h (make-hierarchy))<br>
user=&gt; (isa? h :child :parent)<br>
false<br>
user=&gt; (alter-var-root (var h) derive :child :parent)<br>
user=&gt; (isa? h :child :parent)<br>
true<br>
</code></pre>

默认，多路方法使用全局层次结构。<b>defmulti</b>接受一个可选参数<b>:hierarchy</b>，后跟一个将要使用的不同的层次结构。<br>
</p>

### 总结 ###
<p>
多路方法非常灵活，但是灵活性是有代价的：不是很高效。考虑一下每次我们调用多路方法时候会发生什么：为了能找到正确的方法，它不得不去调用一个分发函数，查找哈希表里的分发值，然后最少做一次<b>isa?</b>对比。像Hotspot这样的小型编译器再优化的烂点儿，oh, my God<br />
结果就是，多路方法对于频繁调用的低级别函数不适合。这就是Clojure的内置函数都不用多路方法实现的原因。尽管如此，它用来创建可扩展的高级别API是非常棒的。分别在1.2节和13章介绍和详细描述的协议，提供了一个严格的方法分发的形式，并有着不错的性能。<br>
</p>


## 译者注 ##
参考 [使用R编写统计程序，第3部分:可重用和面向对象编程 - 面向对象的R](http://www.ibm.com/developerworks/cn/linux/l-r3.html#N10139)