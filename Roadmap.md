# Backend Architect Roadmap

> Target outcome: from Python backend developer → backend architect who can design, justify, and ship production-ready web systems under real constraints (load, security, reliability, operability, cost).

---

## 0. Diagnostic Phase

### Цели диагностики
- Зафиксировать исходный уровень по ключевым плоскостям: **data modeling**, **конкурентность**, **интеграции**, **надёжность**, **операционка**, **безопасность**, **trade-offs**.
- Выявить “слепые зоны”: что мешает принимать архитектурные решения уверенно.
- Сформировать baseline-проект (мини-сервис) для последующего рефакторинга по модулям roadmap.

### Контрольные вопросы (самопроверка)
**Проектирование**
- Как ты превращаешь бизнес-требования в архитектуру? Какая у тебя “матрица решений”?
- Как выбираешь: монолит vs модульный монолит vs микросервисы? Какие критерии остановки?
- Что такое bounded context и где он полезен даже в монолите?

**Данные и транзакции**
- Что такое уровни изоляции и какие баги они предотвращают/не предотвращают?
- Как проектировать идемпотентность для API и фоновых задач?
- Как устроены индексы (B-Tree), почему Seq Scan может быть нормой, и как читать EXPLAIN?

**Надёжность**
- Какие failure modes бывают у сети/БД/кэша/очередей? Как ты их тестируешь?
- Чем отличаются retry, timeout, circuit breaker, bulkhead? Как они взаимодействуют?

**Нагрузка**
- Как измерить throughput/latency, определить SLO, и по ним принять архитектурное решение?
- Где bottleneck: CPU, IO, DB locks, N+1, contention, external calls?

**Безопасность**
- Что такое threat model? Какие типовые угрозы у веб-API?
- Как хранить секреты? Как устроить ротацию? Как ограничить blast radius?

**Операционка**
- Что должно быть в логах/метриках/трейсах для отладки p95 деградаций?
- Как выглядит “production readiness checklist” для сервиса?

### Критерии перехода дальше
- Ты можешь **за 60–90 минут** набросать архитектуру сервиса по ТЗ:
  - компоненты, границы, данные, API, очереди/кэш (если нужны),
  - NFR (SLO, RPO/RTO, security),
  - список рисков и trade-offs.
- Ты поднимаешь baseline-проект локально в docker-compose:
  - API + Postgres,
  - миграции,
  - базовые тесты,
  - minimal observability (структурные логи + health endpoints),
  - README с архитектурой и решениями.

---

## 1. Foundation: Production Thinking

### Module 1.1 — Requirements → Architecture (Engineering Workflow)
- 🎯 Цель  
  Научиться превращать требования в архитектуру через структуру: *assumptions → constraints → decisions → trade-offs → validation plan*.
- 🧠 Теория (концепции)  
  - Functional vs Non-functional requirements (SLO/SLA, cost, compliance, operability)
  - Architecture decision records (ADR), decision matrix
  - Definition of Done для production (runbooks, alerts, rollout)
  - “Make it work / correct / fast / operable” как последовательность
- 🔧 Практика  
  Возьми типовой домен: **Ticket Reservation** (у тебя уже был контекст). Сформируй 2 варианта архитектуры:
  1) простой монолит, 2) монолит + async workers.  
  Для каждого: ограничения, риски, почему “не микросервисы сейчас”.
- 📦 Deliverable (что должно быть в GitHub)  
  - `/docs/requirements.md` (FR/NFR, SLO, assumptions)  
  - `/docs/architecture.md` (C4 L1–L2, data model, sequences)  
  - `/docs/adr/0001-*.md` (минимум 3 ADR)  
  - `README.md` с “how to run” и “production readiness checklist v0”
- ✅ Критерии готовности  
  - Ясно прописаны SLO (p95 latency, error rate), RPO/RTO, budget constraints
  - Есть план валидации: нагрузочные тесты, chaos/failure tests (минимум сценарии)
- ⚠ Типичные ошибки  
  - “Сразу микросервисы” без доказательства необходимости
  - Отсутствие assumptions (потом решения не объяснимы)
  - NFR в конце, а не как ограничение, влияющее на архитектуру

---

### Module 1.2 — Data Modeling & Transactional Correctness (Postgres)
- 🎯 Цель  
  Проектировать схемы данных и транзакции так, чтобы система была корректной при конкуренции и сбоях.
- 🧠 Теория (концепции)  
  - Normalization vs denormalization, invariants
  - Transactions, locks, MVCC, isolation levels
  - Constraints: FK, UNIQUE, CHECK; why constraints > application-only
  - Idempotency keys, outbox/inbox patterns (вводная)
  - Query planning: indexes, EXPLAIN/ANALYZE, Seq Scan, selectivity
- 🔧 Практика  
  Реализуй **резервирование билета** и **оплату**:
  - корректная конкуренция (двойное бронирование невозможно),
  - TTL на бронь,
  - идемпотентная оплата (повторный запрос не списывает второй раз).
- 📦 Deliverable  
  - `/db/migrations/*` (Alembic)  
  - `/docs/data-model.md` (таблицы, инварианты, индексы, rationale)  
  - `/docs/queries.md` (ключевые запросы + EXPLAIN + выводы)  
  - `/tests/test_concurrency.py` (конкурентные тесты)
- ✅ Критерии готовности  
  - Инварианты обеспечены на уровне БД (constraints/locking strategy)
  - Есть объяснение выбора индексов (cardinality/selectivity)
  - Concurrency test воспроизводит гонку и показывает, что баг устранён
- ⚠ Типичные ошибки  
  - Инварианты только в коде
  - “Починим транзакцией” без понимания блокировок
  - Индексы “на всякий случай” → деградация записи/maintenance

---

### Module 1.3 — API Design: Contracts, Versioning, Error Semantics
- 🎯 Цель  
  Проектировать API как контракт: предсказуемый, совместимый, наблюдаемый.
- 🧠 Теория (концепции)  
  - REST semantics, idempotency, pagination, filtering
  - Error taxonomy (4xx/5xx), problem+json, correlation IDs
  - Backward compatibility, versioning strategies
  - Rate limiting: цели и гранулярность
- 🔧 Практика  
  Спроектируй публичное API для Ticketing:
  - reserve, confirm, cancel, pay, status,
  - idempotency-key,
  - унифицированные ошибки.
- 📦 Deliverable  
  - `openapi.yaml`  
  - `/docs/api-guidelines.md`  
  - `/docs/errors.md` (таблица кодов/причин/примеров)
- ✅ Критерии готовности  
  - OpenAPI покрывает happy-path и edge-cases
  - Ошибки диагностируемы: trace_id, error_code, retryable flag
- ⚠ Типичные ошибки  
  - Смешение transport errors и domain errors
  - Непродуманная идемпотентность (особенно для POST)
  - Нет стратегии совместимости

---

### Module 1.4 — Background Jobs & Integration Basics
- 🎯 Цель  
  Освоить async processing: очереди, фоновые воркеры, гарантии доставки, повторные попытки.
- 🧠 Теория (концепции)  
  - Task queue vs event stream
  - At-least-once, at-most-once, exactly-once (реалистично: “effectively once”)
  - Retries, backoff, DLQ, poison messages
  - Outbox (вводная) как мост БД↔очередь
- 🔧 Практика  
  Добавь фоновые задачи:
  - “истекшие брони → освобождение”,
  - “платёж подтверждён → выписка/уведомление”.
- 📦 Deliverable  
  - `/docs/async-processing.md` (гарантии, retries, DLQ стратегия)  
  - worker service (Python) + docker-compose
- ✅ Критерии готовности  
  - Любая операция безопасна к повтору (или имеет дедупликацию)
  - Есть политика retries/timeouts и описание DLQ
- ⚠ Типичные ошибки  
  - Бесконечные ретраи без дедупликации → лавина
  - Нет наблюдаемости по job lifecycle
  - “Exactly-once” без строгого определения эффекта

---

## 2. Architecture Patterns

### Module 2.1 — Modular Monolith Done Right
- 🎯 Цель  
  Научиться проектировать монолит так, чтобы он был управляемым: границы, модули, зависимости.
- 🧠 Теория  
  - Bounded contexts, module boundaries, anti-corruption layer
  - Hexagonal architecture, ports/adapters
  - Dependency rule, packaging strategies, enforcement
- 🔧 Практика  
  Перестрой baseline в модульный монолит:
  - `tickets`, `payments`, `notifications`, `admin`.
  - Запретить прямые зависимости между доменами кроме контрактов.
- 📦 Deliverable  
  - `/docs/modules.md` (границы, публичные интерфейсы)  
  - tooling: simple import-lint/arch test to enforce boundaries
- ✅ Критерии  
  - Границы проверяются тестами/линтом
  - Изменения в одном модуле минимально затрагивают другие
- ⚠ Ошибки  
  - “Модули = папки” без контрактов
  - Domain leaking через shared models

---

### Module 2.2 — CQRS (Pragmatic) + Read Models
- 🎯 Цель  
  Использовать CQRS как инструмент оптимизации read-path без религии.
- 🧠 Теория  
  - Separating read/write concerns, materialized views
  - Eventual consistency, staleness budget
  - Cache vs read model
- 🔧 Практика  
  Добавь отдельные read-модели:
  - быстрый список билетов/статусов,
  - агрегаты продаж (для админки).
- 📦 Deliverable  
  - `/docs/cqrs.md` (где применено и почему)  
  - миграции для read tables/materialized views
- ✅ Критерии  
  - Пояснён staleness budget и как он обеспечивается
  - Есть механизм репроцессинга read модели
- ⚠ Ошибки  
  - CQRS везде, даже без проблем производительности
  - Нет стратегии восстановления read модели

---

### Module 2.3 — Event-Driven Patterns (Outbox, Inbox, Saga)
- 🎯 Цель  
  Проектировать согласованность между компонентами при сбоях.
- 🧠 Теория  
  - Transactional outbox, inbox/dedup
  - Saga (orchestration vs choreography)
  - Compensating actions, state machines
- 🔧 Практика  
  Смоделируй flow “Оплата → подтверждение → выпуск билета”:
  - outbox на публикацию событий,
  - inbox на consumer side,
  - компенсация при частичном фейле.
- 📦 Deliverable  
  - `/docs/events.md` (event schema, versioning, contracts)  
  - `/docs/saga.md` (state machine + компенсации)
- ✅ Критерии  
  - Можно “проиграть” события повторно без порчи данных
  - Есть версия событий и backward compatibility
- ⚠ Ошибки  
  - События как “второй API” без контрактов
  - Отсутствие дедупликации

---

## 3. Scalability & Distributed Systems

### Module 3.1 — Performance Model & Load Testing (Not Guessing)
- 🎯 Цель  
  Уметь оценивать производительность системой измерений, а не интуицией.
- 🧠 Теория  
  - Throughput, latency, concurrency, Little’s Law (инженерно)
  - p50/p95/p99, tail latency причины
  - Capacity planning: headroom, peak factors
- 🔧 Практика  
  - Определи target SLO и нагрузочный профиль (RPS, burst, read/write ratio)
  - Напиши load tests (k6/locust) и отчёт: bottlenecks + план оптимизаций
- 📦 Deliverable  
  - `/load/` тесты + результаты  
  - `/docs/perf-report.md` (графики/таблицы, выводы, next steps)
- ✅ Критерии  
  - Есть измерения до/после оптимизаций
  - Bottleneck подтверждён профилем/метриками, а не “кажется”
- ⚠ Ошибки  
  - Микрооптимизации без подтверждённого bottleneck
  - Игнорирование tail latency

---

### Module 3.2 — Caching Strategies (Correctness First)
- 🎯 Цель  
  Применять кэширование без нарушения корректности и без “cache as database”.
- 🧠 Теория  
  - Cache-aside, write-through, write-behind
  - TTL, stampede protection, negative caching
  - Consistency: invalidation стратегии, versioned keys
- 🔧 Практика  
  Добавь Redis:
  - кеширование read-path (например, статус брони),
  - защита от stampede,
  - invalidation через события/outbox.
- 📦 Deliverable  
  - `/docs/caching.md` (что кэшируем, TTL, invalidation, риски)
- ✅ Критерии  
  - Кэш не источник истины
  - При деградации Redis сервис остаётся корректным
- ⚠ Ошибки  
  - Нет fallback при падении кэша
  - Сложная invalidation без необходимости

---

### Module 3.3 — Horizontal Scaling & Stateless Services
- 🎯 Цель  
  Спроектировать сервисы так, чтобы они масштабировались горизонтально.
- 🧠 Теория  
  - Stateless vs stateful components
  - Session management, sticky sessions anti-pattern
  - Connection pooling, DB saturation, backpressure
- 🔧 Практика  
  - Разверни API в нескольких репликах (docker-compose/k8s local)
  - Проверь поведение под нагрузкой и при рестартах
- 📦 Deliverable  
  - `/docs/scaling.md` (лимиты, saturation signals, backpressure)
- ✅ Критерии  
  - Автоматически выдерживает перезапуск реплики без потери консистентности
- ⚠ Ошибки  
  - Скрытый state в памяти процесса (кроме кэша)
  - Переподключения к БД как DDoS на Postgres

---

### Module 3.4 — Distributed Data: Replication, Sharding (Conceptual + Minimal Practice)
- 🎯 Цель  
  Понимать инструменты масштабирования данных и их цену.
- 🧠 Теория  
  - Read replicas, lag, consistency implications
  - Partitioning, sharding keys, hotspots
  - Multi-region basics, latency vs consistency
- 🔧 Практика  
  - Смоделируй read replica (логически): какие запросы можно уводить на реплику
  - Спроектируй shard key для домена ticketing и опиши миграционную стратегию
- 📦 Deliverable  
  - `/docs/distributed-data.md` (варианты, trade-offs, критерии выбора)
- ✅ Критерии  
  - Ясно описано, какие операции требуют strong consistency
- ⚠ Ошибки  
  - “Шардим заранее”
  - Игнорирование operational complexity

---

## 4. Security Engineering

### Module 4.1 — Threat Modeling & Security Baseline
- 🎯 Цель  
  Научиться думать угрозами и строить базовую защиту как часть архитектуры.
- 🧠 Теория  
  - STRIDE / attack surfaces
  - AuthN vs AuthZ, RBAC/ABAC
  - Secrets management, least privilege
  - Secure defaults, secure by design
- 🔧 Практика  
  - Сделай threat model для baseline (таблица угроз → меры)
  - Реализуй auth (JWT/OAuth2 conceptually), RBAC для admin endpoints
- 📦 Deliverable  
  - `/docs/threat-model.md`  
  - `/docs/security-baseline.md` (контроли, секреты, политики)
- ✅ Критерии  
  - Каждая ключевая угроза имеет mitigation и residual risk
- ⚠ Ошибки  
  - “Безопасность = JWT”
  - Нет контроля доступа на уровне доменных операций

---

### Module 4.2 — Web/AppSec: OWASP for API Engineers
- 🎯 Цель  
  Закрыть типовые уязвимости на уровне API и инфраструктурных интеграций.
- 🧠 Теория  
  - Injection, SSRF, broken access control, auth weaknesses
  - CSRF relevance для API, CORS, security headers
  - Input validation vs output encoding
- 🔧 Практика  
  - Добавь rate limiting, request size limits
  - Введи audit logging для чувствительных операций
  - Набор security tests (минимум: access control, injection sanity)
- 📦 Deliverable  
  - `/docs/appsec.md`  
  - `/tests/test_security.py`
- ✅ Критерии  
  - Есть контроль доступа, логирование, ограничения, безопасная обработка ошибок
- ⚠ Ошибки  
  - Подробные ошибки наружу (утечки)
  - “Валидация = pydantic”, без понимания угроз

---

### Module 4.3 — Data Protection & Compliance Thinking
- 🎯 Цель  
  Привить архитектурное мышление про PII, retention, encryption и аудит.
- 🧠 Теория  
  - Data classification, encryption at rest/in transit
  - Tokenization/pseudonymization basics
  - Retention policies, right-to-delete implications
- 🔧 Практика  
  - Определи PII в системе
  - Добавь политики хранения и удаления (миграции/скрипты)
- 📦 Deliverable  
  - `/docs/data-protection.md` (PII inventory, retention, controls)
- ✅ Критерии  
  - Понятно, какие данные где живут и как уменьшается blast radius
- ⚠ Ошибки  
  - Секреты в env без ротации/процесса
  - Отсутствие audit trail для критических действий

---

## 5. Observability & Reliability

### Module 5.1 — Observability Stack: Logs/Metrics/Traces
- 🎯 Цель  
  Сделать систему “дебажимой” по сигналам, а не по догадкам.
- 🧠 Теория  
  - RED/USE, SLI/SLO/SLA
  - Correlation IDs, structured logging
  - OpenTelemetry basics, tracing для distributed calls
- 🔧 Практика  
  - Внедри: структурные логи + метрики + трейсы
  - Дашборд: latency p95, error rate, saturation (DB connections, queue lag)
- 📦 Deliverable  
  - `/docs/observability.md` (сигналы, naming, cardinality rules)  
  - `/dashboards/` (json/инструкции)
- ✅ Критерии  
  - Любой запрос можно проследить end-to-end по trace_id
  - Есть алерты на SLO burn rate (концептуально + thresholds)
- ⚠ Ошибки  
  - Высокая cardinality метрик (label explosion)
  - Логи без контекста (нет request_id/user_id где уместно)

---

### Module 5.2 — Reliability Patterns: Timeouts, Retries, Backpressure
- 🎯 Цель  
  Строить отказоустойчивость через контроль деградации.
- 🧠 Теория  
  - Timeout budgeting
  - Retry storms, jittered backoff
  - Circuit breaker, bulkhead, rate limiting как защита
  - Graceful degradation
- 🔧 Практика  
  - Введи строгие timeouts на внешние/внутренние вызовы
  - Реализуй circuit breaker для зависимости (например, payment provider mock)
  - Добавь backpressure при перегрузке (429/503 + Retry-After)
- 📦 Deliverable  
  - `/docs/reliability.md` (политики, параметры, rationale)
- ✅ Критерии  
  - При деградации зависимости сервис остаётся предсказуемым
- ⚠ Ошибки  
  - Ретраи без таймаутов
  - Нет лимитов, всё валится каскадом

---

### Module 5.3 — Failure Testing & GameDays
- 🎯 Цель  
  Уметь проверять надёжность практикой: инъекция сбоев, сценарии восстановления.
- 🧠 Теория  
  - Fault injection, chaos-lite
  - Runbooks, incident response basics
  - Postmortems: blameless, corrective actions
- 🔧 Практика  
  Проведи 5 сценариев:
  - Postgres restart
  - Redis down
  - queue lag growth
  - payment timeout spike
  - disk full / log explosion (симуляция)
- 📦 Deliverable  
  - `/docs/gamedays.md` (шаги, ожидаемое поведение, результат, fixes)  
  - `/runbooks/` (минимум 3)
- ✅ Критерии  
  - Есть документация, что делать при инциденте
  - Выявленные проблемы превращены в actionable changes
- ⚠ Ошибки  
  - “Мы протестировали” без воспроизводимых шагов
  - Непонятно, что считается успехом/провалом

---

## 6. CI/CD & DevOps Thinking

### Module 6.1 — Build, Test, Quality Gates
- 🎯 Цель  
  Сделать пайплайн, который реально снижает риск прод-ошибок.
- 🧠 Теория  
  - Test pyramid, contract tests, smoke tests
  - Linters, formatters, type checking (mypy) как guardrails
  - Dependency scanning (conceptually), SBOM basics
- 🔧 Практика  
  GitHub Actions:
  - lint + mypy + unit tests
  - integration tests (docker-compose)
  - build artifacts
- 📦 Deliverable  
  - `.github/workflows/ci.yml`  
  - `/docs/testing-strategy.md`
- ✅ Критерии  
  - PR не мёржится без прохождения quality gates
- ⚠ Ошибки  
  - Только unit tests для системы с БД
  - Отсутствие интеграционных тестов на миграции

---

### Module 6.2 — Release Engineering: Deployments & Rollbacks
- 🎯 Цель  
  Уметь планировать релизы, минимизируя риск: canary, blue/green, migrations.
- 🧠 Теория  
  - Release strategies, feature flags
  - Backward compatible migrations (expand/contract)
  - Rollback vs roll-forward
- 🔧 Практика  
  - Введи feature flags для критичной фичи
  - Сделай миграцию по схеме expand/contract
- 📦 Deliverable  
  - `/docs/release-strategy.md`  
  - `/docs/migrations-playbook.md`
- ✅ Критерии  
  - Любая миграция имеет план отката/вперёд и совместимость версий
- ⚠ Ошибки  
  - “Сначала миграция, потом код” без совместимости
  - Rollback ломает схему данных

---

### Module 6.3 — Runtime Operations: Config, Secrets, Environments
- 🎯 Цель  
  Привить прод-операционку: конфиги, секреты, окружения, ограничение доступа.
- 🧠 Теория  
  - 12-factor практики (прагматично)
  - Config drift, secret rotation, environment parity
- 🔧 Практика  
  - Раздели config по env (dev/stage/prod) с безопасными дефолтами
  - Добавь скрипт smoke check после деплоя
- 📦 Deliverable  
  - `/docs/ops.md`  
  - `/scripts/smoke_check.py`
- ✅ Критерии  
  - Сервис поднимается одинаково локально и в CI
  - Секреты не в репозитории
- ⚠ Ошибки  
  - “Окружение уникально” → невозможно репродьюсить баги
  - Секреты в docker-compose как постоянная практика

---

## 7. Capstone Project (Production Simulation)

### Описание
Финальный проект должен симулировать реальный production: требования, ограничения, нагрузка, сбои, наблюдаемость, релизы.  
Домен: **High-Load Ticketing + Payments + Notifications** (или аналогичный marketplace/order service, но ticketing предпочтительнее как уже знакомый контекст).

### Требования (Functional)
- Пользователь может:
  - просматривать события и доступные билеты,
  - резервировать билет на ограниченное время,
  - подтвердить покупку (платёж),
  - получить подтверждение/билет (email/webhook notification).
- Администратор может:
  - управлять событиями/квотами,
  - видеть отчёты продаж,
  - получать аудит по критическим операциям.

### Нагрузка (Load Profile)
- Peak: 2 000 RPS read, 200 RPS write (reserve/pay)
- Burst: x5 на 30 секунд при “дропе билетов”
- Latency SLO:
  - read endpoints p95 < 150 ms
  - reserve/pay p95 < 300 ms
- Availability SLO: 99.9% monthly for core API
- Error budget: определить и использовать в алертинге (burn rate)

### Архитектурные ограничения
- Язык: Python (FastAPI/Flask допустимо)
- Источник истины: Postgres
- Кэш: Redis (только для ускорения read-path)
- Асинхронщина: очередь/воркеры (например, RabbitMQ/Redis queue) — с дедупликацией
- Observability: метрики + логи + трейсы (OpenTelemetry)
- Всё поднимается локально docker-compose; CI гоняет интеграционные тесты

### Нефункциональные требования (NFR)
- Correctness under concurrency:
  - no double booking
  - idempotent pay
  - consistent state machine для заказа/оплаты
- Security:
  - RBAC для админки
  - rate limiting
  - secrets hygiene
  - audit log для критических действий
- Reliability:
  - timeouts/retries policies
  - circuit breaker для payment provider
  - graceful degradation при падении Redis/очереди
- Operability:
  - dashboards + alerts по SLO
  - runbooks для top-3 инцидентов
  - structured logs + trace correlation
- Release readiness:
  - expand/contract migrations
  - feature flags
  - rollback plan

### Критерии “production-ready”
- Документация:
  - `/docs/requirements.md` (FR/NFR/SLO/RPO/RTO)
  - `/docs/architecture.md` (C4 L1–L3 + sequence diagrams)
  - `/docs/adr/` (минимум 8 ADR с trade-offs)
  - `/docs/threat-model.md`
  - `/docs/perf-report.md` (нагрузка, результаты, bottlenecks, оптимизации)
  - `/docs/observability.md` + `/docs/reliability.md`
  - `/runbooks/` (минимум 3) + `/docs/gamedays.md` (минимум 5 сценариев)
- Код/инфра:
  - миграции, constraints, индексы, идемпотентность
  - worker + DLQ стратегия
  - CI: lint/mypy/unit/integration
  - локальный запуск одной командой
- Проверки:
  - concurrency tests покрывают гонки
  - load tests достигают target профиля или честно фиксируют лимиты
  - демонстрация деградации зависимостей (Redis down / payment timeouts) без хаоса в данных

---

## Learning Strategy

### Как проходить
- Работай итерациями: **Design → Implement → Measure → Break → Fix → Document**.
- Каждый модуль начинается с документа:
  1) requirements/constraints,
  2) варианты решений,
  3) выбор + trade-offs,
  4) план валидации.
- Принцип усложнения: добавляй новый класс проблем **только после** того, как предыдущий стабилен и измерим.

### Как работать с GitHub
Рекомендуемая структура репозитория:
- `/apps/api` — API service
- `/apps/worker` — background worker
- `/db/migrations`
- `/docs` (requirements, architecture, adr, security, perf, ops)
- `/runbooks`
- `/load`
- `docker-compose.yml`
- `Makefile` (или task runner) для единых команд

### Как фиксировать прогресс
- Каждый модуль = отдельный milestone + checklist:
  - “docs done”
  - “tests done”
  - “load/failure tests done”
  - “ADR merged”
- Веди `CHANGELOG.md` (что изменилось и почему).
- Веди `DECISIONS.md` как индекс ADR.

### Как проводить self-review
Перед тем как считать модуль завершённым:
- Можешь ли ты объяснить **почему** это решение лучше альтернатив в рамках NFR?
- Что сломается при росте нагрузки x10? Что первым упирается?
- Какие 3 инцидента наиболее вероятны и как ты их заметишь/починишь?
- Где риск silent data corruption и как он предотвращён?
- Что ты сделаешь, если завтра нужно выделить модуль в отдельный сервис?

### Как часто возвращаться к пересмотру решений
- После каждого крупного изменения нагрузки/требований — пересмотр ADR (новый ADR вместо редактирования старого).
- Раз в 2–4 недели — “архитектурный ретроспектив”:  
  bottlenecks, operational pain, security gaps, simplifications.
- После каждого GameDay — обязательные корректирующие действия (или явное принятие риска).

---