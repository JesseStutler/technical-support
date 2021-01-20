# Rook-ceph部署记录

## quickstart

https://rook.github.io/docs/rook/master/ceph-quickstart.html

节点：compute08.dc(master)，compute09.dc，compute10.dc

设备：裸分区存储，/dev/sda4(compute08.dc)，/dev/sda4(compute09.dc)，加起来大约3个T

## trouble-shooting

### rook-ceph-mon无法被调度到master上

**这个问题需要最先解决，否则其他容器之后无法启动！**

reason:

似乎ceph cluster的本意是搭建在除master之外的至少三节点集群上，但我们的现有环境是master+另外两节点，所以未指定tolerations的话pod无法被调度到master上，所以要修改cluster.yaml中的placement字段

![image-20210119105859770](/Users/chenzicong/Library/Application Support/typora-user-images/image-20210119105859770.png)



这里all或者指定组件mon，osd，mgr都可以



### rook-osd-prepare pods无法completed，所有的rook-osd pods未被创建

Bug report:

https://github.com/rook/rook/issues/6849

reason:

```shell
ceph-volume lvm batch: error: /dev/sda4 is a partition, please pass LVs or raw block devices failed to configure devices: failed to initialize devices: failed ceph-volume report: exit status 2
```

Ceph:v15.2.8的bug，无法将裸分区作为数据存储地，修改cluster.yaml中的image为v15.2.7就好了

![image-20210119145647595](/Users/chenzicong/Library/Application Support/typora-user-images/image-20210119145647595.png)



### clock skew detected on mon.b, mon.c

reason:

集群时间不同步

solution:

使用chrony进行集群时间同步

https://www.cnblogs.com/JetpropelledSnake/p/10175763.html

注意将其他client的server改成compute08.dc的ip地址，然后compute08上允许10.0.0.0/24网段

已将chronyc sources -v写入crontab，每一小时时钟同步一次

