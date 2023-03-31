# Transaction

## 目次

1. DB トランザクションのライフサイクルについて
2. トランザクション管理の実装例

## 1.DB トランザクションのライフサイクルについて

- DB トランザクションを開始してから Commit/Rollback するまでを DB トランザクションのライフサイクルとする
- DB トランザクションのライフサイクルは基本的には Application 層のアプリケーションサービスのメソッド単位とする

## 2.トランザクションの実装例

- 実装例として使うコードは[認証 API のサンプルコード](./auth-api-sample-code.md)から抜粋

### Application 層のトランザクション管理オブジェクトの interface 定義

- Application 層にトランザクション管理用オブジェクトの interface を定義
  - `context.Context`とアプリケーションサービスで行う処理を記載した関数を引数として渡すメソッドを具備する
  - DB 接続に関する情報は Infrastructure 層の範疇なので、Application 層の実装には記載しない

```go
type Transaction interface {
    Transaction(ctx context.Context, fn func(ctx context.Context) error) error
}
```

### Infrastructure 層のトランザクション管理オブジェクトの struct 定義

- Application 層の`Transaction`を実装した struct を Infrastructure 層に定義
  - 以下の例は`DB`という struct が Application 層の`Transaction`を実装している
- `tx := d.Begin()`でトランザクションを開始
- `txCtx := context.WithValue(ctx, txKey, tx)`で開始したトランザクションオブジェクトを保持した Context を新規作成
  - その Context を`fn`の引数として渡すことで、アプリケーション内で同一トランザクションを引き回すことを実現する
- `fn`が`error`を返却した場合は`tx.Rollback()`で Rollback
- `fn`が`nil`を返却した場合は`tx.Commit()`で Commit
- 途中で panic が発生した場合は defer で補足して Rollback

```go
type DB struct {
    *gorm.DB
}

func (d *DB) Transaction(ctx context.Context, fn func(ctx context.Context) error) (err error) {
    tx := d.Begin()
    defer func(tx *gorm.DB) {
        if r := recover(); r != nil {
            tx.Rollback()
            err = errors.New(fmt.Sprintf("recovered from panic, err: %s", r))
        }
    }(tx)

    txCtx := context.WithValue(ctx, txKey, tx)
    if err = fn(txCtx); err != nil {
        tx.Rollback()
        return err
    }

    tx.Commit()

    return nil
}
```

### Application 層のメソッドの実装方法

- 以下の例では`authService`というアプリケーションの`Auth`メソッド内で DB トランザクションのライフサイクルが完結している
  - `s.repo.PasswordAuth(ctx, uID, password)`でリポジトリに Context を渡している
    - この`ctx`はトランザクションオブジェクトを保持した Context になっている

```go
type authService struct {
    repo domain.AuthRepository
    tx   Transaction
}

func (s *authService) Auth(ctx context.Context, uID domain.UserID, password domain.Password) error {
    err := s.tx.Transaction(ctx, func(ctx context.Context) error {
        return s.repo.PasswordAuth(ctx, uID, password)
    })

    return err
}
```

### Infrastructure 層のリポジトリの実装方法

- Infrastructure 層のリポジトリ実装
  - `conn`メソッド
    - 引数として渡された`ctx`から、開始済みのトランザクションオブジェクト(`*gorm.DB`型)が取得できた場合はそのオブジェクトをそのまま使用
    - 取得できない場合は新たに`*gorm.DB`を生成
  - `PasswordAuth`メソッド
    - `conn`メソッドで`*gorm.DB`型のオブジェクトを取得してから DB 操作を行う

```go
type AuthRepository struct {
    db *DB
}

func (r *AuthRepository) conn(ctx context.Context) *gorm.DB {
    tx, ok := ctx.Value(txKey).(*gorm.DB)
    if ok && tx != nil {
        return tx
    }

    return r.db.Session(&gorm.Session{})
}

func (r *AuthRepository) PasswordAuth(ctx context.Context, uID domain.UserID, password domain.Password) error {
    //*gorm.DBオブジェクトを取得
    c := r.conn(ctx)

    //細かい処理は割愛
    return nil
}
```
