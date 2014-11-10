# javascript动画设计简介 --- 动画效果指导思想

通过连贯的改变dom元素的位置达到视觉上动画效果

## 名词解释
* 帧
* 帧率
* 进度
* 缓动函数

## 大纲
* 效果
	* 效果实现（属性分解）
	* 缓动函数
* 平滑度
	* 帧率控制
	* 变化量控制
* 动画的控制（暂停，继续，加速，减速，播放到指定进度，跳帧）
	* 动画实现思路（连续，瞬时）
* 性能问题
	* 重绘
	* 脏属性
* 代码结构
	* 解耦
	* 可复用

## STEP 1. 起步
先从最简单的动画效果说起，如下代码。

```javascript
var play = function(dom, end){
	var timer = setInterval(function(){
		var left = parseInt(dom.style.left);    //后文简化为getLeft
		if(left >= end){
			clearInterval(timer);           //后文简化为stop
			return;
		}
		dom.style.left = left + 10 + 'px'；//后文简化为setLeft
	}, 16)；
}
//useage
play($('elem'), 1000);
```


> Question-1: 如何缓动，实现非线性变化

## STEP 2.匀加速缓动 
线性变化一般是指匀速运动，缓动是模拟显示物理效果，实现近似摩擦减速或逐步加速的运动过程，

```javascript
var play = function(dom, add, end){
	var change = 10;
	setInterval(function(){
		change += add;
		var left = getLeft();
		if(left >= end){
			stop();
		}
		setLeft(left + change);
	}, 16);
}
//useage
play($('elem'), 1, 1000);
```

> Question-2:匀加速不能满足需要，为了更好地模拟显示各种物理效果，需要实现变加速、减速效果

## STEP 3.缓动函数

非线性变化的过程需要数学函数的参与来简化代码和提高可控性，引入变化函数的概念

##### 变化函数
简单的缓动函数是由`x,y`组成的二元表达式，输入`x`，输出`y`，表示一种依赖变化关系

* 线性变化函数 

```javascipt
function (x){
	return x;
}
```
* 加速变化函数 

```javascipt
function (x){
	return x * x;
}
```

##### 在动画函数中如何应用？

为了便于控制和人类理解，一般的缓动函数都采取如下策略设计

* 当`x`由 0~1变化时，`y`的值也从0~1变化， 即 `x=0, y=0 && x=1,y=1`
* `x`表示播放进度的百分比，`y`值表示变化值的百分比，乘以变化总量获得当前变化值

```javascript
var leaner = function(x){
	return x;
}
var easeIn = function(x){
	return x * x;
}
var play = function(dom, end, timingFunction){
	var process = 0;
	var left = getLeft();
	var change = end - left;
	setInterval(function(){
		process += 0.1;
		left = left + timingFunction(process) * change;
		setLeft(left);
		if(process == 1){
			stop()
		}
	}, 16);
}
//useage
play($('elem'), 1000, leaner);
```

### 更复杂的缓动函数

资料: [让界面动画更自然](http://djt.qq.com/article/view/1249)



> Question-3:什么`process`只能每次递增`0.1`，想要控制时间怎么办

## STEP 3.帧率和时间

两者需要结合起来一起解决，如果试图通过增加参数来解决问题，随着日后的功能扩展，会逐渐增加更多的参数

另外还有用一个问题，当增加了时间限制，而帧率则与`setInterval`的频率相关，就同时需要调整`del``

```javascript
var play = function(dom, end, timingFunction, duration, fps, ...){
	...
}
//useage
play($('elem'), 1000, leaner, 3000, 60, ...);
```

另外，如果简单的增加时间参数，则会在调整效果时面对更多的复杂的参数，此处需要吧时间作为唯一考量标准，由此推算出其他参数

> 程序设计的重要思想，变量归一化，将多个相关的变量，总结整理为尽可能少的输入项，由程序完成推算过程

```javascript
var play = function(dom, end, timingFunction, duration){
	var process = 0;
	var left = getLeft();
	var change = end - left;
	var startTime = new Date();
	
	setInterval(function(){
		var pass = new Date() - startTime;
		process = pass / duratioin;
		left = left + timingFunction(process) * change;
		if(process >= 1){
			stop();
			left = end;
		}
		setLeft(left);
	}, 16);
}
```

此处由时间计算进度，控制了总体执行时间，如果出现性能问题，则会表现出跳帧现象，从而总时间不受影响，另外此种情况下可能出现最后一帧超过预定结束值的情况，故在最后时多增加了一次设置位置。

> Question-4:我想改`top`,想做透明度渐变，想做斜线运动

## STEP 4.绘制过程解耦

面对复杂的需求，上面的函数显然已经不够用了，不同的动画效果都要写一个不同的`play`函数岂不是太麻烦了，此时需要抽取出变化过程（绘制函数），以确保我们的动画函数自身功能单一

```javascript
var play = function(timingFunction, duration, renderer){
	var process = 0;
	var startTime = new Date();
	var timer = setInterval(function(){
		var pass = new Date() - startTime;
		process = pass / duratioin;
		process = timingFunction(process);
		renderer(process);
		if(process >= 1){
    		stop();
		}
	}, 16);
}
//usage
var startValue = getLeft()
var changeValue = 500 - startValue;
play(leaner, 3000, function(progress){
	setLeft(startValue + changeValue * process);
})
```
此处的`usage`部分跟最原始的写法是不是很相近？此处将绘制部分剥离，`play`只负责计算进度和缓动函数变化，按一定频率调用外部传入的回执函数，这样如何绘制动画就交给了外部使用者来决定，例如我们可以按下面这样使用来进行不同属性，不同变化率的动画
```javascript
//usage
play(leaner, 3000, function(progress){
	setLeft(geValue * process);             //后文统一简化为draw
	setTop(startValue + changeValue * process);
})
play(easeIn, 3000, function(progress){
	setOpacity(1 * process);
})
```

> Question-5: 说好的暂停，继续，加速，减速，播放到指定进度

## STEP 5. 动画控制对象

说到控制，其实就是在动画期间进行动态调整参数

动画效果的两大类设计思路

* 连续：在`step3`之前的函数均是连续型，动画的每一帧的绘制依赖于前一个帧的状态
* 瞬时：从`step3`之后，每一帧的绘制只依赖于进度`process`


```javascript
	var Animate = function(fps, timingFunction, duration, draw){
		var process = 0;
		var k = 1;
		var startTime = new Date();
		var timer = null;
		
		this.start = function(){
			startTime = new Date();
			timer = setInterval(function(){
				var pass = new Date() - startTime;
				process = pass / duratioin;
				process = timingFunction(process * k);
				draw(process);
				if(process >= 1){
					clearInterval(timer);
				}
			}, 16);
		}
		this.changeSpeed = function(t){
			k = t;
		}
		this.pause = function(){
			
		}
		this.stop = function(){
			process = 1;
			clearInterval(timer);
		}
	}
```
> Question-5:过多的绘制对象产生大量`setInterval`,导致性能下降,进程阻塞,如何解决

### STEP.6 集中重绘
#### a.多个对象的绘制过程
```javascript
    setInterval(function(){
        render_a();
    }, 16);
    setInterval(function(){
        render_b();
    }, 16);
    ...
```
#### b.整合在一次绘制中

```javascript
    setInterval(function(){
        render_a();
        render_b();
         ...
    }, 16);
```
#### c.支持复数个对象绘制

```javascript
    setInterval(function(){
        items.each(function(index, draw){
            draw();
        });
    }, 16);
    ...
```
#### d.场景对象

```javascript
var Scene = function(){
    var items = [];
    setInterval(function(){
        items.each(function(index, draw){
            draw();
        });
    }, 16);
    this.add = function(){...}
    this.remove = function(){...}
}
```
> Question-6:不停的重绘对于, dom节点的高频操作会导致严重的性能问题, 如何解决?

### STEP.7 脏属性

每次重绘只处理变化了的属性,对于不变的属性略过(js的逻辑判断相比于dom节点属性设置,性能损耗小太多了)

```javascript
draw:function(){
    if(this.top_changed){
        this.elem.style.top = this.top + 'px';
    }
    if(this.left_changed){
        this.elem.style.left = this.left + 'px';
    }
}
```

### STEP.8 结构解耦,可替换的渲染器

要实现该种逻辑,需要将dom节点与绘制元素对象分离,抽象出`数据对象`和`渲染器对象`,除了更好的分离数据和`dom`节点,更可以方便的替换`渲染器对象`实现不同的渲染方式


```javascript
    //简单的数据对象
    var CustomImage = function(x, y, src){
        this.src = src;
        this.left = x;
        ...
    }
    
    //Dom型渲染器
    var DomRender = function(){
        ...
        var context = document.createElement('div');
        this.draw = function(data){
            var img = new Image();
            img.src = data.src;
            if(data.left_changed){
                 img.style.left = data.x + 'px';
            }
            ...
            context.appendChild(img);
        }
    }
    
    //Canvas型渲染器
    var CanvasRender = function(){
        ...
        var canvas = document.createElement('canvas');
        var context = canvas.getContext('2d');
        this.draw = function(data){
            if(data.left_changed){
                context.clearRect();
                var img = new Image();
                img.src = data.src;
                ...
                context.drawImage(img, x ...);
            }

        }
    }
    
    //useage
    var img = new CustomImage(...);
    var render = new DomRender();   //new CanvasRender();
    render.draw(img)
```
