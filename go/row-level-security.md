# Row Level Security

## 目次

1. PostgreSQL の Row Level Security について
2. テーブル定義方法
3. Gorm での Row Level Security の実装方法

## 1.PostgreSQL の Row Level Security について

- PostgreSQL で設定できる Row Level Security に関する実装方法を紹介する
- Gorm で実現する前提で記載

### Row Level Security とは

- [公式ドキュメント](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- 複数のユーザーや企業で共通で使われるテーブルに対して、誤って他のユーザーや企業のデータを参照・更新できないように行レベルでガードをかける機構
  - テーブル単位で有効化して、ガードのポリシーを定義できる
  - トランザクション開始後にしか使用できない
- RLS と省略されることが多い

## 2. テーブル定義方法

- `user`テーブルを宣言
  - `ALTER TABLE user ENABLE ROW LEVEL SECURITY;`で`user`テーブルに対して Row Level Security を有効化
  - `user_isolation_policy`で、セッション変数の`app.user_id`の値が`user`テーブルの`id`カラムと一致していないレコードの操作をできないよう設定
  - `admin_policy`で、セッション変数の`app.user_id`の値が`'0'`の場合は全レコードに対して操作ができるように設定(Row Level Security の一時的な無効化)
    - ユーザーをまたいで複数のレコードを参照する必要がある場合に使われるポリシー
    - `id`カラムに絶対に`'0'`という値が入ってこない前提の設定

```sql
CREATE TABLE
    IF NOT EXISTS user (
        id VARCHAR(255) PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        status SMALLINT NOT NULL DEFAULT 1,
        created_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
        updated_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
    );

ALTER TABLE user ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_isolation_policy ON user USING (
    id:: TEXT = current_setting('app.user_id'):: TEXT
);

CREATE POLICY admin_policy ON user USING (
    '0':: TEXT = current_setting('app.user_id'):: TEXT
);
```

## 3. Gorm での Row Level Security の実装方法

- トランザクション開始後に使用される前提の実装

### ユーザー ID をセッション変数に設定する例

```go
func (r *SampleRepository) SetRLS(ctx context.Context, uID domain.UserID) error {
    // Gorm のパースを使用すると Syntax error を吐いてしまって実現できなかったため fmt を使用している
    // SQL インジェクションの可能性を自前のバリデーションで潰す必要があるので注意
    sql := fmt.Sprintf("SET LOCAL app.current_user_id = '%s'", uID)
    if err := r.conn(ctx).Exec(sql).Error; err != nil {
        return err
    }

    return nil
}
```

### セッション変数に'0'を指定して一時的に Row Level Security を無効化する例

```go
func (r *SampleRepository) DisableRLS(ctx context.Context) error {
    if err := r.conn(ctx).Exec("SET LOCAL app.current_user_id = '0'").Error; err != nil {
        return err
    }

    return nil
}
```
