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
	//这是供JQ内部使用的数组方法
	push: push,
	sort: arr.sort,
	splice: arr.splice
};
```

#### toArray：转数组

`.toArray()`将当前`jQuery`对象转换成真正的数组，执行时通过方法`call()`指定方法执行的环境，即关键字`this`所引用的对象.

```
slice.call( this ) == [].slice.call(this);
slice如果不传参就是截取整个数组  call 改变了截取环境 返回的就是this所对应的数组
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

#### get：获取集合

`.get([index])：`中括号表示可选。该方法返回当前`jQuery`对象中指定位置的元素或包含了全部元素的数组。如果指定参数`index`，则返回一个单独的元素；参数`index`从0开始，并且支持负数，负数表示从末尾开始算起。

`183行首先判断`num`是否小于0，则用`length+num`重新计算下标，然后用`[]`获取指定位置的元素。如果`num`大于等于0，则直接返回指定位置的元素。否则调用`[].slice.call(this)`返回所有元素，存入空数组里。

#### pushStack：对象入栈

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

传入一个函数,然后返回部分拆分来看 

1.this.pushStack（fn）  把函数结果赋给新的jq对象

2.fn = jQuery.map( 

参数一：this, 

参数二：function( elem, i ) {return callback.call( elem, i, elem );} ) 就是把传进来的JQ对象elem 每个元素都执行一遍回调函数，并把索引和elem 当做回调参数，参考jquery.map 方法中的 callback( elems[ i ], i, arg );

第447行

第一参数:待遍历的数组或对象；也就是上面的this

第二参数:回调函数，也就是上面的参数二

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
		//防止ret数组是个二维数组 apply会把ret拆分一项项和空数组执行concat合并 这样就成了一个一维数组
		return concat.apply( [], ret );
	},
```

#### slice

call方法: 
语法：call([thisObj[,arg1[, arg2[,   [,.argN]]]]]) 
定义：调用一个对象的一个方法，以另一个对象替换当前对象。 
说明： 
call 方法可以用来代替另一个对象调用一个方法。call 方法可将一个函数的对象上下文从初始的上下文改变为由 thisObj 指定的新对象。 
如果没有提供 thisObj 参数，那么 Global 对象被用作 thisObj。 

apply方法： 
语法：apply([thisObj[,argArray]]) 
定义：应用某一对象的一个方法，用另一个对象替换当前对象。 
说明： 
如果 argArray 不是一个有效的数组或者不是 arguments 对象，那么将导致一个 TypeError。 
如果没有提供 argArray 和 thisObj 任何一个参数，那么 Global 对象将被用作 thisObj， 并且无法被传递任何参数。

他们的区别在于：

对于apply和call两者在作用上是相同的，但两者在参数上有区别的。
对于第一个参数意义都一样，但对第二个参数：
apply传入的是一个参数数组，也就是将多个参数组合成为一个数组传入，而call则作为call的参数传入（从第二个参数开始）。
如 func.call(func1,var1,var2,var3)对应的apply写法为：func.apply(func1,[var1,var2,var3])同时使用apply的好处是可以直接将当前函数的arguments对象作为apply的第二个参数传入。
```
slice: function() {
		return this.pushStack( slice.apply( this, arguments ) );
	},
```

这段代码的作用其实也就是把jq对象转化成数组，再通过pushStack函数返回一个新的JQ对象

#### first，last，eq

```
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
```

eq先获取合集长度存入len，然后如果传入参数为正数，j = j + i ，否则 j = j + len - - i

最终 j大于0 并且小于总长度，就根据索引取值this[j]并放在数组中调用pushStack，否则结果为空数组

#### end：返回前一个选择栈

```
end: function() {
// // 返回当前jQuery对象的prevObject对象，简单直接
	return this.prevObject || this.constructor();
},
//在pushStack()函数中 pushStack会将传入的DOM元素创建为一个新的jQuery对象，并将这个新的jQuery对象的prevObject属性进行修改。

```

举例：

选择所有段落，找到这些段落中的 span 元素，然后将它们恢复为P标签，并把P标签设置为两像素的红色边框：

一般情况下$("p").find("span")这时候的this应该是对应的span集合，但find和eq等方法都使用了pushStack函数 在这个函数中有这样的一行代码：194行 `ret.prevObject = this;`

pushStack中用到了jQuery.merge()方法，是将第2个参数中的各个参数拼接到第1个参数的尾部，第一个参数this.constructor()，其实是创建了一个空的jQuery对象。然后我们将当前对象赋值给新对象的prevObject属性，最后返回新对象，这样，新对象就持有了一个对之前对象的引用，这样就实现了一个维护前一个jQuery对象的栈。 调用end（）后就把之前的对象返回

```
$("p").find("span").end().css("border", "2px red solid");
```

参考：<https://www.cnblogs.com/MnCu8261/p/6046532.html> 

<https://oychao.github.io/2017/07/13/javascript/29_jquery_prevobject/> 

## 六.240： jQ的继承方法

### `$.extend 和$.fn.extend`

#### 1.只有一个对象字面量参数时  

```
$.extend({ //这种是扩展工具方法
    aaa:function(){
    	alert(1)
    }
})
$.aaa() //弹出1
$.fn.extend({ //扩展JQ实例方法
    bbb:function(){
        alert(2)
    }
})
$().bbb() //弹出2
为什么连等同一个函数却能不同的操作？因为不同的调用方式，其中的this是不同的
因为$是JQ函数，$.fn是JQ原型对象
$.extend()->this指向$ -> this.aaa() -> $.aaa()
$.fn.extend() -> this指向JQ原型 -> this.bbb() ->原型调方法必定是创建对象去调用是->$().bbb()
```

#### 2.多个字面量 后面的对象都扩展到第一个参数里面

```
var a = {}
$.extend(a,{name:'zhangsan'},{age:18})

console.log(a) //{name:'zhangsan',age:18}
```

#### 3.深拷贝

```
//当第一个参数为true时 开启深拷贝
var a = {}
var b = {haha:{xixi:345}}
$.extend(true,a,{name:'zhangsan'},{age:18},b)
b.haha.xixi = 789
console.log(a) //{age: 18,haha: {xixi: 345},name: "zhangsan"}
console.log(b) //{haha: {xixi:789}}
```

### 源码解析

```
jQuery.extend = jQuery.fn.extend = function() {
//声明变量
/*
    变量 options：指向某个源对象。
    变量 name：表示某个源对象的某个属性名。
    变量 src：表示目标对象的某个属性的原始值。
    变量 copy：表示某个源对象的某个属性的值。
    变量 copyIsArray：指示变量 copy 是否是数组。
    变量 clone：表示深度复制时原始值的修正值。
    变量 target：指向目标对象,申明时先临时用第一个参数值。
    变量 i：表示源对象的起始下标，申明时先临时用第二个参数值。
    变量 length：表示参数的个数，用于修正变量 target。
    变量 deep：指示是否执行深度复制，默认为 false。
    ps:源对象指的是把自己的值付给别人的对象；目标对象指的是被源对象赋值的对象
    */

	var options, name, src, copy, copyIsArray, clone,
	//默认目标元素为第一个参数
		target = arguments[ 0 ] || {},
		//目标元素的参数索引
		i = 1,
		length = arguments.length,
		deep = false;
//判断是否深拷贝 
	if ( typeof target === "boolean" ) {
		deep = target;
		//目标元素就是第二个参数
		target = arguments[ i ] || {};
		//参数索引一起改变
		i++;
	}

// 判断是否是对象或者函数 不是 目标元素赋值一个空对象
	if ( typeof target !== "object" && !isFunction( target ) ) {
		target = {};
	}
//判断是不是扩展插件 也就是判断参数数量
	if ( i === length ) {
		target = this;
		i--;
	}
	for ( ; i < length; i++ ) {
//判断参数是否有意义 防止为null
		if ( ( options = arguments[ i ] ) != null ) {
			for ( name in options ) {
			//获取源对象上，属性名对应的属性
				copy = options[ name ];
//如果复制值copy 与目标对象target相等，
//为了避免深度遍历时死循环，因此不会覆盖目标对象的同名属性。
				if ( name === "__proto__" || target === copy ) {
					continue;
				}
			//深浅拷贝
			//递归参数存在 并且后面是有效参数 并且是对象或者数组
				if ( deep && copy && ( jQuery.isPlainObject( copy ) ||
					( copyIsArray = Array.isArray( copy ) ) ) ) {
					src = target[ name ];
					//如果是数组
					if ( copyIsArray && !Array.isArray( src ) ) {
						clone = [];
					} else if ( !copyIsArray && !jQuery.isPlainObject( src ) ) {
						clone = {};
					} else {
						clone = src;
					}
					copyIsArray = false;
			//递归拷贝
					target[ name ] = jQuery.extend( deep, clone, copy );
				} else if ( copy !== undefined ) {
					target[ name ] = copy;
				}
			}
		}
	}
	return target;
};
```

```
if ( name === "__proto__" || target === copy ) {
	continue;
}
举例：
var a = {}
$.extend(a,{name:a})
这里a 被重复引用 有了上面的判断 会直接跳过
```

```
if ( copyIsArray && !Array.isArray( src ) ) {
	clone = [];
} else if ( !copyIsArray && !jQuery.isPlainObject( src ) ) {
	clone = {};
} else {
	clone = src;
}
这里这个条件判断是什么意思呢 -> !Array.isArray( src )，!jQuery.isPlainObject( src )
意思就是当属性值是个数组，并且目标对象上没有 就重新创建一个新的数组或者对象，否则就沿用目标对象src
举例：
var a = {name:{job:'it'}}
var b = {name:{age:18}}
$.extent(true,a,b)
console.log(a) //{name:{job:'it',age:18}}
要是没有这判断条件 程序就判断为对象后就会直接新建一个空对象然后复制 相当于把之前的结果覆盖了
打印a 的结果就是 {name:{age:18}} 
```

## 七.JQ扩展函数

 如果只为$.extend()指定了一个参数，则意味着参数target被省略。此时，target就是jQuery对象本身。通过这种方式，我们可以为全局对象jQuery添加新的函数。 

参考：<https://www.cnblogs.com/guaidianqiao/p/7771427.html> 

```
jQuery.extend( {

	// 生成唯一JQ字符串(内部)
	expando: "jQuery" + ( version + Math.random() ).replace( /\D/g, "" ),
	// DOM是否加载完(内部)
	isReady: true,
	//抛出异常	
	error: function( msg ) {
		throw new Error( msg );
	},
	//空函数  有啥用暂时不知道
	noop: function() {},
	//是否为对象自变量  纯粹的对象"，就是该对象是通过 "{}" 或 "new Object" 创建的
	//之所以要判断是不是 plainObject，是为了跟其他的 JavaScript对象如 null，
	//数组，宿主对象（documents）等作区分，因为这些用 typeof 都会返回object。
	isPlainObject: function( obj ) {
		var proto, Ctor;
		//判断非对象 排除掉明显不是obj的以及一些宿主对象如Window
		if ( !obj || toString.call( obj ) !== "[object Object]" ) {
			return false;
		}
		//如果没有继承属性，则返回 null 。
		proto = getProto( obj );

		// 没有原型的对象 也就是普通对象 如object.create（null）就返回 true
		if ( !proto ) {
			return true;
		}

		//以下判断通过 new Object 方式创建的对象
     	//判断 proto 是否有 constructor 属性，如果有就让 Ctor 的值为 proto.constructor
     	//如果是 Object 函数创建的对象，Ctor 在这里就等于 Object 构造函数
		Ctor = hasOwn.call( proto, "constructor" ) && proto.constructor;
		
		// 在这里判断 Ctor 构造函数是不是 Object 构造函数，用于区分自定义构造函数和 Object 构造函数
		return typeof Ctor === "function" && fnToString.call( Ctor ) === ObjectFunctionString;
	},
//是否为空的对象
	isEmptyObject: function( obj ) {
		var name;

		for ( name in obj ) {
			return false;
		}
		return true;
	},

	//  全局解析JS
	globalEval: function( code, options ) {
		DOMEval( code, { nonce: options && options.nonce } );
	},
//遍历集合
	each: function( obj, callback ) {
		var length, i = 0;

		if ( isArrayLike( obj ) ) {
			length = obj.length;
			for ( ; i < length; i++ ) {
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

		return obj;
	},

	// 去前后空格
	trim: function( text ) {
		return text == null ?
			"" :
			( text + "" ).replace( rtrim, "" );
	},

	// 类数组转真数组
	makeArray: function( arr, results ) {
		var ret = results || [];

		if ( arr != null ) {
			if ( isArrayLike( Object( arr ) ) ) {
				jQuery.merge( ret,
					typeof arr === "string" ?
					[ arr ] : arr
				);
			} else {
				push.call( ret, arr );
			}
		}

		return ret;
	},
//数组的indexOf
	inArray: function( elem, arr, i ) {
		return arr == null ? -1 : indexOf.call( arr, elem, i );
	},

	//合并数组
	merge: function( first, second ) {
		var len = +second.length,
			j = 0,
			i = first.length;

		for ( ; j < len; j++ ) {
			first[ i++ ] = second[ j ];
		}

		first.length = i;

		return first;
	},
//过滤新数组
	grep: function( elems, callback, invert ) {
		var callbackInverse,
			matches = [],
			i = 0,
			length = elems.length,
			callbackExpect = !invert;

		// Go through the array, only saving the items
		// that pass the validator function
		for ( ; i < length; i++ ) {
			callbackInverse = !callback( elems[ i ], i );
			if ( callbackInverse !== callbackExpect ) {
				matches.push( elems[ i ] );
			}
		}

		return matches;
	},

	// 映射新数组
	map: function( elems, callback, arg ) {
		var length, value,
			i = 0,
			ret = [];

		// Go through the array, translating each of the items to their new values
		if ( isArrayLike( elems ) ) {
			length = elems.length;
			for ( ; i < length; i++ ) {
				value = callback( elems[ i ], i, arg );

				if ( value != null ) {
					ret.push( value );
				}
			}

		// Go through every key on the object,
		} else {
			for ( i in elems ) {
				value = callback( elems[ i ], i, arg );

				if ( value != null ) {
					ret.push( value );
				}
			}
		}
		// Flatten any nested arrays
		return concat.apply( [], ret );
	},
	// 唯一标识符(内部)
	guid: 1,

	//备用附加属性
	support: support
} );
```

