#Symbol

###背景

ES5及之前的对象属性名都是字符串，这样容易造成属性名的冲突。为了防止这一问题，保证每个属性的名字是独一无二的，因此ES6中引入了Symbol。

原始数据类型Symbol是继 Undefined、Null、布尔值（Boolean）、字符串（String）、数值（Number）、对象（Object）之外的第七种数据类型，表示独一无二的值。

###声明

Symbol值是由`Symbol`函数生成，这样对象的属性名既可以是字符串类型，又可以是Symbol值。凡是Symbol类型的，都是独一无二不会和其他属性名产生冲突的。

```
Symbol([description])
```
其中，description是一个字符串，用来描述符号，用于调试而不是访问符号本身。因此相同参数的Symbol函数的返回值实不相等的。

以下是Symbol的几个特点：

- Symbol函数前不能使用new命令，否则会报错。这是因为生成的Symbol是一个原始类型的值，不是对象。也就是说，由于Symbol值不是对象，所以不能添加属性。基本上，它是一种类似于字符串的数据类型。

```
var sym = new Symbol(); // TypeError
```
- Symbol函数的参数只是表示对当前Symbol值的描述，因此相同参数的Symbol函数的返回值是不相等的。

```
// 没有参数的情况
var s1 = Symbol();
var s2 = Symbol();

s1 === s2 // false

// 有参数的情况
var s1 = Symbol("foo");
var s2 = Symbol("foo");

s1 === s2 // false
```
- Symbol值不能与其他类型的值进行运算，会报错。

```
var sym = Symbol('My symbol');

"your symbol is " + sym
// TypeError: can't convert symbol to string
`your symbol is ${sym}`
// TypeError: can't convert symbol to string
```
-Symbol可以显示转化为字符串和布尔值。

```
var sym = Symbol('My symbol');
String(sym) // 'Symbol(My symbol)'
sym.toString() // 'Symbol(My symbol)'

var sym = Symbol();
Boolean(sym)// true
!sym  // false
```

###作为属性名

为了使对象的属性不出现同名的情况，可以用Symbol作为标识符。

```
var mySymbol = Symbol();

// 第一种写法
var a = {};
a[mySymbol] = 'Hello!';

// 第二种写法
var a = {
  [mySymbol]: 'Hello!'
};

// 第三种写法
var a = {};
Object.defineProperty(a, mySymbol, { value: 'Hello!' });

// 以上写法都得到同样结果
a[mySymbol] // "Hello!"
```
**当Symbol作为标识符时，不能用点运算符。**

1.因为点运算符后面总是字符串。 

```
var mySymbol = Symbol();
var a = {};

a.mySymbol = 'Hello!';
a[mySymbol] // undefined
a['mySymbol'] // "Hello!"
```
2.在对象内部，定义时必须在方括号中。

```
let s = Symbol();

let obj = {
  [s]: function (arg) { ... }
};

obj[s](123);
```

###定义一组值不相等的常量
Symbol类型还可以用于定义一组常量，保证这组常量的值都不相等。

```
const COLOR_RED    = Symbol();
const COLOR_GREEN  = Symbol();

function getComplement(color) {
  switch (color) {
    case COLOR_RED:
      return COLOR_GREEN;
    case COLOR_GREEN:
      return COLOR_RED;
    default:
      throw new Error('Undefined color');
    }
}
```
常量使用Symbol值最大的好处，就是其他任何值都不可能有相同的值了，因此可以保证上面的switch语句会按设计的方式工作。

###Symbol的遍历
当Symbol作为属性名时，不会出现在`for...in`、`for...of`中，也不会被`Object.keys()`、`Object.getOwnPropertyNames()`。它不是私有属性，可以利用`Object.getOwnPropertySymbols`方法返回对象的所有Symbol属性名。

```
var obj = {};
var a = Symbol('a');
var b = Symbol.for('b');

obj[a] = 'Hello';
obj[b] = 'World';

var objectSymbols = Object.getOwnPropertySymbols(obj);

objectSymbols
// [Symbol(a), Symbol(b)]
```
**利用新的API，`Reflect.ownKeys`可以返回所有类型的键名，包括Symbol键名**

```
let obj = {
  [Symbol('my_key')]: 1,
  enum: 2,
  nonEnum: 3
};

Reflect.ownKeys(obj)
// [Symbol(my_key), 'enum', 'nonEnum']
```
###Symbol类型的方法
####Symbol.for() 
有时希望重新用同一个Symbol的值，`Symbol.for`方法接收一个字符串作为参数，然后搜索是否有以该参数作为名称的Symbol值，如果有则返回这个Symbol值，如果没有则新建并返回以这个字符串作为参数的Symbol值。

```
var s1 = Symbol.for('foo');
var s2 = Symbol.for('foo');
s1 === s2 // true

var s1 = Symbol('foo');
var s2 = Symbol.for('foo');
s1 === s2 // false
```
Symbol.for()与Symbol()这两种写法，都会生成新的Symbol。它们的区别是，前者会被登记在全局环境中供搜索，后者不会。Symbol.for()不会每次调用就返回一个新的Symbol类型的值，而是会先检查给定的key是否已经存在，如果不存在才会新建一个值。

####Symbol.keyFor()
`Symbol.keyFor`返回一个`已登记`的Symbol类型值的key。

```
var s1 = Symbol.for("foo");
Symbol.keyFor(s1) // "foo"

var s2 = Symbol("foo");
Symbol.keyFor(s2) // undefined
```
上面代码中，变量s2属于未登记的Symbol值，所以返回undefined。


###内置的Symbol值

####Symbol.hasInstance
对象的`Symbol.hasInstance`属性，指向一个内部方法。该对象使用`instanceof`运算符时，会调用这个方法，判断该对象是否为某个构造函数的实例。比如，foo instanceof Foo在语言内部，实际调用的是`Foo[Symbol.hasInstance](foo)`。

###应用 TODO
魔术贴属性VS私有的内部方法