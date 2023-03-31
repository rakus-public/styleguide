# Routing And Middleware

## 目次

1. Gin のルーティング方法
2. サブモジュールの`injection`パッケージを使ったルーティング

## 1.Gin のルーティング方法

- このページは Go の Web フレームワーク[gin-gonic/gin](https://github.com/gin-gonic/gin)を使う前提で記述している
- `src/`直下のメインモジュールの`src/main.go`で Gin を起動し、ルーティング情報は`src/handlers.go`にまとめる
- `src/main.go`の中で各サブモジュールの`infrastructure`層の`RDBConnect`関数を読んで DB 接続を確立している
- 以下の例は認証コンテキスト(`src/authentication/`配下)とユーザーコンテキスト(`src/user/`配下)の 2 つのサブモジュールがある前提で記載

### src/main.go

```go
package main

import (
    "log"

    authdb "authentication/infrastructure"
    userdb "user/infrastructure"

    "github.com/gin-gonic/gin"
)

func main() {
    setupRDB() //DB接続を確立
    engine := gin.Default()

    RegisterHandlers(engine) //ルーティング情報を定義

    //8080ポートでアプリケーションを起動
    if err := engine.Run(":8080"); err != nil {
        log.Fatal(err)
    }
}

func setupRDB() {
    authDSN := "認証コンテキスト用DBへの接続情報"
    if err := authdb.RDBConnect(authDSN); err != nil {
        log.Fatal(err)
    }

    userDSN := "ユーザー管理コンテキスト用DBへの接続情報"
    if err := userdb.RDBConnect(userDSN); err != nil {
        log.Fatal(err)
    }
}
```

### src/handlers.go

```go
package main

import (
    "fmt"
    "net/http"

    "github.com/gin-gonic/gin"

    authInjection "authentication/injection"
    userInjection "user/injection"
)

func RegisterHandlers(e *gin.Engine) {
    root := e.Group("/api/v1")

    {
        RegisterAuthenticationHandlers(root)
        RegisterContractHandlers(root)
    }
}

func RegisterUserHandlers(root *gin.RouterGroup) {
    user := userInjection.InitializeUser()

    users := root.Group("/users")
    {
        users.GET("/profile", user.GetProfileHandler) //ユーザー情報参照API。GET /api/v1/users/profile
    }
}

func RegisterAuthenticationHandlers(root *gin.RouterGroup) {
    auth := authInjection.InitializeAuthController()

    session := root.Group("/session")
    {
        session.GET("/", auth.GetSessionHandler) //セッション確認API。GET /api/v1/session/
        session.POST("/", auth.PasswordAuthHandler) //パスワード認証API。POST /api/v1/session/
        session.DELETE("/", auth.LogoutHandler) //ログアウトAPI。DELETE /api/v1/session/
    }
}
```

## 2.サブモジュールの`injection`パッケージを使ったルーティング

- 各サブモジュールの`injection`パッケージで生成した UserInterface 層の struct のファクトリ関数を、Gin のルーティングに使用する
  - `injection`パッケージにファクトリ関数を生成する方法は[Dependency Injection](./dependency-injection.md)を参照
  - ファクトリ関数は`type gin.HandlerFunc func(*gin.Context)`を満たす必要がある

```go
session.POST("/", auth.PasswordAuthHandler)
```
