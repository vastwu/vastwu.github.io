---
layout: post
title: webapp视频导航播放调研
tags: 动画 3D
category: webgl
image: /images/earth_roll.jpg
---

## 根据sohu和youku的视频,做了下调研,结论如下

* 分端视频采用`HLS (Http Live Streaming)`实现
* 原理是`html5`标签的`src`引用一个`.m3u8`文件, 该文件内容为描述播放列表,一个简单的`.m3u8文件格式如下`
 
~~~
#EXTM3U
#EXT-X-TARGETDURATION:30
#EXT-X-VERSION:3
#EXTINF:4,
video/ts/a.ts
#EXTINF:4,
video/ts/b.ts
#EXT-X-ENDLIST
~~~

* 每行解释如下:
	1. 固定的头信息
	2. 每个分段视频文件的最大长度(秒)
	2. 描述版本类型(固定3即可)
	3. 从本行开始, 当前行和下一行为一对描述信息,当前行描述当前分段的持续时间(秒),下一行为改分段视频的地址, 更多的分段在后面成对添加即可
	4. 最后一行为结尾结束标记

* 该列表格式需要提供`.ts`的视频压缩类型, 跟大全确认了下我们可以生成, 文件尺寸上会比`mp4`略大
* 该方式直接支持分段下载,播放,拖拽,加速等功能,只要生成了正确的`.m3u8`文件,前端无需关注针对视频的操作, 仅支持开始播放即可.
* 该方式支持`ios`和`较新的android`系统,在低端机上未做测试
* 应用该方案需要提供一个可以根据参数返回`.m3u8`文件内容的接口, 且该接口可以正确的拼接视频地址和视频时长
