# K8S

k8s的service有多种

ClusterIP **集群内访问**，其他 Pod 可以访问这个 Service

NodePort 在每个节点的端口暴露给集群外部

LoadBalancer  云厂商提供公网 IP

ExternalName 转发到外部的 DNS 名称

```bash
brew install kubectl
kubectl version --client

brew install kind
kind create cluster --name dev-cluster
kubectl cluster-info
kubectl get nodes
kubectl get ns
```

输出应该显示集群信息、节点列表、命名空间等

# continuous integration and continuous delivery

```
          你（开发者）
               ↓
    ┌─────────────────────┐
    │  Push 到 GitHub 仓库 │   （触发 GitHub Actions）
    └─────────────────────┘
               ↓
    ┌─────────────────────┐
    │ GitHub Actions 构建镜像│  ➝ 构建并推送到 Docker Hub: zjuchy/cicd:latest
    └─────────────────────┘
               ↓
    ┌─────────────────────┐
    │    ArgoCD 发现变更    │  （你配置的 deployment.yaml 使用最新镜像）
    └─────────────────────┘
               ↓
    ┌─────────────────────┐
    │ 自动更新 Kubernetes   │
    └─────────────────────┘

```

其中CI主要依靠的Github Actions

CD主要依靠的ArgoCD

github添加dockerHub的access Token

https://github.com/hychen11/CICD/settings/secrets/actions

`DOCKERHUB_USERNAME` zjuchy

`DOCKERHUB_TOKEN` 

# Github Actions

注意这里不仅需要在github上创建repo，同时dockerhub也要创建repo！比如这里的zjuchy/cicd

这个`ci.yml`的作用就是告诉 GitHub Actions 去构建并推送镜像到 Docker Hub

create `.github/workflows/ci.yml`

```
(base) ➜  workflows git:(main) ✗ cat ci.yml 
name: Build and Push Docker Image to DockerHub

on:
  push:
    branches:
      - main  # 推送到 main 分支触发

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: zjuchy/cicd:latest

```

# ArgoCD

如果kind重新运行，也就是集群重新部署，这里也要重新运行argocd！

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
#上面的失效就用下面的，绕过浏览器
mkdir -p argo-install
curl -L https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -o argo-install/install.yaml
kubectl apply -n argocd -f argo-install/install.yaml

#在k8s集群里运行argocd，监听git仓库，hychen11/CICD
(base) ➜  ~ kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          151m
argocd-applicationset-controller-655cc58ff8-kmdjv   1/1     Running   0          151m
argocd-dex-server-7d9dfb4fb8-nfktm                  1/1     Running   0          151m
argocd-notifications-controller-6c6848bc4c-bxgjr    1/1     Running   0          151m
argocd-redis-656c79549c-4ts2x                       1/1     Running   0          151m
argocd-repo-server-856b768fd9-w6kzp                 1/1     Running   0          151m
argocd-server-99c485944-zj42f                       1/1     Running   0          151m
```

```shell
# step 0 
brew install argocd
# step 1 端口转发 ArgoCD Server 到本地，把集群内的 argocd-server 映射到你本地的 https://localhost:8080
kubectl port-forward svc/argocd-server -n argocd 8080:443
# step 2 打开另一个终端login argocd cli
argocd login localhost:8080 --username admin --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
# step 3
argocd app create cicd-app \
  --repo https://github.com/hychen11/CICD.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

这里argocd部署在k8s集群里，它的作用是持续监控 Git 仓库（比如你自己的 `hychen11/CICD`），自动同步仓库中的声明式配置到集群中，实现自动部署和配置管理

而我们本地也需要安装argocd，ArgoCD 是部署在 Kubernetes 集群内部的 GitOps 控制器，本地安装的 `argocd` CLI 只是连接它、控制它的工具。

* argocd-application-controller将 Git 仓库中的期望状态 **与 Kubernetes 中实际状态进行比较，并自动同步**。
* argocd-repo-server从 Git 仓库中拉取代码，解析 K8s YAML 或 Helm charts 等配置
* argocd-server 提供 **Web UI 和 API 接口**
* argocd-applicationset-controller支持动态批量生成应用（ApplicationSet CRD
* argocd-dex-server 负责用户认证
* argocd-notifications-controller 处理通知
* argocd-redis用于缓存 Git 仓库的解析内容、资源状态、session 等

```
       +---------------------+
       |   Git Repository    |
       +---------------------+
                |
                v
      +------------------------+
      |  argocd-repo-server    | ← 拉取和解析 Git 内容
      +------------------------+
                |
                v
      +-----------------------------+
      | argocd-application-controller | ← 控制资源同步 Apply 等
      +-----------------------------+
                |
                v
         [ Kubernetes Cluster ]

    ^                            |
    |                            v
[argocd CLI / Web UI] <---- argocd-server
             |
             +---(通过 dex-server 认证)

```

触发问题，可能因为国内原因无法拉取到镜像

```shell
argocd app get cicd-app #查看health问题
argocd app sync cicd-app #手动同步
kubectl get pods -n default -l app=cicd-app  #检查 Pod 状态
```

<img src="https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png" alt="Argo CD Architecture" style="zoom:33%;" />

登陆密码

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

k8s的配置文件`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cicd-app
  template:
    metadata:
      labels:
        app: cicd-app
    spec:
      containers:
        - name: cicd-app
          image: zjuchy/cicd:latest
          ports:
            - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: cicd-service
spec:
  selector:
    app: cicd-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: ClusterIP
```

创建一个`cicd-service` 的 Service，内部访问端口80

service会把收到的请求转发到Pod到5000端口，然后Service会找到所有标签 app=cicd-app的pod

`kustomization.yaml `

```yaml
resources:
 \- deployment.yaml
```

暴露端口的话 type是NodePort，适合本地测试

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cicd-service
spec:
  selector:
    app: cicd-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
      nodePort: 30001
  type: NodePort
```

```shell
kubectl get svc cicd-service
kubectl get nodes -o wide
kubectl get pods -l app=cicd-app
```

kind是模拟docker容器的，(Kubernetes In Docker)这里配置一个`kind-config.yaml` ，kind用于快速创建一个kubernetes cluster的

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30001
        hostPort: 30001
```

```shell
kind delete cluster    # 删除已有的集群（如果有）
kind create cluster --config kind-config.yaml
```

```css
[ Cluster 内部访问 80 端口 ] --> [ cicd-service (ClusterIP) ] --> [ cicd-app Pod :5000 ]
```





```
kind get clusters

```

