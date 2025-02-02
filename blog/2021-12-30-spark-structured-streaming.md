---
slug: spark-structured-streaming
title: 如何支持的 Spark StructuredStreaming
tags: [Spark, StructuredStreaming]
---

# Seatunnel 最近支持的 StructuredStreaming 怎么用

### 前言

StructuredStreaming是Spark 2.0以后新开放的一个模块，相比SparkStreaming，它有一些比较突出的优点：<br/> &emsp;&emsp;一、它能做到更低的延迟;<br/>
&emsp;&emsp;二、可以做实时的聚合，例如实时计算每天每个商品的销售总额；<br/>
&emsp;&emsp;三、可以做流与流之间的关联，例如计算广告的点击率，需要将广告的曝光记录和点击记录关联。<br/>
以上几点如果使用SparkStreaming来实现可能会比较麻烦或者说是很难实现，但是使用StructuredStreaming实现起来会比较轻松。
### 如何使用StructuredStreaming
可能你没有详细研究过StructuredStreaming，但是发现StructuredStreaming能很好的解决你的需求，如何快速利用StructuredStreaming来解决你的需求？目前社区有一款工具 **Seatunnel**，项目地址：[https://github.com/apache/incubator-seatunnel](https://github.com/apache/incubator-seatunnel) ,
可以高效低成本的帮助你利用StructuredStreaming来完成你的需求。

### Seatunnel

Seatunnel 是一个非常易用，高性能，能够应对海量数据的实时数据处理产品，它构建在Spark之上。Seatunnel 拥有着非常丰富的插件，支持从Kafka、HDFS、Kudu中读取数据，进行各种各样的数据处理，并将结果写入ClickHouse、Elasticsearch或者Kafka中

### 准备工作

首先我们需要安装 Seatunnel，安装十分简单，无需配置系统环境变量

1. 准备Spark环境
2. 安装 Seatunnel
3. 配置 Seatunnel

以下是简易步骤，具体安装可以参照 [Quick Start](/docs/quick-start)

```
cd /usr/local
wget https://archive.apache.org/dist/spark/spark-2.2.0/spark-2.2.0-bin-hadoop2.7.tgz
tar -xvf https://archive.apache.org/dist/spark/spark-2.2.0/spark-2.2.0-bin-hadoop2.7.tgz
wget https://github.com/InterestingLab/seatunnel/releases/download/v1.3.0/seatunnel-1.3.0.zip
unzip seatunnel-1.3.0.zip
cd seatunnel-1.3.0

vim config/seatunnel-env.sh
# 指定Spark安装路径
SPARK_HOME=${SPARK_HOME:-/usr/local/spark-2.2.0-bin-hadoop2.7}
```

### Seatunnel Pipeline

我们仅需要编写一个 Seatunnel Pipeline的配置文件即可完成数据的导入。

配置文件包括四个部分，分别是Spark、Input、filter和Output。

#### Spark

这一部分是Spark的相关配置，主要配置Spark执行时所需的资源大小。

```
spark {
  spark.app.name = "seatunnel"
  spark.executor.instances = 2
  spark.executor.cores = 1
  spark.executor.memory = "1g"
}
```

#### Input

下面是一个从kafka读取数据的例子

```
kafkaStream {
    topics = "seatunnel"
    consumer.bootstrap.servers = "localhost:9092"
    schema = "{\"name\":\"string\",\"age\":\"integer\",\"addrs\":{\"country\":\"string\",\"city\":\"string\"}}"
}
```

通过上面的配置就可以读取kafka里的数据了 ，topics是要订阅的kafka的topic，同时订阅多个topic可以以逗号隔开，consumer.bootstrap.servers就是Kafka的服务器列表，schema是可选项，因为StructuredStreaming从kafka读取到的值(官方固定字段value)是binary类型的，详见http://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html
但是如果你确定你kafka里的数据是json字符串的话，你可以指定schema，input插件将按照你指定的schema解析

#### Filter

下面是一个简单的filter例子

```
filter{
    sql{
        table_name = "student"
        sql = "select name,age from student"
    }
}
```
`table_name`是注册成的临时表名，以便于在下面的sql使用

#### Output

处理好的数据往外输出，假设我们的输出也是kafka

```
output{
    kafka {
        topic = "seatunnel"
        producer.bootstrap.servers = "localhost:9092"
        streaming_output_mode = "update"
        checkpointLocation = "/your/path"
    }
}
```

`topic` 是你要输出的topic，` producer.bootstrap.servers`是kafka集群列表，`streaming_output_mode`是StructuredStreaming的一个输出模式参数，有三种类型`append|update|complete`，具体使用参见文档http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#output-modes

`checkpointLocation`是StructuredStreaming的checkpoint路径，如果配置了的话，这个目录会存储程序的运行信息，比如程序退出再启动的话会接着上次的offset进行消费。

### 场景分析

以上就是一个简单的例子，接下来我们就来介绍的稍微复杂一些的业务场景

#### 场景一：实时聚合场景

假设现在有一个商城，上面有10种商品，现在需要实时求每天每种商品的销售额，甚至是求每种商品的购买人数（不要求十分精确）。
这么做的巨大的优势就是海量数据可以在实时处理的时候，完成聚合，再也不需要先将数据写入数据仓库，再跑离线的定时任务进行聚合，
操作起来还是很方便的。

kafka的数据如下

```
{"good_id":"abc","price":300,"user_id":123456,"time":1553216320}
```

那我们该怎么利用 Seatunnel 来完成这个需求呢，当然还是只需要配置就好了。

```
#spark里的配置根据业务需求配置
spark {
  spark.app.name = "seatunnel"
  spark.executor.instances = 2
  spark.executor.cores = 1
  spark.executor.memory = "1g"
}

#配置input
input {
    kafkaStream {
        topics = "good_topic"
        consumer.bootstrap.servers = "localhost:9092"
        schema = "{\"good_id\":\"string\",\"price\":\"integer\",\"user_id\":\"Long\",\"time\":\"Long\"}"
    }
}

#配置filter    
filter {
    
    #在程序做聚合的时候，内部会去存储程序从启动开始的聚合状态，久而久之会导致OOM,如果设置了watermark，程序自动的会去清理watermark之外的状态
    #这里表示使用ts字段设置watermark，界限为1天

    Watermark {
        time_field = "time"
        time_type = "UNIX"              #UNIX表示时间字段为10为的时间戳，还有其他的类型详细可以查看插件文档
        time_pattern = "yyyy-MM-dd"     #这里之所以要把ts对其到天是因为求每天的销售额，如果是求每小时的销售额可以对其到小时`yyyy-MM-dd HH`
        delay_threshold = "1 day"
        watermark_field = "ts"          #设置watermark之后会新增一个字段，`ts`就是这个字段的名字
    }
    
    #之所以要group by ts是要让watermark生效，approx_count_distinct是一个估值，并不是精确的count_distinct
    sql {
        table_name = "good_table_2"
        sql = "select good_id,sum(price) total,	approx_count_distinct(user_id) person from good_table_2 group by ts,good_id"
    }
}

#接下来我们选择将结果实时输出到Kafka
output{
    kafka {
        topic = "seatunnel"
        producer.bootstrap.servers = "localhost:9092"
        streaming_output_mode = "update"
        checkpointLocation = "/your/path"
    }
}

```
如上配置完成，启动 Seatunnel，就可以获取你想要的结果了。

#### 场景二：多个流关联场景

假设你在某个平台投放了广告，现在要实时计算出每个广告的CTR(点击率)，数据分别来自两个topic，一个是广告曝光日志，一个是广告点击日志,
此时我们就需要把两个流数据关联到一起做计算，而 Seatunnel 最近也支持了此功能，让我们一起看一下该怎么做：


点击topic数据格式

```
{"ad_id":"abc","click_time":1553216320,"user_id":12345}

```

曝光topic数据格式

```
{"ad_id":"abc","show_time":1553216220,"user_id":12345}

```

```
#spark里的配置根据业务需求配置
spark {
  spark.app.name = "seatunnel"
  spark.executor.instances = 2
  spark.executor.cores = 1
  spark.executor.memory = "1g"
}

#配置input
input {
    
    kafkaStream {
        topics = "click_topic"
        consumer.bootstrap.servers = "localhost:9092"
        schema = "{\"ad_id\":\"string\",\"user_id\":\"Long\",\"click_time\":\"Long\"}"
        table_name = "click_table"
    }
    
    kafkaStream {
        topics = "show_topic"
        consumer.bootstrap.servers = "localhost:9092"
        schema = "{\"ad_id\":\"string\",\"user_id\":\"Long\",\"show_time\":\"Long\"}"
        table_name = "show_table"
    }
}

filter {
    
    #左关联右表必须设置watermark
    #右关左右表必须设置watermark
    #http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#inner-joins-with-optional-watermarking
    Watermark {
              source_table_name = "click_table" #这里可以指定为某个临时表添加watermark，不指定的话就是为input中的第一个
              time_field = "time"
              time_type = "UNIX"               
              delay_threshold = "3 hours"
              watermark_field = "ts" 
              result_table_name = "click_table_watermark" #添加完watermark之后可以注册成临时表，方便后续在sql中使用
    }
    
    Watermark {
                source_table_name = "show_table" 
                time_field = "time"
                time_type = "UNIX"               
                delay_threshold = "2 hours"
                watermark_field = "ts" 
                result_table_name = "show_table_watermark" 
     }
    
    
    sql {
        table_name = "show_table_watermark"
        sql = "select a.ad_id,count(b.user_id)/count(a.user_id) ctr from show_table_watermark as a left join click_table_watermark as b on a.ad_id = b.ad_id and a.user_id = b.user_id "
    }
    
}

#接下来我们选择将结果实时输出到Kafka
output {
    kafka {
        topic = "seatunnel"
        producer.bootstrap.servers = "localhost:9092"
        streaming_output_mode = "append" #流关联只支持append模式
        checkpointLocation = "/your/path"
    }
}
```

通过配置，到这里流关联的案例也完成了。

### 结语
通过配置能很快的利用StructuredStreaming做实时数据处理，但是还是需要对StructuredStreaming的一些概念了解，比如其中的watermark机制，还有程序的输出模式。

最后，Seatunnel 当然还支持spark streaming和spark 批处理。
如果你对这两个也感兴趣的话，可以阅读我们以前发布的文章《[如何快速地将Hive中的数据导入ClickHouse](2021-12-30-hive-to-clickhouse.md)》、
《[优秀的数据工程师，怎么用Spark在TiDB上做OLAP分析](2021-12-30-spark-execute-tidb.md)》、
《[如何使用Spark快速将数据写入Elasticsearch](2021-12-30-spark-execute-elasticsearch.md)》

希望了解 Seatunnel 和 HBase, ClickHouse、Elasticsearch、Kafka、MySQL 等数据源结合使用的更多功能和案例，可以直接进入官网 [https://seatunnel.apache.org/](https://seatunnel.apache.org/)

## 联系我们
* 邮件列表 : **dev@seatunnel.apache.org**. 发送任意内容至 `dev-subscribe@seatunnel.apache.org`， 按照回复订阅邮件列表。
* Slack: 发送 `Request to join SeaTunnel slack` 邮件到邮件列表 (`dev@seatunnel.apache.org`), 我们会邀请你加入（在此之前请确认已经注册Slack）.
* [bilibili B站 视频](https://space.bilibili.com/1542095008)
