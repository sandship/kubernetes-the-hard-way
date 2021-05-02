# 前提条件

## Google Cloud Platform

本チュートリアルでは、Kubernetesクラスターのブートストラップに必要な計算資源のプロビジョニングを一から効率的に行うために[Google Cloud Platform](https://cloud.google.com/)を活用しています。300ドル分の無料クレジットに[サインアップ](https://cloud.google.com/free/)してください。

本チュートリアルの[推定実行コスト](https://cloud.google.com/products/calculator#id=873932bc-0840-4176-b0fa-a8cfd4ca61ae)は1時間あたり$0.23(1日あたり$5.50)です。

> 本チュートリアルに必要な計算資源は、Google Cloud Platformの無料枠を超えています。

## Google Cloud Platform SDK

### Google Cloud SDKのインストール

Google Cloud SDKの[ドキュメント](https://cloud.google.com/sdk/)に従って `gcloud` コマンドをインストールし、設定してください。

Google Cloud SDKのバージョンが338.0.0以上であることを確認してください:

```
gcloud version
```

### デフォルトのリージョンとゾーンの設定

本チュートリアルでは、デフォルトのリージョンとゾーンが既に設定されている前提で進められます。

はじめて `gcloud` コマンドをお使いの場合、`init` を使うと最も簡単に初期設定が行えます:

```
gcloud init
```

その後、ご自身のGoogleユーザーの認証情報でgcloudがCloud Platformにアクセスすることを必ず確認してください:

```
gcloud auth login
```

次に、デフォルトのリージョンを設定します:

```
gcloud config set compute/region us-west1
```

次に、デフォルトのゾーンを設定します:

```
gcloud config set compute/zone us-west1-c
```

> 追加で利用できるリージョンやゾーンを確認するには、`gcloud compute zones list` を使ってください。

## tmuxを使った並列なコマンド実行

[tmux](https://github.com/tmux/tmux/wiki)を使用すると、複数のcomputeインスタンスで同時にコマンドを実行できます。本チュートリアルでは、同じコマンドを複数のコンピュートインスタンスで実行する必要がある場合があります。その場合、tmuxを使用して、プロビジョニングプロセスを高速化するために同期を有効にした複数のペインにウィンドウを分割することを検討してください。

> tmuxの使用はオプションであり、このチュートリアルを完了するために必須ではありません。

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [クライアントツールのインストール](02-client-tools.md)
