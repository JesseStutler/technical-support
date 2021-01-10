# DC - Harbor 使用

​	内网环境下的集群，通常会配置一个镜像仓库，一是出于节省带宽的考虑，可以做到一次拉取，多次分发；二是为项目提供一个安全的私有的镜像管理仓库。

<br/>

## 0. 日常使用说明

+ 配置 `/etc/docker/daemon.json`

	```json
	{
	    "insecure-registries": [
		"10.2.0.46"
	    ],
	    "log-opts": {
	        "max-file": "5",
	        "max-size": "50m"
	    }
	}
	```

+ 登陆 harbor : 

	```bash
	$ docker login docker login 10.2.0.46
	```

	按分配的账号密码登录。

+ 拉取镜像：

	harbor 已配置好 `Docker Hub` 的 proxy cache ，需要拉取 `Docker Hub` 中的镜像，需要按照如下格式

	```bash
	$ docker pull 10.2.0.46/dockerhub/<IMAGE_NAME>:<VERSION>
	```

	`<>` 中的内容请自行替换。

	*注：某些使用场景可能需要在本地重新 `docker tag` 一下*

+ 上传镜像：

	```bash
	$ docker tag <IMAGE_NAME>:<VERSION> 10.2.0.46/<PROJECT_NAME>/<IMAGE_NAME>:<VERSION>
	$ docker push 10.2.0.46/<PROJECT_NAME>/<IMAGE_NAME>:<VERSION>
	```

	由于 harbor 中是按照项目进行管理，所以镜像上传时一定要指定对应的项目名。

+ harbor 详细配置过程见下文。

<br/>

## 1. 简单搭建

+ 从 [官方 Release](https://github.com/goharbor/harbor/releases) 中下载最新的离线包
+ `tar -xvf ` 解压到本地
+ 修改 `harbor.yml` 中的 `hostname` 与 `harbor_admin_password`
+ 执行 `./install.sh`  （修改配置后，直接执行 `./prepare` 即可)

> 在 `harbor` 路径下执行 `docker-compose down` 停止所有容器
>
> 执行 ` docker-compose up -d ` 启动所有容器

<br/>

## 2. 设置缓存仓库

进入 harbor 管理界面，`Registries` -> `New Registry Endpoint` 选择对应的 `Provider`

![ZnOJF5oQ9AYiLW4](https://i.loli.net/2021/01/10/ZnOJF5oQ9AYiLW4.png)



<br/>

# 3. 新建项目

使用上述镜像，还需要新建一个项目，项目是 harbor 中的一个管理单位。

`Project` 中点新建，如果是一个镜像缓存项目，需要开启 `Proxy Cache`

![IzmSgsTDCZhKPNH](https://i.loli.net/2021/01/10/IzmSgsTDCZhKPNH.png)

更多使用还需探索。