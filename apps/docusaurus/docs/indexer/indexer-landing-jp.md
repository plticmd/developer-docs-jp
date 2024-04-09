---
title: "インデクサー(in Japanese)"
slug: "indexer-indexer-landing-jp"
---

# インデクサーを学ぶ

import BetaNotice from '../../src/components/\_indexer_beta_notice_jp.mdx';
import ThemedImage from '@theme/ThemedImage';
import useBaseUrl from '@docusaurus/useBaseUrl';

<BetaNotice />

## クイックスタート

この役割別ガイドを参照すると関連ドキュメントを素早く見つけられます。

- コアインフラプロバイダー: スタックの残りの部分に加えて、独自のトランザクションストリームサービスを実行したい場合...
  - [セルフホスト型トランザクションストリームサービス](/indexer/txn-stream/self-hosted)のドキュメントを御覧下さい。
- APIオペレーター: ホストされたトランザクションストリーム サービス上でインデクサーAPIを実行したい場合...
  - [セルフホスト型インデクサーAPI](/indexer/api/self-hosted)のドキュメントを御覧下さい。
-  カスタムプロセッサービルダー: ホストされたトランザクションストリームサービス上にカスタムプロセッサ-を構築したい場合...
  - [カスタムプロセッサー](/indexer/custom-processors)に関するドキュメントを御覧下さい。
- インデクサーAPI消費者: ホストされたインデクサーAPIを使用したい場合...
  - [ラボホスト型インデクサーAPI](/indexer/api/labs-hosted)のドキュメントを御覧下さい。
  - [インデクサーAPI使用ガイド](/indexer/api/usage-guide)を御覧下さい。
  
# 構造の概要

Aptosブロックチェーン、さらに言えばどのブロックチェーン上に構築された典型的なアプリケーションでも、生のブロックチェーンデータをアプリケーション固有の方法で整形し、保存する必要があります。これは、何百万ものユーザーからエンドユーザーアプリでブロックチェーンデータを使用する際、低レイテンシーと豊富なエクスペリエンスをサポートするために不可欠です。[AptosノードAPI](https://aptos.dev/nodes/aptos-api-spec#/)は、低レベルの安定した汎用APIを提供しますが、データ整形をサポートするように設計されていないため、豊富なエンドユーザーエクスペリエンスを直接サポートすることはできません。

Aptosインデクサーはこの要望への答えであり、リアルタイム アプリの使用に不可欠なデータ整形を可能にします。Aptosのインデックス作成の仕組みについては、以下の概要図を御覧下さい。

<center>
<ThemedImage
alt="Signed Transaction Flow"
sources={{
    light: useBaseUrl('/img/docs/aptos-indexing.svg'),
    dark: useBaseUrl('/img/docs/aptos-indexing-dark.svg'),
  }}
/>
</center>

大まかに言うと、Aptosブロックチェーンでのインデックス作成は以下のように機能します。

1. dappのユーザー、例えばNFTマーケットプレイス上では、dappが提供する多彩なUIを介してAptosブロックチェーンと対話します。これらのやりとりは舞台裏で、スマートコントラクトを介してトランザクションデータとイベントデータを生成します。この生データは、Aptosフルノードなどの分散台帳データベースに保存されます。
2. この生の台帳データは、アプリケーション固有のデータモデルを使用して読み取り、インデックス付けされます。この場合はNFTマーケットプレイス固有のデータモデル(上図「ビジネスロジック」)を使用。このNFTマーケットプレイス固有のインデックスは別々のデータベース(上図「インデックス付きデータベース」)に保存され、API経由で公開されます。
3. dappは、このインデックス付きデータベースにNFT固有のグラフQLクエリを送信し、豊富なデータを受信して​​ユーザーに提供します。

ステップ2は、Aptosインデクサーによって促進されます。上図は、システムが高レベルでどのように動作するかを簡略化したものです。実際には、システムは多くのコンポーネントで構成されています。興味がある場合は、以下の[詳細な概要](#detailed-overview)をご覧下さい。

## インデクサーAPI

Aptosは、インデックス付きデータにアクセスする以下の方法をサポートしています。

1. [ラボがホストするインデクサーAPI](/indexer/api/labs-hosted):このAPIはレート制限があり、ウォレットなどの軽量アプリケーションのみを対象としています。このオプションは、高帯域幅アプリケーションには推奨されていません。 
2. [セルフホスト型インデクサーAPI](/indexer/api/self-hosted):ラボがホストするインデクサースタックであなたのデプロイメントを実行します。
3. [カスタムプロセッサ](/indexer/custom-processors): カスタムプロセッサを作成してデプロイし、必要な固有の方法でデータをインデックスし、公開します。

## トランザクションストリームサービス

インデクサーAPIとカスタムプロセッサは、トランザクションストリームサービスに依存します。つまり、このサービスは、プロセッサが消費するトランザクションのGRPCストリームを提供します。このサービスの詳細については、[こちら](/indexer/txn-stream/)を御覧下さい。Aptosラボは[このサービスでホストされたインスタンス](/indexer/txn-stream/labs-hosted)を提供していますが、[独自のインスタンスを実行](/indexer/txn-stream/self-hosted)することもできます。

## 詳細な概要

この図は、Aptosインデクサー技術スタックがどのように機能するのか詳しく解説しています。

<div style={{textAlign:"center"}}>
<div style={{marginBottom: 20}}>
<iframe
  style={{border: "1px solid rgba(0, 0, 0, 0.1);"}}
  width="100%"
  height="750"
  src="https://www.figma.com/embed?embed_host=share&url=https%3A%2F%2Fwww.figma.com%2Ffile%2FsVhSOGR7ZT4CdeUzlXyduD%2FIndexer-Overview%3Ftype%3Dwhiteboard%26node-id%3D0%253A1%26t%3DUnUKeEaBE7ETMksb-1"
  allowfullscreen>
</iframe>
</div>
</div>

<!-- TODO: Write an explanation of this diagram. -->

## 従来のインデクサー

従来のインデクサーに関する情報は、[こちら](/indexer/legacy/)を御覧下さい。



