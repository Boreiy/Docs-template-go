## Тестирование и дев‑среда (testing/testify/ginkgo, покрытие, Makefile, Docker/compose)

Документ — самодостаточный. Описывает структуру тестов на Go, юнит‑ и интеграционные тесты (репозитории на `pgx`), использование `testify` (assert/require), опционально `ginkgo/gomega`, измерение покрытия, Makefile‑целей для dev, запуск Postgres через Docker Compose, smoke‑тесты интеграций (Telegram/LLM) с отключением по ENV.

### Переменные HTTP-клиента

- `HTTP_CLIENT_TIMEOUT` — таймаут одного HTTP-запроса.
- `HTTP_CLIENT_RETRIES` — количество повторных попыток при временных ошибках.
- `HTTP_CLIENT_HEADERS` — дополнительные заголовки, записываются через запятую в формате `Key=Value`.

### 1) Структура тестов

- Файлы тестов рядом с кодом, `*_test.go`.
- Именование тестов: `TestXxx(t *testing.T)`. Для таблиц — table‑driven tests.
- Юнит‑тесты для домена/юзкейсов; интеграционные — для адаптеров (`internal/adapter/...`) под build‑тегом или отдельным пакетом.

Пример table‑driven для доменных инвариантов:

```go
package domain_test

import (
    "testing"
    "time"
    "yourbot/internal/domain"
)

func TestNewMenuDraft_DaysBounds(t *testing.T) {
    cases := []struct{ days int; wantErr bool }{
        {0, true}, {1, false}, {14, false}, {15, true},
    }
    for _, c := range cases {
        _, err := domain.NewMenuDraft("m1", "u1", c.days, time.Now())
        if (err != nil) != c.wantErr {
            t.Fatalf("days=%d: err=%v", c.days, err)
        }
    }
}
```

### 2) Testify: assert/require

`github.com/stretchr/testify/{assert,require}` — удобные проверки и группы.

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestProfileUpsert(t *testing.T) {
    got, err := someFunc()
    require.NoError(t, err)
    assert.Equal(t, "ok", got.Status)
}
```

### 3) Ginkgo/Gomega (опционально)

Подходит для BDD‑стиля и асинхронных ожиданий. Для проекта достаточно stdlib+testify; ginkgo/gomega — по необходимости.

### 4) Интеграционные тесты репозиториев (pgx)

- Используйте реальный Postgres в Docker.
- DSN для тестов через `TEST_DATABASE_URL`; миграции применяйте перед тестами.

Пример `TestMain` для применения миграций один раз:

```go
package postgres_test

import (
    "os"
    "testing"
    migrate "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func TestMain(m *testing.M) {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" { os.Exit(m.Run()) } // пропускаем интеграционные
    mg, err := migrate.New("file://migrations", dsn)
    if err == nil {
        _ = mg.Up()
        _ = mg.Close()
    }
    os.Exit(m.Run())
}
```

Тест репозитория:

```go
func TestUserRepo_GetOrCreateByTelegramID(t *testing.T) {
    dsn := os.Getenv("TEST_DATABASE_URL")
    if dsn == "" { t.Skip("TEST_DATABASE_URL not set") }
    ctx := context.Background()
    pool, err := pg.NewPool(ctx, dsn)
    require.NoError(t, err)
    defer pool.Close()

    repo := &postgres.UserRepo{Pool: pool}
    u, err := repo.GetOrCreateByTelegramID(ctx, 12345)
    require.NoError(t, err)
    assert.NotEmpty(t, u.ID)
    assert.Equal(t, int64(12345), u.TelegramID)
}
```

### 5) Покрытие

- Локально: `go test ./... -cover`.
- С отчётом: `go test ./... -coverprofile=coverage.out && go tool cover -func=coverage.out`.
- HTML‑отчёт: `go tool cover -html=coverage.out -o coverage.html`.
- С поиском гонок: `go test -race ./...`.

Детектор гонок помогает выявлять небезопасный параллельный доступ к памяти; его стоит включать при работе с горутинами и при расследовании нестабильных ошибок.

### 6) Установка golangci-lint

1) `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest` — установка последней версии.
2) Проверьте установку: `golangci-lint --version`.
3) Для скачивания бинарника используйте официальный скрипт: `curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s -- -b $(go env GOPATH)/bin`.

Минимальная конфигурация `.golangci.yml`:

```yaml
run:
  timeout: 5m
linters:
  enable:
    - govet
    - errcheck
    - staticcheck
```

Больше настроек — в [примерной конфигурации проекта](https://github.com/golangci/golangci-lint/blob/master/.golangci.example.yml).

### 7) Makefile (dev‑цели)

```makefile
.PHONY: run build test cover compose-up compose-down lint

run:
        ENV=dev LOG_LEVEL=debug go run ./cmd/bot

build:
        go build ./...

test:
        go test ./...

cover:
        go test ./... -coverprofile=coverage.out
        go tool cover -func=coverage.out

compose-up:
        docker compose up -d postgres

compose-down:
        docker compose down

lint:
        golangci-lint run ./...
```

Цели:

- `run` — запуск бота в дев‑режиме.
- `build` — компиляция всех пакетов.
- `test` — запуск всех тестов.
- `cover` — сбор отчёта покрытия.
- `compose-up` / `compose-down` — запуск и остановка сервисов Docker Compose.
- `lint` — проверка кода через `golangci-lint`.
- Миграции применяются автоматически при запуске, отдельная цель не требуется.

### 8) Docker Compose для Postgres (dev)

`docker-compose.yml` (фрагмент):

```yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: yourbot
    ports:
      - "5432:5432"
    volumes:
      - yourbot_pg:/var/lib/postgresql/data
volumes:
  yourbot_pg: {}
```

Настройте `DATABASE_URL=postgres://user:pass@localhost:5432/yourbot?sslmode=disable`.

### 9) Smoke‑тесты интеграций (условные)

- Telegram: выполняйте только при `TELEGRAM_BOT_TOKEN` установленном, используйте polling и `t.Skip` по умолчанию.

```go
func TestTelegram_SendMessage_Smoke(t *testing.T) {
    if os.Getenv("TELEGRAM_BOT_TOKEN") == "" { t.Skip("no token") }
    // Инициализировать бота и отправить сообщение себе/в тестовый чат
}
```

#### Telegram: запуск smoke-теста

1. Получите `chat_id`:
   - напишите [@userinfobot](https://t.me/userinfobot) и скопируйте поле `Id`;
   - или отправьте сообщение своему боту и запросите `https://api.telegram.org/bot<токен>/getUpdates`.
2. Экспортируйте переменные окружения:

   ```bash
   export TELEGRAM_BOT_TOKEN=<токен>
   export TELEGRAM_CHAT_ID=<ваш chat_id>
   ```

3. Запустите тест:

   ```bash
   go test ./internal/adapter/telegram -run TestSendMessageSmoke -count=1
   ```

   Тест отправит сообщение в указанный чат и завершится при успехе.

- LLM (OpenAI): выполняйте при наличии `OPENAI_API_KEY`, с контекстным таймаутом (20–30s) и малым квотой.

### 10) Локальный запуск

1) `docker compose up -d postgres`
2) Экспортируйте ENV (`.env`) и `go run ./cmd/bot`
3) Миграции применяются автоматически при старте (см. `postgresql.md`).

### 11) Советы

- Для тестов с БД используйте разные схемы/префиксы тестовых данных, очищайте таблицы между кейсами.
- Для внешних API — стабилизируйте сеть таймаутами и отключайте тесты по умолчанию, включайте только вручную/по метке.
- Не логируйте PII; при отладке используйте метаданные (`chat_id`, `update_id`).

---

Чек‑лист
- Тесты покрывают домен (инварианты), use‑cases, репозитории (интеграционно)
- Покрытие собирается локально через `go tool cover`
- Makefile содержит `run/build/test/cover/lint`
- Установлен и настроен `golangci-lint`
- Postgres поднимается через Docker Compose
- Интеграционные smoke‑тесты Telegram/LLM отключены по умолчанию и зависят от ENV

