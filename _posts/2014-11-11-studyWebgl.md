---
layout: post
title: webgl学习经验整理(一)
tags: 动画 3D
category: webgl
image: /images/earth_roll.png
---

# 大纲

* 绘制流程概述
* 绘制流程详解
    * 创建绘制环境
    * 准备着色器(shader)
    * 组织模型数据
    * 上传到GPU
    * 管线渲染
* 动画设计
    * 绘制刷新
* 性能优化

<!--more-->


# 绘制流程详解

## 创建绘制环境

首先需要一个`canvas`,`webgl`依然是使用canvas来作为画布进行绘制,只不过是通过`getContext`其他参数来获取另一种绘制环境

~~~javascript
var gl;
function initGlContext(canvas){
    var gl;
    try{ 
        gl = canvas.getContext('experimental-webgl'); 
    }catch(e){}
    if(!gl){
        return null;
    }
    return gl;
}
~~~

随后对获取的绘制环境做一些初始化操作,此处的只是示例,不用太深究具体含义

~~~javascript
//启用深度测试
gl.enable(gl.DEPTH_TEST);
gl.depthFunc(gl.LESS);
//激活面剔除
gl.enable(gl.CULL_FACE);
//启用融合
gl.enable(gl.BLEND);
//融合方式
gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
~~~

到此为止基本的绘制环境就完工了,此处可以看到webgl很多配置的常量均是在绘制对象下面,意味着如果脱离该对象则可能无法进行参数设置,
这个会给后续的代码结构造成非常恶心的影响,但是也有办法来绕个圈解决,稍后再说

## 准备着色器

着色器可以简单的理解为,告诉计算机如何绘制某个图形,其中包括顶点着色器和片段着色器
webgl中的着色器是一种可编程着色器,虽然麻烦了不少,但是提供了更多的可控性质

### 创建着色器
   
着色器分为两种,顶点着色器和片元着色器,前一种用来描述如何绘制顶点,后一个用来描述如何绘制光栅化之后的片元,也可以理解为绘制像素.

#### 一个简单的顶点着色器

~~~javascript
attribute vec3 aVertexPosition;
attribute vec4 aVertexColor;
varying vec4 vColor;
void main(void){
    gl_Position = vec4(aVertexPosition, 1.0);
    vColor = aVertexColor;
}
~~~

该着色器包含了三个变量和一个入口函数

* attribute 表示该变量是可以由js传入的,后面的`vec3/vec4`则表示该变量的类型, 最后是变量名
* varying 表示该变量是会被传递到片元着色器中
* main 表示入口函数
    * 第一行 gl_Position 为内置变量,用来表达绘制位置,该变量需要一个4分量的数据对象,此处用aVertexPosition 与常量1.0 构建.
    * 第二行 表示将js提供的vVertexColor变量传递给vColor,稍后会再继续传递给片元着色器

以上代码完成了表达顶点坐标和传递顶点颜色的工作,在实际绘制中,每个顶点均会经过该顶点着色器处理

#### 一个简单的片元着色器

~~~javascript
precision mediump float;
varying vec4 vColor;
void main(void){
    gl_FragColor = vColor;    
}
~~~

* 第一行表示绘制精度,先不详细说,知道必须包含次行就是了
* 第二行表示接收顶点着色器传递过来的变量,此处命名必须雨顶点着色器中的一致,并且类型为varying
* main函数中,将接收到的vColor传递给`gl_FragColor`, `gl_FragColor`为内置变量,决定如何渲染每个片元的颜色

### 如何在js中创建一个顶点着色器对象

source就是上面的着色器代码的字符串,我们可以用`\n\n`来连接上面每一行,以便于报错时确定行号

~~~javascript
//创建着色器对象
var shader = gl.createShader(type);
//绑定顶点着色器代码
gl.shaderSource(gl.VERTEX_SHADER, source); // 片元着色器使用 gl.FRAGMENT_SHADER 创建
//编译着色器
gl.compileShader(shader);
~~~

经过编译后的着色器就可以使用了,要想用在gl的渲染上,还需要一个中间环节,链接程序对象

### 程序对象

javascript, shader和gl之间通过程序对象关联起来,一个program绑定一组着色器(顶点和片元), 
gl每次绘制时,会根据当前所选用的program对象的着色器进行操作,

~~~ javascript
//创建程序对象
var shaderProgram = gl.createProgram();
//绑定着色器
gl.attachShader(shaderProgram, vShader);
gl.attachShader(shaderProgram, fShader);
//链接编译
gl.linkProgram(shaderProgram);
//指定gl使用该对象
gl.useProgram(glProgram.program);
~~~

至此为止,就准备好了一个可以用来绘制简单图形的webgl-context环境



