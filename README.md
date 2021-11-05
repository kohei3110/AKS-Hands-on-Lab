![Microsoft Cloud Workshop](images/ms-cloud-workshop.png)

Azure Kubernets Service
Hands-on lab  
November 2021

<br />

**Contents**

1. Docker コンテナの仕組み（10 分）

2. Kubernetes 入門（10 分）

3. Azure Kubernetes Service 概要（10 分）

4. コンテナ化されたアプリケーションを Azure Kubernetes Service にデプロイ（30 分）

5. GitOps の考え方

## Requirements

- `Az` コマンド
- Bash

## ハンズオン手順

### AKS クラスター作成

- **環境変数の定義**

```bash
export RESOURCE_GROUP_NAME=gitops-demo-rg
export LOCATION=westus2
export CLUSTER_NAME=GitOpsDemoCluster
```

- **リソースグループの作成**

```bash
az group create -n $RESOURCE_GROUP_NAME -l $LOCATION
```

- **AKS クラスターの作成**

  - 単一ノードプール（3 ノード）
  - システム割り当てマネージド ID 
  - Azure CNI の有効化

```bash
az aks create -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME \
  --enable-managed-identity \
  --network-plugin azure
```

- **資格情報の取得**

```bash
az aks get-credentials -g $RESOURCE_GROUP_NAME -n $CLUSTER_NAME
```

- **確認**

```bash
kubectl get nodes
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-39017211-vmss000000   Ready    agent   74s   v1.20.9
aks-nodepool1-39017211-vmss000001   Ready    agent   73s   v1.20.9
aks-nodepool1-39017211-vmss000002   Ready    agent   72s   v1.20.9
```

### Flux CD のセットアップ

ローカル環境に Flux CD をインストール
**（※以下のコマンドは PowerShell を管理者権限で実行）**

```powershell
choco install flux
```

Flux のデプロイに必要な前提条件を満たしていることを確認

```bash
flux check --pre
► checking prerequisites
✔ Kubernetes 1.20.9 >=1.19.0-0
✔ prerequisites checks passed
```

GitHub アクセストークンを生成し、環境変数として定義（アクセストークン生成手順は[こちら](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)）

```bash
export GITHUB_TOKEN=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
export GITHUB_USER=kohei3110
export GITHUB_REPO=AKS-GitOps-Hands-on-Lab
export CLUSTER_NAME=GitOpsDemoCluster
```

`flux bootstrap` コマンド実行（Flux がインストール・同期を開始するまでの出力も示す）

```bash
flux bootstrap github \
--owner=$GITHUB_USER \
--repository=$GITHUB_REPO \
--branch=main \
--path=./clusters/$CLUSTER_NAME

► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/kohei3110/AKS-GitOps-Hands-on-Lab.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed sync manifests to "main" ("007a0ddf765151743a617134e0d874c1dc44ee78")
► pushing component manifests to "https://github.com/kohei3110/AKS-GitOps-Hands-on-Lab.git"
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ public key: ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDoVZUkzWVuD4yfrr+Y/fWd3Ts324EBbrnPk67UKOS6mUVszrR3Mmt+TeTcfOuX9qVStFq1gKEJXQMSbKqFs5+OprucFI9mBhIweM6XzVRdDN9pAJeXEBwSkLIs2Kd/FIpZZqKfeHcf5rQI59rde+2nm0VMJL9qBuumC3Iz5fxcAy1Cd1VQ3JXHkrX+fBEROPVHv2+XT+Nmm9IPusQM8XEaE5ZzlaNkCbY0aelduIxuLfroewuU5zX//pO96P+knYtlDbWNgJ+3tpesVCkD9rPYz8zyw7jB8jQUw8RYkET4JUg1lwGfOWiDQiJHDGgnjmE87dP6ZGlNLcKPvhWsFfC1
✔ configured deploy key "flux-system-main-flux-system-./clusters/GitOpsDemoCluster" for "https://github.com/kohei3110/AKS-GitOps-Hands-on-Lab"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("0456da6839479756a5284b3caf392b289af54f88")
► pushing sync manifests to "https://github.com/kohei3110/AKS-GitOps-Hands-on-Lab.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

```bash
flux check
► checking prerequisites
✔ Kubernetes 1.20.9 >=1.19.0-0
► checking controllers
✔ helm-controller: deployment ready
► ghcr.io/fluxcd/helm-controller:v0.12.1
✔ kustomize-controller: deployment ready
► ghcr.io/fluxcd/kustomize-controller:v0.16.0
✔ notification-controller: deployment ready
► ghcr.io/fluxcd/notification-controller:v0.18.1
✔ source-controller: deployment ready
► ghcr.io/fluxcd/source-controller:v0.16.1
✔ all checks passed
```

リポジトリに Flux がインストールされていることを確認

![alt text](./images/GitRepos.png)

```bash
git pull origin main
```

### アプリケーションのセットアップ

- **GitOps の構成**

  - "demoapp" というアプリケーション用に GitOps を構成する
  - Flux とアプリケーションの両方に同じリポジトリを使用しているため、同じ「flux-system」ソースを参照するKustomizationを作成する（Flux とアプリケーションに異なるリポジトリを使用することも可能）

```bash
flux create kustomization demoapp \
  --namespace=flux-system \
  --source=flux-system \
  --path="./manifests" \
  --prune=true \
  --validation=client \
  --interval=5m \
  --export > ./clusters/$CLUSTER_NAME/demoapp-kustomization.yaml
```

  - 上記の設定をもとに、`./manifests` ディレクトリを作成する
  - Flux によって、manifests ファイルの変更があった場合、`./manifests` ディレクトリの内容が同期される

```bash
mkdir manifests
```

  - 名前空間の作成

```bash
cat > ./manifests/namespace.yaml <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: demoapp
EOF
```

  - Deployment の作成

```bash
cat > ./manifests/deployment.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapp
  namespace: demoapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demoapp
  template:
    metadata:
      labels:
        app: demoapp
    spec:
      containers:
        - name: demoapp
          image: "mcr.microsoft.com/dotnet/core/samples:aspnetapp"
          ports:
            - containerPort: 80
              protocol: TCP
EOF
```

  - Service 作成

```bash
cat > ./manifests/service.yaml <<EOF
---
apiVersion: v1
kind: Service
metadata:
  name: demoapp
  namespace: demoapp
spec:
  type: ClusterIP
  selector:
    app: demoapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
EOF
```

```bash
git add clusters/GitOpsDemoCluster/demoapp-kustomization.yaml manifests/
git commit -m "add demoapp & manifests/"
git push -u origin master
```

  - Azure Portal から、flux-system の Reconciliation を確認

![alt text](./images/flex-system-reconciliation-log.png)

  - Kustomization によって最新の commit が適用されていることを確認

```bash
kubectl get kustomization -A
NAMESPACE     NAME          READY   STATUS                                                            AGE
flux-system   demoapp       True    Applied revision: main/31d729e7dc2d251fed5cd5f76693546095108178   4m54s 
flux-system   flux-system   True    Applied revision: main/31d729e7dc2d251fed5cd5f76693546095108178   13h  
```

  - アプリケーションが自動デプロイされていることを確認

```bash
kubectl -n demoapp get pod,deploy,svc
NAME                           READY   STATUS    RESTARTS   AGE
pod/demoapp-6b757bbc47-w5lnt   1/1     Running   0          5m59s
pod/demoapp-6b757bbc47-zjr29   1/1     Running   0          5m59s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/demoapp   2/2     2            2           5m59s

NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/demoapp   ClusterIP   10.0.199.123   <none>        80/TCP    5m59s
```

## 参考

- [GitOps and secret management with AKS, Flux CD, SOPS, and Azure Key Vault](https://techcommunity.microsoft.com/t5/azure-global/gitops-and-secret-management-with-aks-flux-cd-sops-and-azure-key/ba-p/2280068)
- [Flux Installation](https://fluxcd.io/docs/installation/#generic-git-server)