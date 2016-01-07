在经历启动阶段和编译、链接阶段之后，进入运行阶段。Angular提供了双向绑定机制，在$scope变量中使用脏值检查来实现双向绑定。

本文主要借鉴`build your own AngularJS`中`Scope`部分，通过自己实现一个简单的Scope构造函数的形式来分析Angular的双向数据绑定部分。

我们想要实现一个双向数据绑定来说，可以根据[`AngularJS实战`](http://www.imooc.com/video/4279)列出的内容考虑下面几个问题：

1. 如何把一个Model绑定到多个View?（观察者模式）
2. 如何才能知道Model发生了变化？（脏值检测`$watch`和`$digest`）
3. 如果绑定的是对象，且有深层嵌套的结构，如何判断某个属性是否发生变化？（对象深比较）
4. **深层次**：如果A和B两个方法互相watch对方，如何避免发生“振荡”？（TTL机制）
5. 绑定过程中如何支持表达式？（`$parser`和`$eval`）

下面带着这些问题来构建一个基于脏值检测的双向数据绑定。

###Scope对象
首先定义一个简单的Scope构造函数。

```
function Scope(){
}
```
通过new来生成scope实例，由于是基于脏值检测的，无需设置`getter`，而是在原型方法`$watch`和`$digest`中进行检测。

```
var scope = new Scope();
scope.name = '大明';
scope.feature = 'handsome';
```

###$watch 和 $digest
- `$watch`负责监控数据的变化。使用`$watch`，可以在Scope上添加一个监听器，并在数据变化时及时执行回调。因此`$watch`指定两个参数来创建一个监听器。
	- 监控函数，用来指定关注的数据并且返回该数据改动后的值。
	- 监听函数，用于在数据变更时执行相关操作。
- `$digest`执行所有在作用域上注册过的监听器。遍历所有的监听器，并且调用他们的监听函数。

因此，可以着手两个方法的设计：

1. 首先，需要在Scope构造函数中创建一个数组`$$watchers`来存储所有的监听器。

	```
	function Scope() {
	  	this.$$watchers = [];
	}
	```
	**在这里，我们用$$的前缀表示该属性为对象的私有属性。**

2. 定义`$watch`方法时，根据前面的参数设置，传入两个函数作为参数，并将其存入`$$watchers`数组中。

	```
	Scope.prototype.$watch = function(watchFn, listenerFn) {
	  var watcher = {
	    watchFn: watchFn,
	    listenerFn: listenerFn
	  };
	  this.$$watchers.push(watcher);
	};
	```
	**我们需要在scope实例中调用这些方法，因此放在构造函数Scope的原型中。**
3. 定义`$digest`方法，需要遍历所有的监听器，并调用其监听函数。

	```
	Scope.prototype.$digest = function() {
	  _.forEach(this.$$watchers, function(watch) {
	    watch.listenerFn();
	  });  
	};
	```
	
TODO: 需要检查监控函数指定的值是否发生变更，再调用监听函数。

###脏值检测
上面说道，监控函数应该返回我们关注的数据的变化，而数据一般存储在作用域之中。因此为了使访问作用域更加便利，在监控函数中将作用域作为参数传入，然后进行属性的返回。

```
function(scope) {
  return scope.firstName;
}
```
在`$digest`中，就是调用这个监控函数，获得该属性的最新值，并与之前的返回值进行比较。如果不相同，则该监控器就是“脏”的，监听函数就该被调用。因此需要一个值来记录函数上次返回的值，因此可以把监控器多一个属性来保存上一次的值。`$digest`加入比较的功能：

```
Scope.prototype.$digest = function() {
  var self = this;
  _.forEach(this.$$watchers, function(watch) {
    var newValue = watch.watchFn(self);
    var oldValue = watch.last;
    if (newValue !== oldValue) {
      watch.listenerFn(newValue, oldValue, self);
    }
    watch.last = newValue;
  });  
};
```
此时，在`$digest`循环中处理每个监听器时，将会取出该数值的最新值并与上次返回值作对比，如果不同会执行监听函数。并用watch.last来存储新返回的值。

[查看demo](http://jsbin.com/OsITIZu/3/embed?js,console)

到这里，我们可以总结出，Angular作用域的作用就是添加监听器，并且在digest里运行他们。有以下两个特性：

- 作用域添加数据并不会有性能折扣，Angular并不会遍历作用域上的属性，只遍历所有监听器属性。
- 在`$digest`中会调用监控函数，因此需要关注监听器的数量，并提高监控函数的性能。

###在digest时获得提示
如果想在每次进行digest时获得通知，可以注册一个没有监听函数的监听器，这样在每次进行digest时都会执行该监听器，并在监控函数中返回digest提示。需要在下面两个地方进行改进：

- watch一个没有监听函数的监听器：

	```
	scope.$watch(function() {
	  console.log('digest listener fired');
	});
	```
- $watch定义中处理一下空函数的情况：

	```
	var watcher = {
	    watchFn: watchFn,
	    listenerFn: listenerFn || function() { }
	  };
	```

[查看demo](http://jsbin.com/OsITIZu/4/embed?js,console)
	
这样，监听器返回的值始终是未定义的，可以在每次digest都执行监听函数进行提示。但是这时有个问题，如果在监听函数中修改了作用域中另外一个正在被监听的属性，这时无法在同一个digest里面监测出改动。

###持续digest
当数据持续变化时，需要一直进行digest，直到监控的值停止变更。因此，可以将`$$digestOnce`定义为一次遍历watchers，返回布尔值表示是否有数据变更。`$$digestOnce`定义如下：

```
Scope.prototype.$$digestOnce = function() {
  var self  = this;
  var dirty;
  _.forEach(this.$$watchers, function(watch) {
    var newValue = watch.watchFn(self);
    var oldValue = watch.last;
    if (newValue !== oldValue) {
      watch.listenerFn(newValue, oldValue, self);
      dirty = true;
    }
    watch.last = newValue;
  });
  return dirty;
};
```
将`$digest`中嵌套多重`$$digestOnce`，直到所有监听数据不再发生变化。

```
Scope.prototype.$digest = function() {
  var dirty;
  do {
    dirty = this.$$digestOnce();
  } while (dirty);
};
```

[查看demo](http://jsbin.com/Imoyosa/3/embed?js,console)

到这里，可以知道在digest过程中，可能所有的监视器会遍历好多遍，因此，需要注意在监听器中改动其他挂载到作用域上被监视的属性。

当两个监视器相互变更了对方监控的数据时，会发生“振荡”，及一直循环不会停止。因此我们需要放弃不稳定的digest，即控制在一个可以接受的迭代数量内。如果连续发生了这么多次digest后作用域还在不断的变化，则判断它为不稳定的状态，抛出异常。

可以设置这个值为TTL(short for Time To Live)，这个值在Angular中是可调的，在digest中添加一个计数器，当达到TTL时抛出异常。

```
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  do {
    dirty = this.$$digestOnce();
    if (dirty && !(ttl--)) {
      throw "10 digest iterations reached";
    }
  } while (dirty);
};
```
这样，当两个监视器循环引用时，便可抛出异常。

[查看demo](http://jsbin.com/uNapUWe/2/embed?js,console)

在`$$digestOnce`中，我们采用严格等于`===`来比较`newValue`和`oldValue`的值，但如果比较的是两个对象，这样只能比较一个对象或数组是否变成新的了。如果对象或数组内的值发生变化时，我们需要针对监控值的值来比较，而不是针对引用。

###基于值的脏检查

在平常使用`$watch`时，第三个参数为`deepWatch`。当设置为true时，则去检查被监控对象的每个属性是否发生了变化，即值的检测。因此我们需要重新定义`$watch`，存储这个值。

```
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn,
    valueEq: !!valueEq
  };
  this.$$watchers.push(watcher);
};
```
当valueEq不存在时，通过两次取反，可以获得布尔值作为valueEq。基于值要求我们必须遍历被监控值中的所有内容，若存在嵌套对象或数组，还要递归地对比。

在这里关于值比较，我们定义一个新函数`$$areEqual`来进行对比，`_.isEqual`为`Lo-Dash`提供的值相等检测函数。

```
Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
  if (valueEq) {
    return _.isEqual(newValue, oldValue);
  } else {
    return newValue === oldValue;
  }
};
```
然后修改`$digestOnce`中相等检测部分，同时当valueEq为true时深拷贝最新的值并赋值给last暂存。

```
Scope.prototype.$$digestOnce = function() {
  var self  = this;
  var dirty;
  _.forEach(this.$$watchers, function(watch) {
    var newValue = watch.watchFn(self);
    var oldValue = watch.last;
    if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
      watch.listenerFn(newValue, oldValue, self);
      dirty = true;
    }
    watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
  });
  return dirty;
};
```
相比于检查引用，检查值需要进行的操作更复杂，当有嵌套结构时，递归和深拷贝也比较消耗资源。

**Angular默认是不使用基于值的检查的，需要用户手动设置标记。**

引用检测和值检测： [查看demo](http://jsbin.com/ARiWENO/3/embed?js,console)

###非数字的情况（NaN）
由于我们的引用检测使用全等的方式，但如果我们传入的数据为`NaN`时，则会一直检测不相等。因此需要对非数字的情况进行特殊处理，在`$$areEqual`的修改如下：

```
Scope.prototype.$$areEqual = function(newValue, oldValue, valueEq) {
  if (valueEq) {
    return _.isEqual(newValue, oldValue);
  } else {
    return newValue === oldValue ||
      (typeof newValue === 'number' && typeof oldValue === 'number' &&
       isNaN(newValue) && isNaN(oldValue));
  }
};
```
[查看demo](http://jsbin.com/ijINaRA/2/embed?js,console)

###$eval 在作用域上执行代码
`$eval`和`$parse`都是用来解析表达式的，不过`$parse`是作为单独一个服务存在，而`$eval`是作为scope的一个方法来使用的。它接收一个函数作为参数，然后立即执行这个函数，并将作用域作为参数传递给它。

在`$eval`中，可以保证运行的函数在当前作用域下执行，同时，在下面的`$apply`也调用`$eval`来执行函数。

当传入表达式时，它会将表达式编译，然后在作用域的上下文中执行。`$eval`的实现：

```
Scope.prototype.$eval = function(expr, locals) {
  return expr(this, locals);
};
```

###$apply

我们平常使用时，常常会使用到`$apply`这个方法，它可以将外部库集成到Angular中。在`$apply`中，调用`$eval`这个函数，并且触发digest循环。

为了防止执行`$eval`时抛出异常，将`$eval`执行部分放在`try..catch`中，并将`$digest`放入`finally`中确保一定可以执行。

在执行外部库或者自己扩展的代码时，会改变作用域上绑定的值。此时`$apply`可以保证作用域可以检测到这些值的变更，相当于将外部代码利用`$apply`集成到Angular的生命周期中。

```
Scope.prototype.$apply = function(expr) {
  try {
    return this.$eval(expr);
  } finally {
    this.$digest();
  }
};
```

[查看demo](http://jsbin.com/UzaWUC/2/embed?js,console)

###$evalAsync

若有需求将某一段代码延迟执行，在Angular中有两种方式：

- `$timeout`：用`$apply`封装后的`setTimeOut`方法，将延迟执行的函数集成到digest生命周期里。
- `$evalAsync`：集成在Scope上的方法，接受一个函数，将其在正持续的digest中或者下一次digest之前执行。通常把需要延迟执行的函数放到监听函数中。

要实现这样的效果，首先要在Scope中初始化一个`$$asyncQueue`来存储`$evalAsync`列入计划的任务。

```
function Scope() {
  this.$$watchers = [];
  this.$$asyncQueue = [];
}
```
然后定义函数`$evalAsync`，将当前的作用域以及需要执行的函数存入`$$asyncQueue`中。

```
Scope.prototype.$evalAsync = function(expr) {
  this.$$asyncQueue.push({scope: this, expression: expr});
};
```
之后，在`$digest`中，从队列中取出每一个延迟执行的方法，调用`$eval`来执行。

```
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  do {
    while (this.$$asyncQueue.length) {
      var asyncTask = this.$$asyncQueue.shift();
      this.$eval(asyncTask.expression);
    }
    dirty = this.$$digestOnce();
    if (dirty && !(ttl--)) {
      throw "10 digest iterations reached";
    }
  } while (dirty);
};
```

[查看demo](http://jsbin.com/ilepOwI/1/embed?js,console)

###作用域阶段
如果当前没有正在执行的$digest时，需要触发一个digest来执行这部分延迟代码，需要保证当调用`$evalAsync`时，需要延迟执行的函数能尽快地被执行。

因此，我们需要一个状态来判断当前是否有digest正在运行，如果有，不想影响到正在被执行的digest，如果没有，则延迟触发一个digest。所以我们在Scope中用`$$phase`来保存当前的阶段，存储正在运行的信息。

```
function Scope() {
  this.$$watchers = [];
  this.$$asyncQueue = [];
  this.$$phase = null;
}
```
对于阶段的处理，建立两个方法来控制`$$parse`。一个用于设置，一个用于清除，在设置中额外判断一下当前是否有digest正在执行。

```
Scope.prototype.$beginPhase = function(phase) {
  if (this.$$phase) {
    throw this.$$phase + ' already in progress.';
  }
  this.$$phase = phase;
};

Scope.prototype.$clearPhase = function() {
  this.$$phase = null;
};
```
在`$digest`和`$apply`方法的外层，通过不同的字面量来自设置阶段属性。

```
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  this.$beginPhase("$digest");
  do {
    while (this.$$asyncQueue.length) {
      var asyncTask = this.$$asyncQueue.shift();
      this.$eval(asyncTask.expression);
    }
    dirty = this.$$digestOnce();
    if (dirty && !(ttl--)) {
      this.$clearPhase();
      throw "10 digest iterations reached";
    }
  } while (dirty);
  this.$clearPhase();
};
Scope.prototype.$apply = function(expr) {
  try {
    this.$beginPhase("$apply");
    return this.$eval(expr);
  } finally {
    this.$clearPhase();
    this.$digest();
  }
};
```
这样完成了对当前状态的保存和判断，下面在`$evalAsync`中，加入对`$$phase`的判断，如果没有，则主动触发一次digest。来保证当调用`$evalAsync`时总有一个digest会发生。

```
Scope.prototype.$evalAsync = function(expr) {
  var self = this;
  if (!self.$$phase && !self.$$asyncQueue.length) {
    setTimeout(function() {
      if (self.$$asyncQueue.length) {
        self.$digest();
      }
    }, 0);
  }
  self.$$asyncQueue.push({scope: self, expression: expr});
};
```

[查看demo](http://jsbin.com/ilepOwI/1/embed?js,console)

###$$postDigest在digest之后执行代码
在Scope上，有属性`$$postDigest`来存储在digest之后执行的代码块，然后在digest之后执行。如果在`$$postDigest`中修改了作用域，需要手动的调用`$digest`或`$apply`使改动生效。

首先在Scope中添加`$$postDigestQueue`数组。

```
function Scope() {
  this.$$watchers = [];
  this.$$asyncQueue = [];
  this.$$postDigestQueue = [];
  this.$$phase = null;
}
```
然后定义`$$postDigest`方法，就是将执行的方法加入队列中。

```
Scope.prototype.$$postDigest = function(fn) {
  this.$$postDigestQueue.push(fn);
};
```

最后在digest循环中，最后执行`$$postDigestQueue`中的所有方法。

```
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  this.$beginPhase("$digest");
  do {
    while (this.$$asyncQueue.length) {
      var asyncTask = this.$$asyncQueue.shift();
      this.$eval(asyncTask.expression);
    }
    dirty = this.$$digestOnce();
    if (dirty && !(ttl--)) {
      this.$clearPhase();
      throw "10 digest iterations reached";
    }
  } while (dirty);
  this.$clearPhase();

  while (this.$$postDigestQueue.length) {
    this.$$postDigestQueue.shift()();
  }
};
```

[查看Demo](http://jsbin.com/IMEhowO/1/embed?js,console)

###异常处理

我们需要完善Scope的异常处理机制，在Angular中，当遇到异常的时候，不管在`$evalAsync`、`$$digestOnce`和`$$postDigest`中遇到异常都不会中止正在执行的digest。

因此我们需要在以上三个地方加入`try...catch`。当遇到错误时，将异常抛给日志，并继续执行。

`$$digestOnce`:

```
Scope.prototype.$$digestOnce = function() {
  var self  = this;
  var dirty;
  _.forEach(this.$$watchers, function(watch) {
    try {
      var newValue = watch.watchFn(self);
      var oldValue = watch.last;
      if (!self.$$areEqual(newValue, oldValue, watch.valueEq)) {
        watch.listenerFn(newValue, oldValue, self);
        dirty = true;
      }
      watch.last = (watch.valueEq ? _.cloneDeep(newValue) : newValue);
    } catch (e) {
      (console.error || console.log)(e);
    }
  });
  return dirty;
};
```
`$evalAsync`和`$$postDigest`:

```
Scope.prototype.$digest = function() {
  var ttl = 10;
  var dirty;
  this.$beginPhase("$digest");
  do {
    while (this.$$asyncQueue.length) {
      try {
        var asyncTask = this.$$asyncQueue.shift();
        this.$eval(asyncTask.expression);
      } catch (e) {
        (console.error || console.log)(e);
      }
    }
    dirty = this.$$digestOnce();
    if (dirty && !(ttl--)) {
      this.$clearPhase();
      throw "10 digest iterations reached";
    }
  } while (dirty);
  this.$clearPhase();

  while (this.$$postDigestQueue.length) {
    try {
      this.$$postDigestQueue.shift()();
    } catch (e) {
      (console.error || console.log)(e);
    }
  }
};
```
[查看Demo](http://jsbin.com/IMEhowO/2/embed?js,console)

###销毁监听器

当一个监听器创建之后，会一直存在作用域的生命周期中，但有时我们有需求需要销毁某个监听器。

因为监听器是存储在`$$scope.watchers`中，若想移除，直接把这个监控器从`$$watchers`数组中去除就可以了。我们把销毁监听器的方法写在`$watch`的回调中，获取该监听器的index，并在数组中移除。

```
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  var self = this;
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn,
    valueEq: !!valueEq
  };
  self.$$watchers.push(watcher);
  return function() {
    var index = self.$$watchers.indexOf(watcher);
    if (index >= 0) {
      self.$$watchers.splice(index, 1);
    }
  };
};
```
至此，我们可以用变量暂存`$watch`的返回值，即该监听器的销毁函数。

[查看Demo](http://jsbin.com/IMEhowO/4/embed?js,console)

###小结

至此，我们基本实现了Angular中的Scope构造函数，并且对脏值检测和双向绑定有了一个基本的认识。下面在内置指令的源码解析中，结合Scope的部分深入分析Angular中的双向数据绑定部分。

