##启动流程

Angular版本为1.3.0，本文主要分析Angular的基本启动流程。首先根据流程图分析整体的启动过程，然后详细分析各个模块。启动流程如下：

###angular启动流程

```
(function(window, document, undefined) {'use strict';

  // 如extend()等一些通用方法

  // 判断代码angularjs重复加载
  if (window.angular.bootstrap) {
       console.log('WARNING: Tried to load angular more than once.');
    return;
  }

  // 绑定jQuery或者jqLite，实现angular.element  
  bindJQuery();

  // 暴露api，挂载一些通用方法，如：angular.forEach
  // 实现angular.module，并定义模块ng，以及ngLocale
  publishExternalAPI(angular);

  // 当dom ready时，开始执行程序的初始化
  jqLite(document).ready(function() {
    // 初始化入口
    angularInit(document, bootstrap);
  });

})(window, document);
```
流程图如下:

![angular启动流程图](Angular启动流程.png =348x720)

其中初始化有三个函数，下面针对`bindJQuery`、`publishExternalAPI`和`angularInit`三个部分详细分析。

####bindJQuery
这个方法的主要作用是检测当前是否引用`JQuery`，没有则引入Angular自带的`jQLite`模块，否则使用引用的`jQuery`。

```
// 是否已经绑定过jQuery
var bindJQueryFired = false;

function bindJQuery() {  
  // 已经绑定过，不再绑定
  if (bindJQueryFired) {
    return;
  }
  //如果在主页面加载了jQuery库，这里的jQuery就会存在
  jQuery = window.jQuery;

  if (jQuery && jQuery.fn.on) {
     // 如果引入了jQuery，就使用jQuery
     jqLite = jQuery;
　　 // ......
  } else {
     //如果没有引用jQuery库，就使用自带的JQLite。
     jqLite = JQLite;
  }

  // 把结果赋给angular.element上
  // 所以使用angular.element()就和使用jQuery一样
  angular.element = jqLite;

  // 已经绑定过，启动的时候再次调用就不会再绑定了
  bindJQueryFired = true;
}
```

####publishExternalAPI

1. 将全局变量angular对象挂载一些通用方法。
2. 注`ngLocale`和`ng`两个模块（`ng`依赖于`ngLocale`）。
3. `ng`模块用来注册所有内置的`service`和`directive`(在angularInit之后执行)。

```
function publishExternalAPI(angular){
  // 将通用方法挂载到全局变量augular上
  extend(angular, {
    'bootstrap': bootstrap,
    'copy': copy,
    // ...省略若干通用方法
  });

  // 实现angular.module方法并赋值给angularModule，下面具体分析
  angularModule = setupModuleLoader(window);

  //获取ngLocale模块，如果没有，则注册模块
  try {
    angularModule('ngLocale');
  } catch (e) {
    angularModule('ngLocale', []).provider('$locale', $LocaleProvider);
  }

  // 注册ng模块（依赖于ngLocale模块），回调中注册内置的service和directive，等angularInit结束后执行
  angularModule('ng', ['ngLocale'], ['$provide',
    function ngModule($provide) {
      // $$sanitizeUriProvider需要在$compile之前引入，因为会被$compile调用.
      $provide.provider({
        $$sanitizeUri: $$SanitizeUriProvider
      });
      $provide.provider('$compile', $CompileProvider).
        directive({
            a: htmlAnchorDirective,
            input: inputDirective,
            // ...省略若干directive，在ng模块添加各种指令
        }).
        directive({
          ngInclude: ngIncludeFillContentDirective
        }).
        directive(ngAttributeAliasDirectives).
        directive(ngEventDirectives);
      $provide.provider({
        $anchorScroll: $AnchorScrollProvider,
        $animate: $AnimateProvider,
        $browser: $BrowserProvider,
        // ...省略若干service，在ng模块添加各种服务
      });
    }
  ]);
}
```

####setupModuleLoader

在`publishExternalAPI`中，用到了该方法：`angularModule = setupModuleLoader(window)`:这是模块加载器，在angular上添加了module方法，注册模块并返回模块实例（如上边的`ng`和`ngLocale`模块）。

```
function setupModuleLoader(window) {

  // 异常处理部分
  var $injectorMinErr = minErr('$injector');
  var ngMinErr = minErr('ng');

  //如果存在，执行getter，不存在，将obj[name]执行setter
  function ensure(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  }

  // 注册window.angular为空对象并获得angular全局对象
  var angular = ensure(window, 'angular', Object);

  angular.$$minErr = angular.$$minErr || minErr;

  // 定义angular.module方法并返回module这个函数
  return ensure(angular, 'module', function() {

   	// 定义一个模块实例，通过闭包进行管理模块
    var modules = {};

    // angular.module 的方法实现。和ensure类似，如果参数是一个，获取指定name的模块（getter操作），否则，（重新）创建模块实例并存储（setter操作），最后返回
    //这里使用了闭包，只恩呢该通过angular.module访问私有变量modules
    return function module(name, requires, configFn) {

      // 检测模块名不能是'hasOwnProperty'
      var assertNotHasOwnProperty = function(name, context) {
        if (name === 'hasOwnProperty') {
          throw ngMinErr('badname', 'hasOwnProperty is not a valid {0} name', context);
        }
      };

      // 检测模块名
      assertNotHasOwnProperty(name, 'module');

      // 如果参数存在且该模块已存在，则将该模块设为null，重新创建
      if (requires && modules.hasOwnProperty(name)) {
        modules[name] = null;
      }

      // 获取指定name模块的模块,此处获取或创建新的模块实例
      return ensure(modules, name, function() {

        // 程序走到这里，表示是新模块的创建过程
        // 而requires如果为空，则表示是获取已有的模块
        // 两者其实是相互矛盾的，所以抛出异常，说明该模块还没有注册
        // 所以我们在创建模块时，就算没有依赖其他模块，写法也应该是：
        // angular.module('myModule', [])进行定义模块;
        if (!requires) {
          throw $injectorMinErr('nomod', "Module '{0}' is not available! You either misspelled " +
             "the module name or forgot to load it. If registering a module ensure that you " +
             "specify the dependencies as the second argument.", name);
        }

        var invokeQueue = [];  //调用队列

        var configBlocks = [];   //配置快

        var runBlocks = [];    //运行块

		//invokeLater返回一个闭包，当调用config,provider时，会调用这个闭包，下面详细解释
        var config = invokeLater('$injector', 'invoke', 'push', configBlocks);

        // 将要返回的module实例，通过invokeLater建立了各种常用接口
        var moduleInstance = {

          _invokeQueue: invokeQueue,
          _configBlocks: configBlocks,
          _runBlocks: runBlocks,

          requires: requires,

          name: name,

          provider: invokeLater('$provide', 'provider'),

          factory: invokeLater('$provide', 'factory'),

          service: invokeLater('$provide', 'service'),

          value: invokeLater('$provide', 'value'),

          constant: invokeLater('$provide', 'constant', 'unshift'),

          animation: invokeLater('$animateProvider', 'register'),

          filter: invokeLater('$filterProvider', 'register'),

          controller: invokeLater('$controllerProvider', 'register'),

          directive: invokeLater('$compileProvider', 'directive'),

          config: config,

          run: function(block) {
            runBlocks.push(block);
            return this;
          }
        };

		//如果传入三个参数后，第三个参数就会执行config方法，如上面定义ng模块时会传入
        if (configFn) {
          config(configFn);
        }

        return  moduleInstance;

        // 在调用module.provider时只是将数据储存在queue中，放在moduleInstance实例里。
        // 真正的执行是在angularInit之后loadModules的时候执行。
        function invokeLater(provider, method, insertMethod, queue) {

          // 默认队列是invokeQueue，也可以是configBlocks
          // 默认队列操作是push，也可以是unshift
          if (!queue) queue = invokeQueue;
          return function() {
            queue[insertMethod || 'push']([provider, method, arguments]);
            return moduleInstance;
          };
        }
      });
    };
  });

}
```

在这里`invokeLater`函数会返回一个闭包，当调用一些如`config`，`provider`方法时，就会调用这个闭包，将相应的方法加入`invokeQueue`数组这个队列中。 下面是调用`provider`之后产生的队列：

```
app.provider('myprovider',['$window',function($window){  
  //code
}]);

// 实际是('$provide','provider',['$window',function($window){}]) push进invokeQueue数组中
```

此时只是暂存，并未执行，在后面执行过程中，实际上是由第一个参数`$provide`调用第二个参数`provider`的方法名，把第三个参数`['$window',function($window){}]`当参数传入，即

```
args[0][args[1]].apply(args[0],args[2]);  
```
在run和constant有两个特例：

- `run`方法只是将需要执行的block放入执行区块`runBlock`中，并没有放入`invokeQueue`中，在加载模块时，会将`runBlock`中所有执行区块调用`invoke`进行执行。
- 入队时一般执行`push`操作，`constant`执行`unshift`操作，因为常量需要被其他区块所调用，因此需要在前面进行注入。

####angularInit函数

当dom ready之后，调用angularInit进行初始化部分。

1. 首先遍历names数组，通过document.getElementById(name)或者querySelectorAll(name)检索到element存到elements数组中，然后获取到appElement和module。
2. 将appElement和module传给bootstrap函数启动整个应用。

```
  // ngAttrPrefixes = ['ng-', 'data-ng-', 'ng:', 'x-ng-'];
  // 遍历多种前缀，找到含有"ng-app"的节点作为element,module为该app的名称
  // 先对element（即document）进行判断，再从element中的子孙元素中查找
   
function angularInit(element, bootstrap) {
  var appElement,
      module,
      config = {};

  // 先看指定的element是否包含ng-app属性（4种）
  // 比后面的通过属性来查找的优先级高
  forEach(ngAttrPrefixes, function(prefix) {
    var name = prefix + 'app';

    if (!appElement && element.hasAttribute && element.hasAttribute(name)) {
      appElement = element;
      module = element.getAttribute(name);
    }
  });
  
  // 查找包含ng-app属性（4种）的节点
  // 通过该属性获取启动模块
  forEach(ngAttrPrefixes, function(prefix) {
    var name = prefix + 'app';
    var candidate;

    if (!appElement && (candidate = element.querySelector('[' + name.replace(':', '\\:') + ']'))) {
      appElement = candidate;
      module = candidate.getAttribute(name);
    }
  });

  // 如果应用节点存在，那么启动整个应用（即bootstrap）
  // 如果appElement含有xx-strict-di属性，那么设置严格依赖注入参数
  if (appElement) {
    config.strictDi = getNgAttribute(appElement, "strict-di") !== null;
    bootstrap(appElement, module ? [module] : [], config);
  }
}
```

####bootstrap函数

- 在程序执行过程中，除了可以指定`ng-app`的方式指定入口，可以以直接调用angular.bootstrap()初始化应用
- 在bootstrap过程中，首先获取当前element节点，根据当前的modules数组（modules = [‘myApp’]）,将$provide数组加入['$provide', function($provide){}],添加ng模块加入，调用createInjector创建注射器实例。将实例中调用5个函数，进行编译操作。

```
function bootstrap(element, modules, config) {
  // 预处理config
  var doBootstrap = function() {
    element = jqLite(element);

    // 首先判断该dom是否已经被注入（即这个dom已经被bootstrap过）
    // 注意这里的injector方法是Angular为jqLite中提供的，区别于一般的jQuery api
    if (element.injector()) {
      var tag = (element[0] === document) ? 'document' : startingTag(element);
      //Encode angle brackets to prevent input from being sanitized to empty string #8683
      throw ngMinErr(
          'btstrpd',
          "App Already Bootstrapped with this Element '{0}'",
          tag.replace(/</,'<').replace(/>/,'>'));
    }
	
	//此时的modules应为 modules = ['myApp'];
    modules = modules || [];

    // 添加"$provide"模块
    modules.unshift(['$provide', function($provide) {
      $provide.value('$rootElement', element);
    }]);
	
	//1.3.0新增，调用或启用调试运行时的信息
    if (config.debugInfoEnabled) {
      // Pushing so that this overrides `debugInfoEnabled` setting defined in user's `modules`.
      modules.push(['$compileProvider', function($compileProvider) {
        $compileProvider.debugInfoEnabled(true);
      }]);
    }

    // 添加ng模块 ,到这里modules可能是: ['ng', [$provide, function($provide){...}], 'xx']
    modules.unshift('ng');

    // 把angular应用需要初始化的模块传入，创建injector对象，注册所有的内置模块
    var injector = createInjector(modules, config.strictDi);

    // 调用注册器实例对象的invoke方法，实际就是执行bootstrapApply函数
  	// bootstrapApply函数从$rootElement开始，绑定$rootScope，递归编译
    injector.invoke(['$rootScope', '$rootElement', '$compile', '$injector',
       function bootstrapApply(scope, element, compile, injector) {
        scope.$apply(function() {
          // 标记该dom已经被注入
          element.data('$injector', injector);
          // 编译整个dom
          compile(element)(scope);
        });
      }]
    );
    return injector;
  };

  // ...省略若干代码

  if (window && !NG_DEFER_BOOTSTRAP.test(window.name)) {
    return doBootstrap();
  }
  // ...省略若干代码
}
```

之前的调用`module`、`controller`、`service`等方法都是注册，将其缓存到队列中，只有执行`invoke`方法后才是真正执行该方法并声称实例。

####createInjector函数
- 实现依赖注入，创建injector实例。
- createInjector主要是生成四个属性：
 	
	1. providerCache:用来存储所有provider的缓存。初始化一个的对象，`providerCache= {$provide:{}}`，之后加入了`$injector`属性。
	2. providerInjector:内部的injector实例。调用createInternalInjector方法返回一个`$injector`对象，操作providerCache和抛出错误的回调函数。
	3. instanceCache:所有provider返回实例缓存。有一个`$injector`属性和instanceInjector相同。
	4. instanceInjector：外部可访问的injector实例，实现实例层级的依赖注入。

- 最后，返回这个`instanceInjector`作为最终injector的实例。

源码如下：

```
function createInjector(modulesToLoad, strictDi) {
  strictDi = (strictDi === true);
  var INSTANTIATING = {},
      providerSuffix = 'Provider',
      path = [],
      loadedModules = new HashMap([], true),
      providerCache = {
        $provide: {
            provider: supportObject(provider),
            factory: supportObject(factory),
            service: supportObject(service),
            value: supportObject(value),
            constant: supportObject(constant),
            decorator: decorator
          }
      },
      providerInjector = (providerCache.$injector =
          createInternalInjector(providerCache, function() {
            throw $injectorMinErr('unpr', "Unknown provider: {0}", path.join(' <- '));
          })),
      instanceCache = {},
      instanceInjector = (instanceCache.$injector =
          createInternalInjector(instanceCache, function(servicename) {
            var provider = providerInjector.get(servicename + providerSuffix);
            return instanceInjector.invoke(provider.$get, provider, undefined, servicename);
          }));

  // 循环加载模块，实质上是：
  // 1. 注册每个模块上挂载的service（也就是_invokeQueue）
  // 2. 执行每个模块的自身的回调（也就是_configBlocks）
  // 3. 通过依赖注入，执行所有模块的_runBlocks
  forEach(loadModules(modulesToLoad), function(fn) { instanceInjector.invoke(fn || noop); });

  return instanceInjector;

  // ...省略若干函数定义
 }
```

其中，`providerCache`只是一个对象，然后调用`createInternalInjector`方法返回添加若干重量级的API(`annotate`、`invoke`等等)，赋值给`providerInjector`，生成的实例如下：

```
providerCache.$provide = {  
  provider: supportObject(provider),
  factory: supportObject(factory),
  service: supportObject(service),
  value: supportObject(value),
  constant: supportObject(constant),
  decorator: decorator
}

// 返回的也是这个
providerCache.$injector = providerInjector  
= instanceCache.$injector = instanceInjector = {
  get:getService,
  annotate:annotate,
  instantiate:instantiate,
  invoke:invoke,
  has:has
}
```

至此，`module`和`injector`完成了Angular中依赖注入的部分，下面介绍两个重要的方法`loadModules`和`createInternalInjector`。


####createInternalInjector函数

- createInternalInjector是一个工厂，返回五个方法，创建一个`injector`实例。

```
function createInternalInjector (){
    
    function getService(serviceName) {};

    function invoke(fn, self, locals){};

    function instantiate(Type, locals) {};

    return {
      // 通过service对应的工厂函数还有本地依赖初始化service
      invoke: invoke,
      // 调用invoke实例化service
      instantiate: instantiate,
      // 通过名称和本地依赖，返回对应的service
      get: getService,
      // 返回一个数组，解析依赖的service的名称
      annotate: annotate,
      // 依赖的service是否存在
      has: function(name) {
        //
      }
    };
}
```
- annotate用来获取依赖注入列表
- 有两种传参方式，第一种annotate(fn(injectName))，使$inject = [injectName];第二种annotate([injectName,function(){}])，则取出所有的$inject数组。

```
function annotate(fn) {
  var $inject,
      fnText,
      argDecl,
      last;
  //第一种情况
  if (typeof fn == 'function') {
    if (!($inject = fn.$inject)) {
      $inject = [];
      if (fn.length) {
        // 去除注释
        fnText = fn.toString().replace(STRIP_COMMENTS, '');
        // 获取参数列表
        argDecl = fnText.match(FN_ARGS);
        // 逗号分隔获取所有依赖
        forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg){
          arg.replace(FN_ARG, function(all, underscore, name){
            $inject.push(name);
          });
        });
      }
      fn.$inject = $inject;
    }
  } else if (isArray(fn)) {
  //第二种情况
    last = fn.length - 1;
    assertArgFn(fn[last], 'fn');
    $inject = fn.slice(0, last);
  } else {
    assertArgFn(fn, 'fn', true);
  }
  return $inject;
}
```

- invoke：通过annotate取出依赖注入，将依赖注入为参数调用函数体。

```
function invoke(fn, self, locals){
      var args = [],
          $inject = annotate(fn),
          length, i,
          key;

      for(i = 0, length = $inject.length; i < length; i++) {
        key = $inject[i];
        if (typeof key !== 'string') {
          throw $injectorMinErr('itkn',
                  'Incorrect injection token! Expected service name as string, got {0}', key);
        }
        // 如果是本地变量从locals取出即可
   		// 如果是依赖的service，获取之
        args.push(
          locals && locals.hasOwnProperty(key)
          ? locals[key]
          : getService(key)
        );
      return fn.apply(self, args);
}
```

获取到依赖之后就是处理要执行函数的参数，如果是本地变量从 locals 中获取即可，否则调用 getService 获得实例，然后将参数注入函数执行即可。


```
    function getService(serviceName) {
      if (cache.hasOwnProperty(serviceName)) {
        if (cache[serviceName] === INSTANTIATING) {
          throw $injectorMinErr('cdep', 'Circular dependency found: {0}',
                    serviceName + ' <- ' + path.join(' <- '));
        }
        return cache[serviceName];
      } else {
        try {
          path.unshift(serviceName);
          cache[serviceName] = INSTANTIATING;
          return cache[serviceName] = factory(serviceName);
        } catch (err) {
          if (cache[serviceName] === INSTANTIATING) {
            delete cache[serviceName];
          }
          throw err;
        } finally {
          path.shift();
        }
      }
    }
```

getService 先从 cache 中获取 service 实例，如果缓存中没有就执行 service 的工厂方法生成实例，立即缓存该实例，然后返回该实例，下次有模块需要该 service 实例时直接从 cache 中获取即可。

有了注入器实例，再回到 bootstrap 函数，下一步就可以从 rootElement（ng-app元素所在节点）开始，注入 rootScope，编译整个文档了。

####loadModules函数

- loadModules获得每个module的runBlocks部分。
	1. 遍历modulesToLoad数组，如果某一项为字符串时，加载该模块，返回moduleInstance实例moduleFn.
	2. 加载该实例的moduleFn.requires部分，并将该实例的runBlocks部分一次添加到runBlocks中。
	3. 将每一个实例的_invokeQueue存储的controller和service拿出来遍历执行，第一个参数调用名为第二个参数的方法，传入第三个参数。
	4. 如果某一项是数组时，用invoke方法执行后将结果存入runBlocks中。
	5. 返回runBlocks。
	
```
function loadModules(modulesToLoad){
    var runBlocks = [], moduleFn;

    // 循环加载每个module,
    // 1. 注册每个模块上挂载的service（也就是_invokeQueue）
    // 2. 执行每个模块的自身的回调（也就是_configBlocks）
    // 3. 通过递归搜集所有（依赖）模块的_runBlocks，并返回
    forEach(modulesToLoad, function(module) {

      // 判断模块是否已经加载过
      if (loadedModules.get(module)) return;

      // 设置模块已经加载过
      loadedModules.put(module, true);

      function runInvokeQueue(queue) {
        var i, ii;
        for(i = 0, ii = queue.length; i < ii; i++) {

          var invokeArgs = queue[i],
              provider = providerInjector.get(invokeArgs[0]);

          // 通过providerInjector获取指定服务（类），传递参数并执行指定方法
          provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
        }
      }

      // 模块可以是以下三种情况：
      // 1. 字符串表示模块名（注册过的模块），如：'ng'模块
      // 2. 普通函数（也可以是隐式声明依赖的函数），如：function($provider) {...}
      // 3. 数组（即声明依赖的函数）如：[$provide, function($provide){...}
      try {
        if (isString(module)) {
          // 获取通过模块名获取模块对象
          moduleFn = angularModule(module);
          // 通过递归加载所有依赖模块，并且获取所有依赖模块（包括自身）的_runBlocks
          runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);
          // 遍历_invokeQueue数组依次执行$provider服务的指定方法（如：factory，value等）
          runInvokeQueue(moduleFn._invokeQueue);
          // 遍历_configBlocks数组依次执行$injector服务的invoke方法（即依赖注入并执行回调）
          runInvokeQueue(moduleFn._configBlocks);

        // 如果module是函数或者数组（可认为是匿名模块），那么依赖注入后直接执行
        // 并将返回值保存到runBlocks（可能是函数，又将继续执行）
        } else if (isFunction(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else if (isArray(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else {
          assertArgFn(module, 'module');
        }
      } catch (e) {
        if (isArray(module)) {
          module = module[module.length - 1];
        }
        if (e.message && e.stack && e.stack.indexOf(e.message) == -1) {
          e = e.message + '\n' + e.stack;
        }
        throw $injectorMinErr('modulerr', "Failed to instantiate module {0} due to:\n{1}",
                  module, e.stack || e.message || e);
      }
    });
    return runBlocks;
  }
```
到这里在注册ng模块时的回调，在runInvokeQueue(moduleFn._configBlocks);已经执行过了，也就意味着许许多多的内置模块已经存入providerCache中了，所以在后面的依赖注入中我们可以随意调用。

至此，完成了angular的启动过程。











