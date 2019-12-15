# 認証用Kubernetes設定ファイルの生成

本実習では[Kubernetesのコンフィグファイル](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)(kubeconfigとも呼ばれます)を生成します。これにより、KubernetesクライアントがKubernetesのAPIサーバーを特定して認証できるようになります。

## クライアント認証コンフィグ

本セクションでは、`controller manager`、`kubelet`、`kube-proxy`、および`scheduler`のクライアントと`admin`ユーザー用のkubeconfigファイルを生成します。

### Kubernetesの公開IPアドレス

kubeconfigには接続先のKubernetes APIサーバーが必要です。高可用性をサポートするために、Kubernetes APIサーバの前面に配置した外部ロードバランサに割り当てられたIPアドレスが使用されます。

静的IPアドレス`kubernetes-the-hard-way`を取得します:

```
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

### kubelet用Kubernetesコンフィグファイル

kubelet用のkubeconfigファイルを生成する際、kubeletのノード名に一致するクライアント証明書を使用する必要があります。これにより、kubeletがKubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/)によって適切に許可されます。

> 次のコマンドは、[TLS証明書の生成](04-certificate-authority.md)でSSL証明書を生成するときに使用したディレクトリと同じディレクトリで実行する必要があります。

各ワーカーノード用kubeconfigファイルを生成します:

```sh
for instance in worker-{0..2}; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

結果:

```
worker-0.kubeconfig
worker-1.kubeconfig
worker-2.kubeconfig
```

### kube-proxy用Kubernetesコンフィグファイル

`kube-proxy`サービス用kubeconfigファイルを生成します:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

結果:

```
kube-proxy.kubeconfig
```

### kube-controller-manager用Kubernetesコンフィグファイル

`kube-controller-manager`サービス用kubeconfigファイルを生成します:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

結果:

```
kube-controller-manager.kubeconfig
```


### kube-scheduler用Kubernetesコンフィグファイル

`kube-scheduler`サービス用kubeconfigファイルを生成します:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

結果:

```
kube-scheduler.kubeconfig
```

### admin用Kubernetesコンフィグファイル

`admin`ユーザー用kubeconfigファイルを生成します:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

結果:

```
admin.kubeconfig
```

## Kubernetesコンフィグファイルの配布

適切な`kubelet`及び`kube-proxy`用kubeconfigファイルを各ワーカーノード用インスタンスにコピーします:

```sh
for instance in worker-{0..2}; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

適切な`kube-controller-manager`及び`kube-scheduler`用kubeconfigファイルをコントロールプレーン用インスタンスにコピーします:

```sh
for instance in controller-{0..2}; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```

Next: [データ暗号化の設定とキーの生成](06-data-encryption-keys.md)
