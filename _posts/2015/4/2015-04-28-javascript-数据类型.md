---
layout:     post
title:      javascript 数据类型
date:       2015-04-28 22:53
summary:    javascript 基础篇之数据类型
categories: javascript
---

# JS 数据类型

js 作为弱类型语言，数据类型的坑简直多崩

其实 js 的只有两大种 类型

1. Primitives
    1. number
    2. string
    3. boolean
    4. null
    5. undefined
2. References
    1. object
    2. function
    3. date
    4. array


其中尤其重要的一点就是， 所有的 primitives 都是不可更改的

所以有下面的情况

``` javascript
1++ // ReferenceError: Invalid left-hand side expression in postfix operation
```

还有所有的 string 也是不可更改的， 所有 `substring`, `slice` etc 之类的方法都只是做了一条新的 `string` 而不是在内存的级别上面修改 `string`


## undefined 和 null 的 爱恨情仇

undefined 会被解析成一个 string, null 会被解析成 0 或者 false

``` javascript
var a = null;
a + 1 // 1
a * 1 // 0
1 / a // Infinity

a = undefined;
a + 1 // NaN
a * 1 // NaN
1 / a // NaN
```

此外二者的 type 也不同

``` javascript
var a = null
typeof a // object
var b = undefined
typeof b // "undefined"
```

二者的情景也不同

### null
null 表示这里不应该有东西

1. 作为函数的参数，表示该函数的参数不是对象。
2. 作为对象原型链的终点。

``` javascript
Object.getPrototypeOf(Object.prototype); // prototype 终点
```


### undefined
undefined表示"缺少值"，就是此处应该有一个值，但是还没有定义。典型用法是：

1. 变量被声明了，但没有赋值时，就等于undefined。
2. 调用函数时，应该提供的参数没有提供，该参数等于undefined。
3. 对象没有赋值的属性，该属性的值为undefined。
4. 函数没有返回值时，默认返回undefined。

## 包装对象

这简直坑爹

``` javascript
var s = 'test';
console.log(s.indexOf(e)); //1 !! 说好了 primitive 呢？
console.log(s.length); //4
s.len=4;
console.log(s.len);//undefined
```

原理是这样的

1. 引用了 `string`
2. 每次调用 s 方法的时候 js 内部 new String(s) 创建一个临时对象 并将引用传给 s
3. 调用完了 `indexOf` 临时对象被销毁
4. 每次调用都经历的 new -> destruct 这个过程
5. 所以说到底 s.len = 4 将属性 len 给了 包装对象
6. 包装对象接到 len 的一瞬间就把自己个毁了
7. 所以泥牛入海， log 的时候就发现 undefined 了

## 类型自动转换

防不胜防

``` javascript
var a=[0], b=a, b=[1];
console.log(a+b); //[0] + [1] '01'
```

数组相加则数组都自动调用了 .toString() 方法

``` javascript
[1,2,3] + [4,5,[6,7]] //"1,2,34,5,6,7"
```

减法就没那么幸运了，因为 `-` 并没有 `+` 的 concat 功能

``` javascript
[1,2,3] - [4,5,[6,7]] // NaN
```

事实上 js 的隐式自动转换时时刻刻都在发生

``` javascript
Number('3');//3
String(false);//false
Boolean([]);//true
Object(3);//new Number(3)
1+'234';//"1234"
5+'';//String(5)
+'5';//5，变成了数字 Number("5")
'5'-0;//5,也是数字
```

有一些已经在 运算符 那一堆里面看到过了，但是诸如 `Object(3)` 引发 new Number(3) 这种还是要小心

## 类型转换对照表

---
Value     | String           | Number | Bool |
:-------- |:---------------- |:------ |:---- |
undefined | 'undefined'      | NaN    | false|
null      | 'null'           | 0      | false|
false     | 'false'          | 0      | NA   |
true      | 'true'           | 1      | NA   |
''        | NA               | NaN    | false|
'string'  | NA               | NaN    | true |
'2.5'     | NA               | 2.5    | true |
0         | '0'              | NA     | false|
1         | '1'              | NA     | true |
NaN       | 'NaN'            | NA     | false|
Infinity  | 'Infinity'       | NA     | false|
[2]       | '2'              | 2      | true |
[2,[2,3]] | '2,2,3'          | 2      | true |
{}        | '[object Object]'| NaN    | false|
{1:1}     | '[object Object]'| NaN    | true |
---

## toString 和 valueOf

### object to string

1. 如果 object 内置了 toString 方法则调用它，如果toString返回 primitive，那么 js 将这个 primitive 转化成 string
2. object 没有 toString 方法, 或者 toString 方法没有返回 primitive ，那么 js 会 fallback 到 valueOf 方法，如果存在 valueOf 方法并且 valueOf 方法 返回 primitive, 则返回转化为字符串的 primitive
3. otherwise 报错

平时的 console.log/ alert / String 等等方法实际上都是隐式调用了 toString 方法， 而涉及运算的 Number或者预算符号操作则隐式优先调用 valueOf 方法

``` javascript
var o = {
  x: 8,
  valueOf: function(){
    return this.x + 2;
  },
  toString: function(){
    return this.x;
  }
};
console.log(String(o));//"8"
console.log(Number(o)); //10
console.log(o+1);//11，运算相关
alert(o);//"8"，显现相关
```

## instanceof 小怪癖

这里面其实最最变态的一个就是这个

``` javascript
1 instanceof Number; //false
new Number(1) instanceof Number; //true
```

primitives instanceof 永远会是 false 记住就是

## 应用的深入理解

javascript 将 array 和 object 都当做是 reference ，所以每次构造出来一个 Object 或者 Array，并将它赋值给了一个变量的时候实际上是讲这个应用传给了它

``` javascript
var a = [1,2,3];
var b = [1,2,3];
a === b; // false
```

所以当比较两个相同的 [1,2,3] array 的时候，实际上我们是比较了两个不同的内存地址，所以 === 会返回 false

此外因为是引用所以在修改引用的时候引用本身也会被修改

``` javascript
var a = [1,2,3];
var b = a;
a [0] = 2;
b // [2,2,3]
```

所以回想之前的 array#sort 和 array#slice 现在终于明白了， 诸如 `sort` 这种方法，是直接修改引用, 而 `slice` 方法是赋值首层引用，__所以 `slice` 只能做到浅层拷贝__

参考 \*: 继续感谢 @[Samaritans](http://www.cnblogs.com/dolphinX/p/3524977.html)

happy coding, may the code will always be with you~
