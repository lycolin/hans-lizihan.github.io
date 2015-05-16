---
layout:     post
title:      javascript event(2)
date:       2015-05-10 10:30
summary:    javascript event
categories: javascript
---

# 常用事件

## 1 UI 事件

### load

> 当页面完全加载完毕之后（􏱭􏳙所有􏲯􏱄􏲷JavaScript, CSS, 图片􏰁􏰫􏳿􏰩􏶮􏶯）之后触发

这就是传说中的 `window.onload` 了。 之前一直以为是当文档开始加载的时候就会触发这个事件，今天终于明白了原来是所有资源加载完毕之后才会触发

此外其实 `DOM2` 中规定 `load` 事件应该在 `document` 上面触发而不是 `window` 但是由于历史原因，现实里面我们还是调用 window.onload

除了 window 之外，理论上所有的资源都可以触发 `load` 事件 包括 `<img>`, `<script>` 等等 所以在图片加载时显示 loading 图标可以借助这个事件

``` javascript
img.addEventListener('load', function (event) {
  // remove loading class
});
```

### unload

> 当页面被关闭的时候触发

### resize

> 当窗口高度或者宽度发生变化的时候触发

这个事件一般的用处是根据窗口大小来调整一些元素的宽度/高度等等

注意，最小化浏览器的时候也会触发这个事件

### scroll

> 当元素被滚动的时候触发

这个事件可以事件懒加载或者各种酷炫的动画，太特么有用了，值得注意的时候因为这个事件会被重复触发，所以可能要借助类似 `setInterval` 之类的函数尽量减少代码逻辑的执行

## 2 焦点事件

### blur/focus

这两个事件主要应用在表单上，在验证表单的领域里面这两个事件极为重要

> 当元素失去焦点/得到焦点时触发

``` javascript
var input = document.getElementById('text-input');
input.onblur = function() { // also works onfocus
  alert('hihi');
}
```

但是这两个事件有一个共同点： 不冒泡。 就是说这两个事件只会发生一次，天生不会传递到父级元素里

### focusin/focusout

大部分情况都可以被 blur/focus 解决，但是如果实在需要在父级元素中触发事件并且处理事件那么就要借助这两个了

### 焦点事件触发顺序

除了上面的4种事件之外，焦点事件还有几种 触发顺序如下

1. focusout
2. focusin
3. blur
4. DOMFocusIn
5. focus
6. DOMFocusOut

## 鼠标与滚轮事件

### 鼠标事件

> 当元素被点击时触发

主要有几种 调用顺序如下

1. mousedown
2. mouseup
3. click

有一个特殊的 `dbclick` 事件是在用户双击某一个元素的时候触发

值得注意的是 `click` 由 mousedown + mouseup 触发， `dbclick` 由 `click` + `click` 触发

#### 事件对象中位置属性

clientX clientY, 这 两个概念是针对 __目前__ 浏览器中呈现的区域的 X 坐标 和 Y 坐标
pageX pageY, 对应的是对 __整个__ 浏览器中的位置
screenX screenY, 则是对于 __整个屏幕__ 中的位置

#### 事件对象中的修改键属性

所谓的修改键其实就是当点按一个键盘上的键的时候同时点按屏幕，有时候这种小操作非常有利于某些复杂功能的实现：文本编辑器 etc.

1. shiftKey
2. ctrlKey
3. altKey
4. metaKey (cmd for mac, windows for PC)

#### 事件对象中相关元素属性

这个属性是实现各种酷炫拖拽效果的基石，比如对于一个 button 来说，点按时鼠标在 button 上面，松开左键的时候鼠标在 button 外面，这时候就会 在 事件 对象上面形成一个 `relatedTarget` 属性包含鼠标移出时的元素

这个属性只对于 `mouseover` `mouseout` 有效 

### 滚轮事件

滚轮事件是实现目前各种 one page scroll 的基础。但是悲剧的是这个事件居然是二笔 IE 最早做的实现。。。 不过没关系，html5 已经正式加入了这个事件 `mousewheel`
这个事件最终会冒泡到 document 里面

``` javascript
document.onmousewheel = function(event) {
  event.wheelDelta; // + for moving downward, - for up
}
```

其中比较令人感兴趣的就是这个 wheelDelta 了

## 3 键盘事件

键盘事件分成几个过程，当一个键盘敲击的事后分成下面三个过程

1. keydown
2. keypressed
3. keyup

其中要注意的是当用户一直按住案件不放的时候 keydown 事件会被重复触发，就是说这个事件会持续给文本框输入字符, 但是注意 `input` 中 change的事件会在 `keyup` 之后被触发

其中 keydown 仅仅是描绘某一个键被按下去了而不会真正触发这键的默认功能， 而 `keypress` 则是一个键中的字符被截取并且触发基本打字功能

另外有一个文本时间  `textInput` 这个事件在用户键入之后不会第一时间显示在屏幕上面，而是先经过一个 handler

## 4 复合事件

这个事件本来是一个比较Minus 的事件，但是作为中文用户一定要明白，因为中文输入法其实是将几个拉丁字符转化成为某一个中文字符的，那么在输入一个中文字符的时候就会触发一个复合事件

其中一个非常常见的缩写 __IME__ (Input Method Editor) 当然了，操作系统中的各种输入法其实并没有经过这一层，这一层有点像 google 云端中原生提供的中文输入法

简单知道下有这么几个东西就好  `compositionstart`, `compositionupdated`, `compositioned`

## 5 变动事件

这类事件主要是在听 DOM 本身的变动 

知道几个例子就好了 `DOMNodeInserted`， `DOMNodeRemoved`

## 6 HTML5 事件

HTML5 中新加入了一些事件，90%都是为了更好的网页交互而设计的

### contextmenu

简单点说者就是传说中的右键功能。 目前来看只有 google 对于这个事件做出了比较好的应用。 在 google documentation 里面点击右键的时候不时出现浏览器本身默认的menu 而是出现了一个google定制的右键菜单

### beforeunload

这个事件是在当前tab被关闭的时候触发的，大部分是时候应该是当一个用户正在输入一张表的时候提醒用户是否真的要离开页面 不过不同正常的事件处理，这个事件接受一个返回值

``` javascript
window.onbeforeunload = function(event) {
  return 'are you really want to leave the page'?
}
```

并且把这个返回值作为一个 `prompt` 对话框返回给用户

### DOMContentLoaded

这个事件是 h5中新加入的一个loading事件。 之前说过 `window.onload` 的弊端：等所有资源全部加载完毕之后才会被触发，包括图片。 所以应该有一个更好的加载时间在所有的文档字符串被加载完毕之后就立刻执行。 所有有了这个事件。

其实 `jQuery` 中的 `$(function() {})` 也就是 `$(document).ready(function(){})` 就是用的这个原理

对于不支持该事件的二笔浏览器还有一个牛逼爆了的hack

``` javascript
setTimeout(function() {
  // do something here
}, 0)
```

setTimeout 会在浏览器执行完毕所有的 js 之后才执行

### readyStateChange

大名鼎鼎的 readyStateChange 登场。 对于这个事件其实更多的接触应该是 `xmlhttprequest` 里面的

这个事件对象中有一个属性 `readyState`
分为下面5个状态

虽然在dom加载中 readyStateChange 并没有什么用武之地但是后面的 xhr 还是有它的一席之地的

1. uninitialized
2. loading
3. loaded
4. interactive
5. complete

### hashChange

不多说，听取 浏览器 url 的 hash 

## 设备事件

接触应该比较少，知道下 `orientationchange` 手机的方向 `deviceorientation` `devicemotion`

## 触摸和手势事件

知道下就好 触摸：

1. touchstart
2. touchmove (preventDefault会阻止滚动)
3. touchend
4. touchcancel

手势：

1. guesturestart
2. guestureend
3. guesturechange


参考:

* *javascript 高级编程*

连续剧: [javascript event(1)](/javascript/2015/05/03/javascript-event(1).html)

happy coding, may the code will always be with you~
