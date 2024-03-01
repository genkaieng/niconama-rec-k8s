# niconama-rec-k8s

ニコ生の自動録画サーバーをKubernetesに構築する。

※ Kubernetes上に事前に [live-rec-k8s](https://github.com/genkaieng/live-rec-k8s) をデプロイしている必要があります。

## 手順

### 0. 事前準備

#### 0.1 live-rec-k8s をデプロイ

このプロジェクトは、[Apache Kafka](https://kafka.apache.org/), [Argo Workflows](https://argoproj.github.io/workflows/), [MinIO](https://min.io/) に依存します。

[genkaieng/live-rec-k8s](https://github.com/genkaieng/live-rec-k8s) の手順に従ってデプロイする。

#### 0.2 git clone

```sh
git clone git@github.com:genkaieng/niconama-rec-k8s.git
```

### 1. 設定を追加

#### 1.1 ニコニコの通知を受信するためのキーの生成&設定

ニコニコの通知受信は [genkaieng/nicopush-subscriber](https://github.com/genkaieng/nicopush-subscriber) が行っています。

Webプッシュの受信に必要なキーを生成します。

```sh
docker run ghcr.io/genkaieng/nicopush-subscriber:v0.1.0 ./genkeys
```

生成したキーを [configmap.yaml](https://github.com/genkaieng/niconama-rec-k8s/blob/6bf7ad4b7716734de42241941a5198ebf04111bb/configmap.yaml#L5-L12) に設定する。

https://github.com/genkaieng/niconama-rec-k8s/blob/6bf7ad4b7716734de42241941a5198ebf04111bb/configmap.yaml#L5-L12

#### 1.2 ニコニコのセッションキーを設定

1. ブラウザで[ニコニコのサイト](https://www.nicovideo.jp/)を開く
2. 開発者ツールを開く（F12キー押下）
3. `アプリケーションタブ > Cookie > https://www.nicovideo.jp` を開く
4. **user_session**の値を [configmap.yaml](https://github.com/genkaieng/niconama-rec-k8s/blob/6bf7ad4b7716734de42241941a5198ebf04111bb/configmap.yaml#L21-L24) に設定する。

https://github.com/genkaieng/niconama-rec-k8s/blob/6bf7ad4b7716734de42241941a5198ebf04111bb/configmap.yaml#L21-L24

また、自動録画するコミュニティIDをカンマ区切りで[設定](https://github.com/genkaieng/niconama-rec-k8s/blob/6bf7ad4b7716734de42241941a5198ebf04111bb/configmap.yaml#L24)してください。

### 2. KafkaにTopicを作成

Kubernetesクラスタにkafkaのクライアントをrunする

```sh
kubectl run kafka-client --image=bitnami/kafka --restart=Never --rm -it -- /bin/bash
```

シェルに入れたら `/opt/bitnami/kafka/bin` に移動

```sh
cd /opt/bitnami/kafka/bin
```

KafkaのAPIを叩いて、 `nicopush` トピックを作成する。

```sh
# Topic一覧
./kafka-topics.sh --list --bootstrap-server my-release-kafka:9092

# Topic作成
./kafka-topics.sh --create --topic nicopush --bootstrap-server my-release-kafka:9092
```

### 3. デプロイ

#### 3.1 Argoをデプロイ

```sh
kubectl apply -f argo
```

#### 3.2 niconicoの通知サーバーをデプロイ

```sh
kubectl apply -f .
```
