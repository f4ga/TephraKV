# PRD-001: Product Requirements Document

**Документ:** TephraKV-PRD-001
**Версия:** 1.0
**Статус:** Черновик
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v7.0, TephraKV-GTM-001 v1.0, TephraKV-COMPETITIVE-001 v1.0, TephraKV-PRICING-001 v1.0
**Классификация:** Внутренний / публичный

---

## 1. Problem Statement

Go-разработчики в latency-critical приложениях используют embedded KV-хранилища (Badger, Pebble, BoltDB), но сталкиваются с фундаментальными ограничениями:

1. **GC-паузы.** Go GC даёт stop-the-world паузы. BadgerDB в бенчмарках 2024 года показывает P99 задержки до 18 мс на случайных чтениях. SpacetimeDB аллоцирует 131 КБ на операцию чтения, что вызывает stuttering каждые 10 секунд на 500–1000 мс. Для 128-tick сервера бюджет — 7.8 мс на тик, а 5 мкс на чтение × количество операций съедает до 54% бюджета.

2. **Высокий read amplification.** LSM без KV separation даёт 3+ disk reads per GET. WiscKey снижает read amplification, но страдает от poor update и range query performance.

3. **Отсутствие per-operation durability.** RocksDB предоставляет `WriteOptions.sync`, но это глобальная опция для всего WriteBatch, а не per-operation политика. Приложения вынуждены выбирать между «всё синхронно» (медленно) и «всё асинхронно» (риск потери данных).

4. **cgo overhead.** RocksDB требует cgo. Каждый cgo-вызов стоит ~200 нс. CockroachDB сократил cgo calls с ~800K/s до ~200K/s на write-only workload, но в масштабах базы данных это остаётся ощутимой потерей.

5. **RAM-стоимость.** Redis быстр, но in-memory. Feature serving latency: Feast с Redis online store даёт 2–5 мс per lookup, managed feature stores — 2–10 мс.

**Следствие:** приложения не укладываются в SLA по хвостовой задержке, тратят лишнюю память, не могут тонко управлять durability.

---

## 2. Solution

TephraKV — embedded KV-хранилище на Go с:

- **Zero-alloc на hot path (data plane).** Ни одной аллокации на Put/Get/Delete/Scan/Batch. Гарантия — CI-бенчмарк с `-benchmem` на каждой версии Go. Arena на основе mmap-backed bump allocator: аллокация 64 байт за ~2.9 нс (vs ~40 нс для heap).
- **Per-operation durability.** Приложение выбирает политику для каждой операции: NO_SYNC, SYNC_MASTER, SYNC_LEADER.
- **Read amplification p99 < 3 (v0.1–v0.2), < 2 (v0.3+).** Достигается через LSM с KV separation (WiscKey).
- **Детерминированный хвост.** p999 GET < 5 мс (in-memory, single-node).
- **Pure Go, без cgo.** Один бинарник, ноль runtime-зависимостей.

**Ключевое техническое отличие:**

| Характеристика | Badger | Pebble | RocksDB | TephraKV |
|---|---|---|---|---|
| Zero-alloc hot path | Нет | Нет | Нет | **Да** |
| Read amp p99 (v0.3+) | 3 | 3 | 5 | **< 2** |
| Durability policy per op | Нет | Нет | Частично | **Да** |
| Pure Go | Да | Да | Нет (cgo) | **Да** |
| p999 GET (in-memory) | ~50–100 мкс | ~30–80 мкс | ~20–50 мкс | **< 5 мс** |

---

## 3. Ideal Customer Profile (ICP)

### 3.1. HFT / trading infrastructure

| Поле | Значение |
|---|---|
| **ICP** | CTO / VP Eng в prop trading или market maker, 20–200 инженеров |
| **Текущее решение** | Redis + PostgreSQL, иногда Aerospike |
| **Боль** | GC-паузы Go, p999 Redis на персистентности, стоимость RAM |
| **Критические метрики** | Aerospike обрабатывает миллионы транзакций/с с почти нулевой задержкой; DolphinDB достигает 46 мкс end-to-end. NYSE: 99% решений за <20 мкс. |
| **Триггер покупки** | Инцидент с p999 в проде |
| **Бюджет** | $50–200K/год на infra |
| **Цикл сделки** | 3–6 месяцев |
| **Канал** | Конференции (HFT), GitHub, личные связи |
| **Кто платит** | VP Eng / CTO |
| **Критерий успеха** | p999 < 1 мс в проде на их workload |

### 3.2. Game servers

| Поле | Значение |
|---|---|
| **ICP** | Tech Lead / Backend Eng в game studio, 10–100 инженеров |
| **Текущее решение** | Redis, in-memory maps, иногда Badger |
| **Боль** | SpacetimeDB: аллокации 131KB на операцию чтения вызывают stuttering каждые 10 секунд на 500–1000 мс; для 120 fps бюджет 8.3 мс, GC-пауза 5–10 мс убивает плавность |
| **Критические метрики** | 128-tick сервер: 7.8 мс на тик; 5 мкс на чтение × 100 reads = 0.5 мс; типичный DB query p99 ≤ 50 мс | |
| **Триггер покупки** | Жалобы игроков на лаги |
| **Бюджет** | $10–50K/год |
| **Цикл сделки** | 1–3 месяца |
| **Канал** | GitHub, Reddit r/golang, Discord |
| **Кто платит** | Tech Lead |
| **Критерий успеха** | Нет GC-пауз в проде |

### 3.3. AI-infra (feature stores)

| Поле | Значение |
|---|---|
| **ICP** | ML Infra Eng / CTO в AI-стартапе Series A–B, 20–100 инженеров |
| **Текущее решение** | Redis, Badger, custom |
| **Боль** | Feature serving latency: целевой p99 < 10 мс. Feast с Redis: 2–5 мс per lookup. Managed feature stores: 2–10 мс. ScyllaDB: 10 мс P99 при 200K entities/s. 500K ops/sec с P99 1–3 мс. |
| **Триггер покупки** | Рост latency при масштабировании |
| **Бюджет** | $20–100K/год |
| **Цикл сделки** | 3–6 месяцев |
| **Канал** | GitHub, конференции (MLOps), Twitter/X |
| **Кто платит** | CTO |
| **Критерий успеха** | p99 < 1 мс на feature lookup |

### 3.4. Edge / IoT

| Поле | Значение |
|---|---|
| **ICP** | Embedded Eng / CTO в IoT-компании, 10–50 инженеров |
| **Текущее решение** | SQLite, BoltDB |
| **Боль** | SQLite: только один writer одновременно; LiteFS ограничен ~100 write transactions/с из-за FUSE. Для edge-приложений требование — sub-100 мс на запись, cloud round-trip добавляет 200–400 мс. |
| **Триггер покупки** | Рост объёма данных |
| **Бюджет** | $5–20K/год |
| **Цикл сделки** | 3–6 месяцев |
| **Канал** | GitHub, конференции (embedded) |
| **Кто платит** | CTO |
| **Критерий успеха** | Работает на ограниченных ресурсах |

### 3.5. CDN / edge compute

| Поле | Значение |
|---|---|
| **ICP** | Infra Eng / CTO в CDN или edge-платформе, 50–200 инженеров |
| **Текущее решение** | RocksDB (cgo), custom |
| **Боль** | cgo overhead: каждый вызов ~200 нс; CockroachDB сократил cgo calls с ~800K/s до ~200K/s. Cloudflare CacheDB использует Rust + RocksDB на каждом edge-сервере. | |
| **Триггер покупки** | Проблемы с деплоем |
| **Бюджет** | $20–100K/год |
| **Цикл сделки** | 6–12 месяцев |
| **Канал** | Sales, конференции |
| **Кто платит** | CTO |
| **Критерий успеха** | Упрощение деплоя, latency |

---

## 4. Use Cases

### 4.1. Feature store для ML

```
Приложение: ML-сервис, 1M feature lookups/s.
Проблема: Feast с Redis даёт 2–5 мс per lookup, managed — 2–10 мс.
Решение: TephraKV embedded, p99 < 1 мс, RAM в 5× меньше.
```

### 4.2. Order book для HFT

```
Приложение: Order book, 10M ops/s.
Проблема: GC-паузы Badger дают p999 > 50 мс. NYSE: 99% решений за <20 мкс.
Решение: TephraKV zero-alloc, p999 < 5 мс.
```

### 4.3. Session store для game servers

```
Приложение: Игровые сессии, 100K ops/s на 128-tick сервере.
Проблема: Redis теряет данные при краше, SpacetimeDB stuttering каждые 10 сек.
Решение: TephraKV с SYNC_MASTER, 0 потерь, нет GC-пауз.
```

### 4.4. Edge cache для CDN

```
Приложение: Edge-кэш, 1M ops/s на узел.
Проблема: RocksDB требует cgo, Cloudflare использует Rust для CacheDB.
Решение: TephraKV pure Go, один бинарник.
```

### 4.5. Audit log для fintech

```
Приложение: Append-only audit log, compliance.
Проблема: Нет per-op durability, нет PITR.
Решение: TephraKV с SYNC_MASTER на каждой записи, PITR (v0.5).
```

---

## 5. Success Metrics

| Метрика | Цель | Срок |
|---|---|---|
| GitHub stars | 500+ | v0.2 |
| Design partners | 3 | v0.3 |
| Платящие | 1 | v0.3–v0.4 |
| ARR | $10K | v0.4 |
| ARR | $100K | v0.5 |
| ARR | $1M | v1.0 |
| p999 GET (in-memory) | < 5 мс | v0.1 |
| Read amp p99 | < 3 | v0.1–v0.2 |
| Read amp p99 | < 2 | v0.3 |
| Allocs/op hot path | 0 | v0.1 |

---

## 6. Non-Goals (до v0.6)

| Non-goal | До версии |
|---|---|
| Distributed | v0.6 |
| SQL / PostgreSQL wire | v0.6 |
| Columnar | v0.7 |
| Vector search | v0.7 |
| Multi-region | v1.0 |
| Cross-shard транзакции | v0.6 |
| Enterprise SLA | v1.0 |
| Server mode | v0.6 |
| Windows / macOS production | v0.3 |
| OLAP / аналитика | v0.6 |
| Более 100 млн ключей | v0.3 |
| Свой Raft | v1.0+ (если команда 5+) |
| QUIC для intra-cluster | v1.0+ |

---

## 7. Competitive Landscape

| Сегмент | Что используют | Почему не устраивает | Чем TephraKV лучше |
|---|---|---|---|
| HFT | Redis, Aerospike, custom C++ | RAM дорого, Redis не durable, C++ дорог | Embedded, per-op durability, zero-alloc |
| Game servers | Redis, in-memory maps | GC-паузы, потеря данных | Zero-alloc, WAL |
| AI-infra | Redis, Badger | GC-паузы, RAM | Zero-alloc, embedded |
| Edge/IoT | SQLite, BoltDB | Медленно, не масштабируется | LSM, VLog, p999 |
| CDN/edge | RocksDB (cgo), Rust+custom | cgo overhead, сложность | Pure Go, embedded |

**Сравнение с прямыми конкурентами:**

| Характеристика | Badger | Pebble | RocksDB | TephraKV |
|---|---|---|---|---|
| Язык | Go | Go | C++ | Go |
| Embedded | Да | Да | Через cgo | Да |
| Zero-alloc hot path | Нет | Нет | Нет | **Да** |
| Read amp p99 (v0.3+) | 3 | 3 | 5 | **< 2** |
| Durability policy per op | Нет | Нет | Частично | **Да** |
| p999 GET (in-memory) | ~50–100 мкс | ~30–80 мкс | ~20–50 мкс | **< 5 мс** |
| Pure Go | Да | Да | Нет | **Да** |

---

## 8. Win/Loss Analysis

### Когда выигрываем

- Клиент на Go, не хочет cgo.
- Клиент страдает от GC-пауз.
- Клиент нуждается в per-op durability.
- Клиент готов платить за p999.
- Клиент в HFT, game servers, AI-infra, edge, CDN.

### Когда проигрываем

- Клиент на C++/Rust — RocksDB (Cloudflare CacheDB на Rust).
- Клиент хочет managed — Redis Cloud, ScyllaDB Cloud.
- Клиент хочет SQL — PostgreSQL.
- Клиент не готов платить — Badger (open-source).
- Клиент в enterprise — ждёт SOC2.
- Клиент в edge/IoT с бюджетом <$5K/год — SQLite достаточно.

---

## 9. Pricing Model

| Уровень | Что входит | Цена | Сегмент |
|---|---|---|---|
| Community | Ядро (v0.1–v0.5), Apache 2.0 | $0 | Все |
| Design Partner | Early access, влияние на roadmap, приоритетные багфиксы | $10–30K/год | Ранние |
| Standard Support | SLA 99.9%, response 24ч | $50–100K/год | HFT, AI-infra |
| Premium Support | SLA 99.99%, response 4ч, dedicated engineer | $150–300K/год | HFT, fintech |
| Enterprise | SOC2, encryption, RBAC, audit log | $300–500K/год | Fintech, enterprise |

**Unit Economics:**

| Метрика | Цель |
|---|---|
| CAC | $5–20K |
| LTV | $150–500K |
| Payback period | 6–12 месяцев |
| Gross margin | 80–90% |
| Churn | < 10%/год |

---

## 10. Go-to-Market Strategy

### Фаза 1: v0.1–v0.2 (0–3 месяца)

| Канал | Действие | Метрика |
|---|---|---|
| GitHub | Публикация, README, ROADMAP | 500 stars |
| Hacker News | Show HN | 100 upvotes |
| Reddit r/golang | Пост | 50 comments |
| Twitter/X | Технические треды | 1000 followers |

### Фаза 2: v0.3–v0.4 (3–6 месяцев)

| Канал | Действие | Метрика |
|---|---|---|
| Design partners | 3–5 компаний | $10–30K/год |
| Cold outreach | 50 писем | 5 ответов |
| Конференции | HFT, MLOps | 3 доклада |
| Blog | 5 технических статей | 10K reads |

### Фаза 3: v0.5–v0.6 (6–12 месяцев)

| Канал | Действие | Метрика |
|---|---|---|
| Sales | Прямые продажи | 5–10 платящих |
| Партнёры | Интеграции | 3 партнёра |
| Конференции | GopherCon | 5 докладов |

### Фаза 4: v0.7–v1.0 (12–24 месяца)

| Канал | Действие | Метрика |
|---|---|---|
| Enterprise sales | Прямые продажи | 20–30 платящих |
| SOC2 | Аудит | $1M ARR |

**Messaging по сегментам:**

- **HFT:** «p999 < 5 мс. Zero-alloc. Per-op durability. Embedded Go. Без GC-пауз.»
- **Game servers:** «Без лагов. Без потери данных. 128-tick сервер без stuttering.»
- **AI-infra:** «Feature lookups за < 1 мс. RAM в 5× меньше. Embedded Go.»
- **Edge/IoT:** «Embedded Go. Один бинарник. Sub-100 мс на запись без cloud round-trip.»
- **CDN/edge:** «Pure Go. Без cgo. Один бинарник.»

---

## 11. Open Questions

| # | Вопрос | Как проверить | Срок |
|---|---|---|---|
| 1 | Готовы ли HFT платить за embedded Go? | 10 интервью с CTO/VP Eng | 2 недели |
| 2 | Готовы ли game servers платить вообще? | 5 интервью с Tech Lead | 1 неделя |
| 3 | Готовы ли AI-infra выбрать TephraKV вместо Redis? | 5 интервью с CTO | 1 неделя |
| 4 | Какой сегмент даст первого платящего? | Design partner pipeline | 3 месяца |
| 5 | Какая цена приемлема? | Pricing-тесты | 3 месяца |
| 6 | Готовы ли edge/IoT платить $5–20K/год? | 5 интервью | 2 недели |
| 7 | Готовы ли CDN платить за замену cgo? | 3 интервью | 2 недели |
| 8 | Достаточно ли p999 < 5 мс для HFT? | Бенчмарк на их workload | 1 месяц |

---

## 12. Dependencies & Risks

### Зависимости

| Зависимость | Версия | Риск |
|---|---|---|
| etcd/raft | v3.x | API changes, но стабилен 10 лет |
| gRPC-Go | v1.6x+ | CodecV2 API требует grpc-go 1.66+ |
| Go | Фиксированная в go.mod | Escape analysis drift |

### Риски

| Риск | P | I | Митигация |
|---|---|---|---|
| Ниша не существует | С | К | 20 интервью до v0.1 |
| HFT не готовы платить за Go | С | В | Фокус на AI-infra и game servers |
| Game servers не готовы платить | В | С | Фокус на HFT и AI-infra |
| Solo → выгорание | В | В | 3-недельные релизы |
| Zero-alloc сложен для пользователей | С | С | Документация, примеры |
| Нет revenue до v0.5 | В | В | Design partners, open-core |
| Конкуренты добавят zero-alloc | Н | В | Скорость итераций |

---

## 13. Release Criteria

| Версия | Критерии выхода |
|---|---|
| v0.1 | Zero-alloc CI зелёный; BENCH-001..003, 007; p999 < 5 мс; read amp p99 < 3; threat model + failure model |
| v0.2 | Read amp p99 < 3 (BENCH-008); WA < 5; SLA-driven compaction scheduler |
| v0.3 | WA < 3; space amp < 1.3; read amp p99 < 2; BENCH-006 |
| v0.4 | 50M GET/s на 64-core; 2000+ stars |
| v0.5 | Первый платящий; PITR 100 ГБ; capacity model |
| v0.6 | Freshness p99 < 1 с; BENCH-011..015; 3–5 нод в проде; TPC-C |
| v0.7 | SIMD ≥ 4×; compression ≥ 5×; column pruning ≥ 90% |
| v1.0 | SOC 2 Type II; SLA 99.99%; $1M ARR; joint consensus |

---

## 14. Appendix

### A. Связанные документы

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v7.0 | High-Level Design |
| TephraKV-GTM-001 v1.0 | Go-to-Market Strategy |
| TephraKV-COMPETITIVE-001 v1.0 | Competitive Analysis |
| TephraKV-PRICING-001 v1.0 | Pricing & Packaging |
| TephraKV-ENG-001 v1.1 | Engineering Charter |
| ADR-005 v4 | Transport |
| ADR-006 v2 | Raft core |
| ADR-029 v2 | Wire format |
| ADR-036 | Read path |

### B. Запрещённые утверждения

- «Лучшая СУБД» — нет идеальной СУБД.
- «Быстрее X» без раскрытия профиля.
- «Ноль аллокаций» без «на горячем пути (data plane)».
- «ACID» до v0.6.
- «HTAP» до v0.6.
- «Read amp < 2» без указания версии (v0.3+).
- «Distributed» до v0.6.
- «p999 < 5 мс» без «in-memory, single-node».
- «Свой Raft» до v1.0.
- «$15B valuation» — не заявляется.

### C. Разрешённые утверждения

- «Deterministic tail latency.»
- «p999 GET < 5 ms (in-memory, single-node).»
- «Read amplification p99 < 3 (v0.1–v0.2); < 2 (v0.3+, с KV separation).»
- «Per-operation durability policy.»
- «Two-level metrics: external + internal.»
- «Zero allocations on hot path (data plane).»
- «Cost per million operations with SLA.»
- «Own Multi-Raft, no etcd dependency (v0.6+).»
- «gRPC transport with N Raft-groups per stream.»
- «Shared WAL pool with per-shard LSN namespace.»

---

**Конец документа TephraKV-PRD-001 v1.0**