---
layout:     post
title:      javascript 高级函数
date:       2015-03-19 23:00
summary:    map filter forEach
categories: javascript
---

> 数组高级函数

类似于 `PHP` 的数组高级函数 javascript 中更加原声地支持了各种 `数组高级函数`

相对来讲数组高级函数不光可以大大降低代码的行数 还可以非常良好地支持 javascript 中 chaining 的语法 并且支持 javascript 异步编程的精髓

`Array.prototype.forEach` 实现

``` javascript
Array.prototype.forEach = function(callback) {
    for(var i = 0; i < this.length; i ++) {
        callback(this[i]);
    }
};
```

`forEach` 实际用法

``` javascript
var test = [1,2,3,4,5];

test.forEach(function(value) {
    console.log(value);
});
// 1 2 3 4 5
```

`Array.prototype.map`

``` javascript
Array.prototype.map = function(projection) {
  var result = [];

  // **this** refers to the Array
  this.forEach(function(item) {
    result.push(projection(item));
  });

  return result;
}
```

`map` 实际用法

``` javascript
var squares = [1,4,9,16,25];

console.log(squares.map(Math.sqrt))
// [1,2,3,4,5]
```

`filter` 实现

``` javascript
Array.prototype.filter = function(predicate) {
    var result = [];

    this.forEach(function(item) {
      if(predicate(item))
        result.push(item);
    })

    return result;
}
```

`filter` 用法

``` javascript
var result = [1,3,4,5,6,6].filter(function(value) {
  return value > 5;
});

console.log(result);

// [6,6]
```


happy coding, may the code will always be with you~
