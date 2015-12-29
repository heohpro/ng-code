##基本流程
angular2创建应用主要分为三个部分。
###1.引入Angular2预定义类型
```
import {Component,View,bootstrap} from "angular2/angular2";
```
import是ES6的关键字，用来从模块中引入类型定义。在这里，我们从angular2模块库中引入了三个类型： Component类、View类和bootstrap函数。
###2.实现一个Angular2组件
首先定义一个类，然后给这个类添加`注解`。

```
@Component({selector:"ez-app"})
@View({template:"<h1>Hello,Angular2</h1>"})
class EzApp{}
```
- 其中，@Component和@View是给类EzApp附加的信息，叫做`注解/Annotation`。`@Component`通过`selector`属性，指定这个组件渲染到哪个DOM节点上，`@View`通过`template`属性，指定渲染的模板。
- `class`是ES6的关键字，用来定义一个类。

###3.渲染组件到DOM
要想将该组件渲染到DOM上，需要用`自举/bootstrap`函数，令`EzApp`组件渲染到DOM树上：

```
bootstrap(EzApp);
```
