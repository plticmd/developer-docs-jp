---
title: "インデクサーAPIクエリの例(Japanese)"
slug: "indexer-api-example-queries-jp"
---

# インデクサーAPIクエリの例

import BetaNotice from '../../../src/components/\_indexer_beta_notice_jp.mdx';

<BetaNotice />

## サンプルクエリの実行

1. クエリするネットワークのHasuraエクスプローラーを開きます。URLは[ここ](/indexer/api/labs-hosted#hasura-explorer)で見つけることができます。
2. サンプルの**クエリ**コードをメインクエリセクションに貼り付け、同じサンプルの**クエリ変数**コードをクエリ変数セクション(メインクエリセクションの下)に貼り付けます。

## その他の例

[TypeScript SDK](https://github.com/aptos-labs/aptos-ts-sdk/tree/main/src/internal/queries)には多くのサンプルクエリが含まれています。TypeScript SDKを使用している場合は、[API](https://github.com/aptos-labs/aptos-ts-sdk/tree/main/src/api)を確認する必要があります。

## トークンクエリの例

現在アカウントにあるすべてのトークンを取得します。

**クエリ**

```graphql
query CurrentTokens($owner_address: String, $offset: Int) {
  current_token_ownerships(
    where: {
      owner_address: { _eq: $owner_address }
      amount: { _gt: "0" }
      table_type: { _eq: "0x3::token::TokenStore" }
    }
    order_by: [{ last_transaction_version: desc }, { token_data_id: desc }]
    offset: $offset
  ) {
    token_data_id_hash
    name
    collection_name
    property_version
    amount
  }
}
```

**クエリ変数**

```json
{
  "owner_address": "0xaa921481e07b82a26dbd5d3bc472b9ad82d3e5bfd248bacac160eac51687c2ff",
  "offset": 0
}
```

---

目的のトークンのトークンアクティビティを全て取得します。**注意**`token_id_hash`を取得するには、まずクエリを作成して上記のクエリからトークンを取得する必要があります。

**クエリ**

```graphql
query TokenActivities($token_id_hash: String, $offset: Int) {
  token_activities(
    where: { token_data_id_hash: { _eq: $token_id_hash } }
    # Needed for pagination
    order_by: [{ last_transaction_version: desc }, { event_index: asc }]
    # Optional for pagination
    offset: $offset
  ) {
    transaction_version
    from_address
    property_version
    to_address
    token_amount
    transfer_type
  }
}
```

**クエリ変数**

```json
{
  "token_id_hash": "f344b838264bf9aa57d5d4c1e0c8e6bbdc93f000abe3e7f050c2a0f4dc23d030",
  "offset": 0
}
```

---

アカウントに提供された現在のトークンを取得します。

**クエリ**

```graphql
query CurrentOffers($to_address: String, $offset: Int) {
  current_token_pending_claims(
    where: { to_address: { _eq: $to_address }, amount: { _gt: "0" } }
    # Needed for pagination
    order_by: [{ last_transaction_version: desc }, { token_data_id: desc }]
    # Optional for pagination
    offset: $offset
  ) {
    token_data_id_hash
    name
    collection_name
    property_version
    from_address
    amount
  }
}
```

**クエリ変数**

```json
{
  "to_address": "0xe7be097a90c18f6bdd53efe0e74bf34393cac2f0ae941523ea196a47b6859edb",
  "offset": 0
}
```

## コインクエリの例

コインアクティビティの取得（ガス料金を含む）。

**クエリ**

```graphql
query CoinActivity($owner_address: String, $offset: Int) {
  coin_activities(
    where: { owner_address: { _eq: $owner_address } }
    # Needed for pagination
    order_by: [{ last_transaction_version: desc }, { event_index: asc }]
    # Optional for pagination
    offset: $offset
  ) {
    activity_type
    amount
    coin_type
    entry_function_id_str
    transaction_version
  }
}
```

**クエリ変数**

```json
{
  "owner_address": "0xe7be097a90c18f6bdd53efe0e74bf34393cac2f0ae941523ea196a47b6859edb",
  "offset": 0
}
```

---

現在所有しているコイン(`0x1::coin::CoinStore`)。

**クエリ**

```graphql
query CurrentBalances($owner_address: String, $offset: Int)Ï {
  current_coin_balances(
    where: {owner_address: {_eq: $owner_address}}
    # Needed for pagination
    order_by: [{last_transaction_version: desc}, {token_data_id: desc}]
    # Optional for pagination
    offset: $offset
  ) {
    owner_address
    coin_type
    amount
    last_transaction_timestamp
  }
}
```

**クエリ変数**

```json
{
  "owner_address": "0xe7be097a90c18f6bdd53efe0e74bf34393cac2f0ae941523ea196a47b6859edb",
  "offset": 0
}
```

## エクスプローラークエリの例

すべてのユーザートランザクションバージョンを取得します (ブロックエクスプローラーのユーザートランザクションをフィルターするため)。

**クエリ**

```graphql
query UserTransactions($limit: Int) {
  user_transactions(limit: $limit, order_by: { version: desc }) {
    version
  }
}
```

**クエリ変数**

```json
{
  "limit": 10
}
```
