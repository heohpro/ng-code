#Angular2使用规范
##目录
- 基本流程
	1. 引入Angular2预定义类型
	2. 实现一个Angular2组件
	3. 渲染组件到DOM
- 模块功能
	- 注解/Annotation
	- bootstrap启动流程
	- 模板
	- 指令
	- 数据绑定
	- 逻辑控制
	- 样式
	- 组件封装
	- 表单指令
	- Service
	- 路由


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

##模块功能
###注解/Annotation
ES6规范里没有装饰器，因此利用`traceur`的一个实验特性`注解`。给一个类添加注解，相当于设置这个类的`annotations`属性。

```
//注解写法
@Component({selector:"ez-app"})
class EzApp{...}
```
相当于

```
class EzApp{...}
EzApp.annotations = [new Component({selector:"ez-app"})];
```
很显然，注解可以看做编译器（traceur）层面的`语法糖`，但和python的`装饰器`不同， 注解在编译时仅仅被放在`annotation`里，编译器并不进行解释展开，这个解释的工作是 Angular2完成的：
![注解运行流程](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/annotation.jpg)
据称，注解的功能就是Angular2团队向traceur团队提出的，这不是traceur的默认选项， 因此你看到，我们配置systemjs在使用traceur模块时`打开注解`：

```
System.config({
  map:{traceur:"lib/traceur"},
  traceurOptions: {annotations: true}
});
```
###bootstrap启动流程
相较于Angular1.x来说，Angular2的bootstrap发生了一些变化：
####以组件为核心
- 在Angular1.x中，bootstrap是围绕DOM元素展开的，无论你使用ng-app还是手动执行bootstrap() 函数，自举过程是建立在DOM之上的。

- 而在Angular2中，bootstrap是围绕组件开始的，你定义一个组件，然后启动它。如果没有一个组件， 你甚至都没有办法使用Angular2！

####支持多种渲染引擎
以组件为核心，意味着Angular2在内核已经隔离了对DOM的依赖，DOM仅仅作为一种`可选`的渲染引擎，可以支持服务器端、移动端。

![多种渲染引擎](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/render-arch.jpg)

###模板
组件的`View注解`利用属性`template模板`用来声明组件的`外观`，兼容HTML语法。有两种方式为组件制定渲染模板
1. 内联模板，用`template属性`直接指定`内联模板`。

```
@View({
    template : `<h1>hello</h1>
                <div>...</div>`
})
```

2. 外部模板，将模板写入一个文件，然后用`templateUrl`引用`外部模板`。

```
@View({
    templateUrl : "tpl.html"
})
```

###指令
除了在`View注解`中使用HTML模板，也可以直接使用自定义的组件。令渲染模板和模型视图更加容易区分，也是的模板的语义性更强。相比于Angular1.x，指令的使用更加直接和简单。
![指令渲染过程](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/component-template.jpg)
在自定义组件之前，必须得组件的`View注解`中必须通过`directive属性`声明所有使用的组件。

```
@View({
    directives : [EzComp],
    template : "<ez-comp></ez-comp>"
})
```
###数据绑定
- 文本插值`{{model}}`:
![文本插值](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/intepolate.jpg)

	和Angular1一样，在模板中可以使用`{{表达式}}`的方式来绑定组建模型中的model。

- 绑定属性`[property]`:
![绑定属性](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/prop-bind.jpg)
	
	在模板中可以用`中括号`将HTML元素或组件的`属性`绑定到某个`表达式`上，进行双向数据绑定。可以等价的用`bind-`来进行数据绑定。如：
	
	```
	@View({template:`<h1 bind-text-content="title"></h1>`})
	```
	需要注意，属性的值默认为是以`表达式`的形式进行绑定，如果希望绑定一个常量字符串，需要给字符串加引号或者去掉中括号。
- 监听事件`(event)`
![监听事件](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/event-bind.jpg)	
	在模板中添加监听事件时，使用`(event)`的形式绑定事件到表达式上。可以用等价的书写方式，在事件名称前加一个`on-`前缀。	
	
	```
	@View({template : `<h1 on-click="onClick()">HELLO</h1>`})

	```
- 局部变量`#var`

	在模板中不同元素需要相互调用的时候，可以将需要调用的元素设置为`局部变量`。添加一个以`#`或者`var-`开始的属性，后续部分为变量名，则这个变量为对应元素的实例。
	如将下面的h1元素定义为v_h1，则这个变量指向相对应的DOM对象，可以在其他地方调用该方法和属性。	
	
	```
	@View({
    template : `
        <h1 #v_h1="">hello</h1>
        <button (click)="#v_h1.textContent = 'HELLO'">test</button>
    `
	})
	```
	如果在一个组件上定义局部变量，则其对应该组件的实例。
	
	```
	@View({
    	directives:[EzCalc],
    	template : "<ez-calc #c=""></ez-calc>"
	})
	```

###逻辑控制
逻辑控制主要分为三个部分：`NgIf条件逻辑`、`NgSwitch分支逻辑`和`NgFor循环逻辑`。
####NgIf条件逻辑
NgIf是Angular2预置的指令，使用之前需要：
1. 从angular2库中引入NgIf类型定义。
2. 从组件的`View注解`中通过`directives`声明对指令的引用。

```
//引入NgIf类型
import {Component,View,bootstrap,NgIf} from "angular2/angular2";
@View({
      directives:[NgIf],
      template :
        ` <img [src]="banner" template="ng-if trial==false"> `
})
```

和Angular1.x的`ng-if`一样，评估表达式的值是否为真，来决定是否渲染`template`。有以下两种语法可以使用

```
//使用template attribute
<img src="ad.jpg" template="ng-if tiral==true">
//使用*前缀
<img src="ad.jpg" *ng-if="tiral==true">
```

####NgSwitch分支逻辑
NgSwitch是可以应用到任何HTML元素上，通过评估元素的ngSwitch`值，根据值切换应用的`template`内容。一般和`NgSwitchWhen`和`NgSwitchDefault`搭配使用。

同`NgIf`一样，使用前需要引入并在`directives`中声明。

#####NgSwitchWhen&NgSwitch
`NgSwitchWhen`指令和`NgSwitchDefault`指令必须应用在`NgSwitch`指令上的子`template`上，当有`NgSwitchWhen`匹配时，则显示这个的template上的内容，否则，则显示`NgSwitchDefault`相匹配。

```
<any [ng-switch]="...">
    <!--与变量比较-->
    <template [ng-switch-when]="variable">...</template>
    <!--与常量比较-->
    <template ng-switch-when="constant">...</template>
    <!--如果没有匹配到-->
    <template ng-switch-default="">...</template>
</any>
```

####NgFor循环逻辑
同Angular1.x的`ng-repeat指令`，`NgFor`循环逻辑建立可遍历的动态模板。其中`NgFor`指令应用在`template`上，对`ngForOf`属性指定的数据集中的每一项进行实例化。可以在每一项中声明一个局部变量，可以在模板内引用。

同`NgIf`一样，使用前需要引入并在`directives`中声明。

#####数据项和数据索引项
```
<template ng-for="" [ng-for-of]="items" #item="" #i="index">
    <li>[{{i+1}}] {{item}}</li>
</template>
```
其中item为循环中一个声明的数据项，index为该数据项的索引。

#####语法糖
Angular2也为`NgFor`提供了两种语法：

```
//使用template attribute
<any template="ng-for #item of items;#i=index">...</any>
//使用*前缀
<any *ng-for="#item of items;#i=index">...</any>
```
推荐使用`*ng-for`的写法。

###样式

有两种方式为组件设置CSS样式，其中包括`内联组件`和`外部样式`。

- 内联样式(加在`View注解`的`styles`属性设置`内联样式`)：

	```
@View({
    styles:[`
        h1{background:#4dba6c;color:#fff}     
    `]
})
```
- 外部样式(将外部url引入`View注解`的`styleUrls`中)：

	```
@View({
    styleUrls:["ez-greeting.css"]
})
```
同时，在组件渲染上，有三种方式将模板渲染到DOM： //TODO


###组件封装
主要包括属性声明（暴露成员变量）和事件声明（暴露事件源）。
![组件封装](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/event.jpg)
####属性声明

属性是组件封装后的暴露出来的调用接口，调用组件时通过设置属性定义组件的不同行为。可以在`Component注解`的`properties`属性中声明组件的成员变量。

- 被调用的组件:将name和country暴露为同名属性。

	```
	//EzCard 
@Component({
    properties:["name","country"]
})
```
- 父级组件：用中括号语法来设置子组件的属性。

	```
//EzApp
@View({
    directives : [EzCard],
    template : "<ez-card [name]="'雷锋'" [country]="'中国'"></ez-card>"
})
	```
	
####事件声明

事件由组件内部传出，触发外部某个动作。
首先定义一个`事件源EventEmitter`，然后通过`Component注解`的`events`接口包起来。

- 被调用的组件:在EventEmitter中注册回调。

	```
//EzCard
@Component({
    events:["change"]
})
class EzCard{
    constructor(){
        this.change = new EventEmitter();
    }
}
```
- 父级组件：用小括号直接挂载监听函数。

	```
//EzApp
@View({
    template : "<ez-card (change)="onChange()"></ez-card>"
})
	```
	
每次子模块触发change事件时，父模块的onChange()方法都被调用。
	
###表单指令
Angular2中，类似`NgForm`和`NgController`指令都包含在数组变量`formDirectives`中，因此使用表单时，只需要声明`formDirectives`就可以使用这些指令了，下面详细解释。

```
//angular2/ts/src/forms/directives.ts
export const formDirectives = CONST_EXPR([
  NgControlName,
  NgControlGroup,
 
  NgFormControl,
  NgModel,
  NgFormModel,
  NgForm,
 
  NgSelectOption,
  DefaultValueAccessor,
  CheckboxControlValueAccessor,
  SelectControlValueAccessor,
 
  NgRequiredValidator
]);
```

####NgForm-表单指令

- `NgForm`指令为form建立一个`控件组`对象，作为空间的容器；`NgControllerName`指令建立一个`控件`独享，并加入`NgForm`指令建立的控件组中。

- 通过`#`符号，可以创建一个控件组对象的局部变量`f`，它的`value`属性是一个JSON对象，键对应input元素上的`ng-control`属性，值对应input的值。

![ngForm](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/ngform.jpg)


####NgControlName-命名控件指令
- `NgControlName`指令的选择符是`[ng-control]`，绑定在input等DOM对象上，创建一个`控件`对象。如下就创建了两个`Control`对象。

	```
<form #f="form">
    <input type="text" ng-control="user">
    <input type="password" ng-control="pass">
</form>
```

- 除了ng-control外，还可以通过`ngModel`实现模型和表单的双向绑定。

	```
<form>
    <input type="text" ng-control="user" [(ng-model)]="data.user">
    <input type="password" ng-control="pass" [(ng-model)]="data.pass">
</form>`
```

- 同时，`ngModel`不仅是属性也是时间，因此以下两种写法都可以。

	```
<input type="text" ng-control="user" [(ng-model)]="data.user">
//等价于
<input type="text" ng-control="user" [ng-model]="data.user" (ng-model)="data.user">
```
![双向数据绑定](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/ngform2.jpg)

####NgControlGroup-命名控件组
- NgControlGroup指令的选择符是[ng-control-group]，如果模板中的某个元素具有这个属性， Angular2框架将自动创建一个控件组对象，并将这个对象以指定的名称与DOM对象绑定。
- 具体创建的数据结构如下：
![命名控件组](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/ngcg.jpg)

####NgFormControl-绑定已有控件对象
- 与`NgControlName`指令不同，`NgFormControl`将已有的`控件/Control`对象绑定到DOM元素 上。当需要对输入的值进行初始化时，可以使用`NgFormControl`指令。。
- 首先在View上将DOM元素进行绑定，然后在构造器里创建`Control`对象。：

	```
@View({
    //将输入元素绑定到已经创建的控件对象上
    template : `<input type="text" [ng-form-control]="movie">`
})
class EzComp{
    constructor(){
        //创建控件对象
        this.movie = new Control("Matrix II - Reload");
    }
}
```

####NgFormModel-绑定已有控件组
- `NgFormModel`指令为控件提供容器，将已有的控件组绑定到DOM对象上。

	```
@View({
    template : `
        <!--绑定控件组与控件对象-->
        <div [ng-form-model]="controls">
            <input type="text" ng-control="name">
            <input type="text" ng-control="age">
        </div>`
})
class EzComp{
    constructor(){
        //创建控件组及控件对象
        this.controls = new ControlGroup({
            name :new Control("Jason"),
            age : new Control("45")
        });
    }
}
```

###Service
同Angular1.x，服务用来封装`可复用`的功能性代码。例如将HTTP请求封装在Service中，然后在不同的组件中只调用`HTTP`服务的`API接口`。
![组件调用服务](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/service.jpg)
通常，定义一个类，就可以把它当做一个服务。

```
class EzAlgo{
    add(a,b){return a+b;}
    sub(a,b){return a-b;}
}
```

例如下面的调用：

```
class EzApp{
            constructor(){
            	this.a = 37;
                this.b = 128;
                //实例化服务对象
                this.algo = new EzAlgo();
            }
            add(){
            	var a = +this.a,
                	b = +this.b;
            	return this.algo.add(a,b);
            }
        }
```
在`EzApp`中，在构造函数中实例化了一个`EzAlgo`对象，我们可以直接使用`注入器Injector`进行依赖注入。

首先父组件使用`Component注解`的`appInjector`来声明其依赖于该服务，并在构造函数的参数表中使用`Inject注解`声明注入点，然后就获得了服务的实例。

如果是个复杂的服务，如下面的依赖关系，则有两种实现方式。
![依赖关系](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/httpsvc.jpg)

1. 使用new进行实例化
	
	```
	var xhrbe = new XHRBackend(BrowserXHR);
var options = new BaseRequestOptions();
var http = new Http(xhrbe,options);

	```
2. 使用注入器/Injector
	
	```
	@Component({
    appInjector : [
      bind(BrowserXHR).toValue(BrowserXHR),
      XHRBackend,
      BaseRequestOptions,
      Http
    ]
})
	```
	bind(BrowserXHR).toValue(BrowserXHR)的意思是，如果需要注入BrowserXHR类型的变量，注入 这个类本身而非其实例。
	
###路由

####引入依赖
1. 引入路由文件：

	Angular2的路由文件单独打包在`router.dev.js`中，首先引入这个包。
	
	```
	<script type="text/javascript" src="lib/router.dev.js"></script>
	```
2. 引入路由相关的预定义类型

	Angular2的路由模块名为angular2/router，我们从这里引入常用类型：
	
	```
	import {LocationStrategy,Router,RouterOutlet,routerInjectables} from "angular2/router";
	```
	
3.  声明路由相关依赖类型

	在启动组件时，我们需要声明路由相关的依赖类型（即变量：routerInjectables），以便 根注入器可以解析对这些类型的请求：

	
	```
	bootstrap(EzApp,[routerInjectables]);
	```
	
####配置路由
1. 配置路由：

	为组件注入Router对象并通过config()方法配置路由：
	
	```
router.config([
  {path:"/video", component:EzVideo},
  {path:"/music", component:EzMusic}])
	```
	上面的代码中，配置了两条路由：
	- 如果用户请求路径为`/video`，那么将在路由出口中激活组件EzVideo
	- 如果用户请求路径为`/music`，那么将在路由出口中激活组件EzMusic
2. 设置路由出口

	路由出口是组件激活的地方，使用RouterOutlet指令在组件模板中声明出口：
	
	```
@View({
    directives:[RouterOutlet],
    template : `<router-outlet></router-outlet>`
})
class EzApp{...}
	```
	
3.  执行路由

	使用`Router`的`navigate()`方法可以执行指定路径的路由，在示例中，当用户点击 时我们调用这个方法进行路由：

	
	```
	@View({
    template : `
        <span (click)="router.navigate('/video')">video</span> | 
        <span (click)="router.navigate('/music')">music</span>
        <router-outlet></router-outlet>`
})
	```

	我们向navigate()方法传入的路径，就是我们通过config()方法配置的路径。这样， Router就根据这个路径，找到匹配的组件，在RouterOutlet上进行激活。