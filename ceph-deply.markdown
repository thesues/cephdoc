# ceph deploy

	1. Preparation

##

		1. Basic
			1. Install OS
			2. configure Ntp
			3. configure Hostname
		2. Yum Repositories Configuration
		3. Configure Ssh No-password Login
		4. Plan Your Network, Storage, Journal

	2. Add Monitor
	3. Add Osd
	4. Remove Monitor
	5. Remove Osd
	6. Bash Script
	7. Ceph.conf
	8. Ntp

##

	Nitice:
		1. For old kernels (<2.6.33), disable the write cache if the journal is on a raw drive. Newer kernels should work fine.
		   Hdparm –W 0 /dev/had 0
		2. For Ext4. Setting Filestore xattr use omap = true
		3. One Ceph Monitor per host, and no commingling of Ceph OSD Daemons with Ceph Monitors
		4. The Ceph Monitor name is the host name

#
	
	1) Preparation

##

	1. Basic

### 

		1. Install OS(osd, monitor)
		2. Congigure hostname
		3. Configure ntp client
##
		
	2. Install letv repo
		rpm -ivh http://openstack.oss.letv.cn/repo/letv-rdo-release-havana-8.1.noarch.rpm
	 	//rpm -ivh http://openstack.oss.letv.cn/repo/letv-rdo-release-icehouse-4.1.noarch.rpm 
	3. Choose a machine as deploy node. Make sure you can login on other nodes without input password
		\# ssh-keygen
		\# ssh-copy-id root@node_hostname
		\# yum install ceph-deploy -y
	4. Plan Your Network, Storage, Journal
		1. it better separate public and cluster network. 
			For example. 
			public_network = 10.154.249.0/24
			cluster_network = 11.154.249.0/24
		2. it better use a dedicated disk as journal.
		3. plan your storage space

#

	2. Add initial Monitor
		\# ceph install  node-name1 node-name2 {...} --no-adjust-repos  ; install ceph package
		\# ceph-deploy  new cephosd1-mona {...}
		\# confugre your network. Edit ceph.conf, add public and cluster network.
		\# ceph-deploy mon create-initial cephosd1-mona {...}

		Add more monitors.
		\# Edit ceph.conf, append monitor's hostname to mon_initial_members, monitor's ip to mon_host.
		\# ceph-deploy mon create cephosd2-monb
		All monitors are up now.

#

	3. Add osd. Here i use a raw disk as journal

##
		\# ceph-deploy disk zap cephosd1-mona:vdX ;delete all partitions of the dedicated disk
		\# ceph-deploy --overwrite-conf disk prepare cephosd1-mona:vdd:vdc
		\# ceph-deploy disk activate cephosd1-mona:vdd1:vdc1 ; if fail, try to rm node's /var/lib/ceph/bootstrap-osd/ceph.keyring.

		if you do not use raw disk directly, run the following commands
		\# ssh osd node, mkdir /var/lib/ceph/osd//osd-$id
		\# ceph-deploy osd prepare {ceph-node}:/path/to/directory
		\# ceph-deploy osd activate {ceph-node}:/path/to/directory

#
	4. Remove monitor

##
		\# ceph-deploy mon destroy {host-name [host-name]...}

#

	5. Remove osd. ceph-deploy doesnot support remove osd, so before run the follwoing commands, go to the osd node first.

##
		\# ceph osd out id
		\# /etc/init.d/ceph stop osd
		\# ceph osd crush remove osd.$id
		\# ceph osd crush remove `hostname -s`
		\# ceph auth del osd.$id
		\# ceph osd rm $id

#

	6. Bash

##
		1. script remove osd
		----------removeosd.sh----------------------------------------
		[root@cephosd1-mona ceph]# cat removeosd.sh 
				#!/bin/bash

				if [ $# -ne 1 ] ;then
				exit
				fi

				i=$1
				ceph osd out $i
				/etc/init.d/ceph stop osd
				ceph osd crush remove osd.$i
				ceph osd crush remove `hostname -s`
				ceph auth del osd.$i
				ceph osd rm $i

				#modify ceph.conf, delete the item and distribute
		----------------------------------------------------------------

#
	7. Ceph.conf

##
	
		[osd]
		osd_op_threads = 10
		filestore_op_threads = 10
		osd_recovery_threads = 5
		osd_disk_threads = 5
		osd_data = /var/lib/ceph/osd/ceph-$id
		osd_journal = /dev/vdc1
		[global]
		auth_service_required = cephx
		auth_client_required = cephx
		auth_cluster_required = cephx
		osd_pool_default_size = 3
		osd_pool_default_min_size = 2
		osd_pool_default_pg_num = 128
		osd_pool_default_pgp_num = 128
		public_network = 10.154.249.0/24
		cluster_network = 11.154.249.0/24
		mon_host = 10.154.249.3,10.154.249.4,10.154.249.5
		mon_initial_members = cephosd1-mona,cephosd2-monb,cephosd3-monc
		osd_crush_chooseleaf_type = 1
		[mon]
		mon_clock_drift_warn_backoff = 30
		mon_osd_down_out_interval = 120
		mon_clock_drift_allowed = 1

#

	8. Ntp

##

		Ntp配置，安装ntpd 和ntpdate
		1.	Ntp客户端: /etc/crontab
		  30 8  *  *  * root /usr/sbin/ntpdate 10.154.249.108
		2.	Ntp服务端;  Vi /etc/ntp.conf
		restrict 10.154.249.0 mask 255.255.0.0 nomodify
		server 127.127.1.0
		fudge 127.127.1.0 stratum 8
		3.	restart ntpd

