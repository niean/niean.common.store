# nodata概要设计
<script type="text/javascript" src="./js/jquery-1.7.2.min.js"></script>
<script type="text/javascript" src="./js/menu.js"></script>
<div id="category"></div>

## 系统概述
nodata能够和judge一起，监测采集项的上报异常，过程为: 配置了nodata的采集项超时未上报数据，nodata生成一条非法的mock数据；用户在judge上配置相应的报警策略，收到mock数据就产生报警。采集项上报异常检测，作为judge的一个必要补充，能够使judge的实时报警功能更加完善、可靠。

### 用户需求
nodata只处理如下用户需求，

1. 监测"特征采集项"的上报异常
2. 监测少量的、十分重要的采集项 的上报异常

所谓**特征采集项**，指的是能够表征一组采集项数据上报情况的单个采集项，这个采集项是这组采集项中的一员。例如，falcon-agent默认上报的基础采集项，agent.alive就是一个特征采集项:只要agent.alive上报正常，我们就可以认为其他的基础采集项也上报正常。

### 系统定位
nodata只为少数重要的采集项服务，其处理的采集项的数量，应该不多于judge的十分之一。滥用nodata，将会给falcon的运维管理带来很多问题。

### 系统边界
nodata用于监测falcon采集项的上报异常。这里的异常，限定为 用户数据采集服务异常、falcon数据上报链路异常等，主要场景有：

用户数据采集服务异常，包括

+ 用户数据采集服务，异常终止
+ 用户数据采集服务，与falcon数据收集器之间的通信链路异常，使得数据无法上报
+ 用户数据采集服务，上报的数据格式错误

falcon数据上报链路异常，包括

+ agent异常，无法接收用户的数据推送、无法主动采集监控数据
+ agent与数据转发transfer之间通信异常


出现以下情况时，nodata不应该引发大规模的报警:

1. 由于网络故障，导致大部分的采集项上报异常
2. 由于falcon自身服务故障，导致大量的采集项上报异常

## 系统设计

### 系统流图
![nodata.funtion](./pict/nodata.function.png)

### 模块结构
![nodata.module](./pict/nodata.module.png)

其中，collector需要考虑将请求打散在不同时间片内、防止瞬时高并发的出现。

### 部署架构
![nodata.deploy](./pict/nodata.deploy.png)

如果集群部署nodata，其负载均衡需要由配置中心实现。

## 使用方式
使用nodata时，首先要设置采集项的"nodata上报配置"，然后根据"nodata上报配置"设置相应的报警策略。这里，我们以采集项```endpoint=bj.c3.host01, counter=cps/pdl=falcon,service=hbs,dstype=GAUGE,step=300```为例。```cps```的取值范围是[0, +inf)。

### nodata上报配置
```python
exact(endpoint=bj.c3.host01, metric=cps, tags=[pdl=falcon, service=hbs], dstype=GAUGE, step=300){
	nodata.mock=-1
}
```
这里，我们设置nodata上报值为-1。为什么选择-1呢？因为这个采集项的正常取值范围是[0,+inf)，当我们发现有取值为-1时，就说明一定发生过上报超时。

### 报警策略配置
```python
each(metric=cps, pdl=falcon, service=hbs){
	if all(#3)==-1{
		alarm()
		callback()
	}
}
```
因为我们已经配置nodata上报为-1，针对这个值设置报警策略，就能实时监测到采集项上报超时的情况。

## FAQ
#### nodata为什么不做到judge里
+ judge的逻辑很复杂, nodata的逻辑也不简单。难难合并就是难上加难, 不利于系统开发&维护
+ judge是数据驱动模型, nodata是时间驱动模型, 两者分离开来更合理
+ judge是一个很成熟的模块, nodata是未成熟的新生儿, 为了保护judge的高可用, 我们把nodata拿出来、单独开发维护 也是很合理的

#### nodata是否可以横向扩展
nodata可以横向扩展。怎么实现的？ nodata后端实例，会根据**收到的配置信息**展开业务逻辑，接收的配置多就要多处理一些nodata判断、接收的配置少就会少处理一些nodata判断。我们可以在配置下发的地方(配置中心)做一些文章，让nodata的配置信息 总是能够相对均匀的、以固定的顺序打散到多个后端实例上；后端实例发生缩扩容时，配置中心能够重新调整配置信息的分配、使达到新的均衡。
nodata的配置量不会太大，因此配置中心在负载均衡上耗费的资源也是可以接受的。
