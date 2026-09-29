# TephraKV — High-Level Design

**Документ:** TephraKV-HLD-000
**Версия:** 11.3
**Дата:** 23 сентября 2026
**Статус:** Утверждён к реализации
**Заменяет:** HLD-000 v11.2

**Связь:** [PRD-001 v1.4](#) · [GTM-001 v2.4](#) · [PRICING-001 v1.4](#) · [ROADMAP-001 v4.1](#) · [BACKLOG-001 v6.1](#) · [GLOSSARY-001 v3.5](#) · [API-001 v1.3](#) · [FORMAT-001 v1.3](#) · [ENG-001 v1.2](#) · [ADR-005 v6](#) · [ADR-006 v5](#) · [ADR-009 v2](#) · [ADR-010 v3](#) · [ADR-012 v6](#) · [ADR-026 v5](#) · [ADR-029 v2](#) · [ADR-034 v2.2](#) · [ADR-035 v2.4](#) · [ADR-036 v1](#)

---

## Содержание

1. [Обзор](#1-обзор)
2. [Позиционирование](#2-позиционирование)
3. [Принципы](#3-принципы)
4. [Архитектура](#4-архитектура)
5. [Компоненты](#5-компоненты)
6. [Дорожная карта](#6-дорожная-карта)
7. [Бенчмарки](#7-бенчмарки)
8. [Риски](#8-риски)
9. [Decision Log](#9-decision-log)
10. [Приложения](#10-приложения)
11. [История изменений](#11-история-изменений)

---

## 1. Обзор

### 1.1. Что такое TephraKV

Дисковая KV СУБД на Go. Один движок, три режима развёртывания.

| Режим | Клиенты | Версия |
|---|---|---|
| **Embedded** | Go (in-process) | v0.1 |
| **Server** | Go, C++, Rust, Java, Python, Node.js, Ruby, PHP, C#, Kotlin, Swift, Dart, Objective-C (gRPC) | v0.6a |
| **Distributed** | Go (client), gRPC (server) | v0.6a |

### 1.2. Целевой профиль

**Single-node (embedded / server):**

| Параметр | Значение |
|---|---|
| Ops/s на узел | 1–10M |
| p999 T1 (L1/L2, ≤ 100K keys) | < 200 нс |
| p999 T2 (L3, ≤ 1M keys) | < 500 нс |
| p999 T3 (RAM, ≤ 100M keys) | < 2 мкс |
| p999 T4 (NVMe, cold) | < 50 мкс |
| Аллокаций на hot path | 0 |
| Durability | Per-op |

**Distributed (v0.6a):**

| Параметр | Значение |
|---|---|
| p99 GET | < 10 мс |
| p999 GET | < 50 мс |
| Freshness p99 (HTAP) | < 1 с |
| Dragonboat Multi-Raft | ≥ 1M ops/s на узел |
| `AllocsPerDistributedWrite` | ≤ 10 |

### 1.3. Сравнение

**Single-node, cache-resident:**

| | Badger | Pebble | RocksDB | Redis | TephraKV |
|---|---|---|---|---|---|
| Язык | Go | Go | C++ | C | **Go** |
| Режимы | Embedded | Embedded | Embedded (cgo) | Server | **Embedded + Server + Distributed** |
| Zero-alloc hot path | Нет | Нет | Нет | Нет | **Да** |
| Read amp p99 (v0.3+) | 3 | 3 | 5 | n/a | **< 2** |
| Durability per op | Нет | Нет | Частично | Нет | **Да** |
| p999 T2 (L3, 1M) | ~50–100 мкс | ~30–80 мкс | ~20–50 мкс | ~1 мс | **< 500 нс** |
| p999 T3 (RAM, 10M) | ~50–100 мкс | ~30–80 мкс | ~20–50 мкс | ~1 мс | **< 2 мкс** |
| Pure Go | Да | Да | Нет | Нет | **Да** |

**Distributed:**

| | TiKV | CockroachDB | TephraKV |
|---|---|---|---|
| Язык | Rust | Go | **Go** |
| Режимы | Distributed | Distributed | **Embedded + Server + Distributed** |
| Multi-Raft | Да | Да | **Да (Dragonboat)** |
| Без etcd | Да (PD) | Да | **Да** |
| p999 GET | ~10–50 мс | ~10–50 мс | < 50 мс |
| Per-op durability | Нет | Нет | **Да** |
| Raft core | Свой | Свой | **Dragonboat** |

### 1.4. Сроки и объём

- v0.1: 6 недель (base) / 8 (risk-adjusted), 98 SP.
- v1.0: 84 недели (base) / 100 (risk-adjusted), 448 SP.

---

## 2. Позиционирование

### 2.1. Формулировка

**Disk-based KV store with three deployment modes. Deterministic tail latency. Zero allocations on hot path (embedded). Per-operation durability. Dragonboat Multi-Raft (distributed). p999 GET < 500 нс (T2: L3-resident, 1M keys).**

### 2.2. Три режима

**Embedded (v0.1).** In-process библиотека. Go API. Zero-alloc. Один бинарник приложения.

**Server (v0.6a).** Standalone процесс. gRPC + REST. CodecV2 + SharedBufferPool. TLS 1.3. 13+ языков клиентов.

**Distributed (v0.6a).** Dragonboat Multi-Raft. 3+ nodes. Embedded distributed без etcd. Встроенный TCP для Raft.

Переключение: `Options{Mode: Embedded | Server | Distributed}`.

### 2.3. Tiered latency

| Tier | Условие | Cache level | p50 | p99 | p999 |
|---|---|---|---|---|---|
| T1 | ≤ 100K keys | L1/L2 | < 50 нс | < 100 нс | **< 200 нс** |
| T2 | ≤ 1M keys | L3 | < 100 нс | < 300 нс | **< 500 нс** |
| T3 | ≤ 100M keys | RAM | < 300 нс | < 1 мкс | **< 2 мкс** |
| T4 | Cold SSTable | NVMe | < 5 мкс | < 20 мкс | **< 50 мкс** |

Главная differentiation-цель: **T2 (L3-resident, 1M keys, p999 < 500 нс)**.

### 2.4. Сегменты

| Сегмент | Бюджет/год | Режим | Триггер |
|---|---|---|---|
| HFT / trading | $50–200K | Embedded | Инцидент с p999 |
| Game servers | $10–50K | Embedded | Жалобы игроков |
| AI-infra | $20–100K | Embedded | Рост latency |
| Edge / IoT | $5–20K | Embedded | Рост объёма данных |
| CDN / edge compute | $20–100K | Embedded + Server | Проблемы с деплоем |
| Fintech / audit | $100–500K | Embedded + Server | Аудит |
| Distributed HTAP | $50–200K | Distributed | TiKV/CockroachDB требуют etcd |

### 2.5. Win/Loss

**Выигрываем:**

- Go, без cgo.
- GC-паузы.
- Per-op durability.
- Tiered p999.
- Embedded distributed без etcd.
- C++/Rust/Java/Python — через gRPC server mode.
- Server mode с 13+ языками.

**Проигрываем:**

- Только managed, без self-hosted.
- Enterprise ждёт SOC 2 Type II до v1.0.
- Полноценный SQL с ACID до v0.6b.
- Замену Redis для in-memory-only.

**Снятые проигрыши (были в v11.2):**

| Пункт | Почему снято |
|---|---|
| C++/Rust — RocksDB | gRPC server mode, 13+ языков |
| SQLite — малый бюджет | SQLite урезан (один writer, ограниченный функционал) |
| Distributed-only — TiKV/CockroachDB | TephraKV — дисковая СУБД, embedded — режим |
| SQL — PostgreSQL | SQL subset (v0.6b) — часть продукта |
| Badger — open-source | Community Apache 2.0 |

### 2.6. Не-цели

| Не-цель | До версии |
|---|---|
| OLAP (кроме columnar replica) | v0.6b |
| > 100M keys на single-node | v0.3 |
| Cross-key транзакции | v0.4 |
| Enterprise SLA 99.99% | v1.0 |
| Windows / macOS production | v0.3 |
| SQL / PostgreSQL wire | v0.6b |
| Векторный поиск | v0.7+ |
| Joint consensus | v1.0 |
| Multi-region | v1.0 |
| Server mode под zero-alloc контрактом | никогда |
| Собственный QUIC для intra-cluster | v1.0+ |
| `etcd/raft`, свой Multi-Raft, свой gRPC transport для Raft | отменено (D78, D106) |
| p999 без tier | никогда |
| $15B valuation | никогда |
| Замена PostgreSQL | никогда |

---

## 3. Принципы

1. Один бинарник, ноль runtime-зависимостей.
2. Zero-alloc как контракт на data plane.
3. Offsets вместо указателей.
4. Epoch before free.
5. Двухуровневые честные метрики.
6. Explicit non-goals per version.
7. Решения — в ADR.
8. Keys и values — bytes.
9. Детерминированный recovery.
10. Сообщество как часть дизайна.
11. Детерминированность важнее средней скорости.
12. Read amp p99 < 3 (v0.1–v0.2), < 2 (v0.3+).
13. Durability policy per operation.
14. TCO на единицу полезной работы.
15. Multi-Raft; транспорт — встроенный TCP Dragonboat (v0.6a).
16. ReadIndex (default) + Lease Read (опционально).
17. Static membership до v1.0, learner-based rolling с v0.6a.
18. Incremental snapshot с resume.
19. Явный backpressure.
20. Метрики транспорта с первого дня.
21. CI-гейты как единственный архитектурный критик.
22. ADR до кода.
23. Метрики публикуются попарно.
24. Honest failure: неизвестное — как неизвестное.
25. Shared WAL pool: N=16–64 (single-node).
26. Dragonboat — control plane, аллокации допустимы.
27. CodecV2 + SharedBufferPool, не protobuf.
28. Zero-alloc гарантируется CI-бенчмарком.
29. Одна БД, два профиля (Latency / Compliance).
30. Dragonboat для v0.6a.
31. Arena (mmap-backed), не sync.Pool.
32. Titan-style WriteCallback.
33. Monotonic raw clock для Lease Read.
34. API не должен врать.
35. Безопасный default: SYNC_MASTER.
36. `Get` возвращает `(value, release)`.
37. Dragonboat Raft core — control plane, TephraKV codec и LSM apply — hot path.
38. WAL segment конфигурируемый (256 МБ – 1 ГБ).
39. Tiered bloom + configurable bits/key.
40. Tan engine — Raft log storage в distributed.
41. Per-op durability — через кастомный LogDB.
42. Встроенный TCP Dragonboat для Raft. gRPC — server mode.
43. Zero-alloc в distributed — принимаем честно (`AllocsPerDistributedWrite`).
44. Цели — в tiered-терминах. p999 без tier — запрещено.
45. T2 (L3) — главная differentiation-цель.
46. **Один движок, три режима.**
47. **Embedded — режим, не ограничение.**
48. **gRPC server mode — 13+ языков.**
49. **Community — Apache 2.0. Enterprise — только для очень больших клиентов, нескоро.**

---

## 4. Архитектура

### 4.1. Три режима

```
┌──────────────────────────────────────────────────────┐
│ Режим 1: Embedded (v0.1)                             │
│ In-process · single-node · Go API · zero-alloc       │
├──────────────────────────────────────────────────────┤
│ Режим 2: Server (v0.6a)                              │
│ gRPC + REST · CodecV2 · TLS 1.3 · 13+ языков         │
├──────────────────────────────────────────────────────┤
│ Режим 3: Distributed (v0.6a)                         │
│ Dragonboat Multi-Raft · встроенный TCP · без etcd    │
└──────────────────────────────────────────────────────┘
```

### 4.2. Слои (single-node, embedded)

```
API (Put / Get / Delete / Scan / WriteBatch)
  ↓
Two-Level Metrics + Tier distribution
  ↓
WAL pool (shared, N=16–64, group commit, CRC32C, LSN)
  ↓
MemTable (lock-free skiplist + arena, offsets)
  ↓
LSM (L0 tiered, L1+ leveled, partition index, MinMax)
  ↓
VLog (WiscKey, v0.3+)
  ↓
Compaction worker (SLA-driven, tier-aware)
  ↓
Manifest (append-only + snapshot)
  ↓
Epoch reclamation
  ↓
mmap (Linux x86_64 + ARM64)
```

WAL pool — только single-node. В distributed — Tan engine (Dragonboat LogDB).

### 4.3. Distributed слои (v0.6a+)

```
Cluster Coordinator (static membership до v1.0)
  ↓
Dragonboat NodeHost (10000 Raft groups)
  - Heartbeat batching O(nodes)
  - Quiescing idle groups
  - ReadIndex protocol
  ↓
Raft Core (Dragonboat, control plane — аллокации допустимы)
  - Leader election, log replication
  - ReadIndex (default) + Lease Read (опц.)
  - Incremental snapshot
  - Learner-based rolling
  ↓
Custom LogDB (ILogDB) — per-op durability
  - NO_SYNC / SYNC_MASTER / SYNC_LEADER / SYNC_MAJORITY / SYNC_ALL
  ↓
Tan engine — Raft log storage
  ↓
Raft Transport — встроенный TCP Dragonboat
  ↓
Server Mode — gRPC + REST, CodecV2, TLS 1.3
```

Отображение: **1 shard = 1 Raft group = 1 Tan db namespace**.

### 4.4. Границы модулей

```
internal/
  arena/        bump allocator (mmap), free-list, epoch guard
  memtable/     lock-free skiplist
  wal/          shared pool (single-node only)
  sstable/      block format, bloom, partition index, MinMax
  lsm/          levels, compaction, read path
  vlog/         WiscKey, GC (v0.3+)
  mvcc/         timestamps, snapshots, version GC (v0.4+)
  txn/          begin/commit/rollback (v0.4+)
  shard/        shard-per-core (v0.4+)
  raft/         Dragonboat wrapper, custom LogDB (v0.6a)
  transport/    встроенный TCP (v0.6a), gRPC server mode
  codec/        CodecV2 + SharedBufferPool
  metrics/      two-level + tier distribution
  manifest/     append-only + snapshot
  profile/      Latency / Compliance
  audit/        append-only log (v0.5+)
  crypto/       encryption at rest (v0.6b+)
  pitr/         point-in-time recovery (v0.5+)
  ttl/          expiration (v0.5+)
```

Направление зависимостей:

```
api → txn → mvcc → lsm → sstable → arena
                       ↓
                      vlog → wal → manifest

raft → dragonboat → tan
raft → custom LogDB → tan
transport → raft
metrics → всё (read-only)
```

Проверяется `go-arch-lint` в CI.

### 4.5. Транспорт и консенсус

| Версия | Raft transport | Server mode | Клиенты |
|---|---|---|---|
| v0.6a | Встроенный TCP Dragonboat | gRPC + REST | 13+ языков |
| v0.6b | Встроенный TCP Dragonboat | gRPC + REST + SQL wire | + PostgreSQL-совместимые |
| v0.7 | Встроенный TCP + DRPC (опц.) | gRPC / DRPC | 13+ языков |
| v1.0 | Свой TCP (если DRPC bottleneck) | gRPC / DRPC / свой TCP | 13+ языков |

**Raft core:** Dragonboat (D78). **Raft log storage:** Tan engine (D101). **Per-op durability:** кастомный LogDB через `ILogDB` (D102).

**Почему не etcd/raft (D78):** 5–7 allocs/op, ~44K writes/s, нет Multi-Raft, нет per-op durability. **Почему не свой Raft (D79):** 12–18 мес разработки. **Почему не gRPC для Raft (D106):** теряются оптимизации Dragonboat. **Почему не QUIC (D51):** 66% throughput, 12× RAM, UDP блокируется.

---

## 5. Компоненты

### 5.1. Публичный API

**v0.1 (embedded):**

```go
type Mode int
const (
    ModeEmbedded    Mode = iota // v0.1
    ModeServer                  // v0.6a
    ModeDistributed             // v0.6a
)

type DurabilityPolicy int
const (
    SYNC_MASTER   DurabilityPolicy = iota // 0 — безопасный default
    NO_SYNC                                // 1
    // SYNC_LEADER, SYNC_MAJORITY, SYNC_ALL — v0.6a+
)

type Options struct {
    Mode              Mode
    Profile           Profile
    MemTableSize      int64
    WALSegmentSize    int64 // 256 МБ – 1 ГБ
    WALPoolSize       int   // 16–64
    BloomBitsPerKey   int
    DefaultDurability DurabilityPolicy
    // ...
}

func Open(path string, opts Options) (*DB, error)
func (db *DB) Close() error

func (db *DB) Put(key, value []byte, opts ...WriteOption) error
func (db *DB) Get(key []byte, opts ...ReadOption) (value []byte, release func(), err error)
func (db *DB) Delete(key []byte, opts ...WriteOption) error
func (db *DB) Scan(start, end []byte, fn func(k, v []byte) error) error
func (db *DB) WriteBatch(batch *Batch, opts ...WriteOption) error

func (db *DB) Metrics() (ExternalMetrics, InternalMetrics)
```

**v0.6a (server):** gRPC proto в `docs/api/tephrakv.proto`. REST через grpc-gateway.

**v0.6a (distributed):** `Options{Mode: ModeDistributed, Nodes, RaftID}`.

### 5.2. Компоненты

- **Arena** — mmap-backed bump allocator, epoch reclamation. Alloc 64 Б ~2.9 нс.
- **MemTable** — lock-free skiplist на offsets.
- **WAL pool** — shared, N=16–64, group commit, per-shard LSN namespace. Single-node only.
- **SSTable** — block format, CRC32C, bloom 9.6 бит/ключ (tiered с v0.3).
- **LSM** — L0 tiered, L1+ leveled, partition index, MinMax.
- **Compaction** — SLA-driven, tier-aware, гистерезис.
- **VLog** — WiscKey, threshold 256 Б, GC с Titan-style WriteCallback (v0.3).
- **MVCC** — single-version MemTable, версии в SST/VLog (v0.4).
- **Shard-per-core** — execution unit (v0.4).
- **Dragonboat** — Raft core (v0.6a).
- **Tan engine** — Raft log storage (v0.6a).
- **Custom LogDB** — per-op durability через `ILogDB` (v0.6a).
- **Transport** — встроенный TCP Dragonboat (Raft), gRPC (server mode).
- **CodecV2** — zero-alloc codec (v0.6a).
- **Metrics** — two-level + tier distribution.

### 5.3. Durability policy

| Политика | fsync | Кворум | Потеря при сбое | Версия |
|---|---|---|---|---|
| SYNC_MASTER | Локально | Нет | Нет (single-node) | v0.1 |
| NO_SYNC | Нет | Нет | Последние N мс | v0.1 |
| SYNC_LEADER | Локально на лидере | Нет | Да, при смене лидера | v0.6a |
| SYNC_MAJORITY | На кворуме | Да | Нет | v0.6a |
| SYNC_ALL | На всех | Да | Нет | v0.6a |

Mixed batch запрещён (D61). SYNC_MASTER в distributed — local-only (D54).

---

## 6. Дорожная карта

| Версия | Фокус | Режим | SP | Ключевая метрика |
|---|---|---|---|---|
| v0.1 | Deterministic Foundation | Embedded | 98 | T2 p999 < 500 нс |
| v0.2 | Read-amp Compaction | Embedded | 47 | Read amp p99 < 3 |
| v0.3 | Value Log | Embedded | 25 | WA < 3, read amp < 2 |
| v0.4 | MVCC + Shard | Embedded | 40 | 50M GET/s |
| v0.5 | TTL + PITR | Embedded | 26 | Первый платящий |
| **v0.6a** | **Distributed KV Core** | **+Server +Distributed** | **78** | **Freshness < 1 с** |
| **v0.6b** | **SQL + Columnar Replica** | **+SQL wire** | **28** | **TPC-C** |
| v0.7 | Columnar | Distributed | 30 | SIMD ≥ 4× |
| v1.0 | Enterprise | Все | 76 | SOC 2, $1M ARR |

**Итого:** 448 SP. Base: 84 недели. Risk-adjusted: 100 недель.

Детализация — в [ROADMAP-001 v4.1](#) и [BACKLOG-001 v6.1](#).

---

## 7. Бенчмарки

| # | Название | Версия |
|---|---|---|
| BENCH-001 | Methodology + TCO | v0.1 |
| BENCH-002 | Zero-Alloc Gate | v0.1 |
| BENCH-003 | Cycles and Ticks | v0.1 |
| BENCH-004 | YCSB | v0.1 |
| BENCH-005 | Compaction | v0.2 |
| BENCH-006 | VLog | v0.3 |
| BENCH-007 | Crash-Recovery | v0.1 |
| BENCH-008 | Read Amplification | v0.2 |
| BENCH-009 | Durability Policy | v0.1 |
| BENCH-010 | TCO per Unit | v0.5 |
| BENCH-011 | H-Score (HTAP) | v0.6a |
| BENCH-012 | Server Mode Overhead | v0.6a |
| BENCH-013 | Raft Under Partition | v0.6a |
| BENCH-014 | Dragonboat Multi-Raft Throughput | v0.6a |
| BENCH-015 | Lease Read vs ReadIndex | v0.6a/v0.7 |
| BENCH-016 | Tiered Latency | v0.1 |

Публикация — в `docs/bench/releases/vX.Y.md`.

**Обязательные метрики:** p50/p99/p999 по tier; throughput; cost per million ops; cycles/op; allocs/op; read/write amp; tier distribution; `AllocsPerDistributedWrite` (v0.6a+).

---

## 8. Риски

### 8.1. Технические

| Риск | P | I | Митигация |
|---|---|---|---|
| T2 (L3, p999 < 500 нс) не достигнут | С | В | BENCH-016 в v0.1. Prefetching, tier-aware compaction |
| T3 (RAM, p999 < 2 мкс) не достигнут | С | С | BENCH-016. Skiplist tuning |
| Dragonboat: Tan engine интеграция | Н | С | Spike в начале v0.6a |
| Dragonboat: кастомный LogDB (ILogDB) | С | В | Абстракция, fallback на SyncPropose |
| Dragonboat: throughput на 10000 groups | С | С | BENCH-014 |
| `AllocsPerDistributedWrite` > 10 | С | С | BENCH-014 |
| Lock-free баги в skiplist | С | В | Fuzz 1M ops |
| Read amp > 3 (v0.1–v0.2) | С | С | SLA-driven compaction |
| Clock skew | С | С | ReadIndex (default) |
| WiscKey update-heavy degradation | С | С | Явные ограничения |

### 8.2. Продуктовые

| Риск | P | I | Митигация |
|---|---|---|---|
| Позиционирование не резонирует | С | В | Публикация tiered p999 с первого дня |
| Server mode (v0.6a) не привлечёт C++/Rust | С | С | gRPC — стандарт, 13+ языков |
| SQLite-клиенты не перейдут | С | С | Функционал (durability, tiered, distributed) |
| TiKV/CockroachDB-клиенты не перейдут | С | С | Embedded + per-op durability |
| Enterprise-лицензии (нескоро) не привлекут крупных | С | В | Community — Apache 2.0 |
| Solo → выгорание | В | В | 3-недельные релизы |
| Нет revenue до v0.5 | В | В | Design partners |
| TAM мал | В | В | Три режима расширяют TAM |

### 8.3. Конкурентные

| Риск | P | I | Митигация |
|---|---|---|---|
| Badger добавит zero-alloc | Н | В | Скорость итераций |
| Pebble догонит по read amp | С | С | Дифференциация через durability |
| Redis добавит persistence | С | С | Per-op durability |
| TiDB/CockroachDB в distributed HTAP | В | С | Embedded + tiered p999 |
| Dragonboat догонит по embedded | Н | С | KV-специфика |

### 8.4. Неизвестное

- T2/T3 достижимость — BENCH-016.
- Tier distribution в реальной нагрузке — не знаем.
- Dragonboat throughput на 10000 groups — BENCH-014.
- Dragonboat heartbeat batching на 10000 groups — не знаем.
- Dragonboat quiescing: сколько групп активно — не знаем.
- Tan engine интеграция — не знаем.
- Кастомный LogDB overhead — не знаем.
- Встроенный TCP throughput — не знаем.
- `AllocsPerDistributedWrite` — BENCH-014.
- Worst-case burst 10× — измеряем.
- Clock skew > timeout — тестируем.
- Snapshot transfer больших данных — не знаем.
- Columnar apply 10M ops/s — не знаем.
- TLS AEAD overhead при 1M ops/s — не знаем.
- Lease Read при bounded drift — не знаем.
- CodecV2 миграция — не знаем.
- Path to revenue — не знаем.
- Moat против Google — не знаем.

### 8.5. Failure model

| Сбой | Гарантия v0.1 | Гарантия v0.6a |
|---|---|---|
| Process crash (kill -9) | SYNC_MASTER: 0 потерь; NO_SYNC: N мс | + SYNC_MAJORITY/ALL: 0 потерь |
| OOM kill | То же | То же |
| Disk full | Write error, backpressure | То же |
| Partial write | WAL CRC32C + LSN | Tan engine recovery |
| Bit rot | CRC per block; v0.1 не восстанавливает | v1.0: replication + scrub |
| FS corruption | Manifest + WAL + SSTable | Tan engine + Raft log |
| WAL pool exhaustion | Backpressure | Tan engine + backpressure |
| Clock skew | n/a | ReadIndex: не влияет |
| Network partition | n/a | SYNC_MAJORITY: 0 потерь |
| Leader crash | n/a | SYNC_MAJORITY: 0 потерь |

### 8.6. Threat model

**Storage (v0.1):** corruption (CRC32C), partial write (CRC + LSN), malicious key/value (bounds check), OOM (лимит value), disk full (backpressure), FS corruption (recovery).

**Network (v0.6a):** TLS downgrade (TLS 1.3 only), replay (Raft term/index + TLS session), malicious peer (mTLS, RBAC v1.0), DoS (rate limiting), info leak (метрики без keys).

**Вне scope v0.1:** encryption at rest, RBAC, audit log, multi-tenant — v1.0.

---

## 9. Decision Log

Полный список D1–D122 — в [ADR-реестре](docs/adr/README.md). Ключевые:

| # | Решение |
|---|---|
| D2 | Позиционирование: tiered tail latency |
| D5 | Политики v0.1: SYNC_MASTER (default), NO_SYNC |
| D8 | Read amp: < 3 (v0.1–v0.2), < 2 (v0.3+) |
| D32 | Консенсус: Dragonboat Multi-Raft |
| D53 | WAL: shared pool + per-shard LSN. Single-node only |
| D78 | Raft core v0.6a: Dragonboat |
| D79 | Raft core v1.0: Dragonboat + оптимизации |
| D80 | Zero-alloc: Dragonboat — control plane; TephraKV codec и LSM apply — hot path |
| D85 | Arena (mmap), не sync.Pool |
| D92 | Безопасный default: `SYNC_MASTER = iota` (0) |
| D93 | `Get` возвращает `(value, release)` |
| D101 | Tan engine — Raft log storage в distributed |
| D102 | Кастомный LogDB — per-op durability через `ILogDB` |
| D103 | Встроенный TCP Dragonboat — Raft transport |
| D104 | Zero-alloc в distributed — принимаем честно (`AllocsPerDistributedWrite`) |
| D105 | v0.6 → v0.6a + v0.6b |
| D106 | Свой gRPC transport для Raft — отменено |
| D107 | p999 — tiered (T1/T2/T3/T4) |
| D108 | T2 (L3, 1M keys, p999 < 500 нс) — главная differentiation-цель |
| D110 | `TierDistribution` в метриках |
| D111 | BENCH-016 |
| D113 | p999 без tier — запрещено |
| D115 | Тип: дисковая KV СУБД с тремя режимами |
| D116 | Три режима: Embedded (v0.1), Server (v0.6a), Distributed (v0.6a) |
| D117 | gRPC server mode — first-class, 13+ языков |
| D118 | SQLite — не конкурент (урезан) |
| D119 | Enterprise-лицензии — только для очень больших, нескоро |
| D120 | Managed — отдельный рынок, не наш фокус |
| D121 | Win/Loss переработан |
| D122 | ICP: 7 сегментов по режимам |

---

## 10. Приложения

### 10.1. Разрешённые утверждения

- «Disk-based KV store with three deployment modes: embedded, server, distributed.»
- «Embedded — режим, не ограничение.»
- «Deterministic tail latency.»
- «p999 GET < 200 нс (T1: L1/L2 hot, 100K keys, single-node).»
- «p999 GET < 500 нс (T2: L3-resident, 1M keys, single-node).»
- «p999 GET < 2 мкс (T3: RAM-resident, 10M keys, single-node).»
- «p999 GET < 50 мкс (T4: NVMe-backed, 100M keys, single-node).»
- «Tier distribution ≥ 80% в L1/L2/L3 для 1M keys.»
- «Read amp p99 < 3 (v0.1–v0.2); < 2 (v0.3+).»
- «Per-operation durability policy.»
- «Zero allocations on hot path (data plane).»
- «Dragonboat Multi-Raft, no etcd dependency (v0.6a+).»
- «Dragonboat: 1.25M writes/s на Raft group, 9M writes/s на 22 ядрах.»
- «Tan engine: log-file based LogDB, без space amplification.»
- «Кастомный LogDB для per-operation durability.»
- «Встроенный TCP Dragonboat для Raft.»
- «gRPC server mode: 13+ языков клиентов.»
- «Community — Apache 2.0.»
- «`AllocsPerDistributedWrite` — честная метрика distributed пути.»

### 10.2. Запрещённые утверждения

- «Лучшая СУБД».
- «Быстрее X» без раскрытия профиля.
- «Ноль аллокаций» без «на горячем пути».
- «ACID» до v0.6a.
- «HTAP» до v0.6a.
- «Freshness < 1 с» до v0.6a.
- «Read amp < 2» без указания версии.
- «Distributed» до v0.6a.
- «p999 < 5 мс (in-memory)» — отменено, tiered.
- «in-memory» без tier.
- «p999 < 100 нс» без tier.
- «p999 < 500 нс» без «T2: L3-resident, 1M keys».
- «TephraKV — только embedded KV».
- «TephraKV проигрывает C++/Rust».
- «TephraKV проигрывает SQLite».
- «TephraKV не конкурент TiKV/CockroachDB».
- «etcd/raft для v0.6a».
- «Свой Raft для v0.6a».
- «Свой gRPC transport для Raft».
- «Zero-alloc на Raft path».
- «WAL per shard».
- «$15B valuation».
- «Собственный QUIC» до v1.0+.
- «`ProfileCustom`» — два профиля.

---

## 11. История изменений

### v11.3 — 23 сентября 2026

- TephraKV — дисковая KV СУБД, три режима (embedded, server, distributed).
- Сняты устаревшие «проигрыши» v11.2: C++/Rust, SQLite, TiKV/CockroachDB.
- gRPC server mode — first-class, 13+ языков.
- Community — Apache 2.0. Enterprise — только для очень больших, нескоро.
- Принципы 46–49. D115–D122.

### v11.2 — 22 сентября 2026

- Tiered p999 (T1–T4). T2 — главная differentiation-цель.
- Dragonboat вместо etcd/raft.
- Встроенный TCP Dragonboat для Raft. gRPC — server mode.
- v0.6 → v0.6a + v0.6b.
- SP: 445 → 448. D107–D114.

### v11.1 — 22 сентября 2026

- Tan engine как единственный Raft log storage.
- Кастомный LogDB (`ILogDB`) для per-op durability.
- Встроенный TCP Dragonboat для Raft.
- Zero-alloc в distributed — принимаем честно.
- v0.6 разбит. D101–D106.

### v11.0 — 22 сентября 2026

- Dragonboat вместо собственного Multi-Raft core. D78–D100.

### v10.x и ранее

- Собственный Multi-Raft. QUIC для intra-cluster. p999 < 5 мс (in-memory).
- Отменено в v11.0+ (D78, D51, D107).

---

**Конец документа TephraKV-HLD-000 v11.3**