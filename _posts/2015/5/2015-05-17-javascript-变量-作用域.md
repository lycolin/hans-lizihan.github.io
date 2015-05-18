---
layout:     post
title:      javascript 变量 作用域
date:       2015-05-17 18:35
summary:    javascript 基础篇之 变量 作用域
categories: javascript
---

javascript 的曲线相对而言很高（jQuery 党可以无视） 其中很大的一部分原因都是因为它非常葛的变量和作用域原理。

## 变量

变量应该是所有的程序语言最核心的部分。 js 对于变量有着一点点不一样。之前小小总结过一波(数据类型)[http://hans-lizihan.com/javascript/2015/04/28/javascript-数据类型.html]的相关今天再补充一些。

### Primitives

对于各种 primitives 因为不是引用类的数据类型所以每一次赋值其实是一个复制。

``` javascript
var num1 = 1;
var num2 = num1;
num1 = 2;
console.log(num2); // 1
```

这段代码其实  很好的说明了 js 中对于原始值是赋值传值的。具体流程大概是这样的。

``` text
----------      ----------
|        |      |        |
|        |      |        |
|        |  ->  | num2:1 |
| num1:1 |      | num1:1 |
----------      ----------
```

在内存里面将 `num1` 赋值给 `num2` 的时候其实是直接发生了一次复制。 js 其实是将 `Number(1)`(新建了一个 primitive) 赋值给了 `num2`

### References

引用传值的变量就不太一样了。

``` javascript
var obj = new Object();
var obj1 = obj;
var obj2 = obj;
obj.attribute = 'hihi';
console.log(obj2.attribute); //hihi
```

因为 `obj1` `obj2` 同时指向了堆中的 `obj` 所以对于 `obj1` 的操作会同时反射到 `obj2` 上

但是其实原理上来说无论是 primitives 还是 reference，这个赋值的过程都是一个复制的过程。 不同的是 primitive 的复制是直接新建一个一模一样的 primitive 赋值给 新的 var 然而 reference 是赋值出来一个 reference 的内存地址然后赋值给一个新的 var


``` text
VO                        heap
----------                ---------------
|        |                |             |
|        |                |             |
|        |                |-------------|
| obj1   |---------->---->+ obj         |
----------          ^     |-------------|
                    |     |             |
----------          |     |             |
|        |          |     |             |
|        |          |     |             |
| obj2   | ---------^     |             |
| obj1   | ---------^     |             |
----------                ---------------
```

### 参数传递

其中最难理解的就是参数传递了。 在 C 中我们知道有 call by reference 和 call by value 两种。

> 在 js 中我们只有 call by value

这对于参数是 primitives 的函数来说比较好理解，但是对于将 reference 传入的 function 事情就不太一样了

``` javascript
var obj = new Object();
obj.attribute = 'hihi';
function test(args) {
  args.attribute = 'hoho';
}

test(obj); 
console.log(obj.attribute); //hoho
```

上面这个例子中如果按照常理， call by value 得话那么在 `test` 函数中修改 args 应该不影响外部的 `obj` 才对，但是事实上修改了 `args` 之后 `obj` 也发生了改变。 说好了的 call by value 呢？

其实故事是这样的，reference 的 参数传递是将 reference 的内存地址复制一份传入到 args 里面然后再 `test` 函数中修改 `args` 其实是对 指针指过去的 `obj` 直接进行了修改。然而这并不是 `call by reference`, 因为我们传入的 `args` 并不是 `reference` 的内存指针而是一个 __复制__ 出来的指针。

``` javascript
var obj = new Object();
obj.attribute = 'hihi';
function test(args) {
  args.attribute = 'hoho';
  args = new Object();
  args.attribute = 'fofo';
}

test(obj);
console.log(obj.attribute); // hoho
```

这个例子就很好的证明了函数的传参是 `call by value`， 因为假设是 `call by reference` 得话那么我们在 `test` 内部将新的 argument 赋值给了 `args` 那么这个引用就应该被改变成 `test` 函数中新创建的 `Object` 所以最后的 console.log 应该打印出来 'fofo' 中才对。

## 作用域

我们知道，js 中 __唯一__ 形成作用域的元素就是 `funciton`， 对于 `{}` 包裹的元素来说，按照正常的 cpp 和 java 的理解都是自己形成一个作用域的，然而在 js 中这些规则都不适用。 

真正从底层去探寻 javascript 的作用域实现机理这个事儿就深了。

这里先声明几个概念

1. Variable Object 变量对象 (VO)
2. Active Object 活动对象 (AO)
3. Excecution Context 执行环境 (EC)
4. Excecution Context Stack 执行环境栈 (ECS)
5. scope chain 作用域链  

### VO

> 如果变量与执行环境相关，那么变量知道自己应该储存在哪里，并且知道该如何访问，这种机制叫做变量对象(variable object)

很抽象。具体点来说就是我们声明出来的 

1. 变量声明 (var a = 'foo')
2. 函数声明 (function foo (){})
3. 函数的形式参数 (function foo(bar) {})

都会被 js 内部机制存储在一个对象里面。这个对象就是 VO

VO 是当前执行环境中的一个属性

``` javascript
activeExecutionContext = {
  VO: {
    // 环境数据（var, function declaration, function arguments)
  }
};
```

举个栗子

``` javascript
var a = 10;
 
function test(x) {
  var b = 20;
};
 
test(30);
```

这段代码的背后其实发生了这样的故事: 

```
// 全局环境的变量对象
VO(globalContext) = {
  a: 10,
  test: <reference to function>
};
 
// test函数环境的变量对象
VO(test functionContext) = {
  x: 30,
  b: 20
};
```

#### 全局对象

> 全局对象(Global object) 是在进入任何执行环境(EC)之前就已经创建了的对象；

> 这个对象只存在一份，它的属性在程序中任何地方都可以访问，全局对象的生命周期终止于程序退出那一刻。

理解整个生命周期得花要先看下全局对象的轮廓, 平时用 javascript 的时候一般都是在浏览器的宿主环境里面的。

``` javascript
global = {
  Math: function() {},
  String: function() {}

  window: global //引用自身
};
```

在打开浏览器页面的时候浏览器就帮我们构建了这么一个对象。所以我们所有的 js 代码都生活在这个大的 全局类里面。

这么看得花就明白为什么 `window` 是全局变量的载体了。

全局对象其实是一个 EC 同时也是一个 VO。 可以理解成

``` javascript
VO(globalContext) === global
```

所以在最最外面的 js 环境下面

`(VO === this === global)`

就是说 js 的顶层全局对象其实就是顶层变量对象。 而 window 和 this 都指向这个全局边对象。

#### VO 的执行过程

进入每一个环境的时候都会经历两个过程

1. 初始化 VO（也就是之前看到的概念 '预编译')
2. 执行环境中的逻辑

其中对于初始化 VO 的过程是这样的

1. 函数的形参（当进入函数执行环境时） —— 变量对象的一个属性，其属性名就是形参的名字，其值就是实参的值;对于没有传递的参数，其值为undefined
2. 函数的声明(Function declaration(FD)) —— 变量对象的一个属性，其属性名和值都是函数对象创建出来的;如果变量对象已经包含了相同名字的属性，则替换它的值
3. 变量声明（var，VariableDeclaration） —— 变量对象的一个属性，其属性名即为变量名，其值为undefined;如果变量名和已经声明的函数名或者函数的参数名相同，则不会影响已经存在的属性。 

然后才会真正去执行函数中的逻辑。

这也就是变量提升等等现象出现的本质。

回头来看这段代码就明白了

``` javascript
a(); // hihi
function a() {
  alert('hihi');
}
var a = function() {
  alert('heihei');
}
a(); // heihei
```

因为这个过程是这样的：

1. 看到了 FD 那么直接将它压入 VO
2. 又看到了 VD 但是 FD 已经有了 `a` 所以跳过
3. 真正进入逻辑部分，将 alert('heihei') 的函数赋值给了 `a`,覆盖掉了前面的 `a` 的声明

所以早前提到的一个概念 

> 当函数执行有命名冲突的时候，函数依次填入 变量 => 函数 => 参数

也可以理解到了。因为 VO 初始化阶段的时候的顺序原因，所以 js 语言的执行机制就变成了这样。

### AO

对于函数来说这个又不一样了。因为生命周期不同，函数的变量对象不可能像全局对象那样当用户关闭窗口的时候才被销毁。所以就有了 AO 这个概念。

`VO(functionContext) === AO;`

这里就真的开始有趣了。之前看到的一些 js 怪癖终于可以再这里得到解答了。

``` javascript
function test(a, b) {
  var c = 10;
  function d() {}
  var e = function _e() {};
  (function x() {});
}

test(10);
```

这里在进入 `test` 环境的时候

``` javascript
testEC = {
  AO:{
    arguments:{
      callee:test
      length:1,
      0:10
    },
    a:10,
    c:undefined,
    d:<reference to FunctionDeclaration "d">,
    e:undefined
  }
};
```

在执行完毕 `test` 之后 EC 是这样的, 这里我们看到的是对于每一个 `function` 的参数，在进入函数环境之前都会对 `arguments` 进行一步解析。

这也就是为什么在函数体中我们可以拿到 `arguments` 这个变量。

``` javascript
testEC={
  AO:{
    arguments:{
      callee:test,
      length:1,
      0:10
    },
    a:10,
    c:10,
    d:<reference to FunctionDeclaration "d">,
    e:<reference to FunctionDeclaration "e">
  }
};
```

这里值得注意的就是 __函数表达式不会对VO造成影响，因此，(function x() {})并不会存在于VO中。__

### EC

EC 分三种

1. 全局级别 EC - 这个是默认的代码运行环境，一旦代码被载入，引擎最先进入的就是这个环境。
2. 函数级别 EC - 当执行一个函数时，运行函数体中的代码。
3. eval 级别 EC - 在eval函数内运行的代码。

EC 分两个阶段 在上面的 VO 中已经提前用了这个小概念了。

1. 进入阶段 - 这个阶段是调用函数的时候的过程，初始化各种函数需要的东西。
2. 执行阶段 - 真正执行函数的逻辑的阶段

``` javascript
EC = {
  VO:{/* 函数中的arguments对象, 参数, 内部的变量以及函数声明 */},
  this: thisValue, // 这个概念之后再详细看
  Scope: {}
}
```

### ECS

为什么能形成局部变量？ ECS就是解释。 

因为在 javascript 中，__只有函数执行才会形成一个 Excecution Context__ 所以 AO 也只能在 EC 中才会存活。

js 对于 EC 的执行是基于 stack 的。刚开始的时候 stack 中只有一个 globalContext 的存在。 当执行各种函数的时候，js 就会将这个函数的EC push 进 stack 里面。所以当前的 Context 就是这个函数的 Context 了。

当函数执行完毕的时候 js 会自动将 stack 的首部 pop 出去。所以 刚才 function() 当中的所有局部的 AO 就都被销毁掉了。

这就是作用域的基本原理了。

### scope

然而 ECS 并不是作用域的全部。因为在 ECS 中，变量不光可以访问本域中的 var 还可以访问 global 的各种 var，所以就引出了另一个概念： 作用域链。

对于 head 的 EC 它怎么能知道 global EC 的内容呢？因为在每个 EC 中(除了global EC) 还有一个属性: Scope

> 作用域链与一个执行环境相关，变量对象的链用于在标识符解析中变量查找

一句话：作用域链Scope其实就是对执行环境EC中的变量对象VO|AO有序访问的链表。能按顺序访问到VO|AO，就能访问到其中存放的变量和函数的定义。

``` javascript
Scope = [AO].concat([[Scope]]);
```

AO 顾名思义就是说在在这个传说中的作用域链的最前端。 因为是链表，所以它的 head 肯定是 AO 嘛。

真正的实现好像是用 array 做的。

[[scope]] 又是什么鬼？

> [[Scope]]是一个包含了所有上层变量对象的分层链，它属于当前函数上下文，并在函数创建的时候，保存在函数中。

可以这么理解

``` javascript
AO.next = EC.parent.AO;
```

这种时候用链表简直天经地义嘛。

作用链接这个概念比较坑爹，因为实现的机制又是不一样。

> 每当见到 `function` 关键字的时候 js 就会创建 [[scope]] 静态属性。

我们可以看到静态属性就是一个不可更改的属性，有点类似 java 的 final const 这种。

但是在 js 当中这个属性是一个完全隐藏的， 1. 不可调用 2. 不可更改的静态的属性。

``` javascript
var x=10;
function f1(){
  var y=20;
  function f2(){
    return x+y;
  }
}

上面的这一段可以理解成是这样的：

``` javascript
f2.[[scope]]=[
  f2OuterContext.VO
];
```

所以实际上 f2 实际的 Scope 是这样的:

``` javascript
f2.[[scope]]=[
  f1Context.AO,
  globalContext.VO
];

f2.Scope = [
  this.AO,
  fiContext.AO,
  globalContext.VO
];
```

至于作用域更加具体的各种坑留给后面的笔记再写了。

参考: 

* [javascript 执行环境，变量对象，作用域链](http://segmentfault.com/a/1190000000533094)
* [汤姆大叔的博客](http://www.cnblogs.com/tomxu/archive/2012/01/16/2309728.html)
* _javascript 高级程序设计_
