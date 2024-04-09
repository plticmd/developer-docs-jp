---
title: "トランザクションの解析(Japanese)"
slug: "indexer-custom-processors-txns-jp"
---

# トランザクションの解析

import BetaNotice from '../../../src/components/\_indexer_beta_notice_jp.mdx';

<BetaNotice />

<!--
Things to add:
- We should have tabs for each language that mentions helper functions for extracting the thing you want. For example, if the user is trying to extract the entry function arguments, there should be a function like `get_entry_function_arguments` and we show how to use it in each language and where it comes from in the SDK.
-->

基本的に、インデクサープロセッサは、トランザクションのストリームを消費し、処理されたデータをストレージに書き込む物にすぎません。トランザクションとは何か、そしてトランザクションからどのような情報を抽出できるのか詳しく見てみましょう。

## トランザクションとは何か?

トランザクションは、Aptosブロックチェーン上の実行単位です。トランザクション内のプログラムの実行(例えば、Moveモジュールのエントリ関数から開始)が成功すると、その結果生じた状態の変更が台帳に適用されます。トランザクションのライフサイクルの詳細については、[このページ](../../concepts/blockchain.md#life-of-a-transaction)を御覧下さい。

Aptosには4種類のトランザクションがあります。

- ジェネシス
- ブロックメタデータトランザクション
- 状態チェックポイントトランザクション
- ユーザートランザクション

これらのうち最初の3つはシステムの内部にあり、ほとんどのプロセッサには関係ありません。このガイドではそれらについて解説しません。

一般に、ほとんどのユーザートランザクションは、ユーザーがチェーン上にデプロイされたMoveモジュールのエントリ関数を呼び出すことから始まります。例えば`0x1::coin::transfer`。それ以外の場合はすべて、[Moveスクリプト](/move/move-on-aptos/scripts/index.md)から発生します。別のタイプのトランザクションについて詳細は、[こちら](../../concepts/txns-states.md#types-of-transaction-payloads)を御覧下さい。

プロセッサが処理するユーザートランザクションには、様々な情報が含まれています。高レベルで含まれるのは...

- 送信されたペイロード。
- 関数/スクリプトの実行によって生じる台帳への変更。

これについては以下のセクションで詳しく解説します。

## トランザクションにおいて重要な事は何か?

### ペイロード

ペイロードは、ユーザーがMove関数を実行したい時にブロックチェーンに送信するものです。ペイロード内の重要な情報の一部は...

- 送信者のアドレス
- 実行中の関数のアドレス + モジュール名 + 関数名。
- 関数の引数。


ペイロードには他にも潜在的に興味深い情報が含まれており、[このページ](/concepts/txns-states#contents-of-a-transaction)で学ぶ事が出来ます。

### イベント

イベントはトランザクションの実行中に発行されます。各Moveモジュールは独自のイベントを定義し、関数の実行中のいつイベントを発行するのか選択できます。

たとえば、移動では以下のようになります。


```rust
struct MemberInvitedEvent has store, drop {
    member: address,
}

public entry fun invite_member(member: address) {
    event::emit_event(
        &mut member_invited_events,
        MemberInvitedEvent { member },
    );
}
```

`invite_member`が呼び出されると、トランザクション内に`MemberInvitedEvent`が表示されます。

:::tip Why emit events?
(イベントを発行する理由)
これは良い質問ですね！場合によっては、書き込みセットを解析するだけで済むため、イベントを発行する必要がない場合もあります。ただし、トランザクション内のさまざまな「場所」から必要なデータをすべて取得することが難しい場合や、書き込みセットに含まれていないデータにインデックスを付けたい場合など、それが不可能な場合もあります。その場合、イベントはインデックスを作成するすべてのものを1つにまとめる便利な方法です。
:::

### 書き込みセット

トランザクションが実行されても、その時点ではオンチェーンの状態には直接影響しません。その代わり、書き込みセットと呼ばれる台帳に一連の変更を加え出力します。書き込みセットは、すべてのバリデーターが実行結果に同意した後、台帳に適用されます。

書き込みセットは、トランザクションが発生した後のオンチェーンデータの最終状態を示します。これらは、どのデータがオンチェーンに保存されているかに関する真実の情報源です。
（これらはオンチェーンに保存された真の情報源です。）
書き込みセットの変更にはいくつかの種類があります。

- モジュールの書き込み/モジュールの削除
- リソースの書き込み/リソースの削除
- テーブル項目の書き込み/テーブル項目の削除

<!-- Add more information about writesets, ideally once have the helper functions. -->
