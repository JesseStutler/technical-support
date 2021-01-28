# 构建AIDemo过程中遇到的问题

## 版本依赖问题

这是使用python遇到的最常见的问题了，最好的办法就是找到官方推荐的匹配版本，如果是git的别人的代码，就针对问题百度修改至当前自己所使用的特定版本。

例如tensorflow跟keras的一个版本匹配网址：https://docs.floydhub.com/guides/environments/

## 模型训练

在模型训练过程中，所有的**文件夹一定要已经存在**，python没有权限给你创建新的文件夹（本人因为文件路径问题，重复训练了四次，分别是训练集路径、人脸分类器路径、模型路径，训练时间不长还好，如果时间长，简直崩溃）

## 把应用构建成后端形式

### 后端架构选择：flask

官网中文文档地址：https://dormousehole.readthedocs.io/en/latest/

在这里我采用的是一个微服务器框架flask，也就是一个轻量级的后端服务器架构，并且它原生的适合于python的后端，构建基于image形式的后端简直再适合不过。

### 数据传递的问题

一个整体的应用跑在一台机器上，直接就是从摄像头获取数据，然后继续预测，最后返回结果，最终呈现，这一连贯的过程让我觉得构建成后端应用形式十分简单，殊不知没有很大webAPI接口开发经验的我尝尽了苦头。

首先应用要做的是进行人脸识别，用户端获得图片数据之后应该以什么格式的数据传到Backend上？

经过摸索后采用的方法如下：

用户端先将图片（目前呈现实现的是jpg格式的）以二进制形式读取，之后采用base64进行编码，然后以post的方式传至server并获取识别后的图片流（也是base64格式的）。

```python
# 二进制打开图片
file = open(file_path, 'rb')
encode = base64.b64encode(file.read())

# 发送post请求到服务器端
r = requests.post(url, data=encode)
# 获取服务器返回的图片，字节流返回
result = r.content
# 字节转换成图片
img = base64.b64decode(result)
file = open('test3.jpg', 'wb')
file.write(img)
file.close()
```

在Server上面进行的操作是，先解码，再转换成opencv指定的图片格式(np三维数组)，然后调用model进行识别(predict)得到最终的结果，将图片转换为字节流并进行base64编码回传。

```python
# 格式转换
base64_data = request.get_data()
# print(base64_data)
imgData = base64.b64decode(base64_data)
nparr = np.fromstring(imgData, np.uint8)
frame = cv2.imdecode(nparr, cv2.IMREAD_COLOR)

# cv_image to byte stream
img = cv2.imencode('.jpg', frame)[1]
frame = str(base64.b64encode(img))[2:-1]
```

### 打包成镜像

#### 确定当前python版本

Python 3.7.0，文件夹 AIDemo, 主入口文件 /AIDemo/AIinEdge/Code/Face_recognition.py

#### 导出应用依赖包

pip freeze > requirements.txt

生成的 requirements.txt 复制到/AIDemo/AIinEdge/Code里，确保requirements.txt在/AIDemo/AIinEdge/Code里即可。

#### 编写Dockfile

```dockerfile
# 基于的基础镜像

FROM python:3.7.0

# 维护者信息

MAINTAINER Dejan wxlongnuaa@163.com

# 代码添加到code文件夹

ADD ./AIinEdge /AIinEdge/

# 设置code文件夹是工作目录

WORKDIR /AIinEdge/Code

# 安装支持

RUN pip install -r requirements.txt

CMD ["python", "./Face_recognition.py"]
```

#### 制作镜像

```shell
docker build -t dejan/predictor:v1 .
```

镜像内的服务器默认端口号为5000,所以调用人脸识别的API即为：

```python
url = "http://127.0.0.1:5000/predictor"
encode = base64.b64encode(file.read())
# 发送post请求到服务器端
r = requests.post(url, data=encode)
```

### 外部无法调用问题解决

打包好的AIdemo镜像dejan/predictor:v1以容器run之后无法访问，出现的是访问拒绝。查了官网文档后发现是策略的问题：

------

外部可见的服务器

运行服务器后，会发现只有你自己的电脑可以使用服务，而网络中的其他电脑却不行。缺省设置就是这样的，因为在调试模式下该应用的用户可以执行你电脑中的任意 Python 代码。

如果你关闭了调试器或信任你网络中的用户，那么可以让服务器被公开访问。 只要在命令行上简单的加上 `--host=0.0.0.0` 即可:

```
$ flask run --host=0.0.0.0
```

这行代码告诉你的操作系统监听所有公开的 IP 。

------

引自：

[Flask快速上手]: https://dormousehole.readthedocs.io/en/latest/quickstart.html#quickstart

本次我这里采用的是如下方法：

```python
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80, debug=True)
```

#### 更新后重建镜像

```
docker build -t dejan/face_recog:1.0 ./
```

#### 测试

启动容器把端口暴露在宿主机5000：

```
docker run --name test -p 127.0.0.1:5000:80 -itd dejan/face_recog:1.0
```

![docker_run](https://github.com/JesseStutler/technical-support/blob/master/assets/AIDemo/docker_run.jpg?raw=true)

在外部编写的测试程序Client.py:

```python
# Client.py
import requests
import base64
import time

url = "http://127.0.0.1:5000/predictor"

file_path = './111.jpg'



file = open(file_path, 'rb')

encode = base64.b64encode(file.read())


#r = requests.post(url, data=encode)
for i in range(10):# 针对流式数据传输的多次尝试，以防调用失败
    try:
        r = requests.post(url, data=encode)
    except Exception as e:
        if i >= 9:
            do_some_log()
        else:
            time.sleep(0.5)
    else:
        time.sleep(0.1)
        break
result = r.content

img = base64.b64decode(result)
file = open('test3.jpg', 'wb')
file.write(img)
file.close()
```

调用结果：

![result](https://github.com/JesseStutler/technical-support/blob/master/assets/AIDemo/api_result.jpg?raw=true)

服务器log:

![log](https://github.com/JesseStutler/technical-support/blob/master/assets/AIDemo/server_log.jpg?raw=true)

可以看到调用码200，即成功调用API，外面检查返回的test.jpg也是带有框图的照片