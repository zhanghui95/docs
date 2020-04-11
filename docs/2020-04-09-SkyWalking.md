# SkyWalking

## 1. 概述

### 1.1 什么是APM系统

APM(Application Performance Management) 应用性能管理系统，是对企业系统即时监控以实现对应用程序性能管理和故障管理的系统化的解决方案。应用性能管理主要针对企业的关键业务应用进行检测、优化，提高企业应用的可靠性和质量，保证用户得到良好的服务，降低IT总拥有成本。

**APM系统是可以帮助理解系统行为、用于分析性能问题的工具，以便发生故障的时候，能够快速定位和解决问题**

### 1.2 分布式链路追踪

随着分布式系统和微服务架构的出现，一次用户的请求会经过多个系统，不同服务之间的调用关系十分复杂，任何一个系统出错都可能影响整个请求的处理结果。以往的监控系统往往只能知道单个系统的健康状况、一次请求的成功失败，无法快速定位失败的根本原因。

![分布式系统存在的问题](../images/2020-04-09-分布式系统存在的问题.png)

除此之外，复杂的分布式系统也面临这下面这些问题：
性能分析：一个服务依赖很多服务，被依赖的服务也依赖了其他服务。如果某个接口耗时突然变长 了，那未必是直接调用的下游服务慢了，也可能是下游的下游慢了造成的，如何快速定位耗时变长 的根本原因呢? 
链路梳理：需求迭代很快，系统之间调用关系变化频繁，靠人工很难梳理清楚系统链路拓扑(系统之间的调用关系)

**为了解决这些问题，Google 推出了一个分布式链路跟踪系统 ，之后各个互联网公司都参照 Dapper 的思想推出了自己的分布式链路跟踪系统，而这些系统就是分布式系统下的APM系统。**

### 1.3 什么是OpenTracing
**分布式链路跟踪最先由Google在Dapper论文中提出，而OpenTracing通过提供平台无关、厂商无关的API，使得开发人员能够方便的添加(或更换)追踪系统的实现。**

下图是一个分布式调用的例子，客户端发起请求，请求首先到达负载均衡器，接着经过认证服务，订单服务，然后请求资源，最后返回结果。

![调用事例](../images/2020-04-09-调用事例.png)

虽然这种图对于看清各组件的组合关系是很有用的，但是存在下面两个问题：

* 它不能很好显示组件的调用时间，是串行调用还是并行调用，如果展现更复杂的调用关系，会更加 复杂，甚至无法画出这样的图
* 这种图也无法显示调用间的时间间隔以及是否通过定时调用来启动调用

基于OpenTracing我们就可以很轻松的构建出一种更有效的展现一个调用过程的图

![OpenTracing图](../images/2020-04-09-OpenTracing图.png)

### 1.4 主流的开源APM产品

* PinPoint

Pinpoint是由一个韩国团队实现并开源，针对Java编写的大规模分布式系统设计，通过JavaAgent的机 制做字节代码植入，实现加入traceid和获取性能数据的目的，对应用代码零侵入

![pinpoint](../images/2020-04-09-pinpoint.png)

* SkyWalking

SkyWalking是apache基金会下面的一个开源APM项目，为微服务架构和云原生架构系统设计。它通过探针自动收集所需的指标，并进行分布式追踪。通过这些调用链路以及指标，Skywalking APM会感知应用间关系和服务间关系，并进行相应的指标统计。Skywalking支持链路追踪和监控应用组件基本涵盖 主流框架和容器，如国产RPC Dubbo和motan等，国际化的SpringBoot，SpringCloud

* Zipkin

Zipkin是由Twitter开源，是分布式链路调用监控系统，聚合各业务系统调用延迟数据，达到链路调用监控跟踪。Zipkin基于Google的Dapper论文实现，主要完成数据的收集、存储、搜索与界面展示

* CAT

CAT是由大众点评开源的项目，基于Java开发的实时应用监控平台，包括实时应用监控，业务监控，可以提供十几张报表展示

## 2. 什么是SkyWalking

### 2.1 SkyWalking概述

根据官方的解释，Skywalking是一个可观测性分析平台(Observability Analysis Platform简称OAP) 和应用性能管理系统(Application Performance Management简称APM)。

提供分布式链路追踪、服务网格(Service Mesh)遥测分析、度量(Metric)聚合和可视化一体化解决方案。 下面是Skywalking的几大特点：

* 多语言自动探针，Java、.NET Core和Node.JS
* 多种监控手段，语言探针和service mesh
*  轻量高效，不需要额外搭建大数据平台
*  模块化架构，UI、存储、集群管理多种机制可选
* 支持告警
* 优秀的可视化效果

整体架构如下：

![skywalking架构](../images/2020-04-09-skywalking架构.png)

Skywalking提供Tracing和Metrics数据的获取和聚合

> Metric的特点是，它是可累加的：他们具有原子性，每个都是一个逻辑计量单元，或者一个时间 段内的柱状图。 例如:队列的当前深度可以被定义为一个计量单元，在写入或读取时被更新统 计; 输入HTTP请求的数量可以被定义为一个计数器，用于简单累加; 请求的执行时间可以被定 义为一个柱状图，在指定时间片上更新和统计汇总。
>
> Tracing的最大特点就是，它在单次请求的范围内，处理信息。 任何的数据、元数据信息都被绑定 到系统中的单个事务上。 例如:一次调用远程服务的RPC执行过程;一次实际的SQL查询语句; 一次HTTP请求的业务性ID。
>
> 总结，Metric主要用来进行数据的统计，比如HTTP请求数的计算。Tracing主要包含了某一次请 求的链路数据。
>
> 详细的内容可以查看Skywalking开发者吴晟翻译的文章，Metrics, tracing 和 logging 的关系 : http://blog.oneapm.com/apm-tech/811.html

整体架构包含如下三个部分：

1. 探针(agent)负责进行数据的收集，包含了Tracing和Metrics的数据，agent会被安装到服务所在的服务器上，以方便数据的获取
2. 可观测性分析平台OAP(Observability Analysis Platform)，接收探针发送的数据，并在内存中使 用分析引擎(Analysis Core)进行数据的整合运算，然后将数据存储到对应的存储介质上，比如 Elasticsearch、MySQL数据库、H2数据库等。同时OAP还使用查询引擎(Query Core)提供HTTP查 询接口

3. Skywalking提供单独的UI进行数据的查看，此时UI会调用OAP提供的接口，获取对应的数据然后 进行展示

### 2.2 SkyWalking优势

Skywalking相比较其他的分布式链路监控工具，具有以下特点：

* 社区相当活跃。Skywalking已经进入apache孵化，目前的start数已经超过12.9K，最新版本7.0.0已经发布。开发者是国人，可以直接和项目发起人交流进行问题的解决
* Skywalking支持Java、.NET Core和Node.JS语言。相对于其他平台比如Pinpoint支持Java和 PHP，具有较大的优势
* 探针无倾入性。对比CAT具有倾入性的探针优势较大。不修改原有项目一行代码就可以进行集成
* 探针性能优秀。有网友对Pinpoint和Skywalking进行过测试，由于Pinpoint收集的数据过多，所以对性能损耗较大，而Skywalking探针性能十分出色
* 支持组件较多。特别是对Rpc框架的支持，这是其他框架所不具备的。Skywalking对Dubbo、 gRpc等有原生的支持，甚至连小众的motan和sofarpc都支持

### 2.3 SkyWalking主要概念介绍

* 服务(Service)
* 端点(Endpoint)
* 实例(Instance)

### 2.4 环境搭建

Skywalking默认使用H2内存中进行数据的存储，我们可以替换存储源为ElasticSearch或Mysql保证其查询的高效及可用性

> 具体的安装步骤可以在Skywalking的官方github上找到
>
>  https://github.com/apache/skywalking/blob/master/docs/en/setup/README.md

1. 创建目录
2. 将资源目录中的elasticsearch和













## 3. 