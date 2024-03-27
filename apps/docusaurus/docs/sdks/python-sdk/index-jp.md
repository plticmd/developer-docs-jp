---
title: "日本語(in japanese)"
slug: "python-sdk-in-japanese"
---

# Aptos Python SDK
Aptosは、軽くメンテナンスされた公式のPythonSDKを提供します。 これは、[PyPi](https://pypi.org/project/aptos-sdk/)で[AptosコアGitHubリポジトリ](https://github.com/aptos-labs/aptos-core/tree/main/ecosystem/python/sdk)のソースコードと共に利用可能です。Python SDKの機能の多くは[Typescript SDK](../ts-sdk/index.md)を複製しています。Python SDKの主な目的は、Python開発者がすぐにAptosチュートリアルの付属物としてのAptosに精通する事を支援する事です。

## Python SDKをインストールする。

Python SDKは`pip`を通して、ソースから、または埋め込みで簡単にインストールできます。

### pipでインストールする。

`pip`経由でインストールするには、

```bash
pip3 install aptos-sdk
```

`aptos-sdk`は、ローカルサイトのパッケージディレクトリにインストールされます。たとえば、macOSでは、`~/Library/Python/3.8/lib/python/site-packages/aptos_sdk`ディレクトリで`aptos-sdk`が見つかります。

### ソースコードからインストールする。

ソースからインストールするには、

```bash
git clone https://github.com/aptos-labs/aptos-core
pip3 install aptos-core/ecosystem/python/sdk --user
```

### 埋め込みによるインストール

既存のPythonプロジェクトにPython SDKを埋め込むには、

```
cd /path/to/python/project
cp -r /path/to/aptos-core/ecosystem/python/sdk/aptos-sdk aptos-sdk
```

## Python SDKを使う

Python SDK の使用方法を示すコード例については、[開発者チュートリアル](../../tutorials/index.md)をご覧下さい。 
