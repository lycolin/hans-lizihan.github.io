---
layout:     post
title:      javascript 表达式
date:       2015-04-24 09:23
summary:    Javascript Syntax expression function
categories: javascript
---

# 表达式

表达式是一门语言的基础 所有的一切都可以理解成是 javascript 的表达式

浏览器中 command + option + J 开启 console, 在 console 中随意打什么都有一个反馈。 这就是表达式所计算出来的值

表达式可以理解成人类语言的一句话 一句话由不同的词语拼接而成， 不同的词由不同的字拼接而成

javascript 中的 字 的概念就是各种 `原始表达式`

## 原始表达式

### 1 常量
* null
* undefined
* 'hello'
...

### 2 变量
* $ (jquery)
* _ (underscore)
...

### 3 初始化
* var obj = {1: 0, 2: 0}
* var arr = [1,2,3,4]
...

### 4 函数定义(下面有更详细的)
* var f = function(){};

### 5 属性访问
* Math.PI

### 6 调用
* alert('hihi');

### 7 创建
* new Object


函数作为整个 javascript 中 __唯一__ 构成 scope 的关键字在整个语言架构中起到了至关重要的做用

## 函数定义

### 1 声明式
``` javascript
function f() {
  // do sth
}
```

其实做了这么几件事
  1. 在当前域返回了 f 的引用。
  2. 在内部域创建了 {func_name:f}
  3. 在当前域找 var f
  4. 找不到 var f 那么一路 __提升__ 浏览器里直到够到 `window` 为止

### 2 表达式
``` javascript
var f = function() {
  // do sth
}
```

### 3 构造函数式
``` javascript
var f = new Function('parameter', 'body');
```

这种写法其实是将一个匿名函数的返回引用赋值给了 `var f`

两种写法大部分时候都可以等价，但是使用的时候有一些要注意

1. 声明式 recursive 的时候直接在自己的内部找到了自身的引用，后者则是在外部找到了引用
2. 声明式 会引发传说中的 `hoist` 也就是一路往上找 所以在当前域中即使在`f`声明之前也可以可以引用 `f`
3. 表达式 则会使 `use strict` 中在 var f = function(){} 之前引用 `f` 抛出 `referece error`


先看个例子

``` javascript
var f = function g(){ console.log(g);};
f();//function g(){ console.log(g);};
typeof g();//g is not defined
```

这个逻辑很绕

1. function g(){} 返回一个函数引用
2. 变量 f 被赋值 于 function g(){} 的引用
3. function g(){} 自身在作用于中创建一个对于自己的引用 {func_name:g}
4. function g(){} 内部调用 g 的时候其实是 call 了自己引用所以把自己 print 出来了
5. 外部作用域看不到 function g() {} 内部的引用
6. 所以 typeof g() 报错

``` javascript
(function f(f){
    return typeof f();// 'number'
})(function(){return 1;});
```

上面这个例子应该比较好理解
1. 立即执行函数 __IIFE__ (Immediately Invoked Function Expression) 得到执行 (line 3)
2. IIF 得到一个匿名函数作为参数 (line 3)
3. IIF 内部执行了参数的匿名函数 (line 2)
4. 匿名函数返回数字 1 (line 2)

唯一需要注意的就是 __当函数执行有命名冲突的时候，函数依次填入 变量 => 函数 => 参数__
1. 对应上面的例子，就是 line 1 函数声明的名字是 `f` 参数的名字也是 `f`
2. 所以在执行的时候 函数 `f` 没有将自己当成是参数传入到参数中，而是将下面的匿名函数传入了进去

``` javascript
var f = 1;
(function f(){
    return typeof f;// 'function'
})();

(function f(f){
    return typeof f; // 'string'
}('hi'))

typeof f // 'number'
```

上面面这个例子看完了应该就明白了

## 表达式返回值

基本类型的返回值不用说了，(基本类型，初始化。。。)

### 1 函数定义表达式
返回的是穿件的 new Function() 类
当用 console.log() 输出的时候默认调用了 Function.prototype.toString() 方法

``` javascript
console.log(function g(){console.log('hi')}) // function g(){console.log('hi')};
```

### 2 函数调用表达式

``` javascript
function(){}(); // undefined
function(){return;}(); // undefined
```

### 3 对象创建

``` javascript
type of new Date(); // object
```

这里坑又来了

js 里面创建一个对象是用 function 关键字的

``` javascript
function Test(){
    return new Date();
}
var test = new Test();
console.log(test instanceof Test);//false
console.log(test);//Fri Apr 24 2015 12:59:22 GMT+0800 (HKT)
```

上面代码我们可以看到
1. 创建 constructor `Test`
2. 调用 constructor
3. 期待一个 new 出来的 `Function`
4. 得到了一个 new 出来的 `Date`

为什么？

> 当使用function的构造函数创建对象（new XXX）的时候，如果函数return基本类型或者没有return（其实就是return undefined）的时候， new 返回的是对象的实例；如果 函数return的是一个对象，那么new 将返回这个对象而不是函数实例

所以

``` javascript
'foo' == new function(){ return String('foo'); }; //false
'foo' == new function(){ return new String('foo'); };//true
```

所以 angular.js 中

``` javascript
app.service('foo', function() {
  var thisIsPrivate = "Private";
  this.variable = "This is public";
  this.getPrivate = function() {
    return thisIsPrivate;
  };
});

// 等价于
app.factory('foo2', function() {
  return new Foobar();
});

// 等价于
app.service('foo3', Foobar);

// 等价于
app.factory('foo4', function() {
  var thisIsPrivate = "Private";

  return {
    variable: "This is public";
    getPrivate: function () {
      return thisIsPrivate;
    }
  }
})


function Foobar() {
  var thisIsPrivate = "Private";
  this.variable = "This is public";
  this.getPrivate = function() {
    return thisIsPrivate;
  };
}
```

可以看到 当调用 new factory() 的时候内部将会返回一个 object 此 object 则是 ng 中用到的单例 而不是 factory 类本身

参考:
  * [JavaScript面试时候的坑洼沟洄——表达式与运算符](http://www.cnblogs.com/dolphinX/p/3524977.html)

happy coding, may the code will always be with you~
