---
layout: post
title: ECMA探秘,比较操作符和apply
---

最近打算深入下javascript的源头，就找了ECMAScript来看，全部基于2011年6月的5.1版本


### 关于比较操作

比较操作包括`==`，`!=`， `===`和`!==`，这里先看一下ECMA中如何定义比较这个概念

> `==`操作符的定义原文如下

~~~
The comparison x == y, where x and y are values, produces true or false. Such a 
comparison is performed as follows:
1. If Type(x) is the same as Type(y), then
    a. If Type(x) is Undefined, return true.
    b. If Type(x) is Null, return true.
    c. If Type(x) is Number, then
        i. If x is NaN, return false.
        ii. If y is NaN, return false.
        iii. If x is the same Number value as y, return true.
        iv. If x is +0 and y is -0, return true.
        v. If x is -0 and y is +0, return true.
        vi. Return false.
    d. If Type(x) is String, then return true if x and y are exactly the same sequence of 
        characters (same length and same characters in corresponding positions). 
        Otherwise, return false.
    e. If Type(x) is Boolean, return true if x and y are both true or both false. 
        Otherwise, return false.
    f. Return true if x and y refer to the same object. Otherwise, return false.
2. If x is null and y is undefined, return true.
3. If x is undefined and y is null, return true.
4. If Type(x) is Number and Type(y) is String, 
    return the result of the comparison x == ToNumber(y).
5. If Type(x) is String and Type(y) is Number, 
    return the result of the comparison ToNumber(x) == y.
6. If Type(x) is Boolean, return the result of the comparison ToNumber(x) == y.
7. If Type(y) is Boolean, return the result of the comparison x == ToNumber(y).
8. If Type(x) is either String or Number and Type(y) is Object, 
    return the result of the comparison x == ToPrimitive(y).
9. If Type(x) is Object and Type(y) is either String or Number, 
    return the result of the comparison ToPrimitive(x) == y.
10. Return false.
~~~

大致解释下，从上到下可以理解为优先级判定，一旦匹配了return就是获得了结果，不再执行下面的判断

* 1中定义了x,y同数据类型时如何比较，
    * 当x类型为Undefined，Null时，返回true
    * 当x为Number类型时
        * x或y任一是NaN，返回false
        * x或y相等，或是+0,-0之类的数字，返回true
        * 其他情况返回false，（其实没什么其他情况了）
    * 当x为String类型时，如果x和y有同样的长度，且在同样位置有同样的字符，则返回true，否则为false。
    * 当x为bool类型时，如果x和y都是true或false时，返回true，否则返回false
    * 当x为上面之外的其他类型时，若x和y引用自同一对象，则返回true，否则false
* 2,3中定义了undefined与null比较时返回true
* 4,5,6,7中的意思为，若x,y其中一者为String，Boolean类型时，会将其向Number方式转换，然后进行比较。
* 8,9的意思为，若x,y其中一者为String或者Number，且另一个为Object，则将Object类型的值进行`ToPrimitive`操作后在进行比较，这里字面意思为转换为原始类型，具体内容比较多放在后面说
* 如果以上条件都没有命中，则返回false 

这里对实际应用中最后指导意义的就是会将String和Boolean向Number方式转换，我们有时会定义一些常量，然后使用这些常量在程序里进行分支控制，那么这些常量的值可以尽可能使用Number类型，免去了比较时的转换过程，且尽可能早的命中return条件，减少判断次数。

全等于(===)则是少了类型转换那一块，并且在第一个条件判断上就是
> If Type(x) is different from Type(y), return false.

类型不同则直接return false

### 一直困扰我的`Function.prototype.apply`的第二个参数

关于apply常规的使用方式就不细说了，第二个参数为数组，在执行时会被当做参数列表传给Function的实例。

但是js中有一些具有非Array类型的对象也可以被apply正确的执行，比如Arguments，`getElementsByTagName`的返回值（NodeList），ToucheList这些东西，并不是有`Array`的子类或实例，也没有类似pop，push的成员方法，有时候我们有需要将其转化为真正的数组对象，我曾经用过 `Array.prototype.push.apply(x, y)`的方式将y(NodeList)转换成真正的数组，（x为空数组，该语句执行之后为拥有y内所有元素的Array实例），这在理论上是矛盾的呀，于是带着疑问优先看了Function.prototype.apply的定义，

> 先上原文
~~~
15.3.4.3 Function.prototype.apply (thisArg, argArray)
When the apply method is called on an object func with arguments thisArg and argArray, 
the following steps are taken:
1. If IsCallable(func) is false, then throw a TypeError exception.
2. If argArray is null or undefined, then
    a. Return the result of calling the [[Call]] internal method of func, 
        providing thisArg as the this value and an empty list of arguments.
3. If Type(argArray) is not Object, then throw a TypeError exception.
4. Let len be the result of calling the [[Get]] internal method of argArray
    with argument "length"
5. Let n be ToUint32(len).
6. Let argList be an empty List.
7. Let index be 0.
8. Repeat while index < n
    a. Let indexName be ToString(index).
    b. Let nextArg be the result of calling the [[Get]] internal method of argArray
        with indexName as the argument.
    c. Append nextArg as the last element of argList.
    d. Set index to index + 1.
9. Return the result of calling the [[Call]] internal method of func, 
    providing thisArg as the this value and argList as the list of arguments.
~~~

这里最关键的是8，对于从argArray中取值的方式

* 8.若index小于n，则一直重复，n在4中定义为 [[Get]] "length"属性，并转为Uint32，7中定义了index初始值为0
    * a 声明 indexName = ToString（index）    
    * b 声明 nextArg为 从 argArray 中 [[Get]] indexName 的值，
    * c 将nextArg作为argList的最后一个元素加入
    * d index = index + 1

其中的argList就是最终传给Function实例的参数列表
这样我们就可以看出，只要能被index和index+1枚举的属性，并且该值要小于length，即可以被apply执行和使用

~~~javascript
var x = {
    "0":"xxx",
    "1":"yyy",
    "2":"zzz",
    "length":3
};
var test = function(arg1, arg2, arg3){
    debugger
    console.log(arg1 + arg2 + arg3);
};

test.apply(null, x);
~~~

于是就尝试这些了这么一段奇葩代码，果然，test正确的传入了三个参数，随后也就可以解释为什么NodeList等一些类似数组的类型可以被apply使用了，他们同样也可以被index枚举。