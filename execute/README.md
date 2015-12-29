#AngularJS执行流程
Angular执行流程分为四个部分，分别是启动阶段、初始化阶段、编译链接、运行阶段四个部分，具体如下：

![AngualrJS执行流程](http://static.oschina.net/uploads/space/2015/0525/153837_tPbY_257704.png)

1. 启动阶段： 大家应该都知道，当浏览器加载一个HTML页面时，它会将HMTL页面先解析成DOM树，然后逐个加载DOM树中的每一个元素节点。我们可以把AngularJS当做一个类似jQuery的js库，我们通过`<script>`标签引入到HTML中，那么此时Angular就做为一个普通的DOM节点等待浏览器解析，当浏览器解析到这个节点时，发现它是一个js文件，那么浏览器会停止解析剩余的DOM节点，开始执行这个js（即angular.js），同时Angular会设置一个事件监听器来监听浏览器的DOMContentLoaded事件。当Angular监听到这个事件时，就会启动Angular应用。

2. 初始化阶段：Angular开始启动后，它会查找ng-app指令，然后初始化一系列必要的组件（即$injector、$compile服务以及$rootScope），接着重新开始解析DOM树。

3. 编译、链接：$compile服务通过遍历DOM树的方式查找有声明指令的DOM元素。当碰到带有一个或多个指令的DOM元素时，它会排序这些指令（基于指令的priority优先级），然后使用$injector服务查找和收集指令的compile函数并执行它。
每个节点的编译方法运行之后，$compile服务就会调用链接函数。这个链接函数为绑定了封闭作用域的指令设置监控。这一行为会创建实时视图。
最后，在$compile服务完成后，AngularJS运行时就准备好了。

4. 运行阶段： Angular提供了自己的事件循环。指令自身会注册事件监听器，因此当事件被触发时，指令函数就会运行在AngularJS的$digest循环中。$digest循环会等待$watch表达式列表，当检测到模型变化后，就会调用$watch函数，然后再次查看$watch列表以确保没有模型被改变。
一旦$digest循环稳定下来，并且检测到没有潜在的变化了，执行过程就会离开Angular上下文并且通常会回到浏览器中，DOM将会被渲染到这里。

其中，将执行流程分为：启动过程、遍历链接过程、运行阶段三部分进行源码分析