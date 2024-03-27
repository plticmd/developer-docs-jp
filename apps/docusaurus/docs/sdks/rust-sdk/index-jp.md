---
title: "日本語(in japanese)"
slug: "rust-sdk-in-japanese"
---

# Aptos Rust SDK

## Rust SDKをインストールする

Aptosは[Aptos-core GitHub](https://github.com/aptos-labs/aptos-core/tree/main/sdk)リポジトリで公式のRust SDKを提供しています。Rust SDKを使用するには、以下の様に、依存関係とパッチをgitリポジトリの`Cargo.toml`に直接追加します。 

```toml
[dependencies]
aptos-sdk = { git = "https://github.com/aptos-labs/aptos-core", branch = "devnet" }

[patch.crates-io]
merlin = { git = "https://github.com/aptos-labs/merlin" }
x25519-dalek = { git = "https://github.com/aptos-labs/x25519-dalek", branch = "zeroize_v1" }
```

`.cargo/config.toml`ファイルも以下の内容で作成する必要があります。 

```toml
[build]
rustflags = ["--cfg", "tokio_unstable"]
```

公式Rust SDKのソースコードは、[aptos-core GitHubリポジトリ](https://github.com/aptos-labs/aptos-core/tree/main/sdk)
で入手できます。

## Rust SDKを使う

Rust SDK の使用方法を示すコード例については、[開発者チュートリアル](../../tutorials/index.md)を参照してください。