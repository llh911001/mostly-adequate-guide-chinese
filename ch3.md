# 第 3 章：纯函数的好处

## 再次强调“纯”
我们首先要厘清纯函数的概念。

>纯函数是这样一种函数，即相同的输入，永远会得出相同的输出，而且没有任何可观察的副作用。

比如 `slice` 和 `splice`，这两个函数的作用并无二致——但是注意，它们各自的方式却大不同，不过无论如何作用还是一样的。我们说 `slice` 更*纯*是因为对相同的输入它保证能返回相同的输出。而 `splice` 却会嚼掉调用它的那个数组，然后再吐出来；这就会产生可观察到的副作用，即这个数组永久地改变了。

```js
var xs = [1,2,3,4,5];

// 纯的
xs.slice(0,3);
//=> [1,2,3]

xs.slice(0,3);
//=> [1,2,3]

xs.slice(0,3);
//=> [1,2,3]


// 不纯的
xs.splice(0,3);
//=> [1,2,3]

xs.splice(0,3);
//=> [4,5]

xs.splice(0,3);
//=> []
```

在函数式编程中，我们讨厌像 `splice` 这样会*改变*数据的笨函数。我们追求的是那种可靠的，每次都能返回同样结果的函数，而不是像 `splice` 这样每次调用都把数据弄的一团糟的函数，这不是我们想要的。

来看看另一个例子。

```js
// 不纯的
var minimum = 21;

var checkAge = function(age) {
  return age >= minimum;
};


// 纯的
var checkAge = function(age) {
  var minimum = 21;
  return age >= minimum;
};
```

In the impure portion, `checkAge` depends on the mutable variable `minimum` to determine the result. In other words, it depends on system state which is disappointing because it increases the cognitive load by introducing an external environment. It might not seem like a lot in this example, but this reliance upon state is one of the largest contributors to system complexity[^http://www.curtclifton.net/storage/papers/MoseleyMarks06a.pdf]. This `checkAge` may return different results depending on factors external to input, which not only disqualifies it from being pure, but also puts our minds through the ringer each time we're reasoning about the software. Its pure form, on the other hand, is completely self sufficient. We can  also make `minimum` immutable, which preserves the purity as the state will never change. To do this, we must create an object to freeze.

```js
var immutableState = Object.freeze({
  minimum: 21
});
```

## Side effects may include...

Let's look more at these "side effects" to improve our intuition. So what is this undoubtedly nefarious *side effect* mentioned in the definition of *pure function*? We'll be referring to *effect* as anything that occurs in our computation besides the calculation of a result. There's nothing intrinsically bad about effects and we'll be using them all over the place in the chapters to come. It's that *side* part that bears the negative connotation. Water alone is not an inherent larvae incubator, it's the *stagnant* part that yields the swarms, and I assure you, *side* effects are a similar breeding ground in your own programs.

>A *side effect* is a change of system state or *observable interaction* with the outside world that occurs during the calculation of a result.

Side effects may include, but are not limited to

  * changing the file system
  * inserting a record into a database
  * making an http call
  * mutations
  * printing to the screen / logging
  * obtaining user input
  * querying the DOM
  * accessing system state

And the list goes on and on. Any interaction with the world outside of a function is a side effect, which is a fact that may prompt you to suspect the practicality of programming without them. The philosophy of functional programming postulates that side effects are a primary cause of incorrect behavior. It is not that we're forbidden to use them, rather we want to contain them and run them in a controlled way. We'll learn how to do this when we get to functors and monads in later chapters, but for now, let's try to keep these insidious functions separate from our pure ones.

Side effects disqualify a function from being *pure* and it makes sense: pure functions, by definition, must always return the same output given the same input, which is not possible to guarantee when dealing with matters outside our local function. Let's take a closer look at why we insist on the same output per input. Pop your collars, we're going to look at some 8th grade math.

## 8th grade math

From mathisfun.com:
> A function is a special relationship between values:
> Each of its input values gives back exactly one output value.

In other words, it's just a relation between two values: the input and the output. Though each input has exactly one output, that output doesn't necessary have to be unique per input. Below shows a diagram of a perfectly valid function from `x` to `y`;

<img src="images/function-sets.gif" />[^http://www.mathsisfun.com/sets/function.html]

To contrast, the following diagram shows a relation that is *not* a function since the input value `5` points to several outputs:

<img src="images/relation-not-function.gif" />[^http://www.mathsisfun.com/sets/function.html]

Functions can be described as a set of pairs with the position (input, output): `[(1,2), (3,6), (5,10)]`[^It appears this function doubles it's input].

Or perhaps a table:
<table> <tr> <th>Input</th> <th>Output</th> </tr> <tr> <td>1</td> <td>2</td> </tr> <tr> <td>2</td> <td>4</td> </tr> <tr> <td>3</td> <td>6</td> </tr> </table>

Or even as a graph with `x` as the input and `y` as the output:

<img src="images/fn_graph.png" width="300" height="300" />


There's no need for implementation details if the input dictates the output. Since functions are simply mappings of input to output, one could simply jot down object literals and run them with `[]` instead of `()`.

```js
var toLowerCase = {"A":"a", "B": "b", "C": "c", "D": "d", "E": "e", "D": "d"};

toLowerCase["C"];
//=> "c"

var isPrime = {1:false, 2: true, 3: true, 4: false, 5: true, 6:false};

isPrime[3];
//=> true
```

Of course, you might want to calculate instead of hand writing things out, but this illustrates a different way to think about functions.[^You may be thinking "what about functions with multiple arguments?". Indeed, that presents a bit of an inconvenience when thinking in terms of mathematics. For now, we can bundle them up in an array or just think of the `arguments` object as the input. When we learn about *currying*, we'll see how we can directly model the mathematical definition of a function.]

Here comes the dramatic reveal: Pure functions *are* mathematical functions and they're what functional programming is all about. Programming with these little angels can provide huge benefits. Let's look at some reasons why we're willing to go to great lengths to preserve purity.

## The case for purity

### Cacheable

For starters, pure functions can always be cached by input. This is typically done using a technique called memoization:

```js
var squareNumber  = memoize(function(x){ return x*x; });

squareNumber(4);
//=> 16

squareNumber(4); // returns cache for input 4
//=> 16

squareNumber(5);
//=> 25

squareNumber(5); // returns cache for input 5
//=> 25
```

Here is a simplified implementation, though there are plenty of more robust versions available.

```js
var memoize = function(f) {
  var cache = {};

  return function() {
    var arg_str = JSON.stringify(arguments);
    cache[arg_str] = cache[arg_str] || f.apply(f, arguments);
    return cache[arg_str];
  };
};
```

Something to note is that you can transform some impure functions into pure ones by delaying evaluation:

```js
var pureHttpCall = memoize(function(url, params){
  return function() { return $.getJSON(url, params); }
});
```

The interesting thing here is that we don't actually make the http call - we instead return a function that will do so when called. This function is pure because it will always return the same output given the same input: the function that will make the particular http call given the `url` and `params`. Our `memoize` function works just fine, though it doesn't cache the results of the http call, rather it caches the generated function.

This is not very useful yet, but we'll soon learn some tricks that will make it so. The take away is that we can cache every function no matter how destructive they seem.

### Portable / Self-Documenting

Pure functions are completely self contained. Everything the function needs is handed to it on a silver platter. Ponder this for a moment... How might this be beneficial? For starters, a function's dependencies are explicit and therefore easier to see and understand - no funny business going on under the hood.

```js
//impure
var signUp = function(attrs) {
  var user = saveUser(attrs);
  welcomeUser(user);
};

//pure
var signUp = function(Db, Email, attrs) {
  return function() {
    var user = saveUser(Db, attrs);
    welcomeUser(Email, user);
  };
};
```

The example here demonstrates that the pure function must be honest about it's dependencies and, as such, tell us exactly what it's up to. Just from it's signature, we know that it will use a `Db`, `Email`, and `attrs` which should be telling to say the least. We'll learn how to make functions like this pure without merely deferring evaluation, but the point should be clear that the pure form is much more informative than it's sneaky impure counterpart which is up to God knows what.

Something else to notice is that we're forced to "inject" dependencies, or pass them in as arguments, which makes our app much more flexible because we've parameterized our database or mail client or what have you[^Don't worry, we'll see a way to make this less tedious than it sounds]. Should we choose to use a different Db we need only to call our function with it. Should we find ourselves writing a new application in which we'd like to reuse this reliable function, we simply give this function whatever `Db` and `Email` we have at the time.

In a JavaScript setting, portability could mean serializing and sending functions over a socket. It could mean running all our app code in web workers. Portability is a powerful trait.

Contrary to "typical" methods and procedures in imperative programming which are rooted deep in their environment via state, dependencies, and available effects, pure functions can be run anywhere our hearts desire. When was the last time you copied a method into a new app? One of my favorite quotes comes from Erlang creator, Joe Armstrong: "The problem with object-oriented languages is they’ve got all this implicit environment that they carry around with them. You wanted a banana but what you got was a gorilla holding the banana...and the entire jungle".

### Testable
Next, we come to realize pure functions make testing much easier. We don't have to mock a "real" payment gateway or setup and assert the state of the world after each test. We simply give it input and assert output. In fact, we find the functional community pioneering new test tools that can blast our functions with generated input and assert that properties hold on the output. It's beyond the scope of this book, but I strongly encourage you to search for and try *Quickcheck* - a testing tool that is tailored for  a purely functional environment.

### Reasonable
Many believe the biggest win when working with pure functions is *referential transparency*. A spot of code is referentially transparent when it can be substituted for its evaluated value without changing the behavior of the program. Since pure functions always return the same output given the same input, we can rely on them to always return the same results and thus preserve referential transparency. Let's see an example.

```js

  var decrementHP = function(player) {
    return player.set("hp", player.hp-1);
  };

  var isSameTeam = function(player1, player2) {
    return player1.team === player2.team;
  };

  var punch = function(player, target) {
    if(isSameTeam(player, target)) {
      return target;
    } else {
      return decrementHP(target);
    }
  };

  var jobe = Immutable.Map({name:"Jobe", hp:20, team: "red"});
  var michael = Immutable.Map({name:"Michael", hp:20, team: "green"});

  punch(jobe, michael);
  //=> Immutable.Map({name:"Michael", hp:19, team: "green"})
```

`decrementHP`, `isSameTeam` and `punch` are all pure and therefore referentially transparent. We can use a technique called *equational reasoning* wherein one substitutes "equals for equals" to reason about code. It's a bit like manually evaluating the code without taking into account the quirks of programmatic evaluation. Using referential transparency, let's play with this code a bit.

First we'll inline the function `isSameTeam`.

```js
  var punch = function(player, target) {
    if(player.team === target.team) {
      return target;
    } else {
      return decrementHP(target);
    }
  };
```

Since our data is immutable, we can simply replace the teams with their actual value

```js
  var punch = function(player, target) {
    if("red" === "green") {
      return target;
    } else {
      return decrementHP(target);
    }
  };
```

We see that it is false in this case so we can remove the entire if branch

```js
  var punch = function(player, target) {
    return decrementHP(target);
  };

```

And if we inline `decrementHP`, we see that, in this case, punch becomes a call to decrement the `hp` by 1.

```js
  var punch = function(player, target) {
    return target.set("hp", target.hp-1);
  };
```

This ability to reason about code is terrific for refactoring and understanding code in general. In fact, we used this technique to refactor our flock of seagulls program. We used equational reasoning to harness the properties of addition and multiplication. Indeed, we'll be using these techniques throughout the book.

### Parallel Code
Finally, and here's the coup de grace, we can run any pure function in parallel since it does not need access to shared memory and it cannot, by definition, have a race condition due to some side effect. This is very much possible in a server side js environment with threads as well as in the browser with web workers though current culture seems to avoid it due to complexity when dealing with impure functions.


## In Summary
We've seen what pure functions are and why we, as functional programmers, believe they are the cat's evening wear. From this point on, we'll strive to write all our functions in a pure way. We'll require some extra tools to help us do so, but in the meantime, we'll try to separate the impure functions from the rest of the pure code.

Writing programs with pure functions is a tad laborious, to say the least, without some extra tools in our belt. We have to juggle data by passing arguments all over the place, we're forbidden to use state, not to mention effects. How does one go about writing these masochistic programs? Let's acquire a new tool called curry.

[Chapter 4: Currying](ch4.md)
