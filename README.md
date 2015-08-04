<img src="images/cover.png"/>

# 关于本书

这本书的主题是函数范式（Functional paradigm），我们将使用世界上最流行的函数式编程语言 JavaScript 来讲述这一主题。可能一些人会觉得选择 JavaScript 有违当前对这门语言的主流看法：它主要是命令式（Imperative）的。但我认为，这是学习函数式编程的最好方式，因为：

 * 你很有可能在日常工作中使用它

    这让你有机会在实际的编程过程中学以致用，而不是在空闲时间使用一门晦涩的函数式编程语言去做一些玩具性质的项目。

 * 你不必从头学起就能开始编写程序

    在纯函数式编程语言中，你必须使用 monad 才能打印变量或者读取 DOM 节点。而 JavaScript 就容易得多，可以耍点小聪明，因为毕竟我们要学习的是如何编写纯函数式的代码。JavaScript 也更容易入门，因为它是一门混合范式的语言，你随时可以在感觉吃力的时候退回到原有的编程习惯上去。

 * 这门语言完全有能力书写高级的函数式代码

    只需借助一到两个微型类库就能提供模拟类似 Scala 或 Haskell 这类语言所需的全部特性。虽然面向对象编程（Object-oriented programing）思想主导着业界，但很明显 OOP 在 JavaScript 里非常古怪：就像在高速公路上露营或者穿着橡胶套鞋跳踢踏舞。我们不得不到处使用 `bind` 以防 `this` 不知不觉地变了，这门语言里没有类[^目前还没有]，我们还发明了各种各样的变通方法来应对忘记调用 `new` 关键字的情况，私有成员只有通过闭包（Closure）才能实现，等等。对大多数人来说，函数式编程看起来更加自然。

以上说明，强类型的函数式语言毫无疑问将会成为本书所示范式的最佳试验场。JavaScript 是我们学习这种范式的一种手段，你将它应用于什么地方完全取决于你自己。幸运的是，所有的接口都是数学的，因而也是普适的。最终你会发现你习惯了 swiftz、scalaz、haskell 和 purescript，以及其他各种数学偏向的语言。

# 目录

## 第 1 部分

* [第 1 章: 我们在做什么？](ch1.md)
  * [介绍](ch1.md#介绍)
  * [一个简单例子](ch1.md#一个简单例子)
* [第 2 章: 一等公民的函数](ch2.md)
  * [快速概览](ch2.md#快速概览)
  * [为何钟爱一等公民](ch2.md#为何钟爱一等公民)
* [第 3 章: 纯函数的好处](ch3.md)
  * [再次强调“纯”](ch3.md#再次强调“纯”)
  * [副作用可能包括...](ch3.md#副作用可能包括)
  * [八年级数学](ch3.md#八年级数学)
  * [追求“纯”的理由](ch3.md#追求“纯”的理由)
  * [总结](ch3.md#总结)
* [第 4 章: 柯里化（curry）](ch4.md)
  * [不可或缺的 curry](ch4.md#不可或缺的-curry)
  * [不仅仅是双关语／咖喱](ch4.md#不仅仅是双关语咖喱)
  * [总结](ch4.md#总结)
* [第 5 章: 代码组合（compose）](ch5.md)
  * [函数饲养](ch5.md#函数饲养)
  * [pointfree](ch5.md#pointfree)
  * [debug](ch5.md#debug)
  * [范畴学](ch5.md#范畴学)
  * [总结](ch5.md#总结)
* [第 6章: 示例应用](ch6.md)
  * [声明式代码](ch6.md#声明式代码)
  * [一个函数式的 flickr](ch6.md#一个函数式的-flickr)
  * [有原则的重构](ch6.md#有原则的重构)
  * [总结](ch6.md#总结)

## 第 2 部分

* [第 7 章: Hindley-Milner 类型签名](ch7.md)
  * [初识类型](ch7.md#初识类型)
  * [神秘的传奇故事](ch7.md#神秘的传奇故事)
  * [缩小可能性范围](ch7.md#缩小可能性范围)
  * [自由定理](ch7.md#自由定理)
  * [总结](ch7.md#总结)
* [第 8 章: 特百惠](ch8.md)
  * [强大的容器](ch8.md#强大的容器)
  * [第一个 functor](ch8.md#第一个-functor)
  * [薛定谔的 Maybe](ch8.md#薛定谔的-maybe)
  * [“纯”错误处理](ch8.md#“纯”错误处理)
  * [王老先生有作用...](ch8.md#王老先生有作用)
  * [异步任务](ch8.md#异步任务)
  * [一点理论](ch8.md#一点理论)
  * [总结](ch8.md#总结)
* [Chapter 9: Monadic Onions](ch9.md)
  * [Pointy Functor Factory](ch9.md#pointy-functor-factory)
  * [Mixing Metaphors](ch9.md#mixing-metaphors)
  * [My chain hits my chest](ch9.md#my-chain-hits-my-chest)
  * [Theory](ch9.md#theory)
  * [In Summary](ch9.md#in-summary)



# 未来计划

* 第 1 部分是基础知识。这是初版草稿，所以我会及时更正发现的的错误。欢迎提供帮助！
* 第 2 部分讲述类型类（type class），比如 functor 和 monad，最后会讲到到 traversable。我希望能塞进来一些 monad transformer 相关的知识，再写一个纯函数的应用。
* 第 3 部分将开始游走于编程实践与学院学究之间。我们将学习 comonad、f-algebra、free monad、yoneda 以及其他一些范畴学概念。

