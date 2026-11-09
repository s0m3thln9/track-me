# track-me — Технический стек

> Принцип выбора: каждый инструмент должен **показывать устройство**, а не прятать его. Магию добавим потом, когда будешь понимать, что она делает.

## 0. Архитектура: Next.js + отдельный backend

### Три подхода

| | A. Next.js fullstack | B. Next.js + API, клиент ходит в API напрямую | C. Next.js + API за одним origin (выбран) |
|---|---|---|---|
| Как | Route Handlers / Server Actions + ORM прямо в Next | `web.app` → `api.app`, разные домены | Reverse proxy: `/` → Next, `/api` → Node API; RSC ходят в API по внутренней сети |
| ➕ | Один деплой, общие типы, меньше инфраструктуры, быстро стартовать | Чёткое разделение, API переиспользуется (мобилка, CLI) | Разделение как в B, но без CORS и проблем с cross-site cookies; API не торчит наружу отдельно |
| ➖ | Бизнес-логика вплетена во фреймворк; serverless-ограничения (пулы соединений к БД, WebSocket, долгие задачи); **backend ты не выучишь** | CORS, `SameSite`, CSRF сложнее; двойная работа с auth в RSC | Нужно настроить proxy (это и есть учебная цель); при SSR надо пробрасывать cookie |
| Когда в реальности | MVP стартапа, маленькая команда, контент-сайты | Публичный API, несколько клиентов | Большинство продуктовых команд с отдельным backend |

**Параллель с фронтом.** Server Actions — это RPC-вызов к серверу, который Next генерирует за тебя. Сейчас мы сделаем этот «сервер» сами и явно.

### Схема

```
Browser ──► nginx :443 ─┬─ /api/*  ──► api (Fastify) ──► PostgreSQL
                        └─ /*      ──► web (Next.js)
                                        └─ RSC fetch ──► http://api:3001 (внутренняя сеть Docker)
```

### Монорепо (pnpm workspaces)

```
track-me/
├─ apps/
│  ├─ web/        # Next.js (FSD)          ← твоя текущая папка frontend/ переедет сюда
│  └─ api/        # Fastify (модули)
├─ packages/
│  └─ contracts/  # Zod-схемы и типы, общие для web и api
├─ infra/         # docker-compose, nginx, prometheus, grafana
└─ docs/
```
Альтернативы: два отдельных репо (проще CI, но типы не расшарить); Nx или Turborepo поверх workspaces (добавим Turborepo на этапе CI, если захочешь кэш сборок).

---

## 1. Стек по пунктам

### Runtime: Node.js 24 LTS + TypeScript (strict)
- **Почему.** У тебя уже стоит v24, это LTS. TS запускаем через `tsx` в dev, через `tsc` в build.
- **Альтернативы.** Bun и Deno: быстрые и удобные, но на собеседованиях и в проде чаще Node.
- **Научит:** event loop, модули ESM, `process`, сигналы, потоки.

### Backend-фреймворк: Fastify
- **Почему.** Тонкий слой над `node:http`: видно жизненный цикл запроса (hooks), система плагинов и инкапсуляции, встроенные JSON Schema-валидация и логгер pino, `inject()` для тестов без поднятия порта.
- **Альтернативы.**
  - Express: самый популярный, но архитектурно устарел (обработка async-ошибок, отсутствие схем). Его middleware-модель полезно понимать: изучишь на этапе 1, когда будешь писать свой роутер.
  - NestJS: DI, декораторы, модули. Похож на Angular, популярен в enterprise. Много магии, поэтому хорош **вторым** фреймворком: после Fastify ты поймёшь, что он делает под капотом (NestJS умеет работать поверх Fastify).
  - Hono: лёгкий, для edge и serverless.
- **Научит:** request lifecycle, plugins/DI, hooks, сериализация, error handling.

### БД: PostgreSQL 17/18
- **Почему.** Стандарт индустрии: транзакции, `jsonb`, FTS, отличные индексы и документация.
- **Альтернативы.** MySQL (похожа, но меньше фич); MongoDB (документная модель плохо ложится на реляционные данные задач, ролей и связей).
- **Научит:** SQL, нормализация, индексы, транзакции и изоляция, `EXPLAIN`.

### ORM / query builder: Drizzle ORM + drizzle-kit (миграции)
- **Почему.** API почти один в один повторяет SQL, а у тебя SQL на 1/2. Drizzle заставляет думать запросами, а не объектами. Схема на TS даёт типы.
- **Перед этим** на этапе 3 неделю пишешь сырой SQL через `pg` и `psql`. Без этого ORM — чёрный ящик.
- **Альтернативы.** Prisma (свой DSL, удобна, но прячет SQL); Kysely (чистый type-safe query builder); TypeORM (стар, много проблем).
- **Научит:** миграции, связи, транзакции, типобезопасные запросы.

### Валидация: Zod + `fastify-type-provider-zod`
- **Почему.** Скорее всего, знаком тебе по фронту (react-hook-form). Одна схема даёт runtime-валидацию, TS-типы и OpenAPI. Схемы живут в `packages/contracts` и используются и на web, и на api.
- **Альтернативы.** TypeBox (нативная JSON Schema для Fastify, быстрее); Valibot (меньше бандл).
- **Научит:** «trust boundary»: почему валидировать надо на сервере, даже если фронт уже проверил.

### Авторизация: свои сессии в Postgres + cookie + argon2id, RBAC на уровне проекта
- **Почему.** Пишешь сам, чтобы понять механику. Серверные сессии легко отзывать (US-03), а для web-клиента с одним origin это стандартный и безопасный выбор.
- **Альтернативы.**
  - JWT access + refresh: нужен, когда много сервисов или мобильные клиенты. Минусы: сложный отзыв, частая ошибка — хранить токен в `localStorage`. Разберём сравнение в контрольных вопросах.
  - Готовые решения (Better Auth, Auth.js, Clerk): в реальном проекте часто разумный выбор, но учиться на них нельзя.
- **Научит:** cookies, CSRF, хэширование паролей, rate limiting, authn vs authz, IDOR.

### Документация API: `@fastify/swagger` + Scalar или Swagger UI
- OpenAPI генерируется из Zod-схем. Из него можно сгенерировать клиент для фронта (опционально).

### Тесты: Vitest + `fastify.inject()` + Testcontainers (Postgres) + Playwright (E2E)
- **Почему.** Vitest, скорее всего, тебе знаком. Testcontainers поднимает настоящий Postgres в Docker на время тестов, поэтому моки БД не нужны.
- **Альтернативы.** Jest (медленнее, ESM-боль); Supertest (вместо `inject`); pg-mem (in-memory Postgres, но неполная совместимость).
- **Научит:** пирамида тестов, изоляция тестов, фабрики данных, test doubles.

### Логи: pino (встроен в Fastify)
- JSON-логи, `requestId`, redact. В dev через `pino-pretty`.

### Контейнеризация: Docker + Docker Compose (Docker Desktop у тебя уже установлен)
- **Почему.** Индустриальный стандарт. Compose описывает весь стек одним файлом.
- **Альтернативы.** Podman (daemonless, rootless).
- **Научит:** образы и слои, multi-stage build, сети, volumes, healthchecks.

### CI/CD: GitHub Actions + GHCR (GitHub Container Registry) + self-hosted runner
- **Почему.** Бесплатно для публичных репо, лучшая документация. Карты нет, поэтому настоящего VPS нет. CD будет деплоить на «сервер» внутри WSL2 через **self-hosted runner**: это реальный production-паттерн для приватных сетей.
- **Альтернативы.** GitLab CI; деплой по SSH (тоже разберём, но для локального «сервера» GitHub до него не достучится).
- **Научит:** пайплайны, кэширование, секреты, матрицы, артефакты, environments, стратегия деплоя и отката.

### «Сервер»: отдельный дистрибутив Ubuntu в WSL2
- **Почему.** Настоящий Linux с systemd, пакетами, пользователями, SSH и Docker Engine. По ощущениям почти как VPS.
- **Опционально для портфолио:** публичное демо на бесплатном PaaS без карты. Сейчас это, например, Render (web service) и Neon (Postgres). Условия тарифов часто меняются, проверим на этапе деплоя.
- **Научит:** shell, права, процессы, systemd, сеть, SSH-ключи, firewall.

### Обратный прокси: Nginx (+ mkcert для локального HTTPS)
- **Почему.** Его конфиги встретишь везде, на собеседованиях спрашивают именно про него.
- **Альтернативы.** Caddy (auto-HTTPS, проще конфиг); Traefik (автодискавери контейнеров).
- **Научит:** reverse proxy, TLS, заголовки `X-Forwarded-*`, gzip, rate limit на краю, статика.

### Мониторинг: Prometheus + Grafana + Loki (сбор логов через Grafana Alloy) + Sentry (free)
- **Почему.** Open-source стандарт метрик и дашбордов, всё работает локально в Compose. Sentry на бесплатном Developer-плане собирает ошибки и не требует карты.
- **Альтернативы.** ELK/OpenSearch (тяжелее); Uptime Kuma (простые проверки доступности, можно добавить); OpenTelemetry (трейсинг, бонус-этап).
- **Научит:** метрики vs логи vs трейсы, RED-метрики, PromQL, алерты.

### Качество кода
- ESLint (flat config) + Prettier; commitlint + lefthook (git hooks); Dependabot.
- Альтернатива: Biome (линтер и форматтер в одном, быстрый).

---

## 2. Сводка

| Слой | Выбор | Главная альтернатива |
|---|---|---|
| Frontend | Next.js (App Router) + FSD | — (твой стек) |
| Runtime | Node.js 24 LTS + TS | Bun |
| Framework | Fastify | NestJS (второй фреймворк) |
| БД | PostgreSQL | MySQL |
| ORM | Drizzle (+ сырой SQL сначала) | Prisma |
| Валидация | Zod | TypeBox |
| Auth | Сессии в БД + cookie + argon2id | JWT, Better Auth |
| Тесты | Vitest, Testcontainers, Playwright | Jest, Supertest |
| Контейнеры | Docker + Compose | Podman |
| CI/CD | GitHub Actions + GHCR + self-hosted runner | GitLab CI |
| Сервер | Ubuntu в WSL2 | VPS (Oracle Free, нужна карта) |
| Proxy | Nginx | Caddy |
| Мониторинг | Prometheus, Grafana, Loki, Sentry | ELK, OpenTelemetry |
