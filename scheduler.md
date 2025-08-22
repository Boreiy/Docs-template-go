## Планирование фоновых задач (scheduler)

Документ — самодостаточный. Покрывает: периодические задачи через `github.com/robfig/cron/v3` и простой `time.Ticker`. Код размещайте в `internal/adapter/scheduler`, запуск — из `cmd/bot/main.go`.

### 1) Библиотека cron

```go
package scheduler

import (
    "context"
    "log"

    "github.com/robfig/cron/v3"
)

func Start(ctx context.Context) (*cron.Cron, error) {
    c := cron.New()
    _, err := c.AddFunc("@every 1h", func() {
        log.Println("clean temp data") // example job
    })
    if err != nil { return nil, err }
    go func() {
        <-ctx.Done()
        c.Stop()
    }()
    c.Start()
    return c, nil
}
```

### 2) Простой тикер

```go
func startTicker(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Minute)
    go func() {
        for {
            select {
            case <-ticker.C:
                log.Println("sync stats") // periodic work
            case <-ctx.Done():
                ticker.Stop()
                return
            }
        }
    }()
}
```

Чек-лист
- Останавливайте scheduler при завершении контекста.
- Делайте задачи идемпотентными.
- Для сложных сценариев используйте `robfig/cron/v3`, для простых — `time.Ticker`.
