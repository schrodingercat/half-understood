Monads for the Working Haskell Programmer 
=========================================

>Haskell程序员的单子

-- a short tutorial.
--------------------

>小教程

* Theodore Norvell
* Memorial University of Newfoundland

>加拿大纽芬兰大学留念

New: Updated to Haskell '98

>最新：适合Haskell '98

This short tutorial introduces monads to Haskell programmers. It applies to Haskell '98. Gofer programmers (are there any left?) can read it too; there is a section at the end detailing the differences between Haskell and Gofer around monads and another about the differences between Haskell 1.3, Haskell 1.4, and Haskell 98.

>这篇小教程为Haskell程序员介绍单子。采用Haskell '98版本。Gofer程序员(还有吗？)也可以阅读。在文章最后，有一节会列出Haskell和Gofer在单子上的不同，另一节会是关于Haskell 1.3，Haskell 1.4以及Haskell 98之间的不同。

The reader is assumed to have a familiarity with the basics of Haskell programming such as data declarations, function definitions, and lambda expressions. Familiarity with classes and instance declarations will help.

>这篇文章假定读者熟悉一些基本的Haskell编程知识，比如数据声明，函数声明，以及lambda表达式。熟悉类和实例声明会更好。

###Simulating states without Monads

>模拟状态，不使用单子

Consider the following function

>考虑下面的函数：

```haskell
f1 w a = let(b,x) = g1 w a
            (c,y) = h1 w x b
            (d,z) = i1 w x y c
         in (d,z)
```

where `a`, `b`, `c`, and `d` all have the same type, State.   In a sense this definition is very similar to an imperative program in which a represents the initial state, d represents the final state, and b and c represent intermediate states. The functions f1 w, g1 w, h1 w x, and i1 w x y can be thought of as imperative routines that transform the state and produce results. The results are like the return values in C programs and are bound to variables x, y, and z in the example. A C routine that corresponds to this would look something like this:

>在这儿，`a`， `b`， `c` 和 `d` 有相同的`State` 类型。在某种意义上，这个定义非常类似在命令式程序中，`a`表示的初始状态，`d`表示最终状态，而`b`和`c`表示中间状态。函数`f1 w`，`g1 w`，和`h1 w x`以及`il w x y`可以被认为是将状态转换成结果的命令式例程。而转换的结果可以看成是C程序中的返回值，并且被绑定到`x`，`y`，`z`上。下面这个C例程与上面的例子看起来有点像：

```haskell
int f1(float w) {
    const char   x = g1(w) ;
    const double y = h1(w, x) ;
    const int    z = i1(w, x, y) ;
    return z ; }
```

All this explicit passing of states around can get a bit tedious, especially when programs are modified. It also makes the program hard to read vs. the equivalent C program.

>上面这样到处显示的传递状态看起来有点冗长，特别是需要修改程序的时候。相对于等效的C程序，这么写使程序难以读懂。

Note that if we ignore the w, x, y, and z, then g1, h1, and i1 are being composed. Can we define a kind of "composition operator" that allows us to deal with the returned values as well as the state?

>注意，如果我们忽略`w`，`x`，`y`和`z`，那么程序只由`g1`,`h1`以及`i1`组成。我们能定义某种“组合操作”来允许我们处理返回值以及状态？

###Simulating states with Monads

>采用单子模拟状态

The functions that transform the state in the example above all have the type State -> (State, a) for some result type a. Let's generalize over the type of the state and create a type to represent these transforms

>上面例子中的状态转换函数都具有`State -> (State,a)`的类型，类型`a`是返回值。让我们将状态类型一般化，并创造一种表示转换的类型：

```haskell
type StateTrans s a = s -> (s, a)
```

Let us define two functions. The first is a kind of composition function.

>让我们定义两个函数。第一个是某种组合函数：

```haskell
(>>=) :: StateTrans s a ->
         (a -> StateTrans s b) ->
         StateTrans s b

p>>=k  =  \s0 -> let (s1, a) = p s0
                     q = k a
                 in q s1
```

The second is a function that turns a value into an "imperative program" that does not change the state, but returns the result.

>第二个函数将一个值转为不改变状态并且返回这个值的“命令式程序”，

```haskell
return :: a -> StateTrans s a
return a = \s -> (s, a)
```

Our original function may now be written with all the shunting of state around hidden by these operators:

>我们开始的那个程序现在可以依靠这些操作将所有的状态分离并隐藏起来：

```haskell
f w = (g w) >>= (\x -> (h w x) >>= (\y -> (i w x y) >>= (\z -> return z)))
```

You could also format this as follows

>你也可以按照下面的方式格式化

```haskell
f w = g w >>= \x ->
      h w x >>= \y ->
      i w x y >>= \z ->
      return z
```

You should verify that `f` is equal to `f1`, if `g = g1`, `h = h1`, and `i = i1`.

>你应该验证，如果`g = g1`，`h = h1`， 并且 ` i = i1 `，那么`f`和`f1`相等。

(Warning, if you really try to make the above definitions of `>>=` and `return`, your Haskell compiler will complain, for reasons that will be explained later. However, you could try changing the names a little, e.g. change `>>=` to `>==` and change `return` to `rtn`.)

>(注意，如果你真的试图构建上面`>>=`和`return`定义的函数，你的Haskell编译器将会抗议，具体原因以后会解释。然而，你可以尝试稍微修改一下名字，比如用`>==`代替`>>=`以及用`rtn`代替`return`)

###A nicer syntax

>更优雅的语法

Now Haskell provides a nice syntax to sugar-coat the calls to `>>=`. It turns out that the definition of `f` can also be written as

>现在Haskell提供一种优雅的语糖来调用`>>=`。采用这种语法糖，`f`的定义也可以写成：

```haskell
f w = do x <- g w
         y <- h w x
         z <- i w x y
         return z
```

The "do" is a keyword of Haskell. Any program involving a "do" can be translated to one that doesn't use "do" by the following 4 rules:

>语句“do”是Haskell中的关键词。根据下面4种规则，任何“do”中的程序可以被翻译成一个不使用“do”的程序：

1 An expression

>1 一个表达式

```haskell
do	pattern <- expression
morelines
becomes expression >>= (\ pattern -> do morelines)
```

2 An expression

>2 一个表达式

```haskell
do	expression
morelines
becomes  expression >> do morelines
```

3 An expression

>3 一个表达式

```haskell
do	let declarationList
morelines
becomes

let	declarationList
in	do morelines
```
4 Finally when there is only one line in the do, an expression

>4 一个表达式，但只有一行。

```haskell
do	expression
becomes expression
```

In the second rule, we saw the `>>` operator; this is used when the result of the first operand is not subsequently used. It can always be defined as

>在第二个规则中，我们看见`>>`操作；这是当第一个操作的结果没有在后面用的时候使用的。这个操作总是被定义成：

```haskell
p >> q  =  p >>= (\ _ -> q)
```

So

>因此

```haskell
do p
       q
```

is the same as

>和下面是一回事：

```haskell
    do _ <- p
       q
```

The "do" notation follows the same conventions as other multi-line constructs in Haskell. All lines must be vertically aligned. Alternatively, you may use braces and semicolons if you prefer "free format":

>do语法遵循其他Haskell中的多行构造习惯。所有的行都必须是对齐的。或者，你如果喜欢“自由格式”，就需要使用大括号和分号。

```haskell
    do {x <- g w ; y <- h w x ;
          z <- i w x y ; return z }
```

###How to Really Declare the StateTrans Monad.

>怎样正式声明StateTrans单子

So far we've seen the monad concept applied to implicitly passing state. However monads can be used to implement several other programming features including: consuming input, producing output, exceptions and exception handling, nondeterminisim. To handle this variety, the basic operations of `>>=` and `return` are overloaded functions.

>至今为止，我们见到隐式传递状态的单子概念。然而单子可以用来实现几个其他的程序特性：输入消费，输出处理，异常以及异常捕捉，非确定性。要处理这些特性，需要重载`>>=`和`return`这样的基本操作。

(Haskell requires that overloaded functions be declared in class declarations and defined in instance declarations. This is the reason that `>>=` and `return` can not be defined exactly as described in previous sections.)

>(Haskell要求，需要重载的函数必须在类中声明并在实例声明中定义。这就是`>>=`和`return`确实不能按前面章节那样定义的原因。)

The class that `>>=` and `return`  are declared in  is called Monad and  is itself declared in the Haskell Prelude. That declaration looks like this

>`>>=`和`return`被声明在一个被称之为Monad的类中，并且是由Haskell自己预定义的。这些声明看起来像这样：

```haskell
class  Monad m  where
    (>>=)   :: m a -> (a -> m b) -> m b
    (>>)    :: m a -> m b -> m b
    return  :: a -> m a
    fail    :: String -> m a

    m >> k  =  m >>= \_ -> k
    fail s  = error s
```

To make a state transform monad that actually works with Haskell's "do" notation, we have to declare `>>=` and `return` within an instance declaration.  There is one minor problem; type synonyms can not be used in instance declarations, so we define `StateTrans s a` as a new type.

>为了构造确切能在Haskell的“do”语句中工作的状态转换单子，我们必须在实例中声明`>>=`和`return`。有一个小问题；类型同义词不能在实例声明中使用，因此我们定义`StateTrans s a`作为一个新类型。

```haskell
newtype StateTrans s a = ST( s -> (s, a) )
```

and we define the functions `return` and `>>=` for this type as follows:

>并且我们为这种类型定义`return`和`>>=`函数，就像下面：

```haskell
instance Monad (StateTrans s)
  where
    -- (>>=) :: StateTrans s a -> (a -> StateTrans s b) -> StateTrans s b
    (ST p) >>= k  =  ST( \s0 -> let (s1, a) = p s0
                                    (ST q) = k a
                                in q s1 )
     			   	
    -- return :: a -> StateTrans s a
    return a = ST( \s -> (s, a) )
```

We don't need to define the `>>` operator; it is defined automatically in terms of `>>=`.

>我们不需要定义`>>`操作；Haskell将会采用`>>=`语句自动进行定义。

Note these new definitions change the type of functions like our example function `f`.  Indeed `f` is no longer equal to `f1`, since its type is different. To apply members of the monad, we define a new function

>注意这些新的定义会改变像我们例子中的函数`f`的类型。既然类型都不同了，`f`就确实不再与`f1`相等了。应用monad的成员，我们定义一个新函数：

```haskell
applyST :: StateTrans s a -> s -> (s, a)
applyST (ST p) s = p s
```

Now `applyST f` is equal to `f1`.

>现在`applyST f`就与`f1`相等了。

```haskell
译者猜想：
return = unit
\x' -> (x' >>= (return*f)) = map f
 \x'' -> (x'' >>= (\x'->x')) = join

{ f x | x <- x' } = map f x' = x' >>= return f
{ f y x | x <- x',y <- y'} = join (map (\x -> (map(f x) y')) x') = x'>>= (\x-> y'>>= return*f x)

因此

\f -> (\x'->(x' >>= return*f)) <=> map
```

###Which is the Monad?

>何为单子？

Technically each Monad (in Haskell) is not a type, but a "type contructor". In our state transformer example, we have actually declared an infinite set of monads. For each type `s`, there is a type constructor StateTrans s. Each such type constructor has been declared to be a Monad by the instance declaration above.

>从技术上讲，每一个单子（在Haskell中）都不是类型，而是一个“类型构造器”。在我们状态转换器的例子中，我们确实声明了一个单子的无限集合。对于每一个类型`s`，有一个类型构造器`StateTrans s`。每一个这样的类型构造器都已经被声明成一个可以通过像上面那样的实例声明的单子。

###An example

>一个例子

A short example shows how the StateTrans monad lets you code in a fairly imperative style.

>一个简短的例子展示StateTrans单子怎样让你的代码变为清晰的命令式风格。

We will implement a variation on Euclid's algorithm for finding the greatest common divisor of two positive integers.

>我们将实现一个寻找两个正整数的最大公约数的欧几里德算法的变种。

```haskell
     while x != y do
          if x < y
          then y := y-x
          else x := x-y
     return x
```

First we must define a type to represent the state:

>一开始，我们必须定义一个类型来表示状态：

```haskell
type ImpState = (Int, Int)
```

Next we define some simple state transformers to access and change the state. We use the type () and its sole value, (), when a state transformer does not return a useful value.

>然后，我们定义一些简单的状态转换器来访问和修改状态。当一个状态转换器需要返回一个无意义的值时，我们使用类型`()`和它的唯一的值`()`。

```haskell
getX, getY :: StateTrans ImpState Int
getX = ST(\(x,y)-> ((x,y), x))
getY = ST(\(x,y)-> ((x,y), y))

putX, putY :: Int -> StateTrans ImpState ()
putX x' = ST(\(x,y)->((x',y),()))
putY y' = ST(\(x,y)->((x,y'),()))
```

Now we can write the algorithm with the state squarely in the background:

>现在我们可以写一个算法,并将状态直接隐藏起来：

```haskell
gcdST :: StateTrans ImpState Int
gcdST = do x <- getX
           y <- getY
           (if x == y
            then
                 return x
            else if x < y
            then 
                 do putY (y-x)
                    gcdST
            else
                 do putX (x-y)
                    gcdST)
```

And finally a function to construct an initial state, run the program and discard the final state

>而最后，用一个函数来构造初始状态，并运行程序和丢弃最终状态。

```haskell
greatestCommonDivisor x y = snd( applyST gcdST (x,y) )
```

This small example only hints at the utility of monads. It would be much shorter to write the algorithm in a conventional functional style. The savings from not having to explicitly pass the state around become larger as the program becomes larger.

>这个小例子只是展示了单子的一些用途。它写的算法比传统函数式编程风格短得多。省下来的是随着程序增大而增加的到处显示的传递状态的代码。

###Monads, Monoids, and Identities

>单子，幺半群，恒等式

The term "monad" comes from category theory, which is a branch of algebra (or, depending on whom you talk to, algebra is a branch of category theory). There is no need to understand the algebra at all to understand the use of monads in functional programming. This section and the next, just touch on the algebra a little bit.

>术语“单子”来自范畴论，范畴论是一个代数的分支（或者，根据不同人的解释，可能得到的答案是代数是范畴论的分支）。对于理解单子在函数式编程的应用，你不需要懂任何代数的知识。从本节开始，只会稍微接触一点儿代数知识。

Monads share some similarity with monoids , with `>>=` being similar to the monoid operation and the `return` operator taking the place of the identity member. Specifically we have the following identities

>单子与幺半群类似，操作`>>=`类似积操作并且操作`return`代替一元。尤其是我们有下面的恒等式：

```haskell
return a >>= k   =   k a
  p >>= return   =   p
(p >>= j) >= k   =   p >>= (\x->(j x >>= k))      [provided x is not free in j or k]
```

```haskell
译者猜想：
  j1*j2 = (\a -> j1 a>>=j2)
  (j1*j2)*j3 = (\b -> (\a -> j1 a >>= j2) b >>= j3)
  j1*(j2*j3) = (\a -> j1 a >>= (\b -> j2 b >>= j3))
  return * j = j * return = j

```

These identities hold in the StateTrans monad and ought to hold for any monad we define.

>StateTrans单子满足这些恒等式，并且我们定义的任何单子都要满足。

###Zeros and addition

>零和加法

Monoids sometimes also have a zero member and an addition operator.

>幺半群有时候也有零元和加法操作。

In Haskell, the zero member of a monad is called mzero and the relevant identity is

>在Haskell中，单子的零元被称为`mzero`，并且有关的恒等式是：

```haskell
mzero >>= k   =  mzero
```

```haskell
译者注：
   为什么我觉得应该是:
      \x -> mzero x >>= k = \x -> mzero x
   即：
      mzero*k=mzero
      k*mzero=mzero
```

In a state transformation monad, mzero might represent an exception. I.e. an indication that the task can not be completed.

>在状态转换单子中，`mzero`可能代表一个异常。即一个操作无法被完成的征兆。

When a Monad has a zero member, it often also has an addition operation, mplus, obeying the following additional identities

>当单子具有零元的时候，它常常具有加法操作，`mplus`，服从下面的加法恒等式：

```haskell
p `mplus` mzero = p = mzero `mplus` p
p `mplus` (q `mplus` r)  =  (p `mplus` q) `mplus` r
```

It turns out that monads with zeros and addition are common enough that there is a library class, called MonadPlus, defined to encompass them.

>结果是拥有零元和加法的单子足够普遍，以至于有一个类库，称之为`MonadPlus`，在其中定义了这些。

If `mzero` represents failure to complete a computation, `mplus` might represent a way of combining alternative computations such that, if one fails, the other can succeed instead.

>如果mzero表示计算失败，`mplus`可能表示二选一的组合计算，如果一个失败，那么另一个可以成功取代它。


























