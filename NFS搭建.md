# NFS搭建

## Step 1

```shell
1.
vim /etc/exports

添加字段
[你要共享的目录]*(参数)
e.g:
/home/nuaa/nfsroot *(insecure,rw,sync,no_root_squash)

2.
systemctl reload nfs-kernel-server
```

**insecure参数应该需要加**

| 参数           | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| ro             | 只读                                                         |
| rw             | 读写                                                         |
| root_squash    | 当NFS客户端以root管理员访问时，映射为NFS服务器的匿名用户     |
| no_root_squash | 当NFS客户端以root管理员访问时，映射为NFS服务器的root管理员   |
| all_squash     | 无论NFS客户端使用什么账户访问，均映射为NFS服务器的匿名用户   |
| sync           | 同时将数据写入到内存与硬盘中，保证不丢失数据                 |
| async          | 优先将数据保存到内存，然后再写入硬盘；这样效率更高，但可能会丢失数据 |

## Step2(optional)

完成第一步即可，容器若想使用需指定共享目录和挂载点。若需测试nfs共享目录的可用性，可以在客户端进行挂载测试。

- 直接挂载

`mount -t nfs [ip]:[共享目录] [挂载点] -o nolock`

- 自动挂载（开机自动）

```shell
vim /etc/fstab
添加字段
e.g:
192.168.245.128:/public  /mnt/public       nfs    defaults 0 0
```

