# Array Constructor Method
Array.isArray(obj)




	
# Array-Like Objects
[知识参考](http://speakingjs.com/es5/ch17.html#array-like_objects)
包括index和length，但是没有数组所具有的方法
1. arguments:只存在于函数中，是一种类数组元素。

```
	function logArgs() {
  	  for (var i=0; i<arguments.length; i++) {
       	  console.log(i+'. '+arguments[i]);
      }
	}
	logArgs('hello', 'world');

```

 - 具有length属性；可以通过index访问元素；没有slice等数组方法
 - 是一个对象,可以获取对象的所有方法
 
 ```
 // 此时的1是arguments的index.
function f() {
 arguments.hasOwnProperty(1); 
 return 1 in arguments 
}  
f('a')   // false
f('a', 'b')   //true
 ```
 [all parameters by index:the special variable arguments](193页)

