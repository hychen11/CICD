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

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

![Argo CD Architecture](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)