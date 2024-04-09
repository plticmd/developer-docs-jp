---
title: "ラボがホストするインデクサーAPI(Japanese)"
slug: "indexer-api-labs-hosted-jp"
---

# ラボがホストするインデクサーAPI

import BetaNotice from '../../../src/components/\_indexer_beta_notice_jp.mdx';

<BetaNotice />

## グラフQL APIエンドポイント

ラボがホストするインデクサーAPIに対してグラフQLクエリを作成する場合は、以下のエンドポイントを使用します。

- **メインネット:** https://api.mainnet.aptoslabs.com/v1/graphql
- **テストネット:** https://api.testnet.aptoslabs.com/v1/graphql
- **Devネット:** https://api.devnet.aptoslabs.com/v1/graphql

## ハスラエクスプローラー

以下のURLは、ラボがホストするインデクサーAPIのハスラエクスプローラー用です。

- **メインネット:** https://cloud.hasura.io/public/graphiql?endpoint=https://api.mainnet.aptoslabs.com/v1/graphql
- **テストネット:** https://cloud.hasura.io/public/graphiql?endpoint=https://api.testnet.aptoslabs.com/v1/graphql
- **Deネット:** https://cloud.hasura.io/public/graphiql?endpoint=https://api.devnet.aptoslabs.com/v1/graphql

## レート制限

AptosラボがホストするインデクサーAPIには、以下のレート制限が適用されます。

- クライアント(ウォレットやエクスプローラー等)から直接インデクサーAPI(Aptos提供の)を呼び出すWebアプリケーションのレート制限は現在IPアドレスごとに5分あたり5000リクエストです。**この制限は予告無しに変更される可能性があります。**

より高いレート制限が必要な場合は、AptosインデクサーAPIを自分で実行することを検討してください。[こちら](/indexer/api/self-hosted)でセルフホスティングのガイドを参照してください。

