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

Fundamentally an indexer processor is just something that consumes a stream of a transactions and writes processed data to storage. Let's dive into what a transaction is and what kind of information you can extract from one.
基本的に、インデクサー プロセッサは、トランザクションのストリームを消費し、処理されたデータをストレージに書き込むものにすぎません。トランザクションとは何か、そしてトランザクションからどのような種類の情報を抽出できるのかを詳しく見てみましょう。



## トランザクションとは何か?

A transaction is a unit of execution on the Aptos blockchain. If the execution of the program in a transaction (e.g. starting with an entry function in a Move module) is successful, the resulting change in state will be applied to the ledger. Learn more about the transaction lifecycle at [this page](../../concepts/blockchain.md#life-of-a-transaction).
トランザクションは、Aptos ブロックチェーン上の実行単位です。トランザクション内のプログラムの実行 (たとえば、Move モジュールのエントリ関数から開始) が成功すると、結果として生じる状態の変更が台帳に適用されます。トランザクションのライフサイクルの詳細については、このページをご覧ください。

There are four types of transactions on Aptos:
Aptos には 4 種類のトランザクションがあります。



- ジェネシス
- メタデータトランザクションをブロックする
- State checkpoint transactions状態チェックポイントトランザクション
- ユーザートランザクション

The first 3 of these are internal to the system and are not relevant to most processors; we do not cover them in this guide.
これらのうち最初の 3 つはシステムの内部にあり、ほとんどのプロセッサには関係ありません。このガイドではそれらについては説明しません。

Generally speaking, most user transactions originate from a user calling an entry function in a Move module deployed on chain, for example `0x1::coin::transfer`. In all other cases they originate from [Move scripts](/move/move-on-aptos/scripts/index.md). You can learn more about the different types of transactions [here](../../concepts/txns-states.md#types-of-transaction-payloads).
一般に、ほとんどのユーザー トランザクションは、ユーザーがチェーン上にデプロイされた Move モジュールのエントリ関数を呼び出すことから始まります0x1::coin::transfer。それ以外の場合はすべて、移動スクリプトから発生します。さまざまなタイプのトランザクションについて詳しくは、こちらをご覧ください。

A user transaction that a processor handles contains a variety of information. At a high level it contains:
プロセッサが処理するユーザー トランザクションには、さまざまな情報が含まれています。大まかに言うと、次のものが含まれます。

- The payload that was submitted.
送信されたペイロード。
- The changes to the ledger resulting from the execution of the function / script.
関数/スクリプトの実行によって生じる台帳への変更。

We'll dive into this in the following sections.
これについては次のセクションで詳しく説明します。

## 取引において重要なことは何か?

### ペイロード

The payload is what the user submits to the blockchain when they wish to execute a Move function. Some of the key information in the payload is:
ペイロードは、ユーザーが移動関数を実行したいときにブロックチェーンに送信するものです。ペイロード内の重要な情報の一部は次のとおりです。

- 送信者のアドレス
- 実行中の関数のアドレス + モジュール名 + 関数名。
- 関数の引数。


There is other potentially interesting information in the payload that you can learn about at [this page](/concepts/txns-states#contents-of-a-transaction).
ペイロードには他にも潜在的に興味深い情報が含まれており、このページで知ることができます。


### イベント

Events are emitted during the execution of a transaction. Each Move module can define its own events and choose when to emit the events during execution of a function.
イベントはトランザクションの実行中に発行されます。各 Move モジュールは独自のイベントを定義し、関数の実行中にイベントを発行するタイミングを選択できます。


For example, in Move you might have the following:
たとえば、移動では次のようになります。


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

If `invite_member` is called, you will find the `MemberInvitedEvent` in the transaction.
が呼び出されると、トランザクション内にinvite_memberが表示されます。MemberInvitedEvent

:::tip Why emit events?
This is a good question! In some cases, you might find it unnecessary to emit events since you can just parse the writesets. However, sometimes it is quite difficult to get all the data you need from the different "locations" in the transaction, or in some cases it might not even be possible, e.g. if you want to index data that isn't included in the writeset. In these cases, events are a convenient way to bundle together everything you want to index.
イベントを発行する理由
これは良い質問ですね！場合によっては、書き込みセットを解析するだけで済むため、イベントを発行する必要がない場合もあります。ただし、トランザクション内のさまざまな「場所」から必要なデータをすべて取得することが非常に難しい場合や、書き込みセットに含まれていないデータにインデックスを付けたい場合など、場合によってはそれが不可能な場合もあります。 。このような場合、イベントは、インデックスを作成するすべてのものを 1 つにまとめる便利な方法です。
:::

### Writesets　書き込みセット

When a transaction executes, it doesn't directly affect on-chain state right then. Instead, it outputs a set of changes to be made to the ledger, called a writeset. The writeset is applied to the ledger later on after all validators have agreed on the result of the execution.
トランザクションが実行されても、その時点ではオンチェーンの状態には直接影響しません。代わりに、書き込みセットと呼ばれる台帳に加えられる一連の変更を出力します。ライトセットは、すべてのバリデーターが実行結果に同意した後、レジャーに適用されます。

Writesets show the end state of the on-chain data after the transaction has occurred. They are the source of truth of what data is stored on-chain. There are several types of write set changes:
ライトセットは、トランザクションが発生した後のオンチェーン データの最終状態を示します。これらは、どのデータがオンチェーンに保存されているかに関する真実の情報源です。書き込みセットの変更にはいくつかの種類があります。

- モジュールの書き込み/モジュールの削除
- リソースの書き込み/リソースの削除
- テーブル項目の書き込み/テーブル項目の削除

<!-- Add more information about writesets, ideally once have the helper functions. -->
