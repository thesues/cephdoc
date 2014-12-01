
#准备环境

## 准备/etc/hosts

## 规划网络，分为public_network 和 cluster_network

## 增加ceph的yum repo

编辑文件/etc/yum.repos.d/ceph.repo

	[ceph-updates]
	baseurl=http://10.150.140.95/update/
	name=Ceph packages
	enabled=1
	priority=1
	gpgcheck=0

## 配置ntp


## 增加新用户cephadmin

	ssh root@ceph-server
	useradd -d /home/cephadmin -m cephadmin
	passwd cephadmin

	echo "cephadmin ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephadmin
	chmod 0440 /etc/sudoers.d/cephadmin

## 配置用户cephadmin ssh无密码登陆

## 打开防火墙

* 端口 6789 用于monitor
* 端口 6800:7100 用于osd

## 禁用SELINUX

	sudo setenforce 0

##所有节点安装ceph和客户端工具
	
	yum install ceph striprados


# 安装monitor节点

	yum install ceph-deploy


## 开始配置ceph.conf
	


	su - cephadmin
	mkdir mycluster

此后所有操作都用cephadmin用户在mycluster下进行
	
	ceph-deploy new 初始moniter节点
	
## 修改ceph.conf文件

其中ms nocrc是给cpu受限的机器使用

如果确定osd的文件系统是xfs, filestore xattr use omap 为false. 同时filestore journal writeahead为true

如果使用自定义的crushmap, 设置osd crush update on start = false

如果使用osd-domain, 设置osd crush chooseleaf type = {osd-domain-num}


	[global]
	auth service required = cephx
	filestore xattr use omap = true
	auth client required = cephx
	auth cluster required = cephx
	mon host = 10.182.200.24,10.182.200.78,10.182.200.77
	mon initial members = <初始moniter节点>
	fsid = ee4ea70c-093f-4797-8d3d-871c0aacc92b
	osd pool default size = 3
	osd pool default min size = 2
	osd pool default pg num = 128
	osd pool default pgp num = 128
	ms nocrc=true
	public network = x.x.x.x/x
	cluster network = x.x.x.x/x

	[osd]
	osd max backfills = 1
	osd backfill scan min = 16
	osd backfill scan max = 256
	filestore op threads = 4
	osd recovery max active = 1
	osd recovery max chunk = 32M
	osd recovery threads = 1
	journal max write bytes = 32M
	journal queue max bytes = 32M
	mon osd down out interval  = 900



## 安装monitor节点
	
	ceph-deploy mon create-initial
	
## 推送admin keyring到所有节点
	
	sudo chmod +r /etc/ceph/ceph.client.admin.keyring
	ceph-deploy admin <所有节点>

保留mycluster文件架，为以后自动添加monitor节点准备

## 检查
	
	ceph -s

状态是HEALTH_ERR,无osd，mon节点都已经加入

# 手动安装OSD节点

所有osd节点用手动安装的方式, 使用root用户

创建osd号
	
	ceph osd create

返回一个数字，这个数字就是osd的唯一号,记录为{osd-number}

	ceph osd tree

检查出现osd.0,状态是down.

准备ceph osd文件夹
	
	mkdir /var/lib/ceph/osd/ceph-{osd-number}
	mkfs -t xfs /dev/disk
	mount -o noatime /dev/disk /var/lib/ceph/osd/ceph-{osd-number}

在文件夹内建立osd keyring等数据
	
	ceph-osd -i {osd-num} --mkfs --mkkey


检查文件夹/var/lib/ceph/osd/ceph-{osd-number}内容, 并建立sysvinit

	touch  /var/lib/ceph/osd/ceph-{osd-number}/sysvinit

添加osd权限

	ceph auth add osd.{osd-num} osd 'allow *' mon 'allow rwx' -i /var/lib/ceph/osd/ceph-{osd-num}/keyring	
	
检查权限是否已经添加

	ceph auth list

加入osd，权值1代表1T，所有如果是500G硬盘，就为0.5
	
	ceph osd crush add osd.{osd-num} {weight} root=default host={机器名}

检查osd列表
    
    ceph osd tree

启动ceph osd

	/etc/init.d/ceph start osd

检查是否加入集群

	ceph osd tree
	ceph -s


# 操作ceph集群

## 删除默认的pool

	ceph osd pool delete metadata metadata --yes-i-really-really-mean-it
	ceph osd pool delete rbd rbd --yes-i-really-really-mean-it
	ceph osd pool delete data data --yes-i-really-really-mean-it
	
## 建立新pool
	
pgsnum = (osd数量 * 100) / 副本数 向上对齐

	ceph osd pool create ${poolname} {pgnum} {pgnum}
	ceph osd pool set video size {副本数}

## 查看参数
	

	ceph --admin-daemon /var/run/ceph/ceph-osd.$id.asok config show


## 删除osd

	ceph osd out {osd-num}
	/etc/init.d/ceph stop osd.{osd-num}
	ceph osd crush remove osd.{osd-num}
	ceph osd crush remove `hostname -s`
	ceph auth del osd.{osd-num}
	ceph osd rm osd.{osd-num}

## 查看crushmap
	
	ceph osd getcrushmap -o crush.out
	crushtool -d crush.out -o crush.txt
	ceph osd crush dump
	ceph osd crush rule dump

## 增加一个bucket

例如增加一个bucket, 类型为rack, 名字为L1

	ceph osd crush add-bucket L1 rack
	ceph osd crush move L1 root=default
	ceph osd crush move osd.{osd-num} rack=L1

## 加入一个自定义的osd

	ceph osd crush add osd.{osd-num} {weight} {bucket-type}={bucket-name}


## 清理ceph数据

保证有一个干净的ceph环境

    rm /etc/ceph/*.keyring
    rm /etc/ceph/*.conf
    rm /var/log/ceph/*
    rm /var/lib/ceph/{bootstrap-mds,bootstrap-osd,mds,mon,osd}/*
    ceph auth list #检查keyring被清除
    kill -9 $pid_osd $pid_mds $pid_mon #杀死相关进程

## ceph osd 维护模式

    ceph osd set noout
    /etc/init.d/ceph stop osd.{osd-num}


## 维护完成

    /etc/init.d/ceph start osd.{osd-num}
    ceph osd unset noout


## 手工添加mon节点

1. 取得`client.admin`的keyring，保存到文件`/etc/ceph/ceph.client.admin.keyring`

2. 取得`mon.`的keyring，保存到任意位置，记为`{mon.keyring}`
	
	ceph auth get mon. -o /tmp/auth
	
3. 取得monitor map，保存到任意位置，记为`{mon.map}`

	ceph mon getmap -o /tmp/map
		
4. Optional. 更新**所有mon节点**的配置文件，添加新节点的IP地址到`ceph.conf` `[global]`字段的`mon_host`
5. 依上文配置文件配置新节点的`ceph.conf`
6. 生成mon文件系统
	
		ceph-mon -i {mon-id} --mkfs --monmap /tmp/map --keyring /tmp/key
7. 加入集群

		ceph mon add {mon-id} {ip}

7. 告诉ceph可以启动了
		
		touch /var/lib/ceph/mon/ceph-{名称}/done
		touch /var/lib/ceph/mon/ceph-{名称}/sysvinit
8. 启动mon服务

		service ceph start mon.{名称}


## 手工添加mds server

1. 创建目录：
    
    **`sudo mkdir /var/lib/ceph/mds/mds.0`**

2. 生成mds.0的keyring，保存到/var/lib/ceph/mds/mds.0/mds.0.keyring：
    **`sudo ceph auth get-or-create mds.0 mds 'allow ' osd 'allow *' mon 'allow rwx' > /var/lib/ceph/mds/mds.0/mds.0.keyring`**

3. 编辑ceph.conf:
 
		[mds]

      	mds data = /var/lib/ceph/mds/mds.$id

      	keyring = /var/lib/ceph/mds/mds.$id/mds.$id.keyring
    
    	[mds.0]

      	host = {hostname}