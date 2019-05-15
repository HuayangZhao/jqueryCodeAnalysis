# Jquery源码研究 版本3.4.1

## 一.开始部分匿名函数

```
(function (global, factory) {  //第十四行

    "use strict";    //启用严格模式 
    
    //这里是兼容CommonJS规范的js框架 像nodeJs中没有window和document
    if (typeof module === "object" && typeof module.exports === "object") 
    
    //这里判断有没有document对象 ，global看自调用传参应该是window或者this，this这里指的是非window参数本身，若有window，module.exports = factory(global, true)，没有抛出异常jquery需要具有window和DOM
    
        module.exports = global.document ?
                factory(global, true) :
                
      //这个参数w又是哪里来的？依旧返回jq功能函数，保证不相关功能正常
                function (w) {
                    if (!w.document) {
                        throw new Error("jQuery requires a window with a document");
                    }
                    return factory(w);
                };
    } else {
    // 有window对象第二参数就不传  转到第10547行
        factory(global);
    }
})(typeof window !== "undefined" ? window : this, function (window, noGlobal) {...JQ功能函数})
//参数一是判断是不是window环境  参数二是jq功能函数
```

转到第10547行

~~~
 if (!noGlobal) {
      window.jQuery = window.$ = jQuery;
  }
  //上面说有window也就是不存在CommonJS规范的js框架，将jQuery挂载到了window上，这样就使得外部可以访问到jQuery内部的变量和方法
~~~

参考文档：https://www.cnblogs.com/vajoy/p/3623103.html

总的来看JQ就是一个自调用函数

## 二.48行：开始部分变量

```
		   var arr = [];
            var document = window.document;
    // 获取原型对象
            var getProto = Object.getPrototypeOf;
    // 数组方法
            var slice = arr.slice;
            var concat = arr.concat;
            var push = arr.push;
            var indexOf = arr.indexOf;
            var class2type = {};
            var toString = class2type.toString;
    // asOwnProperty表示是否有自己的属性。这个方法会查找一个对象是否有某个属性，但是不会去查找它的原型链。
            var hasOwn = class2type.hasOwnProperty;
    //属性转字符串
            var fnToString = hasOwn.toString;
            var ObjectFunctionString = fnToString.call(Object);
            var support = {};
            var isFunction = function isFunction(obj) {
       /*
          nodeType 属性返回节点类型。
          如果节点是一个元素节点，nodeType 属性返回 1。
          如果节点是属性节点, nodeType 属性返回 2。
          如果节点是一个文本节点，nodeType 属性返回 3。
          如果节点是一个注释节点，nodeType 属性返回 8。
        */
       // 在部分浏览器中typeof document.createElement( “object” ) === “function”，因此要追加判断
               // 返回非元素节点的函数
                return typeof obj === "function" && typeof obj.nodeType !== "number";
            };
      // 判断是否是window, window.window 为自引用，所以jquery依照此标准来实现isWindow
            var isWindow = function isWindow(obj) {
                return obj != null && obj === obj.window;
            };

```

## 三.91行：全局函数DOMEval

```
 var preservedScriptAttributes = {
                type: true,
                src: true,
                nonce: true,
                noModule: true
            };
            //code:script中要被执行的JS代码
            //node：属性对象
			//doc:DOM节点对象
            function DOMEval(code, node, doc) {
                doc = doc || document;
                var i, val,
                        script = doc.createElement("script");
                script.text = code;
                if (node) {
                    for (i in preservedScriptAttributes) {
                        val = node[i] || node.getAttribute && node.getAttribute(i);
                        if (val) {
                            script.setAttribute(i, val);
                        }
                    }
                }
                doc.head.appendChild(script).parentNode.removeChild(script);
            }

```

DOMEval()方法在head部分插入了需要执行的js脚本，该脚本会立即执行，然后从head部分移除掉了，保持页面的整洁，**这段代码一般用于动态执行从服务器端返回的代码**。这种情况一般总是会要求代码在全局作用域内执行。 

```
//比如现在 声明变量 
var code = "alert(1)"
var url,params;
//调用Ajax
$.get(url,params,function(code){
    var script = document.createElement('script');
    script.text = code
    document.head.appendChild(script).parentNode.removeChild(script)
});
//最终alert会被执行
```

参考文档：<https://blog.csdn.net/qiumingsheng/article/details/81416779> 

## 四.128行：判断类型 toType 

```
 function toType(obj) {
                if (obj == null) {
                    return obj + "";
                }
                // Support: Android <=2.3 only (functionish RegExp)
                return typeof obj === "object" || typeof obj === "function" ?
                        class2type[toString.call(obj)] || "object" :
                        typeof obj;
            }
```

obj为空返回obj字符串,否则再判断obj是否为对象或者函数，是则运行class2type[ toString.call(obj)] || "object"  否则直接返回 typeof obj就可以了，因为出来function和object只能是基本类型，用typeof判断就足够了 

下面来解析 `class2type[ toString.call(obj) ] || "object" `

```
class2type[toString.call(obj) ]:
前面变量定义为：
 var class2type = {};
 var toString = class2type.toString;
 
 这里其实也就是:
 class2type[{}.toString.call(obj)]
 
 来看个列子：
  var arr=new Array();
  console.log({}.toString.call(arr)); //输出为 ：[object Array]
```

现在可以先跳转到491行看看变量class2type空对象中被存放了什么

```
jQuery.each( "Boolean Number String Function Array Date RegExp Object Error Symbol".split( " " ),
function( i, name ) {
	class2type[ "[object " + name + "]" ] = name.toLowerCase();
} );
```

这里就是把所有的数据类型遍历一遍然后用键值对存到class2type中 通过打印可以看看class2type中存入什么

```
var class2type = {}
	var arr = "Boolean Number String Function Array Date RegExp Object Error Symbol".split( " " )
	arr.forEach(function( name,i) {
		class2type[ "[object " + name + "]" ] = name;
	} )
console.log(class2type)
打印如下：{
            [object Array]: "Array"
            [object Boolean]: "Boolean"
            [object Date]: "Date"
            [object Error]: "Error"
            [object Function]: "Function"
            [object Number]: "Number"
            [object Object]: "Object"
            [object RegExp]: "RegExp"
            [object String]: "String"
            [object Symbol]: "Symbol" 
          }
```

现在再来看 如果传入的参数是个数组 [ 1,2,3]

`class2type[toString.call(arr) ] = class2type[{}.toString.call(arr)] = class2type[[object Array]]= "Array"`

参考文档：<https://www.cnblogs.com/y8932809/p/5870970.html> 



 