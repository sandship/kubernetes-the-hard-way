# リモートアクセス用のkubectl設定

本実習では、`admin`ユーザーの認証情報に基づいた`kubectl`コマンド用のkubeconfigファイルを生成します。

> 本実習で使用するコマンドは、管理クライアント証明書の生成に使用したディレクトリと同じディレクトリから実行してください。

## 管理者用Kubernetesコンフィグファイル

kubeconfigには接続先のKubernetes APIサーバーが必要です。HAをサポートするために、Kubernetes APIサーバの前面に配置した外部ロードバランサに割り当てられたIPアドレスが使用されます。

`admin`ユーザとして認証するのに適したkubeconfigファイルを生成します:

```
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## 検証

リモートにあるKubernetesクラスターの状態を確認します:

```
kubectl get componentstatuses
```

> 出力結果

```
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
```

リモートにあるKubernetesクラスター上にあるノードの一覧を表示します:

```
kubectl get nodes
```

> 出力結果

```
NAME       STATUS   ROLES    AGE    VERSION
worker-0   Ready    <none>   2m9s   v1.15.3
worker-1   Ready    <none>   2m9s   v1.15.3
worker-2   Ready    <none>   2m9s   v1.15.3
```

Next: [Podが使うネットワーク経路のプロビジョニング](11-pod-network-routes.md)
