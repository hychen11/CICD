# K8S

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

其中CI主要依靠的Github Actions

CD主要依靠的ArgoCD

github添加dockerHub的access Token

https://github.com/hychen11/CICD/settings/secrets/actions

`DOCKERHUB_USERNAME` zjuchy

`DOCKERHUB_TOKEN` 

# Github Actions

create `.github/workflows/github-actions-demo.yml`

```
name: GitHub Actions Demo
run-name: ${{ github.actor }} is testing out GitHub Actions 🚀
on: [push]
jobs:
  Explore-GitHub-Actions:
    runs-on: ubuntu-latest
    steps:
      - run: echo "🎉 The job was automatically triggered by a ${{ github.event_name }} event."
      - run: echo "🐧 This job is now running on a ${{ runner.os }} server hosted by GitHub!"
      - run: echo "🔎 The name of your branch is ${{ github.ref }} and your repository is ${{ github.repository }}."
      - name: Check out repository code
        uses: actions/checkout@v4
      - run: echo "💡 The ${{ github.repository }} repository has been cloned to the runner."
      - run: echo "🖥️ The workflow is now ready to test your code on the runner."
      - name: List files in the repository
        run: |
          ls ${{ github.workspace }}
      - run: echo "🍏 This job's status is ${{ job.status }}."

```

# ArgoCD

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





![Argo CD Architecture](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)

