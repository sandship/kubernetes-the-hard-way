# DNSクラスターアドオンのデプロイ

本実習では、[CoreDNS](https://coredns.io/)によってサポートされるDNSベースのサービスディスカバリを提供する[アドオン](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)を、Kubernetesクラスター内で稼働するアプリケーションに導入します。

## The DNS Cluster Add-on

クラスターアドオン`coredns`をデプロイします:

```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns-1.8.yaml
```

> 出力結果

```
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
```

Deploymentリソース`kube-dns`によって作られたPodの一覧を表示します:

```
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

> 出力結果

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-8494f9c688-hh7r2   1/1     Running   0          10s
coredns-8494f9c688-zqrj2   1/1     Running   0          10s
```

## 検証

Deploymentリソース`busybox`をデプロイします:

```
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

Deploymentリソース`busybox`によって作られたPodの一覧を表示します:

```
kubectl get pods -l run=busybox
```

> 出力結果

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Podリソース`busybox`のフルネームを取得します:

```
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

`busybox`の中で`kubernetes`のサービスに対するDNSルックアップを実行します:

```
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

> 出力結果

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

Next: [スモークテスト](13-smoke-test.md)
