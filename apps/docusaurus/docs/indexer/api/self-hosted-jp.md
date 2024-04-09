---
title: "セルフホスト型インデクサーAPI(Japanese)"
slug: "indexer-api-self-hosted-jp"
---

# セルフホスト型インデクサーAPI

import BetaNotice from '../../../src/components/\_indexer_beta_notice_jp.mdx';

<BetaNotice />

このガイドでは、セルフホスト型インデクサーAPIのセットアップについて説明します。

:::caution
現在このガイドは、インデクサーAPIのプロセッサー部分の実行方法のみを解説しています。このガイドは、トランザクションストリームサービスからトランザクションを消費するプロセッサーを実行し、解析し、データベースへ保存します。現時点ではこのシステムにAPIを接続する方法は説明していません。
:::

## 前提条件

- 有効なユーザーとデータベースを備えたPostgreSQLインスタンスの実行が必要です。この例では、ユーザー`postgres`とデータベース`indexer`を呼びます。
- Dockerを使用する場合は、Dockerがインストールされている必要があります。[インストールガイド](https://docs.docker.com/get-docker/)。

## 構成

サービスを実行するには、構成ファイルを定義する必要があります。このテンプレートから始めます。

```yaml
health_check_port: 8084
server_config:
  processor_config:
    type: default_processor
  postgres_connection_string: postgresql://postgres:@localhost:5432/indexer
  indexer_grpc_data_service_address: 127.0.0.1:50051
  indexer_grpc_http2_ping_interval_in_secs: 60
  indexer_grpc_http2_ping_timeout_in_secs: 10
  auth_token: AUTH_TOKEN
```

ここから、これらのフィールドのいくつかの値を変更することが必要になる場合があります。そのうちのいくつかを見てみましょう。

### `processor_name`

:::info
サービスの1つのインスタンスは1つのプロセッサのみを実行します。複数のプロセッサを実行する場合は、サービスの複数のインスタンスを実行する必要があります。この場合、同じデータベースを使用するかどうかはあなた次第です。
:::

これはあなたが実行したいプロセッサです。利用可能なプロセッサは[ここ](https://github.com/aptos-labs/aptos-indexer-processors/blob/main/rust/processor/src/processors/mod.rs#L23)で確認できます。例：

- `coin_processor`
- `ans_processor`
- `token_v2_processor`

### `postgres_connection_string`

 これは、PostgreSQLデータベースへの接続文字列です。`postgresql://<username>:<password>@<host>:<port>/<database>`という形式でなければなりません。

:::caution
これをDockerデスクトップ環境から実行している場合 (MacOSまたはWindowsを使用している場合はそうである可能性が高い)、`postgres_connection_string`の代わりに`postgresql://host.docker.internal:5432/indexer`に設定する必要があります。Dockerデスクトップでは、これがバイナリがホストネットワークに到達する方法です。
:::

### `indexer_grpc_data_service_address`

これは、トランザクションストリームサービスのURLです。 Labs-Hostedインスタンスを使用している場合は、[このページ](../txn-stream/labs-hosted)で各ネットワークのURLを見つけることができます。インデックスしたいネットワークの正しいURLを選択してください。このサービスをローカルで実行している場合、値は`127.0.0.1:50051`である必要があります。

### `auth_token`

これは、トランザクションストリームサービスへの接続に使用される認証トークンです。Labs-Hostedインスタンスを使用している場合は、APIゲートウェイを使用してAPIキーを取得できます。詳細については、[このページ](/indexer/txn-stream/labs-hosted)を御覧下さい。


## ソースコードで実行する

Clone the repo:
リポジトリのクローンを作成します。

```
# SSH
git clone git@github.com:aptos-labs/aptos-indexer-processors.git

# HTTPS
git clone https://github.com/aptos-labs/aptos-indexer-processors.git
```

サービスのディレクトリに移動します。

```
cd aptos-indexer-processors
cd rust/processor
```

サービスを実行します。

```
cargo run --release -- -c config.yaml
```

## Dockerで実行

<!--
This doesn't actually work this very moment because:

1. We don't yet publish the image as indexer-processor-rust
2. We don't tag it as latest.

We'll do that soon though: https://aptos-org.slack.com/archives/C04PRP1K1FZ/p1692732083583659
-->

Dockerでサービスを実行するには、以下のコマンドを使用します。

```
docker run -it --network host --mount type=bind,source=/tmp/config.yaml,target=/config.yaml aptoslabs/indexer-processor-rust -c /config.yaml
```

このコマンドは、コンテナをホストネットワークにバインド(結合)し、ホストからコンテナに構成ファイルをマウントします。この特定の呼び出しでは、ホスト内の構成ファイルが`/tmp/config.yaml`にあることを前提としています。

こちらのDockerHubのイメージを御覧下さい: https://hub.docker.com/r/aptoslabs/indexer-processor-rust/tags。
