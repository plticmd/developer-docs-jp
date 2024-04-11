---
title: "ラボがホストするトランザクション ストリームサービス(Japanese)"
slug: "indexer-txn-stream-labs-hosted-jp"
---

# ラボがホストするトランザクションストリームサービス

import BetaNotice from '../../../src/components/\_indexer_beta_notice_jp.mdx';

<BetaNotice />

[インデクサーAPI](/indexer/api)のインスタンス(自前の)または[カスタムプロセッサー](/indexer/custom-processors)を実行している場合は、トランザクションストリームサービスのインスタンスにアクセスする必要があります。このページは、Labs-Hostedトランザクションストリームサービスの使用方法の情報を含んでいます。

## エンドポイント

特に指定がない限り、すべてのエンドポイントはGCP us-central1にあります。

- **メインネット:** grpc.mainnet.aptoslabs.com:443
- **テストネット:** grpc.testnet.aptoslabs.com:443
- **開発ネット:** grpc.devnet.aptoslabs.com:443

<!--
## Rate limits
The following rate limit applies for the Aptos Labs hosted Transaction Stream Service:

- todo todo

If you need a higher rate limit, consider running the Transaction Stream Service yourself. See the guide to self-hosting [here](./self-hosted).
-->

## APIキーで認証する

Labs-Hostedトランザクションストリームサービスを使用するには、APIキーが必要です。APIキーを取得するには、以下の手順を実行します。

1. https://developers.aptoslabs.com へ移動します。
2. サインインし、左側のサイドバーで「APIキー」を選択します。
3. 新しいキーを作成します。最初の表にトークン値が表示されます。

`Authorization`HTTPヘッダー([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization))を設定することでAPIキーを提供できます。例えば、carlの場合は...

```
curl -H 'Authorization: Bearer aptoslabs_yj4donpaKy_Q6RBP4cdBmjA8T51hto1GcVX5ZS9S65dx'
```

トランザクションストリームサービスの使用方法に関するより包括的な情報については、ダウンストリームシステムのドキュメントを御覧下さい。

- [インデクサーAPI](/indexer/api/self-hosted)
- [カスタムプロセッサー](/indexer/custom-processors)
