#准备环境

## 准备/etc/hosts

## 规划网络，分为public_network 和 cluster_network

## 增加ceph的yum repo

编辑文件/etc/yum.repos.d/ceph.repo

	[Ceph]
	baseurl=http://10.150.140.95/update/
	name=Ceph packages
	enabled=1
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

##所有节点安装ceph
	
	yum install ceph


# 安装monitor节点

	yum install ceph-deploy


## 开始配置ceph.conf
	


	su - cephadmin
	mkdir mycluster

此后所有操作都用cephadmin用户在mycluster下进行
	
	ceph-deploy new 初始moniter节点
	
## 修改ceph.conf文件

其中ms_nocrc是给cpu受限的机器使用

	[global]
	auth_service_required = cephx
	filestore_xattr_use_omap = true
	auth_client_required = cephx
	auth_cluster_required = cephx
	mon_host = 10.182.200.24,10.182.200.78,10.182.200.77
	mon_initial_members = <初始moniter节点>
	fsid = ee4ea70c-093f-4797-8d3d-871c0aacc92b
	osd_pool_default_size = 3
	osd_pool_default_min_size = 2
	osd_pool_default_pg_num = 128
	osd_pool_default_pgp_num = 128
	ms_nocrc=true


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
	mount /dev/disk /var/lib/ceph/osd/ceph-{osd-number}

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

启动ceph osd

	/etc/init.d/ceph start osd

检查是否加入集群

	ceph osd tree
	ceph -s



# 操作ceph集群


## 删除默认的pool

	ceph osd pool delete metadata  --yes-i-really-really-mean-it
	ceph osd pool delete rbd  --yes-i-really-really-mean-it
	ceph osd pool delete data  --yes-i-really-really-mean-it
	
## 建立新pool
	
pgsnum = (osd数量 * 100) / 副本数 向上对齐

	ceph osd pool create ${poolname} {pgnum} {pgnum}
	ceph osd pool set video size {副本数}

## 删除osd

	ceph osd out {osd-num}
	/etc/init.d/ceph stop osd.{osd-num}
	ceph osd crush remove osd.{osd-num}
	ceph osd crush remove `hostname -s`
	ceph auth del osd.{osd-num}
	ceph osd rm osd.0

## 增加一个bucket

例如增加啊一个bucket, 类型为rack, 名字为L1

	ceph osd crush add-bucket L1 rack
	ceph osd crush move L1 root=default






