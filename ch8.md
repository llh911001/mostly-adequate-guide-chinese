# 特百惠

（译者注：特百惠是美国家居用品品牌，代表产品是塑料容器。）

## 强大的容器

<img src="images/jar.jpg" alt="http://blog.dwinegar.com/2011/06/another-jar.html" />

我们已经知道如何书写函数式的程序了，即通过管道把数据在一系列纯函数间传递的程序。我们也知道了，这些程序就是声明式的行为规范。但是，控制流（control flow）、异常处理（error handling）、异步操作（asynchronous actions）和状态（state）呢？还有更棘手的作用（effects）呢？本章将对上述这些抽象概念赖以建立的基础作一番探究。

首先我们将创建一个容器（container）。这个容器必须能够装载任意类型的值；否则的话，像只能装木薯布丁的密封塑料袋是没什么用的。这个容器将会是一个对象，但我们不会为它添加面向对象观念下的属性和方法。是的，我们将把它当作一个百宝箱——一个存放宝贵的数据的特殊盒子。

```js
var Container = function(x) {
  this.__value = x;
}

Container.of = function(x) { return new Container(x); };
```

这是本书的第一个容器，我们贴心地把它命名为 `Container`。我们将使用 `Container.of` 作为构造器（constructor），这样就不用到处去写糟糕的 `new` 关键字了，非常省心。实际上不能这么简单地看待 `of` 函数，但暂时先认为它是把值放到容器里的一种方式。

我们来检验下这个崭新的盒子：

```js
Container.of(3)
//=> Container(3)


Container.of("hotdogs")
//=> Container("hotdogs")


Container.of(Container.of({name: "yoda"}))
//=> Container(Container({name: "yoda" }))
```

如果用的是 node，那么你会看到打印出来的是 `{__value: x}`，而不是实际值 `Container(x)`；Chrome 打印出来的是正确的。不过这并不重要，只要你理解 `Container` 是什么样的就行了。有些环境下，你也可以重写 `inspect` 方法，但我们不打算涉及这方面的知识。在本书中，出于教学和美学上的考虑，我们将把概念性的输出都写成好像 `inspect` 被重写了的样子，因为这样写的教育意义将远远大于 `{__value: x}`。

在继续后面的内容之前，先澄清几点：

* `Container` 是个只有一个属性的对象。尽管容器可以有不止一个的属性，但大多数容器还是只有一个。我们很随意地把 `Container` 的这个属性命名为 `__value`。
* `__value` 不能是某个特定的类型，不然 `Container` 就对不起它这个名字了。
* 数据一旦存放到 `Container`，就会一直待在那儿。我们*可以*用 `.__value` 获取到数据，但这样做有悖初衷。

如果把容器想象成玻璃罐的话，上面这三条陈述的理由就会比较清晰了。但是暂时，请先保持耐心。

## 第一个 functor

一旦容器里有了值，不管这个值是什么，我们就需要一种方法来让别的函数能够操作它。

```js
// (a -> b) -> Container a -> Container b
Container.prototype.map = function(f){
  return Container.of(f(this.__value))
}
```

这个 `map` 跟数组那个著名的 `map` 一样，除了前者的参数是 `Container a` 而后者是 `[a]`。它们的使用方式也几乎一致：

```js
Container.of(2).map(function(two){ return two + 2 })
//=> Container(4)


Container.of("flamethrowers").map(function(s){ return s.toUpperCase() })
//=> Container("FLAMETHROWERS")


Container.of("bombs").map(concat(' away')).map(_.prop('length'))
//=> Container(10)
```

为什么要使用这样一种方法？因为我们能够在不离开 `Container` 的情况下操作容器里面的值。这是非常了不起的一件事情。`Container` 里的值传递给 `map` 函数之后，就可以任我们操作；操作结束后，为了防止意外再把它放回它所属的 `Container`。这样做的结果是，我们能连续地调用 `map`，运行任何我们想运行的函数。甚至还可以改变值的类型，就像上面最后一个例子中那样。

等等，如果我们能一直调用 `map`，那它不就是个组合（composition）么！这里边是有什么数学魔法在起作用？是 *functor*。各位，这个数学魔法就是 *functor*。

> functor 是实现了 `map` 函数并遵守一些特定规则的容器类型。

没错，*functor* 就是一个签了合约的接口。我们本来可以简单地把它称为 `Mappable`，但这样就没有 *fun*（译者注：指 `functor` 中包含 `fun` 这个单词，是一双关语）了，对吧？functor 是范畴学里的概念，我们将在本章末尾详细探索与此相关的数学知识；暂时我们先用这个名字很奇怪的接口做一些不那么理论的、实用性的练习。

把值装进一个容器，而且只能使用 `map` 来处理它，这么做的理由到底是什么呢？如果我们换种方式来问，答案就很明显了：让容器自己去运用函数能给我们带来什么好处？答案是抽象，对于函数运用的抽象。当 `map` 一个函数的时候，我们请求容器来运行这个函数。不夸张地讲，这是一种十分强大的理念。

## 薛定谔的 Maybe

<img src="images/cat.png" alt="cool cat, need reference" />

说实话 `Container` 挺无聊的，而且通常我们称它为 `Identity`，与 `id` 函数的作用相同（这里也是有数学上的联系的，我们会在适当时候加以说明）。除此之外，还有另外一种 functor，那就是实现了 `map` 函数的类似容器的数据类型，这种 functor 在调用 `map` 的时候能够提供非常有用的行为。现在让我们来定义一个这样的 functor。

```js
var Maybe = function(x) {
  this.__value = x;
}

Maybe.of = function(x) {
  return new Maybe(x);
}

Maybe.prototype.isNothing = function() {
  return (this.__value === null || this.__value === undefined);
}

Maybe.prototype.map = function(f) {
  return this.isNothing() ? Maybe.of(null) : Maybe.of(f(this.__value));
}
```

`Maybe` 看起来跟 `Container` 非常类似，但是有一点不同：`Maybe` 会先检查自己的值是否为空，然后才调用传进来的函数。这样我们在使用 `map` 的时候就能避免恼人的空值了（注意这个实现出于教学目的做了简化）。

```js
Maybe.of("Malkovich Malkovich").map(match(/a/ig));
//=> Maybe(['a', 'a'])

Maybe.of(null).map(match(/a/ig));
//=> Maybe(null)

Maybe.of({name: "Boris"}).map(_.prop("age")).map(add(10));
//=> Maybe(null)

Maybe.of({name: "Dinah", age: 14}).map(_.prop("age")).map(add(10));
//=> Maybe(24)
```

注意看，当传给 `map` 的值是 `null` 时，代码并没有爆出错误。这是因为每一次 `Maybe` 要调用函数的时候，都会先检查它自己的值是否为空。

这种点记法（dot notation syntax）已经足够函数式了，但是正如在第 1 部分指出的那样，我们更想保持一种 pointfree 的风格。碰巧的是，`map` 完全有能力以 curry 函数的方式来“代理”任何 functor：

```js
//  map :: Functor f => (a -> b) -> f a -> f b
var map = curry(function(f, any_functor_at_all) {
  return any_functor_at_all.map(f);
});
```

这样我们就可以像平常一样使用组合，同时也能正常使用 `map` 了，非常振奋人心。ramda 的 `map` 也是这样。后面的章节中，我们将在点记法更有教育意义的时候使用点记法，在方便使用 pointfree 模式的时候就用 pointfree。你注意到了么？我在类型标签中偷偷引入了一个额外的标记：`Functor f =>`。这个标记告诉我们 `f` 必须是一个 functor。没什么复杂的，但我觉得有必要提一下。

## 用例

实际当中，`Maybe` 最常用在那些可能会无法成功返回结果的函数中。

```js
//  safeHead :: [a] -> Maybe(a)
var safeHead = function(xs) {
  return Maybe.of(xs[0]);
};

var streetName = compose(map(_.prop('street')), safeHead, _.prop('addresses'));

streetName({addresses: []});
// Maybe(null)

streetName({addresses: [{street: "Shady Ln.", number: 4201}]});
// Maybe("Shady Ln.")
```

`safeHead` 与一般的 `_.head` 类似，但是增加了类型安全保证。引入 `Maybe` 会发生一件非常有意思的事情，那就是我们被迫要与狡猾的 `null` 打交道了。`safeHead` 函数能够诚实地预告它可能的失败——失败真没什么可耻的——然后返回一个 `Maybe` 来通知我们相关信息。实际上不仅仅是*通知*，因为毕竟我们想要的值深藏在 `Maybe` 对象中，而且只能通过 `map` 来操作它。本质上，这是一种由 `safeHead` 强制执行的空值检查。有了这种检查，我们才能在夜里安然入睡，因为我们知道最不受人待见的 `null` 不会突然出现。类似这样的 API 能够把一个像纸糊起来的、脆弱的应用升级为实实在在的、健壮的应用，这样的 API 保证了更加安全的软件。

有时候函数可以明确返回一个 `Maybe(null)` 来表明失败，例如：

```js
//  withdraw :: Number -> Account -> Maybe(Account)
var withdraw = curry(function(amount, account) {
  return account.balance >= amount ?
    Maybe.of({balance: account.balance - amount}) :
    Maybe.of(null);
});

//  finishTransaction :: Account -> String
var finishTransaction = compose(remainingBalance, updateLedger); // <- 假定这两个函数已经在别处定义好了

//  getTwenty :: Account -> Maybe(String)
var getTwenty = compose(map(finishTransaction), withdraw(20));


getTwenty({ balance: 200.00});
// Maybe("Your balance is $180.00")

getTwenty({ balance: 10.00});
// Maybe(null)
```

要是钱不够，`withdraw` 就会对我们嗤之以鼻然后返回一个 `Maybe(null)`。`withdraw` 也显示出了它的多变性，使得我们后续的操作只能用 `map` 来进行。这个例子与前面例子不同的地方在于，这里的 `null` 是有意的。我们不用 `Maybe(String)` ，而是用 `Maybe(null)` 来发送失败的信号，这样程序在收到信号后就能立刻停止执行。这一点很重要：如果 `withdraw` 失败了，`map` 就会切断后续代码的执行，因为它根本就不会运行传递给它的函数，即 `finishTransaction`。这正是预期的效果：如果取款失败，我们并不想更新或者显示账户余额。

## 释放容器里的值

人们经常忽略的一个事实是：任何事物都有个最终尽头。那些会产生作用的函数，不管它们是发送 JSON 数据，还是在屏幕上打印东西，还是更改文件系统，还是别的什么，都要有一个结束。但是我们无法通过 `return` 把输出传递到外部世界，必须要运行这样或那样的函数才能传递出去。关于这一点，可以借用禅宗公案的口吻来叙述：“如果一个程序运行之后没有可观察到的作用，那它到底运行了没有？”。或者，运行之后达到自身的目的了没有？有可能它只是浪费了几个 CPU 周期然后就去睡觉了...

应用程序所做的工作就是获取、更改和保存数据直到不再需要它们，对数据做这些操作的函数有可能被 `map` 调用，这样的话数据就可以不用离开它温暖舒适的容器。讽刺的是，有一种常见的错误就是试图以各种方法删除 `Maybe` 里的值，好像这个不确定的值是魔鬼，删除它就能让它突然显形，然后一切罪恶都会得到宽恕似的（译者注：此处原文应该是源自圣经）。要知道，我们的值没有完成它的使命，很有可能是其他代码分支造成的。我们的代码，就像薛定谔的猫一样，在某个特定的时间点有两种状态，而且应该保持这种状况不变直到最后一个函数为止。这样，哪怕代码有很多逻辑性的分支，也能保证一种线性的工作流。

不过，对容器里的值来说，还是有个逃生口可以出去。也就是说，如果我们想返回一个自定义的值然后还能继续执行后面的代码的话，是可以做到的；要达到这一目的，可以借助一个帮助函数 `maybe`：

```js
//  maybe :: b -> (a -> b) -> Maybe a -> b
var maybe = curry(function(x, f, m) {
  return m.isNothing() ? x : f(m.__value);
});

//  getTwenty :: Account -> String
var getTwenty = compose(
  maybe("You're broke!", finishTransaction), withdraw(20)
);


getTwenty({ balance: 200.00});
// "Your balance is $180.00"

getTwenty({ balance: 10.00});
// "You're broke!"
```

这样就可以要么返回一个静态值（与 `finishTransaction` 返回值的类型一致），要么继续愉快地在没有 `Maybe` 的情况下完成交易。`maybe` 使我们得以避免普通 `map` 那种命令式的 `if/else` 语句：`if(x !== null) { return f(x) }`。

引入 `Maybe` 可能会在初期造成一些不适。Swift 和 Scala 用户知道我在说什么，因为这两门语言的核心库里就有 `Maybe` 的概念，只不过伪装成 `Option(al)` 罢了。被迫在任何情况下都进行空值检查（甚至有些时候我们可以确定某个值不会为空），的确让大部分人头疼不已。然而随着时间推移，空值检查会成为第二本能，说不定你还会感激它提供的安全性呢。不管怎么说，空值检查大多数时候都能防止在代码逻辑上偷工减料，让我们脱离危险。

编写不安全的软件就像用蜡笔小心翼翼地画彩蛋，画完之后把它们扔到大街上一样（译者注：意思是彩蛋非常易于寻找。来源于复活节习俗，人们会藏起一些彩蛋让孩子寻找），或者像用三只小猪警告过的材料盖个养老院一样（译者注：来源于“三只小猪”童话故事）。`Maybe` 能够非常有效地帮助我们增加函数的安全性。

有一点我必须要提及，否则就太不负责任了，那就是 `Maybe` 的“真正”实现会把它分为两种类型：一种是非空值，另一种是空值。这种实现允许我们遵守 `map` 的 parametricity 特性，因此 `null` 和 `undefined` 能够依然被 `map` 调用，functor 里的值所需的那种普遍性条件也能得到满足。所以你会经常看到 `Some(x) / None` 或者 `Just(x) / Nothing` 这样的容器类型在做空值检查，而不是 `Maybe`。

## “纯”错误处理

<img src="images/fists.jpg" alt="pick a hand... need a reference" />

说出来可能会让你震惊，`throw/catch` 并不十分“纯”。当一个错误抛出的时候，我们没有收到返回值，反而是得到了一个警告！抛错的函数吐出一大堆的 0 和 1 作为盾和矛来攻击我们，简直就像是在反击输入值的入侵而进行的一场电子大作战。有了 `Either` 这个新朋友，我们就能以一种比向输入值宣战好得多的方式来处理错误，那就是返回一条非常礼貌的消息作为回应。我们来看一下：

```js
var Left = function(x) {
  this.__value = x;
}

Left.of = function(x) {
  return new Left(x);
}

Left.prototype.map = function(f) {
  return this;
}

var Right = function(x) {
  this.__value = x;
}

Right.of = function(x) {
  return new Right(x);
}

Right.prototype.map = function(f) {
  return Right.of(f(this.__value));
}
```

`Left` 和 `Right` 是我们称之为 `Either` 的抽象类型的两个子类。我略去了创建 `Either` 父类的繁文缛节，因为我们不会用到它的，但你了解一下也没坏处。注意看，这里除了有两个类型，没别的新鲜东西。来看看它们是怎么运行的：

```js
Right.of("rain").map(function(str){ return "b"+str; });
// Right("brain")

Left.of("rain").map(function(str){ return "b"+str; });
// Left("rain")

Right.of({host: 'localhost', port: 80}).map(_.prop('host'));
// Right('localhost')

Left.of("rolls eyes...").map(_.prop("host"));
// Left('rolls eyes...')
```

`Left` 就像是青春期少年那样无视我们要 `map` 它的请求。`Right` 的作用就像是一个 `Container`（也就是 Identity）。这里强大的地方在于，`Left` 有能力在它内部嵌入一个错误消息。

假设有一个可能会失败的函数，就拿根据生日计算年龄来说好了。的确，我们可以用 `Maybe(null)` 来表示失败并把程序引向另一个分支，但是这并没有告诉我们太多信息。很有可能我们想知道失败的原因是什么。用 `Either` 写一个这样的程序看看：

```js
var moment = require('moment');

//  getAge :: Date -> User -> Either(String, Number)
var getAge = curry(function(now, user) {
  var birthdate = moment(user.birthdate, 'YYYY-MM-DD');
  if(!birthdate.isValid()) return Left.of("Birth date could not be parsed");
  return Right.of(now.diff(birthdate, 'years'));
});

getAge(moment(), {birthdate: '2005-12-12'});
// Right(9)

getAge(moment(), {birthdate: 'balloons!'});
// Left("Birth date could not be parsed")
```

这么一来，就像 `Maybe(null)`，当返回一个 `Left` 的时候就直接让程序短路。跟 `Maybe(null)` 不同的是，现在我们对程序为何脱离原先轨道至少有了一点头绪。有一件事要注意，这里返回的是 `Either(String, Number)`，意味着我们这个 `Either` 左边的值是 `String`，右边（译者注：也就是正确的值）的值是 `Number`。这个类型签名不是很正式，因为我们并没有定义一个真正的 `Either` 父类；但我们还是从这个类型那里了解到不少东西。它告诉我们，我们得到的要么是一条错误消息，要么就是正确的年龄值。

```js
//  fortune :: Number -> String
var fortune  = compose(concat("If you survive, you will be "), add(1));

//  zoltar :: User -> Either(String, _)
var zoltar = compose(map(console.log), map(fortune), getAge(moment()));

zoltar({birthdate: '2005-12-12'});
// "If you survive, you will be 10"
// Right(undefined)

zoltar({birthdate: 'balloons!'});
// Left("Birth date could not be parsed")
```

如果 `birthdate` 合法，这个程序就会把它神秘的命运打印在屏幕上让我们见证；如果不合法，我们就会收到一个有着清清楚楚的错误消息的 `Left`，尽管这个消息是稳稳当当地待在它的容器里的。这种行为就像，虽然我们在抛错，但是是以一种平静温和的方式抛错，而不是像一个小孩子那样，有什么不对劲就闹脾气大喊大叫。

在这个例子中，我们根据 `birthdate` 的合法性来控制代码的逻辑分支，同时又让代码进行从右到左的直线运动，而不用爬过各种条件语句的大括号。通常，我们不会把 `console.log` 放到 `zoltar` 函数里，而是在调用 `zoltar` 的时候才 `map` 它，不过本例中，让你看看 `Right` 分支如何与 `Left` 不同也是很有帮助的。我们在 `Right` 分支的类型签名中使用 `_` 表示一个应该忽略的值（在有些浏览器中，你必须要 `console.log.bind(console)` 才能把 `console.log` 当作一等公民使用）。

我想借此机会指出一件你可能没注意到的事：这个例子中，尽管 `fortune` 使用了 `Either`，它对每一个 functor 到底要干什么却是毫不知情的。前面例子中的 `finishTransaction` 也是一样。通俗点来讲，一个函数在调用的时候，如果被 `map` 包裹了，那么它就会从一个非 functor 函数转换为一个 functor 函数。我们把这个过程叫做 *lift*。一般情况下，普通函数更适合操作普通的数据类型而不是容器类型，在必要的时候再通过 *lift* 变为合适的容器去操作容器类型。这样做的好处是能得到更简单、重用性更高的函数，它们能够随需求而变，兼容任意 functor。

`Either` 并不仅仅只对合法性检查这种一般性的错误作用非凡，对一些更严重的、能够中断程序执行的错误比如文件丢失或者 socket 连接断开等，`Either` 同样效果显著。你可以试试把前面例子中的 `Maybe` 替换为 `Either`，看怎么得到更好的反馈。

此刻我忍不住在想，我仅仅是把 `Either` 当作一个错误消息的容器介绍给你！这样的介绍有失偏颇，它的能耐远不止于此。比如，它表示了逻辑或（也就是 `||`）。再比如，它体现了范畴学里 *coproduct* 的概念，当然本书不会涉及这方面的知识，但值得你去深入了解，因为这个概念有很多特性值得利用。还比如，它是标准的 sum type（或者叫不交并集，disjoint union of sets），因为它含有的所有可能的值的总数就是它包含的那两种类型的总数（我知道这么说你听不懂，没关系，这里有一篇[非常棒的文章](https://www.fpcomplete.com/school/to-infinity-and-beyond/pick-of-the-week/sum-types)讲述这个问题）。`Either` 能做的事情多着呢，但是作为一个 functor，我们就用它处理错误。

就像 `Maybe` 可以有个 `maybe` 一样，`Either` 也可以有一个 `either`。两者的用法类似，但 `either` 接受两个函数（而不是一个）和一个静态值为参数。这两个函数的返回值类型一致：

```js
//  either :: (a -> c) -> (b -> c) -> Either a b -> c
var either = curry(function(f, g, e) {
  switch(e.constructor) {
    case Left: return f(e.__value);
    case Right: return g(e.__value);
  }
});

//  zoltar :: User -> _
var zoltar = compose(console.log, either(id, fortune), getAge(moment()));

zoltar({birthdate: '2005-12-12'});
// "If you survive, you will be 10"
// undefined

zoltar({birthdate: 'balloons!'});
// "Birth date could not be parsed"
// undefined
```

终于用了一回那个神秘的 `id` 函数！其实它就是简单地复制了 `Left` 里的错误消息，然后把这个值传给 `console.log` 而已。通过强制在 `getAge` 内部进行错误处理，我们的算命程序更加健壮了。结果就是，要么告诉用户一个残酷的事实并像算命师那样跟他击掌，要么就继续运行程序。好了，现在我们已经准备好去学习一个完全不同类型的 functor 了。

## 王老先生有作用...

（译者注：原标题是“Old McDonald had Effects...”，源于美国儿歌“Old McDonald Had a Farm”。）

<img src="images/dominoes.jpg" alt="dominoes.. need a reference" />

在关于纯函数的的那一章（即第 3 章）里，有一个很奇怪的例子。这个例子中的函数会产生副作用，但是我们通过把它包裹在另一个函数里的方式把它变得看起来像一个纯函数。这里还有一个类似的例子：

```js
//  getFromStorage :: String -> (_ -> String)
var getFromStorage = function(key) {
  return function() {
    return localStorage[key];
  }
}
```

要是我们没把 `getFromStorage` 包在另一个函数里，它的输出值就是不定的，会随外部环境变化而变化。有了这个结实的包裹函数（wrapper），同一个输入就总能返回同一个输出：一个从 `localStorage` 里取出某个特定的元素的函数。就这样（也许再高唱几句赞美圣母的赞歌）我们洗涤了心灵，一切都得到了宽恕。

然而，这并没有多大的用处，你说是不是。就像是你收藏的全新未拆封的玩偶，不能拿出来玩有什么意思。所以要是能有办法进到这个容器里面，拿到它藏在那儿的东西就好了...办法是有的，请看 `IO`：

```js
var IO = function(f) {
  this.__value = f;
}

IO.of = function(x) {
  return new IO(function() {
    return x;
  });
}

IO.prototype.map = function(f) {
  return new IO(_.compose(f, this.__value));
}
```

`IO` 跟之前的 functor 不同的地方在于，它的 `__value` 总是一个函数。不过我们不把它当作一个函数——实现的细节我们最好先不管。这里发生的事情跟我们在 `getFromStorage` 那里看到的一模一样：`IO` 把非纯执行动作（impure action）捕获到包裹函数里，目的是延迟执行这个非纯动作。就这一点而言，我们认为 `IO` 包含的是被包裹的执行动作的返回值，而不是包裹函数本身。这在 `of` 函数里很明显：`IO(function(){ return x })` 仅仅是为了延迟执行，其实我们得到的是 `IO(x)`。

来用用看：

```js
//  io_window_ :: IO Window
var io_window = new IO(function(){ return window; });

io_window.map(function(win){ return win.innerWidth });
// IO(1430)

io_window.map(_.prop('location')).map(_.prop('href')).map(split('/'));
// IO(["http:", "", "localhost:8000", "blog", "posts"])


//  $ :: String -> IO [DOM]
var $ = function(selector) {
  return new IO(function(){ return document.querySelectorAll(selector); });
}

$('#myDiv').map(head).map(function(div){ return div.innerHTML; });
// IO('I am some inner html')
```

这里，`io_window` 是一个真正的 `IO`，我们可以直接对它使用 `map`。至于 `$`，则是一个函数，调用后会返回一个 `IO`。我把这里的返回值都写成了*概念性*的，这样就更加直观；不过实际的返回值是 `{ __value: [Function] }`。当调用 `IO` 的 `map` 的时候，我们把传进来的函数放在了 `map` 函数里的组合的最末端（也就是最左边），反过来这个函数就成为了新的 `IO` 的新 `__value`，并继续下去。传给 `map` 的函数并没有运行，我们只是把它们压到一个“运行栈”的最末端而已，一个函数紧挨着另一个函数，就像小心摆放的多米诺骨牌一样，让人不敢轻易推倒。这种情形很容易叫人联想起“四人帮”（译者注：《设计模式》一书作者）提出的命令模式（command pattern）或者队列（queue）。

花点时间找回你关于 functor 的直觉吧。把实现细节放在一边不管，你应该就能自然而然地对各种各样的容器使用 `map` 了，不管它是多么奇特怪异。这种伪超自然的力量要归功于 functor 的定律，我们将在本章末尾对此作一番探索。无论如何，我们终于可以在不牺牲代码纯粹性的情况下，随意使用这些不纯的值了。

好了，我们已经把野兽关进了笼子。但是，在某一时刻还是要把它放出来。因为对 `IO` 调用 `map` 已经积累了太多不纯的操作，最后再运行它无疑会打破平静。问题是在哪里，什么时候打开笼子的开关？而且有没有可能我们只运行 `IO` 却不让不纯的操作弄脏双手？答案是可以的，只要把责任推到调用者身上就行了。我们的纯代码，尽管阴险狡诈诡计多端，但是却始终保持一副清白无辜的模样，反而是实际运行 `IO` 并产生了作用的调用者，背了黑锅。来看一个具体的例子。

```js

////// 纯代码库: lib/params.js ///////

//  url :: IO String
var url = new IO(function() { return window.location.href; });

//  toPairs =  String -> [[String]]
var toPairs = compose(map(split('=')), split('&'));

//  params :: String -> [[String]]
var params = compose(toPairs, last, split('?'));

//  findParam :: String -> IO Maybe [String]
var findParam = function(key) {
  return map(compose(Maybe.of, filter(compose(eq(key), head)), params), url);
};

////// 非纯调用代码: main.js ///////

// 调用 __value() 来运行它！
findParam("searchTerm").__value();
// Maybe(['searchTerm', 'wafflehouse'])
```

lib/params.js 把 `url` 包裹在一个 `IO` 里，然后把这头野兽传给了调用者；一双手保持的非常干净。你可能也注意到了，我们把容器也“压栈”了，要知道创建一个 `IO(Maybe([x]))` 没有任何不合理的地方。我们这个“栈”有三层 functor（`Array` 是最有资格成为 mappable 的容器类型），令人印象深刻。

有件事困扰我很久了，现在我必须得说出来：`IO` 的 `__value` 并不是它包含的值，也不是像两个下划线暗示那样是一个私有属性。`__value` 是手榴弹的弹栓，只应该被调用者以最公开的方式拉动。为了提醒用户它的变化无常，我们把它重命名为 `unsafePerformIO` 看看。

```js
var IO = function(f) {
  this.unsafePerformIO = f;
}

IO.prototype.map = function(f) {
  return new IO(_.compose(f, this.unsafePerformIO));
}
```

看，这就好多了。现在调用的代码就变成了 `findParam("searchTerm").unsafePerformIO()`，对应用程序的用户（以及本书读者）来说，这简直就直白得不能再直白了。

`IO` 会成为一个忠诚的伴侣，帮助我们驯化那些狂野的非纯操作。下一节我们将学习一种跟 `IO` 在精神上相似，但是用法上又千差万别的类型。


## 异步任务

回调（callback）是通往地狱的狭窄的螺旋阶梯。它们是埃舍尔（译者注：荷兰版画艺术家）设计的控制流。看到一个个嵌套的回调挤在大小括号搭成的架子上，让人不由自主地联想到地牢里的灵薄狱（还能再低点么！）（译者注：灵薄狱即 limbo，基督教中地狱边缘之意）。光是想到这样的回调就让我幽闭恐怖症发作了。不过别担心，处理异步代码，我们有一种更好的方式，它的名字以“F”开头。

这种方式的内部机制过于复杂，复杂得哪怕我唾沫横飞也很难讲清楚。所以我们就直接用 Quildreen Motta 的 [Folktale](http://folktalejs.org/) 里的 `Data.Task` （之前是 `Data.Future`）。来见证一些例子吧：

```js
// Node readfile example:
//=======================

var fs = require('fs');

//  readFile :: String -> Task(Error, JSON)
var readFile = function(filename) {
  return new Task(function(reject, result) {
    fs.readFile(filename, 'utf-8', function(err, data) {
      err ? reject(err) : result(data);
    });
  });
};

readFile("metamorphosis").map(split('\n')).map(head);
// Task("One morning, as Gregor Samsa was waking up from anxious dreams, he discovered that
// in bed he had been changed into a monstrous verminous bug.")


// jQuery getJSON example:
//========================

//  getJSON :: String -> {} -> Task(Error, JSON)
var getJSON = curry(function(url, params) {
  return new Task(function(reject, result) {
    $.getJSON(url, params, result).fail(reject);
  });
});

getJSON('/video', {id: 10}).map(_.prop('title'));
// Task("Family Matters ep 15")

// 传入普通的实际值也没问题
Task.of(3).map(function(three){ return three + 1 });
// Task(4)
```

例子中的 `reject` 和 `result` 函数分别是失败和成功的回调。正如你看到的，我们只是简单地调用 `Task` 的 `map` 函数，就能操作将来的值，好像这个值就在那儿似的。到现在 `map` 对你来说应该不稀奇了。

如果熟悉 promise 的话，你该能认出来 `map` 就是 `then`，`Task` 就是一个 promise。如果不熟悉你也不必气馁，反正我们也不会用它，因为它并不纯；但刚才的类比还是成立的。

与 `IO` 类似，`Task` 在我们给它绿灯之前是不会运行的。事实上，正因为它要等我们的命令，`IO` 实际就被纳入到了 `Task` 名下，代表所有的异步操作——`readFile` 和 `getJSON` 并不需要一个额外的 `IO` 容器来变纯。更重要的是，当我们调用它的 `map` 的时候，`Task` 工作的方式与 `IO` 几无差别：都是把对未来的操作的指示放在一个时间胶囊里，就像家务列表（chore chart）那样——真是一种精密的拖延术。

我们必须调用 `fork` 方法才能运行 `Task`，这种机制与 `unsafePerformIO` 类似。但也有不同，不同之处就像 `fork` 这个名称表明的那样，它会 fork 一个子进程运行它接收到的参数代码，其他部分的执行不受影响，主线程也不会阻塞。当然这种效果也可以用其他一些技术比如线程实现，但这里的这种方法工作起来就像是一个普通的异步调用，而且 event loop 能够不受影响地继续运转。我们来看一下 `fork`：

```js
// Pure application
//=====================
// blogTemplate :: String

//  blogPage :: Posts -> HTML
var blogPage = Handlebars.compile(blogTemplate);

//  renderPage :: Posts -> HTML
var renderPage = compose(blogPage, sortBy('date'));

//  blog :: Params -> Task(Error, HTML)
var blog = compose(map(renderPage), getJSON('/posts'));


// Impure calling code
//=====================
blog({}).fork(
  function(error){ $("#error").html(error.message); },
  function(page){ $("#main").html(page); }
);

$('#spinner').show();
```

调用 `fork` 之后，`Task` 就赶紧跑去找一些文章，渲染到页面上。与此同时，我们在页面上展示一个 spinner，因为 `fork` 不会等收到响应了才执行它后面的代码。最后，我们要么把文章展示在页面上，要么就显示一个出错信息，视 `getJSON` 请求是否成功而定。

花点时间思考下这里的控制流为何是线性的。我们只需要从下读到上，从右读到左就能理解代码，即便这段程序实际上会在执行过程中到处跳来跳去。这种方式使得阅读和理解应用程序的代码比那种要在各种回调和错误处理代码块之间跳跃的方式容易得多。

天哪，你看到了么，`Task` 居然也包含了 `Either`！没办法，为了能处理将来可能出现的错误，它必须得这么做，因为普通的控制流在异步的世界里不适用。这自然是好事一桩，因为它天然地提供了充分的“纯”错误处理。

就算是有了 `Task`，`IO` 和 `Either` 这两个 functor 也照样能派上用场。待我举个简单例子向你说明一种更复杂、更假想的情况，虽然如此，这个例子还是能够说明我的目的。

```js
// Postgres.connect :: Url -> IO DbConnection
// runQuery :: DbConnection -> ResultSet
// readFile :: String -> Task Error String

// Pure application
//=====================

//  dbUrl :: Config -> Either Error Url
var dbUrl = function(c) {
  return (c.uname && c.pass && c.host && c.db)
    ? Right.of("db:pg://"+c.uname+":"+c.pass+"@"+c.host+"5432/"+c.db)
    : Left.of(Error("Invalid config!"));
}

//  connectDb :: Config -> Either Error (IO DbConnection)
var connectDb = compose(map(Postgres.connect), dbUrl);

//  getConfig :: Filename -> Task Error (Either Error (IO DbConnection))
var getConfig = compose(map(compose(connectDB, JSON.parse)), readFile);


// Impure calling code
//=====================
getConfig("db.json").fork(
  logErr("couldn't read file"), either(console.log, map(runQuery))
);
```

这个例子中，我们在 `readFile` 成功的那个代码分支里利用了 `Either` 和 `IO`。`Task` 处理异步读取文件这一操作当中的不“纯”性，但是验证 config 的合法性以及连接数据库则分别使用了 `Either` 和 `IO`。所以你看，我们依然在同步地跟所有事物打交道。

例子我还可以再举一些，但是就到此为止吧。这些概念就像 `map` 一样简单。

实际当中，你很有可能在一个工作流中跑好几个异步任务，但我们还没有完整学习容器的 api 来应对这种情况。不必担心，我们很快就会去学习 monad 之类的概念。不过，在那之前，我们得先检查下所有这些背后的数学知识。


## 一点理论

前面提到，functor 的概念来自于范畴学，并满足一些定律。我们先来探索这些实用的定律。

```js
// identity
map(id) === id;

// composition
compose(map(f), map(g)) === map(compose(f, g));
```

*同一律*很简单，但是也很重要。因为这些定律都是可运行的代码，所以我们完全可以在我们自己的 functor 上试验它们，验证它们是否成立。

```js
var idLaw1 = map(id);
var idLaw2 = id;

idLaw1(Container.of(2));
//=> Container(2)

idLaw2(Container.of(2));
//=> Container(2)
```

看到没，它们是相等的。接下来看一看组合。

```js
var compLaw1 = compose(map(concat(" world")), map(concat(" cruel")));
var compLaw2 = map(compose(concat(" world"), concat(" cruel")));

compLaw1(Container.of("Goodbye"));
//=> Container('Goodbye cruel world')

compLaw2(Container.of("Goodbye"));
//=> Container('Goodbye cruel world')
```

在范畴学中，functor 接受一个范畴的对象和态射（morphism），然后把它们映射（map）到另一个范畴里去。根据定义，这个新范畴一定会有一个单位元（identity），也一定能够组合态射；我们无须验证这一点，前面提到的定律保证这些东西会在映射后得到保留。

可能我们关于范畴的定义还是有点模糊。你可以把范畴想象成一个有着多个对象的网络，对象之间靠态射连接。那么 functor 可以把一个范畴映射到另外一个，而且不会破坏原有的网络。如果一个对象 `a` 属于源范畴 `C`，那么通过 functor `F` 把 `a` 映射到目标范畴 `D` 上之后，就可以使用 `F a` 来指代 `a` 对象（把这些字母拼起来是什么？！）。可能看图会更容易理解：

<img src="images/catmap.png" alt="Categories mapped" />

比如，`Maybe` 就把类型和函数的范畴映射到这样一个范畴：即每个对象都有可能不存在，每个态射都有空值检查的范畴。这个结果在代码中的实现方式是用 `map` 包裹每一个函数，用 functor 包裹每一个类型。这样就能保证每个普通的类型和函数都能在新环境下继续使用组合。从技术上讲，代码中的 functor 实际上是把范畴映射到了一个包含类型和函数的子范畴（sub category）上，使得这些 functor 成为了一种新的特殊的 endofunctor。但出于本书的目的，我们认为它就是一个不同的范畴。

可以用一张图来表示这种态射及其对象的映射：

<img src="images/functormap.png" alt="functor diagram" />

这张图除了能表示态射借助 functor `F` 完成从一个范畴到另一个范畴的映射之外，我们发现它还符合交换律，也就是说，顺着箭头的方向往前，形成的每一个路径都指向同一个结果。不同的路径意味着不同的行为，但最终都会得到同一个数据类型。这种形式化给了我们原则性的方式去思考代码——无须分析和评估每一个单独的场景，只管可以大胆地应用公式即可。来看一个具体的例子。

```js
//  topRoute :: String -> Maybe(String)
var topRoute = compose(Maybe.of, reverse);

//  bottomRoute :: String -> Maybe(String)
var bottomRoute = compose(map(reverse), Maybe.of);


topRoute("hi");
// Maybe("ih")

bottomRoute("hi");
// Maybe("ih")
```

或者看图：

<img src="images/functormapmaybe.png" alt="functor diagram 2" />

根据所有 functor 都有的特性，我们可以立即理解代码，重构代码。

functor 也能嵌套使用：

```js
var nested = Task.of([Right.of("pillows"), Left.of("no sleep for you")]);

map(map(map(toUpperCase)), nested);
// Task([Right("PILLOWS"), Left("no sleep for you")])
```

`nested` 是一个将来的数组，数组的元素有可能是程序抛出的错误。我们使用 `map` 剥开每一层的嵌套，然后对数组的元素调用传递进去的函数。可以看到，这中间没有回调、`if/else` 语句和 `for` 循环，只有一个明确的上下文。的确，我们必须要 `map(map(map(f)))` 才能最终运行函数。不想这么做的话，可以组合 functor。是的，你没听错：

```js
var Compose = function(f_g_x){
  this.getCompose = f_g_x;
}

Compose.prototype.map = function(f){
  return new Compose(map(map(f), this.getCompose));
}

var tmd = Task.of(Maybe.of("Rock over London"))

var ctmd = new Compose(tmd);

map(concat(", rock on, Chicago"), ctmd);
// Compose(Task(Maybe("Rock over London, rock on, Chicago")))

ctmd.getCompose;
// Task(Maybe("Rock over London, rock on, Chicago"))
```

看，只有一个 `map`。functor 组合是符合结合律的，而且之前我们定义的 `Container` 实际上是一个叫 `Identity` 的 functor。identity 和可结合的组合也能产生一个范畴，这个特殊的范畴的对象是其他范畴，态射是 functor。这实在太伤脑筋了，所以我们不会深入这个问题，但是赞叹一下这种模式的结构性含义，或者它的简单的抽象之美也是好的。


## 总结

我们已经认识了几个不同的 functor，但它们的数量其实是无限的。有一些值得注意的可迭代数据类型（iterable data structure）我们没有介绍，像 tree、list、map 和 pair 等，以及所有你能说出来的。eventstream 和 observable 也都是 functor。其他的 functor 可能就是拿来做封装或者仅仅是模拟类型。我们身边到处都有 functor 的身影，本书也将会大量使用它们。

用多个 functor 参数调用一个函数怎么样呢？处理一个由不纯的或者异步的操作组成的有序序列怎么样呢？要应对这个什么都装在盒子里的世界，目前我们工具箱里的工具还不全。下一章，我们将直奔 monad 而去。

[第 9 章: Monad](ch9.md)

## 练习

```js
require('../../support');
var Task = require('data.task');
var _ = require('ramda');

// 练习 1
// ==========
// 使用 _.add(x,y) 和 _.map(f,x) 创建一个能让 functor 里的值增加的函数

var ex1 = undefined



//练习 2
// ==========
// 使用 _.head 获取列表的第一个元素
var xs = Identity.of(['do', 'ray', 'me', 'fa', 'so', 'la', 'ti', 'do']);

var ex2 = undefined



// 练习 3
// ==========
// 使用 safeProp 和 _.head 找到 user 的名字的首字母
var safeProp = _.curry(function (x, o) { return Maybe.of(o[x]); });

var user = { id: 2, name: "Albert" };

var ex3 = undefined


// 练习 4
// ==========
// 使用 Maybe 重写 ex4，不要有 if 语句

var ex4 = function (n) {
  if (n) { return parseInt(n); }
};

var ex4 = undefined



// 练习 5
// ==========
// 写一个函数，先 getPost 获取一篇文章，然后 toUpperCase 让这片文章标题变为大写

// getPost :: Int -> Future({id: Int, title: String})
var getPost = function (i) {
  return new Task(function(rej, res) {
    setTimeout(function(){
      res({id: i, title: 'Love them futures'})
    }, 300)
  });
}

var ex5 = undefined



// 练习 6
// ==========
// 写一个函数，使用 checkActive() 和 showWelcome() 分别允许访问或返回错误

var showWelcome = _.compose(_.add( "Welcome "), _.prop('name'))

var checkActive = function(user) {
 return user.active ? Right.of(user) : Left.of('Your account is not active')
}

var ex6 = undefined



// 练习 7
// ==========
// 写一个验证函数，检查参数是否 length > 3。如果是就返回 Right(x)，否则就返回
// Left("You need > 3")

var ex7 = function(x) {
  return undefined // <--- write me. (don't be pointfree)
}



// 练习 8
// ==========
// 使用练习 7 的 ex7 和 Either 构造一个 functor，如果一个 user 合法就保存它，否则
// 返回错误消息。别忘了 either 的两个参数必须返回同一类型的数据。

var save = function(x){
  return new IO(function(){
    console.log("SAVED USER!");
    return x + '-saved';
  });
}

var ex8 = undefined
```
