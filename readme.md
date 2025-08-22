Индекс developer docs для MenuBot-go. Документы минимальные, прикладные и самодостаточные. Каждый содержит практические примеры и достаточен, чтобы сразу применять тему в коде.  В каждом документе — примеры и указания по слоям Clean Architecture: `usecase (application)`, `adapter/infrastructure`, `domain`, `tests`.

1. Архитектура и структура проекта — слои, зависимости, контракты данных, схема БД (в разрезе), как всё связано. [architecture](architecture.md)
2. Telegram и UX — tgbotapi, polling (dev), реплай‑клавиатуры и шаблоны сообщений. [telegram](telegram.md)
3. FSM и пользовательские потоки — онбординг (аллергии, предпочтения, цель), главное меню, составление меню, профиль. [fsm_flows](fsm_flows.md)
4. LLM и guardrails — контекст и агрегация, промпты, JSON‑схемы черновика/финалки, валидация ответов, ретраи/таймауты, рероллы и финализация. [llm_and_guardrails](llm_and_guardrails.md)
5. Домен и репозитории — сущности и инварианты; интерфейсы портов для профиля, кладовой, меню и фидбэка. [domain](domain.md)
6. PostgreSQL и миграции — минимальная схема, pgx/v5, транзакции; автоприменение миграций. [postgresql](postgresql.md)
7. Конфигурация и логирование — .env и конфиги; slog+tint+jsonhandler+lumberjack; приватность/PII. [config_logging](config_logging.md)
8. Тестирование и дев‑среда — testing/testify/ginkgo, покрытие, Makefile/запуск, Docker/compose. [testing_and_dev](testing_and_dev.md)
