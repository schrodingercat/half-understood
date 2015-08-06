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

>>(i)           map id = id,
>>(ii)     map (g * f) = (map g) * (map f)
>>(iii) (map f) * unit = unit * f
>>(iv)  (map f) * join = join * (map (map f))

>>(I)         join * unit = id
>>(II)  join * (map unit) = id
>>(III)         join*join = join * (map join)

Every monad gives rise to a notion of comprehension via laws (1)-(3). The three laws establish a correspondence between the three components of a monad and the three forms of qualifier: (1) associates `unit` with the empty qualifier, (2) associates `map` with generators, and (3) associates `join` with qualifier composition. The resulting notion of comprehension is guaranteed to be sensible in that it necessarily satisfies laws (4)-(6) and (I)-(III).

>每一个单子都通过规则(1)-(3)产生推导的概念。这三条规则建立起单子三个部件与三种限定词的之间的联系：(1)联系`unit`和空限定词，（2）联系`map`和生成器，（3）联系`join`和符合限定词。结果是，推导的意图显然被明智的限制于必须满足规则(4)-(6)和(I)-(III)。

>>(1) [t|A] = unit t
>>(2) [t|x<-u] = map(\x -> t) u
>>(3) [t|(p,q)] = join [[t|q]|p]

>>(4) [f t| q]           =(map f)[t|q]
>>(5) [x| x<-u]          = u
>>(6) [t|p, x<-[u|q], r] = [t_x_u|p, q, r_x_u].

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


















