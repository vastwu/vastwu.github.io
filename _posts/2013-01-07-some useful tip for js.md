---
layout: post
title: js中的小技巧总结
---

前一阵面试被问到js代码的性能优化，不结合业务场景一时发现想不起来什么，之前有些做的真是浪费了，这里大致总结下，都不是什么新鲜玩意。

函数的节流和抽稀
==================
抽稀和节流为便于区分个人起的名字，与其他资料中的可能会概念不一致，莫要纠结于名字...

后面可能会提到跑满帧的概念，即setTimeout的延时为n时，该函数在真实世界中能够保持在n时间后执行，延时大致准确，在有大量异步或dom操作的情况下，js可能无法精准的延迟执行函数。

以下两种方案均是为了实现降低值变更引起的频繁操作，人眼所能识别的频率大约是30-40帧左右，一般UI的变化频率能达到40帧，保持合理的变化幅度，即不会有卡顿感，如果将所有操作都跑到满帧，那么可能当多个UI同时发生变化时，会有个别的出现不规律的卡顿，这显然并不是期望的。

#### 函数抽稀

~~~javascript
    var isLocked = false;
    function update(value){
        if(isLocked){
            return;
        }
        isLocked = true;
        // do something with value
        setTimeout(function(){
            isLocked = false;
        }, 50);
    }
~~~

以上为一个最简单的抽稀示例，该函数被外部调用时，只能以最高以50ms/次的频率调用，每一次成功调用均会被锁定，期间所有的调用据会被直接return，50ms后解锁，允许再次被执行

所谓抽稀，就是忽略到高频率调用时的一些中间值，按照某个固定的频率执行具体逻辑

缺点在于函数执行不够精准，如果有一些同样为异步的函数在依赖该update更新的值，会出现偶尔更新到旧值的情况
有点则在于逻辑相对简单，相比锁定状态下会直接短路结束，没有多余的执行。





#### 函数节流

~~~javascript
    var runner = null;
    function update(value){
        if(runner){
            clearTimeout(runner);
        }
        runner = setTimeout(function(){
            // do something with value
            runner = null;
        }, 50);
    }

~~~
节流，即每一次新值的使用，均会阻止掉旧值生效，并延时使用新值，以确保在50ms内持续更新，直至趋近于一个稳定的状态，即50ms内有没有更新时，生效

以上两种方式独立使用都比较麻烦，需要频繁的操作set/clearTimeout,如果项目中有较多的需要，可以参考我的[TimeShaft][TimeShaftUrl]设计一个全局的时间轴控制器，来管理类似的操作


#### 动态分时任务

名字起得有点那啥，切莫介意，由于浏览器处理过于复杂的计算或dom变更时，会引起卡死，失去响应等现象，即便没有这么极端也会出现卡顿等操作，即1+1>2的处理时间，比较特别的例子是地图中海量marker的处理，拖拽时，如果有上百个marker（每个marker有3-4个图片节点）会让浏览器动但不得，这时候实际的界面要求并没有那么高，我们可以退而求其次，降低重绘的量，同样以帧为单位，每一帧内只处理固定数量的任务，多余的任务留倒下一帧中，中等量的任务中不会有太大的视觉差，会看到marker是逐批更新，

~~~javascript
    TimeShaft.add(null, function(d, p, now){
        var n = 0;
        frames++;
        while(points.length){
            point = points.pop();
            var div = createDiv(point.x, point.y);
            container.appendChild(div);
            n++;
            if(new Date() - now > 20){
                log(frames + ':finish in a frame:' + n);
                break;
            }
        }
        if(points.length == 0){
            log('duration:' + (new Date() - begin));
            return true;
        }
    }, 30);
~~~

大致的一个简单demo实现，该函数用到了一个自定义的时间轴组件，上面的写法大致意思是，将function加入时间轴控制器中，并以最高30帧的帧率运行，该函数每帧执行一次，这个函数的功能是根据points数组创建div，并添加到指定位置，分时是体现在那个`new Date() - now > 20`中，

该函数中的while是会处理完所有的点，示例中用了30000个点，不采用分支直接添加，最新版的chrome（ver.27）大约会使用17秒，并该tab页一直处于卡死，不可操作状态，以上的分时中，每一次while的循环均会判断一下当前时间距离该函数的执行时间是否已经超过20ms，实际中可适当调整时间判断间隔，比如每10个point点检查一次，降低new Date()的消耗，当该帧中执行的时间超过20ms后，break，结束当前帧的执行，就是动态的分配一帧中能执行多少次，单帧超时后余下的任务进入下一帧，即会看到非卡死状态下（可能依然很卡，具体参数需要机动调整），浏览器完成了30000个div的创建，大约只用了2.5秒的时间。

由于是基于绝对时间的运行时间，在页面有其他延时任务时，该方法会动态调整每次执行的任务数。可以用于处理一些不那么重要的任务，某些页面效果的过度也许远比不上最终呈现给用户的数据重要，那么期间的一些渲染任务就可以以此方式来进行，宁可跳帧，也不要卡住浏览器

[分时demo点我][tsdemo]
* `show all`：直接添加在页面，不采用任何处理
* `Show All(with tester)`：直接添加在页面，同时运行一个动画效果，确定浏览器的卡死状态
* `Show All(split frame)`：分时添加，可以看到30000个点的逐批添加
* `Show All(split frame with tester)`：分时添加的同时运行动画过程，可以看到虽然很卡，但仍然可以运行

实例请在chrome下运行，打开控制台可以看到每一帧输出的log，表示当前帧内处理了多少个点，发现打开控制台的情况下，处理效率明显下降，可能跟chrome的log实现有关系，可以先关闭控制台，处理结束后再打开看log的输出情况。

这里仅是介绍一种思路，真正应用于生产中，还需要根据实际应用场景做改进


滤波器
==================
对于频繁变更的UI值，我们还可以采用滤波器的方式优化，使其变得平滑。

~~~javascript
    var Filter = function(filterWide){
        this.maxLength = filterWide || 5;
        this.values = [];
    };
    Filter.prototype.update = function(value){
        if(this.values.length >= this.maxLength){
            this.values.shift();
        }
        this.values.push(value);
    };
    Filter.prototype.getAvg = function(){
        var sum = 0, i, n = this.values.length;
        for(i = 0, i < n; i++){
            sum += this.values[i];
        }
        return sum / m;
    };
    Filter.prototype.getMid = function(){
        var s = this.values.slice();
        s.sort(function(a, b){
            return a > b ? 1 : -1;
        });
        return s[parseInt(s.length/2)];
    };
~~~
关于中值滤波的概念，摘一段baidu百科的
> 中值滤波的基本原理是把数字图像或数字序列中一点的值用该点的一个邻域中各点值的中值代替，让周围的像素值接近的真实值，从而消除孤立的噪声点。

在这里即实现的`getMid`,原理是，给定一个滤波范围,实例化的参数，默认为5，每次数值变化时，使用update更新值，要使用值时，用getMid获取最近的5个变化值中，相对居中的数值。
这个单纯使用似乎没什么意义，但是可以结合前面的函数抽稀，我们在每一次UI变化数值时，均更新滤波器的值，但是真正执行时，采用滤波器获取中值或均值，就可以避免当某个瞬间，数值变化过大时，屏幕上的UI出现大幅度变动的情况。

这个曾经用在wap页里处理Android的陀螺仪事件，Android的陀螺仪事件触发频率和精确度都很不理想，直接使用会出现高频率的晃动，转动过程中很不稳定，使用滤波器监听陀螺仪事件更新三个轴的变化，使用一个setInterval固定频率的获取中值，就可以得到相对稳定的陀螺仪变化过程。



对象池
====================
~~~javascript
    var Pool = {
        elems:[],
        create:function(){
            return new Elem();
        },
        clear:function(){

        },
        use:function(){
            if(this.elems.length == 0){
                var e = this.create();
                this.elems.push(e);
                return e;
            }
            return this.elems.pop();
        },
        recover:function(elem){
            this.clear(elem);
            this.elems.push(elem);
        }
    };

~~~
对象池的应用范围比较窄，条件限定多，常用于一些较复杂的对象类型，该对象的初始化函数复杂，且频繁需要多个实例的创建与销毁，希望尽可能减少new的次数，而将弃用的对象清理干净，回收留待重复利用，
这里外部调用者通过use来获取一个Elem对象，该对象可能是首次被创建的，也可能是被回收的，当外部确定弃用一个Elem对象时，使用recover来回收该对象，清理该对象上的特殊属性后，保存在池内，当下次再有调用申请时，在弹出返回共外部调用。当构造函数够简单或回收操作很复杂时，就不建议用这种方式了，要考虑收益和额外的开销是否合算。

之前的一个使用场景是地图上的marker，一个marker上有大量的事件和比较复杂的dom节点，当用户执行一次poi搜索时，会在地图上出现最多9个marker，早先做法是每次new 9个marker，不用时全部清理掉，下次使用继续new 9个，后改为回收的方式，每个marker在回收时隐藏，下次使用时重新设定位置显示即可。

根据实际场景的不同，可以对内部存储的元素固定时间清空等扩展，比如上例中可以添加30秒内没有再使用就清空释放所有的marker

[TimeShaftUrl]: https://github.com/vastwu/TimeShaft
[tsdemo]: http://jscodelib.sinaapp.com/TimeShaft/b.html