```haskell
原文出自： http://ncatlab.org/nlab/files/WadlerMonads.pdf
作者是： Philip Wadler
本人是函数式编程的初学者，卡在单子这个环节上无法前进。想要理解单子，因此准备了几篇有关单子的文章进行翻译，本文是其中之一。本人英文水准较差，翻译是为了强迫自己理解，不保证准确。
```

Comprehending Monads
====================

**Philip Wadler**

**University of Glasgow**
>     格拉斯哥大学


#####Abstract
>Category theorists invented monads in the 1960's to concisely express certain aspects of universal algebra. Functional programmers invented list comprehensions in the 1970's to concisely express certain programs involving lists. This paper shows how list comprehensions may be generalised to an arbitrary monad, and how the resulting programming feature can concisely express in a pure functional language some programs that manipulate state, handle exceptions, parse text, or invoke continuations. A new solution to the old problem of destructive array update is also presented. No knowledge of category theory is assumed.

>20世纪60年代，Monads出现在范畴理论中，它简洁的表达了泛代数(universal algebra)中的某个概念。20世纪70年代，函数式编程者发展出列表推导式(list comprehesions)，它简洁的表达某些构造列表的程序。本文将展示如何可推广列表推导式为任意monad，以及怎样组织纯函数式语言去简洁的表达状态处理，处理异常，文本解析，或调用延续等操作。同时，对破坏性数组更新这个老问题提出新的解决放案。阅读本文需要范畴理论。

##1. Introduction
Is there a way to combine the indulgences of impurity with the blessings of purity?

>如何在坚持纯粹中容忍不纯粹的存在？

Impure, strict functional languages such as Standard ML [Mil84, HMT88] and Scheme [RC86] support a wide variety of features, such as assigning to state, handling exceptions, and invoking continuations. Pure, lazy functional languages such as Haskell [HPW91] or Miranda1 [Tur85] eschew such features, because they are incompatible with the advantages of lazy evaluation and equational reasoning, advantages that have been described at length elsewhere Hug89, BW88].

>不纯的精确性函数式编程语言，如 Standard ML [MIL84，HMT88] 和 Scheme [RC86] ，支持若干诸如状态赋值，异常处理，调用延续等特性。纯的惰性函数式编程语言，如  Haskell [HPW91] 和 Miranda1 [TUR85] 则避开这些特性，因为它们不利于发挥已经被证明具有优势的延迟计算和公式推导等特性。

Purity has its regrets, and all programmers in pure functional languages will recall some moment when an impure feature has tempted them. For instance, if a counter is required to generate unique names, then an assignable variable seems just the ticket. In such cases it is always possible to mimic the required impure feature by straightforward though tedious means. For instance, a counter can be simulated by modifying the relevant functions to accept an additional parameter (the counter's current value) and return an additional result (the counter's updated value).

>纯也有它的缺陷，所有的纯函数式语言程序员都会记得，当他们需要一个不纯的特性的时候。例如，如果需要一个具有唯一命名的计数器，那么似乎需要一个可被赋值的变量。在这种情况下，总是会去通过简单的但繁琐的手段去模仿所需的不纯特性。例如，通过修改相关函数来接受一个附加的参数（计数器的当前值），并返回最终结果（计数器的更新值），以这样的方式来模拟一个计数器。

This paper describes a new method for structuring pure programs that mimic impure features. This method does not completely eliminate the tension between purity and impurity, but it does relax it a little bit. It increases the readability of the resulting programs, and it eliminates the possibility of certain silly errors that might otherwise arise (such as accidentally passing the wrong value for the counter parameter).

>本文介绍了一种新的方法来构建纯的程序，以此来模拟不纯的功能。这种方法不能完全消除纯与不纯之间的矛盾，但确实缓解了一些。它增加了程序的可读性，并消除了可能出现一些愚蠢的错误(例如，不小心传递错误的计数器参数）。

The inspiration for this technique comes from the work of Eugenio Moggi [Mog89a,Mog89b]. His goal was to provide a way of structuring the semantic description of features such as state, exceptions, and continuations. His discovery was that the notion of a monad from category theory suits this purpose. By defining an interpretation of lambda-calculus in an arbitrary monad he provided a framework that could describe all these features and more.

>该技术的灵感来自欧金尼奥莫吉(Eugenio Moggi)的工作[MOG89a，MOG89b]。他的目标是提供一种方法来建立如状态，异常，延续，等特征的语义描述。他发现范畴理论的Monad可以实现这个目标。通过在任意一个monad中定义一个可被解释的lambda演算，他提供一个不只是描述上面这些特征的框架。

It is relatively straightforward to adopt Moggi's technique of structuring denotational specications into a technique for structuring functional programs. This paper presents a simplied version of Moggi's ideas, framed in a way better suited to functional programmers than semanticists in particular, no knowledge of category theory is assumed.

>在构建函数式程序的技术中相对直接的采用Moggi的构建形式规范的技术。本文将Moggi的成果简化，以更适合函数式编程程序员而不是语义学家方式呈现出来，从而使读者不需要范畴学知识。

The paper contains two significant new contributions.

>本文有两个重要的新贡献。

The first contribution is a new language feature, the monad comprehension. This generalises the familiar notion of list comprehension [Wad87], due originally to Burstall and Darlington, and found in KRC [Tur82], Miranda, Haskell and other languages. Monad comprehensions are not essential to the structuring technique described here, but they do provide a pleasant syntax for expressing programs structured in this way.

>第一个贡献是一种新的语言特征，单子推导式（monad comprehension）。是我们熟悉的列表推导式[Wad87]概念的推广，这个概念由是Burstall和Darlington创造，并存在于KRC[Tur82]，Miranda，Haskell等语言中。单子推导式在这儿并不是一种基本的程序构建技术，但它确切的为表达程序构成提供了一种有趣的句法。

The second contribution is a new solution to the old problem of destructive array update. The solution consists of two abstract data types with ten operations between them. Under this approach, the usual typing discipline (e.g., Hindley-Milner extended with abstract data types) is sufficient to guarantee that array update may safely be implemented by overwriting. To my knowledge, this solution has never been proposed before, and its discovery comes as a surprise considering the plethora of more elaborate solutions that have been proposed: these include syntactic restrictions [Sch85], run-time checks [Hol83], abstract interpretation [Hud86a, Hud86b, Blo89], and exotic type systems [GH90, Wad90, Wad91]. That monads led to the discovery of this solution must count as a point in their favour.

>第二个贡献是对破坏性数组更新这个老问题提出一个新的解决方案。这个方案由两个抽象数据类型以及他们之间的十个操作构成。在这种方法下，通用类型检查（例如，用抽象数据类型扩展的Hindley-Milner）可以绝对保证以重写的形式安全的更新数组。据我所知，这样的方案之前从没有被提出过，这个方案来自于众多绝妙方案的意外提出：包括句法限制[Sch85]，运行时检查[Hol83]，抽象解释[Hud86a, Hud86b, Blo89]，以及exotic类型系统[GH90, Wad90, Wad91]。而单子的出现对这个问题的解决起到了促进作用。

Why has this solution not been discovered before? One likely reason is that the data types involve higher-order functions in an essential way. The usual axiomatisation of arrays involves only rst-order functions (index , update , and newarray , as described in Section 4.3), and so, apparently, it did not occur to anyone to search for an abstract data type based on higher-order functions. Incidentally, the higher-order nature of the solution means that it cannot be applied in rst-order languages such as Prolog or OBJ. It also casts doubt on Goguen's thesis that rst-order languages are sufficient for most purposes [Gog88].

>为什么以前没有发现这个解决方案？一个可能的原因是，数据类型以一种基本的方式涉及高阶函数。通用公理化数组只涉及初阶函数（如4.3章节描述的 index, update, 和 newarray等操作)，因此显然没有任何人基于高阶函数对抽象的数据类型进行过查询。顺带的，这意味着这个解决方案本质上的高阶特性不能应用在类似于ProLog或者OBJ这样的初阶编程语言上。这也让人怀疑Goguen的论断[Gog88]：初阶编程语言对大多数的目标都是充分的。

Monads and monad comprehensions help to clarify and unify some previous proposals for incorporating various features into functional languages: exceptions [Wad85, Spi90], parsers [Wad85, Fai87, FL89], and non-determinism [HO89]. In particular, Spivey's work [Spi90] is notable for pointing out, independently of Moggi, that monads provide a framework for exception handling.

>单子以及单子推导式有助于明晰和统一某些已经存在并参杂在函数式编程语言中的做法：异常[Wad85, Spi90]，解析器[Wad85, Fai87, FL89]，以及非决定性[HO89]。特别是独立于Moggi，由Spivey的工作[Spi90]非常明显的指出，单子提供了一个异常处理框架。

There is a translation scheme from lambda-calculus into an arbitrary monad. Indeed, there are two schemes, one yielding call-by-value semantics and one yielding call-by-name. These can be used to systematically transform languages with state, exceptions, continuations, or other features into a pure functional language. Two applications are given. One is to derive call-by-value and call-by-name interpretations for a simple non-deterministic language: this fits the work of Hughes and O'Donnell [HO89] into the more general framework given here. The other is to apply the call-by-value scheme in the monad of continuations: the result is the familiar continuation-passing style transformation. It remains an open question whether there is a translation scheme that corresponds to call-by-need as opposed to call-by-name.

>有两个从lambda表达式转换为monad的方案。一个遵从call-by-value语义，一个遵从call-by-name语义。这些被用来系统的将一般语言的状态，异常，持续调用等特性转换为纯函数式语言。同样可给出两个应用，一个是基于call-by-value和call-by-name来解释一个简单的非决定性语言：将Hughes 和 O‘Donnell [HO89] 变为适用性更强的框架。另一个是在连续型monad上应用call-by-value方案：结果是类似与cps的转换。最后有一个悬而未决的问题：相对于call-by-name，是否存在一种转换符合call-by-need语义。

A key feature of the monad approach is the use of types to indicate what parts of a program may have what sorts of effects. In this, it is similar in spirit to Gifford and Lucassen's effect systems [GL88]

>单子用法的一个关键点是采用类型去表示怎样的程序部分会具有怎样的影响。在这儿，他本质上有些类似于Gifford和Lucassen的效果系统\[GL88\](effect systems)

The examples in this paper are based on Haskell [HPW91], though any lazy functional language incorporating the Hindley/Milner type system would work as well.

>本文中的例子是基于Haskell [HPW91]，然而，它可以在任何采用Hindley-Milner类型系统的惰性函数式编程语言下工作。

The remainder of this paper is organised as follows. Section 2 uses list comprehensions to motivate the concept of a monad, and introduces monad comprehensions. Section 3 shows that variable binding (as in \let" terms) and control of evaluation order can be modelled by two trivial monads. Section 4 explores the use of monads to structure programs that manipulate state, and presents the new solution to the array update problem. Two examples are considered: renaming bound variables, and interpreting a simple imperative language. Section 5 extends monad comprehensions to include filters. Section 6 introduces the concept of monad morphism and gives a simple proof of the equivalence of two programs. Section 7 catalogues three more monads: parsers, exceptions, and continuations. Section 8 gives the translation schemes for interpreting lambda-calculus in an arbitrary monad. Two examples are considered: giving a semantics to a non-deterministic language, and deriving continuation-passing style.

>本文的其余部分组织如下。第二章采用列表推导式来引出单子的概念，并介绍单子推导式。第三章揭示如何采用两个平凡单子来对变量绑定（就像 let指令一样）和计算顺序控制进行建模。第四章探索如何采用单子构造处理状态的程序并对数组更新问题提出新的解决方案。将会考察两个例子，重命名以绑定变量和解析一个简单的命令式语言。第五章将在用过滤器对单子推导式进行扩展。第六章介绍单子态射（范畴学中的概念）的概念，并给出两个程序等价的简单证明。第七章列出三种单子：解析器，异常，持续调用。第八章给出转换规则来解释在任意单子中的lambda演算。会考察两个例子：给出非确定性语言的语义，以及引出后继传递格式(CPS)。

##2. Comprehensions and monads

>2\. 推导式和单子

###2.1 Lists

>2\.1 列表

Let us write `M x` for the data type of lists with elements of type `x`. (In Haskell, this is usually written `[x].`) For example, `[1 2 3]::M Int` and `['a','b','c']::M Char`. We write map for the higher-order function that applies a function to each elment of a list:

>让我们用 `M x` 代表元素类型为 `x` 的列表类型。（在Haskell中，通常被写成 `[x]`）。例如， `[1,2,3]::M Int` 或者是 `['a','b','c']::M Char`。我们可以通过写一个高阶函数`map`，将一个普通的函数作用于列表的所有元素。

```haskell
map::(x->y) -> (M x->M y)
```

(In Haskell, type variables are written with small letters, e.g. , `x` and `y` , and type constructors are written with capital letters, e.g. , `M` .) For example, if code `::Char->Int` maps a character to its ASCII code, then map code `['a','b','c'] = [97,98,99]`. Observe that

>(在Haskell中，类型变量采用小写字符表示，如 x 和 y， 类型构造器采用大写字母表示，如 M)，例如，如果代码 `::Char->Int` 将一个字符映射为它的ASCII码，那么代码`['a','b','c'] = [97,98,99]`看起来像这样：

```haskell
(i)       map id = id,
(ii) map (g * f) = (map g) * (map f)
```

Here `id` is the identity function, `id x = x` , and `g*f` is function composition, `(g*f ) x = g (f x)`.

>这儿的`id`是identify函数，`id x = x` ，而 `g*f` 是复合函数，`(g*f) x = g (f x)`。

In category theory, the notions of type and function are generalised to object and arrow . An operator `M` taking each object `x` into an object `M x` , combined with an operator map taking each arrow `f:: x -> y` into an arrow `map f:: M x -> M y` , and satisfying (i) and (ii), is called a functor. Categorists prefer to use the same symbol for both operators, and so would write `M f` where we write `map f` .

>在范畴理论中，类型和函数的概念被抽象为对象和箭头。操作 `M` 将一个对象 `x` 变为另一个对象 `M x`，结合将箭头 `f:: x -> y` 变为 另一个箭头 `map f:: M x -> M y` 的操作 `map`， 并且满足条件(i)和(ii)， 被称之为一个函子(functor)。范畴学家喜欢使用相同的标记表达这两种操作，因此可以将 `map f` 记为 `M f` 。

The function `unit` converts a value into a singleton lists, and the function `join` concatenates a list of lists into a list:

>函数`unit`将一个值转换为一个只有一个元素的列表，而函数`join`将一个双层列表中的列表元数链接为一个单层列表。

```haskell
unit:: x -> M x
join:: M(M x) -> M x
```

For example, `unit 3 = [3]` and `join [[1],[2], [3]] = [1,2,3]`. Observe that

>例如，`unit 3 = [3]`和`join [[1],[2], [3]] = [1,2,3]`。看起来像这样：

```haskell
(iii) (map f) * unit = unit * f
(iv)  (map f) * join = join * (map (map f))
```

Laws (iii) and (iv) may be derived by a systematic transformation of the polymorphic types of unit and join . The idea of deriving laws from types goes by the slogan "theorems for free" [Wad89] and is a consequence of Reynolds' abstraction theorem for polymorphic lambda calculus [Rey83].

>unit 和 join 的多态类型的系统变换必须遵循(iii)和(iv)原则。这种类型遵循规则的想法来自于slogan的 “自由定理”[Wad89]，以及它是 Reynolds的 “多态lambda演算的抽象原理”[Rey83] 的重要结果。

In categorical terms, `unit` and `join` are natural transformations. Rather than treat `unit` as a single function with a polymorphic type, categorists treat it as a family of arrows, `unit_x :: x -> M x` , one for each object `x` , satisfying `map f * unit_x = unit_y * f` for any objects `x` and `y` and any arrow `f :: x -> y` between them. They treat join similarly. Natural transformation is a simpler concept than polymorphic function, but we will stick with polymorphism since it's a more familiar concept to functional programmers.

>用范畴论的话说，`unit` 和 `join` 是自由变换。相对于将unit当成接受一个多态类型参数的函数，范畴学家将它看成一族箭头，`unit_x:: x -> M x` 对应每个对象`x`，对于任意的对象 `x` 和 `y`，和任意的箭头 `f:: x->y` ，满足 `map f * unit_x = unit_y * f`。他们对`join`函数同样处理。自然变换是一个比多态的函数更为简单范化的概念，但是，由于对于函数式程序员来说他们更像多态函数，因此我们就认为他们是多态函数好了。

###2.2 Comprehensions

>2.2 推导式

Many functional languages provide a form of list comprehension analogous to set comprehension. For example,

>许多函数式编程语言都提供列表推导式的方法，与集合推导式类似。例如：

```haskell
[(x,y) | x <- [1,2], y <- [3,4]] = [(1,3),(1,4),(2,3),(2,4)]
```
In general, a comprehension has the form `[t|q]`, where `t` is a term and `q` is a qualifier. We use the letters `t , u , v` to range over terms, and `p , q , r` to range over qualifiers. A qualifier is either empty, `A`; or a generator, `x<-u` , where `x` is a variable and `u` is a list-valued term; or a composition of qualifiers, `(p,q)`. Comprehensions are defined by the following rules:

>大体上，推导式会以`[t|q]`的形式表示，在这儿，`t`是语句，`q`是限定词。我们用字母 `t,u,v` 来表示语句，并且用 `p,q,r` 来表示限定词。一个限定词有三种表示形式，一是以`A`表示空，二是一个 `x<-u` 形式的生成器，其中`x`是变量而`u`是一个有值的列表，三是复合限定词，例如 `(p,q)`。推导式被定义为下面的规则：

```haskell
(1) [t|A] = unit t
(2) [t|x<-u] = map(\x -> t) u
(3) [t|(p,q)] = join [[t|q]|p]
```

(In Haskell, lambda-terms are written `(\x -> t )` rather than the more common `( \x.t )`.) Note the reversal of qualifiers in rule (3): nesting q inside p on the right-hand side means that, as we expect, variables bound in `p` may be used in `q` but not vice-versa.

>(在Haskell中，lambda语句被写成`(\x -> t)`而不是更为通俗的`(\x.t)`).特别注意规则(3)限定词翻转的情况：等式右边在p中嵌套q的情况,将有一个变量在p中被绑定且在q中被使用，反之则不其然。

For those familiar with list comprehensions, the empty qualifier and the parentheses in qualifier compositions will appear strange. This is because they are not needed. We will shortly prove that qualifier composition is associative and has the empty qualifier as `unit`. Thus we need not write parentheses in qualifier compositions, since `((p,q ),r)` and `(p,(q,r))` are equivalent, and we need not write `(q,A)` or `(A,q)` because both are equivalent to the simpler q . The only remaining use of A is to write `[t|A]`, which we abbreviate `[t]`.

>对于熟悉列表推导式的人，空的限定词和限定词组合中的圆括号看起来有点奇怪。因为他们不是必须的。我们稍后将证明限定词是可结合的，并且以空的限定词为`unit`。既然`((p,q ),r)` 和 `(p,(q,r))`是等价的，并且我们不需要写`(q,A)` 或者是 `(A,q)`因为他们都等价于单个q限定词，因此我们就不需要在限定词复合中在写圆括号。像`[t|A]`这种单独使用A的情况，我们可以简写为`[t]`.

Most languages that include list comprehensions also allow another form of qualifier, known as a filter, the treatment of which is postponed until Section 5.

>大多数具有列表推导特性的编程语言允许另外一种限定词，称之为过滤器，这个我们推迟到第五章讨论。

As a simple example, we have:

>我们有一个简单的例子：

```haskell
    [sqr x|x<-[1,2,3]]
=   {by (2)}
    map(\x -> sqr x) [1,2,3]
=   {reducing map}
    [1,4,9].
```

The comprehension in the initial example is computed as:

>对本章开始的例子的推导式会像这样计算：

```haskell
    [(x,y)| x<-[1,2], y<-[3,4]]
=   {by (3)}
    join [[(x,y)| y<-[3,4]]| x<-[1,2]]
=   {by (2)}
    join [map(\y->(x,y))[3,4] | x<-[1,2]]
=   {by (2)}
    join (map(\x->map(\y->(x,y))[3,4])[1,2])
=   {reducing map}
    join (map(\x->[(x,3),(x,4)])[1,2])
=   {reducing map}
    join [[(1,3),(1,4)],[(2,3),(2,4)]]
=   {reducing join}
    [(1,3),(1,4),(2,3),(2,4)].
```

From (i)-(iv) and (1)-(3) we may derive further laws:

>从(i)-(iv)和(1)-(3)我们可以导出更多的规则：

```haskell
(4) [f t| q]           =(map f)[t|q]
(5) [x| x<-u]          = u
(6) [t|p, x<-[u|q], r] = [t_x_u|p, q, r_x_u].
```

In (4) function `f` must contain no free occurrences of variables bound by qualifier `q` , and in (6) the term `t_x_u` stands for term `t` with term `u` substituted for each free occurrence of variable `x`, and similarly for the quali er `r_x_u` . Law (4) is proved by induction over the structure of qualifiers the proof uses laws (ii)-(iv) and (1)-(3). Law (5) is an immediate consequence of laws (i) and (2). Law (6) is again proved by induction over the structure of qualifiers, and the proof uses laws (1)-(4).

>在(4)中，`f`函数必须确保不包含被限定词`q`绑定的自由变量，并且在(6)中，语句`t_x_u`代表用语句`u`来置换每个自由变量`x`后的语句`t`，限定词`r_x_u`也是类似的情况。规则(4)采用规则(ii)-(iv)和(1)-(3)通过在限定词结构上的推理可以证明。规则(5)是规则(i)和(2)直接的结果。规则(6)可以采用规则(1)-(4)再次通过限定词结构上的推理证明。

As promised, we now show that qualifier composition is associative and has the empty qualifier as a unit:

>按照约定，我们现在展示以空限定词为`unit`，限定词是组合满足结合律的。

```haskell
(I')       [t|A,q] = [t|q]
(II')      [t|q,A] = [t|q]
(III') [t|(p,q),r] = [t|p,(q,r)]
```

First, observe that (I')-(III') are equivalent, respectively, to the following:

>首先，显而易见(I')-(III')对下面几个式子是分别等价的。

```haskell
(I)         join * unit = id
(II)  join * (map unit) = id
(III)         join*join = join * (map join)
```

To see that (II') and (II) are equivalent, start with the left side of (II') and simplify:

>为说明(II')和(II)是等价的，对(II')式的左侧进行简化：

```haskell
  [t|q,A]
= {by (3)}
  join [[t|A]| q]
= {by (1)}
  join [unit t | q]
= {by (4)}
  join (map unit[t|q])
```

That (II) implies (II') is immediate. For the converse, take [t|q] to be [x|x<-u] and apply (5). The other two equivalences are seen similarly.

>规则(II)直接的隐含规则(II').反过来，应用规则(5)将[t|q]写成[x|x<-u]的形式.剩下两个等价式子看起来是类似的。

Second, observe that laws (I)-(III) do indeed hold. For example:

>其次，注意规则(I)-(III)确实是有效的。例如：

```haskell
                join(unit[1,2]) = join[[1,2]]=[1,2]
            join(map unit[1,2]) = join[[1],[2]]=[1,2]
    join(join[[[1],[2]],[[3]]]) = join[[1],[2],[3]]=[1,2,3]
join(map join[[[1],[2]],[[3]]]) = join[[1,2],[3]]=[1,2,3]
```

Use induction over lists to prove (I) and (II), and over list of lists to prove (III).

>对列表进行推导可以证明规则(I)和(II),对列表的列表进行推导可以证明(III).

###2.3 Monads

>2.3 单子

The comprehension notation suits data structures other than lists. Sets and bags are obvious examples, and we shall encounter many others. Inspection of the foregoing lets us isolate the conditions under which a comprehension notation is sensible.

>推导记号除了适合列表以外，还适合其他数据结构。集合和包装就是显而易见的例子，除此之外，我们会列举其他一些事实。之前的审视让我们可以有一个独立的环境使得推导记号更能被理解。

For our purposes, a monad is an operator `M` on types together with a triple of functions satisfying laws (i)-(iv) and (I)-(III).

>根据我们的意图，单子包括两个部分：一个对类型的操作 `M` ，三个满足规则(i)-(iv)和规则(I)-(III)的函数：

```haskell
map :: (x->y)->(M x->M y)
unit :: x->M x
join :: M(M x) -> Mx
```

```haskell
(i)           map id = id,
(ii)     map (g * f) = (map g) * (map f)
(iii) (map f) * unit = unit * f
(iv)  (map f) * join = join * (map (map f))

(I)         join * unit = id
(II)  join * (map unit) = id
(III)         join*join = join * (map join)
```

Every monad gives rise to a notion of comprehension via laws (1)-(3). The three laws establish a correspondence between the three components of a monad and the three forms of qualifier: (1) associates `unit` with the empty qualifier, (2) associates `map` with generators, and (3) associates `join` with qualifier composition. The resulting notion of comprehension is guaranteed to be sensible in that it necessarily satisfies laws (4)-(6) and (I)-(III).

>每一个单子都通过规则(1)-(3)产生推导的概念。这三条规则建立起单子三个部件与三种限定词的之间的联系：(1)联系`unit`和空限定词，（2）联系`map`和生成器，（3）联系`join`和符合限定词。结果是，推导的意图显然被明智的限制于必须满足规则(4)-(6)和(I)-(III)。

```haskell
(1) [t|A] = unit t
(2) [t|x<-u] = map(\x -> t) u
(3) [t|(p,q)] = join [[t|q]|p]

(4) [f t| q]           =(map f)[t|q]
(5) [x| x<-u]          = u
(6) [t|p, x<-[u|q], r] = [t_x_u|p, q, r_x_u].
```

In what follows, we will need to distinguish many monads. We write `M` alone to stand for the monad, leaving the triple `(map_M,unit_M,join_M )` implicit, and we write `[t | q]_M` to indicate in which monad a comprehension is to be interpreted. The monad of lists as described above will be written List .

>在下文中，我们会有必要区别多种单子。我们单独写一个`M`代表单子，它暗含一个三元组`(map_M,unit_M,join_M )`，我们写`[t | q]_M`代表在某个单子中一个推导式需要被解析。上面描述过的列表的单子将直接写成`List`。

As an example, take `Set` to be the set type constructor, `map_Set` to be the image of a set under a function, `unit_Set` to be the function that takes an element into a singleton set, and `join_Set` to be the union of a set of sets:

>作为例子，将`Set`作为集合类型的构造器，`map_Set`作为集合关于某函数的像。`unit_Set`是一个将单个元素放入一个空集合的函数，`join_Set`是将集合中的集合联合起来的函数：

```haskell
unit_Set  x   = { x }
map_Set f x'  = { f x | x E x'}
join_Set  x'' = U x''
```
The resulting comprehension notation is the familiar one for sets. For instance, `[(x,y) | x<-x', y<-y' ]_Set` specifies the cartesian product of sets `x'` and `y'`.

>由此产生的集合的推导标记是如此熟悉。例如，`[(x,y) | x<-x', y<-y' ]_Set` 制定集合x'和y'的笛卡尔积。

We can recover `unit` , `map` , and `join` from the comprehension notation:

>我们能从推导标记还原`unit`，`map`和`join`：

```haskell
(1') unit  x   = [x]
(2') map f x'  = [f x | x <- x']
(3') join  x'' = [x | x'<-x'', x<-x']
```

Here we adopt the convention that if `x` has type `x` , then `x'` has type `M x` and `x''` has type `M (M x)`.

>在这儿我们按惯例，如果x的类型是x，那么x'的类型是M x,x''的类型是M (M x)。

Thus not only can we derive comprehensions from monads, but we can also derive monads from comprehensions. Define a comprehension structure to be any interpretation of the syntax of comprehensions that satisfies laws (5)-(6) and (I)-(III). Any monad gives rise to a comprehension structure, via laws (1)-(3) as we have seen, these imply (4)-(6) and (I)-(III). Conversely, any comprehension structure gives rise to a monad structure, via laws (1)-(3) it is easy to verify that these imply (i)-(iv) and (1)-(4), and hence (I)-(III).

>如此我们不只是能从单子进行推导，而且我们能从推导得出单子。定义一个推导结构，使其能作为任何符合规则(5)-(6)以及规则(I)-(III)的推导语法的解释。任何单子都可以产生一个推导结构，通过规则(1)-(3)我们可以看出，规则(4)-(6)和规则(I)-(III)是被暗含的。反过来，任何推导结构可以产生一个单子结构,通过规则(1)-(3)我们可以很容易验证，规则(i)-(iv)和规则(1)-(4)是被暗含的，并因此符合规则(I)-(III)。

The concept we arrived at by generalising list comprehensions, mathematicians arrived at by a rather different route. It first arose in homological algebra in the 1950's with the undistinguished name "standard construction" (sort of a mathematical equivalent of "hey you"). The next name, "triple", was not much of an improvement. Finally it was baptised a "monad". Nowadays it can be found in any standard text on category theory [Mac71, BW85, LS86].

>我们靠泛化列表推导式所得到的概念，数学家们靠相当不一样的方式获得。它第一次以一个非常普通的名字“标准构造(standard construction)”(差不多有点“嘿，你”的样子)出现在20世纪50年代的同调代数(homological algebra)中。它的另一个名字，“三元组(triple)”，并没有得到广泛使用。最后，它被命名为“单子(monad)”。现今，这个名字被发现用于各种标准的范畴理论[Mac71, BW85, LS86]文献中。

The concept we call a monad is slightly stronger than what a categorist means by that name: we are using what a categorist would call a strong monad in a cartesian closed category. Rougly speaking, a category is cartesian closed if it has enough structure to interpret lambda-calculus. In particular, associated with any pair of objects (types) `x` and `y` there is an object `[x -> y]` representing the space of all arrows (functions) from `x` to `y` . Recall that `M` is a functor if for any arrow `f :: x -> y` there is an arrow `map f :: M x -> M y` satisfying (i) and (ii). This functor is strong if it is itself represented by a single arrow `map :: [x -> y] -> [M x -> M y]`. This is all second nature to a generous functional programmer, but a stingy categorist provides such structure only when it is needed.

>我们这儿的单子的概念比范畴学家理解的稍微牛叉一些：我们采用的是实际上被范畴学家称为笛卡尔闭范畴上的强单子(a strong in a cartesian closed category)。Rougly说过，一个范畴如果它有足够去解释lambda演算的构造，那么它就是笛卡尔封闭的(cartesian closed)。特别的，为了关联任何一对对象`x`和`y`，有另一个对象`[x -> y]`从概念上描述所有`x`到`y`的之间的箭头(函数)。重申一下，如果对于任何箭头`f :: x->y`，都有一个箭头`map f :: M x -> M y`满足规则(i)和(ii)，那么`M`是一个函子(functor)。如果一个单箭头`map :: [x->y] -> [M x->M y]`能代表这个函子本身，那么这个函子是“强”的。这都是慷慨的函数式程序员的本能,而不是小气的范畴学家只有在必要时候所提供的结构。

It is needed here, as evidenced by Moggi's requirement that a computational monad have a strength, a function t :: (x,M y) -> M (x,y) satisfying certain laws [Mog89a]. In a cartesian closed category, a monad with a strength is equivalent to a monad with a strong functor as described above. In our framework, the strength is defined by t(x,y') = [(x,y) | y<-y']. (Following Haskell, we write (x,y) for pairs and also (x,y) for the corresponding product type.)

>根据Moggi的要求，一个可计算的单子有一个强度，我们需要一个满足某种规则[Mog89a]的函数`t :: (x,M y)->M (x,y)`。在笛卡尔封闭范畴中，根据上面说的，具有强度的单子等价于具有强函子的单子。在我们的框架中，强度被定义为`t(x,y') = [(x,y) | y<-y']`。（在Haskell中，我们采用(x,y) 表示pairs，同时也表示关联类型）。

Monads were conceived in the 1950's, list comprehensions in the 1970's. They have quite independent origins, but fit with each other remarkably well. As often happens, a common truth may underlie apparently disparate phenomena, and it may take a decade or more before this underlying commonality is unearthed.

>单子诞生于20世纪90年代，列表推导式诞生于20世纪70年代。他们都是比较独立的发展出来的，但他们彼此都能很好的相容。一个通用的本质会在两个毫不相干的表面现象之下，而发掘它又需要10年甚至是更多的时间。


##3 Two trivial monads

>3 两个平凡的单子

###3.1 The identity monad

>3.1 恒等单子(identity monad)

The identity monad is the trivial monad specified by

>恒等单子是一个平凡单子，它的定义是：

```haskell
type_Id  x = x
map_Id f x = f x
unit_Id  x = x
join_Id  x = x
```

so `map_Id` , `unit_Id` , and `bind_id` are all just the identity function. A comprehension in the identity monad is like a "let" term:

>因此`map_Id`,`unit_Id`以及`bind_id`都是恒等函数。恒等单子的推导式就像“let”语句一样：

```haskell
  [t | x <- u]_Id
= ((\x->t)u)
= (let x=u in t)
```
Similarly, a sequence of qualifiers corresponds to a sequence of nested "let" terms:

>类似的，顺序发生的限定词对于顺序嵌套的“let”语句：

```haskell
[t|x<-u,y<-v]_Id = (let x = u in (let y = v in t))
```

Since `y` is bound after `x` it appears in the inner "let" term. In the following, comprehensions in the identity monad will be written in preference to "let" terms, as the two are equivalent.

>既然`y`在`x`后被绑定，它出现在内部的“let”语句中。又因为两者是等价的，在下文中，恒等单子的推导式会优先于“let”语句写出。

In the Hindley-Milner type system, lambda-terms and "let" terms differ in that the latter may introduce polymorphism. The key factor allowing "let" terms to play this role is that the syntax pairs each bound variable with its binding term. Since monad comprehensions have a similar property, it seems reasonable that they, too, could be used to introduce polymorphism. However, the following does not require comprehensions that introduce polymorphism, so we leave exploration of this issue for the future.

>在Hindley-Milner类型系统中，lambda语句和“let”语句的不同在于后者会引入多态。关键点是允许“let”语句扮演一个这样的角色，这个语法将被绑定的变量和绑定的语句凑成一对。既然单子推导式有相似的属性，那么它能够被用来引入多态看起来也是合理的。然而，下文中并不要求推导式引入多态特性，因此我们将这个问题留给未来解决。

###3.2 The strictness monad

>3.2 严格单子(strictness monad)

Sometimes it is necessary to control order of evaluation in a lazy functional program. This is usually achieved with the computable function strict , defined by:

>有时候有必要在惰性函数式程序中去控制计算的顺序。通常需要采用严格计算的函数来实现，被定义成：

```haskell
strict f x = if x!= 丄 then f x else 丄
```

Operationally, `strict f x` is reduced by first reducing `x` to weak head normal form (WHNF) and then reducing the application `f x` . Alternatively, it is safe to reduce `x` and `f x` in parallel, but not allow access to the result until `x` is in WHNF.

>从操作上，`strict f x`是按照首先计算x到“弱头正则形式(weak head normal form (WHNF))”然后计算应用`f x`的计算过程。或者是同时计算`x`和`f x`，直到`x`已经是WHNF才会给出`f x`的结果。

We can use this function as the basis of a monad:

>我们能采用这些行书作为基本的单子：

```haskell
type_Str  x = x
map_Str f x = strict f x
unit_Str  x = x
join_Str  x = x
```

This is the same as the identity monad, except for the definition of map_Str . Monad laws (i), (iii)-(iv), and (I)-(III) are satisfied, but law (ii) becomes an inequality,

>除了对map_Str的精确定义，其他都和恒等单子一样。单子满足规则(i),(iii)-(iv)和(I)-(III)，但是规则(ii)有点不一样，

```haskell
(map_Str g) * (map_Str f) <= map_Str (g*f)
```

So Str is not quite a monad; categorists might call it a lax monad. Comprehensions for lax monads are defined by laws (1)-(3), just as for monads. Law (5) remains valid, but laws (4 ) and (6 ) become inequalities.

>因此Str不完全是一个单子；范畴学家可能会叫它宽松单子(a lax monad)。像单子一样，宽松单子的推导式由规则(1)-(3)定义。规则(5)仍然有用，只是规则(4)和(6)有些不一样。

We will use Str-comprehensions to control the evaluation order of lazy programs. For instance, the operational interpretation of

>我们将使用Str推导式控制惰性程序的计算顺序。例如，对下面这个推导式

```haskell
[t| x<-u,y<-v]_Str
```

is as follows: reduce `u` to WHNF, bind `x` to the value of `u` , reduce `v` to WHNF, bind `y` to value of `v` , then reduce `t` . Alternatively, it is safe to reduce `t` , `u` , and `v` in parallel, but not to allow access to the result until both `u` and `v` are in WHNF.

>操作起来像这样：计算`u`到WHNF，绑定`x`到`u`的值,计算`v`到WHNF，绑定`y`到`v`的值，然后计算`t`。或者是，安全的并行计算`t`，`u`，`v`，并且直到`u`和`v`为WHNF的时候返回整个结果。

##4 Manipulating state

>4 状态处理

Procedural programming languages operate by assigning to a state; this is also possible in impure functional languages such as Standard ML. In pure functional languages, assignment may be simulated by passing around a value representing the current state. This section shows how the monad of state transformers and the corresponding comprehension can be used to structure programs written in this style.

>过程式编程语言靠状态赋值来运转。这在不纯的函数式语言例如Standard ML也可以实现。在纯的函数式语言中，赋值有可能靠传递一个代表当前状态的值给函数来模拟。本节说明怎样用状态转换器单子和对应的推导式来构造此类程序。

### 4.1 State transformers

>状态转换器

Fix a type `S` of states. The monad of state transformers `ST` is defined by

>确定状态的类型是`S`。状态转换器单子`ST`定义为：

```haskell
type ST  x   = S->(x,S)
map_ST f x'  = \s->[(f x,s')|(x,s')<-x's]_Id
unit_ST  x   = \s->(x,s)
join_ST  x'' = \s->[(x,s'')|(x',s')<-x''s,(x,s'')<-x's']_Id
```

(Recall the equivalence of Id-comprehensions and "let" terms as explained in Section 3.1.) A state transformer of type `x` takes a state and returns a value of type `x` and a new state. The unit takes the value `x` into the state transformer `\s -> (x,s)` that returns `x` and leaves the state unchanged. We have that

>(回忆3.1节关于恒等推导式和“let”语句等价的说明)一个类型`x`的状态转换器，接受一个状态并且返回一个类型`x`的值和一个新状态。`unit`函数接受一个值`x`放到状态转换器`\s->(x,s)`中，这个转换器返回`x`和保持输入状态不变。我们有：

```haskell
[(x,y)|x<-x',y<-y']_ST = \s->[((x,y),s'')|(x,s')<-x's,(y,s'')<-y's']_Id
```

This applies the state transformer `x'` to the state `s` , yielding the value `x` and the new state `s'` it then applies a second transformer `y'` to the state `s'` yielding the value `y` and the newer state `s"`; finally, it returns a value consisting of `x` paired with `y` and the final state `s"`.

>这个推导式应用状态转换器`x'`到状态`s`上，然后生成`x`和新的状态`s‘`,然后应用第二个转换器`y'`到状态`s'`生成`y`和新的状态`s"`；最后，返回`x`和`y`的对和s"。

Two useful operations in this monad are:

>状态转换单子有两个有用的操作：

```haskell
fetch     :: ST S
fetch     =  \s->(s,s)
assign    :: S -> ST()
assign s' =  \s -> ((),s')
```

The first of these fetches the current value of the state, leaving the state unchanged; the second discards the old state, assigning the new state to be the given value. Here () is the type that contains only the value ().

>第一个用来获取状态的当前值,返回的状态不变；第二个丢弃久的状态，并赋与一个新的状态为返回值。在这儿()是一个类型，这个类型中只有一个值().

A third useful operation is:

>第三个有用的操作是：

```haskell
init      :: S->ST x->x
init s x' =  [x | (x,s')<-x's]_Id
```

This applies the state transformer `x'` to a given initial state `s`; it returns the value computed
by the state transformer while discarding the final state.

>这个操作应用状态转换器`x'`到给定的初始状态`s`；当丢弃掉最后的状态时它返回根据状态转换器计算的x的值。

###4.2 Example: Renaming

>4.2 例子： 重命名

Say we wish to rename all bound variables in a lambda term. A suitable data type Term for representing lambda terms is defined in Figure 1 (in Standard ML) and Figure 2 (in Haskell). New names are to be generated by counting; we assume there is a function

>我们希望重命名在lambda语句中所有绑定的变量。为表示“lambda语句”，一个恰当的数据类型`Term`定义在Figure 1(在Standard ML中)和Figure 2(在Haskell中)中。通过计数生成新的名字；我们假定有一个函数：

```haskell
mkname :: Int -> Name
```

that given an integer computes a name. We also assume a function

>这个函数给定一个整型值产生一个名字。然后我们又假定一个函数：

```haskell
subst :: Name -> Name -> Term -> Term
```

such that `subst x' x t` substitutes `x'` for each free occurrence of `x` in `t` .

>如此`subst x' x t`将每个自由出现在`t`中的`x`替换为`x'`。

A solution to this problem in the impure functional language Standard ML is shown in Figure 1. The impure feature we are concerned with here is state: the solution uses a reference N to an assignable location containing an integer. The "functions" and their types are:

>对这个问题的解决方案，对不纯的函数式编程语言Standard ML，可以参看Figure 1。在这儿我们所担心的不纯的特性是状态：这个方案采用了一个引用N到一个包含一个整型并可被赋值的位置。这个“函数”和它的类型是：

```haskell
newname :: ()->Name
renamer :: Term -> Term
rename  :: Term -> Term
```

Note that `newname` and `renamer` are not true functions as they depend on the state. In particular, `newname` returns a different name each time it is called, and so requires the dummy parameter `()` to give it the form of a "function". However, `rename` is a true function, since it always generates new names starting from 0. Understanding the program requires a knowledge of which "functions" affect the state and which do not. This is not always easy to see -- `renamer` is not a true function, even though it does not contain any direct reference to the state N , because it does contain an indirect reference through `newname`; but `rename` is a true function, even though it references `renamer`.

>注意，`newname`和`renamer`不是真正的函数，因为他们依赖与状态这个特性。特别的，`newname`每次调用都返回不同的名字，如此，需要一个假参数`()`来使它像“函数”的形式。然而，由于`rename`总是生成一个从0开始的新名字，那么它是真正的函数。正确理解这个程序需要知道那些“函数”会影响状态而那些则不会影响。这并不总是显而易见的--`renamer`不是真正的函数，即使它不包括任何直接对状态N的引用，这是因为它同过`newname`间接的引用了；而虽然`rename`引用了`renamer`，但`rename`是真正的函数。

An equivalent solution in a pure functional language is shown in Figure 2. This explicitly passes around an integer that is used to generate new names. The functions and their types are:

>图二展示了一个采用纯函数语言的等价的解决方案。显示的传送一个用来产生新名字的整型值。这样的函数和他们的类型定义为：

```haskell
newname :: Int -> (Name,Int)
renamer :: Term -> Int -> (Term,Int)
rename  :: Term -> Term
```

The function `newname` generates a new name from the integer and returns an incremented integer; the function `renamer` takes a term and an integer and returns a renamed term (with names generated from the given integer) paired with the final integer generated. The function `rename` takes a term and returns a renamed term (with names generated from 0). This program is straightforward, but can be difficult to read because it contains a great deal of "plumbing" to pass around the state. It is relatively easy to introduce errors into such programs, by writing `n` where `n'` is intended or the like. This "plumbing problem" can be more severe in a program of greater complexity.

>函数`newname`通过一个整型值生产一个新名字，并返回下一个整型值；函数`renamer`输入一个语句和一个整型值，返回一个采用新名字的term（采用从传入的整型值生成的名字）以及最终再产生的整型值。函数`rename`接受一个语句并返回一个被重命名后的语句(以0产生的名字)。这个程序是直接的，但又是难以阅读的，因为它包含大量的“管子”去传递状态。它很容易在程序中引入错误,比如可能会将本来需要传`n'`的地方却传递了一个`n`。在异常复杂的程序中，这样的“管子错误”会更加的严重。

finally, a solution of this problem using the monad of state transformers is shown in Figure 3. The state is taken as `S = Int` . The functions and their types are now:

>最后，采用状态转换器单子解决这个问题的方案在图3中。状态以S(= Int)的形式被接受。函数和他们的类型是这样：

```haskell
newname :: ST Name
renamer :: Term -> ST Term
rename  :: Term -> Term
```

The monadic program is simply a different way of writing the pure program: expanding the monad comprehensions in Figure 3 and simplifying would yield the program in Figure 2. Types in the monadic program can be seen to correspond directly to the types in the impure program: an impure "function" of type `U -> V` that affects the state corresponds to a pure function of type `U -> ST V` . Thus, `renamer` has type `Term -> Term` in the impure program, and type `Term -> ST Term` in the monadic program and `newname` has type `() -> Name` in the impure program, and type `ST Name` , which is isomorphic to `() -> ST Name` , in the pure program. Unlike the impure program, types in the monadic program make manifest where the state is affected (and so do the ST-comprehensions).

>采用单子的程序的写法与一般纯函数式程序有点不一样：需要在图3中展开单子推导式并在图2中简化将要生成的程序。单子化程序中的类型看起来和非纯函数式程序中的类型很像：`U->V`这样影响状态的非纯函数类型对应纯函数式程序中的类型`U->ST V`。如此，`renamer`在非纯函数程序中有类型`Term->Term`，在单子化程序中有`Term->ST Term`。而`newname`在非纯函数程序中有`()->Name`，在纯函数式程序中又有类型`ST Name`以及同态的`()->ST Name`与之对应.和非纯函数式程序不一样，在单子化程序中，类型使状态在何处受到影响显现出来（ST-推导式也是这样）。

The "plumbing" is now handled implicitly by the state transformer rather than explicitly. Various kinds of errors that are possible in the pure program (such as accidentally writing `n` in place of `n'`) are impossible in the monadic program. Further, the type system ensures that plumbing is handled in an appropriate way. For example, one might be tempted to write, say, `App (renamer t) (renamer u)` for the right-hand side of the last equation defining `renamer` , but this would be detected as a type error.

>现在“管子”被状态转换器由隐藏起来。各种在纯函数式程序中可能产生的错误（例如意外的将`n'`写成`n`）在单子化程序中不再出现。此外，类型系统肯定会以合适的方式处理“管子”。例如，如果你在定义`renamer`时，在最后一个等式的右边写了`App (renamer t) (renamer u)`，这将会作为一个类型错误被检查出来。

Safety can be further ensured by making ST into an abstract data type on which map_ST , unit_ST , join_ST , fetch , assign , and init are the only operations. This guarantees that one cannot mix the state transformer abstraction with other functions which handle the state inappropriately. This idea will be pursued in the next section.

>此外，确定可以安全的将`ST`作为一个只有map_ST,unit_ST,join_ST,fetch,assign和init操作的抽象数据类型。这样可以保证，不会将状态转换器的抽象行为和一些不合适的状态操作函数混淆。我们将在后面的章节中保持这样的观点。

Impure functional languages (such as Standard ML) are restricted to using a strict (or call-by-value) order of evaluation, because otherwise the effect of the assignments becomes very difficult to predict. Programs using the monad of state transformers can be written in languages using either a strict (call-by-value) or lazy (call-by-name) order of evaluation. The state-transformer comprehensions make clear exactly the order in which the assignments take effect, regardless of the order of evaluation used.

>不纯的函数式语言（例如Standard ML）被限制使用严格(或者叫做call-by-value)的计算顺序，如若不然，赋值的影响会变得难以预测。采用状态转换器单子的程序，可以用严格的（call-by-value）或者惰性的（call-by-name）的语言编写。不管计算顺序如何，状态转换推导式使赋值带来影响的顺序清晰明了.

Reasoning about programs in impure functional languages is problematic (although not impossible -- see [MT89] for one approach). In contrast, programs written using monads, like all pure programs, can be reasoned about in the usual way, substituting equals for equals. They also satisfy additional laws, such as the following laws on qualifiers:

>上面对在非纯函数式语言程序的论证是有问题的（虽然这并非不可能 -- 参看[MT89]中的一个方法）。相反，在所有的纯函数式程序中，采用单子写出的程序，可以采用等式替换的常用方法进行证明。他们同样会满足一些额外的规则，比如下面这些在限定词中的规则：

```haskell
        x<-fetch, y<-fetch = x<-fetch, y<-[x]_ST
    ()<-assign u, y<-fetch = ()<-assign u, y<-[u]_ST
()<-assign u, ()<-assign v = ()<-assign v
```

and on terms:

>以及在语句中的规则：

```haskell
                  init u [t]_ST = t
 init u [t | ()<-assign v,q]_ST = init v [t | q]_ST
init u [t | q, ()<-assign v]_ST = init u [t | q]_ST
```

These, together with the comprehension laws (5), (6), and (I')--(III'), allow one to use equational reasoning to prove properties of programs that manipulate state.

>上面这写，加上推导式规则(5),(6),以及(I')--(III'),可以以之进行恒等推理来证明状态处理程序的特性。

###4.3 Array update

> 数组更新

Let `Arr` be the type of arrays taking indexes of type `Ix` and yielding values of type `Val` . The key operations on this type are

>使`Arr`是数组的类型，它的下标类型为`Ix`，通过下标产生的值的类型是`Val`。在数组上的关键操作是：

```haskell
newarray :: Val -> Arr,
index    :: Ix -> Arr -> Val,
update   :: Ix -> Val -> Arr -> Arr
```

Here `newarray v` returns an array with all entries set to `v`; and `index i a` returns the value at index `i` in array `a`; and `update i v a` returns an array where index `i` has value `v` and the remainder is identical to `a` . In equations,

>这儿的`newarray v`返回一个所有条目都是`v`的数组；`index i a`返回在数组`a`中下标为`i`的值；`update i v a`返回一个数组，这个数组的下标`i`初的值为`v`，其他的都与`a`一样。在下面恒等式中，

```haskell
index i (new array v)  = v
index i (update i v a) = v
index i(update i' v a) = index i a, if i!=i'
```

The efficient way to implement the update operation is to overwrite the specified entry of the array, but in a pure functional language this is only safe if there are no other pointers to the array extant when the update operation is performed.

>有效率的方式实现更新操作是重写数组中那个确定的条目，但是在纯函数编程中只有当没有其他的指针仍指向这个数组，在更新的时候才是安全的。

Now consider the monad of state transformers taking the state type S = Arr , so that

>现在考虑状态转换器单子接受状态的类型 `S=Arr`， 因此：

```haskell
type ST x = Arr -> (x,Arr)
```

Variants of the `fetch` and `assign` operations can be defined to act on an array entry specified by a given index, and a variant of `init` can be defined to initialise all entries in an array to a given value:

>定义变异的`fetch`和`assign`操作，这些操作可以根据给定的下标作用于数组的指定成员，定义变异的`init`，根据给定值来初始化数组中的所有成员。

```haskell
fetch      :: Ix -> ST Val
fetch i    = \a -> [(v,a) | v <- index i a]_Str

assign     :: Ix -> Val -> ST()
assign i v = \a -> ((),update i v a)

init       :: Val -> ST x -> x
init v x'  =  [x | (x,a) <- x'(newarray v)]_Id
```

A Str-comprehension is used in `fetch` to force the entry from `a` to be fetched before `a` is made available for further access; this is essential in order for it to be safe to update `a` by overwriting.

>严格单子推导式用在`fetch`中，强迫在`a`变得可用前,从中取出要取的成员，以提供进一步的访问。这是基本的靠重写的方式安全更新`a`的顺序。

Now, say we make `ST` into an abstract data type such that the only operations on values of `type ST` are `map_ST` , `unit_ST` , `join_ST` , `fetch` , `assign` , and `init` . It is straightforward to show that each of these operations, when passed the sole pointer to an array, returns as its second component the sole pointer to an array. Since these are the only operations that may be used to build a term of `type ST` , this guarantees that it is safe to implement the assign operation by overwriting the specified array entry.

>现在，可以说我们使`ST`成为一个抽象数据类型，以致只有`map_ST` , `unit_ST` , `join_ST` , `fetch` , `assign` , 和 `init`等操作可作用于`type ST`类型的值上。很显然，每一个操作，当传递一个指向数组的单独指针时，将在返回值的第二个分量中返回一个指向数组的单独指针。既然这些是仅有的构造`type ST`语句的操作，这就保证可以安全的实现以重写的方式对一个特点的数组成员进行赋值。

The key idea here is the use of the abstract data type. Monad comprehensions are not essential for this to work, they merely provide a desirable syntax.

>主要思想是使用抽象数据类型。单子推导式不是一切可以工作的基础，它们仅仅提供一个可用的语法。

###4.4 Example: Interpreter

>样例：解释器

Consider building an interpreter for a simple imperative language. The store of this language will be modelled by a state of type Arr , so we will take Ix to be the type of variable names, and Val to be the type of values stored in variables. The abstract syntax for this language is represented by the following data types:

>考虑构建一个简单命令式语言的解释器。这个语言的存储方式采用一个`Arr`类型的状态为模型，因此我们将使`Ix`作为变量名的类型，并且`Val`作为存储在变量中的值的类型。这个语言的抽象语法以下面的数据类型的形式展现：

```haskell
data Exp  = Var Ix | Const Val | Plus Exp Exp
data Com  = Asgn Ix Exp | Seq Com Com | If Exp Com Com
data Prog = Prog Com Exp
```

An expression is a variable, a constant, or the sum of two expressions; a command is an assignment, a sequence of two commands, or a conditional; and a program consists of a command followed by an expression.

>一个表达式是一个变量，常量或者是两个表达式只和；一个命令是一个赋值语句，一个由两个命令组成的序列，或者是一个选择语句；一个程序由一个命令紧接着一个表达式组成。

A version of the interpreter in a pure functional language is shown in Figure 4. The interpreter can be read as a denotational semantics for the language, with three semantic functions:

>图4是采用纯函数语言写的某个版本的解释器。这个语义可以作为这个语言的指称语义来阅读，具有三个语义函数：

```haskell
exp :: Exp -> Arr -> Val
com :: Com -> Arr -> Arr
prog :: Prog -> Val
```

```haskell
--Figure 4 :Interpreter in a pure functional language
exp                 :: Exp -> Arr -> Val
exp (Var i ) a       = index i a
exp (Const v ) a     = v
exp (Plus e1 e2 ) a  = exp e1 a + exp e2 a

com                 :: Com -> Arr -> Arr
com (Asgn i e ) a    = update i (exp e a ) a
com (Seq c1 c2 ) a   = com c2 (com c1 a )
com (If e c1 c2 ) a  = if exp e a = 0 then com c1 a else com c2 a

prog                :: Prog -> Val
prog (Prog c e )     = exp e (com c (newarray 0 ))
```


The semantics of an expression takes a store into a value; the semantics of a command takes a store into a store; and the semantics of a program is a value. A program consists of a command followed by an expression; its value is determined by applying the command to an initial store where all variables have the value 0 , and then evaluating the expression in the context of the resulting store.

>表达式的语义将一个存储器变为一个值；命令语义将一个存储变为另一个存储；程序语义就是一个值。一个程序由表达式以及在它之上的命令构成；它的值由在一个已经被初始化为0的存储上应用命令，并在最后存储器的上下文中，计算表达式来决定。

The interpreter uses the array operations `newarray` , `index` , and `update` . As it happens, it is safe to perform the updates in place for this program, but to discover this requires using one of the special analysis techniques cited in the introduction.

>解释器使用数组的操作`newarray`，`index`，已经`update`。当它运行时，这个程序中进行更新操作是安全的，而发现这点需要使用在前言中引入的一种特别的分析技术。

The same interpreter has been rewritten in Figure 5 using state transformers. The semantic functions now have the types:

>图5是采用状态转换器实现的相同的解释器。语义函数现在的类型是：

```haskell
exp  :: Exp -> ST Val
com  :: Com -> ST ()
prog :: Prog -> Val
```

```haskell
--Figure 5: Interpreter with state transformers
exp               :: Exp -> ST Val
exp (Var i)      = [v | v <- fetch i ]_ST
exp (Const v)    = [v]_ST
exp (Plus e1 e2) = [v1 + v2 | v1 <- exp e1, v2 <- exp e2]_ST

com               :: Com -> ST ()
com (Asgn i e)    = [() | v <- exp e, () <- assign i v ]_ST
com (Seq c1 c2)   = [() | () <- com c1, () <- com c2 ]_ST
com (If e c1 c2)  = [() | v <- exp e, () <- if v = 0 then com c1 else com c2 ]_ST

prog              :: Prog -> Val
prog (Prog c e)   = init 0 [v | () <- com c, v <- exp e]_ST
```

The semantics of an expression depends on the state and returns a value; the semantics of a command transforms the state only; the semantics of a program, as before, is just a value. According to the types, the semantics of an expression might alter the state, although in fact expressions depend the state but do not change it - we will return to this problem shortly.

>表达式语义获取一个状态并返回一个值；命令语义只转换状态；程序语义，和以前一样，只是一个值。根据类型，表达式语义可能会更改状态，虽然事实上表达式不会改变所依赖的状态--我们会直接返回到程序中。

The abstract data type for ST guarantees that it is safe to perform updates (indicated by assign) in place - no special analysis technique is required. It is easy to see how the monad interpreter can be derived from the original, and (using the definitions given earlier) the proof of their equivalence is straightforward.

>`ST`抽象数据类型确保可以安全的进行适当的更新操作（确切的说是用`assign`操作）-- 不需要专门的分析技术。可以很容易看出单子解释器如何演变，并且（采用之前给出的定义）它们等价的证明是显而易见的。

The program written using state transformers has a simple imperative reading. For instance, the line

>采用状态转换器写一个简单的命令式读取程序。例如这一行：

```haskell
com(Seq c1 c2) = [()|() <- com c1, () <- com c2]_ST
```

can be read "to evaluate the command `Seq c1 c2` , first evaluate `c1` and then evaluate `c2` ". The types and the use of the ST comprehension make clear that these operations transform the state; further, that the values returned are of type () makes it clear that only the effect on the state is of interest here.

>可以被读成：“计算命令`Seq c1 c2`，首先计算`c1`，然后再计算`c2`”。类型和ST推导式的使用，使这些操作对状态的转换更清晰；之后，返回`()`类型使清晰的认识到影响状态是这儿唯一感兴趣的事情。

One drawback of this program is that it introduces too much sequencing. The line

>这个程序的缺点是引入太多的对计算顺序的约定。这行：

```haskell
exp (Plus e1 e2) = [v1 + v2 | v1 <- exp e1, v2 <- exp e2]_ST
```

can be read "to evaluate `Plus e1 e2` , first evaluate `e1` yielding the value `v1` , then evaluate `e2` yielding the value `v2` , then add `v1` and `v2` ". This is unfortunate: it imposes a spurious ordering on the evaluation of `e1` and `e2` (the original program implies no such ordering). The order does not matter because although exp depends on the state, it does not change it. But, as already noted, there is no way to express this using just the monad of state transformers. To remedy this we will introduce a second monad, that of state readers.

>可以被读成：“要计算 `Plus e1 e2`，首先计算`e1`并生成`v`，然后计算`e2`并生成`v2`，然后将`v1`和`v2`相加”。不幸的是：强加了一个虚假的顺序在计算`e1`和`e2`上（程序原本暗示是没有这样的顺序）。这个顺序其实是无所谓的，因为虽然exp依赖状态，但并不改变它。但是，如前面所述，依然采用状态转换器单子的话，没有方法可以表达这个（顺序无关特点）。为了弥补这个缺陷，我们将介绍第二个单子，那就是状态阅读器。

###4.5 State readers

>状态阅读器

Recall that the monad of state transformers, for a fixed type `S` of states, is given by

回忆状态转换器单子，如果状态类型确定为`S`，给定为：

```haskell
type ST x = S -> (x,S)
```

The monad of state readers, for the same type S of states, is given by

>状态阅读器单子，同样状态类型确定为`S`，给定为：

```haskell
type SR x   = S -> x
map_SR f x' = \s -> [f x | x <- x' s]_Id
unit_SR x   = \s -> x
join_SR x'' = \s -> [x | x' <- x''s, x <- x's]_Id
```

Here `x'` is a variable of type `SR x` , just as `x'` is a variable of type `ST x` . A state reader of type `x` takes a state and returns a value (of type `x` ), but no new state. The `unit` takes the value `x` into the state transformer `\s -> x` that ignores the state and returns `x` . We have that

>这儿的`x'`是类型`SR x`的变量，就像`x‘`是类型`ST x`的变量一样。一个状态阅读器类型`x`接收一个状态并返回一个值（`x`类型的），但是没有新状态产生。函数`unit`接收一个值`x`，并将它放入忽略新状态而返回`x`的状态转换器中状态转换器。我们有：

```haskell
[(x,y) | x <- x', y <- y']_SR = \s -> [(x,y) | x <- x's, y <- y's]_Id
```

This applies the state readers `x'` and `y'` to the state `s` , yielding the values `x` and `y` , which
are returned in a pair.

>可以应用状态转换器`x'`和`y'`到状态`s`上，并生成`x`和`y`成对返回。

It is easy to see that:

>容易看出：

```haskell
[(x,y) | x <- x', y <- y']_SR = [(x,y) | y <- y', x <- x']_SR
```

so that the order in which `x'` and `y'` are computed is irrelevant. A monad with this property is called commutative, since it follows that

>因此，`x'`和`y'`的计算顺序是无影响的。一个单子如果满足下面的规则，则称为满足交互律。

```haskell
[t | p, q]_SR = [t | q, p]_SR
```

for any term `t` , and any qualifiers `p` and `q` such that `p` binds no free variables of `q` and vice-versa. Thus state readers capture the notion of order independence that we desire for expression evaluation in the interpreter example.

>对于任何语句`t`，和任何限定词`p`和`q`，如果`p`没有绑定任何`q`的自由变量以及反之亦然。如此状态阅读器获得了，我们希望在解释器例子中表达式计算的顺序独立性概念。

Two useful operations in this monad are:

>在这个单子中，两个有用的操作是：

```haskell
fetch :: SR S
fetch =  \s -> s

ro    :: SR x -> ST x
ro x' =  \s -> [(x,s) | x <- x's]_Id
```

The first is the equivalent of the previous `fetch` , but now expressed as a state reader rather than a state transformer. The second converts a state reader into the corresponding state transformer: one that returns the same value as the state reader, and leaves the state unchanged. (The name `ro` abbreviates "read only".)

>第一个与以前的`fetch`函数相同，不过现在专门作为状态阅读器而不是状态转换器。第二个将一个状态阅读器转换为对应的状态转换器：不改变状态，返回与状态阅读器一样的值。（名字 `ro` 是 ”read only“的缩写）

In the specific case where `S` is the array type `Arr` , we define

>特别的，当`S`是数组类型`Arr`时，我们定义：

```haskell
fetch   :: Ix -> SR Val
fetch i =  \a -> index i a
```

In order to guarantee the safety of update by overwriting, it is necessary to modify two of the other definitions to use Str-comprehensions rather than Id-comprehensions:

>为了确保安全的以重写的方式更新，有必要采用Str推导式代替Id推导式修改另外两个定义：

```haskell
map_SR f x' = \a -> [f x | x <- x' a]_Str
ro x'       = \a -> [(x,a) | x <- x' a]_Str
```

These correspond to the use of an Str-comprehension in the ST version of fetch .

>这些对应ST版本采用Str推导式的`fetch`。

Thus, for arrays, the complete collection of operations on state transformers and state readers consists of

>这样，对数组来说，状态转换器和状态阅读器所有的操作集合由下面几个：

fetch  :: Ix -> SR Val
assign :: Ix -> Val -> ST()
ro     :: SR x -> ST x
init   :: Val -> ST x -> x

together with `map_SR` , `unit_SR` , `join_SR` and `map_ST` , `unit_ST` , `join_ST` . These ten operations should be defined together and constitute all the ways of manipulating the two mutually defined abstract data types `SR x` and `ST x` . It is straightforward to show that each operation of type `SR`, when passed an array, returns a value that contains no pointer to that array once it has been reduced to weak head normal form (WHNF); and that each operations of type `ST` , when passed the sole pointer to an array, returns as its second component the sole pointer to an array. Since these are the only operations that may be used to build values of types `SR` and `ST` , this guarantees that it is safe to implement the `assign` operation by overwriting the specified array entry. (The reader may check that the Str-comprehensions in `map_SR` and `ro` are essential to guarantee this property.)

>以及`map_SR` , `unit_SR` , `join_SR` 和 `map_ST` , `unit_ST` , `join_ST`一起。这十个操作应该定义在一起，并构成两种定义上相关的抽象数据类型`SR x`和`ST x`的所有处理方法。直接的表明，当传递一个数组的时候，每个`SR`类型的操作，一但被计算为弱头正则形式，会返回一个不再指向那个数组的值；并且每个`ST`类型的操作，当传入独立指向数组的指针时，将在第二分量返回指向那个数组的指针。既然这些是唯一用来构建`SR`和`ST`类型值的操作，这就确保可以安全的实现以重写某个数组单元的方式的`assign`操作。（阅读器会检查`map_SR`和`ro`的Str推导式，这是确保这个特性的基础）

Returning to the interpreter example, we get the new version shown in Figure 6. The only difference from the previous version is that some occurrences of `ST` have changed to `SR` and that `ro` has been inserted in a few places. The new typing

>回到解释器的例子，我们得到在图6中的新版本。与之前版本的唯一不同在于，一些`ST`中发生的事件已经被修改成`SR`了，而且几个地方加入了`ro`。新的类型是：

```haskell
exp :: Exp -> SR Val
```

```haskell
--Figure 6: Interpreter with state transformers and readers
exp             :: Exp -> SR Val
exp(Var i)      =  [v | v <- fetch i]_SR
exp(Const v)    =  [v]_ST
exp(Plus e1 e2) =  [v1+v2 | v1 <- exp e1, v2 <- exp e2]_SR

com             :: Com -> ST()
com(Asgn i e)   =  [() | v <- ro(exp e), () <- assign i v]_ST
com(Seq c1 c2)  =  [() | () <- com c1, () <- com c2]_ST
com(If e c1 c2) =  [() | v <- ro(exp e), () <- if v=0 then com c1 else com c2]_ST

prog            :: Prog -> Val
prog(Prog c e)  =  init 0 [v | () <- com c, v <- ro(exp e)]_ST
```

makes it clear that `exp` depends on the state but does not alter it. A proof that the two versions are equivalent appears in Section 6.

>可以清晰的看出，`exp`依赖状态而不是改变它。这两个版本等价的证明在第6节。

The excessive sequencing of the previous version has been eliminated. The line

>之前版本的过多的排序已经被消除。这一行：

```haskell
exp(Plus e1 e2) = [v1+v2 | v1 <- exp e1, v2 <- exp e2]_SR
```

can now be read "to evaluate `Plus e1 e2` , evaluate `e1` yielding the value `v1` and evaluate `e2` yielding the value `v2` , then add `v1` and `v2` ". The order of qualifiers in an SR-comprehension is irrelevant, and so it is perfectly permissible to evaluate `e1` and `e2` in any order, or even concurrently.

>现在可以读成”要计算`Plus e1 e2`，需要计算`e1`生成值`v1`并计算`e2`生成`v2`，然后将`v1`和`v2`相加。限定词在SR推导式中是顺序无关的，因此它完美的允许以任何顺序计算`e1`和`e2`，甚至是并发的。

The interpreter derived here is similar in structure to one in [Wad90], which uses a type system based on linear logic to guarantee safe destructive update of arrays. (Related type systems are discussed in [GH90, Wad91].) However, the linear type system uses a "let!" construct that suffers from some unnatural restrictions: it requires hyperstrict evaluation, and it prohibits certain types involving functions. By contrast, the monad approach requires only strict evaluation, and it places no restriction on the types. This suggests that a careful study of the monad approach may lead to an improved understanding of linear types and the "let!" construct.

>这儿的解释器有些类似于[Wad90]中的某个结构，这个结构采用基于线性逻辑的类型系统来确保安全的的更新数组。(类型系统相关的讨论在[GH90,Wad91]中)。然而，线性类型系统采用了一个拥有奇怪限制的“let!”构造：它要求超级严格的计算，并且它禁止类型包含函数。相对的，单子方法只要求严格的计算，并且在类型上并无严格限制。这样我们就可以假定，注重单子方法的研究可能提升对线性类型和“let!”构造的理解。

##5 Filters

>5 过滤器

So far, we have ignored another form of qualifier found in list comprehensions, the filter. For list comprehensions, we can define filters by

>至今为止，我们都忽略了列表推导式中限定词的另外一种格式，过滤器。在列表推导式中，我们能这样定义过滤器：

```haskell
[t | b] = if b then [t] else []
```

where `b` is a boolean-valued term. For example,

>这儿`b`是一个布尔值语句。例如：

```haskell
  [x | x <- [1,2,3], odd x]
= join [[x | odd x] | x <- [1,2,3]]
= join [[1 | odd 1], [2 | odd 2], [3 | odd 3]]
= join [[1], [], [2]]
= [1, 3]
```

Can we define filters in general for comprehensions in an arbitrary monad `M` ?  The answer is yes, if we can define `[]` for `M` . Not all monads admit a useful definition of `[]`, but many do.

>我们能为任意单子`M`的推导式定义一般化的过滤器吗？如果我们能定义`M`的`[]`，那答案为是。不是所有的单子都拥有`[]`定义，但是大多数都是具有的。

Recall that comprehensions of the form `[t]` are defined in terms of the qualifier A, by taking `[t] = [t | A]`, and `A` that is a `unit` for qualifier composition,

>回忆一下型如`[t]`的推导式使用限定词A的语句定义，如`[t] = [t | A]`，并且`A`在限定词符合中作为`unit`。

```haskell
[t | A, q] = [t | q] = [t | q, A]
```

Similarly, we will define comprehensions of the form `[]` in terms of a new qualifier, `Φ`, by taking `[] = [t | Φ]`, and we will require that `Φ` is a zero for qualifier composition,

>类似的，我们将在一个新的限定词`Φ`语句中定义`[]`形式的推导式，如`[] = [t | Φ]`，并且我们要求`Φ`对于限定词组合来说是一个零元。

```haskell
[t | Φ, q] = [t | Φ] = [t | q, Φ]
```

Unlike with `[t | A]`, the value of `[t | Φ]` is independent of `t` !

>和`[t | A]`不一样，`[t | Φ]`的值与`t`无关。

Recall that for `A` we introduced a function `unit :: x -> M x` satisfying the laws:

>回忆一下,对于`A`，我们介绍的函数`unit :: x -> M x`需要满足规则：

```haskell
(iii)    map f * unit = unit * f
(I)       join * unit = id
(II)  join * map unit = id
```

and then defined `[t | A] = unit t` .

>并且我们定义`[t | A] = unit t`。

Similarly, for `Φ` we introduce a function

>类似的，对于`Φ`我们介绍一个函数：

```haskell
zero :: y -> M x
```

满足下面的规则：

```haskell
(v)    map f * zero = zero * g
(IV)    join * zero = zero
(V) join * map zero = zero
```

and define

>并定义：

```haskell
(7) [t | `Φ`] = zero t
```

Law (v) specifies that the result of `zero` is independent of its argument, and can be derived from the type of `zero` (again, see [Rey83, Wad89]). In the case of lists, setting `zero y = []` makes laws (IV) and (V) hold, since `join [] = []` and `join [[],...,[]] = []`. (This ignores what happens when `zero` is applied to `丄`, which will be considered below.)

>规则(v)说明`zero`的结果与他的参数无关，并且能从`zero`类型导出（参考[Rey83, Wad89]）。在列表的例子中，既然`join [] = []`并且`join [[],...,[]] = []`，设定`zero y = []`使规则(IV)和(V)成立。(这些结论忽略了当`zero`应用到`丄`的情况，这种情况将在下面考虑)

Now, for a monad with `zero` we can extend comprehensions to contain a new form of qualifier, the filter, defined by

>现在，我们能为带`zero`的单子扩展推导式，使之拥有一个新的限定词，过滤器，定义为：

```haskell
（8） [t | b] = if b then [t] else []
```

where `b` is any boolean-valued term. Recall that laws (4) and (6) were proved by induction on the form of qualifiers； we can show that for the new forms of qualifiers, defined by (7) and (8), they still hold. We also have new laws

>这儿的`b`可以是任何代表布尔值的语句。回忆规则(4)和(6)以限定词形式的归纳法证明；我们能说靠规则(7)和(8)定义的新限定词仍然满足（(4)和(6)）。我们同样有新规则：

```haskell
(9)  [t | b, c] = [t | (b ^ c)]
(10) [t | q, b] = [t | b, q]
```

where `b` and `c` are boolean-valued terms, and where `q` is any qualifier not binding variables free in `b` .

>这儿的`b`和`c`是代表布尔值的语句，这儿的`q`可以是任何没有在`b`中绑定自由变量的限定词。

When dealing with `丄` as a potential value, more care is required. In a strict language, where all functions (including `zero`) are strict, there is no problem. But in a lazy language, in the case of lists, laws (v) and (IV) hold, but law (V) is an inequality, `join * map zero <= zero` , since `join (map zero ) 丄 = 丄` but `zero 丄 = []`. In this case, laws (1)-(9) are still valid, but law (10) holds only if `[t | q] /= 丄`. In the case that `[t | q] = 丄`, law (10) becomes an inequality, `[t | q, b] <= [t | b, q]`.

>当`丄`作为一个可用的值的时候，需要更加小心。在严格的编程语言中，所有的函数（包括`zero`）是严格的，这没有问题。但是在惰性语言中，比如在列表的例子中，规则(v)和(IV)需要保证，而规则(V)有点不同，既然`join (map zero ) 丄 = 丄`而又有`zero 丄 = []`，那么`join * map zero <= zero`。在这个例子中，规则(1)-(9)是仍然是有效的，而规则(10)只在`[t | q] /= 丄`时有效。在`[t | q] = 丄`的情况下，规则(10)也有点不同，`[t | q, b] <= [t | b, q]`。

As a second example of a monad with a `zero`, consider the strictness monad `Str` defined in Section 3.2. For this monad, a `zero` may be defined by `zero_Str y = 丄`. It is easy to verify that the required laws hold; unlike with lists, the laws hold even when `zero` is applied to `丄`. For example, `[x - 1 | x >= 1 ]_Str` returns one less than `x` if `x` is positive, and `丄` otherwise.

>作为第二个带`zero`的单子的例子，考虑3.2节定义的严格单子`Str`。对于这个单子，`zero`可能以`zero_Str y = 丄`的方式进行定义。很容易验证，所要求的规则都是满足的；与列表不一样，甚至将`zero`应用到`丄`上都是满足的。例如，`[x - 1 | x >= 1]_Str`，如果x是正数，将返回比`x`小一的数，反之是`丄`。

##6 Monad morphisms

>单子态射

If `M` and `N` are two monads, then `h :: M x -> N x` is a monad morphism from `M` to `N` if it preserves the monad operations:

>如果`M`和`N`是两个单子，如果`h :: M x -> N x`保持了下面这些单子的操作，那么它是一个从`M`到`N`的单子态射。

```haskell
h * map_M f = map_N f * h
 h * unit_M = unit_N
 h * join_M = join_N * h^2
```

where `h^2 = h * map_M h = map_N h * h` (the two composites are equal by the first equation).

>这儿`h^2 = h * map_M h = map_N h * h`（根据第一个等式，这两个符合是相等的）。

Define the effect of a monad morphism on qualifiers as follows:

>下面定义单子态射在限定词上的影响：

```haskell
     h(A) = A
h(x <- u) = x <- (h u)
  h(p, q) = (h p),(h q)
```

It follows that if `h` is a monad morphism from `M` to `N` then

>如果`h`是一个`M`到`N`的单子态射，那么它遵循：

```haskell
(11) h [t | q]_M = [t | (h q)]_N
```

for all terms `t` and qualifiers `q` . The proof is a simple induction on the form of qualifiers.

>对于所有的语句`t`和限定词`q`。证据就是限定词形式的简单归纳。

As an example, it is easy to check that `unit M :: x -> M x` is a monad morphism from `Id` to `M` . It follows that

>作为例子，很容易检查`unit M :: x -> M x`是一个从`Id`到`M`的单子态射。它遵循：

```haskell
[[t | x <- u]_Id]_M = [t | x <- [u]_M ]_M
```

This explains a trick occasionally used by functional programmers, where one writes the qualifier `x <- [u]` inside a list comprehension to bind `x` to the value of `u` , that is, to achieve the same effect as the qualifier `x <- u` in an Id comprehension.

>这解释了函数式程序员有时候采用的一个技巧，在这儿，在列表推导式中写一个`x <- [u]`的限定词，将`x`绑定到值`u`上，这样可以实现与同一推导式的限定词`x <- u`相同的作用。

As a second example, the function `ro` from Section 4.5 is a monad morphism from `SR` to `ST` . This can be used to prove the equivalence of the two interpreters in Figures 5 and 6. Write `exp_ST :: Exp -> ST Val` and `exp_SR :: Exp -> SR Val` for the versions in the two figures. The equivalence of the two versions is clear if we can show that

>作为第二个例子，在4.5节中，函数`ro`是一个从`SR`到`ST`的单子态射。这可以用来证明图5和6两个解释器等价。在这两个图中写出的`exp_ST :: Exp -> ST Val` 和 `exp_SR :: Exp -> SR Val`两个版本。这两个版本的等价是明显的，如果我们能展示：

```haskell
ro * exp_SR = exp_ST
```

The proof is a simple induction on the structure of expressions. If the expression has the form `(Plus e1 e2 )`, we have that

>简单的对表达式的结果进行归纳可以证明。如果表达式有型如`(Plus e1 e2 )`的形式，我们有：

```haskell
  ro (exp_SR (Plus e1 e2))
= {unfolding exp_SR}
  ro [v1 + v2 | v1 <- exp_SR e1, v2 <- exp_SR e2]_SR
= {by(11)}
  [v1+v2 | v1 <- ro (exp_SR e1), v2 <- ro (exp_SR e2)]_ST
={hypothesis}
  [v1 + v2 | v1 <- exp_ST e1, v2 <- exp_ST e2]_ST
= {folding exp_ST}
  exp_ST (Plus e1 e2)
```

The other two cases are equally simple.

>另外两个案例同样简单。

All of this extends straightforwardly to monads with `zero`. In this case we also require that `h * zero M = zero N` , define the action of a morphism on a filter by `h b = b` , and observe that (11) holds even when `q` contains filters.

>上面所说的可以直接扩展到带`zero`的单子上。在这个案例里面，我们也要求`h * zero M = zero N`，用`h b = b`定义过滤器在态射上的操作，很明显，当`q`包含过滤器的时候规则(11)是满足的。

##7 More monads

>更多的单子

This section describes four more monads: parsers, expressions, input-output, and continuations. The basic techniques are not new (parsers are discussed in [Wad85, Fai87, FL89],and exceptions are discussed in [Wad85, Spi90]), but monads and monad comprehensions provide a convenient framework for their expression.

>本节描述四个单子：解析器，异常，输入-输出，以及后续调用。这些基本的技术并不是新的（解析在[Wad85, Fai87, FL89]中讨论过，异常在[Wad85, Spi90]讨论过），但是单子和单子推导式为他们的表达提供了一个便利的框架。

### 7.1 Parsers

>解析器

The monad of parsers is given by

>解析器单子以下面的形式给出：

```haskell
type Parse x   = String -> List (x, String)
map_Parse f x' = \i -> [(f x, i') | (x, i') <- x' i]_List
unit_Parse x   = \i -> [(x, i)]_List
join_Parse x'' = \i -> [(x, i'') | (x', i') <- x'' i, (x, i'') <- x' i']_List
```

Here `String` is the type of lists of `Char` . Thus, a parser accepts an input string and returns a list of pairs. The list contains one pair for each successful parse, consisting of the value parsed and the remaining unparsed input. An empty list denotes a failure to parse the input. We have that

>这儿`String`的类型是`Char`的列表。因此，一个解析器接受一个输入字符串，并返回一个二元组的列表。这个列表为每一个成功的解析包含一个二元组，二元组由解析的值与剩下未解析的输入部分组成。空的列表表示解析输入失败。我们有

```haskell
[(x,y) | x <- x', y <- y']_Parse = \i -> [((x,y),i'') | (x,i') <- x' i, (y,i'') <- y' i']_List
```

This applies the first parser to the input, binds `x` to the value parsed, then applies the second parser to the remaining input, binds `y` to the value parsed, then returns the pair `(x,y)` as the value together with input yet to be parsed. If either `x'` or `y'` fails to parse its input (returning an empty list) then the combined parser will fail as well.

>这个表达式应用第一个解析器到输入，并将`x`绑定到解析出的值上，然后应用第二个解析器到剩下的输入上，并绑定`y`到解析出的值上，然后返回二元组`(x,y)`作为所有已经解析出的值。如果`x‘`或者`y’`解析他们的输入失败（返回一个空列表），那么组合起来的解析器将也会失败。

There is also a suitable `zero` for this monad, given by

>这个单子也有一个合适的`zero`。如下：

```haskell
zero_Parse y = \i -> []_List
```

Thus, `[]_Parse` is the parser that always fails to parse the input. It follows that we may use filters in Parse-comprehensions as well as in List-comprehensions.

>如此，`[]_Parse`是一个总是解析输入失败的解析器。它遵循我们可能在解析推导式和列表推导式中采用的过滤器的原则。

The alternation operator combines two parsers:

>交替操作符组合两个解析器：

```haskell
(#)     :: Parse x -> Parse x -> Parse x
x' # y' =  \i -> (x' i) ++ (y' i)
```

(Here `++` is the operator that concatenates two lists.) It returns all parses found by the first argument followed by all parses found by the second.

>（这儿`++`是一个连接两个列表的操作）这个操作返回第一个参数的所有解析并将第二个参数的所有解析跟随在它后面。

The simplest parser is one that parses a single character:

>最简单的解析器是解析一个单独的字符：

```haskell
next :: Parse Char
next =  \i -> [(head i, tail i) |not (null i)]_List
```

Here we have a List-comprehension with a filter. The parser `next` succeeds only if the input is non-empty, in which case it returns the next character. Using this, we may define a parser to recognise a literal:

>这儿我们有一个带过滤器的列表推导式。解析器`next`只在输入非空的时候成功，在这种情况下返回下一个字符。采用这个解析器，我们可以定义一个逐字识别的解析器。

```haskell
lit   :: Char -> Parse()
lit c =  [() | c' <- next, c = c']_Parse
```

Now we have a Parse-comprehension with a filter. The parser `lit c` succeeds only if the next character in the input is `c` .

>现在我们有一个带过滤器的解析器推导式。如果下一个字符是`c`，解析器`lit c`返回成功。

As an example, a parser for fully parenthesised lambda terms, yielding values of the type `Term` described previously, can be written as follows:

>作为例子，一个全部加上括号的lambda语句的解析器，产生之前描述过的`Term`类型的值，可以写成：

```haskell
term :: Parse Term
term = [Var x | x <- name]_Parse
  # [ Lam x t | () <- lit '(', () <- lit '\', x <- name, () <- lit '->', t <- term, () <- lit ')']_Parse
  # [ App t u | () <- lit '(', t <- term, u <- term, () <- lit ')']_Parse

name :: Parse Name
name =  [c | c <- next, 'a'<=c, c <='z']_Parse
```

Here, for simplicity, it has been assumed that names consist of a single lower-case letter, so `Name = Char`; and that `\` and `->` are both characters.

>这儿，为了简单起见，要确保名字由单个小写字符构成，因此`Name = Char`；并且`\`和`->`都是一个字符串。

###7.2 Exceptions

>异常

The type `Maybe x` consists of either a value of type `x` , written `Just x` , or an exceptional value, written `Nothing` :

>类型`Maybe x`由类型`x`的值，写成`Just x`，或者是一个异常值，写成`Nothing`构成：

```haskell
data Maybe x = Just x | Nothing
```

(The names are due to Spivey [Spi90].) The following operations yield a monad:

>（这个名字是来自Spivey的[Spi90]）。下面的操作生成一个单子：

```haskell
map_Maybe f (Just x)       = Just (f x)
map_Maybe f Nothing        = Nothing

unit_Maybe x               = Just x

join_Maybe (Just (Just x)) = Just x
join_Maybe (Just Nothing)  = Nothing
join_Maybe Nothing         = Nothing
```

We have that

>我们有

```haskell
[(x,y) | x <- x', y <- y']
```

returns `Just (x,y)` if `x'` is `Just x` and `y'` is `Just y` , and otherwise returns `Nothing` .

>如果`x'`是`Just x`并且`y'`是`Just y`，那么返回`Just (x,y)`，否则返回`Nothing`。

























