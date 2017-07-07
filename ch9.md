# Monad

## pointed functor

在继续后面的内容之前，我得向你坦白一件事：关于我们先前创建的容器类型上的 `of` 方法，我并没有说出它的全部实情。真实情况是，`of` 方法不是用来避免使用 `new` 关键字的，而是用来把值放到*默认最小化上下文*（default minimal context）中的。是的，`of` 没有真正地取代构造器——它是一个我们称之为 *pointed* 的重要接口的一部分。

> *pointed functor* 是实现了 `of` 方法的 functor。

这里的关键是把任意值丢到容器里然后开始到处使用 `map` 的能力。

```js
IO.of("tetris").map(concat(" master"));
// IO("tetris master")

Maybe.of(1336).map(add(1));
// Maybe(1337)

Task.of([{id: 2}, {id: 3}]).map(_.prop('id'));
// Task([2,3])

Either.of("The past, present and future walk into a bar...").map(
  concat("it was tense.")
);
// Right("The past, present and future walk into a bar...it was tense.")
```

如果你还记得，`IO` 和 `Task` 的构造器接受一个函数作为参数，而 `Maybe` 和 `Either` 的构造器可以接受任意值。实现这种接口的动机是，我们希望能有一种通用、一致的方式往 functor 里填值，而且中间不会涉及到复杂性，也不会涉及到对构造器的特定要求。“默认最小化上下文”这个术语可能不够精确，但是却很好地传达了这种理念：我们希望容器类型里的任意值都能发生 `lift`，然后像所有的 functor 那样再 `map` 出去。

有件很重要的事我必须得在这里纠正，那就是，`Left.of` 没有任何道理可言，包括它的双关语也是。每个 functor 都要有一种把值放进去的方式，对 `Either` 来说，它的方式就是 `new Right(x)`。我们为 `Right` 定义 `of` 的原因是，如果一个类型容器*可以* `map`，那它就*应该* `map`。看上面的例子，你应该会对 `of` 通常的工作模式有一个直观的印象，而 `Left` 破坏了这种模式。

你可能已经听说过 `pure`、`point`、`unit` 和 `return` 之类的函数了，它们都是 `of` 这个史上最神秘函数的不同名称（译者注：此处原文是“international function of mystery”，源自恶搞《007》的电影 *Austin Powers: International Man of Mystery*，中译名《王牌大贱谍》）。`of` 将在我们开始使用 monad 的时候显示其重要性，因为后面你会看到，手动把值放回容器是我们自己的责任。

要避免 `new` 关键字，可以借助一些标准的 JavaScript 技巧或者类库达到目的。所以从这里开始，我们就利用这些技巧或类库，像一个负责任的成年人那样使用 `of`。我推荐使用 `folktale`、`ramda` 或 `fantasy-land` 里的 functor 实例，因为它们同时提供了正确的 `of` 方法和不依赖 `new` 的构造器。

## 混合比喻

<img src="images/onion.png" alt="http://www.organicchemistry.com/wp-content/uploads/BPOCchapter6-6htm-41.png" />

你看，除了太空墨西哥卷（如果你听说过这个传言的话）（译者注：此处的传言似乎是说一个叫 Chris Hadfield 的宇航员在国际空间站做墨西哥卷的事，[视频链接](https://www.youtube.com/watch?v=f8-UKqGZ_hs)），monad 还被喻为洋葱。让我以一个常见的场景来说明这点：

```js
// Support
// ===========================
var fs = require('fs');

//  readFile :: String -> IO String
var readFile = function(filename) {
  return new IO(function() {
    return fs.readFileSync(filename, 'utf-8');
  });
};

//  print :: String -> IO String
var print = function(x) {
  return new IO(function() {
    console.log(x);
    return x;
  });
}

// Example
// ===========================
//  cat :: IO (IO String)
var cat = compose(map(print), readFile);

cat(".git/config")
// IO(IO("[core]\nrepositoryformatversion = 0\n"))
```

这里我们得到的是一个 `IO`，只不过它陷进了另一个 `IO`。要想使用它，我们必须这样调用： `map(map(f))`；要想观察它的作用，必须这样： `unsafePerformIO().unsafePerformIO()`。

```js
//  cat :: String -> IO (IO String)
var cat = compose(map(print), readFile);

//  catFirstChar :: String -> IO (IO String)
var catFirstChar = compose(map(map(head)), cat);

catFirstChar(".git/config")
// IO(IO("["))
```

尽管在应用中把这两个作用打包在一起没什么不好的，但总感觉像是在穿着两套防护服工作，结果就形成一个稀奇古怪的 API。再来看另一种情况：

```js
//  safeProp :: Key -> {Key: a} -> Maybe a
var safeProp = curry(function(x, obj) {
  return new Maybe(obj[x]);
});

//  safeHead :: [a] -> Maybe a
var safeHead = safeProp(0);

//  firstAddressStreet :: User -> Maybe (Maybe (Maybe Street) )
var firstAddressStreet = compose(
  map(map(safeProp('street'))), map(safeHead), safeProp('addresses')
);

firstAddressStreet(
  {addresses: [{street: {name: 'Mulburry', number: 8402}, postcode: "WC2N" }]}
);
// Maybe(Maybe(Maybe({name: 'Mulburry', number: 8402})))
```

这里的 functor 同样是嵌套的，函数中三个可能的失败都用了 `Maybe` 做预防也很干净整洁，但是要让最后的调用者调用三次 `map` 才能取到值未免也太无礼了点——我们和它才刚刚见面而已。这种嵌套 functor 的模式会时不时地出现，而且是 monad 的主要使用场景。

我说过 monad 像洋葱，那是因为当我们用 `map` 剥开嵌套的 functor 以获取它里面的值的时候，就像剥洋葱一样让人忍不住想哭。不过，我们可以擦干眼泪，做个深呼吸，然后使用一个叫作 `join` 的方法。

```js
var mmo = Maybe.of(Maybe.of("nunchucks"));
// Maybe(Maybe("nunchucks"))

mmo.join();
// Maybe("nunchucks")

var ioio = IO.of(IO.of("pizza"));
// IO(IO("pizza"))

ioio.join()
// IO("pizza")

var ttt = Task.of(Task.of(Task.of("sewers")));
// Task(Task(Task("sewers")));

ttt.join()
// Task(Task("sewers"))
```

如果有两层相同类型的嵌套，那么就可以用 `join` 把它们压扁到一块去。这种结合的能力，functor 之间的联姻，就是 monad 之所以成为 monad 的原因。来看看它更精确的完整定义：

> monad 是可以变扁（flatten）的 pointed functor。

一个 functor，只要它定义个了一个 `join` 方法和一个 `of` 方法，并遵守一些定律，那么它就是一个 monad。`join` 的实现并不太复杂，我们来为 `Maybe` 定义一个：

```js
Maybe.prototype.join = function() {
  return this.isNothing() ? Maybe.of(null) : this.__value;
}
```

看，就像子宫里双胞胎中的一个吃掉另一个那么简单。如果有一个 `Maybe(Maybe(x))`，那么 `.__value` 将会移除多余的一层，然后我们就能安心地从那开始进行 `map`。要不然，我们就将会只有一个 `Maybe`，因为从一开始就没有任何东西被 `map` 调用。

既然已经有了 `join` 方法，我们把 monad 魔法作用到 `firstAddressStreet` 例子上，看看它的实际作用：

```js
//  join :: Monad m => m (m a) -> m a
var join = function(mma){ return mma.join(); }

//  firstAddressStreet :: User -> Maybe Street
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

firstAddressStreet(
  {addresses: [{street: {name: 'Mulburry', number: 8402}, postcode: "WC2N" }]}
);
// Maybe({name: 'Mulburry', number: 8402})
```

只要遇到嵌套的 `Maybe`，就加一个 `join`，防止它们从手中溜走。我们对 `IO` 也这么做试试看，感受下这种感觉。

```js
IO.prototype.join = function() {
  return this.unsafePerformIO();
}
```

同样是简单地移除了一层容器。注意，我们还没有提及纯粹性的问题，仅仅是移除过度紧缩的包裹中的一层而已。

```js
//  log :: a -> IO a
var log = function(x) {
  return new IO(function() { console.log(x); return x; });
}

//  setStyle :: Selector -> CSSProps -> IO DOM
var setStyle = curry(function(sel, props) {
  return new IO(function() { return jQuery(sel).css(props); });
});

//  getItem :: String -> IO String
var getItem = function(key) {
  return new IO(function() { return localStorage.getItem(key); });
};

//  applyPreferences :: String -> IO DOM
var applyPreferences = compose(
  join, map(setStyle('#main')), join, map(log), map(JSON.parse), getItem
);


applyPreferences('preferences').unsafePerformIO();
// Object {backgroundColor: "green"}
// <div style="background-color: 'green'"/>
```

`getItem` 返回了一个 `IO String`，所以可以直接用 `map` 来解析它。`log` 和 `setStyle` 返回的都是 `IO`，所以必须要使用 `join` 来保证这里边的嵌套处于控制之中。

## chain 函数

（译者注：此处标题原文是“My chain hits my chest”，是英国歌手 M.I.A 单曲 *Bad Girls* 的一句歌词。据说这首歌有体现女权主义。）

<img src="images/chain.jpg" alt="chain" />

你可能已经从上面的例子中注意到这种模式了：我们总是在紧跟着 `map` 的后面调用 `join`。让我们把这个行为抽象到一个叫做 `chain` 的函数里。

```js
//  chain :: Monad m => (a -> m b) -> m a -> m b
var chain = curry(function(f, m){
  return m.map(f).join(); // 或者 compose(join, map(f))(m)
});
```

这里仅仅是把 map/join 套餐打包到一个单独的函数中。如果你之前了解过 monad，那你可能已经看出来 `chain` 叫做 `>>=`（读作 bind）或者 `flatMap`；都是同一个概念的不同名称罢了。我个人认为 `flatMap` 是最准确的名称，但本书还是坚持使用 `chain`，因为它是 JS 里接受程度最高的一个。我们用 `chain` 重构下上面两个例子：

```js
// map/join
var firstAddressStreet = compose(
  join, map(safeProp('street')), join, map(safeHead), safeProp('addresses')
);

// chain
var firstAddressStreet = compose(
  chain(safeProp('street')), chain(safeHead), safeProp('addresses')
);



// map/join
var applyPreferences = compose(
  join, map(setStyle('#main')), join, map(log), map(JSON.parse), getItem
);

// chain
var applyPreferences = compose(
  chain(setStyle('#main')), chain(log), map(JSON.parse), getItem
);
```

我把所有的 `map/join` 都替换为了 `chain`，这样代码就显得整洁了些。整洁固然是好事，但 `chain` 的能力却不止于此——它更多的是龙卷风而不是吸尘器。因为 `chain` 可以轻松地嵌套多个作用，因此我们就能以一种纯函数式的方式来表示 *序列*（sequence） 和 *变量赋值*（variable assignment）。

```js
// getJSON :: Url -> Params -> Task JSON
// querySelector :: Selector -> IO DOM


getJSON('/authenticate', {username: 'stale', password: 'crackers'})
  .chain(function(user) {
    return getJSON('/friends', {user_id: user.id});
});
// Task([{name: 'Seimith', id: 14}, {name: 'Ric', id: 39}]);


querySelector("input.username").chain(function(uname) {
  return querySelector("input.email").chain(function(email) {
    return IO.of(
      "Welcome " + uname.value + " " + "prepare for spam at " + email.value
    );
  });
});
// IO("Welcome Olivia prepare for spam at olivia@tremorcontrol.net");


Maybe.of(3).chain(function(three) {
  return Maybe.of(2).map(add(three));
});
// Maybe(5);


Maybe.of(null).chain(safeProp('address')).chain(safeProp('street'));
// Maybe(null);
```

本来我们可以用 `compose` 写上面的例子，但这将需要几个帮助函数，而且这种风格怎么说都要通过闭包进行明确的变量赋值。相反，我们使用了插入式的 `chain`。顺便说一下，`chain` 可以自动从任意类型的 `map` 和 `join` 衍生出来，就像这样：`t.prototype.chain = function(f) { return this.map(f).join(); }`。如果手动定义 `chain` 能让你觉得性能会好点的话（实际上并不会），我们也可以手动定义它，尽管还必须要费力保证函数功能的正确性——也就是说，它必须与紧接着后面有 `join` 的 `map` 相等。如果 `chain` 是简单地通过结束调用 `of` 后把值放回容器这种方式定义的，那么就会造成一个有趣的后果，即可以从 `chain` 那里衍生出一个 `map`。同样地，我们还可以用 `chain(id)` 定义 `join`。听起来好像是在跟魔术师玩德州扑克，魔术师想要什么牌就有什么牌；但是就像大部分的数学理论一样，所有这些原则性的结构都是相互关联的。[fantasyland](https://github.com/fantasyland/fantasy-land) 仓库中提到了许多上述衍生概念，这个仓库也是 JavaScript 官方的代数数据结构（algebraic data types）标准。

好了，我们来看上面的例子。第一个例子中，可以看到两个 `Task` 通过 `chain` 连接形成了一个异步操作的序列——它先获取 `user`，然后用 `user.id` 查找 `user` 的 `friends`。`chain` 避免了 `Task(Task([Friend]))` 这种情况。

第二个例子是用 `querySelector` 查找几个 input 然后创建一条欢迎信息。注意看我们是如何在最内层的函数里访问 `uname` 和 `email` 的——这是函数式变量赋值的绝佳表现。因为 `IO` 大方地把它的值借给了我们，我们也要负起以同样方式把值放回去的责任——不能辜负它的信任（还有整个程序的信任）。`IO.of` 非常适合做这件事，同时它也解释了为何 pointed 这一特性是 monad 接口得以存在的重要前提。不过，`map` 也能返回正确的类型：

```js
querySelector("input.username").chain(function(uname) {
  return querySelector("input.email").map(function(email) {
    return "Welcome " + uname.value + " prepare for spam at " + email.value;
  });
});
// IO("Welcome Olivia prepare for spam at olivia@tremorcontrol.net");
```

最后两个例子用了 `Maybe`。因为 `chain` 其实是在底层调用了 `map`，所以如果遇到 `null`，代码就会立刻停止运行。

如果觉得这些例子不太容易理解，你也不必担心。多跑跑代码，多琢磨琢磨，把代码拆开来研究研究，再把它们拼起来看看。总之记住，返回的如果是“普通”值就用 `map`，如果是 `functor` 就用 `chain`。

这里我得提醒一下，上述方式对两个不同类型的嵌套容器是不适用的。functor 组合，以及后面会讲到的 monad transformer 可以帮助我们应对这种情况。

## 炫耀

这种容器编程风格有时也能造成困惑，我们不得不努力理解一个值到底嵌套了几层容器，或者需要用 `map` 还是 `chain`（很快我们就会认识更多的容器类型）。使用一些技巧，比如重写 `inspect` 方法之类，能够大幅提高 debug 的效率。后面我们也会学习如何创建一个“栈”，使之能够处理任何丢给它的作用（effects）。不过，有时候也需要权衡一下是否值得这样做。

我很乐意挥起 monad 之剑，向你展示这种编程风格的力量。就以读一个文件，然后就把它直接上传为例吧：

```js
// readFile :: Filename -> Either String (Future Error String)
// httpPost :: String -> Future Error JSON

//  upload :: String -> Either String (Future Error JSON)
var upload = compose(map(chain(httpPost('/uploads'))), readFile);
```

这里，代码不止一次在不同的分支执行。从类型签名可以看出，我们预防了三个错误——`readFile` 使用 `Either` 来验证输入（或许还有确保文件名存在）；`readFile` 在读取文件的时候可能会出错，错误通过 `readFile` 的 `Future` 表示；文件上传可能会因为各种各样的原因出错，错误通过 `httpPost` 的 `Future` 表示。我们就这么随意地使用 `chain` 实现了两个嵌套的、有序的异步执行动作。

所有这些操作都是在一个从左到右的线性流中完成的，是完完全全纯的、声明式的代码，是可以等式推导（equational reasoning）并拥有可靠特性（reliable properties）的代码。我们没有被迫使用不必要甚至令人困惑的变量名，我们的 `upload` 函数符合通用接口而不是特定的一次性接口。这些都是在一行代码中完成的啊！

让我们来跟标准的命令式的实现对比一下：

```js
//  upload :: String -> (String -> a) -> Void
var upload = function(filename, callback) {
  if(!filename) {
    throw "You need a filename!";
  } else {
    readFile(filename, function(err, contents) {
      if(err) throw err;
      httpPost(contents, function(err, json) {
        if(err) throw err;
        callback(json);
      });
    });
  }
}
```

看看，这简直就是魔鬼的算术（译者注：此处原文是“the devil's arithmetic”，为美国 1988 年出版的历史小说，讲述一个犹太小女孩穿越到 1942 年的集中营的故事。此书亦有同名改编电影，中译名《穿梭集中营》），我们就像一颗弹珠一样在变幻莫测的迷宫中穿梭。无法想象如果这是一个典型的应用，而且一直在改变变量会怎样——我们肯定会像陷入沥青坑那样无所适从。

# 理论

我们要看的第一条定律是结合律，但可能不是你熟悉的那个结合律。

```js
  // 结合律
  compose(join, map(join)) == compose(join, join)
```

这些定律表明了 monad 的嵌套本质，所以结合律关心的是如何让内层或外层的容器类型 `join`，然后取得同样的结果。用一张图来表示可能效果会更好：

<img src="images/monad_associativity.png" alt="monad associativity law" />

从左上角往下，先用 `join` 合并 `M(M(M a))` 最外层的两个 `M`，然后往右，再调用一次 `join`，就得到了我们想要的 `M a`。或者，从左上角往右，先打开最外层的 `M`，用 `map(join)` 合并内层的两个 `M`，然后再向下调用一次 `join`，也能得到 `M a`。不管是先合并内层还是先合并外层的 `M`，最后都会得到相同的 `M a`，所以这就是结合律。值得注意的一点是 `map(join) != join`。两种方式的中间步骤可能会有不同的值，但最后一个 `join` 调用后最终结果是一样的。

第二个定律与结合律类似：

```js
  // 同一律 (M a)
  compose(join, of) == compose(join, map(of)) == id
```

这表明，对任意的 monad `M`，`of` 和 `join` 相当于 `id`。也可以使用 `map(of)` 由内而外实现相同效果。我们把这个定律叫做“三角同一律”（triangle identity），因为把它图形化之后就像一个三角形：

<img src="images/triangle_identity.png" alt="monad identity law" />

如果从左上角开始往右，可以看到 `of` 的确把 `M a` 丢到另一个 `M` 容器里去了。然后再往下 `join`，就得到了 `M a`，跟一开始就调用 `id` 的结果一样。从右上角往左，可以看到如果我们通过 `map` 进到了 `M` 里面，然后对普通值 `a` 调用 `of`，最后得到的还是 `M (M a)`；再调用一次 `join` 将会把我们带回原点，即 `M a`。

我要说明一点，尽管这里我写的是 `of`，实际上对任意的 monad 而言，都必须要使用明确的 `M.of`。

我已经见过这些定律了，同一律和结合律，以前就在哪儿见过...等一下，让我想想...是的！它们是范畴遵循的定律！不过这意味着我们需要一个组合函数来给出一个完整定义。见证吧：


```js
  var mcompose = function(f, g) {
    return compose(chain(f), chain(g));
  }

  // 左同一律
  mcompose(M, f) == f

  // 右同一律
  mcompose(f, M) == f

  // 结合律
  mcompose(mcompose(f, g), h) == mcompose(f, mcompose(g, h))
```

毕竟它们是范畴学里的定律。monad 来自于一个叫 “Kleisli 范畴”的范畴，这个范畴里边所有的对象都是 monad，所有的态射都是联结函数（chained funtions）。我不是要在没有提供太多解释的情况下，拿范畴学里各式各样的概念来取笑你。我的目的是涉及足够多的表面知识，向你说明这中间的相关性，让你在关注日常实用特性之余，激发起对这些定律的兴趣。


## 总结

monad 让我们深入到嵌套的运算当中，使我们能够在完全避免回调金字塔（pyramid of doom）情况下，为变量赋值，运行有序的作用，执行异步任务等等。当一个值被困在几层相同类型的容器中时，monad 能够拯救它。借助 “pointed” 这个可靠的帮手，monad 能够借给我们从盒子中取出的值，而且知道我们会在结束使用后还给它。

是的，monad 非常强大，但我们还需要一些额外的容器函数。比如，假设我们想同时运行一个列表里的 api 调用，然后再搜集返回的结果，怎么办？是可以使用 monad 实现这个任务，但必须要等每一个 api 完成后才能调用下一个。合并多个合法性验证呢？我们想要的肯定是持续验证以搜集错误列表，但是 monad 会在第一个 `Left` 登场的时候停掉整个演出。

下一章，我们将看到 applicative functor 如何融入这个容器世界，以及为何在很多情况下它比 monad 更好用。

[第 10 章: Applicative Functor](ch10.md)


## 练习

```js
// 练习 1
// ==========
// 给定一个 user，使用 safeProp 和 map/join 或 chain 安全地获取 sreet 的 name

var safeProp = _.curry(function (x, o) { return Maybe.of(o[x]); });
var user = {
  id: 2,
  name: "albert",
  address: {
    street: {
      number: 22,
      name: 'Walnut St'
    }
  }
};

var ex1 = undefined;


// 练习 2
// ==========
// 使用 getFile 获取文件名并删除目录，所以返回值仅仅是文件，然后以纯的方式打印文件

var getFile = function() {
  return new IO(function(){ return __filename; });
}

var pureLog = function(x) {
  return new IO(function(){
    console.log(x);
    return 'logged ' + x;
  });
}

var ex2 = undefined;



// 练习 3
// ==========
// 使用 getPost() 然后以 post 的 id 调用 getComments()
var getPost = function(i) {
  return new Task(function (rej, res) {
    setTimeout(function () {
      res({ id: i, title: 'Love them tasks' });
    }, 300);
  });
}

var getComments = function(i) {
  return new Task(function (rej, res) {
    setTimeout(function () {
      res([
        {post_id: i, body: "This book should be illegal"},
        {post_id: i, body: "Monads are like smelly shallots"}
      ]);
    }, 300);
  });
}


var ex3 = undefined;


// 练习 4
// ==========
// 用 validateEmail、addToMailingList 和 emailBlast 实现 ex4 的类型签名

//  addToMailingList :: Email -> IO([Email])
var addToMailingList = (function(list){
  return function(email) {
    return new IO(function(){
      list.push(email);
      return list;
    });
  }
})([]);

function emailBlast(list) {
  return new IO(function(){
    return 'emailed: ' + list.join(',');
  });
}

var validateEmail = function(x){
  return x.match(/\S+@\S+\.\S+/) ? (new Right(x)) : (new Left('invalid email'));
}

//  ex4 :: Email -> Either String (IO String)
var ex4 = undefined;
```
