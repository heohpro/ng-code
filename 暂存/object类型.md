##Object类型详解：

###介绍

Object构造函数为给定的值创建一个对象包装。如果给定的值是`null`或`undefined`，则会创建并返回一个空对象，否则，将创建一个与给定值对应类型的对象。
当以非构造函数形式被调用时，Object等同于new Object()。

```
var o = new Object(); // Object {}

var o = new Object(undefined); // Object {}

var o = new Object(null); // Object {}
```

###属性

- Object.length: 属性值为1。
- Object.prototype: 可以为所有的Object类型的对象添加属性。

###属性描述符

对象中的属性描述符主要有两种形式：数据描述符和存取描述符，描述符必须是两种形式之一，不能同时是两者。

- 数据描述符是一个拥有可写或不可写值的属性，有以下可选键值。
	- value: 该属性对应的值，**默认为undefined**。
	- writable: 当且仅当仅当该属性的`writable`为true时，该属性才能被赋值运算符改变。**默认为 false**。
- 存取描述符是一堆`getter-setter`函数功能来描述的属性。
	- get: 一个给属性提供`getter`的方法，如果没有`getter`则为undefined。该方法将返回一个值，这个值即被当作该属性的值。**默认为undefined**。
	- set: 一个给属性提供`setter`的方法，如果没有`getter`则为undefined。该方法将接受唯一参数，并将该参数的新值分配给该属性。**默认为undefined**。
- 数据描述符和存取描述符均具有以下可选属性.
	- configurable: 当且晋档该属性的`configurable`属性为true时，该属性才能够被改变，也能够被删除。**默认为 false**。
	- enumerable: 当且仅当该属性的`enumerable`为true时，该属性才能够出现在对象的枚举属性中。**默认为 false**。

```
//定义 getter 与 setter
(function () {
    var o = {
        a : 7,
        get b(){return this.a +1;},//通过 get,set的 b,c方法间接性修改 a 属性
        set c(x){this.a = x/2}
    };
    console.log(o.a);
    console.log(o.b);
    o.c = 50;
    console.log(o.a);
})();
```

###对象的创建
javascript中对象的创建主要有几种方式：

- 对象直接量：所谓对象直接量，可以看做是一副映射表，这个方法也是最直接的一个方法	
	
```
//创建简单对象
var obj1 = {}; //空对象
var obj2 = {
  name: "ys",
  age: 12
};
//创建复杂对象
var obj3 = {
  name: "ys",
  age: 12,
  like: {
    drink: "water",
    eat: "food"
  }
};
 
console.log(typeof obj1);  //object
console.log(typeof obj2);  //object
console.log(typeof obj3);  //object
```
- new创建对象

```
//系统内置对象
var obj1 = new Object();
var obj2 = new Array();
var obj3 = new Date();
var obj4 = new RegExp("ys");
 
console.log(typeof obj1);  //object
console.log(typeof obj2);  //object
console.log(typeof obj3);  //object
console.log(typeof obj4);  //object

//自定义对象
function Person(name, age){
  this.name = name;
  this.age = age;
}
var obj1 = new Person("ys", 12);
console.log(Object.prototype.toString.call(obj1));  //object
console.log(Person instanceof Object);        //true
console.log(typeof obj1);              //object
console.log(obj1.age);                //12
```
	 
- Object.create创建

```
var obj1 = Object.create({
  name: "ys",
  age: 12
});
console.log(obj1);     //{}
console.log(obj1.age);   //12
```
	
###方法

####Object.create: 

Object.create创建一个具有指定原型且可选择性地包含指定属性的对象。

#####语法

```
Object.create(prototype, descriptors)

```

#####参数
- prototype: 必须，要用作原型的对象，可以为null。

- descriptors: 可选，包含一个或多个属性描述符的js对象。
	- “数据属性”是可获取且可设置值的属性。 数据属性描述符包含 value 特性，以及 writable、enumerable 和 configurable 特性。 
	- 如果未指定最后三个特性，则它们默认为 false。
	
#####返回值
一个具有指定内部原型并且包含指定属性（如果有）的新对象。

例子 - 使用Object.create 的 propertyObject参数

```
var o;

// 创建一个原型为null的空对象
o = Object.create(null);


o = {};
// 以字面量方式创建的空对象就相当于:
o = Object.create(Object.prototype);


o = Object.create(Object.prototype, {
  // foo会成为所创建对象的数据属性
  foo: { writable:true, configurable:true, value: "hello" },
  // bar会成为所创建对象的访问器属性
  bar: {
    configurable: false,
    get: function() { return 10 },
    set: function(value) { console.log("Setting `o.bar` to", value) }
}})


function Constructor(){}
o = new Constructor();
// 上面的一句就相当于:
o = Object.create(Constructor.prototype);
// 当然,如果在Constructor函数中有一些初始化代码,Object.create不能执行那些代码


// 创建一个以另一个空对象为原型,且拥有一个属性p的对象
o = Object.create({}, { p: { value: 42 } })

// 省略了的属性特性默认为false,所以属性p是不可写,不可枚举,不可配置的:
o.p = 24
o.p
//42

o.q = 12
for (var prop in o) {
   console.log(prop)
}
//"q"

delete o.p
//false

//创建一个可写的,可枚举的,可配置的属性p
o2 = Object.create({}, { p: { value: 42, writable: true, enumerable: true, configurable: true } });
```

例子 - 使用Object.create实现类式继承

```
//Shape - superclass
function Shape() {
  this.x = 0;
  this.y = 0;
}

Shape.prototype.move = function(x, y) {
    this.x += x;
    this.y += y;
    console.info("Shape moved.");
};

// Rectangle - subclass
function Rectangle() {
  Shape.call(this); //call super constructor.
}

Rectangle.prototype = Object.create(Shape.prototype);

var rect = new Rectangle();

rect instanceof Rectangle //true.
rect instanceof Shape //true.

rect.move(1, 1); //Outputs, "Shape moved."
```


####Object.defineProperty

`Object.defineProperty`方法直接在一个对象上定义一个新属性，或者修改一个已经存在的属性， 并返回这个对象。

语法如下：

```
Object.defineProperty(obj, prop, descriptor)
```
- 参数：`obj`为需要定义属性的对象,`prop`为需要被定义或修改的属性名，`descriptor`为属性的描述符。
- 描述：该方法允许精确添加或修改对象的属性。一般情况下，我们为对象添加属性是通过赋值来创建并显示在属性枚举中（for...in 或 Object.keys 方法）， 但这种方式添加的属性值可以被改变，也可以被删除。而使用 Object.defineProperty() 则允许改变这些额外细节的默认设置。例如，默认情况下，使用  Object.defineProperty() 增加的属性值是不可改变的。。

例子 - 创建属性

如果对象中不存在指定的属性，Object.defineProperty()就创建这个属性。当描述符中省略某些字段时，这些字段将使用它们的默认值。拥有布尔值的字段的默认值都是fasle。value，get和set字段的默认值为undefined。定义属性时如果没有get/set/value/writable，那它被归类为数据描述符。

```
var o = {}; // 创建一个新对象

// Example of an object property added with defineProperty with a data property descriptor
Object.defineProperty(o, "a", {value : 37,
                               writable : true,
                               enumerable : true,
                               configurable : true});
// 对象o拥有了属性a，值为37

// Example of an object property added with defineProperty with an accessor property descriptor
var bValue;
Object.defineProperty(o, "b", {get : function(){ return bValue; },
                               set : function(newValue){ bValue = newValue; },
                               enumerable : true,
                               configurable : true});
o.b = 38;
// 对象o拥有了属性b，值为38

// The value of o.b is now always identical to bValue, unless o.b is redefined

// 数据描述符和存取描述符不能混合使用
Object.defineProperty(o, "conflict", { value: 0x9f91102, 
                                       get: function() { return 0xdeadbeef; } });
// throws a TypeError: value appears only in data descriptors, get appears only in accessor descriptors
```

例子 - 修改属性

- 如果属性已经存在，Object.defineProperty()将根据描述符中的值以及对象当前的配置来修改这个属性。如果描述符的configurable特性为false，那么除了writable特性外，其他属性都不能被修改，并且数据和存取描述符也不能相互切换。
- 如果一个属性的configurable为false，则writable特性也只能设置为false。
- 如果修改属性特性non-configurable（同样适用于writable特性），将会产生一个TypeError异常，除非修改为与当前相同的值。

Writable 特性 定义了属性是否可以修改。

```
var o = {}; // 创建一个新对象
Object.defineProperty(o, "a", { value : 37,
                                writable : false });
console.log(o.a); // 打印 37
o.a = 25; // 没有错误抛出（在严格模式下会抛出，即使之前已经有相同的值）
console.log(o.a); // 打印 37， 赋值不起作用。
```

Enumerable 特性 定义了属性是否可以在`for...in`循环和`Object.keys()`中被枚举。

```
var o = {};
Object.defineProperty(o, "a", { value : 1, enumerable:true });
Object.defineProperty(o, "b", { value : 2, enumerable:false });
Object.defineProperty(o, "c", { value : 3 }); // enumerable defaults to false
o.d = 4; // 如果使用直接赋值的方式创建对象的属性，则这个属性的enumerable为true

for (var i in o) {    
  console.log(i);  
}
// 打印 'a' 和 'd' (in undefined order)

Object.keys(o); // ["a", "d"]

o.propertyIsEnumerable('a'); // true
o.propertyIsEnumerable('b'); // false
o.propertyIsEnumerable('c'); // false
```
Configurable 特性 表示对象的属性是否可以被删除，以及除writable特性之外的特性是否可以被修改。

```
var o = {};
Object.defineProperty(o, "a", { get : function(){return 1;}, 
                                configurable : false } );

Object.defineProperty(o, "a", {configurable : true}); // throws a TypeError
Object.defineProperty(o, "a", {enumerable : true}); // throws a TypeError
Object.defineProperty(o, "a", {set : function(){}}); // throws a TypeError (set was undefined previously)
Object.defineProperty(o, "a", {get : function(){return 1;}}); // throws a TypeError (even though the new get does exactly the same thing)
Object.defineProperty(o, "a", {value : 12}); // throws a TypeError

console.log(o.a); // logs 1
delete o.a; // Nothing happens
console.log(o.a); // logs 1
```
属性特性的默认值 使用点运算符和Object.defineProperty()为属性赋值时，属性默认值是不同的。

```
var o = {};

o.a = 1;
// 等同于 :
Object.defineProperty(o, "a", {
  value : 1,
  writable : true,
  configurable : true,
  enumerable : true
});


// 另一方面，
Object.defineProperty(o, "a", { value : 1 });
// 等同于 :
Object.defineProperty(o, "a", {
  value : 1,
  writable : false,
  configurable : false,
  enumerable : false
});
```

####Object.defineProperties

Object.defineProperties方法在一个对象上添加或修改一个或者多个自有属性，并返回该对象。

语法如下：

```
Object.defineProperties(obj, props)
```
- 参数：第一个参数为被添加或修改的对象，后面为多个键值对属性的具体配置。
- 描述：相比于`Object.defineProperty`方法，该方法支持同时定义或修改多个自有属性。

例子

```
var obj = {};
Object.defineProperties(obj, {
  "property1": {
    value: true,
    writable: true
  },
  "property2": {
    value: "Hello",
    writable: false
  }
  // 等等.
});
alert(obj.property2) //弹出"Hello"
```
####Object.preventExtensions

Object.preventExtensions() 方法让一个对象变的不可扩展，也就是永远不能再添加新的属性。

语法如下：

```
Object.preventExtensions(obj)
```
- 参数：obj为将要变得不可扩展的对象。
- 描述：
	如果一个对象可以添加新的属性，则这个对象是可扩展的。preventExtensions 可以让这个对象变的不可扩展，也就是不能再有新的属性。需要注意的是不可扩展的对象的属性通常仍然可以被删除。尝试给一个不可扩展对象添加新属性的操作将会失败，不过可能是静默失败，也可能会抛出 TypeError 异常（严格模式）。
	
**Object.preventExtensions 只能阻止一个对象不能再添加新的自身属性，仍然可以为该对象的原型添加属性。**

例子

```
// Object.preventExtensions将原对象变的不可扩展,并且返回原对象.
var obj = {};
var obj2 = Object.preventExtensions(obj);
assert(obj === obj2);
 
// 字面量方式定义的对象默认是可扩展的.
var empty = {};
assert(Object.isExtensible(empty) === true);
 
// ...但可以改变.
Object.preventExtensions(empty);
assert(Object.isExtensible(empty) === false);
 
// 使用Object.defineProperty方法为一个不可扩展的对象添加新属性会抛出异常.
var nonExtensible = { removable: true };
Object.preventExtensions(nonExtensible);
Object.defineProperty(nonExtensible, "new", { value: 8675309 }); // 抛出TypeError异常
 
// 在严格模式中,为一个不可扩展对象的新属性赋值会抛出TypeError异常.
function fail()
{
  "use strict";
  nonExtensible.newProperty = "FAIL"; // throws a TypeError
}
fail();
 
// 一个不可扩展对象的原型是不可更改的,__proto__是个非标准魔法属性,可以更改一个对象的原型.
var fixed = Object.preventExtensions({});
fixed.__proto__ = { oh: "hai" }; // 抛出TypeError异常
```
####Object.seal

Object.seal() 方法可以让一个对象密封，并返回被密封后的对象。密封对象是指那些不能添加新的属性，不能删除已有属性，以及不能修改已有属性的可枚举性、可配置性、可写性，但可能可以修改已有属性的值的对象。

语法如下：

```
Object.seal(obj)
```
- 参数：obj为将要变得密封的对象。
- 描述：
	通常情况下，一个对象是可扩展的（可以添加新的属性）。密封一个对象会让这个对象变的不能添加新属性，且所有已有属性会变的不可配置。属性不可配置的效果就是属性变的不可删除，以及一个数据属性不能被重新定义成为访问器属性，或者反之。但属性的值仍然可以修改。尝试删除一个密封对象的属性或者将某个密封对象的属性从数据属性转换成访问器属性，结果会静默失败或抛出TypeError 异常（严格模式）。
	
**不会影响从原型链上继承的属性。但 __proto__ (  ) 属性的值也会不能修改。**

例子

```
var obj = {
    prop: function () {},
    foo: "bar"
  };

// 可以添加新的属性,已有属性的值可以修改,可以删除
obj.foo = "baz";
obj.lumpy = "woof";
delete obj.prop;

var o = Object.seal(obj);

assert(o === obj);
assert(Object.isSealed(obj) === true);

// 仍然可以修改密封对象上的属性的值.
obj.foo = "quux";

// 但你不能把一个数据属性重定义成访问器属性.
Object.defineProperty(obj, "foo", { get: function() { return "g"; } }); // 抛出TypeError异常

// 现在,任何属性值以外的修改操作都会失败.
obj.quaxxor = "the friendly duck"; // 静默失败,新属性没有成功添加
delete obj.foo; // 静默失败,属性没有删除成功

// ...在严格模式中,会抛出TypeError异常
function fail() {
  "use strict";
  delete obj.foo; // 抛出TypeError异常
  obj.sparky = "arf"; // 抛出TypeError异常
}
fail();

// 使用Object.defineProperty方法同样会抛出异常
Object.defineProperty(obj, "ohai", { value: 17 }); // 抛出TypeError异常
Object.defineProperty(obj, "foo", { value: "eit" }); // 成功将原有值改变
```

####Object.freeze

Object.freeze() 方法可以冻结一个对象。冻结对象是指那些不能添加新的属性，不能修改已有属性的值，不能删除已有属性，以及不能修改已有属性的可枚举性、可配置性、可写性的对象。也就是说，这个对象永远是不可变的。该方法返回被冻结的对象。

语法如下：

```
Object.freeze(obj)
```
- 参数：第一个参数将要被冻结的对象，返回被冻结的对象。
- 描述：
	- 冻结对象的所有自身属性都不可能以任何方式被修改。任何尝试修改该对象的操作都会失败，可能是静默失败，也可能会抛出异常（严格模式中）。
	- 数据属性的值不可更改，访问器属性（有getter和setter）也同样（但由于是函数调用，给人的错觉是还是可以修改这个属性）。如果一个属性的值是个对象，则这个对象中的属性是可以修改的，除非它也是个冻结对象。

例子

```
var obj = {
  prop: function (){},
  foo: "bar"
};

// 可以添加新的属性,已有的属性可以被修改或删除
obj.foo = "baz";
obj.lumpy = "woof";
delete obj.prop;

var o = Object.freeze(obj);

// 现在任何修改操作都会失败
obj.foo = "quux"; // 静默失败
obj.quaxxor = "the friendly duck"; // 静默失败,并没有成功添加上新的属性

// ...在严格模式中会抛出TypeError异常
function fail(){
  "use strict";
  obj.foo = "sparky"; // 抛出TypeError异常
  delete obj.quaxxor; // 抛出TypeError异常
  obj.sparky = "arf"; // 抛出TypeError异常
}

fail();

// 使用Object.defineProperty方法同样会抛出TypeError异常
Object.defineProperty(obj, "ohai", { value: 17 }); // 抛出TypeError异常
Object.defineProperty(obj, "foo", { value: "eit" }); // 抛出TypeError异常
```

某个属性的值是对象，进行深度冻结

```
obj = {
  internal : {}
};

Object.freeze(obj);
obj.internal.a = "aValue";

obj.internal.a // "aValue"

// 想让一个对象变的完全冻结,冻结所有对象中的对象,我们可以使用下面的函数.

function deepFreeze (o) {
  var prop, propKey;
  Object.freeze(o); // 首先冻结第一层对象.
  for (propKey in o) {
    prop = o[propKey];
    if (!o.hasOwnProperty(propKey) || !(typeof prop === "object") || Object.isFrozen(prop)) {
      // 跳过原型链上的属性和已冻结的对象.
      continue;
    }

    deepFreeze(prop); //递归调用.
  }
}

obj2 = {
  internal : {}
};

deepFreeze(obj2);
obj2.internal.a = "anotherValue";
obj2.internal.a; // undefined
```

####Object.setPrototypeOf 

Object.setPrototypeOf() 将一个指定的对象的原型设置为另一个对象或者null(既对象的[[Prototype]]内部属性).

语法如下：

```
Object.setPrototypeOf(obj, prototype)
```
- 参数：obj为被设置原型的对象，prototype为新的原型。
- 描述：如果对象的[[Prototype]]被修改成不可扩展(通过 Object.isExtensible()查看)，就会抛出 TypeError异常。如果prototype参数不是一个对象或者null(例如，数字，字符串，boolean，或者 undefined)，则什么都不做。否则，该方法将obj的[[Prototype]]修改为新的值。

例子

```
var dict = Object.setPrototypeOf({}, null);
```

####Object.getPrototypeOf

Object.getPrototypeOf() 方法返回指定对象的原型（即对象内部属性[[prototype]]的值）。

语法如下：

```
Object.getPrototypeOf(object)
```
- 参数：返回该对象的原型。
- 描述：在 ES5 中，如果参数不是一个对象类型，将抛出一个 TypeError 异常。在 ES6 中，参数被强制转换为Object。

例子

```
var proto = {};
var obj = Object.create(proto);
Object.getPrototypeOf(obj) === proto; // true

Object.getPrototypeOf("foo");
TypeError: "foo" is not an object  // ES5 code
Object.getPrototypeOf("foo");
String.prototype                   // ES6 code
```
####Object.getOwnPropertyDescriptor

Object.getOwnPropertyDescriptor() 返回指定对象上一个自有属性对应的属性描述符。（自有属性指的是直接赋予该对象的属性，不需要从原型链上进行查找的属性）

语法如下：

```
Object.getOwnPropertyDescriptor(obj, prop)
```
- 参数：obj在该对象上查看属性，prop该属性的属性描述符将被返回。
- 返回值：如果属性存在对象上，则返回属性描述符，否则返回`undefined`。
- 描述：该方法允许对一个属性的描述进行检索。在 Javascript 中， 属性 由一个字符串类型的“名字”（name）和一个“属性描述符”（property descriptor）对象构成。
	- value：该属性的值(仅针对数据属性描述符有效)。
	- writable：当且仅当属性的值可以被改变时为true。(仅针对数据属性描述有效)
	- get：获取该属性的访问器函数（getter）。如果没有访问器， 该值为undefined。(仅针对包含访问器或设置器的属性描述有效)
	- set：获取该属性的设置器函数（setter）。 如果没有设置器， 该值为undefined。(仅针对包含访问器或设置器的属性描述有效)
	- configurable：当且仅当指定对象的属性描述可以被改变或者属性可被删除时，为true。
	- enumerable：当且仅当指定对象的属性可以被枚举出时，为 true。


例子

```
var o, d;

o = { get foo() { return 17; } };
d = Object.getOwnPropertyDescriptor(o, "foo");
// d is { configurable: true, enumerable: true, get: /*访问器函数*/, set: undefined }

o = { bar: 42 };
d = Object.getOwnPropertyDescriptor(o, "bar");
// d is { configurable: true, enumerable: true, value: 42, writable: true }

o = {};
Object.defineProperty(o, "baz", { value: 8675309, writable: false, enumerable: false });
d = Object.getOwnPropertyDescriptor(o, "baz");
// d is { value: 8675309, writable: false, enumerable: false, configurable: false }
```

####Object.getOwnPropertyNames

Object.getOwnPropertyNames() 方法返回一个由指定对象的所有自身属性的属性名（**包括不可枚举属性**）组成的数组。

语法如下：

```
Object.getOwnPropertyNames(obj)
```
- 参数：obj一个对象，其自身的可枚举和不可枚举属性的名称被返回。
- 返回值：如果属性存在对象上，则返回属性描述符，否则返回`undefined`。
- 描述：Object.getOwnPropertyNames 返回一个数组，该数组对元素是 obj 自身拥有的枚举或不可枚举属性名称字符串。 数组中枚举属性的顺序与通过 for...in loop（或 Object.keys)）迭代该对象属性时一致。 数组中不可枚举属性的顺序未定义。


例子 - 获得属性

```
var arr = ["a", "b", "c"];
console.log(Object.getOwnPropertyNames(arr).sort()); // ["0", "1", "2", "length"]

// 类数组对象
var obj = { 0: "a", 1: "b", 2: "c"};
console.log(Object.getOwnPropertyNames(obj).sort()); // ["0", "1", "2"]

// 使用Array.forEach输出属性名和属性值
Object.getOwnPropertyNames(obj).forEach(function(val, idx, array) {
  console.log(val + " -> " + obj[val]);
});
// 输出
// 0 -> a
// 1 -> b
// 2 -> c

//不可枚举属性
var my_obj = Object.create({}, {
  getFoo: {
    value: function() { return this.foo; },
    enumerable: false
  }
});
my_obj.foo = 1;

console.log(Object.getOwnPropertyNames(my_obj).sort()); // ["foo", "getFoo"]
```

要只获取可枚举属性时，可以用`Object.keys`方法或者`for..in`的循环（还会获取到原型链上的可枚举属性，不过可以使用`hasOwnProperty()`方法过滤掉）

例子 - 不获取原型链的属性

```
function ParentClass() {}
ParentClass.prototype.inheritedMethod = function() {};

function ChildClass() {
    this.prop = 5;
    this.method = function() {};
}

ChildClass.prototype = new ParentClass;

ChildClass.prototype.prototypeMethod = function() {};

console.log(Object.getOwnPropertyNames(new ChildClass()) // ["prop", "method"])
```

例子 - 仅获取不可枚举的属性（利用`Array.prototype.filter`进行过滤）

从所有的属性名数组（使用  Object.getOwnPropertyNames() 方法获得）中去除可枚举的属性（使用  Object.keys() 方法获得），剩余的属性便是不可枚举的属性了

```
var target = myObject;
var enum_and_nonenum = Object.getOwnPropertyNames(target);
var enum_only = Object.keys(target);
var nonenum_only = enum_and_nonenum.filter(function(key) {
    var indexInEnum = enum_only.indexOf(key);
    if (indexInEnum == -1) {
        // not found in enum_only keys mean the key is non-enumerable,
        // so return true so we keep this in the filter
        return true;
    } else {
        return false;
    }
});

console.log(nonenum_only);
```

####Object.getOwnPropertySymbols

Object.getOwnPropertySymbols() 方法会返回一个数组，该数组包含了指定对象自身的（非继承的）所有 symbol 属性键。

语法如下：

```
Object.getOwnPropertySymbols(obj)
```
- 参数：obj为任意对象。
- 返回值：如果属性存在对象上，则返回属性描述符，否则返回`undefined`。
- 描述：该方法和 Object.getOwnPropertyNames() 类似，但后者返回的结果只会包含字符串类型的属性键，也就是传统的属性名。


例子

```
var obj = {};
var a = Symbol("a");
var b = Symbol.for("b");

obj[a] = "localSymbol";
obj[b] = "globalSymbol";

var objectSymbols = Object.getOwnPropertySymbols(obj);

console.log(objectSymbols.length); // 2
console.log(objectSymbols)         // [Symbol(a), Symbol(b)]
console.log(objectSymbols[0])      // Symbol(a)
```

####Object.keys

Object.keys() 方法会返回一个由给定对象的所有可枚举自身属性的属性名组成的数组，数组中属性名的排列顺序和使用for-in循环遍历该对象时返回的顺序一致（两者的主要区别是 for-in 还会遍历出一个对象从其原型链上继承到的可枚举属性）。

语法如下：

```
Object.keys(obj)
```
- 参数：obj为返回该对象的所有可枚举自身属性的属性名。
- 描述：Object.keys 返回一个所有元素为字符串的数组，其元素来自于从给定的对象上面可直接枚举的属性。这些属性的顺序与手动遍历该对象属性时的一致。


例子

```
var arr = ["a", "b", "c"];
alert(Object.keys(arr)); // 弹出"0,1,2"

// 类数组对象
var obj = { 0 : "a", 1 : "b", 2 : "c"};
alert(Object.keys(obj)); // 弹出"0,1,2"

// getFoo是个不可枚举的属性
var my_obj = Object.create({}, { getFoo : { value : function () { return this.foo } } });
my_obj.foo = 1;

alert(Object.keys(my_obj)); // 只弹出foo
```

####Object.assign

Object.assign方法可以把任意多个源对象所拥有的自身可枚举的属性拷贝给目标对象，然后返回目标对象。

语法如下：

```
Object.assign(target, ...sources)
```
- 参数：第一个参数为目标对象，后面为任意多个源对象。
- 返回值：目标对象将被返回。
- 描述：该方法会拷贝源对象自身的并且可枚举的属性到目标对象上。对于访问器属性，则会执行访问器属性的getter函数，然后把得到的值拷贝给目标对象。如果想获得访问器属性本身，需使用`Object.getOwnPropertyDescriptor()`和`Object.defineProperties()`方法。

例子：浅拷贝一个对象

```
//
var obj = { a: 1 };
var copy = Object.assign({}, obj);
console.log(copy); // { a: 1 }
```
例子：合并若干个对象

```
var o1 = { a: 1 };
var o2 = { b: 2 };
var o3 = { c: 3 };

var obj = Object.assign(o1, o2, o3);
console.log(obj); // { a: 1, b: 2, c: 3 }
console.log(o1);  // { a: 1, b: 2, c: 3 }, 注意目标对象自身也会改变。
```
例子：继承属性和不可枚举属性是不能拷贝的

```
var obj = Object.create({foo: 1}, { // foo 是个继承属性。
    bar: {
        value: 2  // bar 是个不可枚举属性。
    },
    baz: {
        value: 3,
        enumerable: true  // baz 是个自身可枚举属性。
    }
});

var copy = Object.assign({}, obj);
console.log(copy); // { baz: 3 }
```
例子：原始值会被隐式转换成其包装对象 

```
var v1 = "123";
var v2 = true;
var v3 = 10;
var v4 = Symbol("foo")

var obj = Object.assign({}, v1, null, v2, undefined, v3, v4); 
// 源对象如果是原始值，会被自动转换成它们的包装对象，
// 而 null 和 undefined 这两种原始值会被完全忽略。
// 注意，只有字符串的包装对象才有可能有自身可枚举属性。
console.log(obj); // { "0": "1", "1": "2", "2": "3" }
```
例子：拷贝属性过程中发生异常

```
var target = Object.defineProperty({}, "foo", {
    value: 1,
    writeable: false
}); // target 的 foo 属性是个只读属性。

Object.assign(target, {bar: 2}, {foo2: 3, foo: 3, foo3: 3}, {baz: 4});
// TypeError: "foo" is read-only
// 注意这个异常是在拷贝第二个源对象的第二个属性时发生的。

console.log(target.bar);  // 2，说明第一个源对象拷贝成功了。
console.log(target.foo2); // 3，说明第二个源对象的第一个属性也拷贝成功了。
console.log(target.foo);  // 1，只读属性不能被覆盖，所以第二个源对象的第二个属性拷贝失败了。
console.log(target.foo3); // undefined，异常之后 assign 方法就退出了，第三个属性是不会被拷贝到的。
console.log(target.baz);  // undefined，第三个源对象更是不会被拷贝到的。
```
例子：实现访问器属性的拷贝

```
var obj = {
    foo: 1,
    get bar() {
        return 2;
    }
};

var copy = Object.assign({}, obj); 
console.log(copy); 
// { foo: 1, bar: 2 }，copy.bar 的值成了 obj.bar 的 getter 函数的返回值。

// 下面实现一个自己的 assign 函数，它可以拷贝访问器属性。
function myAssign(target, ...sources) {
    sources.forEach(source => {
        Object.defineProperties(target, Object.keys(source).reduce((descriptors, key) => {
            descriptors[key] = Object.getOwnPropertyDescriptor(source, key);
            return descriptors;
        }, {}));
    });
    return target;
}
      
var copy = myAssign({}, obj);
console.log(copy);
// {foo:1, get bar() {return 2}}，访问器属性也原封不动拷过来了。
```

####Object.is 

Object.is方法用来判断两个值是否是同一个值。

语法如下：

```
Object.is(value1, value2);
```
- 参数：第一个参数为需要比较的第一个值，后面为需要比较的第二个值。
- 返回值：返回一个布尔值，表明传入的两个值是否是同一个值。
- 描述：Object.is() 会在下面这些情况下认为两个值是相同的：
	- 两个值都是 undefined
	- 两个值都是 null
	- 两个值都是 true 或者都是 false
	- 两个值是由相同个数的字符按照相同的顺序组成的字符串
	- 两个值指向同一个对象
	- 两个值都是数字并且
	- 都是正零 +0
	- 都是负零 -0
	- 都是 NaN
	- 都是除零和 NaN 外的其它同一个数字

例子：

```
Object.is('foo', 'foo');     // true
Object.is(window, window);   // true

Object.is('foo', 'bar');     // false
Object.is([], []);           // false

var test = { a: 1 };
Object.is(test, test);       // true

Object.is(null, null);       // true

// 两个特例，=== 也没法判断的情况
Object.is(0, -0);            // false
Object.is(NaN, 0/0);         // true
```

####Object.isExtensible  

Object.isExtensible() 方法判断一个对象是否是可扩展的（是否可以在它上面添加新的属性）。

语法如下：

```
Object.isExtensible(obj)
```
- 描述：默认情况下，对象是可扩展的：即可以为他们添加新的属性。以及它们的 __proto__  属性可以被更改。Object.preventExtensions，Object.seal 或 Object.freeze 方法都可以标记一个对象为不可扩展（non-extensible）。

例子：

```
// 新对象默认是可扩展的.
var empty = {};
Object.isExtensible(empty); // === true

// ...可以变的不可扩展.
Object.preventExtensions(empty);
Object.isExtensible(empty); // === false

// 密封对象是不可扩展的.
var sealed = Object.seal({});
Object.isExtensible(sealed); // === false

// 冻结对象也是不可扩展.
var frozen = Object.freeze({});
Object.isExtensible(frozen); // === false
```

####Object.isFrozen 

Object.isFrozen() 方法判断一个对象是否被冻结（frozen）。

语法如下：

```
Object.isFrozen(obj)
```
- 描述：一个对象是冻结的（frozen）是指它不可扩展，所有属性都是不可配置的（non-configurable），且所有数据属性（data properties）都是不可写的（non-writable）。数据属性是值那些没有取值器（getter）或赋值器（setter）的属性。

例子：

```
// 一个对象默认是可扩展的,所以它也是非冻结的.
assert(Object.isFrozen({}) === false);

// 一个不可扩展的空对象同时也是一个冻结对象.
var vacuouslyFrozen = Object.preventExtensions({});
assert(Object.isFrozen(vacuouslyFrozen) === true);

// 一个非空对象默认也是非冻结的.
var oneProp = { p: 42 };
assert(Object.isFrozen(oneProp) === false);

// 让这个对象变的不可扩展,并不意味着这个对象变成了冻结对象,
// 因为p属性仍然是可以配置的(而且可写的).
Object.preventExtensions(oneProp);
assert(Object.isFrozen(oneProp) === false);

// ...如果删除了这个属性,则它会成为一个冻结对象.
delete oneProp.p;
assert(Object.isFrozen(oneProp) === true);

// 一个不可扩展的对象,拥有一个不可写但可配置的属性,则它仍然是非冻结的.
var nonWritable = { e: "plep" };
Object.preventExtensions(nonWritable);
Object.defineProperty(nonWritable, "e", { writable: false }); // 变得不可写
assert(Object.isFrozen(nonWritable) === false);

// 把这个属性改为不可配置,会让这个对象成为冻结对象.
Object.defineProperty(nonWritable, "e", { configurable: false }); // 变得不可配置
assert(Object.isFrozen(nonWritable) === true);

// 一个不可扩展的对象,拥有一个不可配置但可写的属性,则它仍然是非冻结的.
var nonConfigurable = { release: "the kraken!" };
Object.preventExtensions(nonConfigurable);
Object.defineProperty(nonConfigurable, "release", { configurable: false });
assert(Object.isFrozen(nonConfigurable) === false);

// 把这个属性改为不可写,会让这个对象成为冻结对象.
Object.defineProperty(nonConfigurable, "release", { writable: false });
assert(Object.isFrozen(nonConfigurable) === true);

// 一个不可扩展的对象,值拥有一个访问器属性,则它仍然是非冻结的.
var accessor = { get food() { return "yum"; } };
Object.preventExtensions(accessor);
assert(Object.isFrozen(accessor) === false);

// ...但把这个属性改为不可配置,会让这个对象成为冻结对象.
Object.defineProperty(accessor, "food", { configurable: false });
assert(Object.isFrozen(accessor) === true);

// 使用Object.freeze是冻结一个对象最方便的方法.
var frozen = { 1: 81 };
assert(Object.isFrozen(frozen) === false);
Object.freeze(frozen);
assert(Object.isFrozen(frozen) === true);

// 一个冻结对象也是一个密封对象.
assert(Object.isSealed(frozen) === true);

// 当然,更是一个不可扩展的对象.
assert(Object.isExtensible(frozen) === false);
```

####Object.isSealed  

Object.isSealed() 方法判断一个对象是否是密封的（sealed）。

语法如下：

```
Object.isSealed(obj)
```
- 描述：如果这个对象是密封的，则返回 true，否则返回 false。密封对象是指那些不可 扩展 的，且所有自身属性都不可配置的（non-configurable）对象。

例子：

```
// 新建的对象默认不是密封的.
var empty = {};
Object.isSealed(empty); // === false

// 如果你把一个空对象变的不可扩展,则它同时也会变成个密封对象.
Object.preventExtensions(empty);
Object.isSealed(empty); // === true

// 但如果这个对象不是空对象,则它不会变成密封对象,因为密封对象的所有自身属性必须是不可配置的.
var hasProp = { fee: "fie foe fum" };
Object.preventExtensions(hasProp);
Object.isSealed(hasProp); // === false

// 如果把这个属性变的不可配置,则这个对象也就成了密封对象.
Object.defineProperty(hasProp, "fee", { configurable: false });
Object.isSealed(hasProp); // === true

// 最简单的方法来生成一个密封对象,当然是使用Object.seal.
var sealed = {};
Object.seal(sealed);
Object.isSealed(sealed); // === true

// 一个密封对象同时也是不可扩展的.
Object.isExtensible(sealed); // === false

// 一个密封对象也可以是一个冻结对象,但不是必须的.
Object.isFrozen(sealed); // === true ，所有的属性都是不可写的
var s2 = Object.seal({ p: 3 });
Object.isFrozen(s2); // === false， 属性"p"可写

var s3 = Object.seal({ get p() { return 0; } });
Object.isFrozen(s3); // === true ，访问器属性不考虑可写不可写,只考虑是否可配置
```

####Object.observe 

Object.observe() 方法用于异步的监视一个对象的修改。当对象属性被修改时，方法的回调函数会提供一个有序的修改流。

语法如下：

```
Object.observe(obj, callback)
```
- 参数：obj为被监视的对象，callback为当对象被修改时触发的回调，其参数为changes。其中包含：
	- name: 被修改的属性名称。
	- object: 修改后该对象的值。
	- type: 表示对该对象做了何种类型的修改，可能的值为"add", "update", or "delete"。
	- oldValue: 对象修改前的值。该值只在"update"与"delete"有效。
	
- 描述：callback 函数会在对象被改变时被调用，其参数为一个包含所有修改信息的有序的数组对象。

例子 - 打印出三种不同操作类型的日志

```
var obj = {
  foo: 0,
  bar: 1
};

Object.observe(obj, function(changes) {
  console.log(changes);
});

obj.baz = 2;
// [{name: 'baz', object: <obj>, type: 'add'}]

obj.foo = 'hello';
// [{name: 'foo', object: <obj>, type: 'update', oldValue: 0}]

delete obj.baz;
// [{name: 'baz', object: <obj>, type: 'delete', oldValue: 2}]
```

例子 - 数据绑定

```
// 一个数据模型
var user = {
  id: 0,
  name: 'Brendan Eich',
  title: 'Mr.'
};

// 创建用户的greeting
function updateGreeting() {
  user.greeting = 'Hello, ' + user.title + ' ' + user.name + '!';
}
updateGreeting();

Object.observe(user, function(changes) {
  changes.forEach(function(change) {
    // 当name或title属性改变时, 更新greeting
    if (change.name === 'name' || change.name === 'title') {
      updateGreeting();
    }
  });
});
```

![被废弃的特性](http://i4.tietuku.com/6e90f0fef70ec952.png)


###参考文献

[MDN object对象](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object)

[创建对象的方法](http://www.jb51.net/article/77676.htm)

[object.is方法详解](http://www.zuojj.com/archives/1211.html)


