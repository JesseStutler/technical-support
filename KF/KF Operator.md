# Operator

每一种机器学习框架都有一个Operator，比如Tensorflow的就是TFJob(也就是TF Operator 的一个CRD)，类似还有PyTorch,XGBoost等。

## K8s CRD

通过CRD在K8s API增加自定义资源。

### Custom Resources 和 Custom Controllers

前者是对k8s API的扩展，代表K8s的一个定制化安装。后者将资源设置为用户想要的状态，通常和前者结合使用，独立于K8s集群本身的生命周期。

Operator就是这两者结合使用的例子，允许我们将特殊应用编码至k8s的扩展API中。



## TFJob

### TFJob创建流程图：

!["image"](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFOp1.jpg?raw=true)

### TFJob对象：

apiVersion、kind、metadata、spec。其中tfReplicaSpecs为spec字段的数据类型，也是最重要的。

![img](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFOp2.jpg?raw=true)

在k8s中用kubectl get crd tfjobs.kubeflow.org -o yaml 查看TFJob CRD的定义如下：

  - ```
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      annotations:
        kubectl.kubernetes.io/last-applied-configuration: |
          {"apiVersion":"apiextensions.k8s.io/v1beta1","kind":"CustomResourceDefinition","metadata":{"annotations":{},"labels":{"app.kubernetes.io/component":"tfjob","app.kubernetes.io/instance":"tf-job-crds-v1.0.0","app.kubernetes.io/managed-by":"kfctl","app.kubernetes.io/name":"tf-job-crds","app.kubernetes.io/part-of":"kubeflow","app.kubernetes.io/version":"v1.0.0"},"name":"tfjobs.kubeflow.org"},"spec":{"additionalPrinterColumns":[{"JSONPath":".status.conditions[-1:].type","name":"State","type":"string"},{"JSONPath":".metadata.creationTimestamp","name":"Age","type":"date"}],"group":"kubeflow.org","names":{"kind":"TFJob","plural":"tfjobs","singular":"tfjob"},"scope":"Namespaced","subresources":{"status":{}},"validation":{"openAPIV3Schema":{"properties":{"spec":{"properties":{"tfReplicaSpecs":{"properties":{"Chief":{"properties":{"replicas":{"maximum":1,"minimum":1,"type":"integer"}}},"PS":{"properties":{"replicas":{"minimum":1,"type":"integer"}}},"Worker":{"properties":{"replicas":{"minimum":1,"type":"integer"}}}}}}}}}},"versions":[{"name":"v1","served":true,"storage":true}]}}
      creationTimestamp: "2020-12-31T07:31:56Z"
      generation: 1
      labels:
        app.kubernetes.io/component: tfjob
        app.kubernetes.io/instance: tf-job-crds-v1.0.0
        app.kubernetes.io/managed-by: kfctl
        app.kubernetes.io/name: tf-job-crds
        app.kubernetes.io/part-of: kubeflow
        app.kubernetes.io/version: v1.0.0
      name: tfjobs.kubeflow.org
      resourceVersion: "3660"
      selfLink: /apis/apiextensions.k8s.io/v1beta1/customresourcedefinitions/tfjobs.kubeflow.org
      uid: 4274d18b-4b3a-11eb-bcbe-080027e07adb
    spec:
      additionalPrinterColumns:
    
      - JSONPath: .status.conditions[-1:].type
        name: State
        type: string
      - JSONPath: .metadata.creationTimestamp
        name: Age
        type: date
          conversion:
        strategy: None
          group: kubeflow.org
          names:
        kind: TFJob
        listKind: TFJobList
        plural: tfjobs
        singular: tfjob
          scope: Namespaced
          subresources:
        status: {}
          validation:
        openAPIV3Schema:
          properties:
            spec:
              properties:
                tfReplicaSpecs:
                  properties:
                    Chief:
                      properties:
                        replicas:
                          maximum: 1
                          minimum: 1
                          type: integer
                    PS:
                      properties:
                        replicas:
                          minimum: 1
                          type: integer
                    Worker:
                      properties:
                        replicas:
                          minimum: 1
                          type: integer
          version: v1
          versions:
      - name: v1
        served: true
        storage: true
        status:
          acceptedNames:
        kind: TFJob
        listKind: TFJobList
        plural: tfjobs
        singular: tfjob
          conditions:
      - lastTransitionTime: "2020-12-31T07:31:56Z"
        message: no conflicts found
        reason: NoConflicts
        status: "True"
        type: NamesAccepted
      - lastTransitionTime: null
        message: the initial names have been accepted
        reason: InitialNamesAccepted
        status: "True"
        type: Established
          storedVersions:
      - v1
    ```

    

### 故障定位：

当TFJob出现问题，主要分为4步去检查工作。

1.检查作业状态

kubectl -n ${NAMESPACE} get tfjobs -o yaml ${JOB_NAME}

2.输出如果不包含作业状态则表示作业规范无效。

3.查看相关Pod日志。

kubectl -n kubeflow logs \`kubectl get pods --selector=name=tf-job-operator -o jsonpath='{.items[0].metadata.name}'\`

4.检查作业看Pod是否创建了

常见失败原因：

查看K8s集群资源是否足够；Pod尝试挂载一个不存在或者不可用的PVC；Docker镜像问题，如网络，**镜像策略限制**

### TFJob Python SDK

pip install kubeflow-tfjob

![img](https://github.com/JesseStutler/technical-support/blob/master/assets/KF/KFOp3.jpg?raw=true)

详细API用法见https://github.com/kubeflow/tf-operator



PyTorch以及XGBoost的operator用到的时候再去看。