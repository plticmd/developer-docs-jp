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

:::tip カスタム インデクサーを使用する場合
Currently Aptos-provided indexing service (see above) supports the following core Move modules:
現在、Aptos が提供するインデックス サービス (上記を参照) は、次のコア Move モジュールをサポートしています。:

- `0x1::coin`.
- `0x3::token`.
- `0x3::token_transfers`.

If you need an indexed database for any other Move modules and contracts, then you should develop your custom indexer.
他の Move モジュールおよびコントラクト用にインデックス付きデータベースが必要な場合は、カスタム インデクサーを開発する必要があります。
:::

Creating a custom indexer involves the following steps. Refer to the indexing block diagram at the start of this document.
カスタム インデクサーの作成には、次の手順が含まれます。このドキュメントの冒頭にあるインデックス作成のブロック図を参照してください。

1. Define new table schemas, using an ORM like [Diesel](https://diesel.rs/). In this document Diesel is used to describe the custom indexing steps ("Business logic" and the data queries in the diagram).
Dieselのような ORM を使用して、新しいテーブル スキーマを定義します。このドキュメントでは、Diesel を使用してカスタム インデックス作成ステップ (図内の「ビジネス ロジック」およびデータ クエリ) を説明します。

2. Create new data models based on the new tables ("Business logic" in the diagram).
新しいテーブル (図の「ビジネス ロジック」) に基づいて新しいデータ モデルを作成します。

3. Create a new transaction processor, or optionally add to an existing processor. In the diagram this step corresponds to processing the ledger database according to the new business logic and writing to the indexed database.
新しいトランザクション プロセッサを作成するか、必要に応じて既存のプロセッサに追加します。この図では、このステップは、新しいビジネス ロジックに従って台帳データベースを処理し、インデックス付きデータベースに書き込むことに対応します。

4. Integrate the new processor. Optional if you are reusing an existing processor.
新しいプロセッサを統合します。既存のプロセッサを再利用する場合はオプションです。

In the below detailed description, an example of indexing and querying for the coin balances is used. You can see this in the [`coin_processor`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/processors/coin_processor.rs).
以下の詳細な説明では、コイン残高のインデックス作成とクエリの例が使用されます。これは、 で確認できますcoin_processor。

### 1. 新しいテーブルスキーマを定義する

In this example we use [PostgreSQL](https://www.postgresql.org/) and [Diesel](https://diesel.rs/) as the ORM. To make sure that we make backward-compatible changes without having to reset the database at every upgrade, we use [Diesel migrations](https://docs.rs/diesel_migrations/latest/diesel_migrations/) to manage the schema. This is why it is very important to start with generating a new Diesel migration before doing anything else.
この例では、PostgreSQLとDiesel をORM として使用します。アップグレードのたびにデータベースをリセットすることなく、下位互換性のある変更を確実に行うために、Diesel 移行を使用してスキーマを管理します。このため、他の作業を行う前に、まず新しいディーゼル移行を生成することが非常に重要です。

Make sure you clone the Aptos-core repo by running `git clone https://github.com/aptos-labs/aptos-core.git` and then `cd` into `aptos-core/tree/main/crates/indexer` directory. Then proceed as below.
git clone https://github.com/aptos-labs/aptos-core.git実行してAptos-core リポジトリをディレクトリcdに複製してくださいaptos-core/tree/main/crates/indexer。その後、以下のように進みます。

a. The first step is to create a new Diesel migration. This will generate a new folder under [migrations](https://github.com/aptos-labs/aptos-core/tree/main/crates/indexer/migrations) with `up.sql` and `down.sql`
最初のステップは、新しい Diesel 移行を作成することです。これにより、移行の下に新しいフォルダーが生成されます。up.sqldown.sql

```bash
DATABASE_URL=postgres://postgres@localhost:5432/postgres diesel migration generate add_coin_tables
```

b. Create the necessary table schemas. This is just PostgreSQL code. In the code shown below, the `up.sql` will have the new changes and `down.sql` will revert those changes.
必要なテーブル スキーマを作成します。これは単なる PostgreSQL コードです。以下に示すコードでは、 にup.sql新しい変更が加えられ、down.sqlそれらの変更が元に戻されます。

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

See the [full source for `up.sql` and `down.sql`](https://github.com/aptos-labs/aptos-core/tree/main/crates/indexer/migrations/2022-10-04-073529_add_coin_tables).
との完全なソースup.sqldown.sqlを参照してください。

c. Run the migration. We suggest running it multiple times with `redo` to ensure that both `up.sql` and `down.sql` are implemented correctly. This will also modify the [`schema.rs`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/schema.rs) file.
移行を実行します。とのredo両方が正しく実装されていることを確認するために、 で複数回実行することをお勧めします。これによりファイルも変更されます。up.sqldown.sqlschema.rs

```bash
DATABASE_URL=postgres://postgres@localhost:5432/postgres diesel migration run
DATABASE_URL=postgres://postgres@localhost:5432/postgres diesel migration redo
```

### 2. 新しいデータスキーマを作成する

We now have to prepare the Rust data models that correspond to the Diesel schemas. In the case of coin balances, we will define `CoinBalance` and `CurrentCoinBalance` as below:
次に、Diesel スキーマに対応する Rust データ モデルを準備する必要があります。コイン残高の場合、CoinBalanceとCurrentCoinBalanceを以下のように定義します。:

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

We will also need to specify the parsing logic, where the input is a portion of the transaction. In the case of coin balances, we can find all the details in `WriteSetChanges`, specifically where the write set change type is `write_resources`.
また、入力がトランザクションの一部である解析ロジックを指定する必要もあります。コイン残高の場合、WriteSetChanges特に書き込みセット変更タイプが であるすべての詳細を で見つけることができますwrite_resources。

**Where to find the relevant data for parsing**: This requires a combination of understanding the Move module and the structure of the transaction. In the example of coin balance, the contract lives in [coin.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/coin.move), specifically the coin struct (search for `struct Coin`) that has a `value` field. We then look at an [example transaction](https://api.testnet.aptoslabs.com/v1/transactions/by_version/259518) where we find this exact structure in `write_resources`:
解析に関連するデータを見つける場所: これには、Move モジュールとトランザクションの構造を理解することを組み合わせる必要があります。コイン残高の例では、コントラクトはCoin.move、具体的にはフィールドを持つ Coin 構造体 ( を検索struct Coin)に存在しますvalue。次に、この正確な構造が次の中にあるトランザクションの例write_resourcesを見てみましょう。:

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

See the full code in [coin_balances.rs](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/models/coin_models/coin_balances.rs).
完全なコードは、Coin_balances.rsを参照してください。

### 3. 新しいプロセッサーを作成する

Now that we have the data model and the parsing function, we need to call that parsing function and save the resulting model in our Postgres database. We do this by creating (or modifying) a `processor`. We have abstracted a lot already from that class, so the only function that should be implemented is `process_transactions` (there are a few more functions that should be copied, those should be obvious from the example).
データ モデルと解析関数が完成したので、その解析関数を呼び出して、結果のモデルを Postgres データベースに保存する必要があります。これを行うには、 を作成 (または変更) しますprocessor。そのクラスからすでに多くのことを抽象化しているので、実装する必要がある唯一の関数は次のとおりですprocess_transactions(コピーする必要のある関数がさらにいくつかあります。それらは例から明らかです)。

The `process_transactions` function takes in a vector of transactions with a start and end version that are used for tracking purposes. The general flow should be:
このprocess_transactions関数は、追跡目的で使用される開始バージョンと終了バージョンを持つトランザクションのベクトルを受け取ります。一般的なフローは次のようになります。:

- Loop through transactions in the vector.
ベクトル内のトランザクションをループします。
- Aggregate relevant models. Sometimes deduping is required, e.g. in the case of `CurrentCoinBalance`.
関連するモデルを集約します。場合によっては、重複排除が必要になることがあります (例: の場合) CurrentCoinBalance。
- Insert the models into the database in a single Diesel transaction. This is important, to ensure that we do not have partial writes.
単一の Diesel トランザクションでモデルをデータベースに挿入します。これは、部分的な書き込みが発生しないようにするために重要です。
- Return status (error or success).
ステータス (エラーまたは成功) を返します。

:::tip コイントランザクションプロセッサー
See [coin_process.rs](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/processors/coin_processor.rs) for a relatively straightforward example. You can search for `coin_balances` in the page for the specific code snippet related to coin balances.
比較的単純な例については、coin_process.rsを参照してください。coin_balancesページ内でコイン残高に関連する特定のコード スニペットを検索できます。
:::

**How to decide whether to create a new processor:** This is completely up to you. The benefit of creating a new processor is that you are starting from scratch, so you will have full control over exactly what gets written to the indexed database. The downside is that you will have to maintain a new fullnode, since there is a 1-to-1 mapping between a fullnode and the processor.
新しいプロセッサを作成するかどうかを決定する方法:これは完全にあなた次第です。新しいプロセッサを作成する利点は、最初から開始できるため、インデックス付きデータベースに書き込まれる内容を完全に制御できることです。欠点は、フルノードとプロセッサの間に 1 対 1 のマッピングがあるため、新しいフルノードを維持する必要があることです。

### 4. 新しいプロセッサーを統合する

This is the easiest step and involves just a few additions.
これは最も簡単な手順であり、いくつかの追加を行うだけです。

1. To start with, make sure to add the new processor in the Rust code files: [`mod.rs`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/processors/mod.rs) and [`runtime.rs`](https://github.com/aptos-labs/aptos-core/blob/main/crates/indexer/src/runtime.rs). See below:
まず、新しいプロセッサを Rust コード ファイルに追加してください:mod.rsおよびruntime.rs。以下を参照してください：:

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

2. Create a `fullnode.yaml` with the correct configuration and test the custom indexer by starting a fullnode with this `fullnode.yaml`.
正しい構成でを作成しfullnode.yaml、 this でフルノードを開始してカスタム インデクサーをテストしますfullnode.yaml。

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

Test by starting an Aptos fullnode by running the below command. You will see many logs in the terminal output, so use the `grep` filter to see only indexer log output, as shown below:
以下のコマンドを実行して Aptos フルノードを起動してテストします。ターミナル出力には多くのログが表示されるため、grep以下に示すように、フィルターを使用してインデクサー ログ出力のみを表示します。:

```bash
cargo run -p aptos-node --features "indexer" --release -- -f ./fullnode_coin.yaml | grep -E "_processor"
```

See the full instructions on how to start an indexer-enabled fullnode in [Indexer Fullnode](./indexer-fullnode).
インデクサーが有効なフルノードを開始する方法の完全な手順については、「 インデクサー フルノード 」を参照してください。
