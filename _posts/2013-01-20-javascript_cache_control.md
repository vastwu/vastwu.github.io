---
layout: post
title: js中的内存控制
category: javascript
---

之前是看seajs的那个变态的测试用例，用自己的加载器去尝试运行，一开始就浏览器崩溃崩溃，后来在不断地挖掘和学习过程中，现在终于也能顺利地跑下来啦，这期间有一些多多少少的收获，连自己沉淀带分享，希望能给大家带来些帮助

<!--more-->

先要准备一些辅助测试的函数

~~~javascript
var global = {};
//创建一个指定尺寸的字符串，用于夸张化内存消耗
global.create = function(size){
    var t = new Array(size);
    var x = t.join('a');
    t = null;
    size = null;
    return x;
};
//模拟一个异步的ajax请求
global.ajax = function(arg, callback){
    setTimeout(function(){
        callback && callback(arg);
    }, 100);
};
/*
    用于创建一个占用大内存的递归结构对象，
    通过name来识别当前的this，
    通过level来创建不同的深度
 */
var Model = function(name, level){
    this.name = name;
    if(level > 0){
        this.subs = [];
        level--;
        for(var i = 0; i < 30; i++){
            this.subs.push(new Model(level + '_' + i,level));
        }
    }
}
/*
    填充这个对象及其子成员，来增加该对象的内存消耗
    以便更明显的识别内存是否释放
 */
Model.prototype.init = function(size){
    this.cacheSize = global.create(size || 10240);
    if(this.subs){        
        for(var i = 0, n = this.subs.length; i < n; i++){
            this.subs[i].init();
        }
        this.finish();    
    }
};
/*
    在某些场合保持对module实例的引用，需要让他干点什么（实际什么都没干）
 */
Model.prototype.play = function(){
    
};
~~~

测试方式，通过chrome的`profile`的`Take Heap Snapshot`内存快照来对比，在不同debugger断点情况下，内存的占用情况，由于chrome在进行快照前会自动进行一次GC，为了模拟真实的环境，在断点后面要保留对所需数据的引用，真实环境中是对数据的操作。

##### 基本数据
* 没有进行过js操作的chrome空白页面，内存占用大约在1.9M左右
* 一个被填充过的，level=2的Model对象占用内存在20M左右

### 关于ajax和异步函数，你真的需要一个完整对象的引用么

~~~javascript
function test2(){
    var m = new Model('root', 2);   //创建对象
    m.init();                       //填充尺寸
    debugger                        //此时的内存大约在21M左右
    /*第1种写法*/
    global.ajax(m, function(arg){
        /*
            这里的m对象被完整的保留了下来,arg即是m的引用
            内存占用依旧在21M
         */
        debugger                    
        var x = arg;
    });
    /*第2种写法*/
    global.ajax(null, function(arg){
        /*
            m依旧被好留，回调函数内存在引用
            内存占用依旧在21M
         */
        debugger
        var x = m.name;
    });
    /*第3种写法*/
    global.ajax(m.name, function(arg){
        /*
            m被释放，arg仅仅是m.name，一个字符串的引用
            内存占用降低至2M
         */
        debugger
        var x = arg;
    });
    /*第4种写法*/
    var _name = m.name;
    global.ajax(null, function(arg){
        /*
            m被释放，回调的作用域内仅保留了_name属性，是对m.name的引用，与m无关
            内存占用降低至2M
         */
        debugger
        var x = _name;
    });
}
test2()
~~~
上面类似的代码我相信大家一定都写过，在真实场景中，未必需要整个m对象，在回调函数中呢能仅仅需要部分数据，这样的情况下，我们是完全可以释放掉m所占用的内存，也许有人会觉得这种写法差不太多呀，回调函数执行完毕之后，也就自然会释放m了，你这个例子是卡在正好内存不释放的情况，接下来看下面的写法

~~~javascript
function test3(){
    //closure
    for(var i = 0; i < 5; i++){
        var m = new Model('root' + i, 2);
        m.init();
        global.ajax(m, function(arg){
            debugger
            var x = arg;
            console.log(x.name);
        });
    }
}
test3();
~~~

![mem1](https://f.cloud.github.com/assets/3717980/679785/c577f5da-d97a-11e2-913a-65e8cf5ae05a.png)

这个例子中，当第一个debugger被触发时，会发现内存暴涨到了93M，也就是5个m全部被保留了下来，当多个异步操作同时被触发时，每个闭包内的变量引用都会被保留，即使回调没有被执行，这种情况下，首个返回的请求的回调函数，会在有5个Module实例占用内存的情况下执行，第二个回调会在有4个Module实例占用内存的情况下执行，以此类推，这其实并不是我们想要的状态，所以，在异步操作或闭包中，仅保留需要的变量是非常重要的，不要小看这些零碎变量，多起来就会造成内存的堆积，在函数运行期拖慢速度。

### 及时释放那些你确实不需要的变量
首先为global拓展一个简单的模块管理器，来进行下面的测试,
管理器遵循惰性原则，即当模块第一次被使用时，才会执行factory，导出接口，
~~~javascript
global.modules = {};
global.require = function(name){
    var m = global.modules[name];
    if(!m.export){
        m.export = m.factory();
    }
    return m.export;
}
global.define = function(name, factory){
    global.modules[name] = {
        factory:factory,
        export:null
    };
}
//usage
var x = global.require('a');
~~~

看起貌似没什么问题，但是当define的function内容很大，或者很多时，就会占用很多无谓的内存，factory没有被第二次执行的机会，且一直挂在m上（比如seajs的那个变态的测试用例，大约有上万个模块），这里正确的方式是在require中，已经生成了接口之后，`delete m.factory`，以此类推，所有不需要的对象或属性，及时人道毁灭

### 关于dom节点那点事
~~~javascript
function test4(){
    var m = new Model('root', 2);
    m.init();
    debugger
    var container = document.getElementById('domContainer');    //已经存在的dom节点
    var div = document.createElement('div');
    div.addEventListener('click', function(){
        m.play();
    });
    container.appendChild(div);
    //delete
    //container.removeChild(div);
    container.innerHTML == '';
    debugger
}
test4();
~~~
如果你喜欢用暴力的`innerHTML`方式去清空节点，那么如果有个大对象引用的话，就要和内存说再见了，上文的写法中，即使智慧如chrome也无法回收m的引用，原本在test4函数执行结束之后，m就只有一个div的事件在引用，此时强制删除掉div后，m就变成了引用数为1，且无法再被减少的状态，20M内存妥妥的到天荒地老（刷新页面倒是可以回收，不过ie估计就没这么聪明了），在chrome下用`removeChild`方式不会体现出立刻收回，稍等一会再看内存就降下来了，这里告诉我们，起码要用`removeChild`来移除有事件的元素，最好的处理方式就是先`removeEventListener`

###### 这里顺便说下我对GC的理解，

javascript有五种原始类型，Undefined,Null,Boolean,Number,String，这五种对象是值引用，下面一段中指的对象不包含这5种

抽象的说，`{id:1}`，这是一个对象，会占用一块内存（被系统分配一个标示），`var a = {id:1}`表示javascript的变量a引用了这个对象，可以借此来操作这个对象，`var b = a`时，`{id:1}`引用数变成了2，`var c = a.id + 1`,引用数变成了3，当a,b,c都不再被更多的地方引用时，`{id:1}`这个东西就会被回收，内存被释放;

还是test4的例子，这个函数中一共创建了
`m`,`container`,`div`，元素事件的匿名function（下文简称function），这4个对象，其中`container`和`div`在函数之外不存在引用，所以这三个对象在test4执行结束后即被回收，而绑定事件是将function与div的原始元素（var div仅仅是对原始元素的引用，并非元素本身，js可以通过这个引用操作元素）绑定在了一起，所以div被回收不会影响function，而function中又引用了m，所以通过removeChild的方式移除元素时，元素本身被正常的移除，相关的引用也可以被回收，而`innerHTML`是一种不太正常的吧元素从dom树上抹掉，变成了div的原始元素的内存任然占用，但是无法被任何方式访问到，也就无法释放function和m。

### 小心递归的中间变量
~~~javascript
function test5(){
    var func = function(n){
        var x = new Model('root', 2);
        x.init();
        if(n > 0){
            n--;
            var y = func(n);
            y.play();
        }
        debugger
        return x;
    };
    debugger
    var z = func(4);
    debugger
}
test5();
~~~
这是一个简单的递归调用，n表示执行几次，当n>0时，会递归执行，这里每一次递归的堆栈中都会存在一个x，占用20M内存，当debugger被第一次触发时，是在递归最深层的调用中，同时在上层堆栈中有另外3个x，占用了很多内存，随着递归的逐渐退出，内存被逐渐释放，有点像前面一个异步的例子，这里也基本遵循上文中提到的GC原则，基本把这个理解了就可以很好的了解内存状态了。


参考资料

* [High-Performance, Garbage-Collector-Friendly Code](http://buildnewgames.com/garbage-collector-friendly-code/)
* [seajs的项目代码](https://github.com/seajs/seajs)