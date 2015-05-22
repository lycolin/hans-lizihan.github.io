---
layout:     post
title:      javascript 对象(3) 对象继承
date:       2015-05-22 00:25
summary:    javascript 基础篇之对象继承
categories: javascript
---

总体来说 js 的继承实现总共有两种方法 组合继承 和 组合寄生继承。

后者想在是主要的继承默认方案。因为 组合 继承 虽然简单但是有一些劣势。

## 组合继承

``` javascript
function Animal(name) {
  this.name = name;
  this.food = ['vege', 'fruite'];
}

Animal.prototype.printName = function () {
  console.log(this.name);
};

function Dog(name, color) {
  // 继承属性
  Animal.call(this, name);
  this.color = color;
};

// 继承方法 和原型链
Dog.prototype = new Animal();

Dog.prototype.printColor = function() {
  console.log(this.color);
};

var dog1 = new Dog('Freeman', 'black');
dog1.food.push('meat');
console.log(dog1.food); // ['vege', 'fruite', 'meat']
dog1.printName(); // Freeman
dog1.printColor(); // black

var dog2 = new Dog('John', 'white');
dog2.food.push('tomato');
console.log(dog2.food); // ['vege', 'fruite', 'tomato']
dog2.printName(); // John
dog2.printColor(); // white
```

组合式继承的优点就是简单，容易理解，用下面这副图就能明白

``` text
        +---------------------------------------------------+
        |                                                   |
        v                                                   |
+-----------------+           +-------------------------+   |
| Animal          |     +---->| Animal Prototype        |<--+<-+
+-----------------+     |     +-------------------------+   ^  |
| prototype       |-----+     | [[prototype]](ob.proto) |   |  |
+-----------------+           +-------------------------+   |  |
| [[prototype]]   |-----+     | constructor             |---+  |
+-----------------+     |     +-------------------------+   ^  |
                        |     | printName               |   |  |
                        |     +-------------------------+   |  |
                        |                                   |  |
                        |     +-------------------------+   |  |
                        +---->| Function                |   |  |
                        |     +-------------------------+   |  |
                        |     | [[prototype]](ob.proto) |   |  |
                        |     +-------------------------+   |  |
                        |                                   |  |
+------------------+    |     +-------------------------+   |  |
| Dog              |    |  +->| Dog Prototype           |   |  |
+------------------+    |  |  +-------------------------+   |  |
| prototype        |--->+--+  | [[prototype]]           |---+--+
+------------------+    ^  |  +-------------------------+   |
| [[prototype]]    |--->+  |  | constructor             |---+
+------------------+       |  +-------------------------+
                           +  | food([vete, fruite])    |
                           |  +-------------------------+
                           |  | name(undefined)         |
                           |  +-------------------------+
                           |
+------------------+       |  +-------------------------+
| dog1             |       |  | dog2                    |
+------------------+       |  +-------------------------+
| [[prototype]]    |------>+<-| [[prototype]]           |
+------------------+          +-------------------------+
| name (Freeman)   |          | name (John)             |
+------------------+          +-------------------------+
| color (black)    |          | color (white)           |
+------------------+          +-------------------------+
| food             |          + food                    +
+------------------+          +-------------------------+
```

这样写就终于理顺了。我们看到对于一个函数来说有两个属性 `[[prototype]]` 这个默认指向了 `Function` 内置类，它就是原型继承的起点，当在实例里面找不到的时候就会一路向上搜寻这个，这个搜寻的指针就是 `[[prototype]]`,而整个 javascript 一般而言的原型终点就是 `Object` 对象的原型。

``` javascript
console.log(Object.getPrototypeOf(Object.prototype) === null); // true
```

还有一个是我们可以自己定义的 `prototype` 这个可以指向一个我们自定义的类。 每次用 `new` 关键字创建对象的时候它会赋值这个 `prototype` 的引用并将它赋值给实例的 `[[ptototype]]` 也就是熟知的 `__ptoto__` 属性里面。

除此之外 new 关键字还会调用 `prototype` 中的 `constructor` 函数。这个函数默认指回了函数本身。当然也可以复写。

上面的例子中因为我们整体复写了 `Dog.prototype` 所以 `Dog.ptototype` 的 `constructor` 指向了 `Animal`

### instanceof

> instanceof 运算符可以用来判断某个构造函数的prototype属性是否存在另外一个要检测对象的原型链上。

语法是这样的

``` javascript
object instanceof constructor
```

直接抄一段 MDN 下来比较直观

``` javascript
function C(){} // 定义一个构造函数
function D(){} // 定义另一个构造函数

var o = new C();
o instanceof C; // true,因为:Object.getPrototypeOf(o) === C.prototype
o instanceof D; // false,因为D.prototype不在o的原型链上
o instanceof Object; // true,因为Object.prototype.isPrototypeOf(o)返回true
C.prototype instanceof Object // true,同上

C.prototype = {};
var o2 = new C();
o2 instanceof C; // true
o instanceof C; // false,C.prototype指向了一个空对象,这个空对象不在o的原型链上.

D.prototype = new C();
var o3 = new D();
o3 instanceof D; // true
o3 instanceof C; // true
```

这也就解释了上面动物相关的代码

``` javascript
Dog.prototype = new Animal();
```

的作用了。 这句话的主要目的就是直接将 Animal 的原型链绑定在 Dog 上面，从此 Dog 就可以是 Animal 的子类了。

## 组合寄生继承

因为组合继承虽然有很厉害的优点，简单明了，但是还是有两个缺点

1. 两次触发父类的构造函数
  1. Animal.call(this, name)
  2. Dog.prototype = new Animal()
2. 用实例的属性覆盖 prototype 中的属性(见上图)

可以看到因为 Dog.prototype = new Animal() 创建了一些内部属性放在了 Dog.prototype 中。 (name -> undefined)

这种时候就借助 ES5 新加入的标准 `Object.create()` 来进行寄生组合式继承。

具体代码大概是这样的

``` javascript
// 寄生
function inheritPrototype (subType, superType) {
  var prototype = Object.create(superType.prototype); // 创建对象
  prototype.constructor = subType; // 增强对象
  subType.prototype = prototype; // 指定对象
}

function Animal(name) {
  this.name = name;
  this.food = ['vege', 'fruite'];
}

Animal.prototype.printName = function() {
  console.log(this.name);
}

function Dog(name, color) {
  Animal.call(this, name);
  this.color = color;
}

inheritPrototype(Dog, Animal);

Dog.prototype.printColor = function() {
  console.log(this.color);
}


var dog1 = new Dog('Freeman', 'black');
dog1.food.push('meat');
console.log(dog1.food); // ['vege', 'fruite', 'meat']
dog1.printName(); // Freeman
dog1.printColor(); // black

var dog2 = new Dog('John', 'white');
dog2.food.push('tomato');
console.log(dog2.food); // ['vege', 'fruite', 'tomato']
dog2.printName(); // John
dog2.printColor(); // white
```

这种继承模式的示意图大概是这样的

``` text
        +---------------------------------------------------+
        |                                                   |
        v                                                   |
+-----------------+           +-------------------------+   |
| Animal          |     +---->| Animal Prototype        |<--+<-+
+-----------------+     |     +-------------------------+   ^  |
| prototype       |-----+     | [[prototype]](ob.proto) |   |  |
+-----------------+           +-------------------------+   |  |
| [[prototype]]   |-----+     | constructor             |---+  |
+-----------------+     |     +-------------------------+   ^  |
                        |     | printName               |   |  |
                        |     +-------------------------+   |  |
                        |                                   |  |
                        |     +-------------------------+   |  |
                        +---->| Function                |   |  |
                        |     +-------------------------+   |  |
                        |     | [[prototype]](ob.proto) |   |  |
                        |     +-------------------------+   |  |
                        |                                   |  |
+------------------+    |     +-------------------------+   |  |
| Dog              |    |  +->| Dog Prototype           |   |  |
+------------------+    |  |  +-------------------------+   |  |
| prototype        |--->+--+  | [[prototype]]           |---+--+
+------------------+    ^  |  +-------------------------+   |
| [[prototype]]    |--->+  |  | constructor             |---+
+------------------+       |  +-------------------------+
                           |  
+------------------+       |  +-------------------------+
| dog1             |       |  | dog2                    |
+------------------+       |  +-------------------------+
| [[prototype]]    |------>+<-| [[prototype]]           |
+------------------+          +-------------------------+
| name (Freeman)   |          | name (John)             |
+------------------+          +-------------------------+
| color (black)    |          | color (white)           |
+------------------+          +-------------------------+
| food             |          + food                    +
+------------------+          +-------------------------+
```

我们看到借助 `Object.create` 函数我们不用借助 `new` 这个关键字实例化 一个 Animal 然后赋值给 prototype, 而是直接将 Dog.prototype.[[ptototype]] 指向了 `Animal.prototype`。

这样做带来的好处就是大大清理了 `Dog.prototype` 中不必要的垃圾属性。

### Coffee Script 中继承的实现。

coffee 中继承的实现正是用这种方法。

``` coffee
class Animal
  constructor: (@name)->
    @food = ['vege', 'fruite']

class Dog extends Animal
  constructor: (name, @color)->
    super(name)

class OneOOneDog extends Dog
  constructor: (@name) ->
    super(@name)
```

``` javascript
var Animal, Dog,
  extend = function(child, parent) { 

    // 对父类的属性进行浅复制(将类方法弄下来)
    for (var key in parent) { 
      if (hasProp.call(parent, key)) child[key] = parent[key]; 
    } 

    // 制作一个 temp 对象，并将子类的 constructor 赋值给它
    function ctor() { 
      this.constructor = child; 
    } 

    // 将父类的prototype 给这个临时对象
    ctor.prototype = parent.prototype; 
    child.prototype = new ctor();

    // 上面两个步骤相当与手动 完成了 inheritPrototype() 函数的功能

    // 用 __super__ 属性去迅速找到父类的 prototype
    // 方便在子类中直接 使用 super(arg1, arg2...)
    child.__super__ = parent.prototype; 
    
    return child; 
  },
  hasProp = {}.hasOwnProperty;

Animal = (function() {
  function Animal(name1) {
    this.name = name1;
    this.food = ['vege', 'fruite'];
  }

  return Animal;

})();

Dog = (function(superClass) {
  extend(Dog, superClass);

  function Dog(name, color) {
    this.color = color;
    Dog.__super__.constructor.call(this, name);
  }

  return Dog;

})(Animal);

OneOOneDog = (function(superClass) {
  extend(OneOOneDog, superClass);

  function OneOOneDog(name1) {
    this.name = name1;
    OneOOneDog.__super__.constructor.call(this, this.name);
  }

  return OneOOneDog;

})(Dog);
``` 

虽然 coffee 为了照顾广大屌丝 es3 旧浏览器没有使用 `Object.create` 这样的简单解决方案，但是实际上的思想是一样的。

通过定义一个 `extend` 方法，相当于是寄生继承的大本营。

唯一不同的就是为了方便在子类中使用 `super` 方法去叫 父类的构造器，coffee 将 __super__ 属性赋予了子类，并在call super 方法的时候直接去 `call` __super__.constructor.call(this, name);

这样的做法相当于之前的 `Animal.call(this, name)` 但是这么做更加范式，因为每个子类的 super 方法都可以用 __super__ 属性去找到自己的父类 prototype，而单纯用 `Animal.call()` 则将这个 `super` 方法绑定在了一个父类的种类上。

参考:

* *javascript 高级编程*