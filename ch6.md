# 第 6 章: 示例应用

## 声明式代码

我们要开始转变观念了，从本章开始，我们将不再指示计算机如何工作，而是指出我们明确希望得到的结果。我敢保证，这种做法与那种需要时刻关心所有细节的命令式编程相比，会让你轻松许多。

与命令式不同，声明式意味着我们要写表达式，而不是一步一步的指示。

以 SQL 为例，它就没有“先做这个，再做那个”的命令，有的只是一个指明我们想要从数据库取什么数据的表达式。至于如何取数据则是由它自己决定的。以后数据库升级也好，SQL 引擎优化也好，根本不需要更改查询语句。这是因为，有多种方式解析一个表达式并得到相同的结果。

对包括我在内的一些人来说，一开始是不太容易理解“声明式”这个概念的；所以让我们写几个例子找找感觉。

```js
// 命令式
var makes = [];
for (i = 0; i < cars.length; i++) {
  makes.push(cars[i].make);
}


// 声明式
var makes = cars.map(function(car){ return car.make; });
```

命令式的循环要求你必须先实例化一个数组，而且执行完这个实例化语句之后，解释器才继续执行后面的代码。然后再直接迭代 `cars` 列表，手动增加计数器，把各种零零散散的东西都展示出来...实在是直白得有些露骨。

使用 `map` 的版本是一个表达式，它对执行顺序没有要求。而且，`map` 函数如何进行迭代，返回的数组如何收集，都有很大的自由度。它指明的是`做什么`，不是`怎么做`。因此，它是正儿八经的声明式代码。

除了更加清晰和简洁之外，`map` 函数还可以进一步优化，这么一来我们宝贵的应用代码就无须改动了。

至于那些说“虽然如此，但使用命令式循环速度要快很多”的人，我建议你们先去学学 JIT 优化代码的相关知识。这里有一个[非常棒的视频](https://www.youtube.com/watch?v=65-RbBwZQdU)，可能会对你有帮助。

再看一个例子。

```js
// 命令式
var authenticate = function(form) {
  var user = toUser(form);
  return logIn(user);
};

// 声明式
var authenticate = compose(logIn, toUser);
```

虽然命令式的版本并不一定就是错的，但还是硬编码了那种一步接一步的执行方式。而 `compose` 表达式只是简单地指出了这样一个事实：用户验证是 `toUser` 和 `logIn` 两个行为的组合。这再次说明，声明式为潜在的代码更新提供了支持，使得我们的应用代码成为了一种高级规范（high level specification）。

因为声明式代码不指定执行顺序，所以它天然地适合进行并行运算。它与纯函数一起解释了为何函数式编程是未来并行计算的一个不错选择——我们真的不需要做什么就能实现一个并行／并发系统。

## 一个函数式的 flickr

现在我们以一种声明式的、可组合的方式创建一个示例应用。暂时我们还是会作点小弊，使用副作用；但我们会把副作用的程度降到最低，让它们与纯函数代码分离开来。这个示例应用是一个浏览器 widget，功能是从 flickr 获取图片并在页面上展示。我们从写 html 开始：

```html
<!DOCTYPE html>
<html>
  <head>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/require.js/2.1.11/require.min.js"></script>
    <script src="flickr.js"></script>
  </head>
  <body></body>
</html>
```

flickr.js 如下：

```js
requirejs.config({
  paths: {
    ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
    jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min'
  }
});

require([
    'ramda',
    'jquery'
  ],
  function (_, $) {
    var trace = _.curry(function(tag, x) {
      console.log(tag, x);
      return x;
    });
    // app goes here
  });
```

这里我们使用了 [ramda](http://ramdajs.com) ，没有用 lodash 或者其他类库。ramda 提供了 `compose`、`curry` 等很多函数。模块加载我们选择的是 requirejs，我以前用过 requirejs，虽然它有些重，但为了保持一致性，本书将一直使用它。另外，我也把 `trace` 函数写好了，便于 debug。

有点跑题了。言归正传，我们的应用将做 4 件事：

1. 根据特定搜索关键字构造 url
2. 向 flickr 发送 api 请求
3. 把返回的 json 转为 html 图片
4. 把图片放到屏幕上

注意到没？上面提到了两个不纯的动作，即从 flickr 的 api 获取数据和在屏幕上放置图片这两件事。我们先来定义这两个动作，这样就能隔离它们了。

```js
var Impure = {
  getJSON: _.curry(function(callback, url) {
    $.getJSON(url, callback);
  }),

  setHtml: _.curry(function(sel, html) {
    $(sel).html(html);
  })
};
```

这里只是简单地包装了一下 jQuery 的 `getJSON` 方法，把它变为一个 curry 函数，还有就是把参数位置也调换了下。这些方法都在 `Impure` 命名空间下，这样我们就知道它们都是危险函数。在后面的例子中，我们会把这两个函数变纯。

下一步是构造 url 传给 `Impure.getJSON` 函数。

```js
var url = function (term) {
  return 'https://api.flickr.com/services/feeds/photos_public.gne?tags=' + term + '&format=json&jsoncallback=?';
};
```

借助 monoid 或 combinator （后面会讲到这些概念），我们可以使用一些奇技淫巧来让 `url` 函数变为 pointfree 函数。但是为了可读性，我们还是选择以普通的非 pointfree 的方式拼接字符串。

让我们写一个 `app` 函数发送请求并把内容放置到屏幕上。

```js
var app = _.compose(Impure.getJSON(trace("response")), url);

app("cats");
```

这会调用 `url` 函数，然后把字符串传给 `getJSON` 函数。`getJSON` 已经局部应用了 `trace`，加载这个应用将会把请求的响应显示在 console 里。

<img src="images/console_ss.png"/>

我们想要从这个 json 里构造图片，看起来 src 都在 `items` 数组中的每个 `media` 对象的 `m` 属性上。

不管怎样，我们可以使用 ramda 的一个通用 getter 函数 `_.prop()` 来获取这些嵌套的属性。不过为了让你明白这个函数做了什么事情，我们自己实现一个 prop 看看：

```js
var prop = _.curry(function(property, object){
  return object[property];
});
```

实际上这有点傻，仅仅是用 `[]` 来获取一个对象的属性而已。让我们利用这个函数获取图片的 src。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));
```

一旦得到了 `items`，就必须使用 `map` 来分解每一个 url；这样就得到了一个包含所有 src 的数组。把它和 `app` 联结起来，打印结果看看。

```js
var renderImages = _.compose(Impure.setHtml("body"), srcs);
var app = _.compose(Impure.getJSON(renderImages), url);
```

这里所做的只不过是新建了一个组合，这个组合会调用 `srcs` 函数，并把返回结果设置为 body 的 html。我们也把 `trace` 替换为了 `renderImages`，因为已经有了除原始 json 以外的数据。这将会粗暴地把所有的 src 直接显示在屏幕上。

最后一步是把这些 src 变为真正的图片。对大型点的应用来说，是应该使用类似 Handlebars 或者 React 这样的 template/dom 库来做这件事的。但我们这个应用太小了，只需要一个 img 标签，所以用 jQuery 就好了。

```js
var img = function (url) {
  return $('<img />', { src: url });
};
```

jQuery 的 `html()` 方法接受标签数组为参数，所以我们只须把 src 转换为 img 标签然后传给 `setHtml` 即可。

```js
var images = _.compose(_.map(img), srcs);
var renderImages = _.compose(Impure.setHtml("body"), images);
var app = _.compose(Impure.getJSON(renderImages), url);
```

任务完成！

<img src="images/cats_ss.png" />

下面是完整代码：

```js
requirejs.config({
  paths: {
    ramda: 'https://cdnjs.cloudflare.com/ajax/libs/ramda/0.13.0/ramda.min',
    jquery: 'https://ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min'
  }
});

require([
    'ramda',
    'jquery'
  ],
  function (_, $) {
    ////////////////////////////////////////////
    // Utils

    var Impure = {
      getJSON: _.curry(function(callback, url) {
        $.getJSON(url, callback);
      }),

      setHtml: _.curry(function(sel, html) {
        $(sel).html(html);
      })
    };

    var img = function (url) {
      return $('<img />', { src: url });
    };

    var trace = _.curry(function(tag, x) {
      console.log(tag, x);
      return x;
    });

    ////////////////////////////////////////////

    var url = function (t) {
      return 'https://api.flickr.com/services/feeds/photos_public.gne?tags=' + t + '&format=json&jsoncallback=?';
    };

    var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

    var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

    var images = _.compose(_.map(img), srcs);

    var renderImages = _.compose(Impure.setHtml("body"), images);

    var app = _.compose(Impure.getJSON(renderImages), url);

    app("cats");
  });
```

看看，多么美妙的声明式规范啊，只说做什么，不说怎么做。现在我们可以把每一行代码都视作一个等式，变量名所代表的属性就是等式的含义。我们可以利用这些属性去推导分析和重构这个应用。

## 有原则的重构

上面的代码是有优化空间的——我们获取 url map 了一次，把这些 url 变为 img 标签又 map 了一次。关于 map 和组合是有定律的：

```js
// map 的组合律
var law = compose(map(f), map(g)) == map(compose(f, g));
```

我们可以利用这个定律优化代码，进行一次有原则的重构。

```js
// 原有代码
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var srcs = _.compose(_.map(mediaUrl), _.prop('items'));

var images = _.compose(_.map(img), srcs);

```

感谢等式推导（equational reasoning）及纯函数的特性，我们可以内联调用 `srcs` 和 `images`，也就是把 map 调用排列起来。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(img), _.map(mediaUrl), _.prop('items'));
```

把 `map` 排成一列之后就可以应用组合律了。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var images = _.compose(_.map(_.compose(img, mediaUrl)), _.prop('items'));
```

现在只需要循环一次就可以把每一个对象都转为 img 标签了。我们把 map 调用的 compose 取出来放到外面，提高一下可读性。

```js
var mediaUrl = _.compose(_.prop('m'), _.prop('media'));

var mediaToImg = _.compose(img, mediaUrl);

var images = _.compose(_.map(mediaToImg), _.prop('items'));
```

## 总结

我们已经见识到如何在一个小而不失真实的应用中运用新技能了，也已经使用过函数式这个“数学框架”来推导和重构代码了。但是异常处理以及代码分支呢？如何让整个应用都是函数式的，而不仅仅是把破坏性的函数放到命名空间下？如何让应用更安全更富有表现力？这些都是本书第 2 部分将要解决的问题。

[第 7 章: Hindley-Milner 类型签名](ch7.md)
