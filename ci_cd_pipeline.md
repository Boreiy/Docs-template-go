## CI/CD pipeline (GitHub Actions)

Документ — самодостаточный. Покрывает: настройку Go-среды через `actions/setup-go@v5`, кэш модулей, запуск тестов и сборку бинаря. Пример минимального workflow.

### 1) Простой workflow

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true
      - run: go test ./...
      - run: go build ./cmd/bot
```

### 2) Деплой (опционально)

```yaml
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go build -o bot ./cmd/bot
      - uses: actions/upload-artifact@v4
        with:
          name: bot
          path: bot
```

Чек-лист
- Используйте `actions/setup-go@v5` для актуального Go.
- Включайте кэш модулей (`cache: true`).
- Разделяйте этапы тестов и деплоя.
