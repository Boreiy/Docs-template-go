## FSM и пользовательские потоки (онбординг, меню, профиль)

Документ — практический и самодостаточный. Покрывает: проектирование конечных автоматов для сценариев бота, диаграммы и таблицы состояний, переходы и их правила, хранение и восстановление состояния, обработку отмены/ошибок/неожиданного ввода, идемпотентность, примеры Go‑кода с `tgbotapi/v5` и PostgreSQL.

### Размещение в структуре проекта

FSM находится в каталоге `internal/adapter/telegram/fsm`, который относится к слою адаптеров (delivery) в структуре [Clean Architecture](./architecture.md#2-структура-каталогов-clean-architecture).

- `machine.go` — машина состояний и обработчик `Handle`.
- `store/postgres` — реализация хранилища состояния.
- `steps_onboarding.go`, `steps_menu.go` — шаги конкретных потоков.

### 1) Потоки и состояния (обзор)

Рекомендуемые потоки:

- Онбординг (`onboarding`): сбор аллергий, предпочтений, цели
- Составление меню (`menu`): дни, пожелания, «что в холодильнике», черновик, фидбек, финализация
- Профиль (`profile`): просмотр и изменения

Ключевые принципы:
- У каждого пользователя может быть одновременно **не более одного активного потока**.
- В каждом состоянии должна быть понятная подсказка и клавиатура (вкл. «Отмена»).
- Любое неожиданное сообщение — мягкий ответ + повтор вопроса.
- «Отмена» очищает состояние и возвращает в главное меню.
- Состояние **персистентное** (в БД), чтобы переживать рестарт/разрыв соединения.

### 2) Диаграммы состояний (аски‑схемы)

Онбординг:

```
start -> ASK_ALLERGIES -> ASK_LIKES -> ASK_DISLIKES -> ASK_GOAL -> DONE
                ^                ^                ^                 |
                | отмена         | отмена         | отмена          v
             CANCEL <------------------------------------------- DONE
```

Составление меню:

```
start -> ASK_DAYS -> ASK_DESIRES -> ASK_PANTRY -> GENERATING_DRAFT
                                                        |
                                                        v
                                                   FEEDBACK_LOOP
                                                        |
                                              [ok/кнопка меню]
                                                        v
                                                   FINALIZING -> DONE
                              ^                                 |
                              |------ «Отмена» ------------------
```

### 3) Хранение состояния (PostgreSQL)

DDL для таблицы `user_state`:

```sql
CREATE TABLE IF NOT EXISTS user_state (
    user_id      TEXT         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    chat_id      BIGINT       NOT NULL,
    flow         TEXT         NOT NULL,
    state        TEXT         NOT NULL,
    data         JSONB        NOT NULL DEFAULT '{}'::jsonb,
    updated_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id)
);

CREATE INDEX IF NOT EXISTS idx_user_state_flow ON user_state(flow);
```

`data` — контекст (промежуточные значения), сериализуется в JSON. `user_id` совпадает с доменным `UserID` (строка/UUID) и ссылается на `users(id)`; `PRIMARY KEY (user_id)` гарантирует один активный поток на пользователя и каскадное удаление при удалении пользователя.

Очистка по таймауту (например, 7 дней): шедулер/cron `DELETE FROM user_state WHERE updated_at < now() - interval '7 day';`

### 4) Типы и каркас FSM (Go)

```go
package fsm

import (
    "context"
    "encoding/json"
    "errors"
    "log/slog"
    "strconv"
    "strings"
    "time"
    pgx "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
    tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"
)

type FlowName string

const (
    FlowOnboarding FlowName = "onboarding"
    FlowMenu       FlowName = "menu"
)

// Список состояний для онбординга
const (
    StAskAllergies = "ASK_ALLERGIES"
    StAskLikes     = "ASK_LIKES"
    StAskDislikes  = "ASK_DISLIKES"
    StAskGoal      = "ASK_GOAL"
    StDone         = "DONE"
)

// Список состояний для меню
const (
    StAskDays       = "ASK_DAYS"
    StAskDesires    = "ASK_DESIRES"
    StAskPantry     = "ASK_PANTRY"
    StGenerating    = "GENERATING_DRAFT"
    StFeedbackLoop  = "FEEDBACK_LOOP"
    StFinalizing    = "FINALIZING"
)

type State struct {
    UserID   string
    ChatID   int64
    Flow     FlowName
    Name     string
    Data     map[string]any
    Updated  time.Time
}

type Store interface {
    Load(ctx context.Context, userID string) (*State, error)
    Save(ctx context.Context, st *State) error
    Clear(ctx context.Context, userID string) error
}

// PgxIface defines minimal pgx methods for testing.
type PgxIface interface {
    QueryRow(ctx context.Context, sql string, args ...any) pgx.Row
    Exec(ctx context.Context, sql string, args ...any) (pgconn.CommandTag, error)
}

// PgStore: example implementation using pgx (trimmed)
type PgStore struct { Pool PgxIface }

func (s *PgStore) Load(ctx context.Context, userID string) (*State, error) {
    var chatID int64
    var flow, name string
    var dataBytes []byte
    var updated time.Time
    err := s.Pool.QueryRow(ctx,
        `SELECT chat_id, flow, state, data, updated_at FROM user_state WHERE user_id=$1`, userID,
    ).Scan(&chatID, &flow, &name, &dataBytes, &updated)
    if err != nil { return nil, err }
    m := map[string]any{}
    _ = json.Unmarshal(dataBytes, &m)
    return &State{UserID: userID, ChatID: chatID, Flow: FlowName(flow), Name: name, Data: m, Updated: updated}, nil
}

func (s *PgStore) Save(ctx context.Context, st *State) error {
    b, _ := json.Marshal(st.Data)
    _, err := s.Pool.Exec(ctx, `
        INSERT INTO user_state(user_id, chat_id, flow, state, data, updated_at)
        VALUES ($1,$2,$3,$4,$5,now())
        ON CONFLICT (user_id)
        DO UPDATE SET chat_id=EXCLUDED.chat_id, flow=EXCLUDED.flow, state=EXCLUDED.state, data=EXCLUDED.data, updated_at=now()
    `, st.UserID, st.ChatID, st.Flow, st.Name, b)
    return err
}

func (s *PgStore) Clear(ctx context.Context, userID string) error {
    _, err := s.Pool.Exec(ctx, `DELETE FROM user_state WHERE user_id=$1`, userID)
    return err
}

// StepFunc — обработчик состояния. Возвращает следующее состояние.
type StepFunc func(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (next string, err error)

type Machine struct {
    store Store
    steps map[FlowName]map[string]StepFunc
    log   *slog.Logger
}

func NewMachine(store Store, log *slog.Logger) *Machine {
    return &Machine{store: store, steps: map[FlowName]map[string]StepFunc{}, log: log}
}

func (m *Machine) Register(flow FlowName, name string, fn StepFunc) {
    if _, ok := m.steps[flow]; !ok { m.steps[flow] = map[string]StepFunc{} }
    m.steps[flow][name] = fn
}

func (m *Machine) Handle(ctx context.Context, bot *tgbotapi.BotAPI, upd tgbotapi.Update) error {
    userID, chatID := extractIDs(upd)
    if userID == "" { return nil }

    // Глобальная отмена
    if isCancel(upd) {
        _ = m.store.Clear(ctx, userID)
        send(bot, chatID, "Отменено. Главное меню:")
        // показать главное меню
        return nil
    }

    st, err := m.store.Load(ctx, userID)
    if err != nil { // нет состояния: команда/старт потоков
        if startFlow := tryStartFlow(bot, upd); startFlow != nil {
            // начальная запись
            st = &State{UserID: userID, ChatID: chatID, Flow: startFlow.flow, Name: startFlow.state, Data: map[string]any{}}
            _ = m.store.Save(ctx, st)
        } else {
            // показать главное меню/подсказку
            return nil
        }
    }

    step := m.steps[st.Flow][st.Name]
    if step == nil { return errors.New("unknown step") }
    next, err := step(ctx, bot, st, upd)
    if err != nil { return err }
    if next == "" { return nil } // остаёмся на шаге (повтор ввода)
    st.Name = next
    if next == StDone { _ = m.store.Clear(ctx, st.UserID) } else { _ = m.store.Save(ctx, st) }
    return nil
}

// helper functions (trimmed)
func extractIDs(upd tgbotapi.Update) (userID string, chatID int64) {
    if m := upd.Message; m != nil { return strconv.FormatInt(m.From.ID, 10), m.Chat.ID }
    if cb := upd.CallbackQuery; cb != nil { return strconv.FormatInt(cb.From.ID, 10), cb.Message.Chat.ID }
    return "", 0
}
func isCancel(upd tgbotapi.Update) bool {
    if m := upd.Message; m != nil { return m.Text == "Отмена" }
    return false
}

type startSpec struct { flow FlowName; state string }
func tryStartFlow(bot *tgbotapi.BotAPI, upd tgbotapi.Update) *startSpec {
    if m := upd.Message; m != nil && m.IsCommand() {
        switch m.Command() {
        case "start":
            send(bot, m.Chat.ID, "Привет! Выберите действие в меню.")
            return nil
        case "onboarding":
            send(bot, m.Chat.ID, "Начнём онбординг. Укажите аллергии (через запятую), либо напишите 'нет'.")
            return &startSpec{flow: FlowOnboarding, state: StAskAllergies}
        case "menu":
            send(bot, m.Chat.ID, "На сколько дней составить меню? (3/5/7)")
            return &startSpec{flow: FlowMenu, state: StAskDays}
        }
    }
    return nil
}
```

Интерфейс `PgxIface` задаёт минимальный набор методов `QueryRow` и `Exec`, поэтому в тестах можно подменять работу с базой на фейковую реализацию. В продакшене ему соответствует `*pgxpool.Pool`, что позволяет не привязывать логику FSM к конкретному типу подключения и упрощает изоляцию юнит‑тестов.

Перед тем как перейти к реализациям шагов, рассмотрим саму машину состояний.
`Machine` содержит:

- `store` — реализацию интерфейса `Store`, отвечающую за загрузку и сохранение состояния пользователя.
- `steps` — карту зарегистрированных обработчиков шагов, сгруппированных по потокам.

Инициализация `store` выполняется в точке входа приложения, например в `cmd/bot/main.go`. Создав конкретную реализацию хранилища, её передают в конструктор:

```go
st := &PgStore{Pool: db}
log := slog.New(slog.NewTextHandler(os.Stdout, nil))
m  := fsm.NewMachine(st, log)
```

`Machine` использует `store` внутри `Handle` для чтения текущего состояния и записи нового.

Для краткости примеров ниже используются вспомогательные функции:

```go
// send sends a plain text message and ignores send errors.
func send(bot *tgbotapi.BotAPI, chatID int64, text string) {
    msg := tgbotapi.NewMessage(chatID, text)
    _, _ = bot.Send(msg)
}

// getText extracts message text or callback data and returns empty string when absent.
func getText(u tgbotapi.Update) string {
    if m := u.Message; m != nil {
        return m.Text
    }
    if cb := u.CallbackQuery; cb != nil {
        return cb.Data
    }
    return ""
}
```

### 5) Реализация шагов: Онбординг

```go
// Регистрация шагов
func RegisterOnboarding(machine *Machine) {
    machine.Register(FlowOnboarding, StAskAllergies, stepAskAllergies)
    machine.Register(FlowOnboarding, StAskLikes,     stepAskLikes)
    machine.Register(FlowOnboarding, StAskDislikes,  stepAskDislikes)
    machine.Register(FlowOnboarding, StAskGoal,      stepAskGoal)
}

func stepAskAllergies(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    if text := getText(upd); text != "" {
        st.Data["allergies"] = parseList(text)
        send(bot, st.ChatID, "Отлично. Назовите предпочтения (что любите):")
        return StAskLikes, nil
    }
    send(bot, st.ChatID, "Пожалуйста, перечислите аллергии текстом или напишите 'нет'.")
    return "", nil
}

func stepAskLikes(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    if text := getText(upd); text != "" {
        st.Data["likes"] = parseList(text)
        send(bot, st.ChatID, "Есть ли нелюбимые продукты? Перечислите или 'нет'.")
        return StAskDislikes, nil
    }
    send(bot, st.ChatID, "Перечислите предпочтения через запятую или 'нет'.")
    return "", nil
}

func stepAskDislikes(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    if text := getText(upd); text != "" {
        st.Data["dislikes"] = parseList(text)
        send(bot, st.ChatID, "Какова ваша цель питания? (например: похудение/поддержание/набор)")
        return StAskGoal, nil
    }
    send(bot, st.ChatID, "Перечислите нелюбимые продукты или 'нет'.")
    return "", nil
}

func stepAskGoal(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    if text := getText(upd); text != "" {
        st.Data["goal"] = text
        // Здесь сохранить профиль в БД, идемпотентно (UPSERT)
        send(bot, st.ChatID, "Профиль сохранён. Возвращаю в главное меню.")
        return StDone, nil
    }
    send(bot, st.ChatID, "Опишите цель питания в свободной форме.")
    return "", nil
}

// parseList returns a normalized list of tags.
func parseList(s string) []string {
    parts := strings.Split(s, ",")
    seen := make(map[string]struct{}, len(parts))
    out := make([]string, 0, len(parts))
    for _, p := range parts {
        tag := strings.ToLower(strings.TrimSpace(p))
        if tag == "" {
            continue
        }
        if _, ok := seen[tag]; !ok {
            seen[tag] = struct{}{}
            out = append(out, tag)
        }
    }
    return out
}
```

### 6) Реализация шагов: Составление меню

Шаг `stepAskPantry` замыкает `machine`, чтобы из фоновой горутины явно сохранять состояние.

```go
func RegisterMenu(machine *Machine) {
    machine.Register(FlowMenu, StAskDays,      stepAskDays)
    machine.Register(FlowMenu, StAskDesires,   stepAskDesires)
    machine.Register(FlowMenu, StAskPantry,    stepAskPantry(machine))
    machine.Register(FlowMenu, StGenerating,   stepGenerating)
    machine.Register(FlowMenu, StFeedbackLoop, stepFeedback)
    machine.Register(FlowMenu, StFinalizing,   stepFinalizing)
}

func stepAskDays(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    if d, ok := tryInt(getText(upd)); ok && (d == 3 || d == 5 || d == 7) {
        st.Data["days"] = d
        send(bot, st.ChatID, "Что хочется? (свободный текст)")
        return StAskDesires, nil
    }
    send(bot, st.ChatID, "Введите 3, 5 или 7.")
    return "", nil
}

func stepAskDesires(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    if txt := getText(upd); txt != "" {
        st.Data["desires"] = txt
        send(bot, st.ChatID, "Что есть в холодильнике? (свободный текст)")
        return StAskPantry, nil
    }
    send(bot, st.ChatID, "Опишите пожелания словами.")
    return "", nil
}

func stepAskPantry(machine *Machine) StepFunc {
    return func(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
        if txt := getText(upd); txt != "" {
            st.Data["pantry"] = txt
            send(bot, st.ChatID, "Генерирую черновик меню…")
            // Long operation in background; bot will send draft and switch to FEEDBACK_LOOP
            go func(s State, updID int) {
                // simulate
                time.Sleep(2 * time.Second)
                send(bot, s.ChatID, "Черновик готов. Поставьте оценки блюдам, затем нажмите 'Ок'.")
                // machine updates state itself
                s.Name = StFeedbackLoop
                if err := machine.store.Save(context.Background(), &s); err != nil {
                    machine.log.With(
                        slog.Int64("chat_id", s.ChatID),
                        slog.Int("update_id", updID),
                    ).Error("save state", "err", err)
                }
            }(*st, upd.UpdateID)
            return StGenerating, nil
        }
        send(bot, st.ChatID, "Опишите содержимое холодильника.")
        return "", nil
    }
}

func stepGenerating(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    // Пока ждём фонового результата — любые сообщения мягко игнорируем
    send(bot, st.ChatID, "Черновик в работе, это займёт немного времени…")
    return "", nil
}

func stepFeedback(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    if getText(upd) == "Ок" { // кнопка/слово подтверждения
        send(bot, st.ChatID, "Финализирую меню…")
        return StFinalizing, nil
    }
    // иначе — парсить свободный фидбек и обновлять черновик
    send(bot, st.ChatID, "Принято. Я обновлю черновик, напишите 'Ок' когда готовы финализировать.")
    return "", nil
}

func stepFinalizing(ctx context.Context, bot *tgbotapi.BotAPI, st *State, upd tgbotapi.Update) (string, error) {
    // сохранить финальное меню в БД
    send(bot, st.ChatID, "Готово! Финальное меню сохранено. Возвращаю в главное меню.")
    return StDone, nil
}

func tryInt(s string) (int, bool) { i, err := strconv.Atoi(strings.TrimSpace(s)); return i, err == nil }
```

Контроль ошибок в фоновых задачах обязателен: иначе сбой сохранения или другое исключение останутся незамеченными, и состояние потеряется.

### 7) Правила переходов и обработка ошибок

- Каждое состояние обязано:
  1) Проверить ввод и либо перейти к следующему состоянию, либо повторить вопрос.
  2) Записать данные в `st.Data` (частичные результаты) и сохранить.
  3) Поддерживать «Отмена» глобально (обрабатывается в `Machine.Handle`).

- Неожиданный тип события (фото вместо текста и т.п.) — дружелюбная подсказка и повтор шага.

- Идемпотентность: в шагах, меняющих БД, использовать UPSERT и внешние уникальные ключи. При повторной доставке апдейта состояния не дублируются.

### 8) Восстановление после сбоев

- Состояние хранится в БД — после рестарта бот восстановит контекст: любой новый апдейт пользователя будет обработан текущим `flow/state`.
- Если шаг «долгий» (генерация), запускать работу в фоне и периодически оповещать пользователя (или сразу после готовности).
- Состояния с «ожиданием» не должны зависеть от кэша/памяти — только от БД.

### 9) Клавиатуры и UX в FSM

- Вопросы с ограниченным набором ответов — `ReplyKeyboardMarkup` (например, 3/5/7 дней).
- Многошаговые формы — кнопка «Отмена».
- В `DONE` — удалять клавиатуру (чистый экран) и предлагать «Главное меню».

Пример удаления клавиатуры:

```go
func done(bot *tgbotapi.BotAPI, chatID int64, text string) {
    m := tgbotapi.NewMessage(chatID, text)
    m.ReplyMarkup = tgbotapi.NewRemoveKeyboard(true)
    _, _ = bot.Send(m)
}
```

### 10) Жизненный цикл и интеграция с общим обработчиком

В маршрутизаторе обновлений сначала проверяйте команды (`/onboarding`, `/menu`), иначе — делегируйте в `Machine.Handle`:

```go
func updatesLoop(bot *tgbotapi.BotAPI, m *fsm.Machine, updates tgbotapi.UpdatesChannel) {
    for upd := range updates {
        // глобальные команды вне FSM
        if msg := upd.Message; msg != nil && msg.IsCommand() {
            if msg.Command() == "start" { showMainMenu(bot, msg.Chat.ID); continue }
        }
        _ = m.Handle(context.Background(), bot, upd)
    }
}
```

### 11) Тестирование FSM

Юнит‑тесты шагов строятся на имитации `Update` и проверке возвращаемого `next` и побочных эффектов (сохранение состояния, формирование текста). Репозитории — мокать/фейки.

### 12) Памятка

- Один активный поток на пользователя; `user_id` — доменный `UserID` (строка/UUID) и выступает PK
- Всегда есть «Отмена» и «Главное меню»
- Промежуточные результаты — в `data JSONB`
- Долгие шаги — в фоне; состояние в БД
- Идемпотентность на уровне БД (UPSERT/уникальные ключи)
- Явные подсказки и ограничения ввода (клавиатуры)

---

Готово: у вас есть полный каркас FSM на Go для онбординга и составления меню, с персистентным хранением, обработкой отмены и восстановлением после сбоев, а также примерами шагов и маршрутизацией на основе `tgbotapi/v5` и PostgreSQL.

