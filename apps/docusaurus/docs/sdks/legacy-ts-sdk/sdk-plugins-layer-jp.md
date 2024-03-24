---
title: "Plugins Layer (日本語)"
slug: "typescript-sdk-plugins-layer-jp"
---

プラグインは、タイプスクリプトSDKに追加して機能を拡張できるコンポーネントです。プラグインは、Aptosネットワーク上の一般的なアプリケーションをサポートする様構築されており、新機能の追加、アプリケーション操作の容易化、ユーザーエクスペリエンスのカスタマイズに使用できます。

## Aptosトークンクラス

[Aptosトークン](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/plugins/aptos_token.ts)クラスは[Aptosデジタル資産スタンダード](../../standards/digital-asset.md)と互換性があり、NFTコレクションとトークンを作成およびクエリするためのメソッドを提供します。オンチェーンでのトークンの作成、転送、変更、バーンを支援する書き込み方法を解説します。

AptosTokenクラスが支援する主な書き込メソッドは次のとおりです。

- コレクションの作成
- ミント
- ソウルバウンドのミント
- トークンバーン
- トークン転送を凍結する
- トークン転送の凍結を解除する
- トークンの説明を設定する
- トークン名の設定
- トークンURIの設定
- トークンプロパティの追加
- トークンプロパティの削除
- トークンプロパティの更新
- タイププロパティの追加
- タイプのプロパティを更新する
- トークンの所有権を移転する

## トークンクライアントクラス

[TokenClient](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/plugins/token_client.ts)クラスは[トークンV1スタンダード](../../standards/aptos-token.md)と互換性があり、NFTコレクションとトークンを作成およびクエリするためのメソッドを提供します。これは、 (1) オンチェーンでのトークンの作成、転送、変更、バーンを支援する書き込みメソッド、および (2)逆シリアル化を実行し、TypeScriptオブジェクトでデータを返す読み取りメソッドをカバーします。

トークンクライアントクラスがサポートする主な書き込みメソッドは以下の通りです。

- コレクションの作成
- トークンの作成
- トークンのオファー
- トークンの請求
- トークンを直接転送する
- オプトインでトークンを転送する
- トークンのプロパティを変更する
- 所有者または作成者がトークンをバーンする

オンチェーンデータをTypeScriptオブジェクトに逆シリアル化する主な読み取りメソッドは以下の通りです。

- コレクションデータの取得
- トークンデータの取得
- アカウントのトークンを取得する

## 代替資産クライアントクラス

[代替資産クライアント](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/plugins/fungible_asset_client.ts)クラスは[代替資産コンポーネント](../../standards/fungible-asset.md)と互換性があり、アカウント間で代替資産を転送したり、アカウント残高を確認したりするためのメソッドを提供します。

主な書き込み方法は以下の通りです。

- 送金
- 送金の生成

主な読み取り方法は次のとおりです。

- 初期残高を取得する

## コインクライアントクラス

[コインクライアント](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/plugins/coin_client.ts)クラスは、コインモジュールと対話してアカウント間でコインを送金し、アカウント残高を確認するメソッドを提供します。デフォルトでは `0x1::aptos_coin::AptosCoin`を送金しますが、`coinType`引数で別のコインを指定する事もできます。

主なメソッドは以下の通りです。

- 送金
- 残高確認

## フォーセットクライアントクラス

[フォーセットクライアント](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/plugins/faucet_client.ts)クラスは、Aptosフォーセットのための軽いラッパーで、Aptosアカウントに資金を供給する手段を提供します。このクラスは、Aptosアカウントで使うためのトークンを要求するリクエストメソッドを提供します。

主な読み取りメソッドは以下の通りです。

- アカウントへの資金投入

## ANSクライアントクラス

[ANSクライアント](https://github.com/aptos-labs/aptos-core/blob/main/ecosystem/typescript/sdk/src/plugins/ans_client.ts)クラスは、Aptosネットワーク上でANS名を作成し、ANSデータをクエリするためのメソッドを提供します。これは、(1) Aptos名を登録することでAptosネットワーク上で一意のIDの作成を支援する書き込みメソッド、(2) アドレスを使用してアカウントのANS名を取得する読み取りメソッド、およびANS名を使用してアカウントのアドレスを取得する読み取りメソッドを含みます。

SDKでサポートされている主な書き込みメソッドは以下の通りです。

- アプトス名をミントする
- 逆引きレジストリの初期化

主な読み取りメソッドは以下の通りです。

- 名前でアドレスを取得する
- アドレスでプライマリ名を取得する

## プラグインを構築する

開発者は、プラグインを作成してSDKの機能を拡張し、ユーザーにより良いエクスペリエンスを提供することもできます。プラグイン作成は、以下の手順で出来ます。

1. `src/plugins/`フォルダーの下に新しい`.ts`ファイルを作成し、`<pluginName>.ts`名前を付けます(例:`ans_client`)。
2. 同じ`pluginName`のクラスを作成し(例:`AnsClient`)、実装します。
3. `src/plugins/index.ts`ファイル (例:`export * from "./ans_client";`) からそのファイルをエクスポートします。

最終更新日2024年2月28日
