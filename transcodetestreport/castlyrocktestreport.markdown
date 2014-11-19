# 功能

## 上传

转码机调用striprados上传文件,目前不支持断点续传

## 下载

CDN访问wuzei的http接口下载文件, 支持断点续传.
计划wuzei的部署申请运维LVS, 部署多台做HA

# 性能测试

在正常情况下,无rebalance的情况下, 下载40M/s 在正常情况下,无rebalance的情况下, 上传20M/s

## 满负荷cpu,mem占用率 

![cpu](./normalcpu.png)

![mem](./normalmem.png)

CPU使用在20-50核之间波动，平均值大约在：40

Free + Cache的内存,平均值大约在：20-21G

以上上机器的状态,而ceph-osd/ceph-mom的cpu,memory占用基本维持在5%之下

# 稳定测试

考虑到转码机器经常性的迁移, ceph集群使用3副本的存储策略, 理论上
服务稳定度8个9, 可以抵御单个机柜offline的情况下,仍然可以提供服务.

## rebalance测试

rebalance相关参数设置

	[osd]
	osd max backfills = 2
	osd backfill scan min = 32
	osd backfill scan max = 512
	osd recovery max active = 2
	osd recovery max chunk = 16M
	osd recovery threads = 1
	journal max write bytes = 32M
	journal queue max bytes = 32M

手动测试了只增删一台服务器的情况下, rebalance的时间取决与数据量和网络带宽, 但是目前的测试环境rebalance的网络
与用户网络使用相同的网卡,为了保证客户端的访问,限制了rebalance的速度,但是分布式存储系统,rebalance的时间越短,
系统可靠性越高. 还需要持续优化

### 增加一台OSD服务器

在有转码机运行,转码程序正常上传下载文件. reablance时 ceph cpu占用情况
褐色是增加的OSD

![cpu](./cpu.png)

![mem](./mem.png)

cluster总容量2.2T,
4个osd切换到5个osd, 原集群由weight 10 变为 weight 10.4,容量变化为5%,影响8%的placement group, 新osd
网卡跑满,平均每秒拷贝3个object(1个object32M), 总用时33分钟,新osd增加数据64G. 

对服务可用性无影响, 但由于rebalance的存在
上传降为7MB/s, 下载31MB/s,速度上下浮动超过5MB/s

总之,在rebalance的情况下,对client的访问速度有很大影响,不过预计在线上分2个网段的情况下,可以大大减少对
client访问的影响

### 减少一台OSD服务器

![cpu](./removecpu.png)

![mem](./removemem.png)

褐色是减少的OSD

测试通过,服务无影响, 总时间与增加一台osd类似,但是OSD的减少分为2部分

1. primary placement group在丢失的osd上的数据迁移
2. 其他数据迁移

如果OSD标志为out, 开始步骤1. 如果osd又及时恢复,osd做增量恢复
如果步骤1完成, 管理员手动操作拿掉丢失osd, 进入步骤2

由于减少osd分2个步骤,迁移总量与之前一致,运行时间较长, 性能损失也较少


### 减少一台MON服务器

对服务无影响,对访问速度无影响,但是在高load的情况下,client连接mon节点的速度明显变慢,建议mon使用单独的服务器
或者少转码任务


