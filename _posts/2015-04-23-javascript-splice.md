---
layout:     post
title:      javascript Array#splice
date:       2015-04-23 16:00
summary:    Javascript splice
categories: javascript
---

# Array.prototype.splice()

> The splice() method changes the content of an array by removing existing elements and/or adding new elements.

`splice` 是 `slice` 的进阶版本，之前有写过关于 `slice` 浅复制的作用 和 转化类数组 类的作用

``` javascript
function trigger() {
  Array.prototype.slice.call(arguments);
}
```

这是 Airbnb 给出的建议，因为 slice 方法会将原数组复制出来一份然后返回一个新的数组。

`splice` 虽然和 `slice` 只差了一个字母，但是他们的作用非常不同。因为 `splice` 会修改原数组的内容

``` javascript
array.splice(start, deleteCount[, item1[, item2[, ...]]])
```

最简单的 use case

``` javascript
var test = [1,2,3,4,5,6];

test.splice(0, 4); // 删掉从 0 到 0 + 4的所有内容 [0, 0 + 4)

console.log(test); // [5,6]
```

注意的就是这个 deleteCount 是开区间

此外这个 deleteCount 是 count, 千万不要 native 以为这是 index

``` javascript
var test = [1,2,3,4,5,6];

test.splice(1, 4); // [1, 1 + 4)

console.log(test); // [1,6]
```

这跟 `slice` 的 `start end` api 完全不一样

另外 splice 自身的返回值是刚刚删掉的那一堆

``` javascript
var test = [1,2,3,4,5,6];

var deleted = test.splice(1, 4);

console.log(deleted); // [2,3,4,5]
```

所以 `splice` 简单记忆就是  slice + split => splice

除此之外， split 之后 splice 还负责了一项艰苦卓越的功能 `insert`

``` javascript
var test = [1,2,3,4,5];

test.splice(0,2,'apple','orange');

console.log(test);// ['apple','orange', 3,4,5]
```

在前两个 start, deleteCount 之后就是要插入的内容了。 插入的内容从 start 开始，然后可以无限插入任何长度的东西。

这个功能使得 `splice` 成为数组操作的利器。 它就像一把手术刀一样可以轻易在数组的任何位置横刀一切插入任何想插入的值。

happy coding, may the code will always be with you~
