---
title: "End-to-Endチュートリアル(Japaneses)"
slug: "indexer-custom-processors-e2e-tutorial-jp"
---

# カスタムインデクサープロセッサの作成

このチュートリアルでは、基本的なカスタムインデクサープロセッサーを作成しAptosブロックチェーン上のイベントとデータを追跡する手順を解説します。

非常に単純なスマートコントラクトを使用します。**Coin Flip**と呼ばれていて、すでにイベントを発行しています。

スマートコントラクトはすでにデプロイされており、改造に興味がない限り殆どの場合、それを理解する必要はありません。

## 始める

まず、[aptos-indexer-processors](https://github.com/aptos-labs/aptos-indexer-processors)リポジトリのクローンを作成します。

```
# HTTPS
https://github.com/aptos-labs/aptos-indexer-processors.git

# SSH
git@github.com:aptos-labs/aptos-indexer-processors.git
```

コインフリップディレクトリに移動します。

```
cd aptos-indexer-processors
cd python/processors/coin_flip
```

プロセッサは、トランザクションストリームサービスからのトランザクションを消費します。Labs-Hostedトランザクションストリームサービスを使用するには、認証トークンが必要です[このガイド](../txn-stream/labs-hosted.md#authorization-via-api-key)に従い、開発者ポータルからトークンを取得します。このチュートリアルは`Testnet`用であるため、`Testnet`のAPIキーを作成します。完了すると、以下のようなトークンが作成されるはずです。

```
aptoslabs_yj4bocpaKy_Q6RBP4cdBmjA8T51hto1GcVX5ZS9S65dx
```

次のツールも必要です。

- [Aptos CLI](/tools/aptos-cli/install-cli)
- Python 3.11+: [インストールガイド](https://docs.python-guide.org/starting/installation/#python-3-installation-guides).
- Poetry: [インストールガイド](https://python-poetry.org/docs/#installation).

このチュートリアルはデータベースとしてPostgreSQLを使用します。何を使用しても自由ですが、簡単にするためPostgreSQLを使用します。以下のデータベース構成とツールを使用します。

- [PostgreSQL](https://www.postgresql.org/download/)
  - `5432`ポートの`localhost`でホストされているデータベースを使用します。これがデフォルトである必要があります。
  - ユーザー名を作成したら、そのユーザー名とそれに使用するパスワードを記録して下さい。
  - [ここ](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-22-04-quickstart)でPostgreSQLとpsqlツールをインストールするチュートリアルを参照し、より迅速にデータベースを設定出来ます。
  -  データベースのデータを簡単に表示するには、[DBeaver](https://dbeaver.io/)_推奨_[pgAdmin](https://www.pgadmin.org/)または[Postico](https://eggerapps.at/postico2/)等のGUIの使用を検討してください。

データベースの作成方法の説明は、このチュートリアルの範囲を超えています。データベースの作成方法が分からない場合は、`psql`ツールでデータベースを作成する方法のチュートリアルを御確認下さい。

## 環境設定

### PostgreSQLデータベースを設定する

必ず`postgresql`サービスを開始して下さい。

Linux/WSL のコマンドは以下の様になります。

```shell
sudo service postgresql start
```

Macでbrewを使用している場合は、以下の様に起動します。

```shell
brew services start postgresql
```

`coin_flip`という名前でデータベースを作成します。ユーザー名は`user`、パスワードは`password`です。`DBeaver`を使用している場合は、`Databases`タブを右クリックして`Create New Database`を選択することで、新しいデータベースを作成できます。

データベースが正しく設定されて`psql`ツールがある場合は、`psql coin_flip`コマンドを実行できるはずです。


### PoetryとgRPCでローカル環境を設定する

まだ[カスタムプロセッサーガイド](https://github.com/aptos-labs/aptos-indexer-processors)入門を読んでいない場合は、必ず読んでください。

[ここ](https://github.com/aptos-labs/aptos-indexer-processors/tree/main/python)で、インデクサープロセッサの作成方法に関するPython固有の広範な概要を確認できます。

## インデクサープロセッサーの構成

実際に使用するインデクサープロセッサーの構成の詳細を設定しましょう。

### config.yamlファイルの設定

以下の内容をコピーし、`config.yaml`という名前のファイルに保存します。そのファイルを`coin_flip`フォルダーに保存します。ファイルのディレクトリ構造は以下の様になります。

```
- aptos-indexer-processor
    - python
        - processors
            - aptos_ambassador_token
            - aptos-tontine
            - coin_flip
                - move
                    - sources
                        - coin_flip.move
                        - package_manager.move
                    - Move.toml
                - config.yaml     <-------- Create and edit this config.yaml file
                - models.py
                - processor.py
                - README.md
            - example_event_processor
            - nft_marketplace_v2
            - nft_orderbooks
            __init__.py
            main.py
            README.md
    - rust
    - scripts
    - typescript
```

config.yamlファイルを開いたら、以下の特定のフィールドを更新するだけです。
`processor_config.type` - プロセッサ`chain_id`の名前
`indexer_grpc_data_service_address` - インデクサー データ サービスのアドレス 
`auth_token` - 開発者ポータルで作成したAPIキー
`postgres_connection_string` - PostgreSQLデータベースへの接続文字列
`starting_version` - starting_version- 処理するトランザクションの開始バージョン
`ending_version` - 処理するトランザクションの終了バージョン

```yaml
server_config:
  processor_config:
    type: "coin_flip"
  chain_id: 2
  indexer_grpc_data_service_address: "grpc.testnet.aptoslabs.com:443"
  auth_token: "<API-KEY>"
  postgres_connection_string: "postgresql://localhost:5432/<Database Name>"
  # Optional. Start processor at starting_version
  starting_version: 636585029
  # Optional. Stop processor after ending_version.
  ending_version: 636589723
```

### config.yamlを更にカスタマイズ

`config.yaml`ファイルを使用して追加の構成をカスタマイズできます。

特定の台帳バージョンから開始するには、以下のコマンドで`config.yaml`ファイル内のバージョンを指定できます。

```yaml
starting_version: <Starting Version>
```

これは、インデクサーがイベントの検索を開始するトランザクションバージョンです。


`next_versions_to_process`テーブルは、インデックス作成の現在の状態を追跡します。主キーは`indexer_name`です。この`next_version`フィールドは、次に処理するトランザクションを示し、`updated_at`時刻はインデクサーが最後にトランザクションを処理した時刻を示します。

```yaml
ending_version: <Ending Version>
```

すべての Coin Flip トランザクションのリストを表示したい場合は、[Aptos Explorer](https://explorer.aptoslabs.com/?network=testnet)で検索できます。`0xe57752173bc7c57e9b61c84895a75e53cd7c0ef0855acd81d31cb39b0e87e1d0`アカウントを検索します。 - これはCoinFlipモジュールがデプロイされたアカウントです。結果には、トランザクションのリストとそのバージョン番号が表示されます。 バージョン番号を使用して、config.yamlファイル内で`starting_version`と`ending_version`を指定できます。

別のネットワークを使用する場合は、`indexer_grpc_data_service_address`フィールドを対応する目的の値に変更します。

```yaml
# Devnet
indexer_grpc_data_service_address: grpc.devnet.aptoslabs.com:443

# Testnet
indexer_grpc_data_service_address: grpc.testnet.aptoslabs.com:443

# Mainnet
indexer_grpc_data_service_address: grpc.mainnet.aptoslabs.com:443
```

これらのIPアドレスが機能しない場合は、期限切れである可能性があります。最新のエンドポイントについては、リポジトリのルートフォルダーの`README.md`を確認してください。

別のデータベース名またはプロセッサ名を使用している場合は、固有の要求に応じて`postgres_connection_string`を変更してください。フィールドの一般的な構造は以下の通りです。

```yaml
postgres_connection_string: "postgresql://username:password@database_url:port_number/database_name"
```

### プロセッサーとスキーマ名を構成ファイルへ追加する

まず、使用するデータベーススキーマの名前を作成しましょう。この例では`coin_flip`を使用しているため、以下の2つの場所に追加する必要があります。

1. これを`python/utils/processor_name.py`ファイルに追加する必要があります。

```python
    class ProcessorName(Enum):
        EXAMPLE_EVENT_PROCESSOR = "python_example_event_processor"
        NFT_MARKETPLACE_V1_PROCESSOR = "nft_marketplace_v1_processor"
        NFT_MARKETPLACE_V2_PROCESSOR = "nft_marketplace_v2_processor"
        COIN_FLIP = "coin_flip"
```

2. これを`IndexerProcessorServer`のコンストラクター`utils/worker.py`内のmatch casesへ追加します。

```python
match self.config.processor_name:
    case ProcessorName.EXAMPLE_EVENT_PROCESSOR.value:
        self.processor = ExampleEventProcessor()
    case ProcessorName.NFT_MARKETPLACE_V1_PROCESSOR.value:
        self.processor = NFTMarketplaceProcesser()
    case ProcessorName.NFT_MARKETPLACE_V2_PROCESSOR.value:
        self.processor = NFTMarketplaceV2Processor()
    case ProcessorName.COIN_FLIP.value:
        self.processor = CoinFlipProcessor()
```

3. これを`python/utils/models/schema_names.py`ファイルに追加します

```python
EXAMPLE = "example"
NFT_MARKETPLACE_SCHEMA_NAME = "nft_marketplace"
NFT_MARKETPLACE_V2_SCHEMA_NAME = "nft_marketplace_v2"
COIN_FLIP_SCHEMA_NAME = "coin_flip"
```

### Moveコントラクトでのイベント発行の解説

Moveコントラクト(`coin_flip/move/sources/coin_flip.move`内)では、各ユーザーは自分のアカウントに関連付けられたオブジェクトを持っています。このオブジェクトには、`CoinFlipStats`リソースが含まれていて、ユーザーの勝敗の合計数を追跡し、イベントの発行を担当します。

```rust
// CoinFlipStats object/resource definition
#[resource_group_member(group = aptos_framework::object::ObjectGroup)]
struct CoinFlipStats has key {
    wins: u64,
    losses: u64,
    event_handle: EventHandle<CoinFlipEvent>,  //
    delete_ref: DeleteRef,
}

// event emission in `flip_coin`
fun flip_coin(
    user: &signer,
    prediction: bool,
    nonce: u64,
) acquires CoinFlipStats {
    // ...
    let (heads, correct_prediction) = flip(prediction, nonce);

    if (correct_prediction) {
        coin_flip_stats.wins = coin_flip_stats.wins + 1;
    } else {
        coin_flip_stats.losses = coin_flip_stats.losses + 1;
    };

    event::emit_event<CoinFlipEvent>(
        &mut coin_flip_stats.event_handle,
        CoinFlipEvent {
            prediction: prediction,
            result: heads,
            wins: coin_flip_stats.wins,
            losses: coin_flip_stats.losses,
        }
    );
}
```

発行されるイベントのタイプは`CoinFlipEvent`で、以下に示すとおりです。



```rust
struct CoinFlipEvent has copy, drop, store {
    prediction: bool,     // true = heads, false = tails
    result: bool,
    wins: u64,
    losses: u64,
}
```

### イベントデータがどの様に出力、処理されるのか見て理解する

`coin_flip`エントリ関数を呼び出すトランザクションを送信すると、インデクサーはイベントを解析し、トランザクション内で発生した各イベントのデータを記録します。

各`Event`タイプの`data`フィールド内に、出力された任意のイベントデータが表示されます。このデータは、データベースにイベントデータを保存するために使用されます。

プロセッサは、各トランザクションの各イベントをループして、すべてのイベントデータを処理します。トランザクションではさまざまなタイプのイベントが多数発生する可能性があるため、データベースに保存したくないさまざまなイベントに対処するフィルタリング関数を作成する必要があります。

これは、イベントリストの単純な反復構造です。


```python
for event_index, event in enumerate(user_transaction.events):
    # Skip events that don't match our filter criteria
    if not CoinFlipProcessor.included_event_type(event.type_str):
        continue
```

ここで、`included_event_type`関数は`CoinFlipProcessor`クラスの静的メソッドです。

```python
@staticmethod
def included_event_type(event_type: str) -> bool:
    parsed_tag = event_type.split("::")
    module_address = parsed_tag[0]
    module_name = parsed_tag[1]
    event_type = parsed_tag[2]
    # Now we can filter out events that are not of type CoinFlipEvent
    # We can filter by the module address, module name, and event type
    # If someone deploys a different version of our contract with the same event type, we may want to index it one day.
    # So we could only check the event type instead of the full string
    # For our sake, check the full string
    return (
        module_address
        == "0xe57752173bc7c57e9b61c84895a75e53cd7c0ef0855acd81d31cb39b0e87e1d0"
        and module_name == "coin_flip"
        and event_type == "CoinFlipEvent"
    )
```

プロセッサループ内で自分でイベントデータを確認したい場合は、`processor.py`ファイルに以下の様な物を追加できます。

```python
for event_index, event in enumerate(user_transaction.events):
    # Skip events that don't match our filter criteria
    if not CoinFlipProcessor.included_event_type(event.type_str):
        continue

    # ...

    # Load the data into a JSON object and then use/view it as a regular dictionary
    data = json.loads(event.data)
    print(json.dumps(data, indent=3))
```

この場合、単一のイベントがこれを出力します。

```
{
    'losses': '49',
    'prediction': False,
    'result': True,
    'wins': '51'
}
```

以下の様にデータを取得します。

```python
prediction = bool(data["prediction"])
result = bool(data["result"])
wins = int(data["wins"])
losses = int(data["losses"])

# We have extra data to insert into the database, because we want to process our data.
# Calculate the total
win_percentage = wins / (wins + losses)
```

これをイベントリストに追加します。

```python
# Create an instance of CoinFlipEvent
event_db_obj = CoinFlipEvent(
    sequence_number=sequence_number,
    creation_number=creation_number,
    account_address=account_address,
    transaction_version=transaction_version,
    transaction_timestamp=transaction_timestamp,
    prediction=prediction,
    result=result,
    wins=wins,
    losses=losses,
    win_percentage=win_percentage,
    inserted_at=datetime.now(),
    event_index=event_index,  # when multiple events of the same type are emitted in a single transaction, this is the index of the event in the transaction
)
event_db_objs.append(event_db_obj)
```

### データベースモデルの作成

<!-- CoinFlipEventをデータベースに保存する方法が分かったので、少し遡って、データベースが使用するこのモデルを _作成する_ 方法を明確にしましょう。 -->

`models.py`の`CoinFlipEvent`クラスを構造化し、Moveコントラクトの構造を反映する必要があります。

```python
class CoinFlipEvent(Base):
    __tablename__ = "coin_flip_events"
    __table_args__ = ({"schema": COIN_FLIP_SCHEMA_NAME},)

    sequence_number: BigIntegerPrimaryKeyType
    creation_number: BigIntegerPrimaryKeyType
    account_address: StringPrimaryKeyType
    prediction: BooleanType     # from (event.data["prediction"]
    result: BooleanType         # from (event.data["result"]
    wins: BigIntegerType        # from (event.data["wins"]
    losses: BigIntegerType      # from (event.data["losses"]
    win_percentage: NumericType # calculated from the above
    transaction_version: BigIntegerType
    transaction_timestamp: TimestampType
    inserted_at: InsertedAtType
    event_index: BigIntegerType
```

マークのないフィールドは、デフォルトのイベントデータ(Aptosで発行されたすべてのイベントのデータ)からのものです。マークされたフィールドは、具体的には上記で計算したフィールドからのものです。


他のフィールドの**tablename**と**table_args**は 
使用しているデータベースとスキーマ名をPython SQLAlchemyライブラリに示します。


## インデクサープロセッサーの実行

構成ファイル、データベース、Python データベース モデルが設定されたので、プロセッサを実行できます。

インデクサーリポジトリの`python`ディレクトリに移動します。

```shell
cd ~/aptos-indexer-processors/python
```

以下のコマンドを実行します。

```shell
poetry run python -m processors.main -c processors/coin_flip/config.yaml
```

pyenvのバージョンに関するエラーが発生した場合は、正しいバージョンをインストールする必要がある場合があります。

```shell
 pyenv install 3.11.0
```

依存関係エラーが多数発生する場合は、Poetry を使用して依存関係をインストールする必要がある場合があります。/aptos-indexer-processorsから以下のコマンドを実行します。


```shell
poetry install
```

イベントを正しく処理している場合は、ターミナル出力に以下のような内容が表示されるはずです。

```shell
{"timestamp": "2023-12-07 18:08:26,493", "level": "INFO", "fields": {"message": "[Parser] Kicking off", "processor_name": "coin_flip", "service_type": "processor"}, "module": "worker", "func_name": "__init__", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/worker.py", "line_no": 463}
{"timestamp": "2023-12-07 18:08:26,494", "level": "INFO", "fields": {"message": "[Parser] Initializing DB tables", "processor_name": "coin_flip", "service_type": "processor"}, "module": "worker", "func_name": "run", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/worker.py", "line_no": 592}
{"timestamp": "2023-12-07 18:08:26,694", "level": "INFO", "fields": {"message": "[Parser] DB tables initialized", "processor_name": "coin_flip", "service_type": "processor"}, "module": "worker", "func_name": "run", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/worker.py", "line_no": 600}
{"timestamp": "2023-12-07 18:08:26,712", "level": "INFO", "fields": {"message": "[Config] Starting from config starting_version"}, "module": "config", "func_name": "get_starting_version", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/config.py", "line_no": 77}
{"timestamp": "2023-12-07 18:08:26,712", "level": "INFO", "fields": {"message": "[Parser] Starting fetcher task", "processor_name": "coin_flip", "stream_address": "grpc.testnet.aptoslabs.com:443", "start_version": 636585029, "service_type": "processor"}, "module": "worker", "func_name": "run", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/worker.py", "line_no": 616}
{"timestamp": "2023-12-07 18:08:26,712", "level": "INFO", "fields": {"message": "[Parser] Setting up rpc channel", "processor_name": "coin_flip", "stream_address": "grpc.testnet.aptoslabs.com:443", "service_type": "processor"}, "module": "worker", "func_name": "get_grpc_stream", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/worker.py", "line_no": 56}
{"timestamp": "2023-12-07 18:08:26,730", "level": "INFO", "fields": {"message": "[Parser] Setting up stream", "processor_name": "coin_flip", "stream_address": "grpc.testnet.aptoslabs.com:443", "starting_version": 636585029, "ending_version": 636589723, "count": 4695, "service_type": "processor"}, "module": "worker", "func_name": "get_grpc_stream", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/worker.py", "line_no": 90}
{"timestamp": "2023-12-07 18:08:26,733", "level": "INFO", "fields": {"message": "[Parser] Successfully connected to GRPC endpoint", "processor_name": "coin_flip", "stream_address": "grpc.testnet.aptoslabs.com:443", "starting_version": 636585029, "ending_version": 636589723, "service_type": "processor"}, "module": "worker", "func_name": "producer", "path_name": "/Users/jinhou/Projects/aptos-indexer-processors/python/utils/worker.py", "line_no": 147}
```

この様な出力が出たら大丈夫です！これで、データベースを表示して、保存されているデータを確認できるようになります。データベースには、`coin_flip_events`と`next_versions_to_process`の2つのテーブルが作成されているはずです。解析されたデータは`coin_flip_events`に保存されます。
