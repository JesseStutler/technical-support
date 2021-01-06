## Pipeline

Pipeline 实现了一个工作流模型，所谓工作流或者称之为流水线(pipeline), 可以将其当做一个有向无环图（DAG）。

![img](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/Pipeline1.jpg?raw=true)

如图中，每一个节点在KLP(KubeFlow Pipeline)中被称为一个组件(component), 其会处理真正的逻辑，比如预处理，数据清洗，模型训练等等。每一个组件负责的功能不同，但有一个共同点，即组件都是以 Docker 镜像的方式被打包，以容器的方式被运行的。

实验（experiment）是一个工作空间，在其中可以针对流水线尝试不同的配置。运行（run）是流水线的一次执行，用户在执行的过程中可以看到**每一步**的**输出文件**，以及**日志**。步（step）是组件的一次运行，**步与组件**的关系就像是**运行与流水线**的关系一样。步输出工件（step output artifacts）是在组件的一次运行结束后输出的，能被系统的前端理解并渲染可视化的文件。

![img](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/Pipeline2.jpg?raw=true)

需要自己定义，然后一般是针对在训练或者引用过程中的输出才有ROC曲线。

Pipeline的实现主要分为两步：

1.定义每个组件

例如：

```
def tf_train_op(transformed_data_dir, schema: 'GcsUri[text/json]', learning_rate: float, hidden_layer_size: int, steps: int, target: str, preprocess_module: 'GcsUri[text/code/python]', training_output: 'GcsUri[Directory]', step_name='training'):

  return dsl.ContainerOp(

​    name = step_name,

​    image = 'gcr.io/ml-pipeline/ml-pipeline-kubeflow-tf-trainer:0.0.42',

​    arguments = [

​      '--transformed-data-dir', transformed_data_dir,

​      '--schema', schema,

​      '--learning-rate', learning_rate,

​      '--hidden-layer-size', hidden_layer_size,

​      '--steps', steps,

​      '--target', target,

​      '--preprocessing-module', preprocess_module,

​      '--job-dir', training_output,

​    ],

​    file_outputs = {'train': '/output.txt'}

)
```

这里的例子定义了一个组件，其负责模型的训练。这一组件只是简单地定义了一个 Docker 容器，利用了**镜像中已有的 TensorFlow 框架**进行训练。在更加高级的用法中，组件可以从镜像开始完全自定义。有时用户需要自定义其使用的组件，首先需要打包一个 Docker 镜像，这个镜像是组件的依赖，每一个组件的运行，**就是一个 Docker 容器**。其次需要为其定义一个 **python 函数**，描述组件的输入输出等信息，这一定义是为了能够让流水线理解组件在流水线中的结构，有几个输入节点，几个输出节点，等等。接下来组件的使用就与普通的组件并无二致了。

2.规定执行关系和执行顺序

```
# The pipeline definition

@dsl.pipeline(

 name='TFX Taxi Cab Classification Pipeline Example',

 description='Example pipeline that does classification with model analysis based on a public BigQuery dataset.'

)

def taxi_cab_classification(

  output,

  project,

  column_names=dsl.PipelineParam(name='column-names', value='gs://ml-pipeline-playground/tfx/taxi-cab-classification/column-names.json'),

  key_columns=dsl.PipelineParam(name='key-columns', value='trip_start_timestamp'),

  train=dsl.PipelineParam(name='train', value='gs://ml-pipeline-playground/tfx/taxi-cab-classification/train.csv'),

  evaluation=dsl.PipelineParam(name='evaluation', value='gs://ml-pipeline-playground/tfx/taxi-cab-classification/eval.csv'),

  ...):

  ...

  preprocess = dataflow_tf_transform_op(train, evaluation, schema, project, preprocess_mode, preprocess_module, transform_output)

  training = tf_train_op(preprocess.output, schema, learning_rate, hidden_layer_size, steps, target, preprocess_module, training_output)

...
```

在流水线中，由输入输出关系会确定图上的边以及方向。在上例中，training 的输入是 preprocess 的输出，因此 preprocess 会有一条边指向 training，代表两者的拓扑顺序。在定义好流水线后，可以通过 python 中实现好的流水线客户端提交到系统中运行。

 

### 实现思路：(源自https://zhuanlan.zhihu.com/p/49443232)

整个的架构可以分为五个部分，分别是 **ScheduledWorkflow CRD** 以及其 operator，**流水线前端**，**流水线后端**， **Python SDK** 和 **persistence agent**。

 

ScheduledWorkflow CRD 扩展了 **argoproj/argo** 的 Workflow 定义。这也是流水线项目中的核心部分，它负责真正地在 Kubernetes 上按照拓扑序创建出对应的容器完成流水线的逻辑。

 

Python SDK 负责构造出流水线，并且根据流水线构造出 ScheduledWorkflow 的 YAML 定义，随后将其作为参数传递给流水线系统的后端服务。

 

后端服务依赖关系存储数据库（如 MySQL）和对象存储（如 Amazon S3），处理所有流水线中的 CRUD 请求。

 

前端负责可视化整个流水线的过程，以及获取日志，发起新的运行等。

 

Persistence agent 负责把数据从 Kubernetes Master 的 etcd 中同步到后端服务的关系型数据库中，其实现的方式与 CRD operator 类似，通过 informer 来监听 Kubernetes apiserver 对应资源实现。

![img](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/Pipeline3.jpg?raw=true)

具体操作使用的时候再查看源码，还要涉及到DSL，等一些其他，要用主要就是看SDK的函数接口。