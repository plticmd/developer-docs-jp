---
title: "ローカルで実行する(Japanese)"
slug: "indexer-txn-stream-local-development-jp"
---

# トランザクションストリームサービスをローカルで実行する

import BetaNotice from '../../../src/components/\_indexer_beta_notice_jp.mdx';

<BetaNotice />

:::info
これは、ARM上のmacOS13およびx86_64上のDebian11でテストされています。
:::

カスタムプロセッサを構築するときは、ローカル開発スタックに対して開発すると役立つ場合があります。トランザクションストリームサービスは、複雑な複数のコンポーネントシステムです。ローカル開発を支援するためPythonスクリプトを提供し、Docker構成ファイルをラップしシステム全体を設定しています。

このスクリプトは以下を設定します。

- インデクサーGRPCストリームが有効な単一ノードテストネット。
- Redis インスタンス。
- トランザクションストリームサービス。(以下のコンポーネントが含まれます)
  - [cache-worker](https://github.com/aptos-labs/aptos-core/tree/main/ecosystem/indexer-grpc/indexer-grpc-cache-worker): 
  ノードからトランザクションを取得し、Redisに保存します。
  - [file-store](https://github.com/aptos-labs/aptos-core/tree/main/ecosystem/indexer-grpc/indexer-grpc-file-store): 
  Redisからトランザクションを取得し、ファイルシステムに保存します。
  - [data-service](https://github.com/aptos-labs/aptos-core/tree/main/ecosystem/indexer-grpc/indexer-grpc-data-service): 
  GRPCストリーム経由でダウンストリームクライアントにトランザクションを提供します。トランザクションの経過時間に応じて、キャッシュまたはファイルストアからプルします。
- 共有ボリュームとネットワークですべてを接続します。

トランザクションストリームサービスの構造については[こちら](/indexer/txn-stream)をご覧下さい。Docker構成ファイルについては[こちら](https://github.com/aptos-labs/aptos-core/blob/main/docker/compose/indexer-grpc/docker-compose.yaml)
を御覧下さい。

## 前提条件

ローカル開発スクリプトを使用するには、以下がインストールされている必要があります。

- Python 3.8+: [インストールガイド](https://docs.python-guide.org/starting/installation/#python-3-installation-guides).
- Poetry: [インストールガイド](https://python-poetry.org/docs/#installation).
- Docker: [インストールガイド](https://docs.docker.com/get-docker/).
- Docker Compose v2:これは、最新のDockerインストールのデフォルトでインストールする必要があります。以下のコマンドで確認します。

```bash
docker-compose version --short
```

- grpcurl: [インストールガイド](https://github.com/fullstorydev/grpcurl#installation)
- OpenSSL

## 準備

aptos-coreリポジトリのクローンを作成します。

```
# HTTPS
git clone https://github.com/aptos-labs/aptos-core.git

# SSH
git clone git@github.com:aptos-labs/aptos-core.git
```

`testsuite`ディレクトリに移動します。

```
cd aptos-core
cd testsuite
```

Pythonの依存関係をインストールします。

```
poetry install
```

## スクリプトを実行する

### サービスを開始する

```
poetry run python indexer_grpc_local.py start
```

コマンドが終了すると、これが成功したことがわかり、以下のように表示されます。

```
Attempting to stream from indexer grpc for 10s
Stream finished successfully
```

### サービスを停止する

```
poetry run python indexer_grpc_local.py stop
```

### データを消去する

サービスを開始、停止、再度開始すると、同じローカルテストネットデータが再利用されます。ローカルのテストネットを消去して最初からやり直す場合は、以下のコマンドを実行できます。

```
poetry run python indexer_grpc_local.py wipe
```

## ローカルサービスを使用する

以下の構成値を使用して、カスタムプロセッサなどからローカルのトランザクションストリームサービスに接続できます。

```
indexer_grpc_data_service_address: 127.0.0.1:50052
auth_token: dummy_token
```

以下のアドレスでノードに接続できます。

```
http://127.0.0.1:8080/v1
```

## デバッグ

### ARMシステムの使用法

ARMプロセッサ搭載のマシン(M1/M2 Mac等)を使用しているなら、スクリプトはARMプロセッサを検出し、適切な環境変数を設定して正しいイメージが使用される様にする必要があります。これに問題がある場合は、以下の環境変数を設定してみてください。

```bash
export DOCKER_DEFAULT_PLATFORM=linux/amd64
```

さらに、Dockerデスクトップで以下の設定が正しいことを確認して下さい。

- 有効: 環境設定 > 一般 > 仮想化フレームワークの使用
- 有効: 環境設定 > 一般 > Docker Compose V2を使用する
- 無効: 開発中の機能 -> Appleシリコンのx86/amd64エミュレーションにRosettaを使用

このスクリプトはLinux ARMシステムではテストされていません。

### Redisの起動に失敗する

スクリプトを実行する前に、以下の環境変数を設定してみてください。

```bash
export REDIS_IMAGE_REPO=arm64v8/redis
```

### キャッシュワーカーがクラッシュループしているか、`Redisの最新バージョンへの更新に失敗した`と記録される

データを消去します。

```bash
poetry run python indexer_grpc_local.py wipe
```

これは、履歴データが失われることを意味します。
