# Multi Module Project

## 目次

1. 概要
2. Go Workspace 機能による Multi Module の解決

## 1.概要

- Go Module を複数持つモジュラーモノリス構成のプロジェクトにおいて、各モジュール間の名前解決を Go の Workspace 機能を使って実現する

## 2.Go Workspace 機能による Multi Module の解決

- `go.work`ファイルに羅列した Module であれば、各 Module の`go.mod`ファイルに依存関係を記載しなくても名前解決が可能
- この形をとれば、サブモジュール間の処理呼び出しは直接メソッドを実行することで可能となる

### go.work の記載例

```text
go 1.20

use (
 ./src
 ./src/authentication
 ./src/user
)
```
