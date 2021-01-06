# KFServing

解决机器学习"最后一公里"问题，即如何把训练好的模型发布成为一个服务。主要内容包括KFServing Python SDK、Istio、Knative。

KFServing的架构如下图所示。

![img](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFS0.png?raw=true)

它解决的问题主要是两个：

1.不同机器学习框架训练出来的模型格式不同，KFServing中，只需指定机器学习框架和模型路径就能迅速发布。

2.KFServing使得机器学习的模型能够像其他软件一样具有升级、回退、灰度发布等功能（借助于Istio和Knative)。

KFServing目前对主流的ML框架（TF、PyTorch、XGBoost等）提供高性能模型服务，封装了**自动扩展**、**网络**、**运行状况检查**和**服务器配置**等复杂操作。

## Istio

首先是Service Mesh, 一种分布式的微服务代理，如Sidecar模式，主要是Data Plane 和 Control Plane。前者负责网格内服务的通信，实现功能，后者管理和配置Sidecar容器。有种SDN的感觉。

![KFS1](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFS1.png?raw=true)

基于此架构，Istio的架构如下，具体就不介绍了，知道KFServing中底层的实现依赖即可。KF的所有服务都是以Istio的形式在跑的。

![KFS2](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFS2.png?raw=true)

## Knative

一套无服务架构方案。基于K8s和Istio实现蓝绿发布、回滚功能，监控应用请求，自动扩容和缩容，自动启动和销毁容器，根据名字生成网络访问相关的Service、Ingress等对象。Knative Serving就是KF Serving的核心。

![image-20210106153532761](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFS3.png?raw=true)

## KFServing架构分析

KFServing的架构包括两部分，分别是Data Plane 和 Control Plane。 Control Plane 主要负责管理模型的生命周期，而 Data Plane主要负责在模型部署完成后进行模型间的数据交互。

KFServing的整体架构如图8所示。在图中主要有3个组件，最外层的是K8s集群组件，包含K8s 集群的负载均衡器。在模型在K8s 集群中发布服务后，用户的预测请求首先被K8s 集群组件接收，在K8s 集群级别进行负载均衡(这一步是K8s自身行为，是可选的)。

![image-20210106154957092](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFS4.png?raw=true)

在Inference Service发布成功后，客户端可以发送一个预测请求，预测的工作流程大致如下：

用户的请求首先会被Istio的Ingress Gateway接收，Route的流量指向Activator，Activator在收到请求后会**自动启动**Pod，然后将流量转发过去。

如果发现此服务的Pod数为0(如果长时间没有用户请求，Knative会删除所有相关的Pod，在有用户请求时，快速恢复Pod或根据请求数量增加Pod数量)，则根据Control Plane中定义的canaryTraffic 的比例(在Inference Service spec 中定义)转发给 Default Traffic 路由或Canary Traffic 路由。

在多用户场景中，可能只有 Default Traffic，即当canaryTraffPercent的值为0时，所有请求都会转发给 Default Traffic路由。

Orchestrator 会对请求进行编排处理，在进行预测前，需要预先对接收的数据进行处理，如将人眼可以识别的图片转换为机器识别的二进制文件，然后发送给真正的Serving进行预测。

这些组件会异步执行，最后由Orchestrator汇总，返回统一的结果。到这里，整个预测工作流程结束。

在实际生产环境中，用户可能会针对不同的模型和场景发布不同的服务。例如，在开发环境、测试环境、实际应用环境中发布的服务是不同的，不同的模型(包括模型升级)之间也会同时发布或逐级发布不同的服务。

所以，可以将KF Service理解为一个模型服务单元，其优势是和K8s 深度集成。

其中KFServing Control Plane中的InfereceService字段如下表：

![image-20210106155617630](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFS5.png?raw=true)

一个InferenceService的实例：

```
apiVersion:"serving.kubeflow.org/v1alpha2"
kind:"InferenceService"
metadata:
	name:"flowers-sample"
spec:
	default:
		predictor:
		#90%的流量分配的Default Endpoint上。
			tensorflow:
				storageUri: "default_model_url"
	canaryTrafficPercent：10
	canary:
		predictor:
		#10的流量分配到Canary Endpint上。
			tensorflow:
				storageUri:"canary_model_ur1"
```

## KFServing Python SDK

利用Python对KFServing进行管理。创建、更新、删除、升级、回滚等。该SDK包括Server 和 Client两部分。我们主要是用Client。

![image](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFS6.png?raw=true)

详细的接口在GitHub的上有，https://github.com/kubeflow/kfserving/tree/master/python/kfserving。目前只是知道有这么一个服务发布的组件。等构建好整个机器学习应用之后再回过头来使用。