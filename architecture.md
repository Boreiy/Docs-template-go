## Архитектура и структура проекта (Go, Clean Architecture)

Документ — практический и самодостаточный. Покрывает состав компонентов, правила импортов и зависимостей, запуск, логирование, HTTP/БД, Telegram-поток обновлений, идемпотентность и вебхуки, с готовыми минимальными примерами кода.

### 1) Стек и ключевые зависимости

- Язык: Go 1.22+
- Telegram: `github.com/go-telegram-bot-api/telegram-bot-api/v5` (tgbotapi)
- Веб/API: Gin (`github.com/gin-gonic/gin`) + опционально Swagger (`github.com/swaggo/swag`, `github.com/swaggo/gin-swagger`)
- БД: PostgreSQL, драйвер `github.com/jackc/pgx/v5/pgxpool`
- Миграции: `github.com/golang-migrate/migrate/v4`
- Валидация: `github.com/go-playground/validator/v10`
- Логирование: стандартный `log/slog`, для dev — `github.com/lmittmann/tint`, ротация — `gopkg.in/natefinch/lumberjack.v2`
- HTTP-клиент: `net/http` (кастомный Transport и таймауты) или `github.com/go-resty/resty/v2`

Примечание про tgbotapi: библиотека поддерживает long polling и webhooks. Для dev используется polling (проще, не требует публичного HTTPS). В prod рекомендуется вебхук на `Gin` с проверкой секретного заголовка.

### 2) Структура каталогов (Clean Architecture)

```
yourbot/
├── cmd/
│   └── bot/
│       └── main.go              # composition root: загрузка конфигов, DI, запуск процессов
├── internal/
│   ├── app/                     # сборка зависимостей (wire/fx/ручной DI), запуск
│   ├── config/                  # парсинг env, конфиг-структуры
│   ├── domain/                  # сущности и доменные интерфейсы (репозитории, сервисы)
│   ├── usecase/                 # сценарии приложения, портовые интерфейсы
│   ├── adapter/
│   │   ├── db/postgres/         # реализация репозиториев на pgx
│   │   ├── external/            # внешние интеграции (HTTP/LLM и пр.)
│   │   └── telegram/            # bot router/handlers, long polling/webhook
│   ├── platform/
│   │   ├── logger/              # slog+tint+rotator
│   │   ├── http/                # http client factory (timeouts, transport)
│   │   └── pg/                  # pgxpool init
│   └── shared/                  # общие утилиты, ошибки
├── migrations/                  # SQL миграции (golang-migrate)
├── dev-docs/                    # документация разработчика (этот файл и др.)
└── go.mod / go.sum
```

Правила импортов:
- `domain` не знает об инфраструктуре.
- `usecase` зависит только от `domain` (интерфейсы портов), не от адаптеров.
- `adapter/*` реализует порты (`db`, `external`, `telegram`), может зависеть от `platform`.
- `platform` — технологические детали: логгер, http-клиент, pgxpool.
- `cmd/*` собирает всё вместе (composition root).

### 3) Конфигурация и переменные окружения

Рекомендуемый минимум env:

Для LLM задаются две разные модели: **черновая** для быстрых черновиков и **финальная** для итоговых ответов.

```
ENV=dev|prod
HTTP_ADDR=:8080
TELEGRAM_BOT_TOKEN=...                # обязательный
TELEGRAM_WEBHOOK_URL=https://...      # прод, при webhooks
TELEGRAM_WEBHOOK_SECRET=...           # прод, проверяется по заголовку
OPENAI_API_KEY=...                    # ключ OpenAI
OPENAI_BASE_URL=https://api.openai.com/v1 # базовый URL API
OPENAI_MODEL_DRAFT=gpt-4o-mini-2024-07-18  # модель черновых ответов (Config.OpenAI.ModelDraft)
OPENAI_MODEL_FINAL=gpt-4o-2024-11-20       # модель финальных ответов (Config.OpenAI.ModelFinal)
DATABASE_URL=postgres://user:pass@host:5432/db?sslmode=disable
LOG_LEVEL=info|debug|warn|error
LOG_FILE=./logs/app.log               # если задан — JSON+ротация
```

Используем датированные идентификаторы моделей (`*-YYYY-MM-DD`), чтобы фиксировать поведение и сохранять воспроизводимость. Если указать модель без даты (`gpt-4o` или `gpt-4o-mini`), будет применяться последняя ревизия, что может привести к изменениям в ответах.

Структура конфига (пример):

```go
type Config struct {
    Env   string `validate:"required,oneof=dev prod"`
    HTTP  struct { Addr string `validate:"required"` }
    Telegram struct {
        Token  string `validate:"required"`
        WebhookURL    string
        WebhookSecret string
    }
    DB struct { URL string `validate:"required"` }
    OpenAI struct {
        APIKey     string `validate:"required"`
        BaseURL    string
        ModelDraft string `validate:"required"` // LLM for draft responses
        ModelFinal string `validate:"required"` // LLM for final responses
    }
    Log struct {
        Level string `validate:"required,oneof=debug info warn error"`
        File  string
    }
}
```

### 4) Логирование (slog, tint, ротация)

- Dev: цветной вывод `tint` (краткий формат).
- Prod: `slog.JSONHandler` + `lumberjack` с ротацией.
- В каждую запись добавлять: `request_id`, `chat_id`, `update_id` (если есть), `user_id`.

```go
package platform

import (
    "log/slog"
    "os"
    "strings"

    "github.com/lmittmann/tint"
    "gopkg.in/natefinch/lumberjack.v2"
)

type Config struct {
    Log struct {
        Level string
        File  string
    }
}

func levelFromString(s string) slog.Level {
    switch strings.ToLower(s) {
    case "debug":
        return slog.LevelDebug
    case "info", "":
        return slog.LevelInfo
    case "warn", "warning":
        return slog.LevelWarn
    case "error":
        return slog.LevelError
    default:
        return slog.LevelInfo
    }
}

func NewLogger(cfg Config) *slog.Logger {
    var h slog.Handler
    if cfg.Log.File != "" { // JSON + rotation
        lj := &lumberjack.Logger{Filename: cfg.Log.File, MaxSize: 50, MaxBackups: 10, MaxAge: 30, Compress: true}
        h = slog.NewJSONHandler(lj, &slog.HandlerOptions{Level: levelFromString(cfg.Log.Level)})
    } else { // colorful dev output
        h = tint.NewHandler(os.Stdout, &tint.Options{Level: levelFromString(cfg.Log.Level)})
    }
    return slog.New(h)
}
```

### 5) HTTP-клиент (таймауты, пулы)

Не использовать `http.DefaultClient` в проде. Создаём клиент с полным набором таймаутов и увеличенными пулами для целевых хостов.

```go
import (
    "net/http"
    "time"
)

func NewHTTPClient() *http.Client {
    tr := http.DefaultTransport.(*http.Transport).Clone()
    tr.MaxIdleConns = 100
    tr.MaxConnsPerHost = 100
    tr.MaxIdleConnsPerHost = 100
    tr.IdleConnTimeout = 90 * time.Second
    tr.TLSHandshakeTimeout = 10 * time.Second
    tr.ResponseHeaderTimeout = 10 * time.Second
    tr.ExpectContinueTimeout = 1 * time.Second

    return &http.Client{
        Timeout:   15 * time.Second, // overall request SLA
        Transport: tr,
    }
}
```

Альтернатива: Resty с ретраями и backoff (по проектной необходимости).

### 6) PostgreSQL (pgxpool) и миграции

Инициализация пула с проверкой соединения и health-таймаутами.

```go
func NewPGPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
    cfg, err := pgxpool.ParseConfig(dsn)
    if err != nil { return nil, err }
    cfg.MaxConns = 20
    cfg.MinConns = 2
    cfg.HealthCheckPeriod = 30 * time.Second
    cfg.MaxConnLifetime = time.Hour
    cfg.MaxConnIdleTime = 10 * time.Minute
    pool, err := pgxpool.NewWithConfig(ctx, cfg)
    if err != nil { return nil, err }
    if err := pool.Ping(ctx); err != nil { pool.Close(); return nil, err }
    return pool, nil
}
```

Автоприменение миграций при старте (dev) и по флагу (prod):

```go
import (
    migrate "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func ApplyMigrations(dsn string) error {
    m, err := migrate.New("file://migrations", dsn)
    if err != nil { return err }
    defer m.Close()
    if err := m.Up(); err != nil && err != migrate.ErrNoChange { return err }
    return nil
}
```

#### 6.1 TxRunner — обёртка для транзакций

TxRunner — порт домена для выполнения сценариев в одной транзакции. Реализация на pgx кладёт `pgx.Tx` в `context.Context`, чтобы репозитории использовали её без явной передачи объекта.

```go
package postgres

import (
    "context"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

// TxRunner runs fn inside a transaction
type TxRunner struct{ pool *pgxpool.Pool }

func NewTxRunner(pool *pgxpool.Pool) *TxRunner { return &TxRunner{pool: pool} }

type txKey struct{}

func (r *TxRunner) WithinTx(ctx context.Context, fn func(ctx context.Context) error) error {
    tx, err := r.pool.Begin(ctx)
    if err != nil { return err }
    ctx = context.WithValue(ctx, txKey{}, tx)
    if err := fn(ctx); err != nil {
        _ = tx.Rollback(ctx)
        return err
    }
    return tx.Commit(ctx)
}

func TxFromCtx(ctx context.Context) pgx.Tx {
    tx, _ := ctx.Value(txKey{}).(pgx.Tx)
    return tx
}
```

Пример использования в usecase:

```go
package usecase

import (
    "context"

    "yourbot/internal/domain"
)

type MenuService struct {
    Menus domain.MenuRepo
    Tx    domain.TxRunner
}

func (s *MenuService) DraftMenu(ctx context.Context, userID domain.UserID) error {
    return s.Tx.WithinTx(ctx, func(ctx context.Context) error {
        _, err := s.Menus.CreateDraft(ctx, userID, 7)
        return err
    })
}
```

### 7) Telegram: polling, webhook, идемпотентность

#### 7.1 Long polling (dev)

```go
func runPolling(ctx context.Context, log *slog.Logger, bot *tgbotapi.BotAPI, updates tgbotapi.UpdatesChannel) {
    for {
        select {
        case <-ctx.Done():
            return
        case upd := <-updates:
            // идемпотентность при polling достигается подтверждением offset
            go handleUpdate(ctx, log, bot, upd)
        }
    }
}

func startPolling(bot *tgbotapi.BotAPI) tgbotapi.UpdatesChannel {
    u := tgbotapi.NewUpdate(0) // c самого раннего
    u.Timeout = 30             // long polling 30s
    return bot.GetUpdatesChan(u)
}
```

`GetUpdates` подтверждает обновления через `offset`. Хранить последний `update_id` не обязательно, библиотека управляет циклом.

#### 7.2 Webhook (prod)

Требования: публичный HTTPS, проверка заголовка `X-Telegram-Bot-Api-Secret-Token` на соответствие `TELEGRAM_WEBHOOK_SECRET`.

```go
func webhookHandler(log *slog.Logger, bot *tgbotapi.BotAPI, secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        if c.GetHeader("X-Telegram-Bot-Api-Secret-Token") != secret {
            c.AbortWithStatus(http.StatusForbidden)
            return
        }
        var upd tgbotapi.Update
        if err := c.ShouldBindJSON(&upd); err != nil {
            log.Error("bad update", slog.Any("err", err))
            c.AbortWithStatus(http.StatusBadRequest)
            return
        }
        go handleUpdate(c.Request.Context(), log, bot, upd)
        c.JSON(http.StatusOK, gin.H{"ok": true})
    }
}
```

Идемпотентность при вебхуках: сохранять `update_id` (LRU/TTL-кеш или таблица) и отбрасывать дубликаты. Telegram может ретраить доставку.

```go
import (
    "sync"
    "time"
)

type Dedup interface { Seen(id int) bool }

// Simple in-memory implementation (use Redis/DB for prod)
type lruDedup struct { mu sync.Mutex; m map[int]time.Time; ttl time.Duration }
func (d *lruDedup) Seen(id int) bool {
    d.mu.Lock(); defer d.mu.Unlock()
    if t, ok := d.m[id]; ok && time.Since(t) < d.ttl { return true }
    d.m[id] = time.Now(); return false
}
```

#### 7.3 Порядок и конкурентность

- Обрабатывать обновления по чатам последовательно (сохранить порядок), но параллелить между чатами. Простой вариант — шардированный воркер-пул по `chat_id` (`workers = N`, индекс = `abs(chatID)%N`).
- Долгие операции (HTTP/БД) — с контекстными таймаутами.

```go
import (
    "context"
    "log/slog"

    tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"
)

func handleUpdate(ctx context.Context, log *slog.Logger, bot *tgbotapi.BotAPI, upd tgbotapi.Update) {
    // simple /start responder example
    if msg := upd.Message; msg != nil && msg.IsCommand() {
        switch msg.Command() {
        case "start":
            reply := tgbotapi.NewMessage(msg.Chat.ID, "Привет! Я готов.")
            if _, err := bot.Send(reply); err != nil {
                log.Error("send failed", slog.Any("err", err))
            }
        }
    }
}
```

### 8) Валидация (go-playground/validator)

Пример схем и проверки конфига/DTO:

```go
import "github.com/go-playground/validator/v10"

var v = validator.New()

type CreateProfileDTO struct {
    Name  string `validate:"required,min=2,max=64"`
    Email string `validate:"required,email"`
}

func ValidateDTO[T any](in T) error { return v.Struct(in) }
```

### 9) Swagger / автодокументация (опционально)

Если включены вебхуки или REST эндпоинты, добавить `swag init` и роут `/swagger/*any`.

```go
import (
    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"
)

// @title MenuBot-go API
// @version 1.0
// @BasePath /
func setupSwagger(r *gin.Engine) {
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
}
```

### 10) Composition root: `cmd/bot/main.go`

Последовательность:
1) Парсинг env → `Config` + `validator`.
2) Логгер.
3) Пул PostgreSQL.
4) Миграции (dev — всегда; prod — по флагу/переменной).
5) Инициализация tgbotapi.
6) Dev: запуск polling; Prod: настройка webhook (`SetWebhook`) и запуск Gin.
7) Graceful shutdown.

Перед запуском определим вспомогательные функции:

```go
// must panics on error to keep examples concise
func must[T any](v T, err error) T {
    if err != nil {
        panic(err)
    }
    return v
}

// mustLoadConfig loads configuration and panics on error
func mustLoadConfig() Config {
    return Config{
        Env: os.Getenv("ENV"),
        HTTP: struct{ Addr string }{Addr: os.Getenv("HTTP_ADDR")},
        Telegram: struct {
            Token        string
            WebhookURL   string
            WebhookSecret string
        }{
            Token:        os.Getenv("TELEGRAM_BOT_TOKEN"),
            WebhookURL:   os.Getenv("TELEGRAM_WEBHOOK_URL"),
            WebhookSecret: os.Getenv("TELEGRAM_WEBHOOK_SECRET"),
        },
        DB: struct{ URL string }{URL: os.Getenv("DATABASE_URL")},
    }
}
```

```go
import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"

    "log/slog"

    tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"
    "github.com/gin-gonic/gin"
)

func main() {
    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    cfg := mustLoadConfig()
    log := NewLogger(cfg)

    pool := must(NewPGPool(ctx, cfg.DB.URL))
    defer pool.Close()

    if cfg.Env == "dev" { _ = ApplyMigrations(cfg.DB.URL) }

    bot, err := tgbotapi.NewBotAPI(cfg.Telegram.Token)
    if err != nil { log.Error("bot init", slog.Any("err", err)); os.Exit(1) }

    if cfg.Telegram.WebhookURL != "" { // prod webhook
        wh, _ := tgbotapi.NewWebhookWithCert(cfg.Telegram.WebhookURL, nil)
        wh.SecretToken = cfg.Telegram.WebhookSecret
        if _, err := bot.Request(wh); err != nil { panic(err) }

        r := gin.New()
        r.Use(gin.Recovery())
        r.POST("/telegram/webhook", webhookHandler(log, bot, cfg.Telegram.WebhookSecret))
        srv := &http.Server{Addr: cfg.HTTP.Addr, Handler: r}
        go func() { _ = srv.ListenAndServe() }()
        <-ctx.Done()
        _ = srv.Shutdown(context.Background())
    } else { // dev polling
        updates := startPolling(bot)
        go runPolling(ctx, log, bot, updates)
        <-ctx.Done()
    }
}
```

### 11) Ошибки и контексты

- Все внешние вызовы: с `context.WithTimeout` и обработкой `errors.Is(err, context.DeadlineExceeded)`.
- Доменные ошибки отделять от инфраструктурных; наружу — минимальные сообщения, без PII.

### 12) Идемпотентность (подробно)

- Long polling: подтверждение `offset` (автоматически в `GetUpdatesChan`).
- Webhook: хранить `update_id` в краткоживущем кеше (например, 10 минут) или транзакционно в БД (ключ — `update_id`, значение — время/статус). При дубликате — мгновенно игнорировать.
- При критичных операциях (изменение состояния в БД) использовать уникальные ключи и/или `INSERT ... ON CONFLICT DO NOTHING`.

### 13) Graceful shutdown

`signal.NotifyContext` создаёт контекст, который автоматически отменяется при получении `SIGINT` или `SIGTERM`, позволяя корректно завершить HTTP‑сервер, Telegram-поток и фоновых воркеров.

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM) // catch SIGINT/SIGTERM
defer stop()

pool, err := NewPGPool(ctx, dsn)
if err != nil {
    panic(err)
}

bot, err := tgbotapi.NewBotAPI(token)
if err != nil {
    panic(err)
}
updates := startPolling(bot)
go runPolling(ctx, log, bot, updates)

srv := &http.Server{Addr: ":8080", Handler: mux}
go func() { _ = srv.ListenAndServe() }()

workers := NewWorkerPool()
go workers.Run(ctx)

<-ctx.Done()

workers.Stop()                                     // stop background workers
shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
_ = srv.Shutdown(shutdownCtx)                     // stop HTTP server
pool.Close()                                      // close database pool
```

### 14) Безопасность и приватность

- Не логировать токены, содержимое пользовательских сообщений и PII в открытом виде.
- Webhook: проверять `X-Telegram-Bot-Api-Secret-Token`.
- Минимизировать права БД-пользователя (отдельный db user для приложения).

### 15) Тесты и проверка

- Юнит-тесты: домен и usecase (табличные тесты стандартной библиотекой).
- Интеграционные: репозитории (за build-тегом), smoke-тест Telegram (dev-токен, polling, отправка команда → проверка ответа).

### 16) Чек-лист запуска

- [ ] `ENV`, `TELEGRAM_BOT_TOKEN`, `DATABASE_URL`, `LOG_LEVEL` установлены
- [ ] prod: webhook создан и активен, секрет проверяется
- [ ] миграции применены
- [ ] логи JSON/ротация на месте, алерты по ошибкам
- [ ] контроль времени: HTTP/БД/LLM вызовы с таймаутами

