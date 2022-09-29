# 电商数仓V5.0 :star:
项目分为数据采集、数仓开发、报表可视化3部分 ,花了一周左右从无到有搭建整个项目,现写一下总结，后续有时间再更新具体项目细节

-----
- **项目需求**
  - 1、用户行为日志采集平台搭建
  - 2、业务数据采集平台搭建
  - 3、数据仓库维度建模
  - 4、分析设备、会员、商品、地区、活动等电商核心主题，统计的报表指标近100个。完全对比中型公司
  - 5、采用即席查询工具·随时进行指标分析
  - 6、对集群性能进行监控，发生异常需要报警
  - 7、元数据管理
  - 8、质量监控


- **集群规划**

![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E9%9B%86%E7%BE%A4%E6%9C%8D%E5%8A%A1%E5%99%A81.png "github")  

- **项目架构**

![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E6%95%B0%E6%8D%AE%E4%BB%93%E5%BA%93%E6%A0%B8%E5%BF%83%E6%9E%B6%E6%9E%84.png "github") 
## 1.数据采集
### 1.1 用户行为日志采集
本项目收集和分析的用户行为信息主要有页面浏览记录、动作记录、曝光记录、启动记录和错误记录，格式为json，编写脚本来使用模拟器产生用户行为数据
- 采集Flume
  - Source:TailDir Source
  - 拦截器：使用ETL拦截器以及提取日志的时间戳放在header中方面落盘，解决由于上报时间和网络抖动造成的数据飘逸问题
  - 选择器：multiplexing
  - Kafka Channel,省去了Sink，提高了效率

- Kafka
  -  浏览、点击、停留、曝光 的数据在一个文件夹里，留资、订单关联数据在另外一个文件夹里topic_log 和 topic_order_leads 对应这两个topic文件夹 3分区两副本

- 消费Flume
  - Kafka Source
  - File Channel
  - HDFS Sink
  - Flume拦截器的编写实现Interceptor 接口和Intercepter.Builder接口
- 用户行为日志采集流程
 ![github]( https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E7%94%A8%E6%88%B7%E8%A1%8C%E4%B8%BA%E9%87%87%E9%9B%86%E6%B5%81%E7%A8%8B.png "github") 

### 1.2 业务数据采集
电商数仓开发涉及34张表，以订单表、用户表、SKU商品表、活动表和优惠券表为中心，延伸出了优惠券领用表、支付流水表、活动订单表、订单详情表、订单状态表、商品评论表、编码字典表退单表、SPU商品表等。选用mysql作为元数据存储，后续需要把数据同步到hive上数仓建模，同步策略有全量同步和增量同步，根据表格数据特点和业务需求合理选择。数据同步工具，全量选择datax，增量选Maxwell，选择理由*****
- 业务数据采集流程
![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E4%B8%9A%E5%8A%A1%E9%87%87%E9%9B%86%E6%B5%81%E7%A8%8B.png "github") 

## 2.数仓建模与开发

### 2.1 数仓建模设计
- 1.数据调研
   - 业务调研，对现有业务划分业务模块，弄清每个业务的业务流程
   -  需求分析，明确需求所需的业务过程及维度

- 2.数据域划分
数据域是指面向业务分析，将业务过程或者维度进行抽象的集合
 
| 数据域 | 业务过程 |
|-----|-----|
| 交易域 | 加购、下单、取消订单、支付成功、退单、退款成功 |
| 流量域 | 页面浏览、启动应用、动作、曝光、错误|
| 用户域 | 注册、登录 |
| 互动域 |	收藏、评价 |
| 工具域 |	优惠券领取、优惠券使用（下单）、优惠券使用（支付）|
	
- 3.构建业务总线矩阵
- 包含维度模型所需的所有事实（业务过程）以及维度，以及各业务过程与各维度的关系

![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E4%B8%9A%E5%8A%A1%E6%80%BB%E7%BA%BF%E7%9F%A9%E9%98%B5%E5%9B%BE.png "github") 

- 4.明细模型设计
	- 原子指标，基于某一业务过程的度量值，对指标的聚合逻辑进行定义，是业务定义中不可再拆解的指标
	- 派生指标与衍生指标

![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E6%B4%BE%E7%94%9F%E6%8C%87%E6%A0%87.png "github") 

![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E8%A1%8D%E7%94%9F%E6%8C%87%E6%A0%87.png "github") 

- 5.汇总模型设计
汇总表与派生指标的对应关系是，一张汇总表通常包含业务过程相同、统计周期相同、统计粒度相同的多个派生指标

### 2.2 数仓开发
- 数仓体系分为5层，设计过程创建90多张表，采用hive on spark
	- ods层，1张日志表，29张业务表
	- dim层，6张维度表
	- dwd层，19张事实表
	- dws层，9张最近1日汇总表，10张最近n日汇总表，3张历史至今汇总表
	- ads层，分流量、用户、商品、交易、优惠卷、活动6个主题共15张表
 ![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E6%95%B0%E4%BB%93%E6%80%BB%E8%A1%A8%E6%95%B0.png "github") 

## 3.报表可视化

### 3.1 DolphinScheduler任务调度
 DolphinScheduler对项目中的数据采集和数仓开发工作流进行监控，设置异常报警
![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/%E5%B0%8F%E6%B5%B7%E8%B1%9A%E8%B0%83%E5%BA%A6.png "github") 
### 3.2 superset报表可视化
调用ads层的数据进行可以化，superset是web UI，可能是apache的开源项目，功能交互方面感觉没有Tableau体验感好

![github](https://github.com/AiRanXin/e-commerce_warehouseV5.0/blob/main/demo_picture/superset.png "github") 
