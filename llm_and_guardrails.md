## LLM и guardrails (Go): схемы JSON, промпты, модерация, ретраи/таймауты

Документ — самодостаточный. Покрывает: конфигурацию клиента, вызовы OpenAI Chat Completions c `response_format: json_schema` (strict) для «черновика» и «финалки», tool/function calling (strict) для чтения профиля, guardrails через `nlpodyssey/openai-agents-go` (input/output, модерация, in‑scope, запрет аллергенов), рероллы, политику таймаутов и ретраев. В примерах используется генерация меню; подставьте свой домен при адаптации. Примеры на Go через `net/http` или `resty`.

### 1) Конфигурация и ENV

- `OPENAI_API_KEY` — обязательный ключ.
- `OPENAI_BASE_URL` — опционально, если используете proxy/Azure совместимый endpoint (например: `https://api.openai.com/v1` по умолчанию).
 - `OPENAI_MODEL_DRAFT` — модель для черновиков (→ `Config.OpenAI.ModelDraft`; рекомендуется `gpt-4o-mini-2024-07-18`; без даты — `gpt-4o-mini`).
 - `OPENAI_MODEL_FINAL` — модель для финализации (→ `Config.OpenAI.ModelFinal`; рекомендуется `gpt-4o-2024-11-20`; без даты — `gpt-4o`).

Где хранится: `internal/config/config.go` и валидация через `go-playground/validator`. Не логировать ключи.

Суффикс даты в идентификаторе (`-2024-11-20`) фиксирует конкретную версию модели. Алиасы без даты автоматически используют послед
нее обновление; удобно в dev, но в prod может привести к непредсказуемым изменениям.

### 2) HTTP‑клиент и ретраи

Создавайте клиент с полным набором таймаутов и ограниченными ретраями. Можно `net/http` или `resty`. Пример на `resty` (v2):

```go
import (
    "time"
    "github.com/go-resty/resty/v2"
)

func NewOpenAIClient(baseURL, apiKey string) *resty.Client {
    c := resty.New()
    c.SetBaseURL(baseURL).
        SetHeader("Authorization", "Bearer "+apiKey).
        SetHeader("Content-Type", "application/json").
        SetTimeout(20 * time.Second).
        SetRetryCount(3).
        SetRetryWaitTime(300 * time.Millisecond).
        SetRetryMaxWaitTime(3 * time.Second).
        AddRetryCondition(func(r *resty.Response, err error) bool {
            if err != nil { return true }
            sc := r.StatusCode()
            return sc == 429 || sc == 502 || sc == 503 || sc == 504
        })
    return c
}
```

Замечания:
- Учитывайте `Retry-After` заголовок при 429 (при желании — кастомный `SetRetryAfter`).
- Для долгих операций используйте context с таймаутом и фоновые goroutine.

### 3) Контракты данных (DTO) для ответа LLM

Черновик и финалка доменного объекта (пример: меню) фиксируются схемами (те же поля в Go‑DTO). Пример:

```go
type MealType string

const (
    MealBreakfast MealType = "breakfast"
    MealLunch     MealType = "lunch"
    MealDinner    MealType = "dinner"
    MealSnack     MealType = "snack"
)

type LlmMenuItem struct {
    DayIndex int      `json:"day_index" validate:"gte=1,lte=14"`
    MealType MealType `json:"meal_type" validate:"oneof=breakfast lunch dinner snack"`
    Title    string   `json:"title" validate:"min=1"`
    Steps    []string `json:"steps" validate:"min=1,dive,min=1"`
}

type LlmDraftMenu struct {
    Items  []LlmMenuItem `json:"items" validate:"min=1,dive"`
    Reused []string      `json:"reused"`
}

type LlmFinalMenu struct {
    Items  []LlmMenuItem `json:"items" validate:"min=1,dive"`
    Reused []string      `json:"reused"`
}
```

Пример: структура для парсинга свободного фидбэка:

```go
type FreeformFeedback struct {
    LikedTitles    []string `json:"liked_titles"`
    DislikedTitles []string `json:"disliked_titles"`
    RestPolicy     string   `json:"rest_policy"` // rest_liked|rest_disliked|unspecified
}
```

### 4) JSON Schema для Structured Outputs (strict)

Используем Chat Completions с `response_format: {type:"json_schema", json_schema: {name, strict:true, schema}}`. Пример схемы для черновика:

```json
{
  "type": "object",
  "properties": {
    "items": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "day_index": {"type": "integer", "minimum": 1, "maximum": 14},
          "meal_type": {"type": "string", "enum": ["breakfast","lunch","dinner","snack"]},
          "title": {"type": "string"},
          "steps": {"type": "array", "items": {"type": "string"}, "minItems": 1}
        },
        "required": ["day_index","meal_type","title","steps"],
        "additionalProperties": false
      },
      "minItems": 1
    },
    "reused": {"type": "array", "items": {"type": "string"}}
  },
  "required": ["items","reused"],
  "additionalProperties": false
}
```

Замечания:
- В strict‑режиме избегайте `anyOf` у корня; делайте поля явными и required.
- Для «опциональных» значений при необходимости используйте `type: ["string","null"]` (не для параметров tools в некоторых моделях, там лучше избегать union — деталь API).

### 5) Вызов Chat Completions (Go, resty)
Схему из раздела 4 нужно заранее подготовить и обернуть в `json.RawMessage` — пример `draftSchema` ниже показывает, как это сделать.

```go
type ChatMessage struct{ Role, Content string }

type JsonSchema struct {
    Name   string      `json:"name"`
    Strict bool        `json:"strict"`
    Schema interface{} `json:"schema"`
}

type ResponseFormat struct {
    Type       string     `json:"type"` // "json_schema"
    JsonSchema JsonSchema `json:"json_schema"`
}

type ChatRequest struct {
    Model          string        `json:"model"`
    Messages       []ChatMessage `json:"messages"`
    ResponseFormat ResponseFormat `json:"response_format"`
    // optional: temperature/top_p/max_tokens
}

type Choice struct {
    Message struct { Content string `json:"content"` } `json:"message"`
}
type ChatResponse struct{ Choices []Choice `json:"choices"` }

// JSON schema (see section 4)
var draftSchema = json.RawMessage(`{
    "type": "object",
    "properties": {
        "items": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "day_index": {"type": "integer", "minimum": 1, "maximum": 14},
                    "meal_type": {"type": "string", "enum": ["breakfast","lunch","dinner","snack"]},
                    "title": {"type": "string"},
                    "steps": {"type": "array", "items": {"type": "string"}, "minItems": 1}
                },
                "required": ["day_index","meal_type","title","steps"],
                "additionalProperties": false
            },
            "minItems": 1
        },
        "reused": {"type": "array", "items": {"type": "string"}}
    },
    "required": ["items","reused"],
    "additionalProperties": false
}`)

func CallDraft(ctx context.Context, c *resty.Client, model string, msgs []ChatMessage, draftSchema interface{}) (LlmDraftMenu, error) {
    req := ChatRequest{
        Model:    model,
        Messages: msgs,
        ResponseFormat: ResponseFormat{Type: "json_schema", JsonSchema: JsonSchema{
            Name:   "draft_menu",
            Strict: true,
            Schema: draftSchema,
        }},
    }
    var out ChatResponse
    resp, err := c.R().SetContext(ctx).SetBody(req).SetResult(&out).Post("/chat/completions")
    if err != nil { return LlmDraftMenu{}, err }
    if resp.IsError() { return LlmDraftMenu{}, fmt.Errorf("openai: %s", resp.Status()) }
    var draft LlmDraftMenu
    if err := json.Unmarshal([]byte(out.Choices[0].Message.Content), &draft); err != nil { return LlmDraftMenu{}, err }
    return draft, nil
}
```

Переменная `draftSchema` из примера выше хранит JSON Schema и передаётся в `CallDraft`.

Примечания:
- `Content` вернётся строго соответствующим схеме JSON (strict). Всё равно валидируйте на стороне Go (`validator`).
- Для Azure требуется параметр версии API (`api-version=2024-10-21`) и собственный endpoint. Тело запроса совместимо.

### 6) Промпты и инварианты (draft/final)

Системное сообщение должно задавать инварианты: учесть аллергии, «чего хочется», `pantry`, политику повторов; запрещённые ингредиенты — жёсткое требование. Пример:

```go
func DraftSystem(allergens, desires []string, pantry []string, reuseLeftovers bool, maxRepeats, repeatWindow int) string {
    return fmt.Sprintf(
        "Вы диетолог. Составьте меню. Требования: 1) НЕ использовать аллергены: %v. 2) Учесть пожелания: %v. 3) По возможности использовать: %v. 4) Повторы блюд: reuse_leftovers=%t, max_repeats=%d, repeat_window_days=%d. Возвращайте только JSON по схеме draft_menu.",
        allergens, desires, pantry, reuseLeftovers, maxRepeats, repeatWindow,
    )
}
```

Сами `messages`:

```go
msgs := []ChatMessage{
  {Role: "system", Content: DraftSystem(allergies, desires, pantry, true, 0, 0)},
  {Role: "user", Content: "Сгенерируй черновик меню на 5 дней."},
}
```

Для финалки используйте аналогичное системное сообщение, но потребуйте консолидацию всего фидбэка и точное следование запретам.

### 7) Function/Tool calling (strict) для профиля

В простом варианте профиль подтягивается приложением и передаётся в промпте. Если решите использовать tools — задайте параметры функции c `strict: true`:

```json
{
  "type": "function",
  "function": {
    "name": "get_user_profile",
    "description": "Return user profile with allergies/likes/dislikes/goal",
    "strict": true,
    "parameters": {
      "type": "object",
      "properties": {"user_id": {"type": "string"}},
      "required": ["user_id"],
      "additionalProperties": false
    }
  }
}
```

Ограничение: параллельные tool calls часто несовместимы со strict‑outputs — держите `parallel_tool_calls=false`.

### 8) Input guardrails (до вызова модели)

Используем `nlpodyssey/openai-agents-go` для проверки запросов перед запуском основного агента. Guardrails выполняются параллельно; при срабатывании возвращается `InputGuardrailTripwireTriggeredError`.

Пример guardrail, который блокирует запросы вне тематики еды и меню:

```go
import (
    "context"
    "fmt"

    "github.com/nlpodyssey/openai-agents-go/agents"
)

// scopeCheckOutput describes classification result.
type scopeCheckOutput struct {
    Reason  string `json:"reason"`
    InScope bool   `json:"in_scope"`
}

var scopeCheckAgent = agents.New("Scope check").
    WithInstructions("Определи, относится ли сообщение к еде, рецептам или меню. Ответь JSON {\"in_scope\": bool}.").
    WithOutputType(agents.OutputType[scopeCheckOutput]()).
    WithModel("gpt-4o-mini")

func scopeGuardrail(ctx context.Context, _ *agents.Agent, in agents.Input) (agents.GuardrailFunctionOutput, error) {
    var (
        res *agents.RunResult
        err error
    )
    switch v := in.(type) {
    case agents.InputString:
        res, err = agents.Run(ctx, scopeCheckAgent, v.String())
    case agents.InputItems:
        res, err = agents.RunInputs(ctx, scopeCheckAgent, v)
    default:
        panic(fmt.Errorf("unexpected input type %T", v))
    }
    if err != nil {
        return agents.GuardrailFunctionOutput{}, err
    }
    out := res.FinalOutput.(scopeCheckOutput)
    return agents.GuardrailFunctionOutput{TripwireTriggered: !out.InScope}, nil
}

var InScopeGuardrail = agents.InputGuardrail{
    GuardrailFunction: scopeGuardrail,
    Name:              "in_scope",
}
```

Guardrail подключается к основному агенту. Аналогично можно добавить ещё один guardrail, вызывающий `/v1/moderations` для фильтрации токсичного контента.

```go
agent := agents.New("Menu draft").
    WithInstructions("Составь меню по требованиям пользователя.").
    WithInputGuardrails([]agents.InputGuardrail{InScopeGuardrail}).
    WithModel("gpt-4o-mini")

result, err := agents.Run(ctx, agent, userPrompt)
```

#### Список аллергенов и `must_not_include`

Список аллергенов пользователя хранится в профиле в базе данных (например, поле `allergies` таблицы `user_profile`). Глобальный перечень поддерживаемых аллергенов может лежать в конфиге `configs/allergens.yaml`. Адаптер при подготовке запроса объединяет эти данные в срез `forbidden`.

Этот срез передаётся в поле `must_not_include` объекта `response_format.json_schema`, чтобы модель не использовала перечисленные ингредиенты.

```go
forbidden := append([]string{}, input.Allergies...)
body := map[string]any{
    "model": model,
    "messages": messages,
    "response_format": map[string]any{
        "type": "json_schema",
        "json_schema": map[string]any{
            "name": "draft_menu",
            "schema": draftSchema,
            "strict": true,
            "must_not_include": forbidden,
        },
    },
}
```

Здесь `messages` — стандартные system/user промпты; `draftSchema` — JSON Schema из раздела 4.

### 9) Output guardrails (после вызова модели)

1) Валидация структуры: `validator.Struct(draft)`.
2) Аллерген‑чек через `OutputGuardrail`: запрещённые ингредиенты не должны встречаться ни в `title`, ни в `steps`. При нарушении — регенерация c усиленным указанием `must_not_include: [...]`.

```go
import (
    "context"
    "strings"

    "github.com/nlpodyssey/openai-agents-go/agents"
)

// ContainsForbidden returns true if text contains any forbidden item.
func ContainsForbidden(text string, forbidden []string) bool {
    low := strings.ToLower(text)
    for _, f := range forbidden {
        if strings.Contains(low, strings.ToLower(f)) {
            return true
        }
    }
    return false
}

// DraftAllergenCheck validates menu against forbidden list.
func DraftAllergenCheck(d LlmDraftMenu, forbidden []string) bool {
    for _, it := range d.Items {
        if ContainsForbidden(it.Title, forbidden) {
            return false
        }
        for _, s := range it.Steps {
            if ContainsForbidden(s, forbidden) {
                return false
            }
        }
    }
    return true
}

func allergenGuardrail(ctx context.Context, _ *agents.Agent, out any) (agents.GuardrailFunctionOutput, error) {
    menu := out.(LlmDraftMenu)
    ok := DraftAllergenCheck(menu, []string{"peanut", "shrimp"})
    return agents.GuardrailFunctionOutput{TripwireTriggered: !ok}, nil
}

var AllergenGuardrail = agents.OutputGuardrail{
    GuardrailFunction: allergenGuardrail,
    Name:              "allergen_check",
}

menuAgent := agents.New("Menu draft").
    WithInstructions("Составь меню без запрещённых ингредиентов.").
    WithOutputType(agents.OutputType[LlmDraftMenu]()).
    WithOutputGuardrails([]agents.OutputGuardrail{AllergenGuardrail}).
    WithModel("gpt-4o-mini")

res, err := agents.Run(ctx, menuAgent, userPrompt)
```

Политика: до 2–3 регенераций, затем сообщить пользователю, что нужная комбинация невозможна с текущими ограничениями.

#### Пример input guardrail (перед вызовом модели)

Input guardrail позволяет отфильтровать неуместные или запрещённые запросы ещё до обращения к модели. Ниже показана простая проверка на тематичность: если пользователь упоминает запрещённую тему, запрос блокируется.

```go
import (
    "context"
    "errors"
    "strings"

    "github.com/nlpodyssey/openai-agents-go/agents"
)

// inScopeGuardrail blocks prompts outside allowed topics.
func inScopeGuardrail(ctx context.Context, _ *agents.Agent, in any) (agents.GuardrailFunctionOutput, error) {
    text := strings.ToLower(in.(string))
    if strings.Contains(text, "инвестиции") {
        return agents.GuardrailFunctionOutput{TripwireTriggered: true}, nil
    }
    return agents.GuardrailFunctionOutput{TripwireTriggered: false}, nil
}

var ScopeGuardrail = agents.InputGuardrail{
    GuardrailFunction: inScopeGuardrail,
    Name:              "in_scope",
}

menuAgent := agents.New("Menu draft").
    WithInstructions("Составь меню без запрещённых ингредиентов.").
    WithInputGuardrails([]agents.InputGuardrail{ScopeGuardrail}).
    WithOutputGuardrails([]agents.OutputGuardrail{AllergenGuardrail}).
    WithOutputType(agents.OutputType[LlmDraftMenu]()).
    WithModel("gpt-4o-mini")

res, err := agents.Run(ctx, menuAgent, userPrompt)
if errors.Is(err, agents.ErrTripwireTriggered) {
    // Input was flagged: return friendly error or ask for another prompt
    return
}
```

Если guardrail срабатывает, обработайте `agents.ErrTripwireTriggered`: покажите пользователю дружелюбную ошибку или предложите изменить запрос.

### 10) Рероллы и таймауты

- Реролл: повторный вызов с тем же промптом/контекстом; лимит попыток и экспоненциальный бэкофф.
- Таймауты: `context.WithTimeout` на каждый вызов (напр. 20–30 с). При `DeadlineExceeded` — единый дружелюбный ответ в UI и фоновая попытка (если уместно).

### 11) Интерфейс gateway и адаптер

Пример входной структуры для генерации черновика меню, валидируемой `go-playground/validator` и соответствующей доменным ограничениям:

```go
type ComposeInput struct {
    Goal      string   `json:"goal"`      // dietary goal
    Likes     []string `json:"likes"`     // preferred ingredients
    Dislikes  []string `json:"dislikes"`  // ingredients to avoid
    Allergies []string `json:"allergies"` // allergens
    Desires   []string `json:"desires"`   // user wishes
    Pantry    []string `json:"pantry"`    // available ingredients
    Days      int      `json:"days" validate:"gte=1,lte=14"`      // number of days to plan (1-14)
}
```

Где (пример меню):
- `Goal` — цель питания;
- `Likes` — предпочтительные продукты;
- `Dislikes` — то, что нужно избегать;
- `Allergies` — список аллергенов;
- `Desires` — дополнительные пожелания;
- `Pantry` — продукты, уже имеющиеся у пользователя;
- `Days` — длительность меню в днях (1–14).

Такая структура гарантирует соблюдение доменных ограничений на диапазон дней.

Интерфейс (слой `usecase`):

```go
type LLMGateway interface {
    ComposeDraft(ctx context.Context, input ComposeInput) (LlmDraftMenu, error)
    ApplyFeedback(ctx context.Context, input FeedbackInput) (LlmDraftMenu, error)
    Finalize(ctx context.Context, menu LlmDraftMenu) (LlmFinalMenu, error)
}

В `FeedbackInput` вместе с черновиком и фидбэком передаются цель, предпочтения и аллергии пользователя, чтобы исключить запрещённые блюда при редактировании.
```

Адаптер `internal/adapter/external/openai` реализует этот интерфейс, инкапсулирует: сбор `messages`, подстановку схемы, вызов `/chat/completions`, валидацию (`validator`) и guardrails (allergen‑чек, регенерация). Доменные сущности/репозитории остаются изолированы.

### 12) Политика логирования и приватности

- Логировать только метаданные: `request_id`, `chat_id`, `menu_id`, статус, длительность. Не логировать промпты/ответы целиком; допускается усечь до хешей/первых 50 символов.
- Секреты — никогда в логи.

### 13) E2E‑пример: черновик → фидбэк → финалка (меню)

1) Telegram FSM получает от пользователя: `days`, `desires`, `pantry` → сбор `allergies/likes/dislikes/goal` из профиля.
2) Input guardrails через `agents.InputGuardrail` (in‑scope, модерация).
3) LLM `ComposeDraft` с JSON Schema → парсинг и валидация → allergen‑чек → при нарушении регенерация.
4) Пользователь присылает свободный фидбэк → отдельный запрос к модели со схемой `FreeformFeedback` (strict) для нормализации → применяем к черновику и сохраняем.
5) Финализация: `Finalize` со схемой финалки; проверка повторов/аллергенов; сохранение и отправка пользователю.

### 14) Тестирование

- Юнит‑тесты адаптера: поднимайте `httptest` сервер, эмулируйте OpenAI ответы. Не ходите во внешнюю сеть.
- Табличные тесты на провалы: 429/5xx, таймаут, невалидный JSON, нарушение аллергенов, реролл.

### 15) Быстрые шаблоны

- Безопасный ответ пользователю при блокировке/ошибке: «Сейчас не получается обработать запрос. Попробуйте ещё раз, либо измените параметры (например, уберите часть ограничений).»
- Усиление запретов в регенерации: «must_not_include: …; если рецепт подразумевает скрытые ингредиенты, замените их безопасными альтернативами.»

---

Чек‑лист внедрения
- Конфиг и клиент OpenAI с таймаутами/ретраями готовы
- Схемы JSON для draft/final/feedback заданы и валидируются
- Input guardrails через `agents.InputGuardrail` (in‑scope + moderation)
- Output guardrails через `agents.OutputGuardrail` (аллерген‑чек + регенерации)
- Gateway/адаптер реализован и покрыт тестами
- Логи без PII; ошибки дружелюбны для пользователя

