# Kubernetesコントロールプレーンのブートストラップ

本実習では、3つのインスタンスでコントロールプレーンをブートストラップして可用性の高い構成を実現します。また、KubernetesのAPIサーバーをリモートクライアントに公開する外部ロードバランサも作成します。各ノードには、Kubernetes API Server、Scheduler、Controller Managerの各コンポーネントがインストールされます。

## 前提条件

本実習のコマンドは`controller-0`、`controller-1`、`controller-2`の各コントロールプレーン用インスタンスで実行する必要があります。`gcloud`コマンドを使用して各コントローラインスタンスにログインします。例:

```
gcloud compute ssh controller-0
```

### tmuxを使った並列なコマンド実行

[tmux](https://github.com/tmux/tmux/wiki)を使用すると複数のインスタンスで同時にコマンドを実行できます。前提条件の[tmuxを使った並列なコマンド実行](01-prerequisites.md#tmuxを使った並列なコマンド実行)セクションを参照してください。

## Kubernetesコントロールプレーンのプロビジョニング

Kubenretesのコンフィグ用ディレクトリを作成します:

```
sudo mkdir -p /etc/kubernetes/config
```

### Kubernetesコントローラー用バイナリーのダウンロードとインストール

Kubernetesの公式リリースバイナリーをダウンロードします:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
```

Kubernetesバイナリーをインストールします:

```
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### Kubernetes APIサーバーの設定

```
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

クラスターのメンバーにAPIサーバーを通知するためにインスタンスの内部IPアドレスが使用されます。以下のコマンドで現在作業中のインスタンスが持つ内部IPアドレスを取得します:

```
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

```
REGION=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/project/attributes/google-compute-default-region)
```

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $REGION \
  --format 'value(address)')
```

systemdユニットファイル`kube-apiserver.service`を作成します:

```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetesコントローラーマネージャーの設定

kubeconfig`kube-controller-manager`を以下の場所に配置します:

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

systemdユニットファイル`kube-controller-manager.service`を作成します:

```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Kubernetesスケジューラーの設定

kubeconfig`kube-scheduler`を以下の場所に配置します:

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

設定ファイル`kube-scheduler.yaml`を作成します:

```sh
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

systemdユニットファイル`kube-scheduler.service`を作成します:

```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### コントローラーのサービスを起動

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

> KubernetesのAPIサーバーが完全に初期化されるまでには最大10秒かかります。

### HTTPのヘルスチェックを有効化

[Google Network Load Balancer](https://cloud.google.com/compute/docs/load-balancing/network)を使用して3つのAPIサーバーにトラフィックを分散し、各APIサーバーがTLS接続を終端してクライアント証明書を検証できるようにします。ネットワークロードバランサーは、HTTPヘルスチェックのみをサポートします。つまり、APIサーバーによって公開されたHTTPSエンドポイントは使用できません。回避策として、nginxを使用してHTTPのヘルスチェックをプロキシすることができます。本セクションではnginxをインストールし、ポート`80`でHTTPヘルスチェックを受け入れ、`https://127.0.0.1:6443/healthz` 上のAPIサーバへの接続をプロキシするように設定します。

> APIサーバーエンドポイント`/healthz`は、デフォルトで認証を必要としません。

HTTPヘルスチェックを処理するために基本的なWebサーバーをインストールします:

```
sudo apt-get update
sudo apt-get install -y nginx
```

```sh
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

```
{
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
}
```

```
sudo systemctl restart nginx
```

```
sudo systemctl enable nginx
```

### 検証

```
kubectl cluster-info --kubeconfig admin.kubeconfig
```

```
Kubernetes control plane is running at https://127.0.0.1:6443
```

nginxがHTTPヘルスチェックをプロキシしているかどうかテストします:

```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 02 May 2021 04:19:29 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
Cache-Control: no-cache, private
X-Content-Type-Options: nosniff
X-Kubernetes-Pf-Flowschema-Uid: c43f32eb-e038-457f-9474-571d43e5c325
X-Kubernetes-Pf-Prioritylevel-Uid: 8ba5908f-5569-4330-80fd-c643e7512366

ok
```

> 上記のコマンドは各コントローラノード`controller-0`、`controller-1`、`controller-2`にて忘れずに実行してください。

## RBACを使ったKubeletの認可

本セクションでは、RBACのアクセス権を設定して、KubernetesのAPIサーバーが各ワーカーノード上のKubeletにアクセスできるようにします。メトリクスやログを取得したり、Pod内でコマンドを実行するためにはKubelet APIへのアクセスが必要です。

> 本チュートリアルでは、Kubeletの`--authorization-mode`フラグを`Webhook`に設定します。Webhookモードでは、[SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) APIを使用して認可を判定します。

本セクションのコマンドはクラスター全体に影響するため、1つのコントローラーノードから1回実行するだけで大丈夫です。

```
gcloud compute ssh controller-0
```

`system:kube-apiserver-to-kubelet`という名前の[ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole)を作成し、Kubelet APIにアクセスしたり、Podの管理に関連する一般的なタスクを実行したりするための権限を付与します:

```sh
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Kubernetes APIサーバーは、`--kubelet-client-certificate`フラグで定義されているクライアント証明書を使用して、`kubernetes`ユーザーとしてKubeletに認証します。

ClusterRole`system:kube-apiserver-to-kubelet`を`kubernetes`ユーザーにバインドします:

```sh
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## Kubernetesのフロントエンドロードバランサー

本セクションでは、Kubernetes APIサーバーの前段に外部ロードバランサをプロビジョニングします。生成されるロードバランサには静的IPアドレス`kubernetes-the-hard-way`がアタッチされます。

> 本チュートリアルで作成したインスタンスには、本セクションを完了する権限がありません。**インスタンスの作成に使用したのと同じ作業マシンから次のコマンドを実行します**。

### ネットワークロードバランサーのプロビジョニング

外部ロードバランサーネットワークリソースを作成します:

```sh
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  gcloud compute http-health-checks create kubernetes \
    --description "Kubernetes Health Check" \
    --host "kubernetes.default.svc.cluster.local" \
    --request-path "/healthz"

  gcloud compute firewall-rules create kubernetes-the-hard-way-allow-health-check \
    --network kubernetes-the-hard-way \
    --source-ranges 209.85.152.0/22,209.85.204.0/22,35.191.0.0/16 \
    --allow tcp

  gcloud compute target-pools create kubernetes-target-pool \
    --http-health-check kubernetes

  gcloud compute target-pools add-instances kubernetes-target-pool \
   --instances controller-0,controller-1,controller-2

  gcloud compute forwarding-rules create kubernetes-forwarding-rule \
    --address ${KUBERNETES_PUBLIC_ADDRESS} \
    --ports 6443 \
    --region $(gcloud config get-value compute/region) \
    --target-pool kubernetes-target-pool
}
```

### 検証

> 本チュートリアルで作成したインスタンスには、本セクションを完了する権限がありません。**インスタンスの作成に使用したのと同じ作業マシンから次のコマンドを実行します**。

静的IPアドレス`kubernetes-the-hard-way`を取得します:

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

Kubernetesのバージョン情報を取得するHTTPリクエストを発行します:

```
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> 出力結果

```json
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.0",
  "gitCommit": "cb303e613a121a29364f75cc67d3d580833a7479",
  "gitTreeState": "clean",
  "buildDate": "2021-04-08T16:25:06Z",
  "goVersion": "go1.16.1",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Kubenretesワーカーノードのブートストラップ](09-bootstrapping-kubernetes-workers.md)
