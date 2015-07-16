# 第 1 章：我们在做什么？

## 介绍

大家好，我是 Franklin Risby 教授，非常高兴认识大家。我想你要花点时间跟着我学习函数式编程了。废话不多说，我希望你已经熟悉 JavaScript 语言了，关于面向对象也有一点点的经验了，而且自认为是一个合格的程序员。希望你不需要有昆虫学博士学位就能找到并杀死一些 bug。我并不假设你之前有任何函数式编程相关的知识——我们都知道假设的后果是什么。但我希望你在使用可变状态（mutable state）、无限制副作用（unrestricted side effects）和无原则设计（unprincipled design）的过程中已经碰到一些坑。好了，介绍到此为止，我们进入正题。

本章的目的是给你一种编写函数式程序的感觉。我们必须要对*函数式*有一定的了解，要不然就会像无头苍蝇一样，甚至不问青红照白地避免使用对象——这等于是在做无用功。写代码需要遵循一个重点，就像《激战》游戏里当水变成石头的时候你需要天国罗盘来指引。现在已经有一些通用的编程原则了，各种缩写词带领我们在编程的黑暗隧道里前行：DRY（不要重复自己，dont't repeat yourself），松耦合强一致（loose coupling high cohesion），YAGNI （你不会用到它的，ya ain't gonna need it），最小意外原则（Principle of least surprise），单一责任（single responsibility）等等。我当然不会把这些年我听到的所有原则都列举出来，你知道重点就行。重点是这些原则同样适用于函数式编程，只不过它们与本书的主题不十分相关。在我们深入主题之前，我想先通过本章给你这样一种感觉，即你在敲键盘的时候内心就能强烈感受到的那种函数式的氛围。
<!--BREAK-->

## A brief encounter

Let's start with a touch of insanity. Here is a seagull application. When flocks conjoin they become a larger flock and when they breed they increase by the number of seagulls with whom they're breeding.

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

Who on earth would craft such a ghastly abomination? It is unreasonably difficult to keep track of the mutating internal state. And, good heavens, the answer is even incorrect! It should have been `16`, but `flock_a` wound up permanently altered in the process. Poor `flock_a`. This is anarchy in the I.T.! This is wild animal arithmetic!

Let's try again with a more functional approach:

```js
var conjoin = function(flock_x, flock_y) { return flock_x + flock_y };
var breed = function(flock_x, flock_y) { return flock_x * flock_y };

var flock_a = 4;
var flock_b = 2;
var flock_c = 0;

var result = conjoin(breed(flock_b, conjoin(flock_a, flock_c)), breed(flock_a, flock_b));
//=>16
```

Well, we got the right answer this time. There's much less code. It's better, but let's dig deeper. There are benefits to calling a spade a spade. Had we done so, we might have seen we're just working with simple addition(`conjoin`) and multiplication(`breed`). There's really nothing special at all about these two functions other than their names. Let's rename our custom functions to reveal their true identity.

```js
var add = function(x, y) { return x + y };
var multiply = function(x, y) { return x * y };

var flock_a = 4;
var flock_b = 2;
var flock_c = 0;

var result = add(multiply(flock_b, add(flock_a, flock_c)), multiply(flock_a, flock_b));
//=>16
```
And with that, we gain the knowledge of the ancients:

```js
// associative
add(add(x, y), z) == add(x, add(y, z));

// commutative
add(x, y) == add(y, x);

// identity
add(x, 0) == x;

// distributive
multiply(x, add(y,z)) == add(multiply(x, y), multiply(x, z));
```

Ah yes, those old faithful mathematical properties should come in handy. Don't worry if you didn't know them right off the top of your head. For a lot of us, it's been a while since we've reviewed this information. Let's see if we can use these properties to simplify our little seagull program.

```js
// Original line
add(multiply(flock_b, add(flock_a, flock_c)), multiply(flock_a, flock_b));

// Apply the identity property to remove the extra add (add(flock_a, flock_c) == flock_a)
add(multiply(flock_b, flock_a), multiply(flock_a, flock_b));

// Apply distributive property to achieve our result
multiply(flock_b, add(flock_a, flock_a));
```

Brilliant! We didn't have to write a lick of custom code other than our calling function. We include `add` and `multiply` definitions here for completeness, but there is really no need to write them - we surely have an `add` and `multiply` provided by some previously written library. Contrast this with our absurd object version which ignores built in functions and data types in order to model the "real world".

You may be thinking "how very strawman of you to put such a mathy example up front". Or "real programs are not this simple and cannot be reasoned about in such a way". I've chosen this example because most of us already know about addition and multiplication so it's easy to see how math can be of use to us here. Don't despair, throughout this book, we'll sprinkle in some category theory, set theory, and lambda calculus to write real world examples that achieve the same simplicity and results as our flock of seagulls example. You needn't be a mathematician either, it will feel just like using a normal framework or api.

It may come as a surprise to hear that we can write full, everyday applications along the lines of the functional analog above. Programs that have sound properties. Programs that are terse, yet easy to reason about. Programs that don't reinvent the wheel at every turn. Lawlessness is good if you're a criminal, but in this book, we'll want to acknowledge and obey the laws of math. We'll want to use the theory where every piece tends to fit together so politely. We'll want to represent our specific problem in terms of generic, composable bits and then exploit their properties for our own selfish benefit. It will take a bit more discipline than the "anything goes" approach of imperative[^We'll go over the precise definition of imperative later in the book, but for now it's anything other than functional programming] programming, but the payoff of working within a principled, mathematical framework will astound you.

We've seen a flicker of our functional north star, but there are a few concrete concepts to grasp before we can really begin our journey.

[Chapter 2: First Class Functions](ch2.md)
