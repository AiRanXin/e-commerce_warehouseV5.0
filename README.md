采集Flume
Source:TailDir Source
拦截器：使用ETL拦截器以及提取日志的时间戳放在header中方面落盘，解决由于上报时间和网络抖动造成的数据飘逸问题
选择器：multiplexing
Kafka Channel,采用Kafka Channel，省去了Sink，提高了效率。

Kafka
浏览、点击、停留、曝光 的数据在一个文件夹里，留资、订单关联数据在另外一个文件夹里topic_log 和 topic_order_leads 对应这两个topic文件夹 3分区两副本

消费Flume
Kafka Source
File Channel
HDFS Sink
Flume拦截器的编写
实现Interceptor 接口和Intercepter.Builder接口
