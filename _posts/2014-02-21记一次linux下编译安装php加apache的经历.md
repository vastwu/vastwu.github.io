# 记一次linux下编译安装php加apache的经历

### 大概背景
因工作需要,在一台远程服务器上需要部署个apache + php + mysql的环境,犹豫机器上已经包含了apache和mysql,困难之处就在于php上,由于一些工作网络环境的限制,不能使用全局的自动安装工具,也无法操作自身用户之外的目录(`/home/users/vastwu`)

之前从没有类似的经历,算是一次非常好的学习锻炼过程,这里做个总结记录

<!--more-->

### 开工

#### 1.下载代码

[在这里下载php源代码](http://cn2.php.net/downloads.php)

要安装的机器没法直接访问外网,所以通过`ssh`登录后到远程服务器后,从我的机器上拖走源码包

```
xcp vast@my_ip:Download/php5.tar.gz ./
```
解压缩

```
tar zxvf php5.tar.gz
```

#### 2.编译

一开始在网上找到了这种东西

```
cd php-5.3.8
./configure  
	--prefix=/usr/local/php 
	--with-mysql=/usr/local/mysql 
	--with-apxs=/usr/local/apache2/bin/apxs
	--with-libxml-dir=/usr/ local/libxml2
make 
make install 
```
看了下完全不理解是什么东西,按照内容复制下来直接运行必然是报错,后来逐个研究了下,看看到底各个命令都是干什么的,要改成适合我的环境

```
//进入php源码目录
cd php-5.3.8
//设定配置,我需要编译成什么样的东西, 后面的东西需要写成一行, 这里为了注释方便
./configure
	//php的安装目录,由于我没有root权限,所以这里必须要改成一个我能访问的路径  
	--prefix=/home/users/vastwu/local/php
	//启用mysql,这里可以设置很多类似--with--xxxx格式的内容,
	//表明编译时加入某些功能
	--with-mysql 
	//指定机器上的apache服务的路径,会自动对apache进行一些配置
	//这台机器上撞到了比较奇葩的位置,好一顿找
	--with-apxs=/home/users/vastwu/.jumbo/lib/apxs
	//这里是另一个我手动安装的模块的目录,其中libxml2也是通过同样的模式安装,
	//此处可以指定路径,假如一些php源码包中不存在的模块
	--with-libxml-dir=/home/users/wusen/local/libxml2
//编译开始
make 
//编译结束后执行,不同的程序会执行自带的某些脚本,
//php install时会吧文件拷贝到prefix目录之类的操作
make install
```

到这里基本大工程结束,后面则需要进行一点配置工作

#### 3.配置

首先检查httpd.conf的配置,确保加载了php模块

```
LoadModule php5_module /home/users/wusen/.jumbo/lib/httpd/modules/libphp5.so
```

后来出现个问题,我的phpinfo一直无法正确的加载php.ini的配置文件,所以我在这里加了一个配置项来确保路径

```
PHPIniDir "/home/users/wusen/local/php/lib/php.ini"
```

当在`phpinfo`中看到下面信息时,就可以肯定是正确加载了

```
Loaded Configuration File	/home/users/wusen/local/php/lib/php.ini
```

#### 4.修修补补

后来在实际运行中发现好几个问题,导致又回来重新编译`php`

1. 没有加入zlib库,无法使用相关功能,需要在`./configure`加入`--with-zlib`
>当需要某些动态库时,都可以按照这个方式重新编译,当整体环境中还没有某个库时,可以通过`--with-zlib=dir`的方式指定库路径

2. 无法使用`mb_strimwidth`函数,报未定义,需要在`./configure`加入`-enable-mbstring`
>这个说实话不是特别明白,这种开启之类的操作不能动态操作么....反正我找了好久发现只能重新编译

3. 对于更多的扩展项,需要通过在`php.ini`中设置`extension_dir=xxx`来指定目录,然后再添加`extension=xxx.dll`的方式加载需要的功能


###总结
大概搞了这么一遍,对linux下编译安装程序有了个整体了解, 

某些没有依赖的程序, 通过`./configure + make + make install`的方式即可安装, 有依赖的,则先安装好依赖, 在使用`./configure --with`的方式编译安装,

在没有root权限的环境,可以指定在自己的可读写路径下进行操作

鼓捣了一通超有成就感,又学到了新东西


-------------------
### 追加内容

后来又在基础上追加了curl模块，期间各种误解教程，总的来说是这样的

1. 你需要在网上下载curl模块的源码，通过make方式安装到某个指定目录下，此时与php还没有任何关系。
2. 在php源码中的ext目录下寻找curl的目录，此目录中包含一些类似桥接信息的代码，我之前把这块东西误解为php源码中附带了某些扩展模块的源码，直接用这里的文件进行安装编译发现怎么也不行。
3. 在此目录下先调用`phpize`生成`configure`文件。
4. 执行`./configure --with-curl=DIR`，此处的DIR即是前面安装curl的目录，随后`make && make install`即可，最后会提示生成的`.so`文件所在目录，
5. 去`php.ini`里改动`extensions`目录和打开加载`curl.so`即可，这个文件名可能会有些不同，注意下即可，另外window下是`.dll`，linux下是`.so`，这个需要注意下



