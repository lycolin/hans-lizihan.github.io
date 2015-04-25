---
layout:     post
title:      Angular service provider factory 
date:       2015-04-12 11:06
summary:    Angular services
categories: angular javascript
---

# Angular.js factory service provider value constant 

最近在做一个地图的 SAP 发现 angular.js 非常适合制作这一类的 app (感谢 $watch 实现的数据同步， 使得地图搜索可以无痛同步)

但是问题来了 在制作 model 的时候发现 `factory` `service` `provider` 不知道应该用哪个 所以上网找了很久的解释，现在终于明白了那么一点点 

## `provider`

``` javascript
function provider(name, provider_) {
  assertNotHasOwnProperty(name, 'service');
  // angular 检查 di
  if (isFunction(provider_) || isArray(provider_)) {
    provider_ = providerInjector.instantiate(provider_);
  }
  // 由此看到 angular provider object 中必须有 $get 属性
  if (!provider_.$get) {
    throw $injectorMinErr('pget', "Provider '{0}' must define $get factory method.", name);
  }
  return providerCache[name + providerSuffix] = provider_;
}
```

## `factory`

``` javascript
  function factory(name, factoryFn, enforce) {
    // wtf 原来丫就是原版不动实现了 provider ....
    return provider(name, {
      $get: enforce !== false ? enforceReturnValue(name, factoryFn) : factoryFn
    });
  }
```

## `service`

``` javascript
  function service(name, constructor) {
    // wtf 这货居然也是原班不动 实现了 facotry ...
    return factory(name, ['$injector', function($injector) {
      return $injector.instantiate(constructor);
    }]);
  }
```

好吧 源代码看完了 似乎这个逻辑是这样的

> service < factory < provider

三个东西实现的东西一样但是只是写法不同， 父类有着比子类更多的设置选项， 而子类则有比父类更精简的代码

其中 service 更是一个简单 factory wrapper

下面的例子可以看到 可以理解 service 是一个 constructor, 使用的时候就自动被实例化了

而 factory 则多了一步，我们要再factory 里面自己定义 class 然后返回 这个实例

``` javascript
app.service('foo', function() {
  var thisIsPrivate = "Private";
  this.variable = "This is public";
  this.getPrivate = function() {
    return thisIsPrivate;
  };
});

// 等价于 

app.factory('foo2', function() {
  return new Foobar();
});

// 等价于
app.service('foo3', Foobar);


function Foobar() {
  var thisIsPrivate = "Private";
  this.variable = "This is public";
  this.getPrivate = function() {
    return thisIsPrivate;
  };
}
```

## 单例模式

因为 service 多用于异步服务器获取数据， 所以所有的service都用 `单例模式` 进行的实现

也就是说 每一次 inject 一个 provider 的时候 provider 的 class 要不然是 new 出来的 要不然就是 IoC里面拿出来的实例

最后一个长例子可以说明了

``` javascript
var myApp = angular.module('myApp', []);

//service style, probably the simplest one
myApp.service('helloWorldFromService', function() {
  this.sayHello = function() {
      return "Hello, World!"
  };
});

//factory style, more involved but more sophisticated
myApp.factory('helloWorldFromFactory', function() {
  return {
    sayHello: function() {
      return "Hello, World!"
    }
  };
});


//provider style, full blown, configurable version     
myApp.provider('helloWorld', function() {
  // In the provider function, you cannot inject any
  // service or factory. This can only be done at the
  // "$get" method.

  this.name = 'Default';

  this.$get = function() {
    var name = this.name;
    return {
      sayHello: function() {
        return "Hello, " + name + "!"
      }
    }
  };

  this.setName = function(name) {
    this.name = name;
  };
});

//hey, we can configure a provider!            
myApp.config(function(helloWorldProvider){
  helloWorldProvider.setName('World');
});


function MyCtrl($scope, helloWorld, helloWorldFromFactory, helloWorldFromService) {

  $scope.hellos = [
    helloWorld.sayHello(),
    helloWorldFromFactory.sayHello(),
    helloWorldFromService.sayHello()];
}​
```

最后贴一下传说中的 ng-best-practice [angular-best-practice](https://github.com/johnpapa/angular-styleguide)

happy coding, may the code will always be with you~
