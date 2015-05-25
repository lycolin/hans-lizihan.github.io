---
layout:     post
title:      javascript 变量 作用域(3)
date:       2015-05-22 20:50
summary:    javascript 基础篇之 函数 作用域链 闭包
categories: javascript
---

## 重返 作用域链

> 作用域链与一个执行环境相关，变量对象的链用于在标识符解析中变量查找。

``` javascript
activeExcecutionContext = {
  VO: {},
  this: thisValue,
  Scope: [ //  作用域链
    // 所有VO 的链表
  ]
}
```

Scope = [AO].concat([[Scope]])

### 函数生命周期

说到了 作用域链就必须仔细看看函数的生命周期了。

``` javascript
var x = 10;

function foo() {
  var y = 20;
  console.log(x + y); // 30
}

foo(); // 30
```

这里我们是知道答案的。因为在当前域里面找不到 x 那么就应该去上一层找 x 嘛，这是因为啥? 

``` javascript
fooContext = {
  AO: {
    y: undefined
  },
  Scope: [
    globalContext
  ]
}
```

### 函数激活

像刚开始解释的一样，当一个函数进入执行环境的时候其 Scope会发生这样的改变

``` javascript
Scope = [AO].concat([[Scope]]);
```

举这样一个例子就知道了

``` javascript
var x = 10;

function foo() {
  var y = 20;

  function bar() {
    var z = 30;
    console.log(x + y + z);
  }

  bar();
}

foo(); // 60
```

全局中是这样的

``` javascript
globalContext.VO === Global = {
  x: 10
  foo: <reference to function>
};
```

foo 刚刚创建的时候是这样的

``` javascript
foo.[[Scope]] = [
  globalContext.VO
];
```

foo 刚刚激活的时候是这样的

``` javascript
fooContext.AO = {
  y: 20,
  bar: <reference to function>
}

fooContext.Scope = {
  fooContext.AO,
  globalContext.VO
}
```

bar 刚刚创建的时候是这样的

``` javascript
bar.[[Scope]] = {
  fooContext.AO,
  globalContext.VO
}
```

bar 激活的时候是这样的

``` javascript
barContext.AO = {
  z: 30
};

barContext.Scope = {
  barContext.AO,
  fooContext.AO,
  globalContext.VO
}
```

所以最后的作用域链就是 barContext.Scope 了。当一个变量在 bar 的 AO 中找不到的时候就会一路往上找知道 globalContext.VO 为止。还找不到就是 undefined 了，但是如果找到就返回那个找到的 value 了。

### [[Scope]]

最后重申一下就是当一个函数最初创建的时候 (可以理解为解释器看到了 function 关键字的时候)，它的 [[Scope]] 这个属性就已经确定了，而且不可更改。

## 闭包

``` javascript
var x = 10;

function foo() {
  console(x);
}

(function() {
  var x = 20;
  foo(); // 10, not 20
})();
```

现在再看这个例子就不奇怪了。 因为对于 `foo` 来说，它被创建出来的时候它的 [[Scope]] 就已经固定了。所以在执行的时候

1. foo 创造了 fooContext， AO 中啥都没有。 
2. fooContext.Scope = [{}].concat(globalContext.VO);
3. console.log(x) 回到全局作用域找到了 x
4. 输出 10

闭包中的经典问题大概是这样的

``` javascript
data = [];
for(var i = 0; i < 3; i ++) {
  data[i] = function() {
    console.log(i);
  }
}

data[0](); // 3
data[1](); // 3
data[2](); // 3
```

这是为啥？

这是因为虽然我们赋值了三个不同的引用给三个不同的 data 位置，但是事实上

data[0].[[Scope]] === data[1].[[Scope]] === data[2].[[Scope]]

而这个 Scope 中的 AO 因为在执行完 loop 之后就已经变成了这样

``` javascript
AO: {
  ...,
  i: 3
};
```

而三个匿名函数中的 AO 都没有 i 的存在，所以必须要找他们的 AO.next 才找到了 i, 所以每次都是 print 出来 3

解决方案也很简单

``` javascript
var data = [];
for(var i = 0; i < 3; i ++) {
  data[i] = (function _helper(i) {
    return function() {
      console.log(i);
    }
  })(i);
}

data[0](); // 0
data[1](); // 1
data[2](); // 2
```

在外面包裹一层，然后将 i 作为参数传入外层的匿名函数。这是外层的 `_helper` 函数中就会出现这样的 AO

``` javascript
AO: {
  ...,
  i: 0/1/2
}
```

内层的匿名函数，也就是返回的 data[0/1/2].[[Scope]] 会指向三个不同的`_helper` 函数，他们中 AO 的 i 值分别是 0/1/2 

所以最后的结果就对了。

这样做唯一的坏处就是多用了一层函数的内存，而且只要 data 这个变量没有被 赋值给 null，那么三个 `_helper` 函数的内存就永远不会被释放。

参考:

* [汤姆大叔闭包精讲](http://www.cnblogs.com/TomXu/archive/2012/01/17/2310479.html)
* *javascript 高级程序设计*
