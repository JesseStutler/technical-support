# traefik部署记录

version: traefik2.4

非官网install traefik部分的helm chart形式部署，部署参照：https://www.qikqiak.com/post/traefik-2.1-101/ 一步步的kubectl create -f xxx.yaml

配置文件目录：compute08.dc /root/traefik



## 修改部分

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: traefik
  namespace: kube-system
  labels:
    app: traefik
spec:
  selector:
    matchLabels:
      app: traefik
  template:
    metadata:
      labels:
        app: traefik
    spec:
      serviceAccountName: traefik-ingress-controller
      containers:
      - image: traefik:2.4
        name: traefik
        ports:
        - name: web
          containerPort: 80
          hostPort: 80
        - name: websecure
          containerPort: 443
          hostPort: 443
        args:
        - --log.level=INFO
        - --accesslog
        - --entryPoints.web.address=:80
        - --entryPoints.websecure.address=:443
        - --api=true
        - --api.dashboard=true
        - --providers.kubernetescrd
        
      
```



### dashboard.yaml

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: kube-system
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`traefik.dc`)
    kind: Rule
    services:
    - name: api@internal
      kind: TraefikService


```

