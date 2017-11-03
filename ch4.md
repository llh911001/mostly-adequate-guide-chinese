# 第 4 章: 柯里化（curry）

## 不可或缺的 curry

（译者注：原标题是“Can't live if livin' is without you”，为英国乐队 Badfinger 歌曲 *Without You* 中歌词。）

我父亲以前跟我说过，有些事物在你得到之前是无足轻重的，得到之后就不可或缺了。微波炉是这样，智能手机是这样，互联网也是这样——老人们在没有互联网的时候过得也很充实。对我来说，函数的柯里化（curry）也是这样。

curry 的概念很简单：只传递给函数一部分参数来调用它，让它返回一个函数去处理剩下的参数。

你可以一次性地调用 curry 函数，也可以每次只传一个参数分多次调用。

```js
var add = function(x) {
  return function(y) {
    return x + y;
  };
};

var increment = add(1);
var addTen = add(10);

increment(2);
// 3

addTen(2);
// 12
```

这里我们定义了一个 `add` 函数，它接受一个参数并返回一个新的函数。调用 `add` 之后，返回的函数就通过闭包的方式记住了 `add` 的第一个参数。一次性地调用它实在是有点繁琐，好在我们可以使用一个特殊的 `curry` 帮助函数（helper function）使这类函数的定义和调用更加容易。

我们来创建一些 curry 函数享受下（译者注：此处原文是“for our enjoyment”，语出自圣经）。

```js
var curry = require('lodash').curry;

var match = curry(function(what, str) {
  return str.match(what);
});

var replace = curry(function(what, replacement, str) {
  return str.replace(what, replacement);
});

var filter = curry(function(f, ary) {
  return ary.filter(f);
});

var map = curry(function(f, ary) {
  return ary.map(f);
});
```

我在上面的代码中遵循的是一种简单，同时也非常重要的模式。即策略性地把要操作的数据（String， Array）放到最后一个参数里。到使用它们的时候你就明白这样做的原因是什么了。

```js
match(/\s+/g, "hello world");
// [ ' ' ]

match(/\s+/g)("hello world");
// [ ' ' ]

var hasSpaces = match(/\s+/g);
// function(x) { return x.match(/\s+/g) }

hasSpaces("hello world");
// [ ' ' ]

hasSpaces("spaceless");
// null

filter(hasSpaces, ["tori_spelling", "tori amos"]);
// ["tori amos"]

var findSpaces = filter(hasSpaces);
// function(xs) { return xs.filter(function(x) { return x.match(/\s+/g) }) }

findSpaces(["tori_spelling", "tori amos"]);
// ["tori amos"]

var noVowels = replace(/[aeiou]/ig);
// function(replacement, x) { return x.replace(/[aeiou]/ig, replacement) }

var censored = noVowels("*");
// function(x) { return x.replace(/[aeiou]/ig, "*") }

censored("Chocolate Rain");
// 'Ch*c*l*t* R**n'
```

这里表明的是一种“预加载”函数的能力，通过传递一到两个参数调用函数，就能得到一个记住了这些参数的新函数。

我鼓励你使用 `npm install lodash` 安装 `lodash`，复制上面的代码放到 REPL 里跑一跑。当然你也可以在能够使用 `lodash` 或 `ramda` 的网页中运行它们。

## 不仅仅是双关语／咖喱

curry 的用处非常广泛，就像在 `hasSpaces`、`findSpaces` 和 `censored` 看到的那样，只需传给函数一些参数，就能得到一个新函数。

用 `map` 简单地把参数是单个元素的函数包裹一下，就能把它转换成参数为数组的函数。

```js
var getChildren = function(x) {
  return x.childNodes;
};

var allTheChildren = map(getChildren);
```

只传给函数一部分参数通常也叫做*局部调用*（partial application），能够大量减少样板文件代码（boilerplate code）。考虑上面的 `allTheChildren` 函数，如果用 lodash 的普通 `map` 来写会是什么样的（注意参数的顺序也变了）：

```js
var allTheChildren = function(elements) {
  return _.map(elements, getChildren);
};
```

通常我们不定义直接操作数组的函数，因为只需内联调用 `map(getChildren)` 就能达到目的。这一点同样适用于 `sort`、`filter` 以及其他的高阶函数（higher order function）（高阶函数：参数或返回值为函数的函数）。

当我们谈论*纯函数*的时候，我们说它们接受一个输入返回一个输出。curry 函数所做的正是这样：每传递一个参数调用函数，就返回一个新函数处理剩余的参数。这就是一个输入对应一个输出啊。

哪怕输出是另一个函数，它也是纯函数。当然 curry 函数也允许一次传递多个参数，但这只是出于减少 `()` 的方便。

## 总结

curry 函数用起来非常得心应手，每天使用它对我来说简直就是一种享受。它堪称手头必备工具，能够让函数式编程不那么繁琐和沉闷。

通过简单地传递几个参数，就能动态创建实用的新函数；而且还能带来一个额外好处，那就是保留了数学的函数定义，尽管参数不止一个。
下一章我们将学习另一个重要的工具：`组合`（compose）。

[第 5 章: 代码组合（compose）](ch5.md)

## 练习

开始练习之前先说明一下，我们将默认使用 [ramda](http://ramdajs.com) 这个库来把函数转为 curry 函数。或者你也可以选择由 lodash 的作者编写和维护的 [lodash-fp](https://github.com/lodash/lodash-fp)。这两个库都很好用，选择哪一个就看你自己的喜好了。

你还可以对自己的练习代码做[单元测试](https://github.com/llh911001/mostly-adequate-guide-chinese/tree/master/code/part1_exercises)，或者把代码拷贝到一个 REPL 里运行看看。

这些练习的答案可以在[本书仓库](https://github.com/llh911001/mostly-adequate-guide-chinese/tree/master/code/part1_exercises/answers)中找到。

```js
var _ = require('ramda');


// 练习 1
//==============
// 通过局部调用（partial apply）移除所有参数

var words = function(str) {
  return split(' ', str);
};

// 练习 1a
//==============
// 使用 `map` 创建一个新的 `words` 函数，使之能够操作字符串数组

var sentences = undefined;


// 练习 2
//==============
// 通过局部调用（partial apply）移除所有参数

var filterQs = function(xs) {
  return filter(function(x){ return match(/q/i, x);  }, xs);
};


// 练习 3
//==============
// 使用帮助函数 `_keepHighest` 重构 `max` 使之成为 curry 函数

// 无须改动:
var _keepHighest = function(x,y){ return x >= y ? x : y; };

// 重构这段代码:
var max = function(xs) {
  return reduce(function(acc, x){
    return _keepHighest(acc, x);
  }, -Infinity, xs);
};


// 彩蛋 1:
// ============
// 包裹数组的 `slice` 函数使之成为 curry 函数
// //[1,2,3].slice(0, 2)
var slice = undefined;


// 彩蛋 2:
// ============
// 借助 `slice` 定义一个 `take` curry 函数，该函数调用后可以取出字符串的前 n 个字符。
var take = undefined;
```
