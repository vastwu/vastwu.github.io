---
layout: post
title: 科学的姿势理解javascript的变量引用
---

从上学时代起，就被告知计算机中存在值引用和址引用两种方式，其实一直也不是很明白，在js里也一直是模模糊糊，只知道5中值引用，在对象的引用概念上也不是特别确定，最近在追了以下js的内存问题，顺便也就把地址引用的逻辑试了一下。

有时候函数中的参数变量会被其他场景的代码改变，比如一个数组变量用着用着就变了，一看原来是有其他代码在操作，有时候似乎又不会变，要手动去更新，把下面的都看完相信就都会有明确的答案了。

先上实验代码的基本结构，代码都非常简单，有兴趣的朋友可以粘到控制台里玩一玩

~~~javascript
var keep = function(obj){
    var p = obj;
    setInterval(function(){
        console.debug(p);
    }, 1000);
};

var testOne = {a:1,b:2,c:3};  //set testOne

keep(testOne);                   //keep log

setTimeout(function(){
    //do something with testOne
}, 3000);
~~~

原则是，声明一个基本变量`testOne`，将他传递到`keep`函数中，使用`setInterval`定时输出，来观测其变化，在外部，通过一个3秒的延时来改变这个`testOne`，看看在`keep`内部的变量`p`会有怎样的变化

PS：下中文提及的引用类型均为对象或数组，另外的5大原始类型数据均为值引用，即
~~~javascript
var x = "abc";
var y = x;
~~~`
和
~~~javascript
var x = "abc";
var y = "abc";
~~~`
两种写法是等价的，即相当于进行了同样的赋值操作，互相之间不产生任何关联。

#### 示例一
~~~javascript
var keep = function(obj){
    var p = obj;
    setInterval(function(){
        console.debug(p);
    }, 1000);
};

var testOne = {a:1,b:2,c:3};

keep(testOne);

setTimeout(function(){
    //do something with testOne
    testOne = null;
    testOne = {};
    testOne = 123;
}, 3000);
~~~

可以猜一猜答案如何？
结果是控制台仍然保持正确的输出`Object {a: 1, b: 2, c: 3} `;说明`testOne=null`之类的没有对keep内的p产生影响，不是说地址引用嘛，操作了变量之后应该对所有的引用产生影响呀？我们先把所有的例子看完在最后做总结。

#### 示例二
~~~javascript
var keep = function(obj){
    var p = obj;
    setInterval(function(){
        console.debug(p);
    }, 1000);
};

var testOne = {a:1,b:2,c:3};

keep(testOne);

setTimeout(function(){
    testOne.b = 5;
    testOne = null;    
}, 3000);
~~~

答案是`Object {a: 1, b: 5, c: 3} `，试玩之后可能会大叫，这不科学，但是的确事实如此。

下面来看看这是为什么。

先说下我对址引用的理解：

我们都知道变量的值是保存在内存中，我们所使用的变量名仅仅是一份引用地址，举个例子，内存就像一个大仓库，存着各种各样的货物，每个货物都有自己真实的物品，而引用地址就像一张记录了仓库货架号的纸条，当你需要某个货物时，把纸条交给管理员，管理员去取出真正的货物交给你，而无论你把纸条撕了还是改了，都不会对货物本身产生影响.

就像示例一中，testOne是一个记录了货架号的纸条，你把纸条上的数字改成了各种各样的东西，结果是testOne这张纸条的货架号变成了另外一个货物的号码，只不过是随后会取出其他的东西罢了。

`testOne = {}`这句话的意思是，先在仓库里保存一个货物，该货物的内容是`{}`，然后把testOne纸条上的货架号改成了这个刚刚保存的货物的货架号，从此以后，你吧testOne交给管理员时获取的货物就是`{}`，而`keep`中保存的p，就好比是在改写`testOne`之前，在p纸条上又抄了一遍货架号，这张新纸条的内容不会随着后面改写`testOne`而变化，而是始终记录着正确的货架号。

示例二中为什么`testOne.b`又改写成功了呢，当使用`testOne.b`的引用时，相当于将`testOne`编号中的货物取了出来，然后将其中的`b`位置的值改为了5，而下面的`testOne=null`继续是撕纸条的动作，无效。

#### 示例三
~~~javascript
var keep = function(obj){
    var p = obj;
    setInterval(function(){
        console.debug(p);
    }, 1000);
};

var testOne = {
    a:1,
    b:{
        m:2
    },
    c:3
};
keep(testOne);

var x = testOne.b;
setTimeout(function(){
    x.m = 5;
    x = null; 
}, 3000);
~~~
看了结果估计又有人要抓狂了，说好的取出来改掉怎么又没用了，`testOne`是个对象，是个纸条，然后当`var x = testOne.b`时，而b又是一个对象，即地址引用，此时的赋值操作时从`testOne`这个货物中拿出了一张叫做`b`的纸条，把纸条的内容抄写到了`x`上，这下明白了吧，然后依旧遵循上面讲述的规则，撕了或是涂改`x`这张纸条没用，而是顺着`x`上写的货架号拿出货物来才能改写货物的内容。

有人说那我直接写 `testOne.b.m = 5;`可以么，答案是可以，`m`是原始数据类型，即直接赋值操作即可。

追加一段
-------------
上面有个说法明确下，希望不要误导大家，`testOne = null;`这句话并不是销毁（撕掉）`testOne`这个变量，而是将testOne的纸条改写了，改写成了一个`null`内容的内存地址，javascript中的对象和数组是语言层面无法销毁的，我们所能做的就是解除引用，例如下面的代码
~~~javascript
var x = {};
var a = x;
var b = x;

x = null;
a = null;
b = null;
~~~

这个表示，我们在内存中开辟了一块空间，存储了一个内容是`{}`的东西，这个东西在语言层面有三个引用变量，分别是`x`，`a`和`b`，当随着后面吧这三个变量的地址改写到了叫做`null`的后，即取消了对`{}`这个东西的引用，他的引用计数变成了0，那么随后就会被GC机制回收，而并非是简单的将变量设置为`null`就销毁了，这件事情是浏览器来做的，也就是垃圾回收机制。
