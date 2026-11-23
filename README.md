# TeamHub

SaaS-платформа для хоккейных спортивных школ: расписание и состав, семейные кабинеты,
финансы и турниры, публичная витрина. Дизайн-направление — **RINK OS** (см. `docs/`).

> Статус: **Этап 0 завершён** (0.4: Compliance Gate, связь родитель-ребёнок; критерии: [`docs/STAGE0.md`](docs/STAGE0.md))
> Подэтапы: (Подэтап 0.3: шаблоны документов, согласия, PDF, i18n, Sphinx).
> Ранее: **Подэтап 0.2** — JWT-аутентификация, мультиарендность (схема на тенанта),
> CRUD тенантов и пользователей, аудит. Подробности: [`docs/AUTH.md`](docs/AUTH.md).

## Быстрый старт

```bash
cp .env.example .env
docker compose up --build

# Первого суперадмина API создать не может, он создаётся командой:
docker compose exec backend python -m app.cli create-superadmin \
    --email admin@example.com --full-name "Иван Иванов"
```

Что произойдёт: поднимется PostgreSQL 15 (с ролью `app_user`), backend дождётся его healthcheck, выполнит
`alembic upgrade head` под владельцем схем, `migrate-tenants` и запустит uvicorn с hot reload. Наружу смотрит
только nginx; порт backend не публикуется (иначе можно подделать IP, см. [`docs/SECURITY.md`](docs/SECURITY.md)).

| Что            | Где                              |
|----------------|----------------------------------|
| API (через nginx) | http://localhost:8000         |
| Health check   | http://localhost:8000/health     |
| Swagger (dev)  | http://localhost:8000/docs       |
| PostgreSQL     | `localhost:5432`                 |

Данные БД лежат в volume `pgdata`. Полный сброс: `docker compose down -v`.

## Разработка без Docker

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r backend/requirements-dev.txt
pre-commit install            # хуки на каждый коммит

export DATABASE_URL=postgresql+psycopg://teamhub:change_me_please@localhost:5432/teamhub
export JWT_SECRET_KEY=$(openssl rand -hex 32)
cd backend
alembic upgrade head
uvicorn app.main:app --reload
```

## Проверки качества

Запускать из корня репозитория, виртуальное окружение активно:

```bash
ruff check backend                 # линтер (+ isort, pyupgrade, bugbear...)
black --check backend              # форматирование (единственный форматтер)
mypy backend/app                   # типы, strict
pydoclint backend/app backend/alembic   # docstring'и (Google-style)
bash scripts/lint-sql.sh           # sqlfluff + squawk по SQL миграций
pytest                             # тесты (нужен PostgreSQL, см. ниже)
pre-commit run --all-files         # линтеры сразу
```

### Тесты

Тесты идут на настоящем PostgreSQL (схемы, `search_path` и констрейнты на моках не проверить).
Нужна отдельная БД, имя которой **заканчивается на `_test`**: тесты стирают в ней данные,
и без этого суффикса запускаться откажутся.

```bash
docker compose exec db createdb -U teamhub teamhub_test      # один раз
export TEST_DATABASE_URL=postgresql+psycopg://teamhub:change_me_please@localhost:5432/teamhub_test
pytest
```

То же самое выполняет GitHub Actions (`.github/workflows/lint.yml`), плюс отдельный job гоняет
миграции на живом PostgreSQL 15: `upgrade`, `alembic check`, `downgrade`, `upgrade`.

## Миграции

Запускать из `backend/`:

```bash
alembic upgrade head                      # применить
alembic downgrade -1                      # откатить последнюю
alembic check                             # модели и БД расходятся? (должно быть «No new upgrade operations»)
alembic revision -m "что делает миграция" # новая пустая миграция
python -m app.cli migrate-tenants         # применить миграции из tenant_migrations/ ко всем школам
```

Правила:

- миграции пишем SQL-ом (`op.execute`) с комментариями на русском: так их видят линтеры и ревьюер;
- имена констрейнтов и индексов — по `NAMING_CONVENTION` из `app/database.py`;
- любая новая миграция обязана проходить `scripts/lint-sql.sh`;
- после изменения модели прогоняйте `alembic check`.

## Документы, PDF и i18n

* Шаблоны документов (суперадмин): `/api/v1/super-admin/document-templates` (версии неизменяемы).
* Согласия: `/api/v1/consents/*`; PDF: `POST /api/v1/documents/generate-pdf`. Хранилище PDF
  (`STORAGE_DIR`, в Docker отдельный том) наружу не раздаётся.
* Профиль ребёнка: `POST /api/v1/coach/children` (multipart) только с `consent_confirmed=true` (Compliance Gate, 152-ФЗ);
  доступ семьи к ребёнку задаёт таблица `child_access`. Подробности: [`docs/api/children.rst`](docs/api/children.rst).
* Переводы: `GET /api/v1/locales/{lang}/{namespace}`; публичная витрина понимает `?lang=` и
  `Accept-Language`. Проверка `en/public.json`: `python scripts/check_i18n.py`.
* Документация: `make -C docs html` (Sphinx), подробности в [`docs/`](docs/ARCHITECTURE.md).

## Git

Ветки `feature/<подэтап>-<название>`, коммиты по Conventional Commits, PR в `develop`.
Правила, примеры и список автоматических проверок — в [`docs/GIT.md`](docs/GIT.md).
После клонирования: `pre-commit install` (ставит и хук сообщений коммитов).

## Структура

```
teamhub/
├── docker-compose.yml        PostgreSQL 15 + FastAPI
├── pyproject.toml            настройки ruff, black, mypy, pydoclint
├── .pre-commit-config.yaml   хуки git
├── .sqlfluff / .squawk.toml  настройки SQL-линтеров
├── nginx/nginx.conf          reverse proxy (перезаписывает X-Forwarded-For)
├── db/init/                  создание роли app_user при первом запуске БД
├── scripts/                  lint-sql.sh, dump_migrations_sql.py, проверки веток и коммитов
├── backend/
│   ├── Dockerfile, .dockerignore, requirements*.txt, alembic.ini
│   ├── alembic/              миграции master и audit
│   ├── tenant_migrations/    миграции схем тенантов (отдельный поток)
│   └── app/
│       ├── main.py, config.py, database.py
│       ├── api/              health, v1/ (auth, super_admin, tenant_admin, tenant)
│       ├── core/             security, deps (права, подписка), tenant_context, client_ip, rate_limiter
│       ├── models/master/    tenants, users, tenant_members, refresh_tokens, audit_log...
│       ├── schemas/          pydantic-схемы запросов и ответов
│       ├── services/         бизнес-логика (auth, tenants, tenant_schema_manager, subscriptions, audit)
│       ├── utils/tenant_schema.py   создание схемы tenant_<slug>
│       └── cli.py            create-superadmin, migrate-tenants
├── backend/tests/            pytest на живом PostgreSQL
└── docs/                     архитектурные решения, GIT.md
```

## Решения, отличающиеся от исходного ТЗ

Подробности и причины — в [`docs/README.md`](docs/README.md). Коротко: таблица `tenant_members`,
схема `master` вместо `public`, `timestamptz`, `NOT NULL` для JSONB и флагов, один форматтер (Black),
SQL-линтеры работают по дампу миграций, `.dockerignore` лежит в `backend/`.
