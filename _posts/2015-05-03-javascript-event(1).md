---
layout:     post
title:      javascript event(1)
date:       2015-05-03 19:37
summary:    javascript event 基础篇
categories: javascript
---

web 2.0 时代页面交互变得无比重要，事件作为交互的中心是 js 重中之重

事件实际上时使用了设计模式中的 `observer pattern`, 即事件监听者一直注视事件容器，然后轮询来捕获事件的发生。 一旦相应的事件发生那么监听者就会触发相应事件所绑定的回调函数来执行一些 handler 逻辑

## 事件流

事件流是 认为定义的事件的传播/捕获 流程，现在主要的事件流有两种

## 1 事件冒泡

(event bubbling) 是 ie 做的实现
它的机理是从底层传播到顶层

``` html
<html>
<head>
  <title>title</title>
</head>
<body>
  <div id="click-me"></div>
</body>
</html>
```

对于上面的 html 代码 当 `#click-me` 被点击之后 时间的传播顺序是这样的

1. `div#click-me`
2. `body`
3. `html`
4. `document`

## 2 事件捕获

(event capturing) 是 netscape 等等的实现
不同于事件冒泡的由底到顶的实现机理，事件捕获用了由顶到底的实现机理

这个事件流的出发本意是相对抽象的对象应该及早得到事件的触发而相对具体的对象应该最后被触发

所以每一个事件都是从 `window` 或者 `document` 对象开始传播

``` html
<html>
<head>
  <title>title</title>
</head>
<body>
  <div id="click-me"></div>
</body>
</html>
```

同样一个html，事件捕获的顺序是这样的

1. `document`
2. `html`
3. `body`
4. `div#click-me`

## 3 DOM 事件流

上面的两个实现合并起来就是 DOM 事件流的完全体了

DOM2规定每一个一个事件都要经历三个阶段

1. 捕获阶段
2. 处于目标阶段
3. 冒泡阶段

这三个阶段用作不同的作用

1. 捕获阶段负责父级对象对事件做出一些反应
2. 处于目标阶段负责事件本身的发生
3. 冒泡阶段负责事件的处理

## 4 事件处理

### html 事件处理

``` html
<input type="button" value="Click Me" onclick="alert('Clicked')" />
```

这是最典型的上个世纪的事件处理方法 'on' + event name

当然在 javascript 中定义一个函数之后在 html 内联中调用之也是可以的

``` html
<script>
function showMessage() {
  alert('hihi');
}
<input type="button" onclick="showMessage()" />
</script>
```

这里注意 html 内链模式调用对象可以天生直接调用两个对象 `event` `this`

其中 `event` 代表 `事件对象`

`this` 代表该元素本身


``` html
<input type="button" value="click1" onclick="alert(event.type)" /> // click

<input type="button" value="click2" onclick="alert(this.type)" /> // button
<input type="button" value="click3" onclick="alert(type)" /> // button
```

html 事件处理的背后是 javascript 默默地创建了一个动态的函数来包裹 `onclick` 中的 js 语句的逻辑

``` javascript
function() {
  with(document) {
    with(this) { // this is element
      // onclick logic here
    }
  }
}
```

这个动态函数的创建还可以嵌套在表单中

``` html
<form method="post">
  <input type="text" name="username" value="">
  <input type="button" value="Echo Username" onclick="alert(username.value)">
￼</form>
```

实际上动态函数的绑定是这样的

``` javascript
function(){
  with(document){
    with(this.form){
      with(this){ //元素属性值

      }
    }
  }
}
```

内联 html 事件处理的作用域确实很让人费解，不过好在随着时代的进步，大家发现 html 和 javascript 的高度耦合并不是什么好主意 (angularjs 哭晕在厕所) 所以大的潮流还是用纯 js 来做事件的处理

### DOM0 事件处理

``` javascript
var btn = document.getElementById('btn');

btn.onclick = function() { // attach
  alert(this.id); // use this.** to visit the element's attributes
}

btn.onclick = null; // detach
```

DOM0 事件比较本能，处理阶段默认在 bubble 阶段，而且 onclick 不可以绑定多个 handler

### DOM2 事件处理

简单讲 DOM2 事件处理就是两个核心 `addEventListener` 和 `detachEventListener`

其中 `addEventListener` 接受三个参数

1. event 名 e.g. `click` `keydown` ...
2. handler callback
3. boolean 如果 true 则在捕获阶段处理， false 则是在冒泡阶段处理事件 默认 false

``` javascript
var btn = document.getElementById('btn');
btn.addEventListener('click', function() {
  alert('hihi');
}, true);
```

这里有一个小坑，就是在 detach event 的时候必须传入 handler 的引用才能根除


``` javascript
function handler() {
  alert('hihi');
}

btn.addEventListener('click', handler, false);

btn.removeEventListener('click', handler, false)
```

由于 IE 的历史原因，大家更加钟意用 bubble 阶段来处理一个事件, (因为 w3系 很早就支持 bubble 了，而 ie 迟迟不接受 捕获)

## 事件对象

js 中 handler 的第一个参数是 `事件对象`

事件对象拥有诸多关于这个事件的信息。比如 `keydown` 事件被触发了之后 event 就会有一个 `a` 被按了 的信息

总体来讲 不同事件有不同的 event 信息，但是有一些是公用的

meth/prop        | type             | read/write | 说明                  |
:--------------- |:---------------- |:---------- |:-------------------- |
bubbles          | Boolean          | Read Only  | 事件是否冒泡            |
cancelable       | Boolean          | Read Only  | 事件默认行为是否可以取消  |
currentTarget    | Element          | Read Only  | handler正在处理的元素   |
defaultPrevented | Boolean          | Read Only  | 表示prevDef 是否被调用  |
detail           | Integer          | Read Only  | 与事件相关的细节信息     |
eventPhase       | Integer          | Read Only  | 1->捕获阶段 2->处于目标 3->冒泡阶段|
preventDefault() | Function         | Read Only  | 取消事件的默认行为(only when cancelable === true)|
stopImediatePropagation()|Function  | Read Only  | 取消事件的捕获或者冒泡，阻止所有 handler|
stopPropagation()| Function         | Read Only  | 取消事件的捕获或者冒泡   |
target           | Element          | Read Only  | 事件的目标             |
trusted          | Boolean          | Read Only  | true 是浏览器原声的， false 是开发人员自创的|
type             | String           | Read Only  | 被触发的事件的类型      |
view             | AbstractView     | Read Only  | 与事件关联的抽象视图，等于发生事件的 window 对象(iframe 相关)|

### this, currentTarget, target 的爱怨情仇

上面的表里面我们可以得知通过 this 或者 currentTarget 可以访问到触发事件的元素本身

1. 在 handler 内部，  this === currentTarget
2. 如果直接将 handler 交给了目标元素，那么三者一样

``` javascript
var btn = document.getElementById('btn');

btn.onclick = function(event) {
  console.log(this === event.currentTarget); // true
  console.log(this === event.target); // true
}
```

``` javascript
document.body.onclick = function(event) {
  console.log(event.currentTarget === document.body); // true
  console.log(this === document.body); // true
  console.log(event.target === document.getElementById('btn')); // true
}
```

在上面的第二个例子里面 触发 `click` 事件的其实是 `#btn` 但是由于在 `btn` 自己并没有 注册 handler 所以这个事件就一路冒泡到了 document.body, 被 document.body 的 handler 给处理了。

所以注意这个时候 target 就是原本的事件触发的元素，也就是 `#btn`

### preventDefault() 和 stopPropagation() 的爱恨情仇

这两个总是很像，但是做的事情不一样

1  `preventDefault()` 只负责阻止绑定于该元素的默认事件的发生，例如

``` javascript
var submit = document.getElementById('submit-btn');

submit.onclick = function(event) {
  event.preventDefault();
}
```

这个时候这张表单就不会触发默认的 `form request` 事件而跳页

2 `stopPropagation` 不负责阻碍事件的发生，但是会阻碍事件的进一步 冒泡/捕获

``` javascript
btn.onclick = function(event) {
  alert('hihi');
  event.stopPropagation();
}

document.body.onclick = function(event) {
  alert('hihi body');
}
```

上面的代码就不会触发 'hihi body' 的弹出框，因为事件已经被扼杀在摇篮里了，出不来了

### 事件的阶段

``` javascript
var btn = document.getElementById("btn");

btn.onclick = function(event){
  alert(event.eventPhase); //2
};
document.body.addEventListener("click", function(event){
  alert(event.eventPhase); //1
}, true); // true to capture the event in capture phase
document.body.onclick = function(event){
  alert(event.eventPhase); //3
};
```

### 事件对象的存活周期

注意 `event` 这个对象只有在 `handler` 里面才会被自动生成出来，当 `handler` 函数执行结束的时候 `事件对象` 会被销毁

参考:

* *javascript 高级编程*

连续剧: [javascript event(2)](/javascript/2015/05/10/javascript-event(2).html)


happy coding, may the code will always be with you~
