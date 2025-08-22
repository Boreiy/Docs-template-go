## Telegram и UX (tgbotapi, polling, клавиатуры, шаблоны, идемпотентность)

Документ — практический и самодостаточный. Покрывает: подключение `tgbotapi/v5`, dev‑запуск на long polling, пайплайн обработки обновлений, команды и их семантика, reply/inline‑клавиатуры, шаблоны сообщений и форматирование, идемпотентность и дедупликацию, базовый rate limiting/backpressure, примеры кода.

### 1) Подключение `tgbotapi` и базовый запуск

Устанавливаем библиотеку:

```bash
go get github.com/go-telegram-bot-api/telegram-bot-api/v5
```

Минимальный запуск в dev через long polling:

```go
package main

import (
    "log"
    "os"
    tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"
)

func main() {
    token := os.Getenv("TELEGRAM_BOT_TOKEN")
    bot, err := tgbotapi.NewBotAPI(token)
    if err != nil { log.Fatal(err) }
    bot.Debug = true // dev

    u := tgbotapi.NewUpdate(0)
    u.Timeout = 30
    // limit update types to save traffic and simplify
    u.AllowedUpdates = []string{"message", "callback_query"}

    updates := bot.GetUpdatesChan(u)

    for update := range updates {
        if update.Message != nil {
            msg := tgbotapi.NewMessage(update.Message.Chat.ID, "Привет!")
            if _, err := bot.Send(msg); err != nil { log.Println("send:", err) }
        }
        if update.CallbackQuery != nil {
            // must answer to remove the loading indicator
            _, _ = bot.Request(tgbotapi.NewCallback(update.CallbackQuery.ID, ""))
        }
    }
}
```

Рекомендованный `Timeout` для long polling — 30 секунд. `AllowedUpdates` задаёт фильтрацию событий.

### 2) Пайплайн обработки апдейтов и конкуренция

Цели: (1) сохранить порядок событий внутри одного чата, (2) обрабатывать чаты параллельно, (3) не «утопить» процесс при наплыве.

Стандартный подход — шардированный воркер‑пул по `chat_id`:

```go
import tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"

type Update tgbotapi.Update

type Dispatcher struct {
    bot     *tgbotapi.BotAPI
    workers int
    chans   []chan Update
}

func abs(i int64) int64 {
    if i < 0 { return -i }
    return i
}

func NewDispatcher(bot *tgbotapi.BotAPI, workers int) *Dispatcher {
    d := &Dispatcher{bot: bot, workers: workers, chans: make([]chan Update, workers)}
    for i := 0; i < workers; i++ {
        d.chans[i] = make(chan Update, 100) // backpressure: buffer
        go d.worker(d.chans[i])
    }
    return d
}

func (d *Dispatcher) Dispatch(upd Update) {
    chatID := extractChatID(upd)
    if chatID == 0 { // e.g., service update
        d.chans[0] <- upd
        return
    }
    idx := int(abs(chatID)) % d.workers
    d.chans[idx] <- upd
}

func (d *Dispatcher) worker(in <-chan Update) {
    for upd := range in {
        handleUpdate(d.bot, upd) // sequential processing for the chat
    }
}

func extractChatID(u Update) int64 {
    if u.Message != nil { return u.Message.Chat.ID }
    if u.CallbackQuery != nil { return u.CallbackQuery.Message.Chat.ID }
    return 0
}
```

`buf=100` на канал — простейший backpressure: если нагрузка растёт быстрее обработки, буферы заполняются, upstream будет ждать (или вы можете дропать/логировать по политике).

### 3) Команды и их семантика

Базовый набор команд:

 - `/start` — приветствие и запуск онбординга для новых пользователей; после завершения или при повторном вызове — показ главного меню.
- `/help` — краткая справка по кнопкам.
- Дополнительно: `/profile`, `/menu`, `/cancel`.

Регистрация команд в Bot API (чтобы пользователи видели список в интерфейсе Telegram):

```go
import tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"

func setCommands(bot *tgbotapi.BotAPI) error {
    cmds := []tgbotapi.BotCommand{
        {Command: "start", Description: "Начать"},
        {Command: "help", Description: "Справка"},
        {Command: "profile", Description: "Профиль"},
        {Command: "menu", Description: "Составить меню"},
    }
    cfg := tgbotapi.NewSetMyCommands(cmds...)
    _, err := bot.Request(cfg)
    return err
}
```

Роутер команд:

```go
func routeCommand(bot *tgbotapi.BotAPI, msg *tgbotapi.Message) {
    switch msg.Command() {
    case "start": onStart(bot, msg)
    case "help": onHelp(bot, msg)
    case "profile": onProfile(bot, msg)
    case "menu": onMenu(bot, msg)
    default:
        reply(bot, msg.Chat.ID, "Неизвестная команда. Наберите /help")
    }
}

func handleUpdate(bot *tgbotapi.BotAPI, upd tgbotapi.Update) {
    if msg := upd.Message; msg != nil {
        if msg.IsCommand() { routeCommand(bot, msg); return }
        // ... handle plain text/media
    }
    if cb := upd.CallbackQuery; cb != nil { onCallback(bot, cb) }
}
```

Пример обёртки `reply` и заглушечного обработчика `onProfile`:

```go
import tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"

func reply(bot *tgbotapi.BotAPI, chatID int64, text string) {
    msg := tgbotapi.NewMessage(chatID, text)
    // ignore send errors for brevity
    _, _ = bot.Send(msg)
}

func onProfile(bot *tgbotapi.BotAPI, msg *tgbotapi.Message) {
    // stub handler
    reply(bot, msg.Chat.ID, "Профиль в разработке")
}
```

### 4) Reply‑клавиатуры (главное меню) и UX‑шаблоны

Создание главного меню и кнопки «Назад»/«Отмена»:

```go
import tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"

func mainMenu() tgbotapi.ReplyKeyboardMarkup {
    kb := tgbotapi.NewReplyKeyboard(
        tgbotapi.NewKeyboardButtonRow(
            tgbotapi.NewKeyboardButton("🍽 Составить меню"),
            tgbotapi.NewKeyboardButton("👤 Профиль"),
        ),
    )
    kb.ResizeKeyboard = true
    kb.OneTimeKeyboard = false
    return kb
}

func cancelKeyboard() tgbotapi.ReplyKeyboardMarkup {
    kb := tgbotapi.NewReplyKeyboard(
        tgbotapi.NewKeyboardButtonRow(tgbotapi.NewKeyboardButton("Отмена")),
    )
    kb.ResizeKeyboard = true
    return kb
}

func onStart(bot *tgbotapi.BotAPI, msg *tgbotapi.Message) {
    out := tgbotapi.NewMessage(msg.Chat.ID, "Добро пожаловать! Выберите действие:")
    out.ReplyMarkup = mainMenu()
    _, _ = bot.Send(out)
}

func removeKeyboard(bot *tgbotapi.BotAPI, chatID int64, text string) {
    out := tgbotapi.NewMessage(chatID, text)
    out.ReplyMarkup = tgbotapi.NewRemoveKeyboard(true)
    _, _ = bot.Send(out)
}
```

Рекомендации UX:
- На каждом многошаговом сценарии давать «Отмена».
- В длинных шагах показывать «печатает…»: `bot.Request(tgbotapi.NewChatAction(chatID, "typing"))`.
- «Главное меню» доступно из любого состояния.

### 5) Inline‑клавиатуры и callback‑обработчики

Пример inline‑кнопок «Нравится/Не нравится» и обработчик:

```go
import tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"

func feedbackKeyboard() tgbotapi.InlineKeyboardMarkup {
    return tgbotapi.NewInlineKeyboardMarkup(
        tgbotapi.NewInlineKeyboardRow(
            tgbotapi.NewInlineKeyboardButtonData("👍 Нравится", "fb:like"),
            tgbotapi.NewInlineKeyboardButtonData("👎 Не нравится", "fb:dislike"),
        ),
    )
}

func sendWithFeedback(bot *tgbotapi.BotAPI, chatID int64, text string) {
    m := tgbotapi.NewMessage(chatID, text)
    m.ReplyMarkup = feedbackKeyboard()
    _, _ = bot.Send(m)
}

func onCallback(bot *tgbotapi.BotAPI, cb *tgbotapi.CallbackQuery) {
    // stop the loading indicator
    _, _ = bot.Request(tgbotapi.NewCallback(cb.ID, ""))
    var answer string
    switch cb.Data {
    case "fb:like": answer = "Спасибо за лайк!"
    case "fb:dislike": answer = "Принято, постараемся улучшить."
    default: answer = "Неизвестное действие"
    }
    edit := tgbotapi.NewEditMessageText(cb.Message.Chat.ID, cb.Message.MessageID, answer)
    _, _ = bot.Request(edit)
}
```

### 6) Шаблоны сообщений и форматирование

Для повторно используемых сообщений удобно иметь слой шаблонов. Рекомендуется использовать `HTML`‑парсинг (проще, меньше экранирования), либо `MarkdownV2` (требует тщательного экранирования).

```go
import tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"

func msgHTML(chatID int64, html string) tgbotapi.MessageConfig {
    m := tgbotapi.NewMessage(chatID, html)
    m.ParseMode = tgbotapi.ModeHTML
    return m
}

func onHelp(bot *tgbotapi.BotAPI, msg *tgbotapi.Message) {
    html := "<b>Справка</b>\n\n" +
        "• /menu — составить меню\n" +
        "• /profile — профиль"
    _, _ = bot.Send(msgHTML(msg.Chat.ID, html))
}
```

Если используете `MarkdownV2`, обязательно экранируйте символы (\_\*\[\]\(\)~`>#+-=|{}.!):

```go
import "strings"

func escapeMDv2(s string) string {
    repl := strings.NewReplacer("_", "\\_", "*", "\\*", "[", "\\[", "]", "\\]",
        "(", "\\(", ")", "\\)", "~", "\\~", "`", "\\`", ">", "\\>", "#", "\\#",
        "+", "\\+", "-", "\\-", "=", "\\=", "|", "\\|", "{", "\\{", "}", "\\}", ".", "\\.", "!", "\\!")
    return repl.Replace(s)
}
```

После экранирования нужно явно выбрать режим `MarkdownV2`, иначе Telegram воспримет сообщение как обычный текст и проигнорирует форматирование:

```go
msg := tgbotapi.NewMessage(chatID, escapeMDv2("*bold* _italic_"))
msg.ParseMode = "MarkdownV2" // set MarkdownV2 parse mode
if _, err := bot.Send(msg); err != nil {
    log.Println("send:", err)
}
```

### 7) Идемпотентность и дедупликация

Telegram может ретраить доставку webhook‑запросов; при polling — подтверждение через `offset`.

- Webhook: хранить `update_id` в краткоживущем кеше (LRU/TTL). При повторе — игнорировать.
- Команды с побочными эффектами делать идемпотентными (уникальные ключи в БД, `INSERT ... ON CONFLICT DO NOTHING`).

Пример простого in‑memory TTL‑кеша (10 минут) см. в `dev-docs/architecture.md` (раздел Telegram).

### 8) Rate limiting и anti‑flood

Простейшая защита “не чаще, чем N раз в секунду на пользователя”:

```go
import (
    "sync"
    "time"
)

type RateLimiter struct {
    mu sync.Mutex
    last map[int64]time.Time
    rate time.Duration
}

func (r *RateLimiter) Allow(userID int64) bool {
    r.mu.Lock(); defer r.mu.Unlock()
    now := time.Now()
    if t, ok := r.last[userID]; ok && now.Sub(t) < r.rate { return false }
    r.last[userID] = now
    return true
}
```

Использование:

```go
if msg := upd.Message; msg != nil {
    if !rl.Allow(msg.From.ID) {
        // quietly ignore or reply "Too frequent"
        return
    }
}
```

Для нагрузок повыше — воркер‑пул (см. §2) и ограничение параллелизма: `semaphore.Weighted`.

### 9) Обработка ошибок и ретраи отправки

Любой вызов `bot.Send` / `bot.Request` может вернуть сетевые ошибки/429. Политика:
- Повторить с экспоненциальным backoff (до 2–3 попыток), при 429 учитывать `Retry-After` (если есть).
- Логировать повод и контекст (chat_id, update_id), не логировать PII.

```go
import (
    "time"

    tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"
)

func sendSafe(bot *tgbotapi.BotAPI, cfg tgbotapi.Chattable) error {
    var last error
    for i := 0; i < 3; i++ {
        if _, err := bot.Send(cfg); err != nil {
            last = err
            time.Sleep(time.Duration(300*(1<<i)) * time.Millisecond)
            continue
        }
        return nil
    }
    return last
}
```

### 10) Пример: обработчик «Составить меню» с клавиатурами

```go
import tgbotapi "github.com/go-telegram-bot-api/telegram-bot-api/v5"

func onMenu(bot *tgbotapi.BotAPI, msg *tgbotapi.Message) {
    bot.Request(tgbotapi.NewChatAction(msg.Chat.ID, "typing"))
    // step 1: ask for number of days
    ask := tgbotapi.NewMessage(msg.Chat.ID, "На сколько дней составить меню?")
    ask.ReplyMarkup = tgbotapi.NewReplyKeyboard(
        tgbotapi.NewKeyboardButtonRow(
            tgbotapi.NewKeyboardButton("3"),
            tgbotapi.NewKeyboardButton("5"),
            tgbotapi.NewKeyboardButton("7"),
        ),
        tgbotapi.NewKeyboardButtonRow(tgbotapi.NewKeyboardButton("Отмена")),
    )
    _, _ = bot.Send(ask)
    // then FSM: on next incoming message from this chat parse the choice and proceed
}
```

### 11) UX‑гайдлайн для бота

- Сообщения короткие, одна мысль — одно сообщение.
- Всегда есть явный следующий шаг (кнопка) и «Отмена».
- Подтверждение успеха/ошибки — простое и дружелюбное.
- Не спамить сообщениями: объединять ответы, редактировать прошлые, использовать inline‑клавиатуры.
- Всегда можно вернуться в «Главное меню».

### 12) Webhook (prod) — кратко

В проде вместо polling используем вебхук на `Gin`. Обязательно проверять заголовок `X-Telegram-Bot-Api-Secret-Token`. Мини‑пример см. в `dev-docs/architecture.md`.

---

Готово: у вас есть полный шаблон UX‑слоя для Telegram‑бота на Go: запуск, пайплайн, команды, клавиатуры, шаблоны сообщений, идемпотентность и антифлуд. Все примеры совместимы с `github.com/go-telegram-bot-api/telegram-bot-api/v5`.

