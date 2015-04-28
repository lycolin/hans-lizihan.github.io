---
layout:     post
title:      javascript 运算符
date:       2015-04-28 11:51
summary:    javascript 基础篇之运算符
categories: javascript
---

# Javascript 运算符

javascript 中最最坑爹的就是运算符了， 作为一门弱类型语言，运算符的处理本来就比较棘手，来到 js 的世界就更加棘手了， 从家喻户晓的 `===` 到 string 的加减法， js 中运算符的坑娃沟坎无处不在

首先抛出一个大表 (结核性中 R 表示右结合)(参考 [Samaritans](http://www.cnblogs.com/dolphinX/p/3524977.html) 大大的博客 (POI 粉握手) 和 `javascript 权威指南`)

注意其中两个概念非常吸引人：

__操作数__

说白了就是几元操作符,+-\*/都得有两个基本元素配合才能构成一个表达式，而 !,++ 什么的只需要一个 ?: 则需要3个

``` javascript
++a // 1 operation
a + b // 2 operations
a?b:c // 3 operations
```

__结核性__

正常的加减乘除都是 __左结合__

``` javascript
1 + 2 + 3 === (1 + 2) + 3
```

有些运算符则是 __右结合__

``` javascript
a = b = c = d === a = (b = (c = d))
a:b:c?d:e?f:g === a?b:(c?d:(e?f:g));
```


---

运算符         | 操作           | 结合性    | 操作数    | 类型
:------------ |:------------- |:-------- |:-------- |:---------------
1             |               |          |          |
++            | 前，后自增      | R        | 1        | lval -> num
--            | 前，后自减      | R        | 1        | lval -> num
-             | 求反           | R        | 1        | num -> num
+             | 转换成数字      | R        | 1        | num -> num
~             | 按位求反        | R        | 1        | int -> int
!             | 逻辑非         | R        | 1        | bool -> bool
delete        | 删除属性       | R        | 1        | lval -> bool
typeof        | 检测操作数类型   | R        | 1        | any -> str
void          | 返回 undefined | R        | 1        | any -> undefined
2             |               |          |          |
\*, /, %      | 乘，除，求余    | L        | 2        | num, num -> num  
3             |               |          |          |
+, -          |  加，减        | L        | 2        | num, num -> num  
+             | 字符串链接      | L        | 2        | str, str -> str  
4             |               |          |          |
<<            | 左移位         | L        | 2        | int, int -> int
\>\>          | 有符号右移      | L        | 2        | int, int -> int  
\>\>\>        | 无符号右移      | L        | 2        | int, int-> int
5             |               |          |          |
<、<=、>、>=   | 数字大小顺序     | L        | 2        | num, num -> bool
<、<=、>、>=   | 字母大小顺序     | L        | 2        | str, str -> bool
instanceof    | 测试对象类      | L        | 2        | obj, func -> bool
in            | 测试属性是否存在 | L        | 2        | str, obj -> num
6             |               |          |          |
==            | 判断相等       | L        | 2        | any, any -> bool  
!=            | 判断不等       | L        | 2        | any, any -> bool  
===           | 判断恒等       | L        | 2        | any, any -> bool  
!==           | 判断非恒等      | L        | 2        | any, any -> bool  
7             |               |          |          |
&             | 按位与         | L        | 2        | int, int -> int
8             |               |          |          |
^             | 按位异或        | L        | 2        | int, int -> int
9             |               |          |          |
&#124;        | 按位或         | L        | 2        | int, int -> int
10             |               |          |          |
&&            | 逻辑与         | L        | 2        | any, any -> any
11            |               |          |          |
&#124;&#124;  | 逻辑或         | L        | 2        | any, any -> any
12            |               |          |          |
?:            | 条件运算符      | L        | 3        | bool, any -> any  
13            |               |          |          |
=             | 变量赋值或者对象属性赋值 | R  | 2         | lval, any -> any  
[运算符]=      | 运算且赋值      | R        |2          | lval, any -> any
14            |               |          |          |
,             | 忽略第一个，返回二| L        | 2        | any, any -> any

---

markdown 画table的确有点残疾。。。 不要在意这些细节

重点在于这个表的价值是无限的，整个 js 语言的各种精髓细节都在这张表里了

首先看到

__`+` 和 `-` 不止 “加” 和 “减” 的功能__

可能这就是长久以来一道运算符就觉得要坑的罪魁祸首

## `+`
`+` 有三种用法

1 转换成数字 (number, boolean)

``` javascript
+1 // 1
+0.1 // 0.1
+.1 // 0.1
+-.1 // -0.1
+(-1) // -1
+-1 // -1
+true //1
+false //0
+[] //0
+[0] // 0
+[1] // 1
+[0,1] // NaN
+{} //NaN
+{0:1} //NaN
+'1' === 1 // true
+'1a'// NaN
+'a' // NaN
+'ab' // NaN
```

总结一下, 转换数字优先级巨高, 但实际见到的机会不多，因为大部分时候应该没人怒在行首写 + 号吧
此外，感觉单元素数组引用首位元素，所以 + 可以转化单元素数组的元素
+ 还可以比较严格地将可以转化为数字的字符串转化为数字

2 加法 (Number, Boolean) 比较简单

``` javascript
// Number + Number -> addition
1 + 2 // 3

// Boolean + Number -> addition
true + 1 // 2

// Boolean + Boolean -> addition
false + false // 0

```

3 拼接字符串 (String, Boolean, Number)

``` javascript
// Number + String -> concatenation
5 + 'foo' // '5foo'

// String + Boolean -> concatenation
'foo' + false // 'foofalse'

// String + String -> concatenation
'foo' + 'bar' // 'hoobar'

// Number addition has higher priority
1 + 3 + 'a' // '4a'
```

## -
减就两种用法

1 求反

``` javascript
-1 // -1
-'a' // NaN
-'-1' // 1
-'1a' // NaN
-[1] // -1
-{} // NaN
-{0:1} // NaN
-true // -1
-false // -0
```

总体来说还是能够预估到的，如果true false 不算的话行为和 + 一样
不过有了 - + 之后 true false 就会强制被转化为数字来计算

2. 减法 因为没有了 + 拼接字符串的功能， - 相对正常些

``` javascript
5 - 3 // 2
3 - 5 // -2
'foo' - 3 // NaN
```

其次就是优先级的问题了

## typeof 的超高优先级

``` javascript
typeof 2 * 3 //NaN =>'number' * 3
typeof (2 * 3) // 'number'
typeof 2 + 3 // 'number3' => 'number' + 3
```

## ++,--

首先常量是不可以自增的

``` javascript
4++; //ReferenceError: Invalid left-hand side expression in postfix operation
```

其次不要被一堆 + 整晕

``` javascript
var a = 0,b = 0;
a+++b;//0
a;//1，++优先级比+高，所以相当于(a++)+b
b;//0
```

``` javascript  
var a=1;
b=(a=3)+a++; // 6
b //6
a //4
```

1. 计算b
2. a＝3
3. a++(设为c)
4. 计算a（这时候a变成了4已经，不是再最后才变得，但表达式使用的是a++的结果c，也就是a原来的值）
5. 计算3+c
6. 把3+c赋值给b

这里要注意的就是 __赋值运算的返回值是左值被赋予的值__

## 逻辑运算符 &#124;&#124; ,   &amp;&amp; , &#124; , &amp;

或、且运算符的优先级太低了，加减乘除都不如，所以一定不要忘了加括号

``` javascript
1 && 3; // 3 => 1 is true -> 3
1 && "foo" || 0 // "foo"
1 || "foo" && 0 // 1
```

这里就是考察短路原理和表达式自身的 __返回值__ ->

__常量的返回值就是自身__

## == 和 ===
常识告诉我们 __永远用 ===__

但是其中有一个小例外，就是 `NaN`

``` javascript
NaN === NaN // false
```

相等的原理是这样的

1. 如果两个值类型相同，则执行严格相等的运算
2. 如果两个值的类型不同
    1. 如果一个是null，一个是undefined，那么相等
    2. 如果一个是数字，一个是字符串，先将字符串转为数字，然后比较
    3. 如果一个值是true/false则将其转为1/0比较
    4. 如果一个值是对象，一个是数字或字符串，则尝试使用valueOf和toString转换后比较
    5. 其它就不相等了

``` javascript
null == undefined // true
NaN == NaN // false
'1' == true // true
```



参考 \* [阮大神的博客](http://www.ruanyifeng.com/blog/2014/03/undefined-vs-null.html)
    \* [Samaritans](http://www.cnblogs.com/dolphinX/p/3524977.html)
    \* *javascript 权威指南*

happy coding, may the code will always be with you~
