# track-me — документация

| Документ | Содержание |
|---|---|
| [01-spec.md](01-spec.md) | ТЗ: границы MVP, роли, user stories, требования, модель данных, API |
| [02-stack.md](02-stack.md) | Стек и обоснование, подходы Next.js + backend |
| [03-roadmap.md](03-roadmap.md) | Этапы, задачи, DoD, контрольные вопросы, источники |
| [04-conventions.md](04-conventions.md) | Conventional Commits, FSD, структура backend |

## Прогресс

| # | Этап | Статус | Начат | Закрыт |
|---|---|---|---|---|
| 0 | Окружение, Git CLI, конвенции | ⏳ в работе | 2026-11-09 | |
| 1 | Голый Node.js и HTTP | ⬜ | | |
| 2 | Fastify и архитектура | ⬜ | | |
| 3 | SQL, PostgreSQL, Drizzle | ⬜ | | |
| 4 | Auth и authz | ⬜ | | |
| 5 | Тестирование | ⬜ | | |
| 6 | Next.js + FSD | ⬜ | | |
| 7 | Docker и Compose | ⬜ | | |
| 8 | CI: GitHub Actions | ⬜ | | |
| 9 | Linux, Nginx, CD | ⬜ | | |
| 10 | Мониторинг | ⬜ | | |

## Журнал решений (ADR-lite)
Записывай сюда каждое неочевидное решение: дата, решение, альтернативы, почему.

- 2026-11-09 — Отдельный backend на Fastify за одним origin с Next.js (через proxy). Альтернативы: Next.js fullstack, отдельные домены. См. 02-stack.md §0.
- 2026-11-09 — Сессии в БД вместо JWT: нужен отзыв сессий (US-03), клиент один (web).
- 2026-11-09 — «Сервер» = Ubuntu в WSL2 + self-hosted runner: бесплатного VPS без карты нет.
