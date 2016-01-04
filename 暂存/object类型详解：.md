##Object类型详解：

###介绍

Object构造函数为给定的值创建一个对象包装。如果给定的值是`null`或`undefined`，则会创建并返回一个空对象，否则，将创建一个与给定值对应类型的对象。

当以非构造函数形式被调用时，Object等同于new Object()。

###属性

- Object.length: 属性值为1。
- Object.prototype: 可以为所有的Object类型的对象添加属性。

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

####Object.assign

Object.assign方法可以把任意多个源对象所拥有的自身可枚举的属性拷贝给目标对象，然后返回目标对象。

语法如下：

```
Object.assign(target, ...sources)
```
- 参数：第一个参数为目标对象，后面为任意多个源对象。目标对象将被返回。
- 描述：该方法会拷贝源对象自身的并且可枚举的属性到目标对象上。对于访问器属性，则会执行访问器属性的getter函数，然后把得到的值拷贝给目标对象。如果想获得访问器属性本身，需使用`Object.getOwnPropertyDescriptor()`和`Object.defineProperties()`方法。

