---
title: "日本語 (in japanese)"
slug: "typescript-sdk-tests-jp"
---

# テストと検証

TypeScriptSDKは、`src/tests/`フォルダー下にある`e2e`と`unit`の2種類のテストを使用します。

- `e2e`テスト – エンドtoエンドテストは、SDKメソッドから始まり、REST/インデクサーAPIとスマートコントラクトと対話し、ブロックチェーンレベルに至るまでのエンドtoエンド操作のテストをする事が目的です。たとえば、トランザクションが送信されたかどうかをテストするには、SDKが予期するトランザクションペイロードを構築することから始め、REST APIに送信リクエストをポストし、トランザクションデータを取得してブロックチェーンに完全に送信されたことを確認します。
- `unit`テスト – 単体テストは、提供された入力を使用してSDK内の関数の出力をテストする事が目的です。例えば、アカウントのアドレスが有効かどうかをテストできます。

## トランザクションビルダーとBCS

[BCS](https://docs.rs/bcs/latest/bcs/)は、署名、送信するためのトランザクションペイロードを組み立てシリアル化する際使用されます。

異なるプログラミング言語には異なるプリミティブ型の制約 (例: バイト長、値の範囲など) とさまざまな複合的な型のサポート (例: enum、struct、class等)がある事を考えると、データをシリアル化したコードを検証するのは困難です。

Aptos SDK は、トランザクションビルダーとBCSを以下の2つの方法で検証します。

1. 最初のレベルの検証は、単体テストとエンドtoエンド(e2e)テストを通して行われます。

:::tip
BCS シリアライザーの単体テストの例は、[`serializer.test.ts`](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/tests/unit/serializer.test.ts)にあります。

BCSトランザクションを送信するためのe2eテストの例は、[`aptos_client.test.ts`](https://github.com/aptos-labs/aptos-core/blob/f4a7820a61f22ed8306219621402d96f70379d20/ecosystem/typescript/sdk/src/tests/e2e/aptos_client.test.ts#L78)にあります。
:::

2. 検証の2番目のレベルは、テストベクトルを使用したファジングテストです。テストベクトルは、Aptosブロックチェーンで使用されているのと同じコードによって生成されます。テストベクトルはJSONオブジェクトの配列です。各JSONオブジェクトは、ランダム化された入力と予期された出力を含みます。 AptosSDKは、テストベクトルを解析およびロードして、トランザクションビルダーとBCSの実装を検証できます。

2つのテストベクトルがあります。それぞれが1種類のトランザクションペイロードをカバーします。

- [エントリー関数](https://github.com/aptos-labs/aptos-core/blob/main/api/goldens/aptos_api__tests__transaction_vector_test__test_entry_function_payload.json)ベクトル
- [スクリプト](https://github.com/aptos-labs/aptos-core/blob/main/api/goldens/aptos_api__tests__transaction_vector_test__test_script_payload.json)ベクトル

ベクトルアイテムは一目瞭然ですが、スペースを節約しデータのオーバーフローを回避するため、特別なシリアル化メソッドが使用されます。詳細は以下で解説します。

- すべてのアカウントアドレスは16進コード化されています。
- エントリー関数の`args`(引数)は16進コード化されます。
- U64、U128の数値は文字列リテラルとしてシリアル化され、データの切り捨てを回避します。
- U8は(文字列ではなく)数値としてシリアル化されます。
- スクリプトの`code`とモジュールBundleは16進コード化されています。

:::tip

タイプスクリプトSDKがベクトル検証を行う方法については、[`transaction_vector.test.ts`](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/tests/unit/transaction_vector.test.ts)コード例を参照してください。

:::
