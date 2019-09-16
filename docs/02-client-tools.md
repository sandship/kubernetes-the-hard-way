# クライアントツールのインストール

本実習では、チュートリアルの実行に必要なコマンドである[cfssl](https://github.com/cloudflare/cfssl)、[cfssljson](https://github.com/cloudflare/cfssl)、[kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl)をインストールします。


## cfsslのインストール

[公開鍵基盤](https://ja.wikipedia.org/wiki/%E5%85%AC%E9%96%8B%E9%8D%B5%E5%9F%BA%E7%9B%A4)のプロビジョニングとTLS証明書の生成には、`cfssl`および`cfssljson`コマンドラインユーティリティが使用されます。

`cfssl`と`cfssljson`をダウンロードし、インストールします:

### macOS

```
curl -o cfssl https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/darwin/cfssl
curl -o cfssljson https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/darwin/cfssljson
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

macOSユーザーの中でビルド済みバイナリーに問題があった人は、[Homebrew](https://brew.sh)を使うと改善されるかもしれません:

```
brew install cfssl
```

### Linux

```
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/linux/cfssljson
```

```
chmod +x cfssl cfssljson
```

```
sudo mv cfssl cfssljson /usr/local/bin/
```

### 検証

インストールされた`cfssl`と`cfssljson`のバージョンが1.3.4以上であることを検証します:

```
cfssl version
```

> 出力結果

```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

```
cfssljson --version
```
```
Version: 1.3.4
Revision: dev
Runtime: go1.13
```

## kubectlのインストール

`kubectl`コマンドは、KubernetesのAPIサーバーとの対話に使用されます。公式リリースバイナリから`kubectl`をダウンロードしてインストールします:

### macOS

```
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/darwin/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### Linux

```
wget https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubectl
```

```
chmod +x kubectl
```

```
sudo mv kubectl /usr/local/bin/
```

### 検証

インストールされた`kubectl`のバージョンが1.15.3以上であることを検証します:

```
kubectl version --client
```

> 出力結果

```
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.3", GitCommit:"2d3c76f9091b6bec110a5e63777c332469e0cba2", GitTreeState:"clean", BuildDate:"2019-08-19T11:13:54Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [計算資源のプロビジョニング](03-compute-resources.md)
