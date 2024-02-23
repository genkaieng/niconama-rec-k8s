# niconama-rec-k8s

ニコ生の自動録画サーバーをKubernetesに構築する。

## 依存関係

- [Apache Kafka](https://kafka.apache.org/)
- [Argo Workflows](https://argoproj.github.io/workflows/)
- [genkaieng/nicopush-subscriber](https://github.com/genkaieng/nicopush-subscriber)
- [genkaieng/niconama-recorder](https://github.com/genkaieng/niconama-recorder)

注）Kubernetes上に [genkaieng/live-rec-k8s](https://github.com/genkaieng/live-rec-k8s) をデプロイをしてください。

## 手順

### 0. 事前準備

#### 0.1 live-rec-k8s

[genkaieng/live-rec-k8s](https://github.com/genkaieng/live-rec-k8s) の手順に従ってKubernetes上に構築しておく。

#### 0.2 git clone

```sh
git clone git@github.com:genkaieng/niconama-rec-k8s.git
```

#### 0.3 Kafkaにトピックを作成

[bitnami/kafka](https://hub.docker.com/r/bitnami/kafka) をKubernetes上にrunしてKafkaのAPIを叩いて操作します。

```sh
kubectl run kafka-client --image=bitnami/kafka --restart=Never --rm -it -- /bin/bash
```

Podが作られシェルが起動したら `/opt/bitnami/kafka/bin` に移動します。

```sh
cd /opt/bitnami/kafka/bin
```

KafkaのAPIを叩く。**nicopush** というトピックを作成します。

```sh
# トピック一覧
./kafka-topics.sh --list --bootstrap-server my-release-kafka:9092

# トピックを作成
./kafka-topics.sh --create --topic nicopush --bootstrap-server my-release-kafka:9092

# トピックをサブスクライブ（デバッグなどで必要であれば）
./kafka-console-consumer.sh --bootstrap-server my-release-kafka:9092 --topic nicopush --from-beginning
```

#### 0.4 MinIOにバケットを作成

ポートフォワーディングでMinIOの管理画面を開く

```sh
# ブラウザでhttp://localhost:9001を開く
kubectl port-forward svc/my-release-minio 9001:9001

# 認証情報を取得
# ユーザー名を表示
kubectl get secret my-release-minio -o jsonpath="{.data.root-user}" | base64 --decode
# パスワードを表示
kubectl get secret my-release-minio -o jsonpath="{.data.root-password}" | base64 --decode
```

左ペインのBucketsを開き、Create Bucketボタン押下。

バケットの作成画面が開いたら、バケット名に **niconama** と入力して、Create Bucketボタン押下。

![Screenshot from 2024-02-24 04-59-52](https://github.com/genkaieng/niconama-rec-k8s/assets/154831542/b66b7e02-2693-40a5-9477-f37c84bcfb1e)

バケットが作成される。

### 1. デプロイ

#### 1.1 Argo Workflowsをデプロイ

```sh
kubectl apply -f argo
```

#### 1.2 通知サーバーをデプロイ

ニココニのPush通知受信サーバー（[genkaieng/nicopush-subscriber](https://github.com/genkaieng/nicopush-subscriber)）と連携させることで録画を自動化します。（受信した通知はfluent-bitでKafkaに送っています。）

##### 1.2.1 ConfigMapの設定

https://github.com/genkaieng/niconama-rec-k8s/blob/6bf7ad4b7716734de42241941a5198ebf04111bb/configmap.yaml#L1-L24

Push通知を受信するためのキーペアなどを出力して、ConfigMapに設定する

```sh
docker run ghcr.io/genkaieng/nicopush-subscriber:v0.1.0 ./genkeys
```

ニコニコのセッションキーはブラウザのCookieに保存されているのでそれを指定する<br>
`F12キー押下 > アプリケーションタブ > Cookie > https://www.nicovideo.jp > user_sessionの行` に書かれている。

##### 1.2.2 デプロイ

```sh
kubectl apply -f .
```
