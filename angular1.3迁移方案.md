#Angular1.3新特性

##介绍
本文介绍业务项目由Angular1.2.25到Angular1.3.5的迁移方案。着重在调研1.3版本的新特性以及制定改进性能的方案。AngularJS1.3显著的提升了性能，降低了内存消耗，提升了常规DOM的操作速度。
<!-- more -->

##升级目的
###AngularJS1.3新特性

- Browsers: 不再支持IE8
- input: 类型为date,time,datetime0local,month,weeek的input需要绑定一个Date对象
- $http: 在完成回调中增加xhr状态的文本
- Scope: 增加$watchGroup方法观察一组表达式
- injector: 禁用自动功能注释的严格模式(strict-DI)
- ngModelOptions: 自定义新的触发器
- $compile: 允许通过特殊类型属性的SVG和MathML模板
- FormController: 表单提交时，同时提交所有子空间的$viewValue值
- ngMessages: 介绍NgMessages的模块和指令
- ngTouch: 在ngSwipe里添加可选的ngSwipeDisableMouse属性来忽略鼠标事件
- $interpolate: 支持one-time bingding懒加载
- ngMock: 增加了支持URL匹配的共更能，支持mocha接口

###预期效果
&emsp;&emsp;Angular 1.3相比以前提高了性能，Angular团队专注于降低内存消耗，根据Jeff Cross和Brain Ford在Ng-Europe上的说法，这一版本较v1.2.0在DOM操作方面提升4.3倍，减少了73%的内存消耗，在$digest方面提升3.5被，减少了87%的内存消耗。



##实施方法及改造试点

###新特性实施方式

####[One-time bindings](https://docs.angularjs.org/guide/expression#one-time-binding)

&emsp;&emsp;在1.3之前，Angualr的数据绑定是自动保持UI同步更新。需要建立watcher数组来时刻监视着所有绑定过的数据。比较耗性能。有些项目场景只需要单次绑定，可以减少性能开销。当引进了One-time bingdings之后，通过”::”前缀表达式进行便可以进行单向数据绑定。

&emsp;&emsp;当一次性数据绑定表达式可以在数据稳定之后，不需要再$digest循环中重新计算。解决了之前提到的由监听器太多带来的性能问题。

&emsp;&emsp;一次性数据绑定的出现解决了AngularJS中饱受诟病的性能问题，官方版本原生支持也使我们不需要在使用bindonce这样的第三方模块。可以期待在Angular2中，带来性能更好的数据绑定策略。

原理：

1. :”开头的表达式在一个$digest循环中被检测出有变化时，将当前值存到变量V。
2. 如果V不是undefined的话，将该表达式的状态标为稳定并在退出$digest循环时将该表达式的watch对象放入注销的队列中。
3. 继续进行$digest循环。
4. 当$digest循环完成之后，开始处理注销的队列。对每个在注销队列中的watch对象，如果它当前的值不是undefined，则撤销这个watch对象。否则就继续按步骤1进行$digest循环。

用法:

```
	<p>Hello {{::name}}!</p>	
```
[demo(内网访问)](http://jsbin.sankuai.com/xuq/1/edit?html,js,output)
[demo(公网访问)](https://jsbin.com/zowafameli/edit?html,js,output)

应用场景：

&emsp;&emsp;将只需要一次绑定的变量（展示字段或`ng-repeat`遍历）有双向绑定改为one-time bindings，提升`$digest`性能。

####[ngAria](https://docs.angularjs.org/api/ngAria)
&emsp;&emsp;ngAria在ngModel的前提下提供了一些新属性来帮助制作AngularJS自定义组件的新模块。

应用场景：

&emsp;&emsp;在自定义组件中或者表单组件中使用，增加多种筛选条件。


####[ng-messages](https://docs.angularjs.org/api/ngMessages)
&emsp;&emsp;在Angular再起版本，为了显示错误信息，表单验证错错误信息的显示结合和ng-if指令和大量的布尔逻辑。1.3引入了ng-messages模块来处理复杂的验证错误指令，将解决一个错误在另一个错误之前出现这一情况的复杂性。

用法:

```
<input type="text" placeholder="测试" name="name" ng-model="username.name" ng-minlength=3 ng-maxlength=20 required />
	<div ng-messages="myForm.name.$error" ng-messages-multiple>
    <div ng-message="required">必填项</div>
    <div ng-message="email">邮件格式不对</div>
    <div ng-message="minlength">字符太短小于3</div>
    <div ng-message="maxlength">字符太长大于20</div>
</div> 
```
&emsp;&emsp;可以看出，其实ng是通过$error来监视模型的变化，因为$error中包含了错误的详细信息，同时，如果我们的应用场景中如果同时，有好几处错误，那么，上面代码按照ng-message的顺序只会显示一条错误信息，如果我们需要全部显示出来只需要添加 ng-messages-multiple。

&emsp;&emsp;如果想复用的话，可以将验证信息保存到一个独立的Html静态页面中，然后使用ng-messages-include引入即可。
![demo](http://images.cnitblog.com/blog/360406/201410/202329180586886.gif)

应用场景：

&emsp;&emsp;在表单验证中使用，来处理提示的优先级问题。

####[ngModelOptions](https://docs.angularjs.org/api/ng/directive/ngModelOptions)
&emsp;&emsp;在双向数据绑定中，可以是绑定模块行为更容易自定义。其中有各种参数：

1. debouncing:去抖，单位是毫秒，可以延迟进行双向绑定。
2. update-on-blur:等input失焦之后再进行更新。
3. getterSetter:布尔值决定是否进行getter或setter操作。
用法：

```
	<input type="text" name="userName"
			ng-model="user.name"
            ng-model-options="{ debounce: 1000 }" />
```
[demo(内网访问)](http://jsbin.sankuai.com/xuq/edit?html,output)
[demo(外网访问)](https://jsbin.com/zowafameli/1/edit?html,js,output)

应用场景：

&emsp;&emsp;在一些所见即所得的双向数据绑定中使用，改善用户体验。

####[Strict DI](https://docs.angularjs.org/error/$injector/strictdi)
&emsp;&emsp;寻找应用位置的新选项。

###待改造TODO
####更新需要one-time binding的元素
&emsp;&emsp;现在之前所有的ng-model都是双向数据绑定，可以把所有需要展示不会变动的部分使用一次绑定，提升性能。

####将表单验证改为ng-messages结构
&emsp;&emsp;多重表单验证可以改为依赖ng-messages的结构


###迁移方案
- AngularJS1.2.25 ==> AngularJS1.3.5
- angular-animate(v1.2.25) ==> angular-animate(v1.3.5)

##成果

###性能前后对比

&emsp;&emsp;本次检测利用Chrome调试器中的Timeline工具，检测项目运行中各项性能的运行时间，从而对比版本更新后性能情况。版本为 48.0.2544.0 canary版，以避免第三方插件对实验结果有影响。 
####Angular版本：1.2.25
![1.2.25test](https://raw.githubusercontent.com/xgfe/blog/master/source/uploads/angular1.2.25.png)
####Angular版本：1.3.5
![1.3.5test](https://raw.githubusercontent.com/xgfe/blog/master/source/uploads/angular1.3.5.png)

&emsp;&emsp;为了避免偶然情况，两种AngularJS版本各运行10次，取结果的平均值作为参考值。具体数据如下：
![runTime from 1.2 to 1.3](https://raw.githubusercontent.com/xgfe/blog/master/source/uploads/balance.png =1000x250)
&emsp;&emsp;由于loading时间和网络加载延迟有关，故不作判断，比较scripting(脚本执行时间)、rendering(DOM渲染时间)、painting(页面绘制时间)三个部分。

&emsp;&emsp;由上面的表格可以得知，Angular1.3较1.2的脚本执行时间基本相同，但在DOM渲染部分和页面重绘部分要比1.2版本性能方面有显著提升，而且1.3版本的这两部分也比较稳定，与预期效果里提升DOM操作性能、降低内存消耗比较相符。

###最终结果

&emsp;&emsp;综上所述，目前Angualr1.3版本比较稳定且较1.2版本有性能提升，新版本中加入的一些新特性也对性能改善有一定的帮助。因此有必要将版本由v1.2.25提高到v1.3.5。
