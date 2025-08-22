## PostgreSQL и миграции (pgx/v5, транзакции, автоприменение)

Документ — самодостаточный. Описывает минимальную схему БД под домен, подключение через `pgx/v5`, транзакции и `TxRunner`, эффективные вставки (`COPY`), автоприменение миграций (`golang-migrate`), и практики ошибок/ретраев. Примеры готовы к использованию.

### 1) Минимальная схема

SQL миграции в каталоге `migrations/` (нумерация `0001_*.up.sql/.down.sql`). Минимальный набор таблиц под домен:

```sql
-- 0001_init.up.sql
CREATE EXTENSION IF NOT EXISTS pgcrypto; -- при необходимости uuid_generate_v4

CREATE TABLE users (
    id          TEXT PRIMARY KEY,
    tg_user_id  BIGINT UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_profiles (
    user_id    TEXT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    goal       TEXT NOT NULL DEFAULT '',
    likes      TEXT[] NOT NULL DEFAULT '{}',
    dislikes   TEXT[] NOT NULL DEFAULT '{}',
    allergies  TEXT[] NOT NULL DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pantry_items (
    id        TEXT PRIMARY KEY,
    user_id   TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name      TEXT NOT NULL,
    qty       TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON pantry_items(user_id);

CREATE TABLE menus (
    id            TEXT PRIMARY KEY,
    user_id       TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status        TEXT NOT NULL CHECK (status IN ('draft','final')),
    days          INT  NOT NULL CHECK (days BETWEEN 1 AND 14),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    finalized_at  TIMESTAMPTZ NULL
);
CREATE INDEX ON menus(user_id);

CREATE TABLE menu_items (
    id         BIGSERIAL PRIMARY KEY,
    menu_id    TEXT NOT NULL REFERENCES menus(id) ON DELETE CASCADE,
    day_index  INT  NOT NULL CHECK (day_index >= 1),
    meal_type  TEXT NOT NULL,
    title      TEXT NOT NULL,
    steps      TEXT[] NOT NULL
);
CREATE UNIQUE INDEX uq_menu_items ON menu_items(menu_id, day_index, meal_type);

CREATE TABLE menu_feedback (
    id         BIGSERIAL PRIMARY KEY,
    menu_id    TEXT NOT NULL REFERENCES menus(id) ON DELETE CASCADE,
    day_index  INT  NOT NULL,
    meal_type  TEXT NOT NULL,
    liked      BOOLEAN NOT NULL,
    comment    TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON menu_feedback(menu_id);

CREATE TABLE user_state (
    user_id     TEXT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    chat_id     BIGINT NOT NULL,
    flow        TEXT NOT NULL,
    state       TEXT NOT NULL,
    data        JSONB NOT NULL DEFAULT '{}'::jsonb,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON user_state(flow);
```

Таблица `user_state` хранит текущий поток и состояние конечного автомата, поле `user_id` ссылается на `users(id)` — подробнее см. [fsm_flows.md](fsm_flows.md).

Down‑миграция: удаление таблиц в обратном порядке (`DROP TABLE ...`).

#### Сидирование dev-данных

Для локальной разработки удобно наполнить БД минимальным набором данных. Пример SQL-скрипта `scripts/seed_dev.sql`:

```sql
-- scripts/seed_dev.sql
INSERT INTO users (id, tg_user_id)
VALUES ('00000000-0000-0000-0000-000000000001', 123456789);

INSERT INTO menus (id, user_id, status, days)
VALUES ('11111111-1111-1111-1111-111111111111', '00000000-0000-0000-0000-000000000001', 'draft', 7);
```

Аналогичный скрипт на Go:

```go
package main

import (
    "context"
    "log"
    "os"

    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    ctx := context.Background()
    dsn := os.Getenv("DATABASE_URL")
    pool, err := pgxpool.New(ctx, dsn)
    if err != nil { log.Fatal(err) }
    defer pool.Close()

    if _, err := pool.Exec(ctx, `
        insert into users(id, tg_user_id)
        values ($1, $2)
        on conflict (id) do nothing
    `, "00000000-0000-0000-0000-000000000001", 123456789); err != nil {
        log.Fatal(err)
    }

    if _, err := pool.Exec(ctx, `
        insert into menus(id, user_id, status, days)
        values ($1, $2, 'draft', 7)
        on conflict (id) do nothing
    `, "11111111-1111-1111-1111-111111111111", "00000000-0000-0000-0000-000000000001"); err != nil {
        log.Fatal(err)
    }
}
```

Порядок запуска:

1. Применить миграции (см. ниже).
2. Выполнить скрипт сидирования.

### 2) Инициализация пула pgx и проверка соединения

```go
package pg

import (
    "context"
    "time"
    "github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
    cfg, err := pgxpool.ParseConfig(dsn)
    if err != nil { return nil, err }
    // Fine tuning if needed: MaxConns, HealthCheckPeriod, etc.
    cfg.MaxConns = 20
    cfg.MinConns = 2
    cfg.HealthCheckPeriod = 30 * time.Second
    cfg.MaxConnLifetime = time.Hour
    cfg.MaxConnIdleTime = 10 * time.Minute
    pool, err := pgxpool.NewWithConfig(ctx, cfg)
    if err != nil { return nil, err }
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    if err := pool.Ping(ctx); err != nil { pool.Close(); return nil, err }
    return pool, nil
}
```

`MaxConnIdleTime` задан в 10 минут, чтобы освобождать долго простаивающие соединения, не создавая лишней нагрузки при их частом пересоздании.

### 3) Транзакции и `TxRunner`

Реализация порта `TxRunner` поверх `pgxpool.Pool`: транзакция сохраняется в `context.Context`.

```go
package pg

import (
    "context"
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

type txKey struct{} // context key for pgx.Tx

type TxRunner struct { Pool *pgxpool.Pool }

func (r TxRunner) WithinTx(ctx context.Context, fn func(ctx context.Context) error) error {
    // BeginFunc rolls back on error and commits on nil.
    return pgx.BeginFunc(ctx, r.Pool, func(tx pgx.Tx) error {
        ctx = context.WithValue(ctx, txKey{}, tx)
        return fn(ctx)
    })
}

func PgxTx(ctx context.Context) (pgx.Tx, bool) {
    tx, ok := ctx.Value(txKey{}).(pgx.Tx)
    return tx, ok
}
```

`PgxTx` извлекает транзакцию из контекста.

Пример использования:

```go
err := txRunner.WithinTx(ctx, func(ctx context.Context) error {
    tx, _ := PgxTx(ctx)
    if _, err := tx.Exec(ctx, "insert into users(id,tg_user_id) values($1,$2) on conflict do nothing", id, tgID); err != nil { return err }
    // ...
    return nil
})
```

### 4) Репозитории на pgx/v5 (примеры)

`UserRepo.GetOrCreateByTelegramID` (возвращает доменную ошибку `ErrNotFound` при отсутствии):

```go
package postgres

import (
    "context"
    "errors"
    "time"

    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
    "yourbot/internal/domain"
)

type UserRepo struct { Pool *pgxpool.Pool }

func (r *UserRepo) GetOrCreateByTelegramID(ctx context.Context, tgID int64) (domain.User, error) {
    var id string
    err := r.Pool.QueryRow(ctx, `
        insert into users(id, tg_user_id) values (gen_random_uuid()::text, $1)
        on conflict (tg_user_id) do update set tg_user_id=excluded.tg_user_id
        returning id
    `, tgID).Scan(&id)
    if err != nil { return domain.User{}, err }
    return domain.NewUser(domain.UserID(id), tgID, time.Now())
}

func (r *UserRepo) getByID(ctx context.Context, id string) (domain.User, error) {
    var tg int64; var created time.Time
    err := r.Pool.QueryRow(ctx, `select tg_user_id, created_at from users where id=$1`, id).Scan(&tg, &created)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            // pgx.ErrNoRows means no record in DB
            return domain.User{}, domain.ErrNotFound
        }
        return domain.User{}, err
    }
    return domain.NewUser(domain.UserID(id), tg, created)
}
```

Обновление профиля:

```go
type ProfileRepo struct { Pool *pgxpool.Pool }

func (r *ProfileRepo) Upsert(ctx context.Context, p domain.UserProfile) error {
    _, err := r.Pool.Exec(ctx, `
        insert into user_profiles(user_id, goal, likes, dislikes, allergies, updated_at)
        values ($1,$2,$3,$4,$5, now())
        on conflict (user_id)
        do update set goal=excluded.goal, likes=excluded.likes, dislikes=excluded.dislikes, allergies=excluded.allergies, updated_at=now()
    `, p.UserID, p.Goal, p.Likes, p.Dislikes, p.Allergies)
    return err
}
```

### 5) Быстрые вставки через `COPY`

Для больших батчей меню используйте `CopyFrom` (быстрее `INSERT ... VALUES`).

```go
import (
    "github.com/jackc/pgx/v5"
)

func (r *MenuRepo) ReplaceItemsBulk(ctx context.Context, menuID string, items []domain.MenuItem) (int64, error) {
    rows := pgx.CopyFromSlice(len(items), func(i int) ([]any, error) {
        it := items[i]
        return []any{menuID, it.DayIndex, string(it.MealType), it.Title, it.Steps}, nil
    })
    return r.Pool.CopyFrom(ctx, pgx.Identifier{"menu_items"}, []string{"menu_id","day_index","meal_type","title","steps"}, rows)
}
```

Замечание: если используете `database/sql` совместимость и `COPY` внутри транзакции — см. приём с `Conn.Raw` и `CopyFrom` на одном соединении.

### 6) Автоприменение миграций

Миграции применяются на старте независимо от окружения:

```go
package migrate

import (
    migrate "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func Up(dsn string) error {
    m, err := migrate.New("file://migrations", dsn)
    if err != nil { return err }
    defer m.Close()
    if err := m.Up(); err != nil && err != migrate.ErrNoChange { return err }
    return nil
}
```

Бот применяет миграции сам, отдельный шаг деплоя не требуется.

### 7) Ошибки и ретраи

- Оборачивайте ошибки `%w` и маппьте «нет записей» на `domain.ErrNotFound`.
- На уровне HTTP/LLM‑клиентов используйте ретраи с экспоненциальным бэкоффом. На уровне БД избегайте бездумных ретраев внутри транзакций (опасность повторного эффекта).
- Для конфликтов уникальности (`uq_menu_items`) — используйте `ON CONFLICT DO UPDATE/NOTHING` и/или корректный апстрим.

### 8) Параметры соединения

`DATABASE_URL` формат: `postgres://user:pass@host:5432/db?sslmode=disable`.

Рекомендуемые параметры пула:
- `MaxConns=20`, `MinConns=2`, `HealthCheckPeriod=30s`, `MaxConnLifetime=1h`, `MaxConnIdleTime=10m` — настраиваются под нагрузку.

### 9) Инструменты

- Миграции: `golang-migrate/migrate`
- Драйвер: `github.com/jackc/pgx/v5` (`pgxpool` для пула)
- Генерация кода (опционально): `sqlc` для типобезопасного доступа.

---

Чек‑лист
- Схема хранит сущности из `dev-docs/domain.md`; ограничения обеспечивают инварианты (status/days)
- Инициализация пула и `Ping` при старте
- `TxRunner` реализован и используется в use‑cases
- Для батчей — `COPY`
- Автоприменение миграций при запуске
