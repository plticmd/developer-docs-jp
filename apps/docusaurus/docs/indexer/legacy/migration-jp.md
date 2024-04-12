---
title: "トランザクション ストリーム サービスへの移行(Japanese)"
slug: "indexer-legacy-migration-jp"
---
# トランザクションストリームサービスへの移行

現在レガシーインデクサーを実行を実行している場合、トランザクションストリームサービスの使用へ移行する方法をガイドします。

古いインデクサースタックでは、トランザクションを処理するために追加のスレッドを備えたアーカイブフルノードを実行する必要がありますが、これは維持が難しくコストがかかります。さらにカスタム ロジックを追加するには、より大きなマシンが必要になるか、線形に拡張する複数のフルノードを実行する必要があります。

この新しいインデックス作成方法では[トランザクションストリームサービス](https://aptos.dev/indexer/txn-stream/)を使用します。[ラボがホストするトランザクションストリームサービス](https://aptos.dev/indexer/txn-stream/labs-hosted/)を使用出来、[トランザクションストリームサービスの独自のインスタンスを実行 ](https://aptos.dev/indexer/txn-stream/self-hosted)することもできます。

## 1. リポジトリを複製する

```
# SSH
git clone git@github.com:aptos-labs/aptos-indexer-processors.git

# HTTPS
git clone https://github.com/aptos-labs/aptos-indexer-processors.git
```

サービスのディレクトリに移動します。:

```
cd aptos-indexer-processors
cd rust/processor
```

## 2. プロセッサをトランザクションストリームサービスへ移行する

移行するプロセッサごとに、以下のテンプレートを使用して構成ファイルを作成する必要があります。構成ファイルの各フィールドの情報については[こちら](https://aptos.dev/indexer/api/self-hosted/#configuration).
を御覧下さい。

```yaml
health_check_port: 8084
server_config:
  processor_config:
    type: default_processor
  postgres_connection_string: <postgres_uri, e.g. postgresql://postgres:@localhost:5432/indexer>
  indexer_grpc_data_service_address: <url_from_api_gateway>
  indexer_grpc_http2_ping_interval_in_secs: 60
  indexer_grpc_http2_ping_timeout_in_secs: 10
  auth_token: <auto_token_from_api_gateway>
  starting_version: 0 # optional
  ending_version: 0 # optional
```

プロセッサをトランザクションストリームサービスに接続するには、`indexer_grpc_data_service_address`の URLを設定する必要があります。次のオプションのいずれかを選択します。

### オプションA:ラボがホストするトランザクションストリームサービスへ接続する

ラボがホストするトランザクションストリームサービスを使用する主な利点は、将来トランザクションのストリームを取得するためにアーカイブフルノードを実行する必要が無くなる事です。このサービスはレート制限されています。ラボがホストするトランザクションストリームサービスに接続する手順は[こちら](https://aptos.dev/indexer/txn-stream/labs-hosted)で見つかります。

###  オプションB: セルフホスト型トランザクションストリームサービスを実行する

必要なら、トランザクションストリームサービスの自己ホスト型インスタンスを実行し、それにプロセッサを接続できます。セルフホスト型トランザクションストリームを実行する手順は[こちら](https://aptos.dev/indexer/txn-stream/self-hosted)で見つかります。

## 3. (オプション)カスタムプロセッサをトランザクションストリームサービスへ移行する

古いインデクサーで作成されたカスタムプロセッサがある場合は、新しいデータベースを最初から作成することを強くお勧めします。
新しいデータベースを使用すると、この移行中にすべてのカスタム データベース移行が確実に適用されます。

### a. カスタムテーブルスキーマを移行する

カスタムスキーマを移行するには、各カスタム移行を[`migrations`](https://github.com/aptos-labs/aptos-indexer-processors/tree/main/rust/processor/migrations)フォルダーにコピーします。

### b. カスタムプロセッサーコードを移行する

カスタム プロセッサを[`processors`](https://github.com/aptos-labs/aptos-indexer-processors/tree/main/rust/processor)フォルダーにコピーし、関連するカスタムモデルを[`models`](https://github.com/aptos-labs/aptos-indexer-processors/tree/main/rust/processor/src/models)フォルダーにコピーして、コードを移行します。カスタムプロセッサを以下のRustコードファイルに追加して、残りのコードと統合します。

[`mod.rs`](https://github.com/aptos-labs/aptos-indexer-processors/blob/main/rust/processor/src/processors/mod.rs)

```
pub enum Processor {
    ...
    CoinProcessor,
    ...
}

impl Processor {
    ...
    COIN_PROCESSOR_NAME => Self::CoinProcessor,
    ...
}
```

[`worker.rs`](https://github.com/aptos-labs/aptos-indexer-processors/blob/main/rust/processor/src/worker.rs)

```
Processor::CoinProcessor => {
    Arc::new(CoinTransactionProcessor::new(self.db_pool.clone()))
},
```

## 4. DieselでPostgresデータベーを埋め戻す

新しいプロセッサは古いプロセッサと同じPostgresスキーマを持っていますが、完全なバックフィルを行うことをお勧めします。(理想的には新しいDBに全部書き込む)プロトコルバッファーの変換の結果として一部のフィールドが少し異なるため。 

これらの手順は[Diesel移行](https://docs.rs/diesel_migrations/latest/diesel_migrations/)の使用に精通していることを前提としています。以下のコマンドを使用し、データベース全体の移行を実行します。

```
DATABASE_URL=postgres://postgres@localhost:5432/postgres diesel migration run
```

## 5. 移行されたプロセッサを実行する

単一プロセッサを実行するには、以下のコマンドを使用します。

```
cargo run --release -- -c config.yaml
```

複数のプロセッサがある場合は、プロセッサごとにサービスの個別のインスタンスを実行する必要があります。

プロセッサをDockerイメージとして実行する場合、手順は[ここ](https://aptos.dev/indexer/api/self-hosted#run-with-docker)にリストされています。


## よくある質問

### 1. プロトコルバッファーは更新される予定ですか? 更新される場合、何をする必要がありますか？

プロトコルバッファースキーマは将来更新される可能性があります。下位互換性のない変更については、リリースノートでお知らせします。

### 2. 古いインデクサーでカスタムロジックがすでに記述されている場合はどうすればよいですか?それらを移行するのは簡単ですか?

新しいインデクサースタックには古いインデクサースタックと同じPostgresスキーマがあるため、プロセッサの移行は簡単です。カスタムDB移行が適用される様に、この移行用の新しいDBを作成することを強くお勧めします。

このガイドのステップ 3 に従って、カスタムロジックを新しいプロセッサスタックに移行します。
