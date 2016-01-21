###fugu-UI tooltip调研

关于tooltip部分，有三种可用的指令：
- `tooltip`：只显示提示的文本。
- `tooltip-html`：输出HTML字符组成的表达式，并且HTML内容没有进行编译，如果需要被编译的话，建议使用`tooltip-template`来代替。
- `tooltip-template`：接受一个来自模板文件的模板，这些要被包在一个标签中。