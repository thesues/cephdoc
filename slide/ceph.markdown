# ceph

A unified distributed storage system

- High Performance 
- High Availiable
- High scalability


# Who is using it

- CERN
- Dreamhost
- jd.com
- kingsoft

# Background

1. Starting at 2006 in Storage Systems Research Center University of California
2. Rbd client merged into linux upstream
3. Best friend of openstack
4. Active community

# Architecture

![](img/stack.png)

# Architecture

![](img/ceph-topo.jpg)


# Architecture: Monitor

Cluster Map

1. Monitor Map, the cluster fsid, the position, name address and port of each monitor
2. OSD Map, a list of pools, replica sizes, PG numbers, a list of OSDs and their status
3. PG Map,  the last OSD map epoch, the full ratios, and details on each placement group
4. CRUSH Map, a list of storage devices, the failure domain hierarchy
4. MDS Map, a list of metadata servers


# Architecture

- CRUSH Algorithm:
    
    CRUSH(CRUSH Map, CRUSH Rules, key, n) =  (osd0,osd1 ... osdn)

- Pros:

    1. use *very little* data(CRUSH Map) to compute the postion of replications
    2. CRUSH Map is only changed when adding or removing osd(automatically)

# Client and OSD interact directly

![](img/client-osd.png)

# Client and OSD interact directly
![](img/replication.png)

# Mapping PGs to OSDs

![placement group](img/mapping.png)

# Mapping PGs to OSDs

A layer of indirection between the Ceph OSD Daemon and the Ceph Client

- peering
- recovering
- rebalancing


# Rebalancing

- When: Adding or Removing an OSD
- Change crush map
- Migrate PG

# Rebalancing

![](img/re.png) 

# Smart OSD

- OSDs Check Heartbeats
- OSDs Report Down OSD
- OSDs Report Peering Failur

# Features

1. client和osd直接通信，不需要转发
2. 无单点
3. 强一致性,数据复制
4. 负载均衡,多OSD并发
5. 故障自动检测,自动恢复
6. 容易扩展容量

# CRUSH Map

    root platter {
            id -5
            alg list
            item ceph-osd-platter-server-1 weight 2.00
            item ceph-osd-platter-server-2 weight 2.00
    }
    host ceph-osd-platter-server-1 {
            id -3
            alg straw
            item osd.4 weight 1.00
            item osd.5 weight 1.00
    }
    host ceph-osd-platter-server-2 {
            id -4
            alg straw
            item osd.6 weight 1.00
            item osd.7 weight 1.00
    }

# CRUSH Map

![](img/crush.png)

# Simplest CRUSH Algorithm

    #x:object key
    #n:replication number
    #t:destnation type
    #i:start

    select(x="key", n=3, t=osd, i=platter)
    O = []
    for r <- 1,3 do
        do 
          b <- bucket(i)
          o <- b.c(x,r)
          if type(o) != t then
              b = bucket(o)
              retry_bucket <- true
         while(retry_bucket)
         add o to O
    end for
    return O 


# CRUSH Bucket Algorithm

    #parameter : r,x
    #return    : number

    c(r,x)

# CRUSH Bucket Algorithm

![](img/summary_bucket.png)


# CRUSH Bucket Algorithm

Uniform

    #P is based on hash(x)
    def c(r,x):
        return (hash(x) + r * P) % m

List
    
    def c(r,x):
        for i in back tranverse the list
            if hash(r, x, i) < i.weight / previous_weight_sum then
                return i
            endif
        endfor

Straw

    def c(r,x):
        return max([i.weight * hash(r, x, i) for i in each_item])

# CRUSH Bucket Algorithm

![Tree](img/binarytree5.png)
    
    def c(r,x):
        return get_sub(root, r, x)

    def get_sub(node, r, x):
        if node is leaf:
            return node
        if hash(x, r, node.index) < node.left.totalWeight:
            return get_sub(node.left, r, x)
        else:
            return get_sub(node.right, r, x)


# Librados Prerequires

ceph.conf

    [global]
    mon host = 192.168.1.1
    keyring = /etc/ceph/ceph.client.admin.keyring


# Librados

Configuring a Cluster Handle
    
    #include <rados/librados.h>
    
    rados_t cluster;    
    rados_create(&cluster, NULL);
    rados_conf_read_file(cluster, "/etc/myceph.conf");
    
Writing an Object 

    /* create an "IO context" */
    rados_ioctx_t io;
    char *poolname = "mypool";
    rados_ioctx_create(cluster, poolname, &io);
    rados_write_full(io, "greeting", "hello", 5);

Cleanup

    rados_ioctx_destroy(io);
    rados_shutdown(cluster);


# END

Thank You
