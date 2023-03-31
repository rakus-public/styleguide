# Goroutine Job

## 目次

1. Goroutine を用いた定期実行バッチ(Job)について
2. 実装方法

## 1.Goroutine を用いた定期実行バッチ(Job)について

- 特定のインターバルで定期的に実行したいバッチ処理を、Goroutine を用いて実行し続ける実装方法をこちらの資料に記載する
- この定期実行処理を「Job」と呼ぶ
- Job の基本的な実装方針と制約は以下の通り
  - Job を実行するコンテナは 1 つに限定する
  - 1 つの種類の Job に対して、Job 名・Job の本体となる関数・Job の実行周期を登録する
  - Job が実行されるたびに、Job に登録されている関数を新たな Goroutine として呼び出す
  - Job 実行タイミングで前回実行された Goroutine がまだ実行中の場合は、実行をスキップする
  - 特定のシグナル(SIGTERM, SIGINT)を受信した場合は、実行中 Job に通知してバッチ処理を安全に停止するよう実装する

## 2.実装方法

- `src/job/`配下におかれる`main.go`を実行することで Job を起動する
- サンプルとなる Job
  - メール送信 Job
    - メール送信予約データを DB から全件参照して、1 件ずつメール送信を実行する
  - Job 名: `SendMailBatch`
  - Goroutine として呼び出す関数: `SendAll`
    - `mail/batch` パッケージにある`SendMailBatch`型が持つメソッド
    - `SendMailReservationBatch`は`mail/injection`パッケージの`InitializeSendMailBatch()`関数で初期化できる前提
  - 5 分ごとに起動
- ログ出力や Job に関係ない部分の実装は割愛・簡略化

### `src/job/main.go`

```go
package main

import (
    "sample-project/job/jobs"
)

func main() {
    // すべてのjobを起動
    jobs.Start()
}
```

### `src/job/jobs/job.go`

```go
package jobs

import (
 "context"
 "fmt"
 "math/rand"
 "sync"
 "time"
)

type JobName string

func (n JobName) String() string {
    return string(n)
}

type Job struct {
    Name    JobName
    d       time.Duration
    handler func(context.Context) []error
    once    *sync.Once
    ctx     context.Context
    done    chan bool
}

func NewJob(
    name string,
    d time.Duration,
    handler func(context.Context) []error,
    ctx context.Context,
) *Job {
    return &Job{
        Name:    JobName(name),
        d:       d,
        handler: handler,
        once:    &sync.Once{},
        ctx:     ctx,
        done:    make(chan bool, 1),
    }
}

func (j *Job) Start() {
    go func(j *Job) {
        rand.Seed(time.Now().UnixNano())
        initD := rand.Int63n(int64(j.d) * 2)
        ticker := time.NewTicker(time.Duration(initD))
        for {
            select {
            case t := <-ticker.C:
                go j.once.Do(func() { j.Exec(t) })
                ticker.Reset(j.d)
            case <-j.ctx.Done():
                j.Stop()
            case <-j.done:
                ticker.Stop()
                return
            }
        }
    }(j)
}

func (j *Job) Exec(t time.Time) {
    var errs []error
    defer func() {
        if rec := recover(); rec != nil {
            errs = append(errs, fmt.Errorf("[JOB EXEC ERROR] %s", rec))
        }

        j.once = &sync.Once{}
    }()

    errs = j.handler(ctx)
}

func (j *Job) Stop() {
    j.done <- true
}

```

### `src/job/jobs/jobs.go`

```go
package jobs

import (
    "context"
    mail "mail/injection"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func Start() {
    ctx, cancel := context.WithCancel(context.Background())
    defer func() {
        cancel()
        time.Sleep(10 * time.Second)
    }()

    jobs := &Jobs{}
    jobs.Register(
        NewJob("SendMailBatch", time.Minute*5, mail.InitializeSendMailBatch().SendAll, ctx),
    )
    jobs.Start()

    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
    <-sigs
}

type Jobs struct {
    js []*Job
}

func (jobs *Jobs) Register(js ...*Job) {
    jobs.js = append(jobs.js, js...)
}

func (jobs *Jobs) Start() {
    for _, j := range jobs.js {
        j.Start()
    }
}
```

### `src/mail/batch/send_mail_batch.go`

```go
package batch

import (
    "context"
    "fmt"
    "mail/application"
    "mail/domain"
)

type SendMailBatch struct {
    repo    domain.SendMailReservationRepository
    service application.SendMailReservationService
}

func NewSendMailBatch(
    repo domain.SendMailReservationRepository,
    service application.SendMailReservationService,
) *SendMailBatch {
    return &SendMailBatch{
        repo:    repo,
        service: service,
    }
}

func (b *SendMailBatch) SendAll(ctx context.Context) []error {
    var errs []error
    rs, err := b.repo.FindAll(ctx)

    if err != nil {
        errs = append(errs, fmt.Errorf("failed to find reservation mail list. err:%s", err))
        return errs
    }

    for i, r := range rs {
        select {
        case <-ctx.Done():
            return errs
        default:
            es := b.service.Send(ctx, *r)
            if len(es) > 0 {
                errs = append(errs, fmt.Errorf("[error]Index:%d,ReservationID:%s,Errors:%s", i+1, r.ReservationID, es))
            }
        }
    }

    return errs
}
```
