# ceph配置项 #
1.ceph.conf中使用了五个元变量，包括$cluster、$type、$id、$host和$name，在介绍ceph的配置项之前，首先要介绍一些这个变量的意义：  

    $cluster：指的是ceph集群名，区分部署于同一批机器上的不同集群，$cluster的默认值是ceph。
    $type：指的是进程的类型，如mds，osd或者mon。如/var/lib/ceph/$type。
    $id：指的是进程编号，如果有一个进程osd.0,那么它的$id为0。如/var/lib/ceph/$type/$cluster-$id
    $host：$host指的是进程所运行的机器的hostname。
    $name：指的是进程名，即进程类型.进程编号（$type.$id）。如/var/run/ceph/$cluster-$name.asok。
2.global部分：   
[global]

* public network   = 192.168.0.0/24；用于client访问mon和osd的网络
* cluster network  = 192.168.1.0/24；用于ceph集群内部数据reblance等所使用的网络
* pid file         = /var/run/ceph/$name.pid；存放进程ID
* auth cluster required  = cephx；以下三个表示使用cephx认证方式
* auth service required  = cephx
* auth client required   = cephx
* keyring = /etc/ceph/$cluster.$name.keyring；表示keyring文件的存放路径
* osd pool default size  = 3；使用的副本数，默认值为3
* osd pool default min size  = 2；最小副本数，默认值为2，低于此值则不可写
* osd pool default pg num    = 128；pool的pg数，默认值为8，一般而言8这个值太小了，不利于数据分散，建议设置成相对较大的值。
* osd pool default pgp num   = 128；设置成与上面的pg num相同
* log file                   = /var/log/ceph/$cluster-$name.log；日志文件存放路径
* log to syslog              = true；是否将日志写入到syslog
* filestore_xattr_use_omap   = true; 若使用ext4文件系统，由于XATTRs仅为4KB，所以需要设置此值来使用omap来存储ceph XATTRs 
3.monitor部分：  
[mon]  

* mon initial members        = mycephhost；初始monitor，数目需为奇数个
* mon host                   = cephhost01；monitor的hostname
* mon addr                   = 192.168.0.101；monitor的IP
* mon data                   = /var/lib/ceph/mon/$name；monitor数据存放路径
* mon clock drift allowed    = .05；monitor之间允许的时间差最大值，默认值为0.05，如果不配置ntp服务，容易出现时间差过大的告警
* mon clock drift warn backoff = 30；如果时间差过大，则30s后会重新告警
* mon osd full ratio         = .95；磁盘使用量超过95%，则认为该OSD已满
* mon osd nearfull ratio     = .85；磁盘使用量超过85%，则认为该OSD将要满了
* mon osd down out interval  = 300；当一个OSD被其他OSD标记为down，超过300s后仍没回应，则认为down和out
* [mon.alpha]
  * host                     = alpha;monitor的hostname
  * mon addr                 = 192.168.0.10:6789；monitor的IP:PORT组合
  
4.osd部分：  
[osd]

* filestore xattr use omap = true; 若使用ext4文件系统，由于XATTRs仅为4KB，所以需要设置此值来使用omap来存储ceph XATTRs 
* osd data                 = /var/lib/ceph/osd/$name；osd数据存放地址
* osd journal              = /var/lib/ceph/osd/$name/journal；journal存放路径
* osd journal size         = 1024；journal大小，默认为5120MB。ceph官方提供了一个计算公式：size = {2 * (expected throughput * filestore max sync interval)}
* filestore max sync interval = 5；最多5s将journal中的数据向文件系统进行同步
* filestore op threads     = 4；处理后端文件系统事务的线程数
* osd op threads           = 2；osd线程数，一般设置为cpu数,可通过ceph --admin-daemon /var/run/ceph/ceph-osd.$id.asok config get/set osd_op_threads获取和动态修改此值。
* [osd.0]
  * host                     = alpha;osd的hostname
  
5.client部分：  
[client]

* rbd cache                 = true；使用rbd cache
* rbd cache size            = 33554432；设置cache为33554432bit

6.radosgw client部分：   
[client.radosgw.gateway]

* rgw data                  = /var/lib/ceph/radosgw/$name；数据存放路径
* host                      =  ceph-radosgw；client的hostname
* keyring                   = /etc/ceph/ceph.client.radosgw.keyring