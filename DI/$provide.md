#依赖注入之$provide

在Angular中，提供了三种方法来构建我们需要的服务：`Service`,`Factory`和`Provider`。我们可以通过创建服务，将公用的部分注册在一个服务中，然后在需要引入的地方注入这个服务。

其中，`Service`和`Factory`都是Angular中`$provide`服务提供的方法之一，来构建可重复使用的组件。本文通过源码来讨论通过`$provide`构建的`Provider`、`Service`、`Factory`、`Value`、`Constant`、`Decorator`这几部分的差异。

![$provide构成](https://raw.githubusercontent.com/hexiaoming/ng-code/b73ed309693b14d64c7eb4deb1ba005679db3af9/images/%24provide.png)

##供应商`$provide`

源码如下：

```
////////////////////////////////////
// $provide
////////////////////////////////////

function provider(name, provider_) {
    if (isFunction(provider_)) {
        provider_ = providerInjector.instantiate(provider_);
    }
    if (!provider_.$get) {
        throw Error('Provider ' + name + ' must define $get factory method.');
    }
    return providerCache[name + providerSuffix] = provider_;
}

function factory(name, factoryFn) { return provider(name, { $get: factoryFn }); }

function service(name, constructor) {
    return factory(name, ['$injector', function($injector) {
        return $injector.instantiate(constructor);
    }]);
}

function value(name, value) { return factory(name, valueFn(value)); }

function constant(name, value) {
    providerCache[name] = value;
    instanceCache[name] = value;
}

function decorator(serviceName, decorFn) {
    var origProvider = providerInjector.get(serviceName + providerSuffix),
        orig$get = origProvider.$get;

    origProvider.$get = function() {
        var origInstance = instanceInjector.invoke(orig$get, origProvider);
        return instanceInjector.invoke(decorFn, null, {$delegate: origInstance});
    };
}
```
`$provide`中创建并包含了六种Provider方法，每种方法可以用来定义供应商。而供应商是用来提供服务，被别的模块所注入的部分就是供应商所提供的服务。

由源码可以看出，其中`provider`是最基本的，可以调用`$provide`的`provider()`方法来定义一个供应商，然后可以通过要求`$provide`被注入到一个应用的`config`函数来获得`$provide`服务。

##定义供应商的方法
###provider
`provider`是最基本的方法，基本上除了`constant`之外都是`provider`的封装。在`provider`中必须存在一个`$get`方法，通过实现`$get`方法在应用中注入单例，使用的时候就是`$get`执行后的结果。

```
app.provider('movie', function () {
    //第一部分
    var version;
    var setVersion = function (value) {
      version = value;
    };
    //第二部分
    this.$get =  function () {
      return {
          title: 'The Matrix' + ' ' + version
      }
    }
});
```
其中，我们可以把`provider`的构成分为两部分，第一部分的函数和变量是可以在app.config函数中访问的，因此你可以在它们被其他地方访问到之前来修改它们。第二部分是$get中返回的内容，这部分内容和`factory`类似，都是任何传入了`provider`的控制器可以调用的部分。

当用`provider`创建一个service时，唯一的可以在控制器访问的属性是通过`$get`返回的内容，而用`provider`创建service的独特之处就是可以在`provider`对象传递到应用程序之前用app.config进行修改。具体用法如下：

```
app.config(function(movieProvider){
	movieProvider.setVersion('test');
})
```

###factory

`factory`是一个可注入的function。

有时`provider`的定义比较繁琐，如果只是想定义`$get`部分的话，可以直接定义`factory`。`factory`相当于只有一个`$get`的`provider`。

只需要创建一个对象，为它添加属性，然后返回这个对象就行了。具体的构建过程如下：

```
app.factory('factory', function () {
    var service = {};
    return service;
});
```
如果我们想要声明一些私有变量的时候，可以在factory中声明，并在返回的对象中建立方法来调用这些私有变量。

###service

`service`是一个可注入的构造器。它和`factory`的区别是，`factory`是不同funciton，而`service`是一个构造器，当Angular调用`service`时会用`new`进行实例化，因此需要对`this`添加属性，然后`service`返回这个`this`。

```
app.service('service', function () {
    this.title = "The Matrix";
});
```
在创建一个`service`时，需要理解的最重要一件事就是`service`会使用`new`去实例化这个对象。因此，直接添加`this`上的属性和方法。

下面可以看一个转化的例子：

```
$provide.provider('myDate', {
    $get: function() {
      return new Date();
    }
});
//可以写成
$provide.factory('myDate', function(){
    return new Date();
});
//可以写成
$provide.service('myDate', Date);
```

###constant

`constant`用来定义常量。既然是常量，定义之后就不能被改变。而且不能被装饰器`decorator`装饰。

```
var app = angular.module('app', []);
 
app.config(function ($provide) {
  $provide.constant('movieTitle', 'The Matrix');
});
 
app.controller('ctrl', function (movieTitle) {
  expect(movieTitle).toEqual('The Matrix');
});
```

###value

`value`用来定义变量，可以被修改。**但不能注入到`config`中**。可以被装饰器`decorator`装饰。

```
var app = angular.module('app', []);
 
app.config(function ($provide) {
  $provide.constant('movieTitle', 'The Matrix');
});
 
app.controller('ctrl', function (movieTitle) {
  expect(movieTitle).toEqual('The Matrix');
});
```

###decorator
`decorator`是装饰器，如果想对定义完的`$provide`进行添加逻辑，可以通过`decorator`来进行拦截并扩充。

```
var app = angular.module('app', []);
 
app.value('movieTitle', 'The Matrix');
 
app.config(function ($provide) {
  $provide.decorator('movieTitle', function ($delegate) {
    return $delegate + ' - starring Keanu Reeves';
  });
});
 
app.controller('myController', function (movieTitle) {
  expect(movieTitle).toEqual('The Matrix - starring Keanu Reeves');
});
```
`decorator`比较特殊，他不是`provider`，只是用来装饰其他`provider`的，而且它不能装饰`constant`，因为实际上`constant`不是通过`$provide`创建的。

##小结
- 所有的供应商都只被实例化一次，也就说他们都是单例的
- 除了`constant`，所有的供应商都可以被装饰器(`decorator`)装饰
- `value`就是一个简单的可注入的值
- `service`是一个可注入的构造器
- `factory`是一个可注入的方法
- `decorator`可以修改或封装其他的供应商，当然除了`constant`
- `factory `是一个不可配置的`provider`


##引用
[AngularJS中的Provider们：Service和Factory等的区别](https://segmentfault.com/a/1190000003096933)

[AngularJS 之 Factory vs Service vs Provider](http://www.oschina.net/translate/angularjs-factory-vs-service-vs-provider)

[Provider, Value, Constant, Service, Factory, Decorator](http://hellobug.github.io/blog/angularjs-providers/)

[淺談Angular.js的Provider機制](http://kirkchen.logdown.com/posts/245678-angularjs-talking-about-the-angularjs-provider-mechanisms)