![Microsoft Cloud Workshop](images/ms-cloud-workshop.png)

Azure Kubernetes Cluster
Hands-on lab  
November 2021

<br />

**Table of Contents**

- [**1. Azure Kubernetes Service クラスターの作成**](#--1-azure-kubernetes-service-----------)
- [**2. アプリケーションのデプロイ**](#--2----------------)
- [**3. アプリケーションへのネットワークアクセスを有効化**](#--3---------------------------)
- [**4. AKS ノードにおけるコンピューティングコストの最適化**](#--4-aks--------------------------)
- [**5. Helm を使用したアプリケーションとパッケージ管理**](#--5-helm------------------------)
  * [5-1. リポジトリからアプリケーションをデプロイ](#5-1---------------------)
  * [5-2. サンプルチャートを作成しデプロイ](#5-2-----------------)

**Contents**

## **1. Azure Kubernetes Service クラスターの作成**

- 環境変数定義

```bash
$ export RESOURCE_GROUP=AKS-Hands-on-Lab
$ export CLUSTER_NAME=aks-contoso-video
$ export ACR_NAME=acrhandsonlabdcircle
```

- AKS クラスターの作成

```bash
$ az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --node-count 2 \
    --enable-addons http_application_routing \
    --generate-ssh-keys \
    --node-vm-size Standard_B2s \
    --network-plugin azure
```

- ノードプールの追加

```bash
$ az aks nodepool add \
    --resource-group $RESOURCE_GROUP \
    --cluster-name $CLUSTER_NAME \
    --name userpool \
    --node-count 2 \
    --node-vm-size Standard_B2s
```

- 資格情報取得

```bash
$ az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
$ cat ~/.kube/config
```

- クラスターに接続できることを確認

```bash
$ kubectl get nodes

NAME                                STATUS   ROLES   AGE     VERSION
aks-nodepool1-33081063-vmss000000   Ready    agent   11m     v1.20.9
aks-nodepool1-33081063-vmss000001   Ready    agent   11m     v1.20.9
aks-userpool-33081063-vmss000000    Ready    agent   2m33s   v1.20.9
aks-userpool-33081063-vmss000001    Ready    agent   2m17s   v1.20.9
```

## **2. アプリケーションのデプロイ**

- マニフェストファイルの作成

```bash
$ touch deployment.yaml
$ code .
```

- 以下のコードセクションを追記

```bash
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: contoso-website
spec:
  selector: # Define the wrapping strategy
    matchLabels: # Match all pods with the defined labels
      app: contoso-website # Labels follow the `name: value` template
  template: # This is the template of the pod inside the deployment
    metadata:
      labels:
        app: contoso-website
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - image: mcr.microsoft.com/mslearn/samples/contoso-website
          name: contoso-website
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
          ports:
            - containerPort: 80
              name: http
```

- Pod がラップされていることを確認

```bash
$ kubectl apply -f ./deployment.yaml
$ kubectl get pods

NAME                             READY   STATUS    RESTARTS   AGE
contoso-website-97988f7c-nf8f9   1/1     Running   0          55s

$ kubectl describe pods
Name:         contoso-website-97988f7c-nf8f9
Namespace:    default
Priority:     0
・・・
```

## **3. アプリケーションへのネットワークアクセスを有効化**

- Service マニフェストファイルの作成

```bash
$ touch service.yaml
$ code .
```

- 以下のコードセクションを追記

```bash
#service.yaml
apiVersion: v1
kind: Service
metadata:
  name: contoso-website
spec:
  type: ClusterIP
  selector:
    app: contoso-website
  ports:
    - port: 80 # SERVICE exposed port
      name: http # SERVICE port name
      protocol: TCP # The protocol the SERVICE will listen to
      targetPort: http # Port to forward to in the POD
```

- Service のデプロイ

```bash
$ kubectl apply -f ./service.yaml
$ kubectl get service contoso-website

NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
contoso-website   ClusterIP   10.0.40.132   <none>        80/TCP    8s
```

- Ingress マニフェストファイルの作成

```bash
$ touch ingress.yaml
$ code .
```

- クラスターへのアクセスが許可されるホストの FQDN の設定

```bash
az aks show \
  -g $RESOURCE_GROUP \
  -n $CLUSTER_NAME \
  -o tsv \
  --query addonProfiles.httpApplicationRouting.config.HTTPApplicationRoutingZoneName

6fa7a55d2bdb41b9a199.westus2.aksapp.io
```

- 以下のコードセクションを追記

```bash
#ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: contoso-website
  annotations:
    kubernetes.io/ingress.class: addon-http-application-routing
spec:
  rules:
    - host: contoso.<zone-name> # Which host is allowed to enter the cluster
      http:
        paths:
          - backend: # How the ingress will handle the requests
              service:
               name: contoso-website # Which service the request will be forwarded to
               port:
                 name: http # Which port in that service
            path: / # Which path is this rule referring to
            pathType: Prefix # See more at https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types    
```

- Ingress のデプロイ

```bash
$ kubectl apply -f ./ingress.yaml
$ kubectl get ingress contoso-website

NAME              CLASS    HOSTS                                            ADDRESS          PORTS   AGE
contoso-website   <none>   contoso.6fa7a55d2bdb41b9a199.westus2.aksapp.io   20.115.168.210   80      2m9s
```

- 接続確認

```bash
$ curl -LI contoso.6fa7a55d2bdb41b9a199.westus2.aksapp.io -o /dev/null -w '%{http_code}\n' -s

200
```

- ブラウザからも接続

![alt text](./images/sample-app.png)

## **4. AKS ノードにおけるコンピューティングコストの最適化**

- **ノードプールの手動スケーリング**

特定の間隔で特定の期間にわたって実行されるワークロードを実行している場合、ノード プールのサイズを手動でスケーリングするのが、ノードのコストを制御する方法です。

```bash
$ az aks nodepool scale \
    --resource-group $RESOURCE_GROUP \
    --cluster-name $CLUSTER_NAME \
    --name userpool \
    --node-count 0
```

- ノード一覧を取得

```bash
$ kubectl get nodes

NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-33081063-vmss000000   Ready    agent   50m   v1.20.9
aks-nodepool1-33081063-vmss000001   Ready    agent   50m   v1.20.9
```

- **スポットノードプールの活用**

- サブスクリプションにて `spotpoolpreview` 機能を有効化

```bash
$ az feature register --namespace "Microsoft.ContainerService" --name "spotpoolpreview"
$ az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/spotpoolpreview')].{Name:name,State:properties.state}"
$ az provider register --namespace Microsoft.ContainerService
```

- aks-preview CLI インストール

```bash
$ az extension add --name aks-preview
```

- スポットノードプールを AKS クラスターに追加

```bash
$ az aks nodepool add \
    --resource-group $RESOURCE_GROUP \
    --cluster-name $CLUSTER_NAME \
    --name spotpool01 \
    --enable-cluster-autoscaler \
    --max-count 3 \
    --min-count 1 \
    --priority Spot \
    --eviction-policy Delete \
    --spot-max-price -1 \
    --no-wait
```

- スポットノードプールの確認

```bash
$ az aks nodepool show \
    --resource-group $RESOURCE_GROUP \
    --cluster-name $CLUSTER_NAME \
    --name spotpool01

{
・・・
  "enableAutoScaling": true,
・・・
  "maxCount": 3,
  "maxPods": 30,
  "minCount": 1,
・・・
  "scaleSetEvictionPolicy": "Delete",
  "scaleSetPriority": "Spot",
  "spotMaxPrice": -1.0,
・・・
}
```

- Pod をスポットノードプールにデプロイ

```bash
$ touch spot.yaml
$ code .
```

- 以下のコードセクションを追記

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "kubernetes.azure.com/scalesetpriority"
    operator: "Equal"
    value: "spot"
    effect: "NoSchedule"
```

- スポットノードプールにアプリケーションをデプロイ

```bash
$ kubectl apply -f spot.yaml
```

- 確認

```bash
$ kubectl get pods -o wide

NAME                             READY   STATUS    RESTARTS   AGE   IP             NODE                                 NOMINATED NODE   READINESS GATES
contoso-website-97988f7c-png52   1/1     Running   0          44m   10.240.0.59    aks-nodepool1-33081063-vmss000001    <none>           <none>
nginx                            1/1     Running   0          15s   10.240.0.105   aks-spotpool01-33081063-vmss000001   <none>           <none>
```

- **Kubernetes 用の Azure Policy を構成**

- 以下のリソースプロバイダーを登録

    - Azure Kubernetes Service
    - Azure Policy

```bash
$ az provider register --namespace Microsoft.ContainerService
$ az provider register --namespace Microsoft.PolicyInsights
```

- アドオンのインストールを有効化

```bash
$ az feature register --namespace Microsoft.ContainerService --name AKS-AzurePolicyAutoApprove
$ az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/AKS-AzurePolicyAutoApprove')].   {Name:name,State:properties.state}"
$ az provider register -n Microsoft.ContainerService
```

- **Azure Policy アドオンの有効化**

```bash
$ az aks enable-addons \
    --addons azure-policy \
    --name $CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP
```

- `kube-system` 名前空間に azure ポリシー ポッドがインストールされていること、および gatekeeper-system 名前空間にゲートキーパー ポッドがインストールされていることを確認

```bash
$ kubectl get pods -n kube-system

azure-policy-66489c49db-427vw                                     1/1     Running   0          24m
azure-policy-webhook-7bf799979b-vbrrz                             1/1     Running   0          24m

$ kubectl get pods -n gatekeeper-system

gatekeeper-audit-6fccbf6649-hft6b        1/1     Running   0          25m
gatekeeper-controller-68dbf74b86-kqqnr   1/1     Running   0          25m
gatekeeper-controller-68dbf74b86-wtbq4   1/1     Running   0          25m
```

- 最新のアドオンがインストールされていることを確認

```bash
$ az aks show \
 --resource-group $RESOURCE_GROUP\
 --name $CLUSTER_NAME \
 -o table --query "addonProfiles.azurepolicy.config.version"
```
- **組み込みのポリシー割り当てを実施**

新しい Azure Policy を構成するには、Azure portal の `ポリシー` サービスを使用します。

- [Azure Portal](https://portal.azure.com/)にログイン

- ポータルの上部にある検索バーで、**ポリシー** を検索して選択

- 左側のメニュー ペインの `作成` で、`割り当て` を選択

- 上部のメニュー バーで、`ポリシーの割り当て` を選択

- `基本` タブで、各設定に次の値を入力

| 設定 | 値 |
| :--- | :--- |
|スコープ	|
|Scope | 省略記号ボタンを選択します。`スコープ` ペインが表示されます。 `サブスクリプション` で、自分のリソース グループが保持されているサブスクリプションを選択します。 `リソース グループ` で、ハンズオンで使用しているリソースグループを選んでから `選択` を選択します。 |
| 除外 | 空のままにします。|
|基本操作	|
|ポリシー定義 | 省略記号ボタンを選択します。 `使用可能な定義` ペインが表示されます。 `検索` ボックスに、**CPU**と入力して選択項目を絞り込みます。 `ポリシー定義` タブで、`Kubernetes cluster containers CPU and memory resource limits should not exceed the specified limits(Kubernetes クラスター コンテナーの CPU およびメモリ リソースの制限が指定された制限を超えないようにする)`を選択します。 その後、`選択` を選択します。|
| 割り当て名 | 既定値をそのまま使用します。|
|Description | 空のままにします。|
| ポリシーの適用 | このオプションが `有効` に設定されていることを確認します。|
| 割り当て担当者 | 既定値をそのまま使用します。|

- `パラメーター` タブで、各設定に次の値を入力

| 設定 | 値 |
| :--- | :--- |
| 許可される CPU ユニットの最大数 | 値を **200m** に設定します。 この値は、ポリシーにより、ワークロードのマニフェスト ファイルに指定した、ワークロードのリソース要求値とワークロードの制限値の両方に照合されます。|
| 許可されるメモリの最大バイト数 | 値を **256Mi** に設定します。 この値は、ポリシーにより、ワークロードのマニフェスト ファイルに指定した、ワークロードのリソース要求値とワークロードの制限値の両方に照合されます。|

- `修復`タブは既定のままとする

```bash
$ code policy.yaml
```

以下のコードセクションを追記

```bash
# policy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 500Mi
```

- Pod のデプロイを試みると、エラーが返ってくることを確認

```bash
$ kubectl delete pod nginx
$ kubectl apply -f policy.yaml

```

- マニフェストファイルを以下のように修正

```bash
# policy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

- Pod をデプロイ

```bash
$ kubectl apply -f policy.yaml
$ kubectl get pods
```

## **5. Helm を使用したアプリケーションとパッケージ管理**

### 5-1. リポジトリからアプリケーションをデプロイ

- **prometheus-community** リポジトリを追加

```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
```

- 試しに `README.md` を確認

```bash
$ helm show readme prometheus-community/kube-prometheus-stack
# kube-prometheus-stack

Installs the [kube-prometheus stack](https://github.com/prometheus-operator/kube-prometheus), a collection of Kubernetes manifests, ・・・
```

- Prometheus をデプロイ

```bash
$ helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --debug
・・・
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace dev get pods -l "release=kube-prometheus-stack"
```

- デプロイされたことを確認

```bash
$ kubectl get pods -l "release=kube-prometheus-stack"
NAME                                                   READY   STATUS           RESTARTS   AGE
kube-prometheus-stack-operator-767bbf894f-xh7xv        1/1     Running          0          104s
kube-prometheus-stack-prometheus-node-exporter-8hgh9   1/1     Running          0          104s
kube-prometheus-stack-prometheus-node-exporter-pd4fp   1/1     Running          0          104s
```

- リリースの確認

```bash
$ helm list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART
        APP VERSION
kube-prometheus-stack   dev             1               2021-11-07 16:46:45.7126935 +0900 JST   deployed        kube-prometheus-stack-19.2.3 0.50.0
```

- kube-prometheus の StatefulSet のラベルを確認

```bash
$ kubectl get sts --show-labels
NAME                                              READY   AGE     LABELS
prometheus-kube-prometheus-stack-prometheus       1/1     4m9s    app.kubernetes.io/instance=kube-prometheus-stack,app.kubernetes.io/managed-by=Helm,app.kubernetes.io/part-of=kube-prometheus-stack,app.kubernetes.io/version=19.2.3,app=kube-prometheus-stack-prometheus,chart=kube-prometheus-stack-19.2.3,heritage=Helm,operator.prometheus.io/name=kube-prometheus-stack-prometheus,operator.prometheus.io/shard=0,release=kube-prometheus-stack

$ helm show values prometheus-community/kube-prometheus-stack | grep commonLabels
commonLabels: {}
```

- **myLabel: prometheus-test** というラベルを付与

```bash
$ echo "commonLabels: { myLabel: prometheus-test }" > config.yaml

$ helm upgrade -f config.yaml kube-prometheus-stack prometheus-community/kube-prometheus-stack
Release "kube-prometheus-stack" has been upgraded. Happy Helming!
NAME: kube-prometheus-stack
LAST DEPLOYED: Sun Nov  7 16:57:32 2021
NAMESPACE: dev
STATUS: deployed
REVISION: 2
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace dev get pods -l "release=kube-prometheus-stack"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

- 確認

```bash
$ kubectl get sts --show-labels
NAME                                              READY   AGE   LABELS
prometheus-kube-prometheus-stack-prometheus       1/1     12m   ・・・
myLabel=prometheus-test,・・・
```

- ロールバック

```bash
$ helm rollback kube-prometheus-stack 1
Rollback was a success! Happy Helming!
```

- Revision を確認

```bash
$ helm history kube-prometheus-stack
REVISION        UPDATED                         STATUS          CHART                           APP VERSION     DESCRIPTION
1               Sun Nov  7 16:46:45 2021        superseded      kube-prometheus-stack-19.2.3    0.50.0          Install complete
2               Sun Nov  7 16:57:32 2021        superseded      kube-prometheus-stack-19.2.3    0.50.0          Upgrade complete
3               Sun Nov  7 17:01:06 2021        deployed        kube-prometheus-stack-19.2.3    0.50.0          Rollback to 1
```

- ラベルの確認

```bash
$ helm get values --revision 1 kube-prometheus-stack
USER-SUPPLIED VALUES:
null

$ helm get values --revision 2 kube-prometheus-stack
USER-SUPPLIED VALUES:
commonLabels:
  myLabel: prometheus-test

$ helm get values --revision 3 kube-prometheus-stack
USER-SUPPLIED VALUES:
null
```

- アンインストール

```bash
$ helm uninstall kube-prometheus-stack
release "kube-prometheus-stack" uninstalled
```

- 確認

```bash
$ helm list
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION

```


### 5-2. サンプルチャートを作成しデプロイ

- チャートを作成

```bash
$ helm create helm-sample
Creating helm-sample
```

- 確認

```bash
$ find helm-sample/
helm-sample/
helm-sample/.helmignore
helm-sample/Chart.yaml
helm-sample/charts
helm-sample/templates
helm-sample/templates/deployment.yaml
helm-sample/templates/hpa.yaml
helm-sample/templates/ingress.yaml
helm-sample/templates/NOTES.txt
helm-sample/templates/service.yaml
helm-sample/templates/serviceaccount.yaml
helm-sample/templates/tests
helm-sample/templates/tests/test-connection.yaml
helm-sample/templates/_helpers.tpl
helm-sample/values.yaml
```

- インストール前に DRY-RUN

```bash
$ helm install helm-sample --debug --dry-run ./helm-sample/
NAME: helm-sample
LAST DEPLOYED: Sun Nov  7 19:29:29 2021
NAMESPACE: dev
STATUS: pending-install
REVISION: 1
USER-SUPPLIED VALUES:
・・・
```

- インストール（デプロイ）

```bash
$ helm install helm-sample --debug ./helm-sample/
・・・
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace dev -l "app.kubernetes.io/name=helm-sample,app.kubernetes.io/instance=helm-sample" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace dev $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace dev port-forward $POD_NAME 8080:$CONTAINER_PORT
```

- `values.yaml` の以下の値をアンコメント

```bash
・・・
autoscaling:
・・・
    targetMemoryUtilizationPercentage: 80
```

- アップグレード前に DRY-RUN

```bash
$ helm upgrade helm-sample --debug --dry-run ./helm-sample/
・・・
autoscaling:
・・・
  targetMemoryUtilizationPercentage: 80  ★値が反映されていることを確認
```

- アップグレード

```bash
$ helm upgrade helm-sample --debug ./helm-sample/
```

- Revision の確認

```bash
$ helm ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
helm-sample     dev             2               2021-11-07 19:35:25.0844057 +0900 JST   deployed        helm-sample-0.1.0       1.16.0
```









































- Azure Container Registry 作成

```bash
$ az acr create --name $ACR_NAME --resource-group $RESOURCE_GROUP --sku Standard
```

- Helm リポジトリの追加

```bash
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

- Helm チャートの検索

```bash
$ helm search repo ingress-nginx

NAME                            CHART VERSION   APP VERSION     DESCRIPTION
ingress-nginx/ingress-nginx     4.0.6           1.0.4           Ingress controller for Kubernetes using NGINX a...
```

- Helm グラフで作成されるイメージを ACR にインポート

```bash
REGISTRY_NAME=<REGISTRY_NAME>
CONTROLLER_REGISTRY=k8s.gcr.io
CONTROLLER_IMAGE=ingress-nginx/controller
CONTROLLER_TAG=v0.48.1
PATCH_REGISTRY=docker.io
PATCH_IMAGE=jettech/kube-webhook-certgen
PATCH_TAG=v1.5.1
DEFAULTBACKEND_REGISTRY=k8s.gcr.io
DEFAULTBACKEND_IMAGE=defaultbackend-amd64
DEFAULTBACKEND_TAG=1.5

az acr import --name $ACR_NAME --source $CONTROLLER_REGISTRY/$CONTROLLER_IMAGE:$CONTROLLER_TAG --image $CONTROLLER_IMAGE:$CONTROLLER_TAG
az acr import --name $ACR_NAME --source $PATCH_REGISTRY/$PATCH_IMAGE:$PATCH_TAG --image $PATCH_IMAGE:$PATCH_TAG
az acr import --name $ACR_NAME --source $DEFAULTBACKEND_REGISTRY/$DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG --image $DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG
```

- Helm チャートの実行

```bash
ACR_URL=$ACR_NAME.azurecr.io

# Create a namespace for your ingress resources
kubectl create namespace ingress-basic

# Use Helm to deploy an NGINX ingress controller
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress-basic \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.image.registry=$ACR_URL \
    --set controller.image.image=$CONTROLLER_IMAGE \
    --set controller.image.tag=$CONTROLLER_TAG \
     --set controller.image.digest="" \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.image.registry=$ACR_URL \
    --set controller.admissionWebhooks.patch.image.image=$PATCH_IMAGE \
    --set controller.admissionWebhooks.patch.image.tag=$PATCH_TAG \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.image.registry=$ACR_URL \
    --set defaultBackend.image.image=$DEFAULTBACKEND_IMAGE \
    --set defaultBackend.image.tag=$DEFAULTBACKEND_TAG
```
















