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

当传入表达式时，它会