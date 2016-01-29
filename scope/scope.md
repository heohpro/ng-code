#作用域继承

在Angular，作用域继承主要分成两种。大部分都是子作用域原型继承于父作用域。另外一种情况是在自定义指令中，选择`scope:{...}`的形式，创建独立作用域来创建可复用的组件。

作用域继承一般是很简单的，但有时如果我们需要在自作用域中创建基本类型数据（例如数字、字符串等）来进行双向数据绑定时，往往会出现问题，因为Angular的作用域继承由于javascript原型链的特性，会在子作用域创建属性，而不会影响父作用域上的属性，往往会导致数据不同步等问题。

下面介绍一下javascript中的原型继承原理和Angular中的作用域继承。
###JavaScript原型继承

首先，我们要对JavaScript的原型继承有个良好的认知，这很重要，如果你有服务端编程的背景，更是如此。所以让我们先回顾一下原型继承的原理。

假设父级作用域有以下属性aString、aNumber、anArray、anObject 和 aFunction。如果子作用域原型继承于父作用域，我们有：

![pic1](https://github.com/hexiaoming/ng-code/blob/master/images/scope/pic1.png?raw=true)

当我们试图从子作用域中访问父作用域上定义的属性，JavaScript会先在子作用域上查询该属性，如果没有找到该属性，再访问父级作用域并查询该属性。（如果在父作用域中依旧没有找到这个属性，JavaScript会继续顺着原型链往上查找... 直到根作用域）。因此，以下均为true:

```
childScope.aString === 'parent string'
childScope.anArray[1] === 20
childScope.anObject.property1 === 'parent prop1'
childScope.aFunction() === 'parent output'
```
假设我们接下来进行以下操作：

```
childScope.aString = 'child string';
```
原型链并未被查询，而子作用域中新增了一个 aString 属性。**这个新的属性隐藏/遮蔽了父作用域的同名属性。**当我们下面讨论到ng-repeat指令和ng-include指令时，这特性会变得非常重要。

![pic2](https://github.com/hexiaoming/ng-code/blob/master/images/scope/pic2.png?raw=true)

接下来假设我们执行：

```
childScope.anArray[1] = '22'
childScope.anObject.property1 = 'child prop1'
```
因为在子作用域中没有找到 anArray 和 anObject 对象，所以原型链被查询了。在父作用域中被找到这两个对象，所以属性值被更新到了原始的对象上。子作用域上没有添加新的属性，也没有创建新的对象。（注意，在JavaScript中数组和函数都是对象）。

![pic3](https://github.com/hexiaoming/ng-code/blob/master/images/scope/pic3.png?raw=true)

接着，假设我们这么做：

```
childScope.anArray = [100, 555]
childScope.anObject = { name: 'Mark', country: 'USA' }
```
原形链并未被访问，并且子作用域获得了两个新的对象属性，这两个属性也会遮蔽父作用域上的同名属性。

![pic4](https://github.com/hexiaoming/ng-code/blob/master/images/scope/pic4.png?raw=true)

顺便提一下：

如果我们读取childScope.propertyX，并且子作用域有 propertyX 属性，那么原型链将不会被访问。
如果我们设置childScope.propertyX，那么原型链也不会被访问。
最后一种情况：

```
delete childScope.anArray
childScope.anArray[1] === 22  // true
```
我们先删除子作用域的属性，然后当我们试图再次访问该属性，此时原型链会被访问。

![pic5](https://github.com/hexiaoming/ng-code/blob/master/images/scope/pic5.png?raw=true)

###Angular中的作用域继承

在Angular中，一般是在使用指令的时候涉及到作用域的继承。如开头提到的那样，主要分为两个部分：

- 创建新的作用域，并原型继承父级的作用域。如原生的`ng-repeat`、ng-include、ng-switch、ng-view、ng-controller等，`scope:true`，并设置了`transclude:true`。
- 创建新的作用于不进行原型继承。如自定义需要可复用的指令：`scope:{...}`。

**需要注意一点，默认情况下指令设置为`scope:false`,不会创建作用域**

在作用域继承时，父作用域在子作用域的原型链上，另外，Angular中定义了如下属性进行作用域的选择：

- scope.$parent指向scope的父作用域；
- scope.$$childHead指向scope的第一个子作用域；
- scope.$$childTail指向scope的最后一个子作用域；
- scope.$$nextSibling指向scope的下一个相邻作用域；
- scope.$$prevSibling指向scope的上一个相邻作用域； 

本文主要分析各个指令之间作用域继承的关系，对具体的controller部分不详细展开。

####ng-include

假设我们的控制器中有：

```
$scope.myPrimitive = 50;
$scope.myObject    = {aNumber: 11};
```
而且在我们的HTML中：

```
<script type="text/ng-template" id="/tpl1.html">
    <input ng-model="myPrimitive">
</script>
<div ng-include src="'/tpl1.html'"></div>
<script type="text/ng-template" id="/tpl2.html">
    <input ng-model="myObject.aNumber">
</script>
<div ng-include src="'/tpl2.html'"></div>
```
每一个ng-include指令都生成一个新的子作用域，这些子作用域都原型继承于其父作用域。

![pic1](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi1.png)

在第一个输入框中输入77，子作用域将会得到一个新的myPrimitive属性，该属性会遮蔽了父作用域的同名属性。这可能不是你想要的。

![pic2](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi2.png)

在第二个输入框中输入99不会新建一个子作用域属性。因为tpl2.html绑定的数据是一个对象属性。当ngModel指令查询该对象，原型继承起到了作用，最终在父作用域中查找到该对象。

![pic3](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi3.png)

如果我们不想将我们的数据从基本类型改为对象，我们可以用$parent变量重写第一个模版：

```
<input ng-model="$parent.myPrimitive">
```
在该输入框中输入22不会生成一个新的子作用域属性。现在，这个模型是绑定在父级作用域的一个属性上（因为$parent是子作用域上指向父作用域的属性值）。

![pic4](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi4.png)

对于所有的作用域（无论是否原型继承），Angular总会通过$parent、$$childHead`和`$$childTail记录下父-子关系（即一种层级关系）。以上的图表并没有展示这些属性值。

对于一些不涉及表单元素的情况，另一种解决方法是在父级作用域中定义一个函数用来修改基本类型数值。然后保证其子作用域都调用该函数，由于原型继承，其子作用域都能够访问的该函数。比如：

```
// in the parent scope
$scope.setMyPrimitive = function(value) {
    $scope.myPrimitive = value;
}
```

这里有个[示例fiddle](http://jsfiddle.net/mrajcok/jNxyE/)运用这类”父级函数“方法。

####ng-switch

ng-switch指令的作用域继承的运行原理就类似于ng-include指令。所以如果你需要对父级作用域中的一个基本类型值进行双向版定，你可以使用$parent，或者将数据模型改成对象的形式，然后绑定该对象上的属性。这可以避免子作用域遮蔽到了父作用域上的属性。

####ng-repeat

ng-repeat指令的运行原理有点不一样。假设我们控制器中有：

```
$scope.myArrayOfPrimitives = [ 11, 22 ];
$scope.myArrayOfObjects    = [{num: 101}, {num: 202}];
```

而且我们的HMTL中：

```
<ul><li ng-repeat="num in myArrayOfPrimitives">
       <input ng-model="num"></input>
    </li>
</ul>
<ul><li ng-repeat="obj in myArrayOfObjects">
       <input ng-model="obj.num"></input>
    </li>
</ul>
```

每次迭代，ng-repeat指令都会创建一个新的作用域，该作用会原型继承于其父级作用域，**但是同时该指令会给这个新作用域的一个新的属性分配本次迭代对应数值。**（这个属性的名称就是循环变量的名字）。以下就Angular源码中ng-repeat具体实现：

```
childScope = scope.$new(); // child scope prototypically inherits from parent scope ...     
childScope[valueIdent] = value; // creates a new childScope property
```

如果迭代项为基本类型，实质上把该值的拷贝分配给了子作用域新的属性。改变这个属性值（即子作用域的属性num）不会改变父作用域引用的数组。所以在上述第一个ng-repeat指令中，每个子作用域都获得一个独立于myArrayOfPrimitives数组的`num`属性：

![pi5](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi5.png)

这个ng-repeat指令不会如你期望搬工作。在Angular1.0.2及之前版本中，在输入框中输入，会改变灰色框框内的值，即子作用域的属性值。在Angular 1.0.3+版本，在文本框中输入不会有任何效果。我们想要的是，输入的值能改变myArrayOfPrimitives数组，而不是子作用域的属性值。为了实现这一点，我们需要将模型改成一个包含对象的数组。

所以，如果迭代元素是一个对象，那么分配到子作用域上的就是一个对原始对象的引用（而不是拷贝）。改变子作用域的属性值便会同时改变父级作用域引用的对象。所以在上述第二个ng-repeat指令中，我们有：

![pi6](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi6.png)
 
（我用灰色标记其中一条线，以便清晰展现它的指向）

这将如期工作。在文本框中的输入将改变灰色框框中的值，这将同时反映到子作用域和父级作用域中

####ng-controller
使用ng-controller与ng-include一样也是创建子作用域，会从父级controller创建的作用域进行原型继承。但是，利用原型继承来使父子controller共享数据是一个糟糕的办法，controllers之间应该使用 service进行数据共享。

####directives
指令的scope有三种情况：

- 默认值为`scope:false`。指令直接使用原有的作用域，在指令模板里可以直接使用父作用域中的变量和函数。
- `scope:true`：创建一个原型继承父作用域的新作用域。如果多个指令（在同一个DOM元素上）请求新的作用域，那么只会创建一个作用域。因为涉及到原型继承，就像ng-include和ng-switch，所以我们要谨慎对待父级作用域基本类型数据的双向绑定和子作用域遮掩父级作用域属性的问题。
- `scope:{...}`：创建一个不原型继承父作用域的独立作用域。当创建可复用组件时可以这样设置，因为这指令不会意外地读取或修改父级作用域。然而，有些指令通常需要访问父作用域的数据。设置对象是用来配置父作用域和封闭作用域之间的双向绑定（使用=）或单向绑定（使用@）。这里也可以使用&绑定父作用域上的表达式。所以，这些配置都会将来自父作用域的数据创建到本地作用域属性中。	
	1. = or =attr “Isolate”作用域的属性与父作用域的属性进行双向绑定，任何一方的修改均影响到对方，这是最常用的方式；
	2. @ or @attr “Isolate”作用域的属性与父作用域的属性进行单向绑定，即“Isolate”作用域只能读取父作用域的值，并且该值永远的String类型；
	3. & or &attr “Isolate”作用域把父作用域的属性包装成一个函数，从而以函数的方式读写父作用域的属性，包装方法是$parse

- `transclude: true` 指令新建一个`trancluded`的子作用域，并从父作用域进行原型继承。如果独立作用域存在的话，transclude作用域与独立作用域是相邻关系。他们的`$parent`属性指向同一个父作用域。独立作用于的`$$nextSibing`属性指向`transcluded`作用域。

![pi8](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi8.png)

其中，封闭作用域的`__proto__`引用的是一个Scope对象。封闭作用域的$parent指向父作用域，所以，虽然该作用域保持封闭而且不会原型继承于父作用域，但它依旧是一个子作用域。

如下代码中生成结构如图：

```
//html
<my-directive interpolated="{{parentProp1}} twowayBinding="parentProp2">
//scope
{ interpolatedProp: '@interpolated', twowayBindingProp: '=twowayBinding' }，
//link函数
scope.someIsolateProp = "I'm isolated"
```
![pi7](https://raw.githubusercontent.com/hexiaoming/ng-code/dffc37d1ae38f04db234b20a1b04ec57b3a308d3/images/scope/pi7.png)

###总结

1. 常规的原型继承的作用域 -- `ng-include`, `ng-switch`, `ng-controller`, 设置了`scope: true`的指令。
2. 普通的带原型继承的，并且有赋值行为的作用域 -- `ng-repeat`，`ng-repeat`为每一个迭代项创建一个普通的有原型继承的子作用域，但同时在子作用域中创建新属性存储迭代项；。
3. `Isolate`作用域 -- `scope: {...}`， 该作用域没有原型继承，但可以通过'=', '@', 和 '&'与父作用域通信。
3. `transclude`作用域 -- 设置了`transclude: true`的指令。这种作用域也是常规的原型继承，但它和任何封闭作用域是同级关系。

###引用

[深入浅出 AngularJS 作用域](https://www.zybuluo.com/lxjwlt/note/107324)

[Understanding Scopes](https://github.com/angular/angular.js/wiki/Understanding-Scopes)

