#第二章 基础Rete算法

既然本论文的大部分工作都以Rete匹配算法为基础，本章主要描述Rete算法。不幸的是，大多数文献中对Rete算法的描述都不够特别明晰，
这也许是Rete被冠以`极度难懂`的帽子的原因（Perlin，1990b）。为了扭转这种形势，本章将以教程的形式描述Rete，而不是简略地回顾
一下并引导读者去看完整描述的文献资料。我们首先大体上看一下Rete，然后讨论数据结构和一般情况下实现程序的原则，这就是本章可以
给想在他们自己系统中实现Rete算法或变体的读者的一个指导。已经熟悉Rete或者想了解本论文研究贡献的人可以跳过本章。2.6节和下面
的章节讨论Rete的高级特性，第一性阅读时也可以跳过，本篇论文剩余的大部分没有它们也可以理解。

在开始我们的Rete讨论之前，先回顾下一些基本的术语和记法。Rete（通常发音为`REET`或者`REE-tee`，源自拉丁语中网络的意思）涉及
生产区（Production Memeory，简称PM）和工作区（Working Memory，简称WM）。它们中的每一个随着时间可以逐渐地改变。工作区是一组
代表系统当前形势的事实（Fact）--外部世界的状态和（或者）系统自己内部要解决问题的状态。WM中每个元素被叫做工作区元素（Working
Memory Element,简称WME）。在`积木世界`系统里，比如说，工作区也许由下面的WMEs（记为w1-w9.）组成：
<pre><code>
w1:(B1 ^on B2)
w2:(B1 ^on B3)
w3:(B1 ^color red)
w4:(B2 ^on table)
w5:(B2 ^left-of B3)
w6:(B2 ^color blue) 
w7:(B3 ^left-of B4)
w8:(B3 ^on table) 
w9:(B3 ^color red) 
</code></pre>

为了简化我们的描述，我们会假设WMEs是三元组的形式，把它们写成`(identifier ^attribute value)`。“identifier","attribute","value"
对匹配器来说没有特殊的含义。我们有时把”identifier"缩写成`id`，”attribute“缩写成`attr`。我们会把这部分作为一个WME的三个域，
比如WME(B1 ^on B2)有B1在它的id域里。每个域包含一个符号。什么符号是被允许的唯一约束是它们都必须是常量：WMEs中不允许有变量。
Rete不要求特殊表示--众多的Rete版本中支持其它已经实现的表示，我们会在2.11节讨论这些。在这里我们选择该特殊形式是因为：
* 它非常简单
* 对WMEs这种形式的约束并没有带来表示能力的丧失，由于其它更少约束的表示能够被直接地、机械地转换成这种形式，我们会在2.11节看到它
* 试验台系统在本论文中随后描述使用这种表示

生产区（PM）是一组生产（或者叫规则，rules）。一条规则被指定为一组条件（conditions）,共同被称为左边（left-hand side，LHS），
和一组行为（Actioins），共同被称为右手边（Right-hand Side）。规则通常以下面的形式写成：
<pre><code>
 (name-of-this-production
   LHS  /* 一个或多个条件*/
  -->
   RHS  /* 一个或多个行为 */
 )
</code></pre>

匹配算法通常会忽略行为只处理条件。匹配算法作为一个模块或者子程序被整个系统使用以确定哪个规则的所有条件都是满足的，然后系统的
一些其它部分处理合适的规则的行为。因此，本章将会聚焦在条件上。条件可以包含变量，我们把它写在尖括号里，比如：<x>。在我们的
”积木世界“例子中，下面的规则可以被用作的查找红方块左边两个或更多的方块：
<pre><code>
(find-stack-of-two-blocks-to-the-left-of-a-red-block
  (x ^on y)
  (y ^left-of z)
  (z ^color red)
 -->
  ... RHS ...
)
</code></pre>

图2.1：做为黑盒子的Rete算法的输入输出

如果一条规则的所有条件匹配上了当前工作区里的所有元素，并且条件中的任意变量始终绑定，那么规则被称作匹配当前工作区。比如，
上面的的规则匹配上了上面的工作区，因为它的三个条件被w1、w5、w9分别匹配上，<y>绑定到B2在第一个和第二个条件中，<z>绑定到B3
在第二和第三个条件上。该匹配算法的工作是确定哪个系统中的哪个规则匹配上当前的工作区，并且对每一个，确定哪个一个WME匹配哪个
条件。请注意既然我们假设WMEs都是三元组，我们也能假设条件有同样的形式--一个非该形式的条件永远不会匹配上任何WME，也就是说
它是无用的。

如图2.1所示，Rete被视作一个`黑盒子`。作为输入，它获取对当前工作区的变更信息（例如，增加WME）或者一组规则（例如，增加规则）。
每次它被通知其中一个变化，匹配算法必须输出对匹配规则集的任意改变（比如，现在规则匹配上了这些WMEs）。

##2.1 概览
我们开始对Rete做一个简短的概览。如图2.2所示，Rete用一个数据流网络表示规则的条件。这个网络有两部分。Alpha部分执行常量测试
在WME上（测试像red和left-of这样的常量标识）。它的输出被存储在`alpha寄存器`（AM），每一个AM持有通过所有常量测试的个体条件的
当前WME集合。例如，在图示中，第一个条件(<x> ^on <y>)的AM，持有属性域包含符号`on`的WMEs。alpha网络的实现在2.2节讨论。网络
的beta部分首要包含`join节点`和`beta寄存器`（随后我们讨论几种其它的节点类型）。`Join节点`绑定在条件之间的变量一致性的测试。Beta
寄存器部分规则的实例（例如，匹配规则部分而非全部条件的WME组合）。这些部分实例被称作`tokens`。

图2.2 Rete网络示例，(a)几条规则，（b）一条规则的实例化网络

严格地说，在Rete的大部分版本中，alpha网络不仅执行常量测试并且也执行内部条件变量绑定一致性测试，此时一个变量在一个单独的
条件中出现多于一次，比如，(<x> ^self <x>)。这样的测试在Soar中很少见，因此我们不在这里做过多的讨论。同时，注意`测试`实际
上可以是任何布尔值的函数。在Soar中，大部分测试是相等测试（检查一件事件是否等于另外一件），因此我们在这里主要侧重在这种
情况。无论如何，Rete的实现通常至少支持简单的条件测试（大于，小于，等等），一些版本允许任意用户自定义的测试。任何实现中，
基本思路是alpha网络执行包含单个WME的所有测试，而beta网络执行包含两个或两个以上WME的测试。

和关系数据库的类比也许是有帮助的。把当前的工作区当作一个列表，每条规则当作一个查询。在一个条件中的常量测试表示在WM
列表作`SELECT`操作。对系统中的每一个不同的条件Ci，alpha寄存器存储`SELECT`的结果R(Ci)。现在，用P表示包含条件C1,...,Ck，
P和当前工作区的匹配结果（如果P有任何的匹配）表示为R(C1) & ... & R(Ck)，`JOIN`操作执行合适的变量绑定一致性检查。rete网络
的beta部分中的join节点执行这些join操作，同时每个beta寄存器存储其中一个中间join的结果 R(c1) & ... & R(Ci) (i<k)。

无论何时工作区发生变更，我们更新这些`SELECT`和`JOIN`结果。步骤如下：工作区变更通过alpha网络被发送出去同时相应的alpha寄存器
被更新。然后这些变更被传递给相连的join节点，并`激活`这些节点。如果任何新的部分实例被创建，他们被添加到相应的beta寄存器然后
被向下传递到网络的beta部分，激活其它的节点。每当传递达到网络的最底层，它表明一条规则的条件被完全匹配到。这个通常靠
在网络底层定义一个规则（p-node）的特殊节点来实现。每当p-node被激活，它意味着一个新的发现匹配完成（在一些系统相关的方法）。

Rete算法的大部分代码是由处理各种节点激活的程序组成。一个alpha寄存器节点的激活靠添加一个给定的WME到寄存器来处理，然后传递
WME到寄存器的继承者（Join节点会连接它）。相似地，beta寄存器节点的激活靠添加一个给定的token到寄存器来处理然后把它传递到该
节点的子节点（join节点）。总的来说，beta网络中从一个节点到另外一个节点的激活被称作`左激活`（left activation），来自alpha
寄存器的节点的激活被称作`右激活`（right activation）。因此，一个join节点能够招致两种类型的激活：当一个WME被添加到它的alpha
寄存器的右激活，和一个token被添加到beta寄存器的左激活。左右join节点激活通常在两个不同的程序里实现，但在这两种情况下，都搜索
节点的其它寄存器是否存在和新元素有同样变量绑定的元素，如果有发现，它们将被传向join节点的子节点。

为了激活一个给定的节点，那么我们使用一个程序调用。这个特殊的程序依赖于节点的类型，比如，beta寄存器节点的左激活被一个程序
处理，另外一个不同的程序处理join节点的左激活。为了从某一节点传递数据流到它的继承者，我们迭代这些继承者并且为每个调用适合
的激活程序。我们靠 查找节点的类型来确认哪一个程序是适合的：我们使用switch或者case表达式取决于节点类型，每个分支调用一个不
同的程序，或者我们通过一个根据节点类型进行索引的跳转表来作程序调用。（为了效率有些编译器把switch表达式转成跳转表）。

Rete有两个重要的特性使之比原生匹配算法要快的多。第一个是`状态保存`。每一次对WM的变更，匹配的状态（结果）被保存在alpha和
beta寄存器。在WM再次变更后，很多或大部分结果通常是没有变化的，因此Rete靠保持成功WM变化间的结果避免了很重复计算。（Rete为
这样的系统而设计，在连续匹配的规则上有很少的WME变更。Rete的状态保存在WME每次都改变的系统中不是很有用）。

Rete的每二个重要的特性是在具有相似条件的规则间的`节点共享`。不同类别的共享出现在网络的不同部分。在alpha网络中的共享，如
2.2节讨论的。对alpha网络的输出，当两个或以上规则有同样的条件，Rete只使用一个AM来存储这些条件，而不是为每一个条件创建重复
的寄存器。例如，在图2.2(a)中，C3的AM被规则P1和P3所共享。此外，在网络的beta部分，当两个或以上的规则第一次有相同的少量条件
时，同一节点被来匹配这些条件。这避免了这些规则的重复匹配。在图示中，三个规则胡相同的前两个条件，二个规则有相同的前三个条
件。因为共享，beta部分构成了一棵树。

Rete的实现可以解释或编译。在解释版本中，网络的描述仅被简单地存储为解释器可以执行的数据结构。在编译版本中，网络不是被显式
地表示，而是被一组程序代替，通常一个节点一个或两个程序。比如，一个解释版本要提交一个通用的"left-activation-of-join-node"
程序到一个特殊的join节点（靠把程序的指针指向节点的数据结构），编译版本靠考虑特殊节点部分地评估解释器的通用程序来创建一个
特殊的程序（本章会描述通用解释器程序）。编译版本的优势当然是更快的速度。它的缺点一是需要更大的存储空间（一个节点的编译
程序通常比解释的数据结果占用更多的空间），二是运行时程序很难添加或者删除（修改编译过的代码比修改一个解释的数据结构难多了
--尽管至少一个编译版本解决了这个问题）。

在本章的剩余部分，我们将更深入地描述Rete，为它的基本数据结构和程序给出高级伪代码。这些伪代码会一节一节地给出，为了增加一
个在后续章节才会被讨论的特殊的支持，有时更靠后的章节会修订之前给出的。一个完整版本的伪代码，包括一些在后面章节介绍的重要
改进，会放在附录A里。

一个系统的Rete模块有四个入口点：`add-wme`，`remove-wme`，`add-production`和`remove-production`。我们先从add-wme的调用开始
讨论：2.2节alpha网络做什么，2.3节描述alpha和beta寄存器节点做什么，2.4节描述join节点做什么。我们将在2.5节讨论remove-wme，
2.6节讨论add-production和remove-production。接下来我们将讨论一些Rete更复杂的特性：2.7节展示了如何处理否定条件（测试WME的
不存在），2.8节展示否定结合（测试WME组合的不存在）如何被处理的。然后我们在2.9节给出几种实现说明，并在2.10节纵览一些过去
这些年其它一些Rete的优化。最后，在2.11节讨论一下Rete算法的普遍性，包含它对更少WME限制表示的适应性。

##2.2 Alpha网络实现
当一个WME被添加到工作区后，alpha网络会在上面执行必要的常量（或内在条件）测试并且把它存储到一个或更多合适的alpha寄存器。这
里有几种发现合适alpha寄存器的方法。

###2.2.1 数据流网络
原始并且可能大部分直接的方法是用一个简单的数据流网络。图2.3给出了一个只有10个条件（C1-C10）的小型规则系统的网络示例。这个
网络的构成如下所述。对每一个条件，让T1，...，Tk代表它的常量测试，可以按任何顺序排列（通常的方式是按条件源文本从左到右排列）。
从项节点开始，我们构建一个按照这个顺序对应T1,...,Tk这k个节点的路径。这些节点在文献上通常被称作`常量测试`或者`单输入节点`。
当我们构建这个路径时，每当可能的话我们共享（比如，重用）已经存在的、包含同样的测试的节点（对其它节点来说）。最后，我们把
这个条件的alpha寄存器当作对Tk的节点输出。

请注意这个构建仅关注条件中的常量，同时忽略变量。因此，在图2.3中，条件C2和C10即使包含不同的变量还是会共享一个alpha寄存器。
同时也请注意一个条件根本不包含常量测试也是可能的（例如，图中的C9），在这种情况下它的alpha寄存器仅是alpha网络中顶结节的子
节点。

图2.3 使用alpha网络的数据流网络示例

这些节点的每一个仅是一个这样的数据结构，在节点指定被执行的测试，任何节点输出进入可能进入的alpha寄存器，一组子节点（其它
常量测试节点）：
<pre><code>
structure constant-test-node:
   field-to-test:"identifier","attribute","value",or "no-test"
   thing-the-field-must-equal:symbol
   output-memory:alpha-memory or nil
   children:list of constant-test-node
end
</code></pre>
（“no-test”会被在顶节点）当一个WME被添加到工作区，我们仅仅把它填进数据流网络的项端：
<pre><code>
  procedure add-wme(w:WME) {dataflow version}
     constant-test-node-activaction (the-top-node-of-the-alpha-network,w) 
  end

  procedure constant-test-node-activation (node: constant-test-node; w: WME)
    if node.eld-to-test 6= 'no-test' then
       v <-- w.[node.eld-to-test]
       if v 6= node.thing-the-eld-must-equal then
          return ffailed the test, so don't propagate any furtherg
    if node.output-memory 6= nil then
       alpha-memory-activation (node.output-memory, w) fsee Section 2.3.1g
    for each c in node.children do constant-test-node-activation (c, w)
  end
</code></pre>

上面的描述假设所有的测试都是常量符号的相等测试。像前面所提及的，靠alpha网络执行的测试不限于相等测试。比如，一个条件也许
要求在WME中的某个属性有一个比7大的数值。为了支持这样的测试，我们将扩展constant-test-node数据结构到包括哪种被执行测试的
定义，并且相应地修改constant-test-node-activation程序。

###2.2.2 带哈希的数据流网络
Alpha网络的上述实现是简单且直接的。但是，在大型系统中它有一系列的缺点。读者也许已经从图2.3中猜到了，当一个节点的输出很大时
它带来了很多的浪费。在图中，attr=color?这个节点有5个子节点，并且它们的测试是互不干扰的。当一个WME经过attr=color?测试时，
5个以上的测试被执行，每个子节点一次，最多只有一个节点可能执行成功。当然，一个节点能够有特定颜色的名字；一个药物学习系统
也许要学习越来越多的疾病。随着系统的增长，在Alpha网络中这些点上像这样大量浪费的工作也会随之增长，因此匹配器会急速变慢。

这个问题显而易见的解决方案是用一个特殊的节点来代替网络中这个有巨大输出结果的节点，这个特殊节点使用一个hash表（或者平衡
二叉树）去确定激活需要哪条路径来继续向下。在上面的例子中，直接使用hash去查找匹配WME属性值的节节点来替代激活5个子节点。
事实上，子节点能够被消除，既然hash能够高效地执行同样的测试。图2.4给出了使用hash技术的结果。扩展前面的伪代码去处理这些
hash表很简单。

###2.2.3 详尽的hash表查找
既然WMEs被要求是三元组，有一个简单优雅的办法去实现大多数alpha网络，仅靠几个hash表查询。假设此时所有的常量测试都是等于测试
--比如，没有大于、不等于或者其它特殊的测试。那么我们能够得到下面的观察：给出任意一个WME，WME能够进入的最多只有8个alpha
寄存器。这是因为每个alpha寄存器都有这样的范式（test-1 ^test-2 test-3)，这里每三个测试要么是特定常量符号的等于测试，要么
不关心，我们用“*”来标记。如果一个WME w=(v1 ^v2 v3)进入到alpha寄存器a中，那么a必定是下面八种范式中一其中之一：
<pre><code>
(* ^* *)
(* ^* v3)
(* ^v2 *)
(* ^v2 v3)
(v1 ^* *)
(v1 ^* v3)
(v1 ^v2 *)
(v1 ^v2 v3)
</code></pre>

仅有8种方法能够生成(v1 ^v2 v3)匹配的条件。因此，给定一个WME w，去确定w应该添加到哪个alpha存储，我们仅需检查这8种可能哪个
会真正在系统中。（既然不是任何alpha寄存器都会对应有测试和*的组合，一些情况也许根本不会出现）我们在hash表中存储指向所有系统
的alpha寄存器，根据被测试的特殊的值做索引。执行alpha网络就变成了8种hash表查询的简单匹配：
<pre><code>
procedure add-wme (w: WME) fexhaustive hash table versiong
  let v1, v2, and v3 be the symbols in the three elds of w
  alpha-mem   lookup-in-hash-table (v1,v2,v3)
  if alpha-mem 6= \not-found" then alpha-memory-activation (alpha-mem, w)
  alpha-mem   lookup-in-hash-table (v1,v2,)
  if alpha-mem 6= \not-found" then alpha-memory-activation (alpha-mem, w)
  alpha-mem   lookup-in-hash-table (v1,,v3)
  if alpha-mem 6= \not-found" then alpha-memory-activation (alpha-mem, w)
  ...
  alpha-mem   lookup-in-hash-table (,,)
  if alpha-mem 6= \not-found" then alpha-memory-activation (alpha-mem, w)
end
</code></pre>

上面算法的优雅取决于两个假设：WMEs三元组的范式并且没有不等于的测试。第一个假设能够稍微被放松：为了处理r元组的WMEs，我们能够
使用2^r次hash表查询。当然，只有在r非常小时才能工作的很好。有两个方法去消除第二个假设：
 * 可以用上面2.2.1章节中描述的小数据流网络形式来表示hash表查询的结果，而不是用alpha寄存器。所有常量等于测试用hash处理，而
其它测试用像之前的常量测试节点来处理。重要的是，使用2.2.1章节数据流网络的查询结果，移动所有等于测试到网络的上半部分其它
测试到下半部分，然后用8种hash查询替换掉上半部分
 * 用网络的beta部分代替alpha部分来处理不等于测试。当然，在beta部分而不是alpha部分执行一些常量（或内置条件）测试对于Rete
的实现也是可能的。这导致alpha网络几乎不再重要，并且丝毫不会增加多少beta网络的复杂性。然后，当不同规则有同一个条件时它减少
了这些测试被共享的可能性。(这个正是该理念中实现的途径）。

不管是2.2.2章节中的数据流加hash的实现还是2.2.3章节中的穷举hash表查询实现，alpha网络都是非常高效的，工作区的每次变更都几乎
是常量时间运行。网络的beta部分的消耗在Sora系统中占了大部分，并且之前的研究已经在OPS5中被证实。本章大部分（并且该理论的大
分）因此也会处理网络的beta部分。

##2.3 存储节点实现
现在我们讨论一下alpha和beta存储节点的实现。回忆一下alpha寄存器存储WMEs集合，beta寄存器存储token集合，每一个token代表一个
WMEs的序列--特别的是，满足一些规则前k个条件（具有一致变量绑定）的k个WMEs的序列。

有各种实现寄存器节点的方法。我们可以根据两个标准对实现做分类：
* 集合（WMEs或tokens）是如何构成的？
* 一个token--WMEs的序列--如果表示的？

对每一个问题，最简单的实现没有在集合加强加一个特殊的结构--它们仅用列表来表示，并且元素是无序的。然而，我们经常能够在join
操作上靠强加一个索引结构在寄存器上来增加效率。究竟如何，考虑一下我们之前的例子：
<pre><code>
(find-stack-of-two-blocks-to-the-left-of-a-red-block
 (x ^on y) /* C1 */
 (y ^left-of z) /* C2 */
 (z ^color red) /* C3 */
-->
 ... RHS ...
)
</code></pre>

对这条规则来说，我们想让token表示根据<y>上的绑定做索引的前两个条件的匹配，而不是对<z>的绑定。如果寄存器节点根本没有被索引，
但是替代为简单的无序列表，然后规则就能够共享一个寄存器节点去存储前两个条件的匹配。使用索引，我们需要两个寄存器节点代替一
个（或者一个拥有两个不同索引的节点）。因为这两个小的代价，在一些系统中索引也许不是很值得做，尤其是寄存器永远也不会包含非常
多的元素时。

不管这些潜在的缺点，实践中的经验结果表明，做过hash的寄存器比没有索引的寄存器提升很多。（Gupta et al.,1988）发现在几个OPS5
系统中靠1.2-3.5中的一个因素加速了匹配算法，并且 （Scales，1986）提道了1.2-1.3的一个因素对Soar系统。

注意，我们并不能总是发现一个变量，应该在它的绑定上索引一个寄存器。例如，有时一个条件所有之前的条件是完全无关的：
<pre><code>
  ( x ^on y)  /* C1 */
  ( a ^left-of b ) /* C2 */
</code></pre>

在这个例子中，在C2的join节点之前索引beta寄存器没有意义，既然C2没有测试任何在C1中使用过的变量。像这些的案例，我们只是简单
地使用一个未索引的寄存器节点。

现在转到第二个问题--一个token（WMEs的序列）是如果表示的？--这里有两种主要的可能。一个序列或者靠一个数组（数组形式的token）
或者靠一个列表（列表形式的token）。既然数组能够提供直接访问序列中所有元素的优势，使用它看起来是显而易见的选择--给出i，我们
能在常量时间内发现第i个元素--而一个列表需要循环前i-1个元素为了得到第i个。

然后，数组形式的tokens能导致很多冗余信息存储因此有多得多的空间使用。为了弄明白为什么，注意到对每一个beta寄存器节点存储
一个规则中的前i>1个条件，这里有下一个beta寄存器节点--换句话说，它的祖父（跳过介于中间的join节点）--存储前i-1个条件的匹配
结果。如果(w1,...,wi)表示一个对前i个条件的匹配，那么必定有一个对前i-1个条件的匹配。这意味着在较低的beta寄存器里的任意
token能够被简洁地表示为一对(parent,wi)，在这里parent是在上面的表示前i-1个WMEs的寄存器里的token。如果我们在所有的beta寄存
器里使用这个技术，那么每个token实际上变成了一个link-list，靠prent指针连接，表示一个反序的WME序列，wi在列表的头部，w1在尾
部。同样地，我们使最上面的beta寄存器（仅表示一个WME的序列）里的token的父节点指向虚假的顶点token，它表示一个空的序列<>。
注意到目前所有token的集合形成了一棵树，子节点指向父节点，并且根节点是虚假的顶点token。

使用数据形式的tokens，一个对前i个条件的token花费O(i)的空间，而使用列表形式的tokens，每个token仅花费O(1)的空间。这能够节
省大量的空间，特别是如果有规则有大量条件的时候。如果一有C个条件的规则有一个完全的匹配，数组形式的tokens会至少使用
1+2+...+C=O(C^2)的空间，而列表形式的tokens只需要O(C)的空间。当然，使用更多的空间意味着使用更多的时间去填满空间。每一次一
个beta寄存器节点被激活，它创建并且存储一个新的token。使用数据形式的tokens，在添加第i个元素前需要从上面循环拷贝i-1个元素。
使用列表形式，创建一个token不需要这种循环。

总结一下，使用数组形式的tokens比使用列表形式的tokens需要更多的空间，并且在每个beta寄存器激活上需要更多的时间创建每个token。
然而，对一个指定的序列元素它提供了比列表形式更快的访问速度。为执行变更绑定一致性检查，在join节点激动时经常需要访问任意的
元素。因此我们有一个折衷。两种表示都不是很明显地比所有的系统好。在一些系统中，使用数组形式的tokens会因空间的原因而不可行
--比如，一个规则有巨量的条件。通常来说，使用列表形式的tokens的选择依赖于访问任意元素的成本有多高。一个系统使用越多的变量
一致性检查，并且包含的变更相距越远（比如，两个变量出现在一条规则非常靠前和靠后的位置，相对出现在条件中的Ci和Ci+1位置），
访问的成本越大，那么数组形式的token会越快一点。

现在我们转到伪代码上。为了尽量保持简单，我们在伪代码中使用列表形式的tokens和未索引的寄存器节点。随着进行的深入，我们会指
出导致重要区别的地方（本理论的实际实现使用了列表形式的tokens和hash寄存器节点）。

###2.3.1 alpha寄存器实现
一个WME只包含三个属性：
<pre><code>
   structureWME:
     felds:array[1..3] of symbol
   end
</code></pre>

一个alpha寄存器一个WME的列表，外加一个继续者列表（join节点会依附于它):
<pre><code>
 structurealpha-memory:
   items: list of WME
   successors: list of rete-no de
 end
</code></pre>

无论何时一个新的WME通过alpha网络被过滤并且到达一个alpha寄存器时，我们仅简单地把它添加到寄存器的其它WME的
列表中，并通知每个附属的join节点：
<pre><code>
 procedure alpha-memory-activation (no de: alpha-memory, w: WME)
   insert w at the head of no de.items
   for eachchildin no de.successorsdoright-activation (child, w)
end
</code></pre>

###2.3.2 Beta寄存器实现
如上所述，使用列表形式的tokens，一个token仅是一对：
<pre><code>
 structure token:
   parent: token {points to the higher token, for items 1...i-1}
   wme: WME {gives item i}
end
</code></pre>

一个beta寄存器存储一个它包含的tokens列表，加上它的子节点（在网络的beta部分的其它节点）的列表。在我们
给出它的数据结构之前，我们将要再次通过一个swich或case表达式或者一个根据被激活节点的类型索引的跳转表
对左右激活的程序进行调用。因此，给出一个节点（或者指向它的指针），我们需要能够确定它的类型。如果使用
多样记录（variant records）来表示节点将非常直接。（一个多样记录是指它能够包含任何不同属性集）。在网络
的beta部分的每一个节点将会用一个rete-node结构来表示：
<pre><code>
 structure rete-node:
   type: “beta-memory", "join-node", or "p-node" {or other node types we'll see later}
   children:list of rete-node
   parent: rete-node {we'll need this "back-link" later}
   ...(variant part -- other data dep ending on no de type) . . .
 end
</code></pre>

既然从现在开始我们描述节点的每一个特殊类型，我们给它的数据结构将仅列出节点类型的扩展信息；
记住在网络beta部分的所有节点都有type,children和parent属性。并且，我们仅简单地用left-activation
或者right-activation来表示合适的switch或case表达式或跳转表的使用。

现在转回beta寄存器节点，一个beta寄存器存储的唯一扩展信息是它包含的tokens列表：
<pre><code>
 structure beta-memory:
   items: list of token
 end
</code></pre>

无论何时一个beta寄存器被一个新的匹配（由一个存在的token和一些WME组成）通知，我们创建一个token，
把它添加到beta寄存器的列表中，并通知betta寄存器的每一个子节点：
<pre><code>
 procedure beta-memory-left-activation (node: beta-memory, t: token, w: WME)
   new-token = allocate-memory()
   new-token.parent = t
   new-token.wme = w
   insert new-token at the head of no de.items
   for each child in node.children do left-activation (child, new-token)
 end
</code></pre>

###2.3.3 P节点实现
我们在这里提到规则节点(p-node)的实现是因为它在一些方面经常和寄存器节点的实现很相似。p节点的实现
在不同的系统是有变化的，因此在这里我们的讨论尽量会通用一点并且不会给出伪代码。一个p节点可以存储
tokens，就像beta寄存器一样；这些tokens代表了对规则条件的完全匹配。（在传统规则系统中，在所有p节点
中的所有tokens集合被表示为冲突集(conflict set)）。在一个左激活中，一个p节点会构建一个新的toekn，
或者新的完全匹配的相似表示。然后它用一些适合的方式发出新匹配的信号）。

总的来说，一个p节点包含规则对应的定义--规则名称，右手边行为，等。一个p节点也可以包含出现在规则中的
变量名称的信息。注意，变量名称并没有在本章我们描述的任何Rete节点数据结构中提起过。这是刻意为之--
当两条规则有同一基本形式但有不同的变量名称的条件时它使节点能够被共享。当变量名称被记在某处时，它
可能需要靠查找带有变量名称的Rete网络来重新构建规则的LHS。重新构建LSH消除了保存LSH”原始拷贝“的需要，
这种情况下我们需要随后检查规则。

##2.4 Join节点实现
像在概述中提出的那样，当一个WME被添加到它的alpha寄存器一个join节点能够递归一个右激活或者当一个token
被添加它的beta寄存器能够递归一个左激活。在任何一种情况下，会搜索节点的其它寄存器查找和新元素一致的
变量绑定。如果有任何发现，它们将被传递到join节点的子节点。

一个join节点的数据结构因此必须包含指向它的两个寄存器节点（因此它们能被搜索）的指针，一个任何变量绑定
一致性测试被执行的定义，和子节点的列表。从数据到所有的节点（在上面的rete节点结构），我们已经有了子节
点；并且，parent属性给我们一个指向join节点beta寄存器（beta寄存器总是它的父节点）的指针。我们在join
节点中需要两个额外的属性：
<pre><code>
 structure join-node:
   amem: alpha-memory {points to the alpha memory this node is attached to}
   tests: list of test-at-join-node
 end
</code></pre>

test-at-join-node结构指定两个属性的位置，它们的值必须依次等于几个一贯绑定的值：
<pre><code>
 structure test-at-join-node:
   feld-of-arg1: "identifier", "attribute", or "value"
   condition-number-of-arg2:integer
   feld-of-arg2:  "identifier", "attribute", or "value"
 end
</code></pre>

Arg1是WME（alpha寄存器）的三个属性之一，而arg2是在规则中匹配一些先前条件的WME的一个属性（比如beta
寄存器中token的一部分）。举例来说，在我们的示例规则中：
<pre><code>
 (find-stack-of-two-blocks-to-the-left-of-a-red-block
   (x ^on y)      /* C1 */
   (y ^left-of z) /* C2 */
   (z ^color red) /* C3 */
  -->
   ... RHS ...
  ),
</code></pre>

对C3的join节点，检查z的一致性绑定，会有filed-of-arg1="identifier",conditon-number-of-arg2=2和field-of-arg2="value"，
既然来自join节点的alpha寄存器的WME的id属性的内容必须等于匹配第二个条件的WME的value属性。

在一个右激活（当一个新WME w被添加到alpha寄存器）之上，我们通过beta寄存器并发现任何t-vs-w测试成功的
token t。任何成功的<t,w>组合被传递到join节点的子节点。相似地，在一个左激活（当一个新token t被添加到
beta寄存器）之上，，我们通过alpha寄存器并发现文韬任何t-vs-w测试成功的WME。并且，任何成功的<t,w>组合
被传递给节点的子节点：
<pre><code>
 procedure join-no de-right-activation (node: join-node, w: WME)
   for each t in node.parent.items do {"parent" is the beta memory node}
     if perform-join-tests (node.tests, t, w)then
       for each child in node.children do left-activation (child, t, w)
 end
 
 procedure join-node-left-activation (node: join-node, t: token)
   for each w in node.amem.items do
     if perform-join-tests (node.tests, t, w)then
       for each child in node.children do left-activation (child, t, w)
 end
 
 function perform-join-tests (tests:list of test-at-join-node, t: token, w: WME)
 returning b o olean
   for each this-test in tests do
     arg1 = w.[this-test.field-of-arg1]
     {With list-form tokens, the following statement is really a loop}
     wme2 = the [this-test.condition-number-of-arg2]'th element in t
     arg2 = wme2.[this-test.field-of-arg2]
     if arg1 != arg2 then return false
   return true
 end
</code></pre>

我们注意到上面程序的几点。第一，为了能够在网络中为最顶端的join节点使用这些程序--如2.2章节所述的，他们
都是虚假顶节点的子节点--我们需要虚假顶节点来扮演节点join节点的beta寄存器。我们总是保持一个单一的虚假
顶端token在虚假项节点中，因此他们在join-node-right-activation程序中迭代完都是同一件事情。第二，伪代码
假设所有的测试都是测试两个属性的相等。扩展test-at-join-node结构和perform-join-tests程序去支持其它的
测试（比如，测试一个属性是否小于另外一个）就很简单了。大部分但不是全部Rete实现支持这些测试。最后，伪
代码假设alpha和beta寄存器没有任何索引，像上面2.3节讨论的那样。如果索引寄存器被使用，那么上面的激活
程序需要被修改，使用索引而不是简单迭代完所有的tokens或者寄存器节点中的WMEs。例如，如果寄存器做了Hash，
程序仅会迭代完在合适hash桶中的tokens或WMEs，而不是寄存器中的所有tokens或WMEs。这能够显著地加速Rete
算法。

###2.4.1 避免重复Tokens
无论何时我们添加一个WME到一个alpha寄存器，我们右激活每一个附属到alpha寄存器的join节点。我们的讨论迄今
为止，一个非常重要的细节被遗漏了。结果是join节点被右激活的序列是至关重要的，因为如果join节点以错误的顺序
激活可能生成重复的token（在beta寄存器里有两个或以上token代表同一个WME序列）。

为了弄明白这是如何发生的，考虑一条它的LHS以下面三个条件开始的规则：
<pre><code>
  (x ^self y)    /* C1 */
  (x ^color red) /* C2 */
  (y ^color red) /* C3 */
   ...           /* other conditions */
</code></pre>

假设工作区最初只包含一个WME，匹配第一个条件：
<pre><code>
  w1: (B1 ^self B1)
</code></pre>

图2.5(a)展示这种情况下的Rete网络，包括所有寄存器节点的内容。注意到下面的alpha寄存器在这条规则中为两
个不同的条件使用，C2和C3。现在假设下一个WME被添加到工作区：
<pre><code>
  w2: (B1 ^color red)
</code></pre>

这个WME通过alpha网络做过滤并且alpha-memory-activation程序被调用添加w2到下面的alpha寄存器。我们添加
w2到寄存器的元素列表，然后我们不得不右激活两个join节点（一个连接在x的值上，一个连接在y的值上）。假设
我们先右激活上面的一个（x的那个）。它搜索beta寄存器寻找合适的token，发现一个并把这个新匹配(w1,w2)传递
给它的了节点（保存有C1^C2匹配的beta寄存器）。它被添加到寄存器并传递给下面的join节点。现在该join节点
搜索alpha寄存器寻找适合的WME--并发现w2，它被添加到寄存器--并传递新的匹配(w1,w2,w3)到它的子结点。这靠
上面的join节点的右激活的触发结束执行。这个情况展示在图2.5(b)中。我们仍然不得不右激活下面的join节点。
当我们右激活它时，它搜索beta寄存器寻找适合的token--并且发现<w1,w2>，添加到寄存器中。因此它之后传递
这个新的匹配<w1,w2,w3>到它的子节点，没有识别这个是一个之前的重复匹配。最终的结果如图2.5(c)所示；注意
最下面的beta寄存器包含了同一token的两份拷贝。

处理这个问题的一个办法将是对beta寄存器节点做重复token的检查。每次一个beta寄存被激活，它会检查这个“新“
的匹配实际是否是寄存器中已存在的token。如果是，它将被忽略（比如，丢弃掉）。不幸地是，这将显著地拖慢
beta寄存激活的处理。

一个更好的避免拖慢的方案是以不同的序列右激活join节点。在上面的例子中，如果我们先激活下面的join节点，
不会有重复的token被生成（邀请读者检查这一点；核心的是当下面的join节点被右激活时，它的beta寄存器仍然
是空的）。总的来说，右激活的解决方案依赖于之前它们的先辈节点；比如，如果我们需要从同一个alpha寄存器
右激活join节点J1和J2，并且J1是J2派生的，那么我们在右激活J2之前右激活J1.我们添加一个WME到一个alpha
寄存器的全部程序是：
1.添加WME到alpha寄存的元素列表
2.右激活附属的join节点，派生的在先辈之前
另外一个方案是--在派生节点之前右激活先辈节点，然后添加WME到寄存器的元素列表--也能工作。这个问题的
完整讨论，请看(Lee and Schor,1992)。

当然，在每一个alpha寄存器激活上遍历Rete网络查找先辈派生节点在join节点中的关系太笨了。代之为，我们
提前检查这些，当网络构建时，并确保alpha寄存器的join节点列表是有序的：如果J1和J2都在列表上并且J1是
J2的派生，那么J1必须在J2的前面。靠预先强制排序，我们避免后面在节点激活时的任何额外开销，并且避免
生成任何重复的token。

我们将在第4章描述右断开连接（right unlinking）时回到这个问题，一个优化是join节点将被动态地加进和
移出alpha寄存器的列表。我们需要的是保证合适的顺序并和我们拼接一样。

##2.5 WME的移除
当一个WME从工作区删除，我们需要靠删任何包含WME的入口来相应地更新alpha和beta寄存器节点（和p节点中的token）中的条目。
有几种方法来实现这一点。

在最初的Rete算法中，删除本质上和添加的处理相同的。我们把这种方法叫作`基于再匹配的删除(rematch-based removal)`。基本
的思路是在解释器中的每个程序增加一个额外的参数，叫做`标签(tag)`，用来指定当前操作添加还是删除的标识。对于删除来说，
对alpha和beta寄存器节点的程序仅简单地从它们的寄存器中删除指定的WME或token来代替添加。然后程序调用他们的继承者，只是把
把它们作为一个添加，用`delete`做标签而不是`add`.Join节点的程序同样地处理添加和删除--在这两个案例中，他们用一致性变量
绑定在合适的寄存器中查找条目并且传递任何匹配（连同add/delete标签）到他们的子节点。因为这个基于再匹配的删除方法处理
删除和同一解释器程序处理添加大部分相同，因此它简单而优雅。

不幸地是，它也很慢，至少相对其它可能的方法。对基于再匹配的删除来说，既然同一个程序被调用并且每一个做了差不多的工作，
删除一个WME的代价和添加一个WME的代价其实是一样的。问题是在添加一个新的WME时没有信息可以随后在被删除这个WME时使用。
有至少三个方法可以在处理删除时使用这些信息。

在`基于扫描的删除(scan-based removal`中，我们不用在join节点重复做变量绑定一致性检查，代替的是仅简单地扫描它们的输出
寄存器（它们的子beta寄存器，p节点）来查找任何要被删除条目的入口。当一个join节点被右激活作一个WME w的删除时，它仅把w
传递到它的输出寄存器。寄存器查找token列表中最后一个元素是w的token，删除这些token，并发送这些删除到它的子节点。相似的，
当一个join节点被左激活作token t的删除时，它把t传递到它的输出寄存器。寄存器查找token列表中父亲是t的token，删除这些token，
并发送这些删除到它的子节点。注意查找token的父亲是t的这部分程序仅在使用列表形式时才高效而数组形式不行。在Soar系统中使用
基于扫描的删除比基于再匹配的删除效率提高28%；（Barachini，1991）则产使用轻微变量的这个技术比基于再匹配的删除效率提高10%。

也许处理删除的最快方法是提前精确地记下哪一个要被删除。这个直接的思路是`基于列表的删除`和`基于树的删除`的基础。这个思路
是在WME和token的数据结构增加额外的指针，因此当一个WME被删除时我们发现所有需要被删除的token--并且仅有这些需要被删除--仅
靠指针。

在由（Scales，1986）建议的基于列表的删除中，我们在每一个WME w中存储包含w的所有token列表。然后当w被删除，我们仅迭代这个
列表并删除在上面的每一个token。这个方法的缺点是需要大量额外的存储空间并且潜在的大量额外时间来创建一个token：一个新的
token（w,...,wi)必须被添加到每个w的列表中。然而，既然创建这个新otken无论如何都需要创建一个新的i个元素的数组，如果token
是用数组而不是列表来表示的话，那么空间和时间最多都会是一个常量因子。因此，基于列表的删除是那样的实现也许可以被接受。
论文中没有基于列表的删除的经验报道，因此它是否能在实际中工作的很好（或者甚至它曾被实现过）还不清楚。

在基于树的删除中，在每个WME w上，我们为最后一个w保存所有token的列表。在每个token t上，我们保存t的所有孩子的列表。这些
指向孩子token的指针允许我们在下面的beta寄存器和条件节点发现所有t的派生。（回头看下2.3章节中是用列表形式的token，这里的
token集合是一棵树）。现在当w被删除时，我们仅遍历子树（根token和它们的派生token的），并删除在它们里的一切。当然，所有这
些额外的指针意味着更多的存储空间，外加在WME被添加或token被创建之前的指针设置。然而，从经验上讲，在WME删除期间的时间节省
远胜提前设置指针的。当作者在Soar系统使用基于树的删除替换掉基于再匹配的删除后，匹配器提升了1.3倍的速度；（Barachini,1991)
在一个类OPS5的系统中估计有1.25倍的提升。

为了实现基于树的删除，我们每个WME的数据结构让它包括包含WME的所有alpha寄存器的列表，和把WME做为最后一个元素的所有token
的列表：
<pre><code>
  structure WME {revisedfrom version on page 21}
    felds:array[1..3] of symbol
    alpha-mems:list of alpha-memory {the ones containing this WME}
    tokens:list of token {the ones with wme=this WME}
  end
</code></pre>

token的数据结构被扩展到包含指向寄存器节点（我们会在下面的delete-token-and-descendents中使用）的指针和它的儿子们的列表：
<pre><code>
  structure token {revised from version on page 22}
    parent: token {points to the higher token, for items 1...i-1}
    wme: WME {gives item i}
    node: rete-node {points to the memory this token is in}
    children:list of token {the ones with parent=this token}
  end
</code></pre>

我们现在修改aplha-memory-activation和beta-memeory-left-activation程序提前设置这些列表。无论何时一个WME w被添加到alpha
寄存器a中，我们添加a到w.alpha-mems.
<pre><code>
  procedure alpha-memory-activation (node: alpha-memory, w: WME)
  {revisedfrom version on page 21}
     insert w at the head of node.items
     insert node at the head of w.alpha-mems {for tree-basedremoval}
     for each child in node.successors do right-activation (child, w)
  end
</code></pre>

相似地，无论何时一个新的token t=<t,w>被添加到beta寄存器，我们添加tok到t.children和w.tokens。我们也在token上填充新节点
属性。为了简化我们的伪代码，定义一个helper函数make-token会方便一点，它构建一个新的token并初始化它的变量属性作为基于树
的删除的必备之选。虽然我们把这个作为一个独立的函数，但正常情况下它在代码使用inline会更高效。
<pre><code>
  function make-token (node: rete-node, parent: token, w: wme)
  returning token
    tok =allocate-memory()
    tok.parent = parent
    tok.wme = w
    tok.node = node {for tree-based removal}
    tok.children =nil {for tree-based removal}
    insert tok at the head of parent.children {for tree-based removal}
    insert tok at the head of w.tokens {for tree-based removal}
    return tok
  end
  
  procedure beta-memory-left-activation (node: beta-memory, t: token, w: WME)
  {revised from version on page 23}
    new-token = make-token (node, t, w)
    insert new-token at the head of node.items
    for each childin node.children do left-activation (child, new-token)
  end
</code></pre>

现在，删除一个WME，我们仅从包含它的每个alpha寄存器中删除它（这些alpha寄存器现在很方便在一个列表上）并调用helper子程序
delete-token-and-descendents去删除包含它的所有token（包含它的所有必要的根token也很方便在一个列表中）：
<pre><code>
  procedure remove-wme (w: WME)
    for each am in w.alpha-mems do remove w from the list am.items
    while w.tokens !=nil do
       delete-token-and-descendents (the first item on w.tokens)
  end
</code></pre>

注意w.tokens上使用的是while循环而不是for循环。for循环在这里并不安全，因为每次调用delete-token-and-descendents会破坏性
地修改w.tokens列表因为它要释放token数据结构使用的内存空间。for循环会持有一个指进它运行的列表的中间位置，并且这个指针
可能对这个破坏性的修改变得无效。

这个帮助类子程序delete-token-and-descendents删除一个token和它的整个派生树。为了简化，这里的伪代码是递归的；实际的实现
可以使用非递归树遍历方法会更快一点。
<pre><code>
  procedure  delete-token-and-descendents (tok: token)
    while tok.children !=nil do
      delete-token-and-descendents (the first item on tok.children)
    remove tok from the list tok.node.items
    remove tok from the list tok.wme.tokens
    remove tok from the list tok.parent.children
    deallocate memory for tok
  end
</code></pre>

###2.5.1 列表实现