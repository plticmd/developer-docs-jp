---
title: "インデクサーフルノードを実行する(Japanese)"
slug: "indexer-legacy-indexer-fullnode-jp.md"
---
# インデクサーフルノードを実行する
:::warning 従来のインデクサー
これは従来のインデクサーに関するドキュメントです。最新のインデクサースタックの基盤となるインフラストラクチャを実行する方法については [トランザクションストリームサービス](/indexer/txn-stream)を御覧下さい。
:::

## Aptosインデクサーを実行する

:::danger APPLEシリコン搭載のMacOSのみ
以下のインストール手順は、Appleシリコン搭載のMacOSでのみ検証されています。それ以外で実行する場合は、若干の調整が必要な場合があります。
:::

## 概要

インデクサーのフルノードを実行する手順は、以下の要約を御覧下さい。

1. 以下で解説する必要なツールとパッケージが全て揃っていることを確認してください。
2. 指示に従い[パブリックフルノードを設定](/nodes/full-node/verify-pfn.md)しますが、まだフルノードを起動しないで下さい。
3. 以下で解説する様に`fullnode.yaml`を編集します。
4. 以下の手順に従い、インデクサーのフルノードを実行します。


## 前提条件

以下のパッケージをインストールします。注意：[開発環境の準備](/guides/building-from-source)の際、これらの多くがすでにインストールされている可能性があります。`which command-name`を実行し、出力にパッケージが表示されることで確認できます(ただし、`libpqx`はインストールされても返されません)。

> 重要: MacOSを使用している場合は、`brew`ではなく、 [公式のガイダンスに従ってDockerをインストール](https://docs.docker.com/desktop/install/mac-install/)する必要があります。

Aptosインデクサーのフルノードの場合は、以下のパッケージをインストールします。

- [`brew`](https://brew.sh/) - `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` アウトプットに出力されたコマンドを実行して、コマンドをパスに追加し、依存関係をインストールします。
- [`cargo`Rustパッケージマネージャー](https://www.rust-lang.org/tools/install) - `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
- [`docker`](https://docs.docker.com/get-docker/) - `brew install docker`
- [pg_ctllibpqコマンドを含むlibpq Postgres C APIライブラリ](https://formulae.brew.sh/formula/libpq) - `brew install libpq`インストール後、全てのエクスポートコマンドを実行してください。
- [`postgres` PostgreSQLサーバー](https://www.postgresql.org/) - `brew install postgresql`
- [`diesel`](https://diesel.rs/) - `brew install diesel`

## データベースを設定する

1. PostgreSQL サーバーを起動します。:
   `brew services start postgresql`
2. `psql postgres`が実行できることを確認し:`\q`を入力してプロンプトを終了します。
3. `createuser`コマンドでPostgreSQLユーザー `postgres`を作成します。(コマンドは`which`(以下)で見つかります)。:
   ```bash
   /path/to/createuser -s postgres
   ```
4. まだクローンしていない場合は`aptos-core`リポジトリのクローンを作成します。
   ```bash
   git clone https://github.com/aptos-labs/aptos-core.git
   ```
5. `aptos-core/crates/indexer`ディレクトリに移動 (`cd`)します。
6. データベーススキーマを作成します。
   ```bash
   diesel migration run --database-url postgresql://localhost/postgres
   ```
   これにより、この`aptos-core/crates/indexer`ディレクトリ内に`migrations`サブディレクトリを含むデータベーススキーマが作成されます。何らかの理由でこのデータベースがすでに使用されている場合は、別のデータベースを試してください。例えば：`DATABASE_URL=postgres://postgres@localhost:5432/indexer_v2 diesel database reset`

## フルノードインデクサーを開始する

1. 指示に従い[パブリックフルノード](/nodes/full-node/verify-pfn.md)を設定し、設定の準備をしますが、インデクサーはまだ**開始しないで下さい**(`cargo run`か`docker run`コマンドで)。
2. 以下を使用して、最新のインデクサーDockerイメージをプルします。:
   ```bash
   docker pull aptoslabs/validator:nightly_indexer
   ```
3. `./fullnode.yaml`を編集して、以下の構成を追加します。:
   ```yaml
   storage:
     enable_indexer: true
     # This is to avoid the node being pruned
     storage_pruner_config:
       ledger_pruner_config:
         enable: false

   indexer:
     enabled: true
     postgres_uri: "postgres://postgres@localhost:5432/postgres"
     processor: "default_processor"
     check_chain_id: true
     emit_every: 500
   ```

:::tip フルノードをブートストラップする
インデクサーのフルノードをGenesisから同期する(これには長い時間がかかる場合があります)かわりに、フルノードを開始する前にバックアップデータを使用してフルノードをブートストラップすることを選択できます。これを行うには[バックアップから復元する](/nodes/full-node/aptos-db-restore.md)手順に従ってください。

注: インデクサーは[スナップショット](/nodes/full-node/bootstrap-fullnode.md)または[高速同期](../../guides/state-sync.md#fast-syncing)を使用してブートストラップすることはできません。
:::

1. `cargo run`または`docker run`のいずれかを使用してインデクサーのフルノードを実行します。あなたの特有のノードで必要な引数を必ず指定して下さい。:

   ```bash
   docker run -p 8080:8080 \
     -p 9101:9101 -p 6180:6180 \
     -v $(pwd):/opt/aptos/etc -v $(pwd)/data:/opt/aptos/data \
     --workdir /opt/aptos/etc \
     --name=aptos-fullnode aptoslabs/validator:nightly_indexer aptos-node \
     -f /opt/aptos/etc/fullnode.yaml
   ```
   または、
   ```bash
   cargo run -p aptos-node --features "indexer" --release -- -f ./fullnode.yaml
   ```

## インデクサーを再起動する

PostgreSQLサーバーを再起動するには、

1. `postmaster`プロセスを検索して強制終了することで[サーバーをシャットダウンします](https://www.postgresql.org/docs/8.1/postmaster-shutdown.html):
   ```bash
   ps -ef | grep -i postmaster
   ```
2. プロセスのプロセスID(PID)をコピーし、以下のコマンドに渡してシャットダウンします。:
   ```bash
   kill -INT PID
   ```
1. 以下のコマンドを使用してPostgreSQLサーバーを再起動します。:

   ```bash
   brew services restart postgresql@14
   ```
