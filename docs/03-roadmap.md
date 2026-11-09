# track-me — Roadmap

> Ритм: пн–пт по 2 часа. Оценки даны «в ускоренном темпе»: если этап занял больше времени, это нормально. Этап закрыт, когда выполнен **весь** DoD и ты отвечаешь на контрольные вопросы своими словами.
>
> Каждый этап = отдельная ветка `stage/NN-name` → PR → squash-merge в `main`.

| # | Этап | Оценка |
|---|---|---|
| 0 | Окружение, Git CLI, монорепо, конвенции | 3–4 дня |
| 1 | Голый Node.js и HTTP | 1 нед |
| 2 | Fastify, архитектура модулей, валидация, ошибки | 1 нед |
| 3 | SQL, PostgreSQL, Drizzle, миграции | 2 нед |
| 4 | Аутентификация и авторизация | 1.5 нед |
| 5 | Тестирование | 1 нед |
| 6 | Интеграция с Next.js по FSD | 1 нед |
| 7 | Docker и Docker Compose | 1 нед |
| 8 | CI: GitHub Actions | 0.5–1 нед |
| 9 | Linux-сервер в WSL2, Nginx, CD | 1.5 нед |
| 10 | Мониторинг и логи | 1 нед |
| 11 | v1.x: очереди, real-time, файлы, поиск | по желанию |

**Итого MVP до мониторинга:** ~12–14 недель.

---

## Этап 0. Окружение и конвенции
**Цель.** Рабочее монорепо, уверенный Git из терминала, автоматическая проверка коммитов.
**Концепции.** Git-объекты (commit, tree, ref), ветки как указатели, merge vs rebase, cherry-pick, pnpm workspaces, Conventional Commits, git hooks.

**Задачи**
1. `git init` в корне `track-me`, `.gitignore`, первый коммит с `docs/`. С этого момента — **только терминал**, без GUI.
2. Создать репозиторий на GitHub, подключить remote, запушить. Включить защиту `main` (только через PR).
3. Включить corepack и pnpm, оформить `pnpm-workspace.yaml`, создать `apps/web` через `create-next-app`.
4. Пустые `apps/api`, `packages/contracts` с `package.json` и `tsconfig` (общий `tsconfig.base.json`).
5. commitlint + lefthook: невалидное сообщение коммита отклоняется.
6. Git-тренажёр: пройти Learn Git Branching (разделы Main и Remote) и в песочнице-репо руками сделать `rebase`, `cherry-pick`, `reset --soft/--hard`, `reflog`, восстановить «потерянный» коммит.

**DoD.** Репо на GitHub; `pnpm -r` видит все пакеты; коммит `fix stuff` отклоняется hook-ом; ты можешь без подсказок восстановить коммит после `reset --hard`.

**Контрольные вопросы**
- Чем `merge` отличается от `rebase` по истории? Когда rebase опасен?
- Что физически делает `cherry-pick`? Почему у нового коммита другой hash?
- `reset --soft`, `--mixed`, `--hard`: что меняется в HEAD, индексе и рабочей копии?
- Когда коммит должен быть `feat`, а когда `refactor`? Что означает `!` в `feat!:`?

**Источники.** [Pro Git (ch. 2, 3, 7)](https://git-scm.com/book/ru/v2) · [Conventional Commits](https://www.conventionalcommits.org/ru/v1.0.0/) · [pnpm workspaces](https://pnpm.io/workspaces) · [Learn Git Branching](https://learngitbranching.js.org/?locale=ru_RU)

---

## Этап 1. Голый Node.js и HTTP
**Цель.** Понять, что делает любой фреймворк, написав мини-API без него.
**Концепции.** Event loop, неблокирующий I/O, `node:http`, request/response как streams, метод, URL, заголовки, статус-коды, Content-Type, парсинг тела, CORS, идемпотентность, graceful shutdown.

**Задачи**
1. API `tasks` в памяти на `node:http` (без зависимостей): `GET/POST/PATCH/DELETE`, свой роутер с параметрами пути (`/tasks/:id`).
2. Ручной парсинг JSON-тела из stream с лимитом размера (413) и обработкой битого JSON (400).
3. Правильные статус-коды: 200/201/204/400/404/405/413/500, заголовок `Location` при создании.
4. Свой «middleware pipeline»: логирование времени запроса, `requestId`.
5. Graceful shutdown по `SIGINT`/`SIGTERM`.
6. Исследовать запросы через `curl -v` и DevTools: найти все заголовки, объяснить каждый.

**DoD.** Все CRUD-операции работают через `curl`; на битый JSON — 400, а не падение процесса; сервер корректно завершается по Ctrl+C; в коде ни одной npm-зависимости, кроме dev (TS).

**Контрольные вопросы**
- Что произойдёт с остальными запросами, если в обработчике запустить синхронный цикл на 5 секунд? Почему?
- Почему тело запроса приходит кусками (chunks)? Как это связано со streams?
- PUT vs PATCH vs POST: какие из них идемпотентны и почему это важно?
- Что такое CORS preflight и почему `curl` его не делает, а браузер делает?

**Источники.** [Node.js: Anatomy of an HTTP Transaction](https://nodejs.org/en/learn/modules/anatomy-of-an-http-transaction) · [Node.js: The Event Loop](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick) · [MDN: HTTP](https://developer.mozilla.org/ru/docs/Web/HTTP)

---

## Этап 2. Fastify и архитектура
**Цель.** Переписать этап 1 на Fastify с production-структурой.
**Концепции.** Plugins и encapsulation, decorators, hooks (lifecycle), схемы и сериализация, centralized error handler, Problem Details, конфиг из env, слои route → service → repository, OpenAPI.

**Задачи**
1. Каркас `apps/api`: `buildApp()` отдельно от `listen()` (зачем — поймёшь на этапе 5).
2. Конфиг: валидация env через Zod при старте; приложение падает с понятной ошибкой без нужных переменных.
3. Модуль `tasks` (пока in-memory repository) со слоями; Zod-схемы в `packages/contracts`.
4. Единый error handler → RFC 9457; доменные ошибки (`NotFoundError`, `ForbiddenError`) отдельно от HTTP.
5. pino: `requestId`, redact, `pino-pretty` в dev.
6. Swagger UI на `/docs`, сгенерированный из схем.

**DoD.** Невалидный запрос → 400 Problem Details с описанием полей; неизвестная ошибка → 500 без stack trace наружу, но со stack trace в логе; OpenAPI соответствует реальным схемам; service не импортирует ничего из Fastify.

**Контрольные вопросы**
- Что такое encapsulation в Fastify и чем `fastify-plugin` меняет поведение?
- Почему service не должен знать про `request`/`reply`? Что это даёт тестам?
- Зачем Fastify схема **ответа**, а не только запроса (подсказка: производительность и утечки данных)?
- Параллель с фронтом: что в React похоже на hooks lifecycle Fastify?

**Источники.** [Fastify docs: Getting Started, Plugins, Lifecycle, Hooks, Validation and Serialization](https://fastify.dev/docs/latest/) · [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) · [Zod](https://zod.dev/)

---

## Этап 3. SQL, PostgreSQL, Drizzle
**Цель.** Спроектировать схему из ТЗ (раздел 5), уверенно писать SQL, подключить ORM.
**Концепции.** Нормализация (1–3NF), PK/FK, `ON DELETE`, constraints, индексы (B-tree, составные, partial), транзакции, ACID, уровни изоляции, блокировки (`SELECT ... FOR UPDATE`), N+1, `EXPLAIN ANALYZE`, миграции, seed, offset vs cursor pagination.

**Задачи**
1. Postgres в Docker одной командой `docker run` (Compose будет позже), подключение через `psql`.
2. **Неделя SQL руками:** создать таблицы DDL-скриптом, вставить данные, написать 15 запросов: JOIN, LEFT JOIN, GROUP BY + HAVING, подзапрос, CTE, window function («топ-3 задачи по приоритету в каждом статусе»).
3. ER-диаграмма (dbdiagram.io или mermaid в `docs/`) и ответы на вопросы из таблицы ТЗ, раздел 5.
4. Перенести схему в Drizzle, миграции через drizzle-kit, seed-скрипт.
5. Repository на Drizzle вместо in-memory; генерация номера задачи без гонок (докажи тестом с параллельными запросами).
6. Индексы под фильтры списка задач; сравнить `EXPLAIN ANALYZE` до и после на 100k задач.

**DoD.** `pnpm db:migrate && pnpm db:seed` с нуля поднимает БД; 50 параллельных созданий задачи дают 50 уникальных номеров; список задач с фильтрами использует индекс (видно в плане); нет N+1 в списке задач с исполнителями.

**Контрольные вопросы**
- Чем `READ COMMITTED` отличается от `SERIALIZABLE`? Какую аномалию ты предотвратил при генерации номера?
- Почему cursor pagination лучше offset на больших таблицах? Когда offset всё же ок?
- Почему порядок колонок в составном индексе важен?
- Зачем миграции, если Drizzle может «просто синхронизировать» схему (`push`)?

**Источники.** [PostgreSQL Tutorial (официальный, ч. I–II)](https://www.postgresql.org/docs/current/tutorial.html) · [Use The Index, Luke](https://use-the-index-luke.com/) · [Drizzle docs](https://orm.drizzle.team/docs/overview)

---

## Этап 4. Аутентификация и авторизация
**Цель.** Безопасный вход и права доступа по матрице из ТЗ (2.1).
**Концепции.** Authn vs authz, хэширование паролей (argon2id, соль, почему не SHA-256), сессии vs JWT, cookie-атрибуты, CSRF, timing attacks, enumeration, rate limiting, RBAC, IDOR, принцип наименьших привилегий.

**Задачи**
1. Регистрация и вход, argon2id; одинаковый ответ и время ответа при неверном email и пароле.
2. Сессии в БД: токен генерируется криптостойко, в БД хранится его хэш; sliding expiration; logout; «выйти везде»; инвалидация при смене пароля.
3. Cookie: `HttpOnly`, `Secure`, `SameSite`, `Path`; CSRF-защита (проверка `Origin` и/или double-submit token, выбор обоснуй).
4. `@fastify/rate-limit` на auth-маршрутах, `@fastify/helmet`.
5. Authorization: decorator/hook «загрузить membership и проверить право»; единая функция `can(role, action)`.
6. Применить модули Projects, Members, Statuses, Tasks, Comments с проверкой прав.

**DoD.** Все эндпоинты из ТЗ, раздел 6 работают; viewer не может изменить задачу (403); пользователь вне проекта получает 404 на его ресурсы; 6 неудачных логинов подряд → 429; в БД нет ни одного токена сессии в открытом виде.

**Контрольные вопросы**
- Почему пароль хэшируют медленной функцией? Что такое соль и зачем pepper?
- Почему JWT в `localStorage` — плохая идея? Когда JWT всё же лучше сессий?
- Как `SameSite=Lax` защищает от CSRF и в каких случаях не защищает?
- Почему на чужой проект лучше вернуть 404, а не 403?

**Источники.** [The Copenhagen Book](https://thecopenhagenbook.com/) · [OWASP Cheat Sheets: Authentication, Session Management, Password Storage, CSRF](https://cheatsheetseries.owasp.org/) · [MDN: Using HTTP cookies](https://developer.mozilla.org/ru/docs/Web/HTTP/Cookies)

---

## Этап 5. Тестирование
**Цель.** Уверенность в рефакторинге: тесты ловят регрессии прав и бизнес-логики.
**Концепции.** Пирамида тестов, unit vs integration vs e2e, test doubles (stub/mock/fake), изоляция (транзакция-на-тест или чистая БД), фабрики, AAA, coverage и его ограничения, flaky tests.

**Задачи**
1. Vitest в монорепо; unit-тесты на `can()`, вычисление позиции задачи, доменные правила.
2. Integration: `buildApp()` + `inject()` + Testcontainers Postgres; миграции перед прогоном.
3. Табличный тест матрицы прав: роль × эндпоинт → ожидаемый статус.
4. Тесты на auth-сценарии: истёкшая сессия, logout everywhere, rate limit.
5. Порог покрытия в конфиге.

**DoD.** `pnpm test` зелёный с нуля на чистой машине с Docker; тесты независимы (проходят в любом порядке и параллельно); намеренная поломка проверки прав роняет минимум один тест.

**Контрольные вопросы**
- Почему мы не мокаем БД в integration-тестах? Какие баги пропустили бы моки?
- 100% покрытия — значит, багов нет? Приведи контрпример.
- Как ты изолируешь данные между тестами и какие компромиссы у твоего способа?

**Источники.** [Vitest docs](https://vitest.dev/guide/) · [Fastify: Testing](https://fastify.dev/docs/latest/Guides/Testing/) · [Testcontainers for Node.js](https://node.testcontainers.org/)

---

## Этап 6. Интеграция с Next.js по FSD
**Цель.** Рабочий UI поверх API, правильная структура FSD, auth в RSC.
**Концепции.** FSD-слои и public API, Next.js App Router + FSD, rewrites/proxy, проброс cookie из RSC, общий контракт (`packages/contracts`), кэширование и ревалидация данных, optimistic updates.

**Задачи**
1. Структура FSD в `apps/web` (см. `04-conventions.md`), линтер границ импортов (Steiger или eslint-plugin-boundaries).
2. Next.js rewrites `/api/*` → API в dev (позже это сделает nginx).
3. Защищённые маршруты: middleware и проверка сессии в RSC.
4. Экраны: auth, список проектов, доска с drag-n-drop, карточка задачи с комментариями и активностью.
5. Playwright: 3–5 e2e-сценариев.

**DoD.** Steiger без ошибок; нет импортов «вверх» по слоям; разлогин в одной вкладке ведёт на `/login` в другой при следующем запросе; e2e зелёные.

**Контрольные вопросы**
- Почему `entities` не может импортировать `features`? Что делать, если очень хочется?
- Чем отличается проверка доступа в Next.js middleware от проверки в API? Достаточно ли одной?
- Как RSC узнаёт о сессии пользователя, если запрос к API идёт с сервера Next, а не из браузера?

**Источники.** [Feature-Sliced Design](https://feature-sliced.design/ru/docs) · [FSD: Usage with Next.js](https://feature-sliced.design/docs/guides/tech/with-nextjs) · [Next.js: Authentication](https://nextjs.org/docs/app/guides/authentication)

---

## Этап 7. Docker и Compose
**Цель.** Весь стек поднимается одной командой.
**Концепции.** Образ vs контейнер, слои и кэш, multi-stage build, `.dockerignore`, non-root, healthcheck, сети, volumes, env и secrets, `depends_on` + `condition`, размер образа.

**Задачи**
1. Multi-stage Dockerfile для `api` и `web` (Next standalone output) в монорепо.
2. `infra/docker-compose.yml`: postgres (volume), api, web, миграции как отдельный one-shot сервис.
3. Healthchecks, перезапуск, `compose.override` для dev с hot reload.
4. Оптимизация: сравнить размер и время сборки до и после.

**DoD.** `docker compose up` на чистой машине → рабочее приложение; данные БД переживают `down` (без `-v`); контейнеры запущены не от root; образ api < 250 МБ.

**Контрольные вопросы**
- Почему `COPY package.json` идёт до `COPY . .`?
- Чем volume отличается от bind mount? Что используешь для БД, а что для dev-кода и почему?
- Почему внутри Compose api обращается к БД по имени `postgres`, а не `localhost`?

**Источники.** [Docker docs: Building best practices](https://docs.docker.com/build/building/best-practices/) · [Docker Compose docs](https://docs.docker.com/compose/) · [Next.js: output standalone](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)

---

## Этап 8. CI: GitHub Actions
**Цель.** Каждый PR автоматически проверяется, `main` всегда собирается.
**Концепции.** Workflow, jobs, steps, runners, triggers, кэш pnpm, сервисы и Docker в CI, секреты, матрицы, branch protection + required checks, GHCR, теги образов (sha vs semver).

**Задачи**
1. Workflow `ci.yml`: install → lint → typecheck → test (с Testcontainers) → build.
2. Проверка commitlint для заголовка PR.
3. При merge в `main`: сборка и push образов в GHCR с тегами `sha` и `latest`.
4. Dependabot и `pnpm audit`.
5. Required checks в защите `main`.

**DoD.** PR с упавшим тестом нельзя смержить; повторный прогон быстрее за счёт кэша; образы появляются в GHCR после merge.

**Контрольные вопросы**
- Почему для деплоя лучше тег `sha`, а не `latest`?
- Где и как хранятся секреты в Actions, кто их может прочитать?
- CI vs CD vs Continuous Deployment: в чём разница?

**Источники.** [GitHub Actions docs](https://docs.github.com/ru/actions) · [Publishing Docker images (GHCR)](https://docs.github.com/en/actions/use-cases-and-examples/publishing-packages/publishing-docker-images)

---

## Этап 9. Linux-сервер в WSL2, Nginx, CD
**Цель.** «Сервер» как настоящий VPS: доступ по SSH, reverse proxy, HTTPS, автодеплой.
**Концепции.** FHS, пользователи и группы, права (`chmod`/`chown`), процессы и сигналы, systemd (units, journalctl), пакеты (apt), SSH-ключи, firewall, переменные окружения, cron. Nginx: server blocks, `location`, `proxy_pass`, upstream, заголовки, TLS. Self-hosted runner, стратегия деплоя, миграции при деплое, откат.

**Задачи**
1. Отдельный дистрибутив Ubuntu в WSL2 (`trackme-server`), пользователь `deploy` без пароля sudo на всё подряд.
2. Linux-практика: 20 упражнений из чеклиста (навигация, права, процессы, логи, пайпы, `grep`/`find`).
3. SSH-доступ с Windows по ключу; парольный вход отключён.
4. Docker Engine в этом дистрибутиве (не Desktop), self-hosted GitHub runner как systemd-сервис.
5. Nginx как reverse proxy (`/api` → api, `/` → web), HTTPS через mkcert, gzip, security headers.
6. CD-workflow: merge в `main` → runner тянет образы по `sha` → миграции → `compose up` → smoke-тест `/health/ready` → при провале откат на предыдущий `sha`.
7. Бэкап БД (`pg_dump` по cron) и проверка восстановления.
8. Опционально: публичное демо на бесплатном PaaS.

**DoD.** `https://track-me.localhost` работает через nginx; merge в `main` деплоит без ручных действий; сломанный релиз откатывается автоматически; бэкап восстановлен в чистую БД.

**Контрольные вопросы**
- Что значит `chmod 640` и почему ключ SSH должен быть `600`?
- Зачем nginx перед Node, если Node сам умеет слушать порт?
- Почему миграции БД — самая опасная часть деплоя? Что такое expand/contract миграции?
- Что делает `proxy_set_header X-Forwarded-For` и почему API надо явно ему «доверять» (`trustProxy`)?

**Источники.** [Ubuntu Server docs](https://documentation.ubuntu.com/server/) · [nginx: Beginner's Guide + reverse proxy](https://nginx.org/en/docs/beginners_guide.html) · [GitHub: self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners)

---

## Этап 10. Мониторинг и логи
**Цель.** Узнавать о проблемах раньше пользователей.
**Концепции.** Три столпа наблюдаемости (logs, metrics, traces), RED/USE-методологии, типы метрик (counter, gauge, histogram), labels и кардинальность, PromQL, дашборды, алерты, SLI/SLO.

**Задачи**
1. `prom-client`: HTTP RED-метрики, метрики пула БД; `/metrics` закрыт от внешнего мира в nginx.
2. Prometheus + Grafana в Compose, дашборд «RPS / p95 latency / error rate / активные сессии».
3. Loki + Alloy: логи контейнеров в Grafana, поиск по `requestId`.
4. Алерт: error rate > 5% за 5 минут.
5. Sentry для api и web, с release = git sha.
6. Нагрузочный тест (k6 или autocannon) и проверка NFR из ТЗ, 4.2.

**DoD.** На дашборде видна нагрузка от k6; по `requestId` из ответа находится вся цепочка логов; искусственная 500-ка поднимает алерт и попадает в Sentry.

**Контрольные вопросы**
- Почему `userId` нельзя делать label метрики?
- Чем histogram отличается от summary, почему для latency берут histogram?
- Что такое SLO 99.9% в минутах простоя в месяц?

**Источники.** [Prometheus docs: Concepts, Instrumentation best practices](https://prometheus.io/docs/introduction/overview/) · [Grafana Loki docs](https://grafana.com/docs/loki/latest/) · [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)

---

## Этап 11. v1.x (по желанию)
Каждая фича из ТЗ, 1.4 = мини-этап с тем же форматом. Рекомендуемый порядок: **Redis + BullMQ + email (Mailpit)** → **real-time (SSE → WebSocket)** → **файлы (MinIO)** → **FTS** → **NestJS-порт одного модуля** (сравнение фреймворков).
