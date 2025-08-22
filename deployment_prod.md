## Продовое развёртывание

### Переменные окружения
- `ENV=prod` — включает продовый режим.
- `HTTP_ADDR` — адрес и порт HTTP-сервера, например `:8080`.
- `TELEGRAM_WEBHOOK_URL` — публичный HTTPS‑URL вебхука.
- `TELEGRAM_WEBHOOK_SECRET` — значение для проверки заголовка `X-Telegram-Bot-Api-Secret-Token`.
- `DATABASE_URL` — строка подключения к внешней базе PostgreSQL.

### Подключение к базе
Приложение использует переменную `DATABASE_URL` для подключения к внешнему экземпляру PostgreSQL. Убедитесь, что база доступна из сети и на ней применены все миграции.

### Docker
Соберите и запустите контейнер с нужными переменными окружения:

```bash
docker build -t menubot .
docker run -d --env-file .env -p 8080:8080 menubot
```

### Настройка и проверка вебхука
После запуска контейнера вызовите метод `setWebhook` Bot API, указав HTTPS‑адрес и секрет:

```bash
curl -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/setWebhook" \
  -d url="$TELEGRAM_WEBHOOK_URL" \
  -d secret_token="$TELEGRAM_WEBHOOK_SECRET"
```

Ответ должен содержать `"ok":true`. Вебхук обязан обслуживаться по HTTPS, иначе Telegram его не примет.

---
Этот файл описывает только продовое развёртывание. За дополнительными деталями по конфигурации обращайтесь к другим документам в `dev-docs/`.
