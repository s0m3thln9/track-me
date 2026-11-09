# track-me — Конвенции

## 1. Conventional Commits

### Формат
```
<type>(<scope>)!: <subject>

<body — зачем, а не что>

<footer: BREAKING CHANGE: ..., Refs: #12>
```

### Типы
| type | Когда | Попадает в changelog |
|---|---|---|
| `feat` | новая возможность для пользователя/клиента API | ✅ minor |
| `fix` | исправление бага | ✅ patch |
| `refactor` | изменение кода без изменения поведения | — |
| `perf` | ускорение без изменения поведения | ✅ |
| `test` | только тесты | — |
| `docs` | только документация | — |
| `style` | форматирование, без изменения логики | — |
| `build` | сборка, зависимости, Dockerfile | — |
| `ci` | конфиги CI/CD | — |
| `chore` | прочее (не src, не тесты) | — |
| `revert` | откат коммита | ✅ |

`!` или footer `BREAKING CHANGE:` → major.

### Scopes проекта
`api`, `web`, `contracts`, `db`, `auth`, `infra`, `ci`, `docs`, `deps`. На фронте можно указывать слой или слайс: `web/task-card`.

### Правила
- subject в повелительном наклонении, на английском, без точки, ≤ 72 символов: `feat(api): add task move endpoint`.
- Один коммит = одно логическое изменение. Если в сообщении хочется написать «and», скорее всего, это два коммита.
- В PR — squash merge. **Заголовок PR** тоже в формате Conventional Commits, он станет коммитом в `main`.

> Самопроверка: поменял лимит pageSize со 100 на 50 — это `fix`, `feat!` или `refactor`? Зависит от того, кто клиент API. Порассуждай.

### Ветки
`stage/02-fastify`, `feat/task-comments`, `fix/session-expiry`. Короткоживущие, от `main`, в `main` через PR.

---

## 2. Feature-Sliced Design (apps/web)

### Слои (сверху вниз, импорт только вниз)
| Слой | Что лежит | Пример в track-me |
|---|---|---|
| `app` | провайдеры, глобальные стили, инициализация | QueryClientProvider, theme |
| `pages` | композиция страницы из виджетов | `board-page`, `task-page`, `login-page` |
| `widgets` | крупные самостоятельные блоки UI | `board` (колонки + карточки), `task-sidebar`, `app-header` |
| `features` | действие пользователя, приносящее ценность | `create-task`, `move-task`, `assign-task`, `add-comment`, `login-by-email` |
| `entities` | бизнес-сущности: модель, API, базовый UI | `task`, `project`, `user`, `comment`, `session` |
| `shared` | переиспользуемое без бизнес-логики | `ui` (кнопки), `api` (http-клиент), `lib`, `config` |

### Сегменты внутри слайса
`ui/`, `model/`, `api/`, `lib/`, `config/`. Наружу только через `index.ts` (public API).

### Правила, которые чаще всего нарушают
1. Слайсы **одного** слоя не импортируют друг друга (`features/create-task` ✗→ `features/assign-task`). Если очень нужно, поднимай композицию на слой выше.
2. Импорт только через public API: `@/entities/task`, а не `@/entities/task/ui/TaskCard`.
3. `shared` не знает о бизнесе: в нём нет слова «task».
4. Не создавай `features` «на вырост». Действие, которое используется в одном месте, может жить в `pages`/`widgets`.

### Next.js App Router + FSD
Маршрутизация (`app/` Next.js) и слой `app`/`pages` FSD конфликтуют по именам. Решение описано в официальном гайде FSD «Usage with Next.js»: прочитай его на этапе 6 и объясни своими словами, как устроена твоя структура. Роуты Next.js становятся тонкими: только реэкспорт страницы из FSD-слоя `pages`.

### Контроль
Steiger (официальный линтер FSD) в CI.

---

## 3. Backend (apps/api): модульная архитектура

FSD — методология **фронтенда**. На бэкенде аналог — **модульный монолит**: деление по доменам (как слайсы), внутри — по слоям (как сегменты).

```
apps/api/src/
├─ app.ts            # buildApp(): регистрация плагинов и модулей
├─ server.ts         # listen + graceful shutdown
├─ config/           # env-схема
├─ plugins/          # db, auth, error-handler, swagger (cross-cutting)
├─ shared/           # errors, utils — без доменной логики (как shared в FSD)
└─ modules/
   ├─ auth/
   ├─ projects/
   ├─ tasks/
   │  ├─ tasks.routes.ts      # HTTP: схемы, маршруты → вызывают service
   │  ├─ tasks.service.ts     # бизнес-правила, транзакции; не знает про HTTP
   │  ├─ tasks.repository.ts  # SQL/Drizzle; не знает про правила
   │  └─ index.ts             # public API модуля
   └─ comments/
```

**Правила.** Модуль не лезет в repository другого модуля, только в его service через public API (параллель с FSD правилом 2). Зависимость направлена `routes → service → repository`, никогда наоборот.
