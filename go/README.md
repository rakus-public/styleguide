# Go Style Guide

## 目次

1. 当資料の概要
2. 使用するライブラリ/FW/Middleware
3. 推奨する学習順序

## 1.当資料の概要

- こちらは [株式会社ラクス](https://www.rakus.co.jp/) の SRE 課で担当するプロジェクトにおける、Go 言語での WebAPI 実装に関するノウハウをまとめた資料です。
- 以下のようなアーキテクチャを前提にしています。
  - 複数の Go Module をモノリスで組み合わせたモジュラーモノリス構成
  - 各 Go Module の内部はレイヤードアーキテクチャを採用
- 当資料に記載されたソースコードは動作を保証するものではありません。ご理解いただいた上、自己責任においてご利用ください。

## 2. 使用するライブラリ/FW/Middleware

当資料で使用しているライブラリ/FW は以下です。

- Web フレームワーク: [gin-gonic/gin](https://github.com/gin-gonic/gin)
- ORM: [GORM](https://gorm.io/ja_JP/docs/index.html)
- DI ツール: [google/wire](https://github.com/google/wire)

また、[Row Level Security](./row-level-security.md)ページは PostgreSQL を使用する前提で記述しています。

## 3.推奨する学習順序

各ページは他のページでの知識が前提になっている場合があるため、学習は以下の順で実施することを推奨します。

1. [Basics](./basics.md): DDD の基本説明、ディレクトリ構成等、当資料を参照するにあたって認識いただきたい前提知識を記載しています。
2. [Multi Module Project](./multi-module-project.md): モジュラーモノリス構成のアプリに対して、Go の Workspace 機能を用いて名前解決する方法を記載しています。
3. [Layered Architecture](./layered-architecture.md): 単一の Go Module はレイヤードアーキテクチャの構成を取っています。各レイヤーの説明と基本的な実装方法を記載しています。
4. [Dependency Injection](./dependency-injection.md): 各レイヤーをまたいだオブジェクトのファクトリ関数を`google/wire`で生成する手法を記載しています。
5. [Transaction](./transaction.md): DB トランザクションの制御の実装方法を記載しています。
6. [Routing And Middleware](./routing-and-middleware.md): Web フレームワーク Gin を用いたルーティングの実装方法を記載しています。
7. [Row Level Security](./row-level-security.md): PostgreSQL の Row Level Security を、ORM の Gorm を用いて実装する方法を記載しています。
8. [Goroutine Job](./goroutine-job.md): Goroutine を使ったバッチ処理の実装方法を記載しています。
9. [Unit Test](./unit-test.md): 単体テストの基本的な記述方法を記載しています。
