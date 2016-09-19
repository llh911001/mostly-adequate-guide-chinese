# 第 1 章：我们在做什么？

## 介绍

你好，我是 Franklin Risby 教授，很高兴认识你。接下来我们将共度一段时光了，因为我要教你一些函数式编程的知识。好了，关于我就介绍到这里，你怎么样？我希望你已经熟悉 JavaScript 语言了，关于面向对象也有一点点的经验了，而且自认为是一个合格的程序员。希望你没有昆虫学博士学位也能找到并杀死一些臭虫（bug）。

我并不假设你之前有任何函数式编程相关的知识——我们都知道假设的后果是什么（译者注：此处原文是“we both know what happens when you assume”，源自一句名言“When you assume you make an ASS of U and ME”，意思是“让两人都难堪”）。但我猜想你在使用可变状态（mutable state）、无限制副作用（unrestricted side effects）和无原则设计（unprincipled design）的过程中已经遇到过一些麻烦。好了，介绍到此为止，我们进入正题。

本章的目的是让你对函数式编程的目的有一个初步认识，对一个程序之所以是*函数式*程序的原因有一定了解，要不然就会像无头苍蝇一样，不问青红皂白地避免使用对象——这等于是在做无用功。写代码需要遵循一定的原则，就像水流湍急的时候你需要天文罗盘来指引一样。

现在已经有一些通用的编程原则了，各种缩写词带领我们在编程的黑暗隧道里前行：DRY（不要重复自己，don't repeat yourself），高内聚低耦合（loose coupling high cohesion），YAGNI （你不会用到它的，ya ain't gonna need it），最小意外原则（Principle of least surprise），单一责任（single responsibility）等等。

我当然不会啰里八嗦地把这些年我听到的原则都列举出来，你知道重点就行。重点是这些原则同样适用于函数式编程，只不过它们与本书的主题不十分相关。在我们深入主题之前，我想先通过本章给你这样一种感觉，即你在敲键盘的时候内心就能强烈感受到的那种函数式的氛围。

<!--BREAK-->

## 一个简单例子

我们从一个愚蠢的例子开始。下面是一个海鸥程序，鸟群合并则变成了一个更大的鸟群，繁殖则增加了鸟群的数量，增加的数量就是它们繁殖出来的海鸥的数量。注意这个程序并不是面向对象的良好实践，它只是强调当前这种变量赋值方式的一些弊端。

```js
var Flock = function(n) {
  this.seagulls = n;
};

Flock.prototype.conjoin = function(other) {
  this.seagulls += other.seagulls;
  return this;
};

Flock.prototype.breed = function(other) {
  this.seagulls = this.seagulls * other.seagulls;
  return this;
};

var flock_a = new Flock(4);
var flock_b = new Flock(2);
var flock_c = new Flock(0);

var result = flock_a.conjoin(flock_c).breed(flock_b).conjoin(flock_a.breed(flock_b)).seagulls;
//=> 32
```

我相信没人会写这样糟糕透顶的程序。代码的内部可变状态非常难以追踪，而且，最终的答案还是错的！正确答案是 `16`，但是因为 `flock_a` 在运算过程中永久地改变了，所以得出了错误的结果。这是 IT 部门混乱的表现，非常粗暴的计算方式。

如果你看不懂这个程序，没关系，我也看不懂。重点是状态和可变值非常难以追踪，即便是在这么小的一个程序中也不例外。

我们试试另一种更函数式的写法：

```js
var conjoin = function(flock_x, flock_y) { return flock_x + flock_y };
var breed = function(flock_x, flock_y) { return flock_x * flock_y };

var flock_a = 4;
var flock_b = 2;
var flock_c = 0;

var result = conjoin(breed(flock_b, conjoin(flock_a, flock_c)), breed(flock_a, flock_b));
//=>16
```

很好，这次我们得到了正确的答案，而且少写了很多代码。不过函数嵌套有点让人费解...（我们会在第 5 章解决这个问题）。这种写法也更优雅，不过代码肯定是越直白越好，所以如果我们再深入挖掘，看看这段代码究竟做了什么事，我们会发现，它不过是在进行简单的加（`conjoin`） 和乘（`breed`）运算而已。

代码中的两个函数除了函数名有些特殊，其他没有任何难以理解的地方。我们把它们重命名一下，看看它们的真面目。

```js
var add = function(x, y) { return x + y };
var multiply = function(x, y) { return x * y };

var flock_a = 4;
var flock_b = 2;
var flock_c = 0;

var result = add(multiply(flock_b, add(flock_a, flock_c)), multiply(flock_a, flock_b));
//=>16
```

这么一来，你会发现我们不过是在运用古人早已获得的知识：

```js
// 结合律（assosiative）
add(add(x, y), z) == add(x, add(y, z));

// 交换律（commutative）
add(x, y) == add(y, x);

// 同一律（identity）
add(x, 0) == x;

// 分配律（distributive）
multiply(x, add(y,z)) == add(multiply(x, y), multiply(x, z));
```

是的，这些经典的数学定律迟早会派上用场。不过如果你一时想不起来也没关系，多数人已经很久没复习过这些数学知识了。我们来看看能否运用这些定律简化这个海鸥小程序。

```js
// 原有代码
add(multiply(flock_b, add(flock_a, flock_c)), multiply(flock_a, flock_b));

// 应用同一律，去掉多余的加法操作（add(flock_a, flock_c) == flock_a）
add(multiply(flock_b, flock_a), multiply(flock_a, flock_b));

// 再应用分配律
multiply(flock_b, add(flock_a, flock_a));
```

漂亮！除了调用的函数，一点多余的代码都不需要写。当然这里我们定义 `add` 和 `multiply` 是为了代码完整性，实际上并不必要——在调用之前它们肯定已经在某个类库里定义好了。

你可能在想“你也太偷换概念了吧，居然举一个这么数学的例子”，或者“真实世界的应用程序比这复杂太多，不能这么简单地推理”。我之所以选择这样一个例子，是因为大多数人都知道加法和乘法，所以很容易就能理解数学可以如何为我们所用。

不要感到绝望，本书后面还会穿插一些范畴学（category theory）、集合论（set theory）以及 lambda 运算的知识，教你写更加复杂的代码，而且一点也不输本章这个海鸥程序的简洁性和准确性。你也不需要成为一个数学家，本书要教给你的编程范式实践起来就像是使用一个普通的框架或者 api 一样。

你也许会惊讶，我们可以像上例那样遵循函数式的范式去书写完整的、日常的应用程序，有着优异性能的程序，简洁且易推理的程序，以及不用每次都重新造轮子的程序。如果你是罪犯，那违法对你来说是好事；但在本书中，我们希望能够承认并遵守数学之法。

我们希望去践行每一部分都能完美接合的理论，希望能以一种通用的、可组合的组件来表示我们的特定问题，然后利用这些组件的特性来解决这些问题。相比命令式（稍后本书将会介绍命令式的精确定义，暂时我们还是先把重点放在函数式上）编程的那种“某某去做某事”的方式，函数式编程将会有更多的约束，不过你会震惊于这种强约束、数学性的“框架”所带来的回报。

我们已经看到函数式的点点星光了，但在真正开始我们的旅程之前，我们要先掌握一些具体的概念。

[第 2 章：一等公民的函数](ch2.md)
