## KF部署与应用：

部署工具Kfctl

管理工具kustomize，base文件，分别构建开放、测试、生产三个环境。

KF与k8s版本兼容性问题：

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image002.jpg?raw=true)

当前k8s 1.18，应该安装最高版本

 

Kubeflow:

1. tar -xvf kfctl_v1.2.0-0-gbc038f9_linux.tar.gz#kfctl 命令安装

2. 环境配置

```bash
# The following command is optional making the kfctl binary easier to use.

$ export PATH=$PATH:’/home/nuaa/’

# Set KF_NAME to the name of your Kubeflow deployment. This also becomes the name of the directory containing your configuration. For example, your deployment name can be 'my-kubeflow' or 'kf-test'.

 

$ export KF_NAME=’kf-test’

 

# Set the path to the base directory where you want to store one or more Kubeflow deployments. For example, /opt/. Then set the Kubeflow application directory for this deployment.

 

$ export BASE_DIR=’/opt/’

$ export KF_DIR=${BASE_DIR}/${KF_NAME}

# Set the configuration file to use, such as the file specified below:

$ export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/v1.2-branch/kfdef/kfctl_k8s_istio.v1.2.0.yaml"

 

# Generate and deploy Kubeflow:

$ mkdir -p ${KF_DIR}

$ cd ${KF_DIR}

$ kfctl apply -V -f ${CONFIG_URI}

 

# CUDA：ubuntu:先安装好对应的显卡驱动，再通过apt-get install nvidia-cuda-toolkit安装nvcc管理工具。


```





 



## 通过MiniKF案例入门：

A data scientist starts from a Notebook, builds the pipeline, and uses Rok to take a snapshot of the local data they prepare, with a click of a button. Then, they can seed the Kubeflow Pipeline with this snapshot using only the UIs of KFP and Rok.

1.Create Notebook Servers

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/3.jpg?raw=true)

2. Bring in the Pipelines code and data

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image004.jpg?raw=true)

3. Snapshot the Data Volume

用来存放原始数据，方便回滚，以及多次执行等操作。

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image006.jpg?raw=true)

4. Upload the Pipeline to KFP

把已有的管道文件下载下来然后新建一个Pipeline

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image008.jpg?raw=true)

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image010.jpg?raw=true)

5.Create a new Experiment Run

在上传pipeline之后得到一个执行流图，创建新的experiment（也就是一个AI实例，KF里面统称为experiment）。

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image012.jpg?raw=true)

需要建立pipeline和数据块存储的联系

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image014.jpg?raw=true)

Pipeline 执行的快照

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image016.jpg?raw=true)

6.Run a Pipeline

运行pipeline执行，执行过程中可以通过点击流图查看实时运行的log，有问题解决之后，会自动重试。（在此过程中遇到的docker pull策略问题之后再解决）

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image018.jpg?raw=true)

7. Pipeline snapshot with Rok

Rok将自动为管道拍摄快照，以便我们以后可以将其附加到Notebook上并进一步研究管道结果。

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image020.jpg?raw=true)

所有pipeline组件完成的信息都在这里面

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image022.jpg?raw=true)

8. Explore pipeline results inside a Notebook

为了研究pipeline结果，我们可以创建一个新的Notebook并将其附加到管道快照中。Notebook将使用此不可变快照的副本。这意味着可以使用此数据量进行进一步的探索和试验，而不会丢失管道运行产生的结果。

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image024.jpg?raw=true)

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image026.jpg?raw=true)

添加卷，选择existing：

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image028.jpg?raw=true)

并在Rok中选择上述的pipeline-snapshot

 

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image030.jpg?raw=true)

然后打开Jupyter服务器，管道“ taxi-cab-classification”的输入数据和管道“ taxi-cab-classification-s9qwx”的输出数据，其中可以看到每一步的输出结果

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image032.jpg?raw=true)

![avatar](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/clip_image034.jpg?raw=true)

具体案例网址：

https://www.arrikto.com/tutorials/data-science/an-end-to-end-ml-pipeline-on-prem-notebooks-kubeflow-pipelines-on-the-new-minikf/