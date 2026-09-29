# TephraKV — User Stories & Use Cases

**Документ:** TephraKV-USER-STORIES-001
**Версия:** 4.0
**Статус:** Черновик
**Дата:** 29 сентября 2026
**Связь:** TephraKV-HLD-000 v11.3, PRD-001 v1.3, GTM-001 v2.3, PRICING-001 v1.3, ROADMAP-001 v4.0, BACKLOG-001 v6.1, NFR-001 v1.0, API-001 v1.3, FORMAT-001 v1.3, GLOSSARY-001 v3.5, ADR-010 v3, ADR-034 v2.2
**Соответствие стандартам:** IEEE 29148-2018 (Requirements Engineering), INVEST Criteria, Gherkin (Given-When-Then)

---

## 0. Назначение, аудитория и методология

Этот документ описывает пользовательские требования к TephraKV в форме, пригодной для приёмки, приоритизации и трассировки.

### 0.1. Аудитория

| Роль | Что ищет в документе |
|---|---|
| Продукт-менеджер | Приоритизация, покрытие сегментов, критерии успеха |
| Инженер | Что именно реализовать, как проверить, что готово |
| QA | Тестируемые критерии приёмки для автоматизации |
| Инвестор | Понимание рынка, триггеров покупки, сегментов |
| Партнёр по проектированию | Соответствие продукта его задачам |

### 0.2. Методология

Документ следует стандарту IEEE 29148-2018 в части требований к инженерной документации и использует Agile-форматы для представления пользовательских историй.

**Формат User Story:** канонический шаблон `As a [role], I want [feature], so that [benefit]`.

**Формат критериев приёмки:** Gherkin (Given-When-Then) — сценарный формат, рекомендованный как предпочтительный для верифицируемых требований. Каждый сценарий конвертируется в автоматический тест.

**Валидация историй:** INVEST criteria (Independent, Negotiable, Valuable, Estimable, Small, Testable).

### 0.3. Правила трассировки

1. Каждая история имеет уникальный ID, роль, версию, критерии приёмки, ссылку на NFR и BACKLOG.
2. Если история не имеет критериев приёмки — она не идёт в релиз.
3. Если история не трассируется на BACKLOG — она не реализуется.
4. Anti-stories (что явно не поддерживаем) фиксируются наравне с историями.

### 0.4. Почему роли, а не имена

Персоны описывают **роли в системе**, а не конкретных людей. Имена не помогают инженеру понять, чью проблему он решает; помогает — профиль нагрузки, триггер покупки и бюджетный контекст. Если роль не имеет собственного бюджета и триггера — она не персона, а сегмент.

---

## 1. Роли и сегменты

### 1.1. P1 — Solo Go Developer

| Поле | Значение |
|---|---|
| **Роль** | Разработчик / небольшая команда (1–3 инженера) |
| **Профиль нагрузки** | Embedded KV для локального состояния, кэша, очереди, метаданных |
| **Нагрузка** | ≤ 100K ops/s; working set ≤ 10M ключей |
| **Текущее решение** | Badger, bbolt, Pebble, самописный file-based storage |
| **Ключевая боль** | GC-паузы, отсутствие per-op durability, непредсказуемый хвост |
| **Триггер покупки** | Второй production-инцидент с GC-паузой или потерей данных |
| **Бюджет** | $0 (Community) → $10–30K/год при выходе в prod |

### 1.2. P2 — HFT Engineer

| Поле | Значение |
|---|---|
| **Роль** | CTO / VP Engineering / Lead в prop trading или market maker (20–200 инженеров) |
| **Профиль нагрузки** | Order book, tick data, real-time risk; тик-to-trade SLA ≤ 5 мс |
| **Текущее решение** | Redis + PostgreSQL, Aerospike, QuestDB, kdb+ |
| **Ключевая боль** | p999 Redis на персистентности, GC-паузы, стоимость RAM, cgo overhead |
| **Триггер покупки** | Инцидент с p999, приведший к slippage или пропущенному риску |
| **Бюджет** | $50–200K/год (Premium Support) |

### 1.3. P3 — Game Server Engineer

| Поле | Значение |
|---|---|
| **Роль** | Tech Lead / Backend Engineer в game studio (10–100 инженеров) |
| **Профиль нагрузки** | Состояние сессий, мир, инвентарь; 128-tick (7.8 мс на тик) |
| **Текущее решение** | Redis, in-memory maps, SpacetimeDB, Badger |
| **Ключевая боль** | GC-паузы = видимые лаги; потеря состояния сессии при краше |
| **Триггер покупки** | Жалобы игроков на лаги после деплоя; потеря прогресса |
| **Бюджет** | $10–50K/год (Standard Support) |

### 1.4. P4 — AI-infra Engineer

| Поле | Значение |
|---|---|
| **Роль** | ML Infra Engineer / CTO в AI-стартапе Series A–B (20–100 инженеров) |
| **Профиль нагрузки** | Feature store, embedding storage, state для агентов |
| **Текущее решение** | Redis, Badger, DynamoDB, custom |
| **Ключевая боль** | RAM дорого; latency деградирует при масштабировании; нет per-op durability |
| **Триггер покупки** | Рост latency при масштабировании; счёт за RAM превысил бюджет |
| **Бюджет** | $20–100K/год (Standard Support) |

### 1.5. P5 — Edge/IoT Engineer

| Поле | Значение |
|---|---|
| **Роль** | Embedded Engineer / CTO в IoT-компании (10–50 инженеров) |
| **Профиль нагрузки** | Дроны, промышленное оборудование, автономные системы |
| **Текущее решение** | SQLite, BoltDB, custom |
| **Ключевая боль** | SQLite медленный при параллельной записи; нет tiered latency; cgo |
| **Триггер покупки** | Рост объёма данных; инцидент с потерей данных после power loss |
| **Бюджет** | $5–20K/год (Community или Standard) |

### 1.6. P6 — CDN/Edge Compute Engineer

| Поле | Значение |
|---|---|
| **Роль** | Infra Engineer / CTO в CDN или edge-платформе (50–200 инженеров) |
| **Профиль нагрузки** | Сотни edge-локаций, локальное состояние (кэш, маршруты) |
| **Текущее решение** | RocksDB (cgo), Rust + RocksDB, custom |
| **Ключевая боль** | CGO crash в проде; сложность сборки на новых платформах |
| **Триггер покупки** | CGO crash в проде; невозможность собрать на новой платформе |
| **Бюджет** | $20–100K/год (Standard Support) |

### 1.7. P7 — Fintech/Audit Engineer

| Поле | Значение |
|---|---|
| **Роль** | CTO / Head of Engineering в fintech Series B–C (50–300 инженеров) |
| **Профиль нагрузки** | Ledger, audit, compliance; 7 лет хранения audit log |
| **Текущее решение** | PostgreSQL + custom audit, Kafka + S3, Sumo Logic |
| **Ключевая боль** | Audit собирается вручную (2 чел × 2 нед/мес); RTO 4 ч; нет per-op durability |
| **Триггер покупки** | Аудит; регуляторное требование |
| **Бюджет** | $100–500K/год (Enterprise) |

### 1.8. P8 — Distributed Systems Engineer

| Поле | Значение |
|---|---|
| **Роль** | Staff/Principal Engineer в компании, строящей распределённую систему |
| **Профиль нагрузки** | Multi-Raft KV для конфигураций, метаданных (кластер 20 нод) |
| **Текущее решение** | TiKV, CockroachDB, etcd + custom |
| **Ключевая боль** | Внешний coordination (PD, etcd); нет per-op durability; сложность развёртывания |
| **Триггер покупки** | Сложность развёртывания; инцидент с потерей данных при failover |
| **Бюджет** | $50–200K/год (Premium Support, v0.6a+) |

---

## 2. User Stories

### 2.1. P1 — Solo Go Developer

#### US-P1-01: Быстрый старт

> **As a** solo Go developer, **I want** подключить TephraKV за 5 минут и записать первый ключ, **so that** я могу оценить, подходит ли она для моей задачи, не читая документацию целиком.

**Критерии приёмки:**

```gherkin
Feature: Быстрый старт для embedded-режима

  Scenario: Разработчик запускает первый пример
    Given установлен Go 1.24
    And TephraKV доступна по пути github.com/tephrakv/tephrakv
    When разработчик выполняет "go get github.com/tephrakv/tephrakv"
    And копирует пример из README
    And выполняет "go run main.go"
    Then пример компилируется без ошибок
    And база данных создаётся в указанной директории
    And "Put" записывает ключ-значение
    And "Get" возвращает записанное значение
    And "Close" корректно завершает работу

  Scenario: Директория для данных не существует
    Given директория "/var/lib/tephrakv" не существует
    When выполняется "Open('/var/lib/tephrakv', opts)"
    Then директория создаётся автоматически
    And база данных открывается успешно

  Scenario: Недостаточно прав на запись
    Given директория "/root/data" недоступна для записи
    When выполняется "Open('/root/data', opts)"
    Then возвращается ошибка "ErrPermissionDenied"
    And сообщение ошибки содержит путь и рекомендацию
```

**INVEST-валидация:**

| Критерий | Оценка |
|---|---|
| Independent | Да — не зависит от других историй |
| Negotiable | Да — детали API обсуждаемы |
| Valuable | Да — снижает порог входа |
| Estimable | Да — 2 SP |
| Small | Да — одна задача |
| Testable | Да — Gherkin-сценарии |

**NFR:** UX-007, PORT-001.
**BACKLOG:** CHORE-001, DOC-012.
**Версия:** v0.1.

---

#### US-P1-02: Zero-alloc без веры на слово

> **As a** solo Go developer, **I want** видеть, что `Put` и `Get` действительно не аллоцируют, **so that** я не получаю GC-паузу в проде в самый неподходящий момент.

**Контекст.** Исследование LEGO (ACM) показало, что GC-паузы в Go коррелируют с количеством ссылок в куче: microbenchmark с `map[int]*struct` дал 28–32 мс на GC-цикл, тогда как с `map[int]int` — 0.67–0.73 мс. Аллокации на hot path — не микрооптимизация, а причина деградации хвоста.

**Критерии приёмки:**

```gherkin
Feature: Zero-alloc на hot path

  Scenario: Проверка аллокаций для Get
    Given база данных открыта в embedded-режиме
    And в MemTable есть 1M ключей
    When выполняется "db.Get(key)" 1000 раз
    Then "AllocsPerRun" возвращает 0
    And CI-бенчмарк с "-benchmem" проходит

  Scenario: Проверка аллокаций для Put
    Given база данных открыта в embedded-режиме
    When выполняется "db.Put(key, value)" 1000 раз
    Then "AllocsPerRun" возвращает 0
    And escape analysis не показывает новых heap escapes

  Scenario: Проверка аллокаций для WriteBatch
    Given база данных открыта в embedded-режиме
    And батч содержит 100 операций
    When выполняется "db.WriteBatch(batch)"
    Then "AllocsPerRun" возвращает 0
```

**Нюанс.** `errors.New` в error path — это аллокация, но она вне hot path. CI исключает error paths из zero-alloc gate.

**NFR:** PERF-038, MAINT-006.
**BACKLOG:** TEST-001, FEAT-006.
**Версия:** v0.1.

---

#### US-P1-03: Понимание ошибки

> **As a** solo Go developer, **I want** получать actionable error messages, **so that** я быстро диагностирую проблему без чтения исходников.

**Критерии приёмки:**

```gherkin
Feature: Понятные ошибки

  Scenario: Ключ не найден
    Given база данных открыта
    And ключ "missing-key" отсутствует
    When выполняется "db.Get([]byte("missing-key"))"
    Then возвращается ошибка "ErrNotFound"
    And "errors.Is(err, tephrakv.ErrNotFound)" возвращает true
    And сообщение ошибки не содержит ключ

  Scenario: Слишком большой ключ
    Given база данных открыта
    When выполняется "db.Put(bigKey, value)" где bigKey > 64 Б
    Then возвращается ошибка "ErrKeyTooLarge"
    And сообщение ошибки содержит "key > 64 Б"

  Scenario: База данных закрыта
    Given база данных была закрыта
    When выполняется любая операция
    Then возвращается ошибка "ErrClosed"
```

**NFR:** UX-003, UX-011, UX-013.
**BACKLOG:** FEAT-004.
**Версия:** v0.1.

---

#### US-P1-04: Durability per operation

> **As a** solo Go developer, **I want** выбирать политику durability на уровне каждой операции, **so that** кэш пишется без fsync, а критичные данные — с fsync.

**Критерии приёмки:**

```gherkin
Feature: Выбор политики durability per operation

  Scenario: Асинхронная запись для кэша
    Given база данных открыта с ProfileLatency
    When выполняется "db.PutWithOptions(key, value, WriteOptions{Durability: NO_SYNC})"
    Then запись возвращается без fsync
    And fsync выполняется в окне group commit (100 мкс)

  Scenario: Синхронная запись для критичных данных
    Given база данных открыта
    When выполняется "db.PutWithOptions(key, value, WriteOptions{Durability: SYNC_MASTER})"
    Then fsync выполняется до возврата
    And p99 PUT < 5 мс на NVMe

  Scenario: Смешанный батч запрещён
    Given база данных открыта
    And батч содержит операции с NO_SYNC и SYNC_MASTER
    When выполняется "db.WriteBatch(batch, WriteOptions{Durability: NO_SYNC})"
    Then возвращается ошибка "ErrMixedDurability"
```

**NFR:** DUR-001, PERF-013..016.
**BACKLOG:** FEAT-004.
**Версия:** v0.1.

---

#### US-P1-05: Метрики без внешних агентов

> **As a** solo Go developer, **I want** получить метрики через `db.Metrics()`, **so that** я не разворачиваю Prometheus для embedded-приложения.

**Критерии приёмки:**

```gherkin
Feature: Метрики через публичный API

  Scenario: Two-Level Metrics
    Given база данных открыта
    And выполнено 1000 операций Put/Get
    When вызывается "db.Metrics()"
    Then возвращается "ExternalMetrics" с p50/p99/p999
    And возвращается "InternalMetrics" с cycles/op, allocs/op, read amp
    And "TierDistribution" суммируется к 100%
```

**NFR:** OBS-001..006.
**BACKLOG:** FEAT-006, TEST-006.
**Версия:** v0.1.

---

#### US-P1-06: Один бинарник, без cgo

> **As a** solo Go developer, **I want** один статический бинарник без cgo и внешних зависимостей, **so that** сборка была тривиальной и переносимой.

**Критерии приёмки:**

```gherkin
Feature: Сборка без cgo

  Scenario: Сборка проекта
    Given установлен Go 1.24
    When выполняется "go build ./..."
    Then сборка завершается успешно
    And размер бинарника < 5 МБ
    And "go.mod" не содержит внешних runtime-зависимостей
    And "CGO_ENABLED=0" установлен
```

**NFR:** PORT-010, PORT-012, PERF-062.
**BACKLOG:** CHORE-001.
**Версия:** v0.1.

---

### 2.2. P2 — HFT Engineer

#### US-P2-01: p999 в проде

> **As an** HFT engineer, **I want** видеть p999 GET < 500 нс на 1M ключей в проде, **so that** тик-to-trade остаётся ≤ 5 мс.

**Критерии приёмки:**

```gherkin
Feature: Tiered latency для HFT

  Scenario: T2 tier (L3-resident)
    Given working set 1M ключей
    And key 16 Б, value 64 Б
    And YCSB C workload
    And CPU с L3 ≥ 16 МБ
    When выполняются GET операции
    Then p50 GET < 100 нс
    And p99 GET < 300 нс
    And p999 GET < 500 нс
    And tier distribution ≥ 80% в L1/L2/L3
```

**NFR:** PERF-004..006, PERF-026.
**BACKLOG:** FEAT-001, FEAT-006, DOC-011.
**Версия:** v0.1.

---

#### US-P2-02: Durability без потери latency

> **As an** HFT engineer, **I want** писать критичные ордера с `SYNC_MASTER`, а рыночные данные — с `NO_SYNC`, **so that** я не плачу latency за durability там, где она не нужна.

**Критерии приёмки:**

```gherkin
Feature: Mixed durability в одном процессе

  Scenario: Order book с SYNC_MASTER
    Given база данных открыта с ProfileLatency
    And DefaultDurability = NO_SYNC
    When выполняется "db.PutWithOptions(orderKey, orderValue, WriteOptions{Durability: SYNC_MASTER})"
    Then fsync выполняется до возврата
    And p99 PUT < 5 мс на NVMe

  Scenario: Market data с NO_SYNC
    Given база данных открыта с ProfileLatency
    When выполняется "db.PutWithOptions(tickKey, tickValue, WriteOptions{Durability: NO_SYNC})"
    Then запись возвращается без fsync
    And p99 PUT < 1 мкс
```

**NFR:** PERF-013..016, DUR-001.
**BACKLOG:** FEAT-004, TEST-004.
**Версия:** v0.1.

---

#### US-P2-03: Линейная масштабируемость

> **As an** HFT engineer, **I want** видеть линейный scaling до 64 ядер, **so that** я масштабирую throughput без переписывания архитектуры.

**Критерии приёмки:**

```gherkin
Feature: Shard-per-core scaling

  Scenario: Scaling до 64 ядер
    Given сервер с 64 ядрами
    And working set помещается в RAM
    When выполняются GET операции
    Then throughput ≥ 50M ops/s
    And scaling factor ≥ 0.45× на 64 cores
    And нет contention на shared state
```

**NFR:** PERF-033, SCALE-001..003.
**BACKLOG:** FEAT-031..033.
**Версия:** v0.4.

---

#### US-P2-04: Distributed cluster без etcd

> **As an** HFT engineer, **I want** развернуть distributed KV без внешнего etcd, **so that** я сокращаю операционные компоненты и точки отказа.

**Критерии приёмки:**

```gherkin
Feature: Distributed KV без etcd

  Scenario: Кластер из 5 нод
    Given 5 нод в одной зоне доступности
    And Dragonboat Multi-Raft
    When клиент пишет с SYNC_MAJORITY
    Then freshness p99 < 1 с
    And leader failover p99 < 3 с
    And нет внешнего etcd/PD
    And 10000 Raft groups на узел
```

**NFR:** PERF-019..024, SCALE-008, AVAIL-005..011.
**BACKLOG:** FEAT-041..051.
**Версия:** v0.6a.

---

### 2.3. P3 — Game Server Engineer

#### US-P3-01: Нет GC-пауз

> **As a** game server engineer, **I want** чтобы сервер не имел GC-пауз на hot path, **so that** тики не проскакивали и игроки не видели лагов.

**Контекст.** Кейс BadgerDB: высокий write throughput (100 МБ/с) приводил к stuttering по 10+ секунд. При 120 fps бюджете кадра 8.3 мс пауза 10 секунд = ~1200 пропущенных кадров.

**Критерии приёмки:**

```gherkin
Feature: Zero GC pauses на hot path

  Scenario: Игровой сервер с 128 Hz tick
    Given база данных открыта в embedded-режиме
    And GODEBUG=gctrace=1
    When выполняются Put/Get операции на hot path
    Then "AllocsPerOp" == 0
    And GC-циклов на hot-path-only workload = 0
    And p999 GET < 500 нс (T2, 1M keys)
    And нет пауз > 1 мс за 300 секунд измерения
```

**NFR:** PERF-005..006, PERF-038.
**BACKLOG:** FEAT-001, TEST-001.
**Версия:** v0.1.

---

#### US-P3-02: Session state без потери при краше

> **As a** game server engineer, **I want** сохранять состояние игровых сессий с durability, **so that** при краше сервера игроки не теряли прогресс.

**Критерии приёмки:**

```gherkin
Feature: Durability для игровых сессий

  Scenario: Запись состояния сессии
    Given база данных открыта
    When выполняется "db.PutWithOptions(sessionKey, sessionState, WriteOptions{Durability: SYNC_MASTER})"
    Then fsync выполняется до возврата

  Scenario: Crash-recovery
    Given база данных с 1M записей
    And SYNC_MASTER политика
    When процесс убивается через "kill -9"
    And база данных перезапускается
    Then все подтверждённые записи восстановлены
    And recovery time < 5 с на 10 ГБ
    And 10000/10000 crash-recovery тестов проходят
```

**NFR:** DUR-001..005, AVAIL-001..002.
**BACKLOG:** FEAT-003, TEST-003.
**Версия:** v0.1.

---

#### US-P3-03: TTL для сессий

> **As a** game server engineer, **I want** TTL на ключи сессий, **so that** я не управляю очисткой вручную.

**Критерии приёмки:**

```gherkin
Feature: TTL для игровых сессий

  Scenario: Автоматическое истечение
    Given база данных открыта с EnableTTL
    When выполняется "db.PutWithOptions(sessionKey, sessionState, WriteOptions{TTL: 30*time.Minute})"
    Then ключ истекает через 30 минут
    And auto-GC удаляет expired ключи
    And TTL не блокирует hot path
```

**NFR:** OBS-008.
**BACKLOG:** FEAT-035, FEAT-036.
**Версия:** v0.5.

---

### 2.4. P4 — AI-infra Engineer

#### US-P4-01: Feature lookup с низкой latency

> **As an** AI-infra engineer, **I want** feature lookup < 1 мс p99, **so that** online inference не деградировал.

**Критерии приёмки:**

```gherkin
Feature: Feature lookup latency

  Scenario: In-process lookup
    Given база данных открыта в embedded-режиме
    And 10M features в RAM
    When выполняется "db.Get(featureKey)"
    Then p99 GET < 1 мкс (T3, RAM-resident)
    And RAM per key ≤ 4 Б (с Bloom)

  Scenario: gRPC lookup
    Given база данных открыта в server mode
    And 10M features
    When клиент на Python выполняет "Get" через gRPC
    Then p99 GET < 10 мс
```

**NFR:** PERF-008, PERF-064.
**BACKLOG:** FEAT-001, FEAT-006.
**Версия:** v0.1 (embedded), v0.6a (server).

---

#### US-P4-02: RAM-эффективность

> **As an** AI-infra engineer, **I want** использовать диск вместо RAM для cold features, **so that** я снижаю стоимость кластера.

**Критерии приёмки:**

```gherkin
Feature: Tiered latency для feature store

  Scenario: Hot and cold features
    Given база данных открыта
    And 500M features
    And 50M hot features (топ-1%)
    When выполняются Get операции
    Then T3 (RAM) p999 < 2 мкс для hot features
    And T4 (NVMe) p999 < 50 мкс для cold features
    And RAM относительно Redis при durability ≤ 1.5×
```

**NFR:** PERF-009, PERF-012, PERF-065.
**BACKLOG:** FEAT-001, FEAT-005a/b, DOC-004.
**Версия:** v0.1..v0.3.

---

#### US-P4-03: Server mode для polyglot

> **As an** AI-infra engineer, **I want** обращаться к TephraKV по gRPC из Python-сервиса inference, **so that** я не привязываюсь к Go.

**Критерии приёмки:**

```gherkin
Feature: gRPC server mode

  Scenario: Python client
    Given база данных открыта в server mode
    And TLS 1.3
    When Python-клиент выполняет "Get" через gRPC
    Then запрос обрабатывается
    And CodecV2 используется (не V1 bridge)
    And allocs/op на codec ≤ 2
    And 13+ языков клиентов доступны
```

**NFR:** PERF-039, SEC-008, OBS-014.
**BACKLOG:** FEAT-048, FEAT-049.
**Версия:** v0.6a.

---

### 2.5. P5 — Edge/IoT Engineer

#### US-P5-01: Работа на ограниченных ресурсах

> **As an** edge engineer, **I want** запускать TephraKV на 1–2 vCPU / 4 ГБ RAM, **so that** я работаю на edge-устройстве.

**Критерии приёмки:**

```gherkin
Feature: Работа на ограниченных ресурсах

  Scenario: Raspberry Pi 5
    Given 2 vCPU, 4 ГБ RAM
    And ARM64
    When база данных открыта с MemTableSize=64 МБ
    And 1M keys
    Then RAM ≤ 110 МБ
    And бинарник < 5 МБ
    And WAL segment = 64 МБ
```

**NFR:** PERF-062, SCALE-004, PORT-001..002.
**BACKLOG:** CHORE-001, FEAT-003.
**Версия:** v0.1 (ARM64), v0.3 (macOS/Windows build).

---

#### US-P5-02: Автономная работа без сети

> **As an** edge engineer, **I want** чтобы TephraKV работал полностью offline, **so that** устройство не зависит от cloud.

**Критерии приёмки:**

```gherkin
Feature: Offline-режим

  Scenario: Работа без сети
    Given embedded-режим
    And нет network connection
    When выполняются Put/Get операции
    Then все операции работают локально
    And recovery после power loss без потерь (SYNC_MASTER)
    And zero runtime dependencies
```

**NFR:** PORT-010, DUR-001..004.
**BACKLOG:** FEAT-001, FEAT-003.
**Версия:** v0.1.

---

### 2.6. P6 — CDN/Edge Compute Engineer

#### US-P6-01: Отсутствие cgo

> **As a** CDN engineer, **I want** pure Go без cgo, **so that** я не имею операционных проблем с CGO crash и сложностью сборки.

**Критерии приёмки:**

```gherkin
Feature: Pure Go без cgo

  Scenario: Сборка без cgo
    Given Go 1.24
    When выполняется "go build -a -tags netgo -ldflags '-extldflags \"-static\"' ./..."
    Then сборка успешна
    And нет cgo-зависимостей
    And бинарник статический
```

**NFR:** PORT-012.
**BACKLOG:** CHORE-001.
**Версия:** v0.1.

---

#### US-P6-02: Простой деплой на сотни edge-локаций

> **As a** CDN engineer, **I want** деплоить один бинарник на 330+ edge-локаций, **so that** деплой занимает минуты, а не часы.

**Критерии приёмки:**

```gherkin
Feature: Простой деплой

  Scenario: Деплой на edge-локацию
    Given бинарник < 5 МБ
    When он копируется на edge-сервер
    And запускается с Options (без внешних файлов)
    Then база данных работает в embedded-режиме
    And деплой занимает < 1 минуты
```

**NFR:** PERF-062, PORT-001.
**BACKLOG:** CHORE-001, FEAT-007.
**Версия:** v0.1.

---

### 2.7. P7 — Fintech/Audit Engineer

#### US-P7-01: Audit log

> **As a** fintech engineer, **I want** append-only audit log с WORM-режимом, **so that** я прохожу аудит и соответствую GDPR.

**Критерии приёмки:**

```gherkin
Feature: Audit log

  Scenario: Append-only с WORM
    Given база данных открыта с ProfileCompliance
    And AuditConfig{Immutable: true}
    When выполняется операция Put
    Then запись фиксируется в audit log
    And audit log не может быть изменён или удалён
    And export в S3/Glacier доступен

  Scenario: Retention
    Given audit log с retention 7 лет
    When проходит 1 год
    Then audit logs старше 7 лет удаляются
    And audit logs младше 7 лет сохранены
```

**NFR:** SEC-019..021, COMP-006..008.
**BACKLOG:** FEAT-019a, FEAT-019b.
**Версия:** v0.5 (log), v0.7 (export).

---

#### US-P7-02: PITR

> **As a** fintech engineer, **I want** восстанавливать БД на произвольный момент в прошлом, **so that** я соответствую требованиям регулятора.

**Критерии приёмки:**

```gherkin
Feature: Point-in-Time Recovery

  Scenario: Восстановление на момент в прошлом
    Given база данных с EnablePITR
    And retention 30 дней
    When выполняется "db.RestoreTo(timestamp)" где timestamp = 15 дней назад
    Then база данных восстанавливается на указанный момент
    And RTO на 100 ГБ < 60 с
    And нет потери данных до timestamp
```

**NFR:** DUR-007..010.
**BACKLOG:** FEAT-037, FEAT-038.
**Версия:** v0.5.

---

#### US-P7-03: Encryption at rest

> **As a** fintech engineer, **I want** encryption at rest (AES-256-GCM), **so that** я соответствую требованиям compliance.

**Критерии приёмки:**

```gherkin
Feature: Encryption at rest

  Scenario: Шифрование данных
    Given база данных открыта с ProfileCompliance
    And EncryptionConfig{KeyProvider: KMS, KeyRotationInterval: 30d}
    When данные записываются на диск
    Then данные зашифрованы AES-256-GCM
    And key rotation выполняется каждые 30 дней
    And overhead ≤ 20% CPU (AES-NI)
    And ключи не в логах
```

**NFR:** SEC-013..017.
**BACKLOG:** FEAT-058.
**Версия:** v0.6b.

---

#### US-P7-04: SOC 2

> **As a** fintech engineer, **I want** видеть SOC 2 Type II у вендора, **so that** я прохожу procurement.

**Критерии приёмки:**

```gherkin
Feature: SOC 2 compliance

  Scenario: Аудит
    Given компания-вендор
    When проходит SOC 2 Type II аудит
    Then сертификат получен
    And audit window ≥ 6 месяцев
    And SBOM (SPDX) доступен
    And SLSA Level 3 достигнут
```

**NFR:** COMP-001, COMP-011..012.
**BACKLOG:** (не реализовано в коде, только процессы).
**Версия:** v1.0.

---

### 2.8. P8 — Distributed Systems Engineer

#### US-P8-01: Multi-Raft без etcd

> **As a** distributed engineer, **I want** Multi-Raft KV без внешнего etcd/PD, **so that** я сокращаю компоненты и точки отказа.

**Критерии приёмки:**

```gherkin
Feature: Multi-Raft без etcd

  Scenario: Кластер из 5 нод
    Given 5 нод
    And Dragonboat Multi-Raft
    When кластер запускается
    Then нет внешнего etcd/PD
    And 10000 Raft groups на узел
    And member list управляется через Raft
```

**NFR:** PERF-034..037, SCALE-008..013.
**BACKLOG:** FEAT-041..043.
**Версия:** v0.6a.

---

#### US-P8-02: Per-op durability в distributed

> **As a** distributed engineer, **I want** выбирать политику durability per operation в distributed, **so that** я отделяю критичные операции от некритичных.

**Критерии приёмки:**

```gherkin
Feature: Per-op durability в distributed

  Scenario: SYNC_MAJORITY
    Given кластер 5 нод
    When выполняется "db.PutWithOptions(key, value, WriteOptions{Durability: SYNC_MAJORITY})"
    Then fsync на кворуме до возврата
    And 0 потерь при отказе меньшинства

  Scenario: SYNC_LEADER
    Given кластер 5 нод
    When выполняется "db.PutWithOptions(key, value, WriteOptions{Durability: SYNC_LEADER})"
    Then fsync локально на лидере до возврата
    And возможна потеря при смене лидера

  Scenario: AllocsPerDistributedWrite
    Given distributed режим
    When выполняется distributed write
    Then "AllocsPerDistributedWrite" ≤ 10
```

**NFR:** PERF-040, DUR-001..005 (distributed).
**BACKLOG:** FEAT-044, ADR-010 v3.
**Версия:** v0.6a.

---

#### US-P8-03: Snapshot transfer с resume

> **As a** distributed engineer, **I want** инкрементальный snapshot transfer с resume, **so that** добавление ноды занимает минуты, а не часы, и не падает при обрыве сети.

**Критерии приёмки:**

```gherkin
Feature: Snapshot transfer

  Scenario: Добавление ноды
    Given кластер 5 нод
    And 10000 Raft groups
    When новая нода присоединяется
    Then snapshot transfer инкрементальный (chunked)
    And resume с точки обрыва
    And throughput ≥ 100 МБ/с на ноду
    And при смене лидера — новый snapshot, resume с начала
```

**NFR:** PERF-025, SCALE-012.
**BACKLOG:** FEAT-045.
**Версия:** v0.6a.

---

#### US-P8-04: Линейные чтения без риска stale

> **As a** distributed engineer, **I want** линейные чтения без риска stale при clock skew, **so that** я не нарушаю консистентность при рассинхроне часов.

**Критерии приёмки:**

```gherkin
Feature: Linearizable чтения

  Scenario: ReadIndex (default)
    Given кластер 5 нод
    And clock skew между нодами
    When выполняется "db.GetWithOptions(key, ReadOptions{Consistency: ConsistencyReadIndex})"
    Then чтение linearizable
    And clock skew не влияет на консистентность

  Scenario: Lease Read (опционально)
    Given кластер 5 нод
    And bounded clock drift < 1000 ppm
    When выполняется "db.GetWithOptions(key, ReadOptions{Consistency: ConsistencyLeaseRead})"
    Then чтение linearizable
    And требуется монотонный raw clock
```

**NFR:** CONS-005..006.
**BACKLOG:** FEAT-043, ADR-036 v1.
**Версия:** v0.6a.

---

## 3. Use Cases

Сценарий отличается от истории тем, что описывает конкретное взаимодействие с шагами, альтернативными потоками и ожидаемым результатом. Формат соответствует рекомендациям IEEE 29148 для поведенческих требований.

### 3.1. UC-01: Первая интеграция за 30 минут

| Поле | Значение |
|---|---|
| **Роль** | P1 — Solo Go Developer |
| **Предусловие** | Go 1.24, пустая директория |
| **Триггер** | Разработчик хочет оценить TephraKV |
| **Основной поток** | 1. `go get github.com/tephrakv/tephrakv`<br>2. Скопировать пример из README<br>3. `go run main.go`<br>4. Создать базу, записать ключ, прочитать, закрыть |
| **Альтернативный поток** | Директория не существует → `Open` создаёт её. Нет прав → `ErrPermissionDenied` с подсказкой |
| **Постусловие** | Работает, метрики доступны |
| **Версия** | v0.1 |

### 3.2. UC-02: HFT order book

| Поле | Значение |
|---|---|
| **Роль** | P2 — HFT Engineer |
| **Предусловие** | TephraKV встроена в trading service |
| **Триггер** | Обновление order book |
| **Основной поток** | 1. Order book обновления с `SYNC_MASTER`<br>2. Market data с `NO_SYNC`<br>3. Read path — `Get` по order ID<br>4. Метрики через `db.Metrics()`, экспорт в Prometheus |
| **Альтернативный поток** | fsync latency spike → `wal_fsync_latency_p99` spike → алерт |
| **Постусловие** | p999 GET < 500 нс, тик-to-trade ≤ 5 мс |
| **Версия** | v0.1 (embedded), v0.5 (Prometheus) |

### 3.3. UC-03: Game session state

| Поле | Значение |
|---|---|
| **Роль** | P3 — Game Server Engineer |
| **Предусловие** | 128-tick game server |
| **Триггер** | Игровая сессия активна |
| **Основной поток** | 1. Состояние сессии с `SYNC_MASTER`<br>2. Кэши (leaderboard) — `NO_SYNC`<br>3. TTL на сессии после disconnect<br>4. При краше — recovery < 5 с |
| **Альтернативный поток** | `kill -9` → recovery восстанавливает все подтверждённые записи |
| **Постусловие** | Нет видимых лагов; потеря ≤ окно group commit |
| **Версия** | v0.1 (базово), v0.5 (TTL) |

### 3.4. UC-04: Feature store

| Поле | Значение |
|---|---|
| **Роль** | P4 — AI-infra Engineer |
| **Предусловие** | Python inference service |
| **Триггер** | Online inference request |
| **Основной поток** | 1. Feature computation пишет через gRPC<br>2. Inference service читает через gRPC<br>3. Hot features (топ-1%) → L3<br>4. Cold features → NVMe |
| **Альтернативный поток** | p99 lookup > 10 мс → tier distribution сдвиг в T4 → tier-aware compaction |
| **Постусловие** | p99 < 10 мс через gRPC, p999 < 500 нс in-process |
| **Версия** | v0.6a (server mode) |

### 3.5. UC-05: Edge drone fleet

| Поле | Значение |
|---|---|
| **Роль** | P5 — Edge/IoT Engineer |
| **Предусловие** | 400 дронов, 80 edge-серверов |
| **Триггер** | Телеметрия от дрона |
| **Основной поток** | 1. Дрон пишет локально<br>2. Edge-сервер агрегирует<br>3. При подключении — синхронизация<br>4. Работа offline автономно |
| **Альтернативный поток** | Power loss → recovery при `Open` → все `SYNC_MASTER` записи восстановлены |
| **Постусловие** | Работает offline, recovery без потерь |
| **Версия** | v0.1 (ARM64), v0.6a (distributed) |

### 3.6. UC-06: CDN edge KV

| Поле | Значение |
|---|---|
| **Роль** | P6 — CDN/Edge Compute Engineer |
| **Предусловие** | 330+ edge-локаций |
| **Триггер** | Деплой на edge |
| **Основной поток** | 1. Один бинарник на каждую локацию<br>2. Embedded-режим<br>3. TTL на кэш-записи<br>4. Периодическая синхронизация с origin |
| **Альтернативный поток** | Миграция с RocksDB (cgo) → `go build` без cgo → CGO crash невозможен |
| **Постусловие** | Деплой минут, нет cgo-проблем |
| **Версия** | v0.1 (embedded), v0.5 (TTL) |

### 3.7. UC-07: Ledger с audit

| Поле | Значение |
|---|---|
| **Роль** | P7 — Fintech/Audit Engineer |
| **Предусловие** | Fintech ledger service |
| **Триггер** | Ledger entry |
| **Основной поток** | 1. Ledger entries с `SYNC_MASTER`<br>2. Audit log фиксирует каждую операцию<br>3. Encryption at rest<br>4. PITR 30 дней |
| **Альтернативный поток** | Аудит → export в S3/Glacier → audit trail полный |
| **Постусловие** | SOC 2 compliance, audit trail полный |
| **Версия** | v0.5 (PITR, audit), v0.6b (encryption), v1.0 (SOC 2) |

### 3.8. UC-08: Distributed KV без etcd

| Поле | Значение |
|---|---|
| **Роль** | P8 — Distributed Systems Engineer |
| **Предусловие** | 5 нод в одной зоне |
| **Триггер** | Клиентская операция |
| **Основной поток** | 1. 5 нод TephraKV<br>2. Dragonboat Multi-Raft, 10000 групп<br>3. Клиент пишет с `SYNC_MAJORITY`<br>4. Убить 1 ноду — кластер работает<br>5. Добавить ноду — snapshot с resume |
| **Альтернативный поток** | Лидер падает → election p99 < 3 с → SYNC_MAJORITY записи не потеряны |
| **Постусловие** | Freshness < 1 с, failover < 3 с, без etcd |
| **Версия** | v0.6a |

---

## 4. Anti-stories

Anti-stories фиксируют, что явно **не поддерживается**. Это защита от scope creep и необоснованных ожиданий. Формат соответствует рекомендациям IEEE 29148 для ограничений области применения.

| ID | Anti-story | Обоснование |
|---|---|---|
| AS-01 | Замена PostgreSQL с сохранением SQL | Разные ниши. TephraKV — KV, не реляционная СУБД. SQL subset только с v0.6b |
| AS-02 | In-memory-only cache вместо Redis | TephraKV — дисковая БД с tiered latency. Если durability не нужна — Redis лучше |
| AS-03 | OLAP-нагрузки в v0.1–v0.5 | Columnar storage — отдельная инфраструктура. Только KV до v0.6b |
| AS-04 | Zero-alloc на server mode | Принципиально невозможно: framing — control plane (ADR-030) |
| AS-05 | Работа на HDD | Физическое ограничение: десятки мс на random read. T4 требует NVMe/SSD |
| AS-06 | Замена Redis Cluster без изменений | API другой. Миграция требует re-import |
| AS-07 | Мультиарендность до v1.0 | Требует RBAC, resource quotas, отдельных namespace |
| AS-08 | Векторный поиск до v0.7+ | HNSW — отдельная инфраструктура, не ядро KV |
| AS-09 | Отсутствие compaction-пауз на любом workload | LSM требует компакции. SLA-driven scheduler ограничивает, но не исключает |
| AS-10 | Гарантия latency на SATA SSD | Физическое ограничение: fsync 5–15 мс. PERF-013 специфицирован для NVMe |
| AS-11 | Гарантия нулевой потери при NO_SYNC | Осознанный компромисс. Для нулевых потерь — SYNC_MASTER |
| AS-12 | Поддержка ARM64 с теми же latency-целями | На ARM64 метрика в ticks, не cycles. Прямое сравнение запрещено |

---

## 5. Трассировка

### 5.1. Story → NFR

| Story | NFR |
|---|---|
| US-P1-01 | UX-007 |
| US-P1-02 | PERF-038, MAINT-006 |
| US-P1-03 | UX-003, UX-011, UX-013 |
| US-P1-04 | DUR-001, PERF-013..016 |
| US-P1-05 | OBS-001..006 |
| US-P1-06 | PORT-010, PORT-012, PERF-062 |
| US-P2-01 | PERF-004..006, PERF-026 |
| US-P2-02 | PERF-013..016, DUR-001 |
| US-P2-03 | PERF-033, SCALE-001..003 |
| US-P2-04 | PERF-019..024, SCALE-008, AVAIL-005..011 |
| US-P3-01 | PERF-005..006, PERF-038 |
| US-P3-02 | DUR-001..005, AVAIL-001..002 |
| US-P3-03 | OBS-008 |
| US-P4-01 | PERF-008, PERF-064 |
| US-P4-02 | PERF-009, PERF-012, PERF-065 |
| US-P4-03 | PERF-039, SEC-008, OBS-014 |
| US-P5-01 | PERF-062, SCALE-004, PORT-001..002 |
| US-P5-02 | PORT-010, DUR-001..004 |
| US-P6-01 | PORT-012 |
| US-P6-02 | PERF-062, PORT-001 |
| US-P7-01 | SEC-019..021, COMP-006..008 |
| US-P7-02 | DUR-007..010 |
| US-P7-03 | SEC-013..017 |
| US-P7-04 | COMP-001, COMP-011..012 |
| US-P8-01 | PERF-034..037, SCALE-008..013 |
| US-P8-02 | PERF-040, DUR-001..005 |
| US-P8-03 | PERF-025, SCALE-012 |
| US-P8-04 | CONS-005..006 |

### 5.2. Story → BACKLOG

| Story | BACKLOG |
|---|---|
| US-P1-01 | CHORE-001, DOC-012 |
| US-P1-02 | TEST-001, FEAT-006 |
| US-P1-03 | FEAT-004 |
| US-P1-04 | FEAT-004 |
| US-P1-05 | FEAT-006, TEST-006 |
| US-P1-06 | CHORE-001 |
| US-P2-01 | FEAT-001, FEAT-006, DOC-011 |
| US-P2-02 | FEAT-004, TEST-004 |
| US-P2-03 | FEAT-031..033 |
| US-P2-04 | FEAT-041..051 |
| US-P3-01 | FEAT-001, TEST-001 |
| US-P3-02 | FEAT-003, TEST-003 |
| US-P3-03 | FEAT-035, FEAT-036 |
| US-P4-01 | FEAT-001, FEAT-006 |
| US-P4-02 | FEAT-001, FEAT-005a/b, DOC-004 |
| US-P4-03 | FEAT-048, FEAT-049 |
| US-P5-01 | CHORE-001, FEAT-003 |
| US-P5-02 | FEAT-001, FEAT-003 |
| US-P6-01 | CHORE-001 |
| US-P6-02 | CHORE-001, FEAT-007 |
| US-P7-01 | FEAT-019a, FEAT-019b |
| US-P7-02 | FEAT-037, FEAT-038 |
| US-P7-03 | FEAT-058 |
| US-P7-04 | — |
| US-P8-01 | FEAT-041..043 |
| US-P8-02 | FEAT-044, ADR-010 v3 |
| US-P8-03 | FEAT-045 |
| US-P8-04 | FEAT-043, ADR-036 v1 |

---

## 6. Приоритизация

Приоритизация основана на матрице RICE (Reach × Impact × Confidence / Effort):

| Приоритет | Роли | Обоснование |
|---|---|---|
| **Must-have (v0.1)** | P1, P2, P3 | Core value proposition. Без них продукт не имеет смысла |
| **Should-have (v0.3–v0.5)** | P4, P5, P6 | Расширяют аудиторию, но не критичны для первых релизов |
| **Could-have (v0.6a+)** | P7, P8 | Enterprise и distributed сегменты |

---

## 7. Открытые вопросы

| № | Вопрос | Статус |
|---|---|---|
| 1 | Готовы ли HFT-инженеры платить $150–300K/год? | Не проверено |
| 2 | Конверсия Community → Design Partner? | Не проверено |
| 3 | Реальный интерес к embedded distributed (P8)? | Гипотеза: 20 компаний. Не подтверждено |
| 4 | Что важнее для P5: free tier или per-op durability? | Не проверено |
| 5 | Сколько пользователей работают на SATA SSD? | Не знаем |
| 6 | Реальная частота GC-инцидентов у P1? | Не знаем |
| 7 | Нужно ли явно документировать, что LSM физически не может исключить compaction-паузы? | Открыто |

---

## 8. Связанные документы

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v11.3 | Архитектура, позиционирование |
| TephraKV-PRD-001 v1.3 | ICP, use cases, TAM |
| TephraKV-GTM-001 v2.3 | Go-to-Market Strategy |
| TephraKV-PRICING-001 v1.3 | Pricing & Packaging |
| TephraKV-ROADMAP-001 v4.0 | OKR, timeline |
| TephraKV-BACKLOG-001 v6.1 | Задачи по версиям |
| TephraKV-NFR-001 v1.0 | Нефункциональные требования |
| TephraKV-API-001 v1.3 | Public API |
| TephraKV-FORMAT-001 v1.3 | Data formats |
| TephraKV-GLOSSARY-001 v3.5 | Термины |
| ADR-010 v3 | Durability semantics |
| ADR-034 v2.2 | Engineering practices |
| IEEE 29148-2018 | Requirements Engineering |
| INVEST Criteria | User Story Quality Filter |
| Gherkin Reference | BDD Scenario Format |

---

**Конец документа TephraKV-USER-STORIES-001 v4.0**