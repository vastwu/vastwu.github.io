## linux安装配置elk日志系统

elk由三大部分组成

#### elasticsearch
搜索引擎，负责筛选和查找过滤

#### logstash
日志收集整理，另外负责输出，可以近似理解为由客户端到搜索引擎的传输通道

#### kibana
日志分析可视化系统，简单说就是一个web网站，负责制定监控图表和可视化日志搜索

### 下载
以下安装全部采用最新的beta版本安装，要玩就玩最新的嘛，这三个必须要搭配使用，和旧版本均不兼容

[Download页](https://www.elastic.co/downloads)

* Elasticsearch 5.0.0
* Logstash 5.0.0 （需要java8或更高版本）
* Kibana 5.0.0 （基于node构建，安装包内自带node环境）

### 配置 + 启动
这三个系统都无需安装，配置和启动都非常傻瓜
#### Logstash
需要自己写一个配置文件`logstash.conf`，作为中间传输灌倒，自然是要用来指明输入和输出方式的



##### 一个简单的配置文件
```yml
# 定义logstash的输入方式都有哪些，
# 指定之后，logstash启动后会自动监听相应的端口
# 支持多种输入和输出
input {
  # 可以从redis中导入数据
  redis {
    host => "10.103.16.66"     # redis主机地址
    port => 16379              # redis端口号
    db => 1                  # redis数据库编号
    data_type => "list"          # 使用发布/订阅模式
    key => "app_debug"        # 发布通道名称
  }
  # 可以通过tcp连接的方式导入
  tcp {
    codec => "json"
    port => 5959
    data_timeout => 10
    type => "tcp-input"
  }
}
# 定义输出方式
output {
  # 输出到文件
  file {
    # 指定写入文件路径
    path => "/Users/xxx/logTools/log/logstash.log"     
    # 指定刷新间隔，0代表实时写入
    flush_interval => 0                                     
  }
  # 输出到搜索引擎，不配置意味着全部采用默认参数
  elasticsearch {
  }
  # 输出到控制台，一般用来调试时确保日志信息是否已经发送到了
  stdout {
    codec => rubydebug
  }
}
```
> 更多的配置项可以看这里[https://www.elastic.co/guide/en/logstash/current/index.html](https://www.elastic.co/guide/en/logstash/current/index.html)，提供了非常多的输入和输出方式

配置结束之后，使用以下命令直接启动即可

```
bin/logstash -f logstash.conf
```


#### Elasticsearch

这个就更简单了，配置文件在 `config/elasticsearch.yml`，一般情况下可以不做修改直接启动，直接运行`bin/elasticsearch`即可

#### Kibana

配置文件在 `config/kibana.yml`，其中有几项需要修改的
```yml
# 网站的访问端口
server.port: 5601
# 网站的域名，这个若不指定，直接通过ip的方式是不能访问的
server.host: "kibana.xxxxx.com"
# 搜索引擎的端口，http服务默认即使9200
elasticsearch.url: "http://localhost:9200"
```
之后`bin/kibana`启动即可
