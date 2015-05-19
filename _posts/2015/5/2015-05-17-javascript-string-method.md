---
layout:     post
title:      javascript 字符串方法
date:       2015-05-19 12:42
summary:    javascript 基础篇之 字符串操作
categories: javascript
---

# slice() substr() substring()

js 字符串操作就这么几个，但是如果不明白得话真的很容易乱。

## substr()

这个就是祸乱之源，所有的字符串操作都可以用这个推演出来。

> str.substr(start[, length])

``` javascript
var str = 'abcdef';

str.substr(0); // 'abcdef'
str.substr(0, 2); // 'ab'
str.substr(1, 2); // 'bc'
str.substr(1, 0); // ''
str.substr(1, -1); // ''
str.substr(-1); // 'abcdef'
str.substr(7); // ''
```

substr 的原理 是

1. 将 length 负值转化为 0
2. start = string.length + start 如果是负数则 start = 0
3. 由 `start` 的位置开始数 `length` 长的字符
4. 如果 `start` 大于字符串本身的长度或者小于0则 直接返回 ''

话不多说直接手撸库

``` javascript
String.prototype.substr = function(start, length) {
  // set undefined start to 0
  if(start === undefined) {
    start = 0;
  }

  // set undefined length to string's length
  if(length === undefined) {
    length = this.length;
  }  

  if(length < 0) {
    length = 0;
  }

  // set negative start to positive
  if(start < 0) {
    start += this.length;
    // if start still less than 0 after adding string length
    // then set it to 0
    if(start < 0) start = 0;
  }

  if(start >= this.length) return '';
  if(length === 0) return '';

  var end = start + length > this.length ? this.length : start + length;

  var result = '';
  for(var i = start; i < end; i ++) {
    result += this.charAt(i);
  }

  return result;
}
```

好了撸完了第一个库后面的库就好说了。

## substring()

substring 的实现比较凶残。

1. 如果参数有负数就直接变成 0
2. 比较两个参数谁大谁小，将小的放在前面

``` javascript
String.prototype.substring = function(start, end) {
  if(start === undefined || start < 0) start = 0;
  if(end === undefined) end = this.length;
  if(end < 0) end = 0;
  
  // swap the two in the right order
  if(start > end) {
    var tmp = start;
    start = end;
    end = tmp;
  }

  return this.substr.call(this, start, end - start);
}
```

## slice

之前曾经仔细研究过这个函数了。现在回来手撸一下这个库。

1. 如果起始点为负数且 start + length < 0, 则起始点归 0
2. 如果起始点为负数且 start + length > 0, 则起始点就是这个加和
3. 终止点应用同一个逻辑于起始点

``` javascript
String.prototype.slice = function(start, end) {

  if(start === undefined) start = 0;
  
  if(start < 0) {
    var sum = start + this.length;
    start = sum < 0 ? 0 : sum;
  }
  
  if(end === undefined) end = this.length;
  
  if(end < 0) {
    var sum = end + this.length;
    end = sum < 0 ? 0 : sum;
  }  
  
  return this.substr.call(this, start, end - start);
}
```

有点长但是能工作.