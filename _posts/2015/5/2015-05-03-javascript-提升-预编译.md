---
layout:     post
title:      javascript 预编译
date:       2015-05-03 15:35
summary:    javascript 基础篇之 预编译
categories: javascript
---

# js 解释原理

js 是解释性语言

整个 js 的运行原理是

1. 语法检查
	1. 词法分析
	2. 语法分析
2. 运行时
	1. 预编译
	2. 运行

所以没一段 js 的表达式 或者表达式组合 都会进过这样几个 过程

例如

``` javascript
0 = 1 = 2;
// ReferenceError: Invalid left-hand side in assignment
```

这个错误实在语法检查阶段抛出来的，所以程序根本不会执行


``` javascript
a = b = c;
// ReferenceError: c is not defined
```

而这个则是在运行时被跑出来的

## 预编译

这个阶段让人又爱又恨，因为很多非常坑爹的结果会跑来。

预编译做的两件事

1. 将一段 scope 中的 var 放进 栈(stack) 中并赋值 `undefined`
2. 读入 `声明式` 函数


``` javascript
// 扫描整段代码，看到 var a 并赋值 undefined
alert(a); // undefined
var a = 1; // a 赋值为 a
```

上面的例子比价好地说明了变量预编译的流程

``` javascript
a(); // hihi
function a() {
	alert('hihi');
}
var a = function() {
	alert('heihei');
}
a(); // tom
```

1. 看到 var a，将 `a` 放入栈 赋值为 `undefined`
2. 看到声明 a `a` a 从此变成了一个函数定义
3. 执行 第一个 `a()` 弹出 `hihi`
4. 赋值新的 `heihei` 给 `a`
5. 执行 `a()` 弹出 `heihei`

总结一下就是这样的

``` javascript
function a() {} // 预编译定义，运行时略过
var a = function() {} // 预编译声明，运行时赋值
a = function() {} // 运行时变量赋值
```

所以看个例子

``` javascript
a(); // 2
function a() {
	alert(1);
}

a(); // 2

function a() {
	alert(2);
}

a(); // 2

var a = function () {
	alert(3);
}

a(); // 3
```

有了前面的解释理解整段代码的输出就不难了

因为所有的 `声明式` 函数在预编译的过程就已经被定义了，并且后来声明的函数会重载之前声明的函数

所以运行时的时候两个代码块 `function a(){}` 并没有影响运行时的权利, 只有 `var a = **` 之后 a 的引用才被修改

以此引出了一些常见使用的最佳实践

### 1 判断变量是否存在

``` javascript
// bad
// if a is not defined via `var`, an error is thrown
a === undefined

// good
typeof a === 'undefined'
```

### 2. 变量提升

由于预编译的性质，作用域内的所有 `var` 和 `声明式` 函数可以被等同看做在作用域首出现

``` javascript
function () {
	console.log(b); // undefined
	if(false) var b = 1;
}

// is same as

function () {
	var b;
	console.log(b); // undefined
	if(false) b = 1; // 不会被执行到
}
```

``` javascript
function global() {
	oops = 'hihi';
}

function local() {
	var oops = 'heihei';
}

console.log(window.oops); // undefined

global();

console.log(window.oops); // 'hihi'

local();

console.log(window.oops); // 'hihi';
```

在作用域中如果没有用 `var` 关键字定义变量，那么就会出现 `修改全局变量` 这种坑爹的情况

因为在 js 世界中有这么一条规则

> 非严格模式下，若果给一个未声明的变量赋值，那么实际上 javascript 会给全局变量创建一个 同名属性

所以在现实世界中经常可以看到 `use strict` 即严格模式的强制实行 (jshint 等工具)

最佳实践是这样的

``` javascript
function a() {  
	// bad: lack of strict mode, may cause untraceable errors
	// logic here
}

function b() {
	// good, always specific strict mode in the first level of global scope
	'use strict';
}


function c() {
	// bad, directly variable assignment without `var` keyword
	someVar = 'ihih';
}

function d() {
	'use strict';
	// good always use strict mode and `var` keyword
	var someVar = 'hihi';
}
```

此外对于 'use strict' 的使用的地点也有讲究。 通常不推荐在全局变量域中加入 'use strict', 因为这么做可能会导致一些没有套用严格模式的js库崩溃。

为了充分解耦，推荐在各个 `IIFE` 中使用 'use strict' 来使用严格模式


``` javascript
// window scope

// bad: use strict mode in the window scope, may cause unexpected
// errors in those packages not following strict mode
'use strict';

// good: use function level of strict mode with IIFE
(function () {
	'use strict';
	function f(){};
	function g(){};
	...
})();
```

happy coding, may the code will always be with you~
