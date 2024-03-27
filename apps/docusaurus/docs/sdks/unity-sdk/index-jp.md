---
title: "日本語(in japanese)"
slug: "unity-sdk-in-japanese"
---

# Aptos Unity SDK

[Aptos Unity SDK](https://github.com/aptos-labs/Aptos-Unity-SDK)は[Aptos SDK](../index.md)の.NET実装であり、.NET Standard 2.0 と .NET 4.x for Unityと互換性があります。この SDKの目標は、UnityゲームエンジンとAptosブロックチェーンインフラストラクチャを使用して、開発者がマルチプラットフォームアプリケーション(モバイル、デスクトップ、Web、VR)を構築するためのツールセットを提供することです。

Aptos Unity SDK によってゲーム開発者に提供されるすべての機能については、[Aptos LabsがUnity開発者向けの新しいSDKでWeb3をゲームに導入する](https://medium.com/aptoslabs/aptos-labs-brings-web3-to-gaming-with-its-new-sdk-for-unity-developers-e6544bdf9ba9)と言う記事と、Unity SDK READMEの[技術的な詳細](https://github.com/aptos-labs/Aptos-Unity-SDK#technical-details)セクションを参照してください。 

## ユーザーフロー

Aptos Unity SDKは、以下のユースケースをサポートしています。

- ユーザーが電子メールでゲームにログインできる進行性オンボーディングフロー。このフローではトランザクションがプロキシされ、Aptosは分散型のキーシステムを使用します。その後希望するならユーザーは、完全なカストディアルウォレットにオンボードできます。
- _ゲーム内の非カストディアルウォレットの統合_。ゲーム開発者は、ユーザーがゲーム内で完全な非カストディアルウォレットを作成できるようにするオプションがあります。
- _ゲーム外の非カストディアルウォレットの統合_。ゲーム開発者がユーザーをゲーム内のデスクトップウォレットまたはモバイルウォレットに接続するか、親ウォレットからバーナーウォレット(使い捨てウォレット)をシームレスに作成出来る様にします。

## 前提条件

### サポートされているUnityバージョン

| サポートされているバージョン: | テスト済み |
| ------------------------ | -------- |
| 2021.3.x                 | ✅       |
| 2022.2.x                 | ✅       |

| Windows | macOS | iOS | Android | WebGL |
| ------- | ----- | --- | ------- | ----- |
| ✅      | ✅    | ✅  | ✅      | ✅    |

### 依存関係

> Unity 2021.xx以降、Newtonsoft Jsonは一般的な依存関係です。Unity の以前のバージョンでは、Newtonsoftをインストールする必要があります。

- [Chaos.NaCl.スタンダード](https://www.nuget.org/packages/Chaos.NaCl.Standard/)
- Microsoft.Extensions.Logging.Abstractions.1.0.0 — NBitcoin.7.0.22 で必要
- Newtonsoft.Json
- NBitcoin.7.0.22
- [ポータブル.BouncyCastle](https://www.nuget.org/packages/Portable.BouncyCastle)
- Zxing

## Unity SDKをインストールする

 Unity SDKは、弊社`unitypackage`または[Unity パッケージマネージャ](https://docs.unity3d.com/Manual/Packages.html)を通じてインストールできます。

### `unitypackage`のインストール方法

1. Unityを起動します。
2. 最新の`Aptos.Unity.unitypackage`ファイルを [Unity Asset Store](https://assetstore.unity.com/packages/decentralization/aptos-sdk-244713)からダウンロードします。
3. **資産** → **パッケージのインポート** → **カスタムパッケージ**をクリックし、ダウンロードしたファイルを選択します。


### Unityパッケージマネージャのインストール方法

1. [Unityパッケージマネージャ](https://docs.unity3d.com/Manual/upm-ui.html)ウィンドウを開きます。
2. 上部ステータスバーにある追加 **+** ボタンをクリックします。
3. ドロップダウンメニューで _git URLからパッケージを追加_ を選択します。
4. URL *https://github.com/aptos-labs/Aptos-Unity-SDK.git* を入力し、**追加**をクリックします。