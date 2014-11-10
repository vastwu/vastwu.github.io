## googleMap - V3 部分解析
#### 模块加载机制

google没有用目前很流行cmd，amd的那套方式，起码在生产环境中没有，而是通过闭包的方式加大了压缩强度，使得代码变得更简短，查看代码成本也更高（这个对学习来说难度陡增）

首先，google的后台支持批量加载，这个没啥新鲜的
`https://maps.gstatic.com/cat_js/intl/zh_tw/mapfiles/api-3/13/12/%7Bcommon,map%7D.js`
加载了common和map两个模块

该请求的返回值为以下方式

```javascript
	google.maps.__gjsload__('common', '\'use strict\';var Lj=isNaN,Pj=parseInt,Q.....');
	google.maps.__gjsload__('map', '\'use strict\';function ny(a){this.b=a||[]}Up[....');
```

我们可以将其识别为

```javascript
	google.maps.__gjsload__(moduleName, moduleCode);
```

看起来似乎有点像 cmd的 `define(moduleName, factory)`，但这里的两个参数均为String

<!--more-->

>说一下此处的阅读方式，在控制台里输入`new Function(moduleCode)`后会输出一大堆代码，这里是用code构造的新function，已经处理过了单引号双引号等问题，扔到IDE里格式化一下就可以看了，不过依旧痛苦，需要耐心

我们在其几乎所有的模块中都会找到一段类似的代码

```javascript
	Jf[cf] = function(a) {
		eval(a)
	};
```

教科书中的绝对禁忌`eval`，`eval`的功能大家可以自行查阅一下，此处的作用是使一段代码在`eval`所处的上下文中运行，以便可以访问作用域内的变量，

先看一段常规的跨模块调用

```javascript
//util.js
define(function(){
	return {
		$:function(id){
			return document.getElementById(id);
		}
	}
});
//map.js
define(function(){
	var util = require('src/common/util.js')
	var layer = require('src/modules/layer.js');
	var elem = util.$('mapContainer')
});
```

当加载器发现引用了util后，会将util的接口保存。并在运行map.js的factory时，将util的接口返回;

而google的eval则将依赖调用转变为另一种模式
google所有的模块都对外提供了一个run接口（暂且叫这个吧，不知道压缩前叫什么），而不提供实际的功能接口

```javascript
//util.js
(function(){
	var $ = function(){
		return document.getElementById(id);
	}
	global['util'].run = function(code){
		eval(code)
	}
})();

```

当另一个模块依赖util时，会调用该接口注入自身代码，让程序变成类似如下的模式

```javascript
//util.js
(function(){
	var $ = function(){
		return document.getElementById(id);
	}
	global['util'].run = function(code){
		//注入的map.js
		(function(){
			var elem = $('mapContainer');
		})();
	}
})();
```

由于js的作用域访问性质，内层闭包可以访问外部变量，即$可以在内部直接访问上层闭包的变量，压缩工具可以直接略去长长的一串require（path），甚至当一个模块内有数十个require时，都可以使用这种方式压缩掉path，特别是单嵌套依赖时，这种结构更神奇

```javascript
//main.js
(function(){
	var LOGO_SRC = 'http://map.soso.com/images/logo.png';
	var printlog = function(text){
		console.log(text);
    }
	global['main'].run = function(code){
		//util.js
		(function(){
			var $ = function(){
				return document.getElementById(id);
			}
			global['util'].run = function(code){
				//注入的map.js
				(function(){
					var elem = $('mapContainer');
					printlog('debug' + LOGO_SRC);
				})();
			}
		})();
	}
})();
```

main内注入了util，util内注入了map，构成了层层依赖关系
googleMap代码中，例如control的模块里，会有非常多的变量不在本模块内找不到声明的，就是因为使用了这种模式，调用的某个函数在上层依赖的文件中，而构造的嵌套闭包结构又使其在运行期可以访问，IDE又无法识别引用，让代码阅读困难更进一层
另外google在代码中加入了严格模式，是为了防止eval中的变量外泄。


构建出这种结构过程是相当复杂的，重点在于分析每个模块的依赖，这个我们一直在尝试效仿的，现在学到了一部分，并且已经在soso地图apiV2版本中实战，这个就不是一两句话说得清的了，有兴趣可以与zhengxu同学详谈


#### 关于组件

这种构造组件的方式很常见，返回一个类似代理控制类的对象，来代替直接操作dom节点

```javascript
    //define
    function TextView(text){
    	this._dom = document.createElement('span');
    	this._dom.innerText = text;
	}
	TextView.prototype = {
		setColor:function(color){
			this._dom.style.color = color;
		},
		hide:function(){
			this._dom.style.display = 'none'
		}
	}

	//useage
	var span = new TextView('hello world');
	span.setColor('red');
	span.hide();
```

再来看google的代码，代码中很常见如下的这种小函数

```javascript
	function xO(a) {
		var b = a.d[w];
		b.WebkitBoxSizing = b.mozBoxSizing = b.boxSizing = "border-box";
		Wj(b, "relative");
		Yj(b, ks(b, 0));
		dk(b, Jp.b ? "0 0 0 5px" : "0 5px 0 0");
		ck(b, "inline-block");
		lk(b, "#fff");
		Bk(b, X(1) + " solid");
		qu(a.d, X(1));
		b = 13;
		Jt() && (b -= 2);
		Nh(a.d, new Q(b, b));
		zu(a.e, !1);
		yO(a, !1)
	}
```
> 此处阅读指南，由于googleMap的模块代码是以字符串形式加载，并eval运行，使得常规的断点方式无法打入运行过程中，我们通过chrome控制台中的Sources的EventListenerBreakpoints和dom元素的Break on Attributes modifications来插入执行过程中，而插入后的代码即会看到是[VM]模式，并非真是代码中的某一部分中（main.js除外）。

大致看来，该函数是在设置各种样式，其实此处更像是装饰器，在整体代码中有很多这种函数，即对一个dom节点进行包装，使其具有某种能力，而该包装器是可以被重复使用，其中的核心就是dom节点，这与我们尽量像隐藏dom的思路正相反

```javascript
function wrapTextView = function(dom){
	dom.style.fontSize = 14;
	dom.style.color = 'red';
}
function wrapTableView = function(dom){
	return {
		add:function(child){
            dom.appendChild(child);
	    }
    }
}

//useage
var span = document.createElement('div');
wrapTextView(span);


var div = document.createElement('div');
var table = wrapTableView(div);
table.add(span);

var child_child = document.createElement('div');
var child = wrapTableView(span);
child.add(child_child);
```

上面是我随便写的两个包装器，包装器将任意dom节点包装秤具有某种能力或某种外观的对象，并且该函数可以重复使用，之前提到的ui构建方式，当功能增加时，需要不断的新增接口，会导致接口越来越多，越庞大，google的方式核心始终为dom节点，所有的包装行为执行结束后，对外使用的仍然是dom节点，
