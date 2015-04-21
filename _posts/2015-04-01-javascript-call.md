---
layout:     post
title:      javascript Function.prototype.call
date:       2015-04-01 00:20
summary:    call
categories: javascript
---

# `Function.prototype.call`

在 `slice` 中 偶遇 `call` 方法

> To convert an array-like object to an array, use Array#slice.

``` javascript
function trigger() {
     var args = Array.prototype.slice.call(arguments);
     ...
}
```

## `arguments` property

要先理解 `trigger` 方法，首先要理解一个关键概念， `arguments`

> The arguments object is an Array-like object corresponding to the arguments passed to a function.

``` javascript
function test () {
    console.log(arguments);
}

test(); // {}
test(1,2,3);  // { '0': 1, '1': 2, '2': 3 }
test([1,2,3]); // { '0': [ 1, 2, 3 ] }
test({0:1, 1:1}); // { '0': { '0': 1, '1': 1 } }
```

可以看到， 在任何 `function` 里面， 都可以直接获取 `argument` 对象。 这个对象有 0,1,2,3,... 等 index 所以被称为 `类数组对象`

好了前言结束了。 正式进入 `call` 专场

> The call() method calls a function with a given this value and arguments provided individually.

``` javascript
fun.call(thisArg[, arg1[, arg2[, ...]]])
```

不言而喻，到这个时候就很好理解了，`call` 叫一次这个方法

注意 call 函数中 thisArg 的作用是类似于 .bind(this)， 就是告诉 call 中的函数 `this` 代表了什么

一般来讲， `call` 有三个作用

1 子类继承父类时触发父类 constructor

``` javascript
function Product(name, price) {
  this.name = name;
  this.price = price;

  if (price < 0) {
    throw RangeError('Cannot create product ' +
                      this.name + ' with a negative price');
  }

  return this;
}

function Food(name, price) {
  Product.call(this, name, price);
  this.category = 'food';
}

Food.prototype = Object.create(Product.prototype);

function Toy(name, price) {
  Product.call(this, name, price);
  this.category = 'toy';
}

Toy.prototype = Object.create(Product.prototype);

var cheese = new Food('feta', 5);
var fun = new Toy('robot', 40);
```

2 匿名函数触发

``` javascript
var animals = [
  { species: 'Lion', name: 'King' },
  { species: 'Whale', name: 'Fail' }
];

for (var i = 0; i < animals.length; i++) {
  (function(i) {
    this.print = function() {
      console.log('#' + i + ' ' + this.species // 'this' is animals[i] out side
                  + ': ' + this.name);
    }
    this.print();
  }).call(animals[i], i);
}
```

3 Convert Array like Object to Array

``` javascript
function trigger() {
  var args = Array.prototype.slice.call(arguments);
  // bind arguments to this, and passes no argument to 'slice' method
  ...
}
```

豁然开朗，因为 slice 方法不光 Array有，String也有，为了不冲突，所以当要将一个 Array-like Object转化成 Array 的时候，可以用 `trigger` 方法这样触发 Array.prototype.slice 方法。所以可以得到一个浅复制过来的 `array`


happy coding, may the code will always be with you~
