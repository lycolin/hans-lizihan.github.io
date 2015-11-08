---
layout:     post
title:      javascript 逗号，冒号，括号
date:       2015-04-29 21:55
summary:    javascript 基础篇之 逗号，冒号，括号
categories: javascript
---

# 语言基本元素 `,` `:` `()`

这三种符号简直熟得不能再熟了，但是依然有一些知识死角

## `,` 逗号

### 1 同时声明多个变量

``` javascript
var a=1,
  b=2,
  c=3;
```

注意上面这种声明变量的方法比较简约，但是却是有不少的问题。

1. 如果不小心加了分号，那么很难找出来哪里错了
2. 声明的时候有机会不小心加了分号
3. 全部用 `var x = xxx;` 的 形式更加统一简单

github 上面 airbnb 团队总结出来的一套东西很有说服力 (airbnb 似乎已经进军 es6 了，汗颜啊。。。)

[https://github.com/airbnb/javascript/tree/master/es5#variables](https://github.com/airbnb/javascript/tree/master/es5#variables)

### 2 分割方法参数

``` javascript
function(arg1, arg2) {

}
```

### 3 对象,数组 属性分割

``` javascript
var a = {
  a: 1,
  b: 2
}
```

### 4 作为运算符

> 忽略第一个操作数，返回第二个操作数

> 左结合

``` javascript
var a=(1,2,3);
// return undefined (新建变量赋值返回 undefined)
// a = 3

a=1,2; // return 2
// a = 1

var a,b;
a=(b=1,2);
console.log(a);//2
console.log(b);//1
```

这个见得很少，但是遇见了就是坑

`,` 是众多运算符中优先级 __最低__ 的所以最后执行

所以上面源代码的理解是这样的

``` javascript
(1,2,3)=((1,2),3)
=> (2,3)
=> 3

a=1,2 //(a=1),2
// 返回 2

a=(b=1,2)
//1 (b=1)
//2 ((b=1),2) 返回2
//3 a=2
```

## `:` 冒号

### 1 ?: 运算符

``` javascript
var a = true ? 'hihi': 'nono';
```

### 2 对象 value

``` javascript
var obj = {
  'a' : 1,
  'b' : 1
}
```

### 3 switch

``` javascript
switch(condition) {
  case 1:
    console.log('hihi');
    break;
  case 2:
    console.log('nono');
    break;
  default:
    break;
}
```

### 4 声明 label

这个功能因为在就作为语言糟粕被基本忽略了，想想 `goto` 带来的痛苦 label 带来的一点点便利性远小于对代码的可读性摧毁性的打击

``` javascript
x:y:z:1,2,3 // return 3

///
x:
  y:
    z: 1,2,3
```

``` javascript
var x=1;
foo:{
  x=2;
  break foo;
  x=3;
}
console.log(x); // 2
```

## 大括号

### 1 对象的直接声明

``` javascript
var a = {
  'a': 1,
  'b': 2
}
```

### 2 函数的声明或者直接量

``` javascript
function f1() {

}

var f2 = function() {

}
```

### 3 组合复合语句

``` javascript
for() {

}

if() {

} else if() {

}
```

这里要注意一个坑: __javascript 只有函数能够构造 scope__

所以之前的 c 语言的惯性在这里不适用

``` javascript
for(var i = 0; i < 10; i ++) {
  console.log(i);
}

console.log(i) // 10
```

因为大括号并不形成作用域，下面的情况就是可预测的了

``` javascript
function fn(n){
  if(n>1){
    var a = n;
  } else {
    var b = n;
  }
  console.log(a);
}

fn(2) // 2
```


这种语法在 `c` `java` 里面肯定就跪了, `php` 里面会允许这种事儿不过一旦到 else 块会抛个 notice `undefined variable` 出来

不过这种变成习惯无疑是最差实践之一 好的习惯是讲所有 function 要用到的 var 都声明在 function 的最前面

### 语句优先

``` javascript
{a:1}; // label a => 1
var x={a:1}; // 赋值
{a:1,b:2}; // SyntaxError: Unexpected token :
var y={a:1,b:2}; // 赋值
```

在 js 执行的时候， 解释器会优先解释 `语句`, 什么意思？  `{a:1}` 明明可以是一个 object 直接量的，但是 js 优先将它看做一个用 {} 包裹的执行语句

而当  {a:1} 作为右值出现的时候， 我们的意思明显是一个赋值的表达式了，所以会解释成一个对象直接量

然而 {a:1,b:2} 会报错，因为 __逗号后面必须是表达式__，所以都好看到了 b之后没有期待 `:` 的出现所以 unexpected token :

__另外 js 自动在 } 之后添加 ;__

这使得写在同行的带有 {} 的逻辑可以被分割

``` javascript
{foo:[1,2,3]}[0] // [0]
// foo:[1,2,3]; // [1,2,3] 写在一行所以最终返回值被覆盖了
// [0] // [0]
```

``` javascript
{a:1}+2
// a:1 //1
// +2 //2
```

``` javascript
2+{a:1}
// 2 + {a:1}.toString()
// '2[object Object]'
```

## () 小括号

### 函数声明，调用函数 分割参数

``` javascript
function fn(name,age){
	//...
}
fn('hans',24);
var f = new fn('hans', 24)
```

### 与一些关键字组成条件语句

``` javascript
if() {

} else {

}
...
```

### 更改优先级

``` javascript
(1 + 2) * 3
```

小插曲 JSON.parse 黑暗取代

``` javascript
var jsonString = '{"a":1, "b":2}';
var jsonObj = eval('(' + jsonString + ')');
```

为啥加小括号？

因为上面讨论过大括号的问题

`{"a":1, "b":2}` 会被黑暗地转化成语句

加上了小括号就 __强迫小括号内部的内容被解析成表达式__

由此引出了多年来难以理解的 IIFE

## IIFE

``` javascript
// method1
(function() {
// force the function as an expression and call it
})()

// method2
(function() {
// call the function and force the process of call function an expression
}());

// method3
!function() {
//unary operand MUST follow an expression
}()
```

这里必须引出大招: es 的函数声明和表达式的区别

> 函数声明必须带有标示符（Identifier）（就是大家常说的函数名称），而函数表达式则可以省略这个标示符：

函数声明:
  function 函数名称 (参数：可选){ 函数体 }
函数表达式:
　　function 函数名称（可选）(参数：可选){ 函数体 }

一般来讲，区分的主要方法是这样的

``` javascript
// 典型声明
function f () {

}

// 表达式
var a = function g() {

}

var b = function () {

}

// SyntaxError: Unexpected token (
function () {

}
```

值得注意的就是上面的匿名函数会报错 作为一个语句它本来期待的是一个 __函数声明__ 但是我们用 __表达式__ 的方式给了一个匿名函数所以读到了 `(` 的时候就抛 error 出来了

但是小括号或者 单元操作符就能很好地解决匿名函数到表达式的转换

``` javascript
(function() {
  console.log('meth1')
})()
// function(){}
// 'meth1'

+function() {
  console.log('meth2')
}()
// NaN
// 'meth2'
```


前面总结过带标示符的函数表达式赋值的原理了 。。。 这回看就没有那么不可预测了

## [] 中括号

### 1 数组相关

``` javascript
var a = [1,2,3]
```

### 2 对象值获取

``` javascript
var a = {'b': 1};
a['b']  // 1
```

``` javascript
[1,2,3,4,5][0..toString.length];//2
'foo'.split('') + []; // 'f,o,o'
```

参考:

* [Samaritans](http://www.cnblogs.com/dolphinX/p/3524977.html)
* *javascript 权威指南*

happy coding, may the code will always be with you~
