## Продовое развёртывание

### Переменные окружения
- `ENV=prod` — включает продовый режим.
- `HTTP_ADDR` — адрес и порт HTTP-сервера, например `:8080`.
- `TELEGRAM_WEBHOOK_URL` — публичный HTTPS‑URL вебхука.
- `TELEGRAM_WEBHOOK_SECRET` — значение для проверки заголовка `X-Telegram-Bot-Api-Secret-Token`.
- `DATABASE_URL` — строка подключения к внешней базе PostgreSQL.

#### Рекомендуемые переменные для продакшена
- `GIN_MODE=release` — отключает отладочный вывод Gin.
- `LOG_LEVEL=info` — устанавливает уровень логирования.
- `TZ=UTC` — фиксирует временную зону.
- `TRUSTED_PROXIES=0.0.0.0/0` — доверять всем прокси (нужна настройка на уровне инфраструктуры).
- `OTEL_EXPORTER_OTLP_ENDPOINT` — адрес приёмника трассировок и метрик.

Пример:

```bash
export GIN_MODE=release
export LOG_LEVEL=info
export TZ=UTC
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

### Подключение к базе
Приложение использует переменную `DATABASE_URL` для подключения к внешнему экземпляру PostgreSQL. Убедитесь, что база доступна из сети и на ней применены все миграции.

### Docker
Соберите и запустите контейнер с нужными переменными окружения:

```bash
docker build -t menubot .
docker run -d --env-file .env -p 8080:8080 menubot
```

### Обратный прокси и TLS
Для выдачи TLS‑сертификатов и маршрутизации удобно использовать Traefik.

Конфигурация `traefik.yml`:

```yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

http:
  routers:
    bot:
      rule: "Host(`example.com`)"
      service: bot
      entryPoints:
        - websecure
      tls:
        certResolver: letsencrypt
  services:
    bot:
      loadBalancer:
        servers:
          - url: "http://bot:8080"

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@example.com
      storage: acme.json
      httpChallenge:
        entryPoint: web
```

Запуск:

```bash
docker network create traefik
docker run -d --name traefik \
  --network traefik \
  -p 80:80 -p 443:443 \
  -v $PWD/traefik.yml:/etc/traefik/traefik.yml \
  -v $PWD/acme.json:/acme.json \
  traefik:v3
docker run -d --name bot \
  --network traefik \
  --env-file .env \
  menubot
```

### Health-check, мониторинг и логирование
Приложение должно отвечать на `/health` кодом `200`.

Пример проверки:

```bash
curl -f http://localhost:8080/health
```

В Docker можно добавить директиву `HEALTHCHECK`:

```Dockerfile
HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1
```

Для мониторинга и централизованного логирования используйте любой совместимый стек, например Prometheus + Grafana и вывод логов в stdout/stderr:

```bash
docker logs -f bot
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
