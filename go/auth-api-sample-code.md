# Auth API Sample Code

## 概要

- このページは、複数の資料で共通的に使う認証 API のサンプルコードを記載しています。

### `authentication/domain`パッケージ

#### src/authentication/domain/values.go

```go
package domain

type UserID string
type Password string
```

#### src/authentication/domain/repositories.go

```go
package domain

import "context"

type AuthRepository interface {
    PasswordAuth(context.Context, UserID, Password) error
}
```

### `authentication/userintarface`パッケージ

#### src/authentication/userinterface/auth_controller.go

```go
package userinterface

import (
    "authentication/application"
    "authentication/domain"

    "net/http"
    "github.com/gin-gonic/gin"
)

type AuthController struct {
    service application.AuthService
}

func NewAuthController(service application.AuthService) *AuthController {
    return &AuthController{service: service}
}

type AuthRequest struct {
    ID       string `json:"id"`
    Password string `json:"password"`
}

func (c *AuthController) PasswordAuthHandler(ctx *gin.Context) {
    var req AuthRequest
    if err := ctx.ShouldBindJSON(&req); err != nil {
        ctx.Status(http.StatusUnauthorized)
        return
    }

    if err := c.service.Auth(ctx, domain.UserID(req.ID), domain.Password(req.Password)); err != nil {
        ctx.Status(http.StatusUnauthorized)
        return
    }
    ctx.Status(http.StatusOK)
}
```

### `authentication/application`パッケージ

#### src/authentication/application/services.go

```go
package application

import (
    "authentication/domain"
    "context"
)

type AuthService interface {
    Auth(context.Context, domain.UserID, domain.Password) error
}
```

#### src/authentication/application/auth_service.go

```go
package application

import (
    "authentication/domain"
    "context"
)

type authService struct {
    repo domain.AuthRepository
    tx   Transaction
}

func NewAuthService(repo domain.AuthRepository, tx Transaction) AuthService {
    return &authService{
        repo: repo,
        tx:   tx,
    }
}

var _ AuthService = new(authService)

func (s *authService) Auth(ctx context.Context, uID domain.UserID, password domain.Password) error {
    err := s.tx.Transaction(ctx, func(ctx context.Context) error {
        return s.repo.PasswordAuth(ctx, uID, password)
    })

    return err
}
```

#### src/authentication/application/transaction.go

```go
package application

import "context"

type Transaction interface {
    Transaction(ctx context.Context, fn func(ctx context.Context) error) error
}
```

### `authentication/infrastructure`パッケージ

#### src/infrastructure/database.go

```go
package infrastructure

import (
    "errors"
    "context"
    "fmt"

    "gorm.io/gorm"
)

type TxKey string

const txKey TxKey = "auth_tx"

type DB struct {
    *gorm.DB
}

var d *DB

// アプリケーション起動時に呼ばれる関数
func RDBConnect(dsn string) error {
    var db *gorm.DB = createDB() // *gorm.DBのインスタンスを生成する関数。DB接続設定等は割愛

    d = &DB{db}
    return nil
}

func GetDB() *DB {
    return d
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

#### src/infrastructure/auth_repository.go

```go
package infrastructure

import (
    "authentication/domain"
    "context"

    "gorm.io/gorm"
)

type AuthRepository struct {
    db *DB
}

var _ domain.AuthRepository = new(AuthRepository)

func NewAuthRepository() *AuthRepository {
    return &AuthRepository{GetDB()}
}

func (r *AuthRepository) conn(ctx context.Context) *gorm.DB {
    tx, ok := ctx.Value(txKey).(*gorm.DB)
    if ok && tx != nil {
        return tx
    }

    return r.db.Session(&gorm.Session{})
}

func (r *AuthRepository) PasswordAuth(ctx context.Context, uID domain.UserID, password domain.Password) error {
    // パスワード認証の処理は割愛
    return nil
}
```
