# metallb部署记录

## concepts

使用的是metallb的layer2模式，使用虚拟地址池（10.0.0.1-10.0.0.39，后续可增加），与dc k8s cluster在同一个网段。metallb会在每一个节点上跑一个daemonset（名为speaker）用来进行ARP responding，**当外部请求load balancer的external ip时，实际上是ARP响应elected node（metallb选举出的领主节点）的mac地址，流量经此节点再到各个pod**，metallb还会维护一个deployment（controller）用来assign虚拟ip

e.g:

这里kubectl describe 某个load balancer service

https://tva1.sinaimg.cn/large/008eGmZEly1gn3p2d1u4qj317y04gwfu.jpg



## installation

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

这之后还无法运作，需要配置configmap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: dc-k8s-lb
      protocol: layer2
      addresses:
      - 10.0.0.1-10.0.0.39

```



