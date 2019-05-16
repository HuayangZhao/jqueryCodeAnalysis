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

## 二.48：开始部分变量

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

## 三.91：全局函数DOMEval

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

## 四.128：判断类型 toType 

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

## 五.144：设置jquery对象

```
 var version = "3.4.1", //这个版本号变量的作用暂且不知道 先不管
     jQuery = function (selector, context) {
     	return new jQuery.fn.init(selector, context);
     },
     rtrim = /^[\s\uFEFF\xA0]+|[\s\uFEFF\xA0]+$/g; //这个正则是匹配DOM和空格的
     //参考地址：https://imququ.com/post/bom-and-javascript-trim.html
```

 ```
这里jQuery是一个函数，通过前面已经知道jquery最终暴露出来的就是我们使用的$，这里的jquery返回的是new出来的新对象，只有返回一个对象,在外面使用时才能像$().css()这样对象点出方法使用
 ```

```
一般我们写面向对象的时候会这么写：
1.构造函数
function Arr(){}
2.在原型上加方法
Arr.prototype.init=function(){‘给个初始函数，调用就执行’}
Arr.prototype.css=function(){}
3.创建对象调用方法
var a1 = new Arr()
a1.init() //先初始化
a1.css() //再执行其他方法

JQ中是这样：
function jquery(){
    return new jQuery.fn.init(selector, context);
}
jquery.prototype.init = function(){}
jquery.prototype.css = function(){}
jquery()返回一个对象
平时代码就是这么写的：jquery().css()
但是看代码jquery()返回的是new jQuery.fn.init(selector, context);
也就是说jQuery.fn.init是一个构造函数，但是css()方法是在jquery的原型上
jquery().css()这么写明显会找不到css方法
在159行的代码中 jQuery.fn = jQuery.prototype 把jquery的原型赋值给了jquery.fn
jquery的原型对象和jquery.fn就成了同一个对象
再这么写jquery().css()就没问题了
这么写的好处就是可以省去代码步骤（我觉得就是链式编程，但感觉不完全是因为此处。。待往下看）
至于这个init函数 往下搜了一下 发现第2934行代码
init = jQuery.fn.init = function( selector, context, root ) {-----}
第3034行
init.prototype = jQuery.fn;
可以发现调用这个init函数就是再调用jquery
```

### 参数

```
selector从字面意思上可以看出是选择器也就是我们平时$('selector'), 

context从字面意思上来看也就是承上启下的,比如我们$('li')选择的是页面上的所有的li,要是这么写$('li','ul')就是选择ul下面的li,这里的ul就是第二个参数,起限制作用
```

### jquery原型

```
jQuery.fn = jQuery.prototype = {

	// 当前版本
	jquery: version,
	
	//修正原型指向问题
	constructor: jQuery,

	//对象默认长度
	length: 0,
	
	//调用此方法可以转换成数组
	toArray: function() {
		return slice.call( this );
	},
	//获取数组指定元素
	get: function( num ) {
		//如果get参数为空 直接转换成数组返回
		if ( num == null ) {
			return slice.call( this );
		}
		//如果不为空 且大于0则返回指定下标 this什么时候转成数组的 我猜应该是再init函数里面 待往后看
		//小于零 用总长度-num 从末尾返回指定
		return num < 0 ? this[ num + this.length ] : this[ num ];
	},

	pushStack: function( elems ) {
		var ret = jQuery.merge( this.constructor(), elems );
		ret.prevObject = this;
		return ret;
	},

	// Execute a callback for every element in the matched set.
	each: function( callback ) {
		return jQuery.each( this, callback );
	},

	map: function( callback ) {
		return this.pushStack( jQuery.map( this, function( elem, i ) {
			return callback.call( elem, i, elem );
		} ) );
	},

	slice: function() {
		return this.pushStack( slice.apply( this, arguments ) );
	},

	first: function() {
		return this.eq( 0 );
	},

	last: function() {
		return this.eq( -1 );
	},

	eq: function( i ) {
		var len = this.length,
			j = +i + ( i < 0 ? len : 0 );
		return this.pushStack( j >= 0 && j < len ? [ this[ j ] ] : [] );
	},

	end: function() {
		return this.prevObject || this.constructor();
	},

	// For internal use only.
	// Behaves like an Array's method, not like a jQuery method.
	push: push,
	sort: arr.sort,
	splice: arr.splice
};
```

#### toArray

`.toArray()`将当前`jQuery`对象转换成真正的数组，执行时通过方法`call()`指定方法执行的环境，即关键字`this`所引用的对象.

```
slice.call( this ) == [].slice.call(this);
```

也就是说该方法在执行时，会将原本属于数组对象的`slice`方法交给当前执行环境（`this`所引用的对象）的对象去处理。

看下面一个实例：

```
function fn1(){
   console.log(1);
}
function fn2(){
    console.log(2);
}

fn1.call(fn2);     //输出 1   fn2调用了fn1方法，并将fn2对象this指向fn1对象
 
```

 参考：https://blog.csdn.net/bowen11233/article/details/53286826,

<https://segmentfault.com/a/1190000002770303> 

call底层实现: https://www.cnblogs.com/donghezi/p/9742778.html

#### get

`.get([index])：`中括号表示可选。该方法返回当前`jQuery`对象中指定位置的元素或包含了全部元素的数组。如果指定参数`index`，则返回一个单独的元素；参数`index`从0开始，并且支持负数，负数表示从末尾开始算起。

`183行首先判断`num`是否小于0，则用`length+num`重新计算下标，然后用`[]`获取指定位置的元素。如果`num`大于等于0，则直接返回指定位置的元素。否则调用`[].slice.call(this)`返回所有元素，存入空数组里。

#### pushStack

191行:`this.constructor`指向构造函数`jQuery()`,通过merge方法把参数`elems` 合并到`this.constructor`并赋给新的`jQuery`对象`ret`

194行：`ret.prevObject = this;`,赋予JQ对象,最后返回新的`jQuery`对象`ret`.

413行 关于`merge`方法：实现了两个数组合并

```
merge: function( first, second ) {
        var len = +second.length,
            j = 0,
            i = first.length;
        for ( ; j < len; j++ ) {
            first[ i++ ] = second[ j ];
        }
        //我觉得这里重新赋予length是多余的..可能我太渣....
        first.length = i;
        return first;
    },
```

#### each

为毛要这么写?因为在361行JQ的工具方法中有each方法的定义

为毛再原型上定义了还再工具方法中定义 因为我们可以这么用`$.each()`也可以这么用`$().each()`

```
each: function( obj, callback ) {
		var length, i = 0;
		//判断是不是类数组 496行定义isArrayLike函数
		if ( isArrayLike( obj ) ) {
			length = obj.length;
			for ( ; i < length; i++ ) {
			//call的作用就是把callback放到obj[ i ]上使用，
			//后面的 `i, obj[ i ]`这些做为参数传入callback。回调函数返回false，则遍历结束
				if ( callback.call( obj[ i ], i, obj[ i ] ) === false ) {
					break;
				}
			}
		} else {
			for ( i in obj ) {
				if ( callback.call( obj[ i ], i, obj[ i ] ) === false ) {
					break;
				}
			}
		}
	//返回传入的形参object来支持链式语法
		return obj;
	},
```

#### map

```
map:function( callback ) {
		return this.pushStack( jQuery.map( this, function( elem, i ) {
			return callback.call( elem, i, elem );
		} ) );
	},
```

传入一个函数,然后返回部分着来看 先要执行 jQuery.map( this, function( elem, i ) )也就是下面的map函数

第447行

第一参数:待遍历的数组或对象；

第二参数:回调函数，会在数组每个元素或对象上执行

第三参数:仅限jQuery内部使用，如果调用jQuery.map()时传入了参数arg，则会传给回调函数callback

```
map: function( elems, callback, arg ) {
		var length, value,
			i = 0,
			ret = [];
		if ( isArrayLike( elems ) ) {
			length = elems.length;
			for ( ; i < length; i++ ) {
			//每个元素回调函数处理一边存入新数组ret中
				value = callback( elems[ i ], i, arg );
				if ( value != null ) {
					ret.push( value );
				}
			}
		} else {
			for ( i in elems ) {
				value = callback( elems[ i ], i, arg );

				if ( value != null ) {
					ret.push( value );
				}
			}
		}
		return concat.apply( [], ret );
	},
```

