# Kubernetes The Hard Way(日本語版)

### 【注意】この翻訳は、1.15.3時点で動作が確認できているバージョンの凍結版です。動作確認できた次のバージョンがリリースされない限り、しばらくの間更新の予定はありません。

本チュートリアルでは、Kubernetesを地道にセットアップする方法を説明します。本ガイドは、Kubernetesクラスターを立てるための自動化コマンドを探している人には向いていません。そういう人は、[Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine)や[Getting Started Guides](https://kubernetes.io/docs/setup)を御覧ください。

Kubernetes The Hard Wayは勉強に適しています。長い道のりを経て、Kubernetesクラスターを起動するのに必要な各タスクを理解してください。

> 本チュートリアルの結果はプロダクションレディではなく、コミュニティからのサポートも限られていますが、だからといって勉強しない理由にはなりません！

## Copyright

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.


## 対象者

本チュートリアルの対象読者は、Kubernetesの本番クラスターのサポートを予定していて、すべてがどのように連携しているかを理解したいという人です。

## クラスターの詳細

Kubernetes The Hard Wayは、コンポーネント間のエンドツーエンドの暗号化とRBAC認証を使用して、HAなKubernetesクラスターをブートストラップする手順を説明します。

* [kubernetes](https://github.com/kubernetes/kubernetes) v1.15.3
* [containerd](https://github.com/containerd/containerd) v1.2.9
* [coredns](https://github.com/coredns/coredns) v1.6.3
* [cni](https://github.com/containernetworking/cni) v0.7.1
* [etcd](https://github.com/coreos/etcd) v3.4.0

## Labs

本チュートリアルは、Google Cloud Platformへのアクセス権があることを前提としています。GCPは基本的なインフラストラクチャ要件に使用されますが、本チュートリアルで学習したレッスンは他のプラットフォームにも適用できます。

* [前提条件](docs/01-prerequisites.md)
* [クライアントツールのインストール](docs/02-client-tools.md)
* [計算資源のプロビジョニング](docs/03-compute-resources.md)
* [CA証明書のプロビジョニングとTLS証明書の生成](docs/04-certificate-authority.md)
* [認証用Kubernetes設定ファイルの生成](docs/05-kubernetes-configuration-files.md)
* [データ暗号化の設定とキーの生成](docs/06-data-encryption-keys.md)
* [etcdクラスターのブートストラップ](docs/07-bootstrapping-etcd.md)
* [Kubernetesコントロールプレーンのブートストラップ](docs/08-bootstrapping-kubernetes-controllers.md)
* [Kubenretesワーカーノードのブートストラップ](docs/09-bootstrapping-kubernetes-workers.md)
* [リモートアクセス用のkubectl設定](docs/10-configuring-kubectl.md)
* [Podネットワークルートのプロビジョニング](docs/11-pod-network-routes.md)
* [DNSクラスターアドオンのデプロイ](docs/12-dns-addon.md)
* [スモークテスト](docs/13-smoke-test.md)
* [お掃除](docs/14-cleanup.md)
