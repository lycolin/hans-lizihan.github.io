---
layout:     post
title:      javascript 对象(2) 对象创建
date:       2015-05-21 12:06
summary:    javascript 基础篇之对象创建
categories: javascript
---

对象创建可以说是 javascript 里面最最令人迷惑的一块儿了。 无论是从 `java` 还是从 `cpp` 走过来的人都会对与 `javascript` oop 的实现方式非常不习惯。

## 函数即对象

首先必须先搞明白一件事儿

``` javascript
var test = function () {
  alert('hihi');
}

var test = new Function('alert("hihi")');
```

上面这两个函数的声明是等价的。可以理解为带有 `function` 的函数是 `object` 对应的字面量。

## 所有对象都继承自 `Object`

这个概念必须时时刻刻记住。因为无论是最原始的 `Number` `String` `Array` `Function` 还是后面会自己建立的对象类型，他们最顶层的祖先都是 `Object`

下一个连续剧应该接着记录 js 的继承原理。

## 工厂模式创建 对象

工厂模式的本意是通过传入不同类型的参数而输出不同的返回值 `Object` 函数就是一个很好的例子

``` javascript
Object(1); //Number(1)
Object('abc'); //String('abc');
Object([1,2,3]); // [1,2,3]
```

我们可以看到 Object 函数通过不同的数据类型而返回不同的实例或者 reference 这就是一个很简单的工厂模式。

所以创建对象的时候也可以套用一下工厂模式的思想

``` javascript
function createPerson (name, age, job) {
  var person = new Object();
  person.name = name;
  person.age = age;
  person.job = job;

  return person;
}
```

然后通过调用 `createPerson('Hans', 22, 'Software Engineer')` 就可以得到一个 实例的返回。

## 构建函数模式创建 对象

更加本能的是从 `java` 得来的习惯，声明一个 `class` 之后然后就直接用 `new` 关键字实例化 用 `new` 实例化就是 `构建函数模式`

``` javascript
function Person(name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
}
```

可以看到构建函数模式更加简约，省去了 `return` 这个步骤。同时省区了 `new` 一个 `Object` 的步骤。

此外注意 js 中默认的 convention 是用大写字母开头做 constructor 的函数名。

因为 js 最凶残的一个特性就是 __任何函数__ 都可以被用作为 constructor 所以为了区分我们刚开始就想当成构造函数的函数和一般的函数，大写开头的函数永远都是构造函数。

好了回到构造函数模式。之前无论任何语言都有 new 这个关键字。但是这个关键在在后面做了什么总是那部分被隐藏的细节。今天头一次看到  `new` 关键字做了什么，在 javascript 中。

1. 创建一个新的对象
2. 将构造函数的作用域给新的对象(所以 this.name 是对象的属性)
3. 执行构造函数的代码
4. 返回新的对象。

这么一看了然。其实 new 的作用就是刚刚我们在工厂模式中手动做的一些事儿，只不过中间缺了一步实在 `new Object()` 之后将 实例中的 Object 作用域转化一步。

这也就是 javascript 中为什么 __所有__ 对象的祖先都是 `Object`

除此之外用构造函数法创建对象还有一个好处，那就是这个对象可以作为一个我们自定义的对象类型被我们引用。

``` javascript
console.log(person1 instanceof Person); // true
```

除此之外还要理解一点就是通过构造函数法构造出来的实例都有一个  `constructor` 的属性。这是一个指向 构造函数的指针。

``` javascript
console.log(person1.constructor === Person); // true
```

### 构造函数作为函数调用

和别的语言不一样，js 中的构造函数不是严格跟着 `class` 存在的，所以它本身是可以被作为函数调用的。

举个简单的例子 `String`

``` javascript
String('hihi'); // hihi

new String('hihi') // String {0: "h", 1: "i", 2: "h", 3: "i", length: 4, [[PrimitiveValue]]: "hihi"}
```

`String` 方法直接返回了一个字符串值，而带了 new 之后就直接返回了一个 String 的实例

### 对象的方法

构造函数法已经很不错了，但是碰到了对象方法这个事儿就没有那么美好了，因为每一个实例的属性其实都是不太一样的。但是一旦所有的实例都要有个方法的调用就会额外地创建多个函数实例。

``` javascript
function Person (name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
  this.sayHi = function() {
    console.log(this.name + this.age + this.job);
  }
}
```

因为我们前面看到过，当使用了 `function` 关键字的时候我们相当于是创建了一个实例并且将这个实例赋值给了每一个创建出来的实例。

``` javascript
var person1 = new Person('Hans', 22, 'SE');
var person2 = new Person('Lee', 23, 'ES');

console.log(person1.sayHi === person2.sayHi); // false
```

上面这段代码证明了两个 sayHi 并不是同一个 reference

有一个简单的解决方案可以让实例用同一个方法引用，那就是在外部声明一个函数并将这个引用赋值给方法。

``` javascript
function Person(name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
  this.sayHi = sayHi;
}

function sayHi() {
  console.log(this.name + this.age + this.job);
}
```

但是这样的话导致了给全局变量声明了太多的全局函数，和 oop 封装的理念大相径庭。 所以就终于引出了 javascript 最大的难点: 原型模式 `prototype pattern`

## 原型模式

可以说原型模式是 javascript 里面最最精髓但是又是最为巧妙的一部分。因为原型模式让我们真正从底层去探寻一门语言的集成的实现原理而不是无脑地使用 `extend`

> The prototype pattern is a creational design pattern in software development. It is used when the type of objects to create is determined by a prototypical instance, which is cloned to produce new objects.

啥意思？就是说 prototype pattern 就是 js 的典型实现原理。所有的对象都有一个 指针指向了一个 prototype 对象，当所有对象的实例被创建的时候内部都有一个指针指向了这个 prototype, 而 prototype 本身又是一个对象，所以 prototype 也有一个指针指向了 prototype 的 prototype 从而形成了一个原型链。

具体的继承实现留给下一个连续剧记录。这里先记录一下 javascript 中 prototype 实现的机理。

不同于其他语言，javascript 封装的东西很多，很多内核引擎的实现都被省略了，这有点坑。

``` javascript
function Person(name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
}

Person.prototype.sayHi = function() {
  console.log(this.name + this.age + this.job);
}

var person1 = new Person('hans', 22, 'SE');
var person2 = new Person('Hans', 23, 'ES');

console.log(person1.sayHi === person2.sayHi); // true
```

上面代码就证明了两个实例的 prototype 确实是指向了同一个对象。

### 原型实现的内存示意图

``` text
       ---------------------------------------------------------------
       |                                                             |
       v                                                             |
+-------------------+         +-------------------------------+      |
|  Person           |    +--->| Person prtotype               |      |
+-------------------+    |    +-------------------------------+      |
| prototype |       |----+    | constructor     |             |------+
+-------------------+    |    +-------------------------------+
                         |    | properties      |  someValue  |
                         |    +-------------------------------+
                         |
                         +<------------------------------------------+
                         |                                           |
+-------------------+    |    +-------------------------------+      |
|  person1          |    |    |  person2                      |      |
+-------------------+    |    +-------------------------------+      |
| [[prototype]] |   |----+    | [[prototype]]   |             |------+
+-------------------+         +-------------------------------+
```

这副图就比较明确了。为一要注意的就是在 wiki 里面也看到了， prototype pattern 的精髓在于 `clone` 所以其实最后的实例中都有一个隐藏的 [[prototype]] 的特性。这个特性是 js 核心在创建实例的时候自动 `复制` 过来的。作为一个指向 prototype 的指针。

标准中这个特性是不可见的，在非标准的实现里面 (chrome, safari, firefox) 可以通过 `__proto__` 这个属性来访问复制过来的 [[prototype]]

对于原型有两个常用的函数来确定 prototype 是谁的 prototype

### *.prototype.isPrototypeOf()

这个函数可以在任何对象上面 call 出来。本质上来讲它会调查参数中的 [[prototype]] 是不是当前的对象的指针。

``` javascript
console.log(Person.prototype.isPrototypeOf(person1)); // true
```

### Object.getPrototypeOf()

这个是 Object 对象上面的一个方法，直接返回 [[prototype]] 的值。

``` javascript
console.log(Object.getPrototypeOf(person1) === Person.prototype); // true
```

### 属性查询的先后顺序。

属性查询的先后顺序设这样的

1. 查询实例中有没有这个属性
2. 去原型链递归查询

``` javascript
function Person () {
  
}

Person.prototype.name = 'Hans';

var person1 = new Person();
var person2 = new Person();
person1.name = 'Lee';
console.log(person1.name); // Lee
console.log(person2.name); // Hans
```

### hasOwnProperty() 

这个方法可以简单查询到某一个属性是否是实例自身的属性。这个简单的方法就可以过滤到 n 多个 prototype 中我们像隐藏的属性，在 for-in loop 里面尤其常用。

此外值得注意的是前面看到过属性的 [[enumberable]] 特性。默认自己注册的属性都是 true 也就是可枚举。在 for-in loop里面默认会便利出来所有可枚举的属性。所以在用 for-in 的时候通常加上 hasOwnProperty 做一个filter以防便利时不慎将原型中的方法或者属性都拿了出来。

``` javascript
for(property in object) {
  if(!object.hasOwnProperty(property)) continue;
  // do something here
}
```
### in 关键字

in 会像调用方法一样现在实例中找属性，如果找不到再去原型链递归找。所以如果像要确定一个实例的原型有没有某个属性属性要这样

``` javascript
function hasPrototypeProperty(object, name) {
  return !object.hasOwnProperty(name) && (name in object));
}
```

### Object.keys()

比较方便地取得一个实例中可枚举的属性可以用 `Object.keys(object)` 来取得

这个方法会返回一个数组，里面装满了可枚举的属性 key

``` javascript
console.log(Object.keys(Person.prototype)); 
// ['name', 'job', 'age', 'sayHi']
```

### Object.getOwnPropertyNames();

同时取得所有属性的 key，无论该key是否可枚举

``` javascript
Object.getOwnPropertyNames(Person.prototype); 
// ['constructor', 'name', 'age', 'job', 'sayHi']
```

### 字面量创建 prototype

如果需要用字面量创建 prototype 实例就有一点点麻烦了。

因为通过 `Person.prototype.name` 这种赋值的方法其实是向  `Person` 这个 function 的默认 prototype 中加入新的属性。但是如果用字面量来直接赋值得话有一个问题就是会覆盖掉 `constructor` 的指针。此时新建出来的实例的 constructor 就会fallback 到 `Object`

``` javascript
function Person() {
  
}
Person.prototype = {
  name : 'Hans',
  job: 'SE',
  age : 22
};

var person1 = new Person();
console.log(person1.constructor === Person); // false
console.log(person1.constructor === Object); //true
```

这样做得话在后面的继承中会出问题。所以要再 hack 一步。

``` javascript
function Person() {
  
}
Person.prototype = {
  name: 'Hans',
  job: 'SE',
  age: 22
};
Object.defineProperty(Person.prototype, 'constructor', {
  enumberable: false,
  value: Person
};
```

注意这里用了 `defineProperty` 是因为默认设置 constructor 会导致 enumberable 为 true 这样每次 for-in 就悲剧了。

### 动态原型

看到了这个 `constructor` 的例子之后也可以看到， `prototype pattern` 的核心是将原型的指针复制一份然后放在实例的 [[prototype]] 当中。

所以当我们在原型上面加入新方法或者移除旧方法的时候原型的行为都会映射到创建的实例上面。

``` javascript
var person1 = new Person();
Person.prototype.sayHi() {
  console.log('hihi');
}

person1.sayHi(); // hihi
```

然而如果重写了整个原型，也就是直接修改了原型的引用事情就没有那么美好了。

``` javascript
function Person() {}
var person1 = new Person();
Person.prototype.sayHi = function() {
  console.log('hihi');
}

Person.prototype = {
  sayHi: function() {
    console.log('hoho');
  }
}

person1.sayHi(); // hihi
```

这是因为如果直接改变了 prototype 的赋值之后实例并不知道 prototype 的新的应用所以它还会回去找旧的 prototype 所以按照字面量赋值重写的 prototype 无法映射到实例上。

``` text
       ---------------------------------------------------------------
       |                                                             |
       v                                                             |
+-------------------+         +-------------------------------+      |
|  Person           |    +--->| new Person prototype          |
+-------------------+    |    +-------------------------------+      |
| prototype |       |----|    | constructor     |             |------+
+-------------------+         +-------------------------------+      |
                              | sayHi           |  'hoho'     |      |
                              +-------------------------------+      |
                                                                     |
+-------------------+         +-------------------------------+      |
|  person1          |    +--->|  old Person Prototype         |      |
+-------------------+    |    +-------------------------------+      |
| [[prototype]] |   |----+    | constructor     |             |------+
+-------------------+         +-------------------------------+
                              | sayHi           | 'hihi'      |
                              +-------------------------------+
```

### 原声对象原型

这个之前看过了很多。很多有用的方法都是定义在原声原型对象上面的。

`Array.prototype.map` `String.prototype.substr` etc.

不过总体来将非常不建议重写原声原型的方法或者直接修改 `Array.prototype.*` 因为这样可能会导致意外重写或者命名冲突而非常混乱。

### 原型模式的缺点。

因为大家都是应用一个单例里面的各种属性，所以自然而然地一旦某个实例修改了原型中的某一个属性那么大家都受影响了。

``` javascript
function Person() {};
Person.prototype = {
  constructor: Person,
  hobbies: ['dota', 'javascript']
};
var person1 = new Person();
var person2 = new Person();
person1.hobbiles.push('lol');
console.log(person2.hobbies); 
// ['dota', 'javascript', 'lol']
```

对于 primitives 来说这不算什么因为可以用 `person1.name = 'hans'` 这样在实例上面声明一个属性，这样就可以覆盖掉原型中的属性了。但是对于引用来说我们知道 `Array` 和 `Object` 都可以直接被某些他们的方法修改的，就好像 `Array.prototype.push` 一样，直接修改了 `Array` 的内存属性所以这种修改就可以直接映射到各种实例上了。

正是因为这样所以一般地创建实例会组合使用 构造器模式 和 原型模式

## 组合使用创建对象实例

``` javascript
function Person(name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
}

Person.ptototype = {
  constructor: Person,
  sayHi: function() {
    console.log('hihi');
  }
};
```

这就是一般比较完整的对象创建的全部了。

这里值得注意的是

1. 一般每个实例都会不同的属性用构造器模式构建
2. 所有实例统一继承的属性或者方法放在 Prototype 中。

这样就是比较完整的 对象实现了。

### 动态原型模式

感觉真正的 OO 应该只有 constructor? 所以上面的代码可以进行这样的重构

``` javascript
function Person(name, age, job) {
  this.name = name;
  this.age = age;
  this.job = job;
  if(typeof this.sayHi !== 'function') {
    Person.prototype.sayHi = fucntion() {
      console.log('hihi');
    }
  }
}
```

值得注意的是这样生成之后就不可以再用 对象字面量复写 `prototype`，新的实例就找不回旧的 prototype 了。

## 寄生构造函数模式 (parasitic)

``` javascript
function Person(name, age, job) {
  var o = new Object();
  o.name = name;
  o.age = age;
  o.job = job; 
  o.sayHi = function() {
    console.log('hihi');
  }
  return o;
}

var person1 = new Person('Hans', 22, 'SE');
console.log(person1 instanceof Person); //false
console.log(person1 instanceof Object); //true
```

这里值得注意的是之前看到  `new` 关键字会默认返回 new 出来的新实例，但是如果说构造函数有了 `return` 关键字，那么 `new` 关键字就会返回构造函数中的 return 回来的对象。

这种模式的好处是可以简单得做到一些继承的效果，但是坏处也很明显： 就是这么重载 `return` 的做法使得 `Person` 不能作为一个类型存在了 所以气势上并不推荐这样构建对象。

## 稳妥构造函数模式 (durable object)

所谓稳妥对象就是没有公共属性，方法不引用 `this` 指针的对象。在某些对安全性要求比较高的二笔环境里面可能会用到，目的是防止数据遭到破坏或者篡改。

它有两个特点

1. 创建对象的时候不用 `new`
2. 内部的方法不引用 `this`

``` javascript
function Person(name, age, job) {
  var o = new Object();
  // private proerties goes here

  o.sayHi = function() {
    console.log('hihi' + name);
  }

  return o;
}

var person1 = Person('Hans', 22, 'SE');
```

这种模式实际上和寄生模式是同一个套路，只不过是没有 `this` 和 `new` 的引用而已。

## Object.create()

这个方法也可以新建一个对象。 这个方法接受一个参数，作为创建出来的实例的 prototype, 相当于将参数的引用复制一份然后给信创建的实例的 [[prototype]]

``` javascript
function Person() {};
var myPrototype = {
  constructor: Person,
  name: 'Hans'
};

var me = Object.create(myPrototype);
console.log(me.name); // Hans
console.log(me instanceof Person); // false
```

同样的问题也存在，就是 type 不会被创建出来。

参考:

* *javascript 高级编程*