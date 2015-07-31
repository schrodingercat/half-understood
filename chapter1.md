Comprehending Monads
====================

**Philip Wadler**

**University of Glasgow**
>     格拉斯哥大学


#####Abstract
>Category theorists invented monads in the 1960's to concisely express certain aspects of universal algebra. Functional programmers invented list comprehensions in the 1970's to concisely express certain programs involving lists. This paper shows how list comprehensions may be generalised to an arbitrary monad, and how the resulting programming feature can concisely express in a pure functional language some programs that manipulate state, handle exceptions, parse text, or invoke continuations. A new solution to the old problem of destructive array update is also presented. No knowledge of category theory is assumed.

>20世纪60年代，Monads出现在范畴理论中，它简洁的表达了泛代数(universal algebra)中的某个概念。20世纪70年代，函数式编程者发展出递推式构造列表(list comprehesions)，它简洁的表达某些构造列表的程序。本文将展示如何可推广递推式构造列表为任意monad，以及怎样组织纯函数式语言去简洁的表达操作状态，处理异常，解析文本，或调用延续等操作。同时，对破坏性数组更新这个老问题提出新的解决放案。本文不涉及任何范畴理论。

##1. Introduction
Is there a way to combine the indulgences of impurity with the blessings of purity?

>如何在坚持纯粹中容忍不纯粹的存在？

Impure, strict functional languages such as Standard ML [Mil84, HMT88] and Scheme [RC86] support a wide variety of features, such as assigning to state, handling exceptions, and invoking continuations. Pure, lazy functional languages such as Haskell [HPW91] or Miranda1 [Tur85] eschew such features, because they are incompatible with the advantages of lazy evaluation and equational reasoning, advantages that have been described at length elsewhere Hug89, BW88].

>不纯的精确性函数式编程语言，如 Standard ML [MIL84，HMT88] 和 Scheme [RC86] ，采用若干诸如状态赋值，异常处理，调用延续等特性来实现。纯的惰性函数式编程语言，如  Haskell [HPW91] 和 Miranda1 [TUR85] 则避开这些特性，因为它们不利于发挥已经被证明的延迟计算和公式推导等特性的优势。

Purity has its regrets, and all programmers in pure functional languages will recall some moment when an impure feature has tempted them. For instance, if a counter is required to generate unique names, then an assignable variable seems just the ticket. In such cases it is always possible to mimic the required impure feature by straightforward though tedious means. For instance, a counter can be simulated by modifying the relevant functions to accept an additional parameter (the counter's current value) and return an additional result (the counter's updated value).

>纯也有它的缺陷，这时所有的纯函数式语言程序员都会记得，当一个不纯的特性吸引他们的时候。例如，如果需要一个具有唯一命名的计数器，那么似乎需要一个可被赋值的变量。在这种情况下，总是会去通过简单的但繁琐的手段去模仿所需的不纯特性。例如，通过修改相关函数来接受一个附加的参数（计数器的当前值），并返回最终结果（计数器的更新值），以这样的方式来模拟一个计数器。

This paper describes a new method for structuring pure programs that mimic impure features. This method does not completely eliminate the tension between purity and impurity, but it does relax it a little bit. It increases the readability of the resulting programs, and it eliminates the possibility of certain silly errors that might otherwise arise (such as accidentally passing the wrong value for the counter parameter).

>本文介绍了一种新的方法来构建纯的程序，以此来模拟不纯的功能。这种方法不能完全消除纯与不纯之间的矛盾，但确实缓解了一点。它增加了程序的可读性，并消除了可能出现一些愚蠢的错误(例如，不小心传递错误的计数器参数）。

The inspiration for this technique comes from the work of Eugenio Moggi [Mog89a,Mog89b]. His goal was to provide a way of structuring the semantic description of features such as state, exceptions, and continuations. His discovery was that the notion of a monad from category theory suits this purpose. By defining an interpretation of lambda-calculus in an arbitrary monad he provided a framework that could describe all these features and more.

>该技术的灵感来自欧金尼奥莫吉(Eugenio Moggi)的工作[MOG89a，MOG89b]。他的目标是提供一种方式来构建如状态，异常，延续，等特征的语义描述。他发现范畴理论的Monad可以实现这个目标。通过在任意一个monad中定义一个可被解释的lambda演算，他提供一个描述不只是实现上面这些特征的框架。

