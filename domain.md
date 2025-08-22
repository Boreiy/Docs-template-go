## Домен и репозитории (сущности, инварианты, порты)

Документ — самодостаточный. Описывает подход к определению доменных сущностей, инвариантов, value objects и портов (репозитории, транзакции) в независимом от инфраструктуры виде. В примерах используется сервис составления меню; замените эти сущности на свои при адаптации.

### 1) Базовые принципы

- Домен размещается в `internal/domain/` и не импортирует внешние библиотеки (ни `pgx`, ни `validator`).
- Все инварианты проверяются в конструкторах/методах домена; публичные конструкторы возвращают ошибку.
- Ошибки — значения. Используйте обёртку `%w` и `errors.Is/As` для маппинга на инфраструктурные причины (not found, unique violation и т.д.).
- Типы идентификаторов — отдельные типы (`UserID`, `MenuID`), чтобы исключить случайные подстановки.

```go
package domain

import (
    "errors"
    "fmt"
    "strings"
    "time"
)

// Identifiers (may be stored as UUID in DB)
type UserID string
type MenuID string
type PantryItemID string

// Domain errors (stable for upper layers)
var (
    ErrNotFound          = errors.New("not found")
    ErrInvariantViolated = errors.New("invariant violated")
)

// Helper wrapper for invariants
func invariant(cond bool, msg string) error {
    if cond { return nil }
    return fmt.Errorf("%w: %s", ErrInvariantViolated, msg)
}

// Normalize tags (allergies/likes/dislikes)
func NormalizeTag(s string) string {
    return strings.ToLower(strings.TrimSpace(s))
}
```

### 2) Value‑objects и перечисления

Пример: перечисление типа приёма пищи для планирования меню.

```go
package domain

type MealType string

const (
    MealBreakfast MealType = "breakfast"
    MealLunch     MealType = "lunch"
    MealDinner    MealType = "dinner"
    MealSnack     MealType = "snack"
)

func (m MealType) Valid() bool {
    switch m {
    case MealBreakfast, MealLunch, MealDinner, MealSnack:
        return true
    default:
        return false
    }
}
```

### 3) Сущности и инварианты

Пример: пользователь, его профиль, кладовая и план меню.

```go
package domain

type User struct {
    ID        UserID
    TelegramID int64
    CreatedAt time.Time
}

func NewUser(id UserID, tgID int64, now time.Time) (User, error) {
    if err := invariant(id != "", "empty user id"); err != nil { return User{}, err }
    if err := invariant(tgID != 0, "empty telegram id"); err != nil { return User{}, err }
    return User{ID: id, TelegramID: tgID, CreatedAt: now.UTC()}, nil
}

type UserProfile struct {
    UserID    UserID
    Goal      string
    Likes     []string
    Dislikes  []string
    Allergies []string
    UpdatedAt time.Time
}

func NewUserProfile(uid UserID, goal string, likes, dislikes, allergens []string, now time.Time) (UserProfile, error) {
    if err := invariant(uid != "", "empty user id"); err != nil { return UserProfile{}, err }
    nl := normalizeSet(likes)
    nd := normalizeSet(dislikes)
    na := normalizeSet(allergens)
    // simple invariant: set without duplicates, no empty strings
    return UserProfile{UserID: uid, Goal: strings.TrimSpace(goal), Likes: nl, Dislikes: nd, Allergies: na, UpdatedAt: now.UTC()}, nil
}

func normalizeSet(in []string) []string {
    seen := map[string]struct{}{}
    out := make([]string, 0, len(in))
    for _, s := range in {
        t := NormalizeTag(s)
        if t == "" { continue }
        if _, ok := seen[t]; ok { continue }
        seen[t] = struct{}{}
        out = append(out, t)
    }
    return out
}

type PantryItem struct {
    ID      PantryItemID
    UserID  UserID
    Name    string
    Qty     string // free form: "2 eggs", "200 g"
    Updated time.Time
}

func NewPantryItem(id PantryItemID, uid UserID, name, qty string, now time.Time) (PantryItem, error) {
    if err := invariant(id != "", "empty pantry id"); err != nil { return PantryItem{}, err }
    if err := invariant(uid != "", "empty user id"); err != nil { return PantryItem{}, err }
    if err := invariant(strings.TrimSpace(name) != "", "empty name"); err != nil { return PantryItem{}, err }
    return PantryItem{ID: id, UserID: uid, Name: strings.TrimSpace(name), Qty: strings.TrimSpace(qty), Updated: now.UTC()}, nil
}

type MenuStatus string

const (
    MenuDraft MenuStatus = "draft"
    MenuFinal MenuStatus = "final"
)

type MenuPlan struct {
    ID          MenuID
    UserID      UserID
    Status      MenuStatus
    Days        int
    CreatedAt   time.Time
    FinalizedAt *time.Time
}

func NewMenuDraft(id MenuID, uid UserID, days int, now time.Time) (MenuPlan, error) {
    if err := invariant(id != "", "empty menu id"); err != nil { return MenuPlan{}, err }
    if err := invariant(uid != "", "empty user id"); err != nil { return MenuPlan{}, err }
    if err := invariant(days >= 1 && days <= 14, "days must be within 1..14"); err != nil { return MenuPlan{}, err }
    return MenuPlan{ID: id, UserID: uid, Status: MenuDraft, Days: days, CreatedAt: now.UTC()}, nil
}

type MenuItem struct {
    MenuID    MenuID
    DayIndex  int    // 1..MenuPlan.Days
    MealType  MealType
    Title     string
    Steps     []string // short steps
}

func NewMenuItem(menuID MenuID, day int, meal MealType, title string, steps []string) (MenuItem, error) {
    if err := invariant(menuID != "", "empty menu id"); err != nil { return MenuItem{}, err }
    if err := invariant(day >= 1, "day index must be >=1"); err != nil { return MenuItem{}, err }
    if err := invariant(meal.Valid(), "invalid meal type"); err != nil { return MenuItem{}, err }
    if err := invariant(strings.TrimSpace(title) != "", "empty title"); err != nil { return MenuItem{}, err }
    if err := invariant(len(steps) > 0, "empty steps"); err != nil { return MenuItem{}, err }
    return MenuItem{MenuID: menuID, DayIndex: day, MealType: meal, Title: strings.TrimSpace(title), Steps: copyNonEmpty(steps)}, nil
}

func copyNonEmpty(in []string) []string { 
    out := make([]string, 0, len(in))
    for _, s := range in { t := strings.TrimSpace(s); if t != "" { out = append(out, t) } }
    return out
}

type MenuFeedback struct {
    MenuID   MenuID
    DayIndex int
    MealType MealType
    Liked    bool
    Comment  string
}

func NewFeedback(menuID MenuID, day int, meal MealType, liked bool, comment string) (MenuFeedback, error) {
    if err := invariant(menuID != "", "empty menu id"); err != nil { return MenuFeedback{}, err }
    if err := invariant(day >= 1, "day index must be >=1"); err != nil { return MenuFeedback{}, err }
    if err := invariant(meal.Valid(), "invalid meal type"); err != nil { return MenuFeedback{}, err }
    return MenuFeedback{MenuID: menuID, DayIndex: day, MealType: meal, Liked: liked, Comment: strings.TrimSpace(comment)}, nil
}
```

Примечания по инвариантам:
- `MenuItem.DayIndex` должен находиться в пределах `1..MenuPlan.Days` — это проверяется на уровне приложения при добавлении элементов к плану (требуется доступ к `MenuPlan.Days`).
- Проверки аллергенов — ответственность уровня приложения/guardrails (см. `llm_and_guardrails.md`), но доменная модель допускает вспомогательные проверки, если профиль пользователя известен.

### 4) Порты (интерфейсы репозиториев и транзакций)

```go
package domain

import "context"

// UserRepo handles users
type UserRepo interface {
    GetOrCreateByTelegramID(ctx context.Context, tgUserID int64) (User, error)
}

// ProfileRepo handles user profile
type ProfileRepo interface {
    Get(ctx context.Context, userID UserID) (*UserProfile, error)
    Upsert(ctx context.Context, profile UserProfile) error
}

// PantryRepo handles user pantry
type PantryRepo interface {
    List(ctx context.Context, userID UserID) ([]PantryItem, error)
    ReplaceAll(ctx context.Context, userID UserID, items []PantryItem) error
}

// MenuView is a convenient snapshot of menu for reading
type MenuView struct {
    Plan  MenuPlan
    Items []MenuItem
}

// MenuRepo manages menu plans and their content
type MenuRepo interface {
    CreateDraft(ctx context.Context, userID UserID, days int) (MenuID, error)
    ReplaceItems(ctx context.Context, menuID MenuID, items []MenuItem) error
    AddFeedback(ctx context.Context, menuID MenuID, fb []MenuFeedback) error
    Finalize(ctx context.Context, menuID MenuID) error
    GetCurrent(ctx context.Context, userID UserID) (*MenuView, error)
}

// TxRunner is a port for transactional use-case execution
type TxRunner interface {
    WithinTx(ctx context.Context, fn func(ctx context.Context) error) error
}
```

Реализация PostgreSQL кладёт `pgx.Tx` в `context.Context`, чтобы репозитории могли использовать одну транзакцию без явной передачи объекта.

Рекомендации:
- Возвращайте `ErrNotFound` из реализаций репозиториев, когда запись не найдена. Слои выше используют `errors.Is(err, domain.ErrNotFound)`.
- Для конфликтов уникальности используйте оборачивание с контекстом: `fmt.Errorf("create draft: %w", err)` на уровне адаптера БД, а маппинг в доменную ошибку — там же.

### 5) Правила ошибок (Go, актуально)

- Используйте `%w` при обёртке: `fmt.Errorf("profile upsert: %w", err)`; проверяйте через `errors.Is/As` (Go 1.13+).
- Sentinel‑ошибки (`ErrNotFound`) — уместны как стабильный контракт домена; конкретику БД не «протекаем» наружу.
- Не сравнивайте строки ошибок; избегайте экспонирования инфраструктурных текстов.

### 6) Примеры использования домена

Создание черновика меню и элементов с проверкой инвариантов:

```go
package application

import (
    "context"
    "errors"
    "time"
    "yourbot/internal/domain"
)

type MenuService struct {
    Menus  domain.MenuRepo
    Tx     domain.TxRunner
    Clock  func() time.Time
}

func (s *MenuService) ComposeDraft(ctx context.Context, userID domain.UserID, days int, items []struct{ Day int; Meal domain.MealType; Title string; Steps []string }) (domain.MenuID, error) {
    now := s.Clock()
    var menuID domain.MenuID
    err := s.Tx.WithinTx(ctx, func(ctx context.Context) error {
        id, err := s.Menus.CreateDraft(ctx, userID, days)
        if err != nil { return err }
        menuID = id

        mis := make([]domain.MenuItem, 0, len(items))
        for _, it := range items {
            mi, err := domain.NewMenuItem(menuID, it.Day, it.Meal, it.Title, it.Steps)
            if err != nil { return err }
            // extra bounds check for days (domain unaware of Days):
            if it.Day < 1 || it.Day > days { return errors.New("day index out of range") }
            mis = append(mis, mi)
        }
        return s.Menus.ReplaceItems(ctx, menuID, mis)
    })
    return menuID, err
}

func NowUTC() time.Time { return time.Now().UTC() }
```

Работа с профилем:

```go
func (s *ProfileService) UpdateProfile(ctx context.Context, uid domain.UserID, goal string, likes, dislikes, allergens []string) error {
    p, err := domain.NewUserProfile(uid, goal, likes, dislikes, allergens, s.Clock())
    if err != nil { return err }
    return s.Profiles.Upsert(ctx, p)
}
```

### 7) Тестирование доменных инвариантов

```go
package domain_test

import (
    "testing"
    "time"
    "yourbot/internal/domain"
)

func TestNewMenuDraft_DaysBounds(t *testing.T) {
    _, err := domain.NewMenuDraft("m1", "u1", 0, time.Now())
    if err == nil { t.Fatal("expected error for days=0") }
}

func TestNewMenuItem_Valid(t *testing.T) {
    mi, err := domain.NewMenuItem("m1", 1, domain.MealDinner, "Паста", []string{"Вскипятить воду", "Сварить"})
    if err != nil { t.Fatalf("unexpected error: %v", err) }
    if mi.MealType != domain.MealDinner { t.Fatal("wrong meal") }
}
```

### 8) Границы ответственности

- Домен не знает о БД и Telegram/LLM. Он задаёт типы, инварианты и ошибки.
- Реализации портов (репозитории) — в `internal/adapter/db/postgres`, используют `pgx` и маппят БД‑ошибки в доменные.
- Use‑case уровни (`internal/usecase`) оркестрируют транзакции и интеграции, опираясь на доменные порты.

---

Чек‑лист ревью доменного слоя
- Сущности имеют явные конструкторы с проверкой инвариантов
- Идентификаторы — типизированы (`UserID`, `MenuID`), не «сырые» строки
- Ошибки стабилизированы (`ErrNotFound`, обёртки с `%w`), без сравнения строк
- Порты определяют ровно необходимые операции; нет утечек инфраструктуры
- Тесты на инварианты покрывают негативные случаи (табличные тесты)

