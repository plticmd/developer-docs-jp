---
title: "フルノードREST API (in Japanese)" 
slug: "apis-fullnode-rest-api-jp"
---

# AptosフルノードREST APIを使う

Aptos APIを使用したい場合はこのガイドが最適です。このガイドではAptos APIを使用してAptosブロックチェーンをプラットフォームに統合するための手順を全て解説します。

:::tip
Aptos統合の詳細なチュートリアルについては[システム統合ガイド](../guides/system-integrators-guide.md)も参照して下さい。
:::

## レート制限を理解する

[Aptosインデクサー](../indexer/api/labs-hosted.md#rate-limits)と同様に、Aptos REST APIはトランザクションを送信するかAptosが提供するノード上でAPIをクエリするかに関わらず、IPアドレスごとに5分あたり5000リクエストのレート制限が有ります。(ノードオペレーターは自分のノードの制限を引き上げる事が出来ます。)予告の有無に関わらずこの制限は変更される可能性があります。 

## 現在と過去の状態の閲覧

Aptosブロックチェーンへの統合の殆どは、ブロックチェーンの現在および過去の状態の全体的で包括的な概要から恩恵を受けます。Aptosはトランザクション履歴、状態、イベント、全てのトランザクション実行の結果を提供します。

- トランザクション履歴は実行状態、出力を指定し、関連イベントと連動します。各トランザクションはトランザクションと関連する固有のバージョン番号を持ち、ブロックチェーン台帳の履歴におけるトランザクションのグローバルな連続順序を決定します。　
- 状態は特定のバージョンまでの全てのトランザクション出力を代表します。言い換えると状態バージョンは、そのトランザクションバージョンを含む全てのトランザクションの累積です。
- トランザクションの実行時、イベントが発行される場合があります。[イベント](../concepts/events.md)は、オンチェーンデータの変更に関するヒントです。


:::important
必ず最新の[フルノード](../nodes/networks.md)で通信して下さい。フルノードは、トランザクションから適切なデータを取得するため、あなたのトランザクションを含んだバージョンに達している必要があります。フルノードが[バリデーターフルノード](../concepts/fullnodes.md)から状態を取得する際遅延が発生する可能性があり、真の情報源として[バリデーターノード](../concepts/validator-nodes.md)に依存しているためです。 
:::

ノード上のストレージサービスは、ノードからデータを消去する以下の2つの形式のプルーニングを採用しています。

- 状態
- イベント、トランザクション、その他全て

これらのいずれかを無効にする事は出来ますが、状態バージョンの保存は特に持続可能ではありません。

イベントとトランザクションのプルーニングは、`storage_config.rs`内で[`enable_ledger_pruner`](https://github.com/aptos-labs/aptos-core/blob/cf0bc2e4031a843cdc0c04e70b3f7cd92666afcf/config/src/config/storage_config.rs#L141)を`false`に設定することで無効に出来ます。これはメインネットのデフォルトの動作です。近い将来Aptosはノードから直接クエリする必要性の低いインデクサーを提供する予定です。

REST APIでは以下の方法でトランザクションのクエリとイベントを実行できます。

- [アカウントのトランザクション](https://api.devnet.aptoslabs.com/v1/spec#/operations/get_account_transactions)
- [バージョン別のトランザクション](https://api.devnet.aptoslabs.com/v1/spec#/operations/get_transaction_by_version)
- [イベントハンドル別のイベント](https://api.devnet.aptoslabs.com/v1/spec#/operations/get_events_by_event_handle)

## View関数で状態を読みとる

View関数はAPIから呼び出された時、ブロックチェーンの状態を変更しません。[View](https://github.com/aptos-labs/aptos-core/blob/main/api/src/view_function.rs)関数とその[入力](https://github.com/aptos-labs/aptos-core/blob/main/api/types/src/view.rs)を使用すると、潜在的に複雑なオンチェーン状態をMoveを使用して読み取る事が出来ます。例えば、オークションコントラクトで誰が最も高い入札をしたのか評価できます。関連ファイルは以下の通りです。

- [`view_function.rs`](https://github.com/aptos-labs/aptos-core/blob/main/api/src/tests/view_function.rs)の例
- [Move](https://github.com/aptos-labs/aptos-core/blob/90c33dc7a18662839cd50f3b70baece0e2dbfc71/aptos-move/framework/aptos-framework/sources/coin.move#L226)と関連するコード
- [仕様](https://github.com/aptos-labs/aptos-core/blob/90c33dc7a18662839cd50f3b70baece0e2dbfc71/api/doc/spec.yaml#L8513).

view関数は[Aptos Simulation API](../guides/system-integrators-guide.md#testing-transactions-or-transaction-pre-execution)と同様に動作しますが副作用はなく、出力パスにアクセス出来ます。view関数は`/view`エンドポイント経由で呼び出す事が出来ます。view関数の呼び出しにはモジュール名と関数名、およびインプットタイプのパラメーターと値が必要です。

関数は`#[view]`としてタグ付けされるために不変(immutable)である必要はありませんが、関数が可変(mutable)である場合、APIから呼び出された時状態の変更は発生しません。可変関数に`#[view]`のタグを付けたい場合は、実行時に悪意を持って呼び出されない様に関数をプライベートにする事を検討して下さい。

View関数を使用するには、[Aptos CLI](../tools/aptos-cli/install-cli/index.md)を介して
[モジュールを公開](../move/move-on-aptos/cli.md#publishing-a-move-package-with-a-named-address)する必要があります。

Aptos CLIでは、view関数リクエストは以下の様になります。

```
aptos move view --function-id devnet::message::get_message --profile devnet --args address:devnet
{
  "Result": [
    "View functions rock!"
  ]
}
```

TypeScript SDKでは、view関数リクエストは以下の様になります。

```
    const payload: Gen.ViewRequest = {
      function: "0x1::coin::balance",
      type_arguments: ["0x1::aptos_coin::AptosCoin"],
      arguments: [alice.address().hex()],
    };

    const balance = await client.view(payload);

    expect(balance[0]).toBe("100000000");
```

view関数は、値のリストをベクトルとして返します。デフォルトでは結果はJSON形式で返されます。ただし、オプションでBinary Canonical Serialization(BCS)エンコード形式で返す事もできます。


最終更新日2024年3月23日
