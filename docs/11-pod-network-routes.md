# Podネットワークルートのプロビジョニング

ノードにスケジュールされたPodは、ノードが持つPod CIDR範囲からIPアドレスを受け取ります。この時点ではネットワーク[経路](https://cloud.google.com/compute/docs/vpc/routes)が欠落しているため、Podは異なるノード上で動作している他のPodと通信できません。

本実習では、ノードのPod CIDR範囲をノードの内部IPアドレスにマップするための経路を各ワーカーノード上に作成します。

> Kubernetesのネットワーキングモデルを実装する方法は[他にも]((https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) )あります。

## ルーティングテーブル

ここでは、`kubernetes-the-hard-way`VPCネットワーク上に経路を作成するために必要な情報を収集します。

各ワーカーインスタンスの内部IPアドレスとPod CIDR範囲を表示します:

```
for instance in worker-0 worker-1 worker-2; do
  gcloud compute instances describe ${instance} \
    --format 'value[separator=" "](networkInterfaces[0].networkIP,metadata.items[0].value)'
done
```

> 出力結果

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## 経路

各ワーカーインスタンス用のネットワーク経路を作成します:

```
for i in 0 1 2; do
  gcloud compute routes create kubernetes-route-10-200-${i}-0-24 \
    --network kubernetes-the-hard-way \
    --next-hop-address 10.240.0.2${i} \
    --destination-range 10.200.${i}.0/24
done
```

VPCネットワーク`kubernetes-the-hard-way`内の経路一覧を表示します:

```
gcloud compute routes list --filter "network: kubernetes-the-hard-way"
```

> 出力結果

```
NAME                            NETWORK                  DEST_RANGE     NEXT_HOP                  PRIORITY
default-route-081879136902de56  kubernetes-the-hard-way  10.240.0.0/24  kubernetes-the-hard-way   1000
default-route-55199a5aa126d7aa  kubernetes-the-hard-way  0.0.0.0/0      default-internet-gateway  1000
kubernetes-route-10-200-0-0-24  kubernetes-the-hard-way  10.200.0.0/24  10.240.0.20               1000
kubernetes-route-10-200-1-0-24  kubernetes-the-hard-way  10.200.1.0/24  10.240.0.21               1000
kubernetes-route-10-200-2-0-24  kubernetes-the-hard-way  10.200.2.0/24  10.240.0.22               1000
```

Next: [DNSクラスターアドオンのデプロイ](12-dns-addon.md)
