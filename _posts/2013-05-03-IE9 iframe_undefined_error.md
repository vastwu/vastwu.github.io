---
layout: post
title: IE9 iframe中原生对象未定义错误原因
---

#### 起因
今天碰到个朋友发现ie9会报"Date"未定义（如图），帮其定位问题时发现的

`SCRIPT5009:"Date 未定义"`

#### 排查
先行google了一下，发现有很多类似问题提到了与`iframe`有关，于是便询问了下对方是否有使用`iframe`，确有，接下来发现该问题产生与操作`iframe`有关，大概整理了下，

<!--more-->

#### 复现

父页面

    <head>
        <meta charset="utf-8" />
    </head>    
    <body>
        <iframe id="xyz" src="ifm.html"></iframe>
        <div id="other"></div>
    </body>

    <script type="text/javascript">
        var ifm = document.getElementById('xyz');
        document.body.removeChild(ifm);
        //document.getElementById('other').appendChild(ifm);
    </script>


iframe页面


    <script type="text/javascript">
        alert(1);
        var a = new Date();
    </script>

在ie9下加载父页面，会报错，提示`Date未定义`，更换iframe的节点位置也会报错


#### 原因分析

<!--more-->

ie9在页面被关闭时会回收所有的内置对象和本地对象，类似下面的所有对象都会无法访问

* Object
* Function
* Array
* String
* Boolean
* Number
* Date
* RegExp
* Error
* EvalError
* RangeError
* ReferenceError
* SyntaxError
* TypeError
* URIError
* decodeURI()
* decodeURIComponent()
* encodeURI()
* encodeURIComponent()
* escape()
* eval()    
* getClass()    
* isFinite()    
* isNaN()    
* Number()    
* parseFloat()    
* parseInt()    
* String()
* unescape()

iframe的加载需要时间，即便是本地文件也会是异步处理，需要当前页处理完毕后才行执行，但此时iframe的请求已经准备好并发出，父页面的代码中一旦有操作iframe的行为，会导致iframe准备的运行环境被回收，而之后回来的请求会发现所需的对象已经不存在了，随后就会报错

还有一种类似的场景是iframe中包含着其他的js文件，这些js文件从请求发出到执行也需要时间，在请求发出后的时间内如果ifame被变动了，这些请求成功后的运行环境中仍然没有内置对象和本地对象，但是像`alert`这种还可以被使用。

结论就是不要轻易变更iframe的位置，即便要变动也请确保内容稳定之后再进行处理
