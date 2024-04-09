---
title: "インデクサーAPI(in Japanese)"
slug: "indexer-api-index-jp"
---

# インデクサーAPI

import BetaNotice from '../../../src/components/\_indexer_beta_notice_jp.mdx';

<BetaNotice />

このセクションには、aptosインデクサーAPIのドキュメントが含まれています。このAPIは、[aptos-labs/aptos-indexer-processors](https://github.com/aptos-labs/aptos-indexer-processors)リポジトリで提供するプロセッサの標準セットに基づいて構築されています。 

## 利用ガイド

### アドレス形式

クエリパラメータの1つがアカウントアドレス(所有者など)であるクエリを作成する場合は、アドレスが`0x`プレフィックスで始まり、その後64個の16進数文字が続くことを確認してください。例えば、`0xaa921481e07b82a26dbd5d3bc472b9ad82d3e5bfd248bacac160eac51687c2ff`。

### TypeScriptクライアント

 Aptos TypeScript SDKは、AptosインデクサーAPIにクエリを実行するためのAPI関数を提供します。詳細は[こちら](../../sdks/ts-sdk/fetch-data-from-chain.md)を御覧下さい。

