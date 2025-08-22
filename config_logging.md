## Конфигурация и логирование (.env, slog+tint+JSON+lumberjack, приватность/PII)

Документ — самодостаточный. Покрывает: переменные окружения и загрузку конфига, валидацию (`go-playground/validator/v10`), инициализацию `slog` с цветным выводом для dev (`tint`) и JSON‑логами с ротацией (`lumberjack`) для prod, редактирование чувствительных полей (PII/секреты), контекстные поля (`request_id`, `chat_id`, `update_id`, `user_id`), уровни логов и примеры использования.

### 1) Переменные окружения

Минимум:

```env
# .env.example
ENV=dev               # dev|prod
HTTP_ADDR=:8080

DATABASE_URL=postgres://user:pass@localhost:5432/menubot?sslmode=disable

TELEGRAM_BOT_TOKEN=123456:ABC...  # секрет — не логировать
TELEGRAM_WEBHOOK_URL=
TELEGRAM_WEBHOOK_SECRET=

OPENAI_API_KEY=sk-...             # секрет — не логировать
OPENAI_BASE_URL=https://api.openai.com/v1  # кастомный URL (прокси)
OPENAI_MODEL_DRAFT=gpt-4o-mini-2024-07-18  # модель для черновых ответов (Config.OpenAI.ModelDraft)
OPENAI_MODEL_FINAL=gpt-4o-2024-11-20       # модель для финальных ответов (Config.OpenAI.ModelFinal)

LOG_LEVEL=info                     # debug|info|warn|error
LOG_FILE=                          # если задан — JSON+ротация в файл
```

Версии моделей закрепляйте датой (`*-YYYY-MM-DD`), например `gpt-4o-2024-11-20`, чтобы воспроизводить ответы и проще отлаживать; алиасы без даты (`gpt-4o`, `gpt-4o-mini`) автоматически обновляются и могут менять вывод, поэтому в prod фиксируйте датированные версии, в dev допустимо использовать недатированные.

Храните `.env` вне VCS, коммитьте только `.env.example`.


#### TEST_DATABASE_URL для интеграционных тестов

Для интеграционных тестов используется отдельная база данных. DSN задаётся через переменную окружения `TEST_DATABASE_URL`; она не требуется при обычном запуске бота, но нужна для тестов репозиториев.

```env
TEST_DATABASE_URL=postgres://user:pass@localhost:5432/menubot_test?sslmode=disable
```

Подробности использования — в [testing_and_dev](testing_and_dev.md).

### 2) Структура конфига и загрузка из окружения

```go
package config

import (
    "errors"
    "os"
    "strings"
    "github.com/go-playground/validator/v10"
)

type Config struct {
    Env   string `validate:"required,oneof=dev prod"`
    HTTP  struct { Addr string `validate:"required"` }
    DB    struct { URL string `validate:"required"` }
    Telegram struct {
        Token         string `validate:"required"`
        WebhookURL    string
        WebhookSecret string
    }
    OpenAI struct{
        APIKey     string `validate:"required"`
        BaseURL    string `validate:"required"`    // custom base URL for OpenAI API
        ModelDraft string `validate:"required"`    // pinned model ID for draft responses
        ModelFinal string `validate:"required"`    // pinned model ID for final responses
    }
    Log struct {
        Level string `validate:"required,oneof=debug info warn error"`
        File  string
    }
}

var validate = validator.New()

func Load() (Config, error) {
    var c Config
    c.Env = getenv("ENV", "dev")
    c.HTTP.Addr = getenv("HTTP_ADDR", ":8080")
    c.DB.URL = os.Getenv("DATABASE_URL")
    c.Telegram.Token = os.Getenv("TELEGRAM_BOT_TOKEN")
    c.Telegram.WebhookURL = os.Getenv("TELEGRAM_WEBHOOK_URL")
    c.Telegram.WebhookSecret = os.Getenv("TELEGRAM_WEBHOOK_SECRET")
    c.OpenAI.APIKey = os.Getenv("OPENAI_API_KEY")
    c.OpenAI.BaseURL = getenv("OPENAI_BASE_URL", "https://api.openai.com/v1")
    c.OpenAI.ModelDraft = getenv("OPENAI_MODEL_DRAFT", "gpt-4o-mini-2024-07-18")
    c.OpenAI.ModelFinal = getenv("OPENAI_MODEL_FINAL", "gpt-4o-2024-11-20")
    c.Log.Level = strings.ToLower(getenv("LOG_LEVEL", "info"))
    c.Log.File = os.Getenv("LOG_FILE")
    if err := validate.Struct(c); err != nil { return Config{}, err }
    if c.Telegram.WebhookURL != "" && c.Telegram.WebhookSecret == "" {
        return Config{}, errors.New("WEBHOOK_SECRET required when WEBHOOK_URL is set")
    }
    return c, nil
}

func getenv(k, def string) string { if v := os.Getenv(k); v != "" { return v }; return def }
```

### 3) Инициализация логгера: dev (tint) и prod (JSON+ротация)

```go
package logger

import (
    "context"
    "log/slog"
    "os"
    "strings"
    "time"

    "github.com/lmittmann/tint"
    "gopkg.in/natefinch/lumberjack.v2"
)

type Options struct {
    Env    string // dev|prod
    Level  string // debug|info|warn|error
    File   string // если не пуст — писать JSON c ротацией
    App    string // имя приложения
}

func New(opts Options) *slog.Logger {
    lvl := parseLevel(opts.Level)
    var h slog.Handler
    if opts.File != "" { // prod JSON + ротация
        lj := &lumberjack.Logger{Filename: opts.File, MaxSize: 50, MaxBackups: 10, MaxAge: 30, Compress: true}
        h = slog.NewJSONHandler(lj, &slog.HandlerOptions{Level: lvl, AddSource: false})
    } else { // dev цветной вывод
        h = tint.NewHandler(os.Stdout, &tint.Options{Level: lvl, TimeFormat: time.TimeOnly, NoColor: false})
    }
    // редактирование чувствительных полей
    rh := NewRedactingHandler(h, []string{"token", "secret", "api_key", "password"})
    log := slog.New(rh).With(slog.String("app", opts.App), slog.String("env", opts.Env))
    return log
}

func parseLevel(s string) slog.Leveler {
    switch s {
    case "debug": return slog.LevelDebug
    case "info": return slog.LevelInfo
    case "warn": return slog.LevelWarn
    case "error": return slog.LevelError
    default: return slog.LevelInfo
    }
}

// --- RedactingHandler ---
type RedactingHandler struct {
    inner slog.Handler
    keys  map[string]struct{}
}

func NewRedactingHandler(inner slog.Handler, sensitive []string) *RedactingHandler {
    m := make(map[string]struct{}, len(sensitive))
    for _, k := range sensitive { m[k] = struct{}{} }
    return &RedactingHandler{inner: inner, keys: m}
}

func (h *RedactingHandler) Enabled(ctx context.Context, l slog.Level) bool { return h.inner.Enabled(ctx, l) }
func (h *RedactingHandler) WithAttrs(attrs []slog.Attr) slog.Handler { return &RedactingHandler{inner: h.inner.WithAttrs(h.sanitize(attrs...)), keys: h.keys} }
func (h *RedactingHandler) WithGroup(name string) slog.Handler { return &RedactingHandler{inner: h.inner.WithGroup(name), keys: h.keys} }

func (h *RedactingHandler) Handle(ctx context.Context, r slog.Record) error {
    nr := slog.NewRecord(r.Time, r.Level, r.Message, r.PC)
    // переложить атрибуты с редактированием
    var attrs []slog.Attr
    r.Attrs(func(a slog.Attr) bool { attrs = append(attrs, a); return true })
    nr.AddAttrs(h.sanitize(attrs...)...)
    return h.inner.Handle(ctx, nr)
}

func (h *RedactingHandler) sanitize(attrs ...slog.Attr) []slog.Attr {
    out := make([]slog.Attr, 0, len(attrs))
    for _, a := range attrs {
        k := a.Key
        if _, ok := h.keys[strings.ToLower(k)]; ok {
            out = append(out, slog.String(k, "[REDACTED]"))
            continue
        }
        // простая защита для строк с ключеподобными значениями
        if s, ok := a.Value.Any().(string); ok && looksSensitive(s) {
            out = append(out, slog.String(k, "[REDACTED]"))
            continue
        }
        out = append(out, a)
    }
    return out
}

func looksSensitive(s string) bool {
    if len(s) > 12 && (strings.Contains(s, "sk-") || strings.Contains(strings.ToLower(s), "token")) { return true }
    return false
}
```

Пример инициализации:

```go
cfg, _ := config.Load()
log := logger.New(logger.Options{Env: cfg.Env, Level: cfg.Log.Level, File: cfg.Log.File, App: "menubot-go"})
log.Info("service start", slog.String("addr", cfg.HTTP.Addr))
```

### 4) Контекстные поля и корелляция

Соглашение по полям: `request_id`, `chat_id`, `update_id`, `user_id` — добавляйте через `With`/`WithGroup`.

```go
func WithUpdateCtx(log *slog.Logger, chatID int64, updateID int) *slog.Logger {
    return log.With(
        slog.Int64("chat_id", chatID),
        slog.Int("update_id", updateID),
    )
}

// В хендлере апдейтов
ll := WithUpdateCtx(log, upd.Chat.ID, upd.UpdateID)
ll.Info("incoming message")
```

Если используете Gin для вебхуков — добавляйте `request_id` (из заголовка/генерируйте) и статус/тайминги в middleware.

### 5) Политика приватности и PII

- Никогда не логируйте: содержимое пользовательских сообщений, токены/секреты, полные ответы LLM.
- Разрешены метаданные: `chat_id`, `update_id`, длительность, размеры, статусы.
- Для ошибок — логируйте тип/код и усечённое сообщение, без включения приватных данных.

### 6) Примеры

Инициализация в `cmd/bot/main.go`:

```go
func main() {
    cfg, err := config.Load(); if err != nil { panic(err) }
    log := logger.New(logger.Options{Env: cfg.Env, Level: cfg.Log.Level, File: cfg.Log.File, App: "menubot-go"})

    // Дальше: пул БД, миграции, бота — используйте log.With(...) для контекста
    log.Info("starting", slog.String("env", cfg.Env))
}
```

Пример dev‑лога (tint):

```
14:22:10 INF service start addr=:8080 app=menubot-go env=dev
```

Пример prod JSON (с ротацией):

```json
{"time":"2025-08-10T11:22:10Z","level":"INFO","msg":"service start","addr":":8080","app":"menubot-go","env":"prod"}
```

---

Чек‑лист
- Конфиг загружается из ENV и валидируется
- Логгер: dev — `tint`, prod — JSON+`lumberjack`
- Чувствительные поля редактируются
- Контекстные поля (`request_id`, `chat_id`, `update_id`, `user_id`) добавляются единообразно
- Уровни логов настраиваются через `LOG_LEVEL`

