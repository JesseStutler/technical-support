# 一、前置条件

## 集群状况：

| 节点信息 | 数量 | 名称           |
| -------- | ---- | -------------- |
| 部署节点 | 1台  | router         |
| 控制节点 | 1台  | controller01   |
| 计算节点 | 7台  | compute[01:07] |
| 存储节点 | 7台  | cinder[01:07]  |

*需要注意的是，计算节点与存储节点共用。*

## 网络状况：

![network-topology](https://github.com/JesseStutler/technical-support/blob/master/assets/OcataDeployment/network_topology.jpg?raw=true)

这是此前的网络规划，其中 Router eno1 网卡上具有唯一的外网连接，其他节点均通过路由转发实现外网访问。

且由于校园网环境的变动，此前的网络配置被认为是不稳定，容易出现问题的，目前各个网段的规划划分如下：

| 网段        | 对应网卡 | 用途                                         |
| ----------- | -------- | -------------------------------------------- |
| 10.0.0.0/24 | ens6f0   | 镜像下发，registry位于 http://10.0.0.41:4000 |
| 10.1.0.0/24 | ens6f1   | neutron_external_network                     |
| 10.2.0.0/24 | eno2     | openstack 管理网络                           |



# 安装过程

## 安装 Docker

https://docs.docker.com/engine/install/centos/ ，安装 1.17版本。

配置 docker

```bash
$ mkdir /etc/systemd/system/docker.service.d
$ tee /etc/systemd/system/docker.service.d/kolla.conf << 'EOF'
[Service]
MountFlags=shared
EOF
```

重启 docker

```bash
$ systemctl daemon-reload
$ systemctl enable docker
$ systemctl restart docker
```

Docker python 库：

```bash
$ yum install python-docker-py
```

或使用 pip:

```bash
$ pip install -U docker-py
```



## Ansible

```bash
$ pip install -U 'ansible<2.10'
```

## Registry

```bash
$ docker run -d -v /opt/registry:/var/lib/registry -p 4000:5000 \
--restart=always --name registry registry:2
```

镜像存放于 Router 节点中，对镜像打标签与上传的脚本位于 `router:/root/scripts`

**注意，此处有个坑：kolla-ansible Ocata 版本中将部署文件中所有 source 写成了 sourece ，因此在 tag 时需要注意 source -> sourece**



## 关闭 libvirt

```bash
# CentOS 7
$ systemctl stop libvirtd.service
$ systemctl disable libvirtd.service

```



## Kolla-ansible

下载 Ocata 版本：

```bash
$ git clone http://git.trystack.cn/openstack/kolla-ansible –b stable/ocata
```

安装依赖:

```bash
$ pip install -r ./kolla-ansible/requirements.txt
```

复制相关文件

```bash
$ cp -r etc/kolla /etc/kolla/
$ cp ansible/inventory/* /home/
```



## Kolla-ansible 配置

`/etc/kolla/globals.yml` 见仓库

生成密码：

```bash
$ kolla-genpwd
```



## 安装

初始化：

```bash
$ nohup kolla-ansible -i /root/ocata/multinode bootstrap-servers &
```

预检查:

```bash
$ nohup kolla-ansible -i /root/ocata/multinode prechecks &
```

部署：

```bash
$ nohup kolla-ansible -i /root/ocata/multinode deploy &
```

完成部署:

```bash
$ nohup kolla-ansible -i /root/ocata/multinode post-deploy &
```



## 初始化 Openstack

编辑 /usr/share/kolla-ansible/init-runonce，

网络需要根据实际情况修改

```
EXT_NET_CIDR='10.1.0.0/24'
EXT_NET_RANGE='start=10.1.0.30,end=10.1.0.200'
EXT_NET_GATEWAY='10.1.0.41'
```

运行：

```bash
$ source /etc/kolla/admin-openrc.sh
$ cd /usr/share/kolla-ansible
$ ./init-runonce
```



