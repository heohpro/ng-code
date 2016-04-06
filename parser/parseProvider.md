#RootScope&$parse
本篇着重介绍`rootscope`中的各个接口以及文法解析器`$parse`。

##RootScope
今天这个`rootscope`可是angularjs里面比较活跃的一个provider,大家可以理解为一个模型M或者VM,它主要负责与控制器或者指令进行数据交互。

###从$get属性说起
说起这个`$get`属性,是每个系统provider都有的,主要是先保存要实例化的函数体，等待`instanceinjector.invoke`的时候来调用，因为`$get`的代码比较多，所以先上要讲的那部分,大家可以注意到了，在`$get`上面有一个`digestTtl`方法

```
this.digestTtl = function(value) {
    if (arguments.length) {
      TTL = value;
    }
    return TTL;
  };
```
这个是用来修改系统默认的dirty check次数的，默认是10次,通过在config里引用rootscopeprovider,可以调用这个方法传递不同的值来修改ttl(short for Time To Live)

下面来看下`$get`中的scope构造函数,和`自定义$scope`基本一致。

```
function Scope() {
    this.$id = nextUid();
    this.$$phase = this.$parent = this.$$watchers =
                     this.$$nextSibling = this.$$prevSibling =
                     this.$$childHead = this.$$childTail = null;
    this['this'] = this.$root =  this;
    this.$$destroyed = false;
    this.$$asyncQueue = [];
    this.$$postDigestQueue = [];
    this.$$listeners = {};
    this.$$listenerCount = {};
    this.$$isolateBindings = {};
}
```
可以看到在构造函数里定义了很多属性,我们来一一说明一下

- $id, 通过nextUid方法来生成一个唯一的标识
- $$phase, 这是一个状态标识,一般在dirty check时用到，表明现在在哪个阶段
- $parent, 代表自己的上级scope属性
- $$watchers, 保存scope变量当前所有的监控数据,是一个数组
- $$nextSibling, 下一个兄弟scope属性
- $$prevSibling, 前一个兄弟scope属性
- $$childHead, 第一个子级scope属性
- $$childTail, 最后一个子级scope属性
- $$destroyed, 表示是否被销毁
- $$asyncQueue, 代表异步操作的数组
- $$postDigestQueue, 代表一个在dirty check之后执行的数组
- $$listeners, 代表scope变量当前所有的监听数据,是一个数组
- $$listenerCount, 暂无
- $$isolateBindings, 暂无

通过这段代码，可以看出,系统默认会创建根作用域，并作为`$rootScopeprovider`实例返回.

```
var $rootScope = new Scope();

return $rootScope;
```
创建子级作用域是通过$new方法,我们来看看.

```
$new: function(isolate) {
        var ChildScope,
            child;

        if (isolate) {
          child = new Scope();
          child.$root = this.$root;
          // ensure that there is just one async queue per $rootScope and its children
          child.$$asyncQueue = this.$$asyncQueue;
          child.$$postDigestQueue = this.$$postDigestQueue;
        } else {
          // Only create a child scope class if somebody asks for one,
          // but cache it to allow the VM to optimize lookups.
          if (!this.$$childScopeClass) {
            this.$$childScopeClass = function() {
              this.$$watchers = this.$$nextSibling =
                  this.$$childHead = this.$$childTail = null;
              this.$$listeners = {};
              this.$$listenerCount = {};
              this.$id = nextUid();
              this.$$childScopeClass = null;
            };
            this.$$childScopeClass.prototype = this;
          }
          child = new this.$$childScopeClass();
        }
        child['this'] = child;
        child.$parent = this;
        child.$$prevSibling = this.$$childTail;
        if (this.$$childHead) {
          this.$$childTail.$$nextSibling = child;
          this.$$childTail = child;
        } else {
          this.$$childHead = this.$$childTail = child;
        }
        return child;
      }
```
通过分析上面的代码，可以得出

- `isolate`标识来创建独立作用域,这个在创建指令，并且scope属性定义的情况下，会触发这种情况，还有几种别的特殊情况,假如是独立作用域的话，会多一个$root属性,这个默认是指向rootscope的

- 如果不是独立的作用域,则会生成一个内部的构造函数,把此构造函数的prototype指向当前scope实例

- 通用的操作就是,设置当前作用域的`$$childTail`,`$$childTail.$$nextSibling`,`$$childHead,this.$$childTail`为生成的子级作用域;设置子级域的`$parent`为当前作用域,`$$prevSibling`为当前作用域最后一个子级作用域

说完了创建作用域，再来说说`$watch`函数,这个比较关键

```
$watch: function(watchExp, listener, objectEquality) {
        var scope = this,
            get = compileToFn(watchExp, 'watch'),
            array = scope.$$watchers,
            watcher = {
              fn: listener,
              last: initWatchVal,
              get: get,
              exp: watchExp,
              eq: !!objectEquality
            };

        lastDirtyWatch = null;

        // in the case user pass string, we need to compile it, do we really need this ?
        if (!isFunction(listener)) {
          var listenFn = compileToFn(listener || noop, 'listener');
          watcher.fn = function(newVal, oldVal, scope) {listenFn(scope);};
        }

        if (typeof watchExp == 'string' && get.constant) {
          var originalFn = watcher.fn;
          watcher.fn = function(newVal, oldVal, scope) {
            originalFn.call(this, newVal, oldVal, scope);
            arrayRemove(array, watcher);
          };
        }

        if (!array) {
          array = scope.$$watchers = [];
        }
        // we use unshift since we use a while loop in $digest for speed.
        // the while loop reads in reverse order.
        array.unshift(watcher);

        return function deregisterWatch() {
          arrayRemove(array, watcher);
          lastDirtyWatch = null;
        };
      }
```

`$watch`函数有三个参数,第一个是监控参数,可以是字符串或者函数,第二个是监听函数,第三个是代表是否深度监听,注意看这个代码

```
get = compileToFn(watchExp, 'watch')
```
这个`compileToFn`函数其实是调用`$parse`实例来分析监控参数,然后返回一个函数,这个会在`dirty check`里用到，用来获取监控表达式的值,这个`$parseprovider`也是angularjs中用的比较多的,下面来重点的说下这个provider

##parseProvider

`$parse`的代码比较长，在源码文件夹中的ng目录里,`parse.js`里就是`$parse`的全部代码,当你了解完parse的核心之后，这部份代码其实可以独立出来，做成自己的计算器程序也是可以的,因为它的核心就是`解析字符串`,而且默认支持四则运算,运算符号的优先级处理,只是额外的增加了对变量的支持以及过滤器的支持，想想，把这块代码放在模板引擎里也是可以的，说多了，让我们来一步一步的分析parse代码吧.

###$get
记住，不管是哪个provider,先看它的`$get`属性,所以我们先来看看`$parse`的`$get`吧

```
this.$get = ['$filter', '$sniffer', '$log', function($filter, $sniffer, $log) {
    $parseOptions.csp = $sniffer.csp;

    promiseWarning = function promiseWarningFn(fullExp) {
      if (!$parseOptions.logPromiseWarnings || promiseWarningCache.hasOwnProperty(fullExp)) return;
      promiseWarningCache[fullExp] = true;
      $log.warn('[$parse] Promise found in the expression `' + fullExp + '`. ' +
          'Automatic unwrapping of promises in Angular expressions is deprecated.');
    };

    return function(exp) {
      var parsedExpression;

      switch (typeof exp) {
        case 'string':

          if (cache.hasOwnProperty(exp)) {
            return cache[exp];
          }

          var lexer = new Lexer($parseOptions);
          var parser = new Parser(lexer, $filter, $parseOptions);
          parsedExpression = parser.parse(exp, false);

          if (exp !== 'hasOwnProperty') {
            // Only cache the value if it's not going to mess up the cache object
            // This is more performant that using Object.prototype.hasOwnProperty.call
            cache[exp] = parsedExpression;
          }

          return parsedExpression;

        case 'function':
          return exp;

        default:
          return noop;
      }
    };
  }];
```
可以看出，假如解析的是函数，则直接返回，是字符串的话，则需要进行`parser.parse`方法,这里重点说下这个
###parser.parse

通过阅读`parse.js`文件，你会发现，这里有两个关键类

- `lexer`, 负责解析字符串，然后生成token,有点类似编译原理中的词法分析器

- `parser`, 负责对lexer生成的token,生成执行表达式，其实就是返回一个执行函数

看这里

```
var lexer = new Lexer($parseOptions);
var parser = new Parser(lexer, $filter, $parseOptions);
parsedExpression = parser.parse(exp, false);
```
第一句就是创建一个`lexer实例`,第二句是把`lexer实例`传给parser构造函数，然后生成`parser实例`,最后一句是调用`parser.parse`生成执行表达式，实质是一个函数

现在转到`parser.parse`里去

```
parse: function (text, json) {
    this.text = text;

    //TODO(i): strip all the obsolte json stuff from this file
    this.json = json;

    this.tokens = this.lexer.lex(text);

    console.log(this.tokens);

    if (json) {
      // The extra level of aliasing is here, just in case the lexer misses something, so that
      // we prevent any accidental execution in JSON.
      this.assignment = this.logicalOR;

      this.functionCall =
      this.fieldAccess =
      this.objectIndex =
      this.filterChain = function() {
        this.throwError('is not valid json', {text: text, index: 0});
      };
    }

    var value = json ? this.primary() : this.statements();

    if (this.tokens.length !== 0) {
      this.throwError('is an unexpected token', this.tokens[0]);
    }

    value.literal = !!value.literal;
    value.constant = !!value.constant;

    return value;
  }
```
视线移到这句`this.tokens = this.lexer.lex(text)`,然后来看看`lex`方法

```
lex: function (text) {
    this.text = text;

    this.index = 0;
    this.ch = undefined;
    this.lastCh = ':'; // can start regexp

    this.tokens = [];

    var token;
    var json = [];

    while (this.index < this.text.length) {
      this.ch = this.text.charAt(this.index);
      if (this.is('"\'')) {
        this.readString(this.ch);
      } else if (this.isNumber(this.ch) || this.is('.') && this.isNumber(this.peek())) {
        this.readNumber();
      } else if (this.isIdent(this.ch)) {
        this.readIdent();
        // identifiers can only be if the preceding char was a { or ,
        if (this.was('{,') && json[0] === '{' &&
            (token = this.tokens[this.tokens.length - 1])) {
          token.json = token.text.indexOf('.') === -1;
        }
      } else if (this.is('(){}[].,;:?')) {
        this.tokens.push({
          index: this.index,
          text: this.ch,
          json: (this.was(':[,') && this.is('{[')) || this.is('}]:,')
        });
        if (this.is('{[')) json.unshift(this.ch);
        if (this.is('}]')) json.shift();
        this.index++;
      } else if (this.isWhitespace(this.ch)) {
        this.index++;
        continue;
      } else {
        var ch2 = this.ch + this.peek();
        var ch3 = ch2 + this.peek(2);
        var fn = OPERATORS[this.ch];
        var fn2 = OPERATORS[ch2];
        var fn3 = OPERATORS[ch3];
        if (fn3) {
          this.tokens.push({index: this.index, text: ch3, fn: fn3});
          this.index += 3;
        } else if (fn2) {
          this.tokens.push({index: this.index, text: ch2, fn: fn2});
          this.index += 2;
        } else if (fn) {
          this.tokens.push({
            index: this.index,
            text: this.ch,
            fn: fn,
            json: (this.was('[,:') && this.is('+-'))
          });
          this.index += 1;
        } else {
          this.throwError('Unexpected next character ', this.index, this.index + 1);
        }
      }
      this.lastCh = this.ch;
    }
    return this.tokens;
  }
```
这里我们假如传进的字符串是1+2，通常我们分析源码的时候，碰到代码复杂的地方，我们可以简单化处理，因为逻辑都一样，只是情况不一样罢了。

上面的代码主要就是分析传入到`lex`内的字符串,以一个`whileloop`开始,然后依次检查当前字符是否是数字，是否是变量标识等，假如是数字的话，则转到`readNumber`方法,这里以1+2为例，当前ch是1,然后跳到`readNumber`方法

```
readNumber: function() {
    var number = '';
    var start = this.index;
    while (this.index < this.text.length) {
      var ch = lowercase(this.text.charAt(this.index));
      if (ch == '.' || this.isNumber(ch)) {
        number += ch;
      } else {
        var peekCh = this.peek();
        if (ch == 'e' && this.isExpOperator(peekCh)) {
          number += ch;
        } else if (this.isExpOperator(ch) &&
            peekCh && this.isNumber(peekCh) &&
            number.charAt(number.length - 1) == 'e') {
          number += ch;
        } else if (this.isExpOperator(ch) &&
            (!peekCh || !this.isNumber(peekCh)) &&
            number.charAt(number.length - 1) == 'e') {
          this.throwError('Invalid exponent');
        } else {
          break;
        }
      }
      this.index++;
    }
    number = 1 * number;
    this.tokens.push({
      index: start,
      text: number,
      json: true,
      fn: function() { return number; }
    });
  }
```

上面的代码就是检查从当前index开始的整个数字，包括带小数点的情况,检查完毕之后跳出loop,当前index向前进一个，以待以后检查后续字符串,最后保存到`lex实例`的`token`数组中,这里的`fn属性`就是以后执行时用到的,这里的`return number`是利用了JS的闭包特性,number其实就是检查时外层的number变量值.以1+2为例，这时index应该停在+这里，在lex的`while loop`中，+检查会跳到最后一个else里,这里有一个对象比较关键,OPERATORS,它保存着所有运算符所对应的动作，比如这里的+,对应的动作是

```
'+':function(self, locals, a,b){
      a=a(self, locals); b=b(self, locals);
      if (isDefined(a)) {
        if (isDefined(b)) {
          return a + b;
        }
        return a;
      }
      return isDefined(b)?b:undefined;}
```
大家注意了，这里有4个参数,可以先透露一下,第一个是传的是当前上下文对象，比喻当前`scope实例`,这个是为了获取字符串中的变量值,第二个参数是本地变量,是传递给函数当入参用的,基本用不到,最后两个参是关键,+是二元运算符，所以a代表左侧运算值,b代表右侧运算值.

最后解析完+之后，index停在了2的位置，跟1一样，也是返回一个`token`,`fn属性`也是一个返回当前数字的函数.

当解析完整个1+2字符串后，`lex`返回的是`token数组`,这个即可传递给parse来处理,来看看

```
var value = json ? this.primary() : this.statements();
```
默认json是false,所以会跳到`this.statements()`,这里将会生成执行语句.

```
 statements: function() {
    var statements = [];
    while (true) {
      if (this.tokens.length > 0 && !this.peek('}', ')', ';', ']'))
        statements.push(this.filterChain());
      if (!this.expect(';')) {
        // optimize for the common case where there is only one statement.
        // TODO(size): maybe we should not support multiple statements?
        return (statements.length === 1)
            ? statements[0]
            : function(self, locals) {
                var value;
                for (var i = 0; i < statements.length; i++) {
                  var statement = statements[i];
                  if (statement) {
                    value = statement(self, locals);
                  }
                }
                return value;
              };
      }
    }
  }
```

代码以一个无限loop的while开始,语句分析的时候是有运算符优先级的,默认的顺序是,这里以函数名为排序

**filterChain<expression<assignment<ternary<logicalOR<logicalAND<equality<relational<additive<multiplicative<unary<primary**

中文翻译下就是这样的

**过滤函数<一般表达式<赋值语句<三元运算<逻辑or<逻辑and<比较运算<关系运算<加减法运算<乘法运算<一元运算**,最后则默认取第一个token的fn属性

这里以`1+2`的`token`为例,这里会用到`parse`的`expect方法`,`expect`会用到`peek方法`

```
peek: function(e1, e2, e3, e4) {
    if (this.tokens.length > 0) {
      var token = this.tokens[0];
      var t = token.text;
      if (t === e1 || t === e2 || t === e3 || t === e4 ||
          (!e1 && !e2 && !e3 && !e4)) {
        return token;
      }
    }
    return false;
  },

  expect: function(e1, e2, e3, e4){
    var token = this.peek(e1, e2, e3, e4);
    if (token) {
      if (this.json && !token.json) {
        this.throwError('is not valid json', token);
      }
      this.tokens.shift();
      return token;
    }
    return false;
  }
```
`expect`方法传空就是默认从`token数组`中弹出第一个`token`，数组数量减1

`1+2`的执行语句最后会定位到加法运算那里additive

```
 additive: function() {
    var left = this.multiplicative();
    var token;
    while ((token = this.expect('+','-'))) {
      left = this.binaryFn(left, token.fn, this.multiplicative());
    }
    return left;
  }
```
最后返回一个二元操作的函数`binaryFn`

```
binaryFn: function(left, fn, right) {
    return extend(function(self, locals) {
      return fn(self, locals, left, right);
    }, {
      constant:left.constant && right.constant
    });
  }
```
这个函数参数里的left,right对应的'1','2'两个`token`的`fn属性`，即是

```
function(){ return number;}
```
`fn函数`对应`additive`方法中+号对应`token`的`fn`

```
function(self, locals, a,b){
      a=a(self, locals); b=b(self, locals);
      if (isDefined(a)) {
        if (isDefined(b)) {
          return a + b;
        }
        return a;
      }
	return isDefined(b)?b:undefined;
}
```
最后生成执行表达式函数,也就是`filterChain`返回的left值,被push到`statements`方法中的`statements数组`中,仔细看`statements`方法的返回值，假如表达式数组长度为1，则返回第一个执行表达式，否则返回一个包装的函数，里面是一个loop,不断的执行表达式，只返回最后一个表达式的值

```
return (statements.length === 1)
            ? statements[0]
            : function(self, locals) {
                var value;
                for (var i = 0; i < statements.length; i++) {
                  var statement = statements[i];
                  if (statement) {
                    value = statement(self, locals);
                  }
                }
                return value;
              }
```
好了，说完了`生成执行表达式`，其实`parse`的任务已经完成了，现在只需要把这个作为`parseprovider`的返回值了.

等会再回到`rootscope`的`$watch函数`解析里去,我们可以先测试下parse解析生成执行表达式的效果，这里贴一个独立的带parse的例子，不依赖angularjs。
