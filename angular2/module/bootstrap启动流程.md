###bootstrap启动流程
相较于Angular1.x来说，Angular2的bootstrap发生了一些变化：
####以组件为核心
- 在Angular1.x中，bootstrap是围绕DOM元素展开的，无论你使用ng-app还是手动执行bootstrap() 函数，自举过程是建立在DOM之上的。

- 而在Angular2中，bootstrap是围绕组件开始的，你定义一个组件，然后启动它。如果没有一个组件， 你甚至都没有办法使用Angular2！

####支持多种渲染引擎
以组件为核心，意味着Angular2在内核已经隔离了对DOM的依赖，DOM仅仅作为一种`可选`的渲染引擎，可以支持服务器端、移动端。

![多种渲染引擎](http://www.hubwiz.com/course/5599d367a164dd0d75929c76/img/render-arch.jpg)
