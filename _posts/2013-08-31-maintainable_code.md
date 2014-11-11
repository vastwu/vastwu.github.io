---
layout: post
title: 关于代码的可维护性
---


之前看到出了这么本书就兴冲冲的去定了,到手一看发现挺薄的,一天多点看完了,有些错字和翻译歧义,但对我这个水平的人来说收获还是蛮大的,大致总结下经验


之前总是强调要编写易于维护的代码,其实并没有什么实感,仅局限于缩进,命名,格式的风格上,在逻辑上总觉得抓不住可以指导实践的思路,
过于随性,看完书以后总结了一些,以组件的实现简单大致列几项我的原则


对于内部互相调用的公开接口,不要传递内部属性
--------------------------------------------------------------------------

~~~javascript

    var Dialog = function(param1, param2, config){
        var defaultConfig = {};
        this.config = config || defaultConfig;
        //some thing
        this.open(param1, param2, this.config);
    };
    //此处为了方便改个有区别的参数名
    Diaglog.prototype.open = function(param1, param2, open_config){
        // do something with open_config
    };
~~~

上面这段在我的代码里也经常出现,乍看没啥问题,仔细想一下,open接口允许被外部调用,
并且需要依赖于open_config参数的某些数值,但从Dialog的定义看来,config项并非必须,
并且有default的存在,就会有问题

* 只要在实例化的过程中传递了config项,那么所有defaultConfg都会失效(可以通过非覆盖拷贝的方式解决),
本质问题是,open接口难以被测试,defaultConfig对外部来书应该是隐藏的,但是需要测试open接口在不提供/提供部分配置项时是否能正常工作的时候,
是无法应用到defaultConfig的,这个在实例化的过程中已经被确定了

~~~javascript

    var Dialog = function(param1, param2, config){
        this.defaultConfig = {};
        this.config = fill(this.defaultConfig, config || {});
                //some thing
        this.open(param1, param2);
    };
    //此处为了方便改个有区别的参数名
    Diaglog.prototype.open = function(param1, param2, open_config){
        var this_config = fill(this.defaultConfig, open_config || {});
                // do something with open_config
    };
~~~

改为上面这种处理方式,即每个接口都重新要进行一遍填充拷贝的方式来获取正确的配置项,或是禁止在open中传递配置项, 在open内部直接通过`this.config`方式使用配置项


dom事件接口的暴露方式
--------------------------------------------------------------------------

~~~javascript

    var Dialog = function(){
                //some thing
        var closeBtn = document.createElement('div');
        closeBtn.addEventListener('click', function(){
            self.parentElem.style.dislpay = 'none';
        });
                //some thing
    };
    Dialog.prototype.close = function(){
        this.parentElem.style.dislpay = 'none';
    };
~~~

这段代码的问题在于,close按钮的逻辑被完全封闭了起来,外部代码除了模拟点击外没有办法尝试去触发这个close的操作,也就无从验证关闭操作是否正确,
close的由于也需要暴露接口,那么关闭的真实逻辑被分解成了两部分,等组件规模扩大,close中需要增添额外的逻辑时,就比较麻烦了


对参数的依赖
--------------------------------------------------------------------------

~~~javascript

    parentElem.addEventListener('click', function(e){
        var target = e.target;
        //获取真实target过程略
        var index = target.getAttribute('index');
        if(index){
            var data = this.data[index];
            handler(data);
        }           
    });
~~~

为一个列表绑定点击代理事件,在父元素上绑定事件,操作者点击其中一项后,根据该项dom所附带的index属性,从数据中获取某一项,
在传递给真是要执行的函数.这种情况下,handler变得很脆弱,其依赖于event和target属性,且参数为私有的`data`中的某一项,
handler自身是比较难于独立测试.故此种情况下只需要将index传入handler,由handler去进行`data`数据的获取,这样handler所需要的仅仅是index,
且这个index就很容易模拟了

