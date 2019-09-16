# 計算資源のプロビジョニング

Kubernetesには、コントロールプレーンとコンテナが最終的に実行されるワーカーノードをホストするマシンのセットが必要です。本実習では、安全で可用性の高いKubernetesクラスターを単一の[ゾーン]((https://cloud.google.com/compute/docs/regions-zones/regions-zones))で実行するために必要な計算資源をプロビジョニングします。

> デフォルトのゾーンおよびリージョンが、[前提条件](01-prerequisites.md#デフォルトのリージョンとゾーンの設定)セクションの説明通りに設定されていることを確認してください。

## ネットワーク

Kubernetesの[ネットワークモデル](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)は、コンテナとノードが相互に通信できるフラットなネットワークを想定しています。これが望ましくない場合は、[ネットワークポリシー](https://kubernetes.io/docs/concepts/services-networking/network-policies/)を使ってコンテナのグループが相互に、もしくは外部ネットワークエンドポイントと通信することを制限できます。

> ネットワークポリシーの設定は、本チュートリアルの対象外です。

### 仮想プライベートクラウドネットワーク(VPC)

本セクションではKubernetesクラスタをホストする専用の[仮想プライベートクラウド](https://cloud.google.com/compute/docs/networks-and-firewalls#networks)(VPC)を設定します。

`kubernetes-the-hard-way`という名前のカスタムVPCネットワークを作成します:

```
gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
```

[サブネット](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets)は、Kubernetesクラスター内の各ノードにプライベートIPアドレスを割り当てるのに十分なIPアドレス範囲でプロビジョニングする必要があります。

`kubernetes`という名前のサブネットを`kubernetes-the-hard-way`VPCネットワーク内に作成します:

```
gcloud compute networks subnets create kubernetes \
  --network kubernetes-the-hard-way \
  --range 10.240.0.0/24
```

> IPアドレス範囲`10.240.0.0/24`を指定すると、254個までのインスタンスまで作成することができます。

### Firewall Rules

内部通信にて全プロトコルを許可するファイアウォールルールを作成します:

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
  --allow tcp,udp,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

外部からのSSH、ICMP、およびHTTPS通信を許可するファイアウォールルールを作成します:

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network kubernetes-the-hard-way \
  --source-ranges 0.0.0.0/0
```

> KubernetesのAPIサーバーをリモートクライアントに公開するために[外部ロードバランサ](https://cloud.google.com/compute/docs/load-balancing/network/)が利用されます。

`Kubernetes-the-hard-way`VPCネットワークにおけるファイアウォールルール一覧を確認します:

```
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```

> 出力結果

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY
kubernetes-the-hard-way-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp
kubernetes-the-hard-way-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp
```

### Kubernetesの公開IPアドレス

KubernetesのAPIサーバの前面に置かれる外部ロードバランサにアタッチする静的IPアドレスを割り当てます:

```
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```

`kubernetes-the-hard`という名前の静的IPアドレスがデフォルトリージョンに作成されたことを確認します:

```
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```

> 出力結果

```
NAME                     REGION    ADDRESS        STATUS
kubernetes-the-hard-way  us-west1  XX.XXX.XXX.XX  RESERVED
```

## インスタンス

本実習のインスタンスでは、コンテナランタイムの[containerd](https://github.com/containerd/containerd)にて推奨される[Ubuntu Server](https://www.ubuntu.com/server) 18.04を使用します。各インスタンスは、Kubernetesのブートストラッピング処理を単純化するために、固定のプライベートIPアドレスでプロビジョニングされます。

### Kubernetesコントロールプレーン

Kubernetesコントロールプレーンをホストする3つのインスタンスを作成します:

```
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```

### Kubernetesワーカー

各ワーカーインスタンスは、KubernetesクラスターにおけるCIDR範囲からのPodサブネット割り当てを必要とします。Podサブネットの割り当ては、後のセクションにてコンテナネットワークの設定に使用します。`pod-cidr`インスタンスメタデータは、実行時にインスタンスを計算するためのPodサブネット割り当てを公開するために使用されます。

> KubernetesクラスターのCIDR範囲は、コントローラーマネージャーの`--cluster-cidr`フラグで定義されます。本チュートリアルでは、クラスターのCIDR範囲を`10.200.0.0/16に設定します。これは、254のサブネットをサポートします。

Kubernetesワーカーノードをホストするインスタンスを3つ作成します:

```
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

### 検証

デフォルトのゾーン内にあるインスタンスの一覧を表示します:

```
gcloud compute instances list
```

> 出力結果

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
controller-0  us-west1-c  n1-standard-1               10.240.0.10  XX.XXX.XXX.XXX  RUNNING
controller-1  us-west1-c  n1-standard-1               10.240.0.11  XX.XXX.X.XX     RUNNING
controller-2  us-west1-c  n1-standard-1               10.240.0.12  XX.XXX.XXX.XX   RUNNING
worker-0      us-west1-c  n1-standard-1               10.240.0.20  XXX.XXX.XXX.XX  RUNNING
worker-1      us-west1-c  n1-standard-1               10.240.0.21  XX.XXX.XX.XXX   RUNNING
worker-2      us-west1-c  n1-standard-1               10.240.0.22  XXX.XXX.XX.XX   RUNNING
```

## SSHアクセスの設定

SSHは、コントローラーおよびワーカーインスタンスの設定に使用されます。初めてインスタンスに接続するとSSHキーが生成され、プロジェクトまたはインスタンスのメタデータに格納されます ([インスタンスへの接続](https://cloud.google.com/compute/docs/instances/connecting-to-instance)のマニュアルを参照してください)。

試しに、`controller-0`インスタンスにSSHしてみます:

```
gcloud compute ssh controller-0
```

インスタンスに初めて接続する場合はSSHキーが生成されます。ターミナル上でパスフレーズを入力して続行します:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

この時点で生成されたSSHキーがアップロードされ、GCPのプロジェクトに保存されます::

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

SSHキーが更新されると`controller-0`インスタンスにログインします:

```
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1042-gcp x86_64)
...

Last login: Sun Sept 14 14:34:27 2019 from XX.XXX.XXX.XX
```

ターミナルで`exit`と入力し、`controller-0`インスタンスを終了します:

```
$USER@controller-0:~$ exit
```
> 出力結果

```
logout
Connection to XX.XXX.XXX.XXX closed
```

Next: [CA証明書のプロビジョニングとTLS証明書の生成](04-certificate-authority.md)
