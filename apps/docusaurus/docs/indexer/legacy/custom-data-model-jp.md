---
title: "カスタムデータモデル(in Japanese)"
slug: "indexer-legacy-custom-data-model-jp"
---
# カスタムデータモデル
:::warning 従来のインデクサー
これは従来のインデクサーに関するドキュメントです。最新のインデクサースタックを使用してカスタムプロセッサを作成する方法については[カスタムプロセッサー](/indexer/custom-processors)を御覧下さい。
:::

## 独自のデータモデルを定義する

Aptos台帳データ用のカスタムインデクサーを開発する場合は、この方法を使用します。

:::tip カスタムインデクサーを使用する場合
現在、Aptos提供のインデックスサービス(上記参照)は、以下のコアMoveモジュールをサポートしています。:

- `0x1::coin`.
- `0x3::token`.
- `0x3::token_transfers`.

(他のMoveモジュールおよびコントラクト用の)インデックス付きデータベースが必要な場合は、カスタムインデクサーを開発する必要があります。
:::

カスタムインデクサーの作成は以下の手順を使います。インデックス作成のブロック図を参照してください、このドキュメントの冒頭にあります。

1. [Diesel](https://diesel.rs/)のようなORMを使用して、新しいテーブルスキーマを定義します。このドキュメントでは、Dieselを使用してカスタムインデックス作成の手順(図内の「ビジネスロジック」およびデータクエリ)を解説します。
2. 新しいテーブル(図内「ビジネスロジック」)に基づいて新しいデータモデルを作成します。
3. 新しいトランザクションプロセッサを作成するか、任意で既存のプロセッサに追加します。この図では、このステップは、新しいビジネスロジックに従って台帳データベースを処理し、インデックス付きデータベースに書き込むことに該当します。
4. 新しいプロセッサを統合します。既存のプロセッサを再利用する場合はこのオプションが選択出来ます。

以下の詳細な説明では、インデックス作成例とコイン残高のクエリが使用されます。これは[`coin_processor`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/processors/coin_processor.rs)で確認できます。

### 1. 新しいテーブルスキーマを定義する

この例では[PostgreSQL](https://www.postgresql.org/) と[Diesel](https://diesel.rs/)をORMとして使用します。アップグレードのたびにデータベースをリセットすることなく下位互換性のある変更を確実に行うため、[Diesel移行](https://docs.rs/diesel_migrations/latest/diesel_migrations/)を使用してスキーマを管理します。そのため、他の作業を行う前にまず新しいディーゼル移行を生成することが非常に重要なのです。

`git clone https://github.com/aptos-labs/aptos-core.git`
を実行してAptos-coreリポジトリを複製し、`aptos-core/tree/main/crates/indexer`ディレクトリに`cd`します。その後、以下の様に進めます。

a. 最初のステップでは、新しいDiesel移行(migration)を作成します。[migrations](https://github.com/aptos-labs/aptos-core/tree/main/crates/indexer/migrations)の下に新しいフォルダー(`up.sql`と`down.sql`を含む)が生成されます。

```bash
DATABASE_URL=postgres://postgres@localhost:5432/postgres diesel migration generate add_coin_tables
```
b. 必要なテーブルスキーマを作成します。これは単なるPostgreSQLコードです。以下に示すコードでは、`up.sql` は新しい変更が加えられ、`down.sql` はそれらの変更が元に戻されます。

```sql
-- up.sql
-- coin balances for each version
CREATE TABLE coin_balances (
  transaction_version BIGINT NOT NULL,
  owner_address VARCHAR(66) NOT NULL,
  -- Hash of the non-truncated coin type
  coin_type_hash VARCHAR(64) NOT NULL,
  -- creator_address::name::symbol<struct>
  coin_type VARCHAR(5000) NOT NULL,
  amount NUMERIC NOT NULL,
  transaction_timestamp TIMESTAMP NOT NULL,
  inserted_at TIMESTAMP NOT NULL DEFAULT NOW(),
  -- Constraints
  PRIMARY KEY (
    transaction_version,
    owner_address,
    coin_type_hash
  )
);
-- latest coin balances
CREATE TABLE current_coin_balances {...}
-- down.sql
DROP TABLE IF EXISTS coin_balances;
DROP TABLE IF EXISTS current_coin_balances;
```

[`up.sql`と`down.sql`の完全なソース](https://github.com/aptos-labs/aptos-core/tree/main/crates/indexer/migrations/2022-10-04-073529_add_coin_tables)を御覧下さい。

c. 移行を実行します。`up.sql`と`down.sql`の両方が正しく実装されていることを確認するため、`redo`で複数回実行することをお勧めします。これは[`schema.rs`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/schema.rs)ファイルも変更します。

```bash
DATABASE_URL=postgres://postgres@localhost:5432/postgres diesel migration run
DATABASE_URL=postgres://postgres@localhost:5432/postgres diesel migration redo
```

### 2. 新しいデータスキーマを作成する

Dieselスキーマに対応するRustデータモデルを準備します。コイン残高の場合、`CoinBalance`と`CurrentCoinBalance`を以下の様に定義します。:

```rust
#[derive(Debug, Deserialize, FieldCount, Identifiable, Insertable, Serialize)]
#[diesel(primary_key(transaction_version, owner_address, coin_type))]
#[diesel(table_name = coin_balances)]
pub struct CoinBalance {
    pub transaction_version: i64,
    pub owner_address: String,
    pub coin_type_hash: String,
    pub coin_type: String,
    pub amount: BigDecimal,
    pub transaction_timestamp: chrono::NaiveDateTime,
}

#[derive(Debug, Deserialize, FieldCount, Identifiable, Insertable, Serialize)]
#[diesel(primary_key(owner_address, coin_type))]
#[diesel(table_name = current_coin_balances)]
pub struct CurrentCoinBalance {
    pub owner_address: String,
    pub coin_type_hash: String,
    pub coin_type: String,
    pub amount: BigDecimal,
    pub last_transaction_version: i64,
    pub last_transaction_timestamp: chrono::NaiveDateTime,
}
```

入力がトランザクションの一部である解析ロジックを指定する必要もあります。コイン残高の場合、すべての内容を`WriteSetChanges`(特に書き込みセット変更タイプ`write_resources`)で見つけることができます。

**解析関連のデータを見つける場所**:
 Moveモジュールとトランザクション構造の理解を組み合わせる必要があります。コイン残高の例では、コントラクトは
t[coin.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/coin.move)具体的には`value`フィールドを持つコイン構造体(`struct Coin`を検索)に存在します。
次は`write_resources`内の正確な構造[トランザクション例](https://api.testnet.aptoslabs.com/v1/transactions/by_version/259518)を見てみましょう。: 

```json
"changes": [
  {
    ...
    "data": {
      "type": "0x1::coin::CoinStore<0x1::aptos_coin::AptosCoin>",
      "data": {
        "coin": {
          "value": "49742"
      },
      ...
```

完全なコードは[coin_balances.rs](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/models/coin_models/coin_balances.rs)を御覧下さい。

### 3. 新しいプロセッサーを作成する

データモデルと解析関数が完成したので、その解析関数を呼び出し、結果のモデルをPostgresデータベースに保存する必要があります。これを行うには、`processor`を作成(または変更)します。そのクラスからすでに多くのことを抽象化しているので、実装する必要がある唯一の関数は`process_transactions`です。
(コピーする必要のある関数がいくつかあります)。

`process_transactions`関数は、追跡目的で使用される開始と終了バージョンを持つトランザクションのベクターを受け取ります。一般的なフローは以下の通り。:

- ベクター内のトランザクションをループします。
- 関連するモデルを集約します。場合によっては重複排除が必要です。例:`CurrentCoinBalance`の場合。
- 単一のDieselトランザクションでモデルをデータベースに挿入します。これは不完全な書き込みを発生させないため重要です。
- 状態(エラーまたは成功)を返します。

:::tip コイントランザクションプロセッサー
比較的単純な例については[coin_process.rs](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/processors/coin_processor.rs)を御覧下さい。コイン残高に関連する特定のコードスニペットページ内で
`coin_balances`を検索できます。
:::

**新しいプロセッサーを作成するかどうかを決める方法**:
これは完全にあなた次第です。新しいプロセッサを作成する利点は、最初から開始できるため、インデックス付きデータベースに何を書き込むかを完全に制御できる事です。欠点は、フルノードとプロセッサの間に1対1のマッピングがあるため、新しいフルノードを維持する必要がある事です。

### 4. 新しいプロセッサーを統合する

これは最も簡単な手順で、ほんの少し追加するだけです。

1. まず、新しいプロセッサをRustコードファイル:[`mod.rs`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/processors/mod.rs)と[`runtime.rs`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/runtime.rs)に追加してください。以下を御覧下さい。:

[**mod.rs**](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/processors/mod.rs)

```rust
pub enum Processor {
  CoinProcessor,
  ...
}
...
  COIN_PROCESSOR_NAME => Self::CoinProcessor,
```

[**runtime.rs**](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/runtime.rs)

```rust
Processor::CoinProcessor => Arc::new(CoinTransactionProcessor::new(conn_pool.clone())),
```

2. 正しい構成で`fullnode.yaml`を作成し、`fullnode.yaml`でフルノードを開始してカスタムインデクサーをテストします。

**fullnode.yaml**

```yaml
storage:
  enable_indexer: true
  storage_pruner_config:
    ledger_pruner_config:
      enable: false

indexer:
  enabled: true
  check_chain_id: true
  emit_every: 1000
  postgres_uri: "postgres://postgres@localhost:5432/postgres"
  processor: "coin_processor"
  fetch_tasks: 10
  processor_tasks: 10
```

以下のコマンドを実行し、Aptosフルノードを起動してテストします。ターミナル出力には多くのログが表示されるため、`grep`フィルターを使用してインデクサーログ出力のみを表示します。以下を参照:

```bash
cargo run -p aptos-node --features "indexer" --release -- -f ./fullnode_coin.yaml | grep -E "_processor"
```
インデクサーが有効なフルノードの開始方法(フルインストラクション)は[Indexer Fullnode](./indexer-fullnode)を御覧下さい。
