---
layout:     post
title:      javascript 常用工具详解
date:       2015-08-09 16:00
summary:    javascript 常用工具 原生 API
categories: javascript
---

这两天写 javascript 的时候经 看到同事引入了一个 `util.js` 
看了看源代码很少但是感觉自己看了之后老是一知半解的，所以特此来写下这个小的 工具到底做了些什么。

## 1 dataUrltoBlob

prerequisite: 

这段代码用了很多 html5 中新的 API 所以突然出现了很多不太认识的类和 原生 API 调用(所实话这个有点难) 所以先过一遍这些东西

### 1 首先复习下 dataUrl 的知识

这个是 [RFC2397](https://tools.ietf.org/html/rfc2397) 的定义

`data:[<mediatype>][;base64],<data>`

e.g. `data:image/gif;base64,R0lGODlhyAAiALM...DfD0QAADs=`

### 2 atob (ascii data back to binary)

首先这个特性 IE10+

这个函数是 html5 中的 `recommendation` 并且在 5.1 中变成了 `living standard`

signiture 简单到爆了 知识简单的将 ascii 转码成为二进制而已

``` javascript
var decodedData = window.atob(encodedData);
```

[MDN-atob](https://developer.mozilla.org/en-US/docs/Web/API/WindowBase64/atob)

### 3 ArrayBuffer

[array buffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)的定义是一段定长的二进制数据缓存。 这个缓存不可以被直接操作，但是可以使用 `类型化数组` 来创建一个 `view`

所谓 `view` 就是不同的类型对二进制文件有着完全不同的格式，所以叫做 `view`。 在 `view` 中就可以操作这个 buffer 了。


### 4 Uint8Array

这个就引出来了 javascript 中`类型化数组` 这个概念。 这种数组有点像传统的 C 数组，要有固定的长度和固定的类型。

比如说 `Uint8Array` 就是只装 8 位的 integer

类似的共有8种类型可以选择 `Int8`, `Uint8`, `Int16`, `Uint16`, `Int32`, `Uint32`, `Float32`, `Float64`

`类型化数组` 显著的优势就是性能。 在计算量比较大的字节操作和图形处理中，类型化数组提供的 2~3 倍的性能优势非常重要

类型化数组可以接受两种构造方法

1. new Uint8Array(length);
2. new Uint8Array([1,2,3,4,5]);

### 5 Blob

`blob` (binary large object)

这个东西说白了就是个黑盒子。里面是个二进制的东西，坑是音频也可能是视频

BlobBuilder 已经被废除了，出于兼容性考虑下面代码还是做了一些让步 [BlobBuilder Obsolete](https://developer.mozilla.org/en-US/docs/Web/API/BlobBuilder)

看起来这个 `BlobBuilder` 被新的 `Blog` constructor 给替代了

三个方法

1. Blob(array, options)
2. size 二进制大小 返回 bytes
3. type 返回 mime type

好了有这个前提就可以来逐句翻译这段代码了

``` javascript
$.dataURLtoBlob = function(data) {
  // 取到了 `.gif`
  var mimeString = data.split(',')[0].split(':')[1].split(';')[0];
  // 将 base64 string 转化成二进制 string
  var byteString = atob(data.split(',')[1]);
  // 构造一个 buffer 给即将构造的 Blob
  var ab = new ArrayBuffer(byteString.length);
  var ia = new Uint8Array(ab);
  // 因为 Uint8Array 这能存储 数字 所以必须将所有的string 都转义成数字
  for (var i = 0; i < byteString.length; i++) {
    ia[i] = byteString.charCodeAt(i);
  }
  // Blog 兼容性考虑
  var bb = (window.BlobBuilder || window.WebKitBlobBuilder || window.MozBlobBuilder);
  if (bb) {
    // 老版本写法
    bb = new (window.BlobBuilder || window.WebKitBlobBuilder || window.MozBlobBuilder)();
    bb.append(ab);
    return bb.getBlob(mimeString);
  } else {
    // 新版本写法
    bb = new Blob([ab], {
      'type': (mimeString)
    });
    return bb;
  }
};
```

呼 长知识了，之前都没有仔细考虑过 dataUrl 和二进制文件之间的关系。

## 2 convertImgToBase64

这个方法也是很重要的，但是它的实现有点 hacky 借助了 h5 中 `canvas` 的一点小特性

这里必须来预习一下 `Image` 对象

### Image

``` javascript
Image(width, height);
```

期中有一个非常模糊地属性 `img.crossOrigin`。这个是啥意思？

是这样的，浏览器实现的静态的 `<img>` 标签可以自由的直接从其他域里面拽东西下来。(其实就是一个 ger request) 这时候浏览器是不会做出CORS的警告的。

如果说用程序做出来一个 `Image` 就不一样了。这种时候就要告诉浏览器你需要怎么做

1. 默认：不跨域。如果跨域就会被打回来
2. 'anonymous'：跨域但是不传 cookie
3. 'use-credentials': 跨域而且要求服务器必须传回来 `acceess-allow-credentials: true` + cookie

``` javascript
$.convertImgToBase64 = function(url, callback, outputFormat) {
  var canvas = document.createElement('CANVAS');
  var ctx = canvas.getContext('2d');
  var img = new Image();
  // 告诉浏览器我们想跨域但是不需要 cookie 那些破玩意儿
  img.crossOrigin = 'Anonymous';
  // 一个异步函数
  img.onload = function() {
    canvas.height = img.height;
    canvas.width = img.width;
    ctx.drawImage(img, 0, 0);
    // 这就是重点了。因为 `canvas` 才有一个 toDataURL 的方法。
    // 我们好像并不能读取一个非 `input` 中 file 的 Blog 所以必须这么做
    var dataURL = canvas.toDataURL(outputFormat || 'image/png');
    callback.call(this, dataURL);
    // Clean up
    canvas = null;
  };

  // 同步中: 发起图片的 request
  img.src = url;
};
```

## 3 serializeObject

这个函数有时候比较常用，不过现在用 RESTful 就比较少了。因为基本直接传 Json 了

``` javascript
$.fn.serializeObject = function () {

  var result = {};
  // 因为 jQuery.serializeArray 会返回一个这样的类型
  // [{name:'foo', value: 'bar'}]
  var extend = function (i, element) {
    // 1 如果 result 中没有这个 name 那么 undefined
    // 2 如果 result 中已经有了 那么就拿出来
    var node = result[element.name];

    // If node with same name exists already, need to convert it to an array as it
    // is a multi-value field (i.e., checkboxes)
    // [{name: 'checkbox', value: 1}, {name: 'checkbox': value: 2}]
    // => {name: 'checkbox', value: [1,2]}

    // !isEmpty(node)
    // node 应该是一个 checkbox 
    if ('undefined' !== typeof node && node !== null) {
      // 如果说 node 已经是一个 array 了 那么就 push 进去
      if ($.isArray(node)) {
        node.push(element.value);
      } else {
        // 将原本的 string 转化为 array
        result[element.name] = [node, element.value];
      }
    } else {
      // 正常情况下应该都是 key => value 这样的名值对
      result[element.name] = element.value;
    }
  };

  $.each(this.serializeArray(), extend);
  return result;
};
```

这个函数应该算是比较简单的了

## 4 convertHex

这个应该是着一些里里面最简单的，唯一要知道的 API 就是 

### parseInt

这个函数有两个参数

``` javascript
parseInt(string, radix);
```

radix 就是要将这个字符串本来是多少进制的, 默认为 10

e.g.

``` javascript
parseInt(100, 2); // => 2
```

``` javascript
$.convertHex = function(hex, opacity) {
  // 将 '#' 扔掉
  hex = hex.replace('#', '');
  // 分别将 r g b 的值转移成10进制
  var r = parseInt(hex.substring(0, 2), 16);
  var g = parseInt(hex.substring(2, 4), 16);
  var b = parseInt(hex.substring(4, 6), 16);

  if (!opacity) opacity = 100

  return 'rgba(' + r + ',' + g + ',' + b + ',' + opacity / 100 + ')';
};
```

## 4 getParameterByName

欢迎来到 regex 天堂

``` javascript
$.getParameterByName = function(name) {
  // foo[bar] -> foo\[bar\]
  name = name.replace(/[\[]/, "\\[").replace(/[\]]/, "\\]");
  // [\\?&]foo\[bar\]=([^&#]*)
  // -> foo[bar]后面直到不是 # 或者 &
  var regex = new RegExp("[\\?&]" + name + "=([^&#]*)"),
      results = regex.exec(location.search);
  // 如果没找到那就是没找到
  // 周到了就把value弄回来 因为加了分组 () 所以 $1 就直接是了
  // 因为空格一般会被 `+` 替代，所以decode回来时会吧 `+` 转义回空格
  return results === null ? "" : decodeURIComponent(results[1].replace(/\+/g, " "));
};
```

## 6 scrollTo

这个方法就涉及到了 jQuery 的 animate 方法 比较简单了

``` javascript
$.fn.scrollTo = function(target, options, callback) {
  // 2 argument only options + callback
  // 3 arguments, add a target element in the front
  if (typeof options === 'function' && arguments.length === 2) { callback = options; options = target; }
  var settings = $.extend({
    scrollTarget  : target,
    offsetTop     : 50,
    duration      : 500,
    easing        : 'swing'
  }, options);
  return this.each(function() {
    var scrollPane = $(this);
    var scrollTarget = (typeof settings.scrollTarget === "number") ? settings.scrollTarget : $(settings.scrollTarget);
    var scrollY = (typeof scrollTarget === "number") ? scrollTarget : scrollTarget.offset().top + scrollPane.scrollTop() - parseInt(settings.offsetTop);
    scrollPane.animate({scrollTop : scrollY}, parseInt(settings.duration), settings.easing, function() {
      if (typeof callback === 'function') { callback.call(this); }
    });
  });
};
```

## 7 humanFileSize

不解释了 太直白

``` javascript
$.humanFileSize = function(bytes, si) {
  var thresh = si ? 1000 : 1024;
  if (Math.abs(bytes) < thresh) {
    return bytes + ' B';
  }
  var units = si        ?
              ['kB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB']
      : ['KiB', 'MiB', 'GiB', 'TiB', 'PiB', 'EiB', 'ZiB', 'YiB'];
  var u = -1;
  do {
    bytes /= thresh;
    ++u;
  } while (Math.abs(bytes) >= thresh && u < units.length - 1);
  return bytes.toFixed(1) + ' ' + units[u];
};
```

