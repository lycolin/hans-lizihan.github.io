---
layout:     post
title:      javascript 变量 作用域(2)
date:       2015-05-22 00:37
summary:    javascript 基础篇之 变量 作用域(2)
categories: javascript
---

之前小小看了下 js 的各种变量作用域专题，但是还没有看完全。今天特地补全之前没明白完全的空白。

## EC ECS 重访

最底层的时候所有的 javascript 都会进入一个全局的 EC, GlobalContext, 这个 EC 比较特殊，在 js 引擎激活的时候就创建，在用户关闭窗口 （浏览器）或者当前进程结束的时候销毁。

初始的时候 ECS 将会是一个空的 数组 来实现一个 stack 的功能

最初的 EC 最凶残的一点就是在于它本身是一个 VO

``` javascript
globalContext === VO(globalContext);
```

此时ECS 就是 globalContext 垫底了。

``` javascript
ECStack = [globalContext];
```

举个栗子就是这样的

``` javascript
(function foo(bar) {
  if(bar) return;
  foo(true);
})();
```

foo 被 IIFE 包裹，所以直接就运行了。

``` javascript
// 第一次 IIFE 执行
ECStack = [
  <foo> functionContext, 
  globalContext
];

// 第二次 递归调用
ECStack = [
  <foo> functionContext,
  <foo> functionContext,
  globalContext
];

// 遇到 return
ECStack = [
  <foo> functionContext,
  globalContext
];

// 没有 return，函数执行完毕
ECStack = [
  globalContext
];
```

在 eval 函数中会讲当前调用 eval 函数的 环境算成是 `调用环境`(callingContext)

``` javascript
eval('var a = 1');
```

这个时候 eval 内部的所有作用域只会影响 callingContext

``` javascript
ECStack = [
  evalContext,
  callingContext: globalContext
];
```

## 引用类型重访

对于应用类型的值大致有这么两个东西的存在

``` javascript
var valueOfReferenceType  = {
  base: <base object>,
  propertyName: <propertyName>
}
```

对于一个应用类型的值分两种情况而不同

1. 叫一个 identifier
2. 叫一个 object 的 accessor

### 1 identifier

``` javascript
function foo() {};
var bar = 10;
```

他们俩的引用类型是这样的

``` javascript
var fooReference = {
  base: global,
  propertyName: 'foo'
}

var barReference = {
  base: global,
  prototypeName: 'bar'
}
```

对于任何值有一个伪代码可以取得它的 value

``` javascript
function GetValue() {
  // 如果是 primitive 则直接返回
  if(Type(value) != Reference) {
    return value;
  }

  var base = GetBase(value);

  if(base === null) {
    throw new ReferenceError;
  }

  return base.[[Get]](GetPropertyName(value));
}
```

注意这里面 [[Get]] 就是对象的 accessor 方法

总结来说就是 identifier base 的 call 中的 reference base 是当前 call 这个函数的 caller

### 2 object accessor

``` javascript
foo.bar();
```

这么叫得话 foo.bar 的 reference 是这样的

``` javascript
fooBarReference = {
  base: foo,
  prototypeName: 'bar'
}
```

## this

总的来说影响 this 的作用域的东西有几个, 它直接相关上面提到的几点 reference object 的内部构造。

> 在一个函数环境中，this由调用者提供，由调用函数的方式来决定。如果调用括号()的左边是引用类型的值，this将设为引用类型值的base对象（base object），在其他情况下（与引用类型不同的任何其它属性），这个值为null。不过，实际不存在this的值为null的情况，因为当this的值为null的时候，其值会被隐式转换为全局对象。 p.s. ES5 中赋值为 undefined

太特么晦涩了

### 引用 类型在() 左边

``` javascript
function foo() {
  return this;
}

foo(); // global
```

() 左边是一个引用类型的值，(因为 foo 是一个 identifier)

``` javascript
fooReference {
  base: global,
  prototypeName: 'foo'
}
```

所以 this 直接指向了 base 这个属性

``` javascript
function foo() {
  bar: fucntion () {
    return this;
  }
}

foo.bar(); // foo
```

解释通了

``` javascript
fooBarReference = {
  base: foo,
  prototypeName: 'bar'
}
```

但是赋值之后再激活结果又不一样了。

``` javascript
function foo() {
  bar: function() {
    return this;
  }
}

var test = foo.bar;
test(); // global
```

这是因为 test 中的 base 又悄然被改变了

``` javascript
testReference = {
  base: global,
  prototypeName: 'test'
}
```

这就可以解释通很多怪癖了。

### 非引用类型在 () 左边

``` javascript
(function() {
  'use strict';
  console.log(this); // undefined
})();
```

``` javascript
var foo = {
  bar: function () {
    console.log(this);
  }
};
foo.bar(); // foo
(foo.bar)(); // foo
(foo.bar = foo.bar)(); // foo
(false || foo.bar)(); // undefined
(foo.bar, foo.bar)(); // undefined
```

这里要明白的一点就是各种运算符的返回结果都会调用 GetValue 伪代码，而不是返回赋值的引用本身。所以对于这种非引用类型，加一个括号在右边导致了内部的 this 被设置成了 undefined


### 函数中的调用

``` javascript
function foo() {
  function bar () {
    console.log(this); // foo
  }
  bar(); // AO.bar();
}
```

### with 语句的调用

``` javascript
var x = 10;
 
with ({
 
  foo: function () {
    alert(this.x);
  },
  x: 20
 
}) {
 
  foo(); // 20
 
}
 
// because
 
var  fooReference = {
  base: __withObject,
  propertyName: 'foo'
};
```

因为在引用中 __withObject 的调用比上一层的AO还要靠前，所以优先在 withObject 里面寻找属性。

### catch 块

``` javascript
function test() {
  function error() {
    throw new Error('alert');
  }

  try{
    error();
  } catch (e) {
    console.log(this); // window
  }
}

test();

var eReference = {
  base: global,
  propertyName: 'e'
};
```

直接活在 global 里面


### 构造函数调用

``` javascript
function A() {
  console(this); // {}
  this.x = 10;
}

var a = new A(); 
console.log(a.x); // 10
```

1. new运算符调用 A 函数的内部的[[Construct]] 方法创建一个新的对象
2. A 调用内部的[[Call]] 方法。 将this的值设置为新创建的对象

### 手动 binding

.bind(this), .call(this), .apply(this) 都可以告诉那个函数中的 this 是什么东西.

参考: 

* [tom 大叔 this 精讲](http://www.cnblogs.com/TomXu/archive/2012/01/17/2310479.html)
