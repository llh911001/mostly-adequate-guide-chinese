# 第 2 章: 一等公民的函数

## 快速概览

当我们说函数是“一等公民”的时候，我们实际上说的是它们和其他对象都一样...所以就是普通公民（坐经济舱的人？）。函数真没什么特殊的，你可以像对待任何其他数据类型一样对待它们——把它们存在数组里，当作参数传递，赋值给变量...等等。

这是 JavaScript 语言的基础概念，不过还是值得提一提的，因为在 Github 上随便一搜就能看到对这个概念的集体无视，或者也可能是无知。我们来看一个杜撰的例子：

```js
var hi = function(name){
  return "Hi " + name;
};

var greeting = function(name) {
  return hi(name);
};
```

这里 `greeting` 指向的那个把 `hi` 包了一层的包裹函数完全是多余的。为什么？因为 JavaScript 的函数是*可调用*的，当 `hi` 后面紧跟 `()` 的时候就会运行并返回一个值；如果没有 `()`，`hi` 就简单地返回存到这个变量里的函数。我们来确认一下：

```js
hi;
// function(name){
//  return "Hi " + name
// }

hi("jonas");
// "Hi jonas"
```

`greeting` 只不过是转了个身然后以相同的参数调用了 `hi` 函数而已，因此我们可以这么写：

```js
var greeting = hi;


greeting("times");
// "Hi times"
```

换句话说，`hi` 已经是个接受一个参数的函数了，为何要再定义一个额外的包裹函数，而它仅仅是用这个相同的参数调用 `hi`？完全没有道理。这就像在大夏天里穿上你最厚的大衣，只是为了跟热空气过不去，然后吃上个冰棍。真是脱裤子放屁多此一举。

用一个函数把另一个函数包起来，目的仅仅是延迟执行，真的是非常糟糕的编程习惯。（稍后我将告诉你原因，跟可维护性密切相关。）

充分理解这个问题对读懂本书后面的内容至关重要，所以我们再来看几个例子。以下代码都来自 npm 上的模块包：

```js
// 太傻了
var getServerStuff = function(callback){
  return ajaxCall(function(json){
    return callback(json);
  });
};

// 这才像样
var getServerStuff = ajaxCall;
```

世界上到处都充斥着这样的垃圾 ajax 代码。以下是上述两种写法等价的原因：

```js
// 这行
return ajaxCall(function(json){
  return callback(json);
});

// 等价于这行
return ajaxCall(callback);

// 那么，重构下 getServerStuff
var getServerStuff = function(callback){
  return ajaxCall(callback);
};

// ...就等于
var getServerStuff = ajaxCall; // <-- 看，没有括号哦
```

各位，以上才是写函数的正确方式。一会儿再告诉你为何我对此如此执着。

```js
var BlogController = (function() {
  var index = function(posts) {
    return Views.index(posts);
  };

  var show = function(post) {
    return Views.show(post);
  };

  var create = function(attrs) {
    return Db.create(attrs);
  };

  var update = function(post, attrs) {
    return Db.update(post, attrs);
  };

  var destroy = function(post) {
    return Db.destroy(post);
  };

  return {index: index, show: show, create: create, update: update, destroy: destroy};
})();
```

这个可笑的控制器（controller）99% 的代码都是垃圾。我们可以把它重写成这样：

```js
var BlogController = {index: Views.index, show: Views.show, create: Db.create, update: Db.update, destroy: Db.destroy};
```

...或者直接全部删掉，因为它的作用仅仅就是把视图（Views）和数据库（Db）打包在一起而已。

## 为何钟爱一等公民？

好了，现在我们来看看钟爱一等公民的原因是什么。前面 `getServerStuff` 和 `BlogController` 两个例子你也都看到了，虽说添加一些没有实际用处的间接层实现起来很容易，但这样做除了徒增代码量，提高维护和检索代码的成本外，没有任何用处。

另外，如果一个函数被不必要地包裹起来了，而且发生了改动，那么包裹它的那个函数也要做相应的变更。

```js
httpGet('/post/2', function(json){
  return renderPost(json);
});
```

如果 `httpGet` 要改成可以抛出一个可能出现的 `err` 异常，那我们还要回过头去把“胶水”函数也改了。

```js
// 把整个应用里的所有 httpGet 调用都改成这样，可以传递 err 参数。
httpGet('/post/2', function(json, err){
  return renderPost(json, err);
});
```

写成一等公民函数的形式，要做的改动将会少得多：

```js
httpGet('/post/2', renderPost);  // renderPost 将会在 httpGet 中调用，想要多少参数都行
```

除了删除不必要的函数，正确地为参数命名也必不可少。当然命名不是什么大问题，但还是有可能存在一些不当的命名，尤其随着代码量的增长以及需求的变更，这种可能性也会增加。

项目中常见的一种造成混淆的原因是，针对同一个概念使用不同的命名。还有通用代码的问题。比如，下面这两个函数做的事情一模一样，但后一个就显得更加通用，可重用性也更高：

```js
// 只针对当前的博客
var validArticles = function(articles) {
  return articles.filter(function(article){
    return article !== null && article !== undefined;
  });
};

// 对未来的项目友好太多
var compact = function(xs) {
  return xs.filter(function(x) {
    return x !== null && x !== undefined;
  });
};
```

在命名的时候，我们特别容易把自己限定在特定的数据上（本例中是 `articles`）。这种现象很常见，也是重复造轮子的一大原因。

有一点我必须得指出，你一定要非常小心 `this` 值，别让它反咬你一口，这一点与面向对象代码类似。如果一个底层函数使用了 `this`，而且是以一等公民的方式被调用的，那你就等着 JS 这个蹩脚的抽象概念发怒吧。

```js
var fs = require('fs');

// 太可怕了
fs.readFile('freaky_friday.txt', Db.save);

// 好一点点
fs.readFile('freaky_friday.txt', Db.save.bind(Db));

```

把 Db 绑定（bind）到它自己身上以后，你就可以随心所欲地调用它的原型链式垃圾代码了。`this` 就像一块脏尿布，我尽可能地避免使用它，因为在函数式编程中根本用不到它。然而，在使用其他的类库时，你却不得不向这个疯狂的世界低头。

也有人反驳说 `this` 能提高执行速度。如果你是这种对速度吹毛求疵的人，那你还是合上这本书吧。要是没法退货退款，也许你可以去换一本更入门的书来读。

至此，我们才准备好继续后面的章节。

[第 3 章: 纯函数的好处](ch3.md)
