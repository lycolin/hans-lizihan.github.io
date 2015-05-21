---
layout:     post
title:      javascript 模块模式
date:       2015-05-21 15:48
summary:    javascript 设计模式 模块模式
categories: javascript
---

这两天在看 coffeescript 的时候看到 coffee 里面的 `class` 实现编译出来之后是这样的:

``` javascript
Person = (function() {
  Person.name = 'Person';

  function Person(first_name, last_name) {
    this.first_name = first_name;
    this.last_name = last_name;
  }

  Person.prototype.fullname = function () {
    return this.first_name + ' ' + this.last_name;
  };

  return Person;
})();
```

这就是传说中的模块模式了。

我们看到 `constructor` 被放在了一个 iife 里面，所以在 iife 里面我们就可以定义一个 `constructor` 然后各种操作之后返回这个函数的引用了。

感觉这个用法之前再用 angularjs 的时候 `factory` 一直在用。这种模式的好处就是可以有选择地暴露一些想要暴露的公共方法给外面，然后封装一些私有方法在里面。

除此之外这种模式还有一个非常酷炫的特性就是可以将全局变量 Pass 进去。并且自己规定它的 alias

``` javascript
var jQueryPlugin = (function($) {
  // jQuery logic here
})(window.jQuery)
```

很短但是豁然开朗啊

参考:

* [全面解析Module模式](http://www.cnblogs.com/TomXu/archive/2011/12/30/2288372.html)