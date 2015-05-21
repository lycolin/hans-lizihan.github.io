---
layout:     post
title:      javascript 对象(1) 属性 特性
date:       2015-05-20 12:07
summary:    javascript 基础篇之对象
categories: javascript
---

js是底层就是面向对象的语言。但是相比较传统的 OO 语言，js本身有着一些非常'奇怪' 的实现。这就直接导致了 js 的学习曲线相对比较高。

## 属性和特性

对于每一个对象的属性都有4个守护天使(特性) 在规定着它们的行为。

### 特性

> 特性是为了实现 javascript 内部属性的。 javascript 不能直接访问它们。 用两个中括号表示 e.g. [[Enumberable]]

回到对象的属性。对象的属性有两种

1. data property 数据属性
2. accessor property 访问器属性

### 数据属性

数据属性有4个特性决定它的表现

1. [[Enumberable]] 是否可以通过 for-in 遍历到 对于直接定义出来的属性 默认为 `true`
2. [[Configurable]] 能否通过 `delete` 删除属性， 能否修改属性的特性，能否把属性修改为 `accessor`。对于直接定义的属性默认为 `true`
3. [[Writable]] 能否修改属性的值。 直接定义出来的属性该值默认为 `true`
4. [[Value]] 可以是 primitive 也可以是 reference 的地址。我们方位该值的时候就是根据这个特性出来的。该值的默认值为 `undefined`

### 访问器属性

访问器属性也有4个特性规范它的表现

1. [[Enumberable]] 是否可以通过 for-in 遍历到 对于直接定义出来的属性 默认为 `true`
2. [[Configurable]] 能否通过 `delete` 删除属性， 能否修改属性的特性，能否把属性修改为 `数据属性`。对于直接定义的属性默认为 `true`
3. [[Get]] 访问函数，默认为 `undefined`
4. [[Set]] 写入函数，默认为 `undefined`

我们看到因为有了 `get` 和 `set` 两个函数，一个访问器属性的最终输出值可能和 data property 是一样的。不过内部的实现机制却完全不同

想要修改`属性` 的 `特性` 则需要用到一个神奇的方法: `Object.defineProperty`

### Object.defineProperty

> Object.defineProperty(ObjectName, PropertyName, [attributes])

#### 数据属性

``` javascript
var person = {};
Object.defineProperty(person, 'name', {
  writable: false,
  value: 'hans'
});
console.log(person.name); // hans
person.name = 'Hans'; // throws errors in strict mode
console.log(person.name); // hans
```

``` javascript
var person = {};
Object.defineProperty(person, 'name', {
  configurable: false,
  value: 'hans',
});

console.log(person.name); // hans
delete person.name; // throws error in strict mode
console.log(person.name); // hans
```

#### 访问器属性

``` javascript
var person = {
  first_name: 'Hans',
  last_name: 'Lee'
};

Object.defineProperty(person, 'full_name', {
  get: function() {
    return this.first_name + ' ' + this.last_name;
  },
  set: function(full_name) {
    var splited = full_name.split(' ');
    this.first_name = splited[0];
    this.last_name = splited[1];
  }
});

console.log(person.full_name); // Hans Lee
person.full_name = 'Lee Hans';
console.log(person.first_name, person.last_name); // Lee Hans
```

对于访问器方法来说因为一个 getter 一个 setter 而且两个都是可选的。

所以当一个访问器属性只有 getter 没有 setter的时候该属性只读不可写。严格模式下写入会报错

当一个访问器属性只有 setter 没有 getter 的时候该属性只可写不可读。读取严格模式下会报错

#### 区别

可以看到正常使用的时候两种属性的区别就在于在定义的时候是定义了 `value` 还是 `get` `set`。

#### Object.defineProperties()

这个方法可以一次性批量定义很多的属性

``` javascript
var person = {};

Object.defineProperties(person, {
  first_name: {
    value: 'Hans'
  },
  last_name: {
    value: 'Lee',
  },
  full_name: {
    get: function() {
      return this.first_name + ' ' + this.last_name;
    },
    set: function() {
      var splited = full_name.split(' ');
      this.first_name = splited[0];
      this.last_name = splited[1];
    }
  }
});
```

这里注意的是，当 特性 `configurable` 被设置成 false 之后就不可更改了。有点像 `finally` 的感觉。所以这样设置属性的时候要谨慎。因为一次性的没有回头路了。

此外，使用 `Object.defineProperty` 的时候如果不指定，则 `configurable`, `writable`, `enumberable` 默认都是 `false`

#### Object.getOwnPropertyDescription()

这个方法旨在读取属性的特性 返回值是一个对象，内部只有属性的4个特性

``` javascript
var person = {};
Object.defineProperty(person, 'name', {
  configurable: false,
  value: 'hans',
});

var attributes = Object.getOwnPropertyDescription(person, name);
console.log(attributes.configurable, attributes.value, attributes.enumberable, attributes.writable);
// false, 'hans', false, fasle
```

参考:

* *javascript 高级编程*