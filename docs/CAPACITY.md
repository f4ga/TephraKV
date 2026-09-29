# TephraKV — ADR-035: Capacity Model

**Документ:** TephraKV-ADR-035
**Версия:** 2.3
**Статус:** Proposed
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v11.2, TephraKV-PRD-001 v1.3, TephraKV-GTM-001 v2.3, TephraKV-PRICING-001 v1.3, TephraKV-ROADMAP-001 v4.0, TephraKV-BACKLOG-001 v6.0, TephraKV-GLOSSARY-001 v3.4, ADR-005 v5, ADR-006 v5, ADR-010 v3, ADR-012 v6, ADR-026 v5, ADR-036 v1
**Заменяет:** TephraKV-ADR-035 v2.0
**Тип:** Architecture Decision Record

---

## 1. Контекст

TephraKV позиционируется как embedded KV для Go-разработчиков в latency-critical приложениях: HFT, game servers, AI-infra, edge, CDN, fintech/audit (HLD v11.2 §3.1). Целевой профиль (HLD v11.2 §2.3):

- 1–10M ops/s на узел (single-node).
- **Tiered p999 GET:** T1 (L1/L2) < 200 нс, T2 (L3) < 500 нс, T3 (RAM) < 2 мкс, T4 (NVMe) < 50 мкс.
- Zero-alloc hot path (single-node + LSM apply).
- Read amp p99 < 3 (v0.1–v0.2) / < 2 (v0.3+).
- Per-operation durability (кастомный LogDB в v0.6a+).
- Distributed на **Dragonboat Multi-Raft** (v0.6a) поверх встроенного TCP.

Это накладывает **жёсткие требования к ресурсам**:

- Работать на 1–4 vCPU, 8–16 GB RAM (embedded, edge).
- Уложиться в Redis × 1.5 по RAM при durability (иначе теряем нишу).
- Давать durability, которой нет у Redis (иначе неотличимы от кэша).
- Укладываться в SSD/eMMC-бюджеты edge-устройств.
- Обслуживать две ниши (latency + compliance) одной БД через профили.
- **Обеспечить T2 (L3-resident, p999 < 500 нс)** — главную differentiation-цель (D108).

Без формальной capacity model невозможно:

1. Отвечать design partners на «сколько RAM/CPU/Disk на 1M ключей» и «какой tier достижим».
2. Планировать BENCH-010 (TCO per unit) и BENCH-016 (Tiered Latency).
3. Оценивать differentiation: где экономим, где осознанно тратим.
4. Ловить регрессии при добавлении фич (Bloom, partition index, VLog, MVCC, columnar, CodecV2, Dragonboat, audit).
5. Обосновывать выбор решений из HLD v11.2 (arena vs sync.Pool, CodecV2 vs V1, shared WAL vs WAL per shard, Dragonboat vs свой Raft, Tan engine vs RocksDB LogDB).

**Решение:** ADR-035 v2.3 — формальная модель ресурсов с явным разделением на business requirements, industry standards, differentiation budget, economy targets и tiered capacity analysis. Синхронизирована с HLD v11.2, ROADMAP v4.0, BACKLOG v6.0.

---

## 2. Business Requirements

### 2.1. Профили развёртывания (по сегментам)

| Сегмент | vCPU | RAM | Disk | Целевой tier | Profile |
|---|---|---|---|---|---|
| **Edge / IoT** | 1–2 | 2–4 GB | 16–64 GB eMMC | T3 (RAM) | Latency |
| **Game servers** | 2–4 | 8–16 GB | 100 GB–500 GB NVMe | T2/T3 | Latency |
| **Embedded app** | 2–4 | 4–16 GB | 100 GB–1 TB NVMe | T2/T3 | Latency |
| **AI-infra (feature store)** | 4–8 | 16–64 GB | 500 GB–2 TB NVMe | T2/T3 | Latency |
| **HFT / trading** | 4–16 | 32–128 GB | 1–4 TB NVMe | T1/T2 | Latency |
| **CDN / edge compute** | 4–8 | 16–32 GB | 500 GB–1 TB NVMe | T2/T3 | Latency |
| **Fintech / audit** | 4–8 | 16–64 GB | 500 GB–2 TB NVMe | T3 | Compliance |
| **Distributed node (v0.6a+)** | 8–16 | 32–128 GB | 1–4 TB NVMe | T3 | Custom |

**Tier-соответствие:** T1 (L1/L2) ≤ 100K keys; T2 (L3) ≤ 1M keys; T3 (RAM) ≤ 100M keys; T4 (NVMe) — cold.

### 2.2. Ключевые бизнес-требования

| # | Требование | Источник | Как проверяем |
|---|---|---|---|
| BR-1 | RAM Redis × 1.5 при durability | Конкурентное позиционирование | BENCH-010 (TCO) |
| BR-2 | Embedded на 2 vCPU / 4 GB RAM для edge | Edge-ниша | Benchmark on Raspberry Pi 5 / Jetson |
| BR-3 | 100M ключей на single-node без деградации | HLD v11.2 §2.3 | BENCH-004 (YCSB C, 100M keys) |
| BR-4 | 1M ops/s на 4 vCPU при T2 p999 < 500 нс | Core differentiation | BENCH-003, BENCH-016 |
| BR-5 | Disk overhead < 2× (LSM-only); < 5× (VLog) | Экономия диска | BENCH-005 (WA) |
| BR-6 | Startup time < 5 сек на 10 GB data | Edge restart | BENCH-007 (recovery) |
| BR-7 | Нет OOM при burst 10× | Надёжность | Backpressure tests |
| BR-8 | TCO 1M useful ops < $0.01 (solo, 3y) | HLD v11.2 §4.6 | BENCH-010 |
| BR-9 | Server mode: allocs/op ≤ 2 на codec (CodecV2), p999 < 100 мкс per hop | HLD v11.2 §8.3 | BENCH-012 |
| BR-10 | gRPC server mode: memory per connection < 20 МБ (100–250 streams) | D63 | BENCH-013 |
| BR-11 | Shared WAL pool: O(1) по файлам, не O(N_shards) | D53, D62 | BENCH-007 |
| BR-12 | Arena: alloc 64 Б < 3 нс (vs ~40 нс heap) | D85 | BENCH-003 |
| **BR-13** | **T2 (L3-resident, 1M keys): p999 < 500 нс** | **HLD v11.2 §2.3, D108** | **BENCH-016** |
| **BR-14** | **T3 (RAM-resident, 10M keys): p999 < 2 мкс** | **HLD v11.2 §2.3** | **BENCH-016** |
| **BR-15** | **Tier distribution ≥ 80% в L1/L2/L3 для 1M keys** | **HLD v11.2 §11** | **BENCH-016** |
| **BR-16** | **Dragonboat Multi-Raft ≥ 1M ops/s на узел** | **HLD v11.2 §6.7** | **BENCH-014** |
| **BR-17** | **`AllocsPerDistributedWrite` ≤ 10** | **HLD v11.2 §6.7, D104** | **BENCH-014** |
| **BR-18** | **Tan engine: Raft log storage без space amplification** | **D101** | **BENCH-014** |
| **BR-19** | **Кастомный LogDB: per-op durability без потери throughput** | **D102** | **BENCH-014** |
| **BR-20** | **Dragonboat heartbeat RPC/s ≤ 200 при 10000 groups** | **HLD v11.2 §6.7** | **BENCH-014** |
| **BR-21** | **Dragonboat memory ≤ 500 МБ на 10000 groups** | **HLD v11.2 §6.7** | **BENCH-014** |

### 2.3. Что НЕ требуется

- Работа на HDD (только NVMe / SSD).
- Работа на 512 MB RAM (не embedded-MCU).
- Обслуживание >1B ключей на single-node до v1.0.
- Замена Redis для in-memory-only workloads (мы embedded + durable).
- gRPC на 10000 streams/conn для server mode (используем 100–250, D63).
- Zero-alloc на server mode (ADR-030).
- **Zero-alloc на Raft path** (Dragonboat — control plane, D80, D104).
- **T1/T2 в distributed** (Dragonboat Raft RTT — миллисекунды, другая физика).

---

## 3. Industry Standards

### 3.1. RAM per key

| Система | Overhead на ключ | Источник |
|---|---|---|
| **Redis** | 50–100 Б (dictEntry + sds + robj) | antirez |
| **Badger** | ~2–4 Б (offset в LSM) + bloom | Badger docs |
| **Pebble** | ~1–2 Б + bloom | CockroachDB blog |
| **ScyllaDB** | ~1 Б + bloom 9.6 бит/ключ | ScyllaDB docs |
| **RocksDB** | ~1–2 Б + bloom (default 10 бит) | RocksDB wiki |
| **TephraKV** | ~1–2 Б + bloom 9.6 бит/ключ | ADR-003, D9 |

### 3.2. Disk overhead

| Система | WA (средняя) | Space amp | Источник |
|---|---|---|---|
| **Badger** | 2–10 | 1.2–2.0 | Badger benchmarks |
| **Pebble** | 5–15 | 1.1–1.5 | Pebble docs |
| **RocksDB** | 10–30 | 1.1–1.5 | RocksDB wiki |
| **ScyllaDB** | 1–5 | 1.1–1.3 | ScyllaDB docs |
| **TephraKV (target)** | < 5 (v0.1–v0.2), < 3 (v0.3+) | < 1.3 (VLog, после GC) | HLD v11.2 §2.3 |

### 3.3. Latency (tiered, single-node)

| Система | p99 GET | p999 GET | Источник |
|---|---|---|---|
| **Badger (in-memory)** | ~10–30 мкс | ~50–100 мкс | Badger issues |
| **Pebble (in-memory)** | ~5–15 мкс | ~30–80 мкс | Pebble benchmarks |
| **RocksDB (in-memory)** | ~5–10 мкс | ~20–50 мкс | RocksDB benchmarks |
| **Redis** | ~100–200 мкс (network) | ~1 мс (GC-подобные пики) | Redis benchmarks |
| **TephraKV T1 (L1/L2, 100K keys)** | < 100 нс | **< 200 нс** | HLD v11.2 §2.3 |
| **TephraKV T2 (L3, 1M keys)** | < 300 нс | **< 500 нс** | HLD v11.2 §2.3 |
| **TephraKV T3 (RAM, 10M keys)** | < 1 мкс | **< 2 мкс** | HLD v11.2 §2.3 |
| **TephraKV T4 (NVMe, cold)** | < 20 мкс | **< 50 мкс** | HLD v11.2 §2.3 |

**Замечание:** сравнение Badger/Pebble (disk-backed in-memory) с TephraKV (cache-resident) некорректно без указания tier. T2 — cache-resident, T4 — NVMe-backed.

### 3.4. Transport memory (per connection)

| Система | Streams/conn | Memory/conn | Источник |
|---|---|---|---|
| **gRPC default** | 100 (MAX_CONCURRENT_STREAMS) | ~6.5 МБ | HTTP/2 spec |
| **gRPC на 10000 streams** | 10000 | ~625 МБ | D63 (отвергнуто) |
| **TiKV** | 4 conn × 100–250 streams | ~26 МБ | `grpc-raft-conn-num` |
| **TephraKV (target, server mode)** | 100–250 | **< 20 МБ** | D63, BR-10 |
| **TephraKV (Raft, встроенный TCP)** | N/A (Dragonboat batch) | ~5–10 МБ | D103 |

### 3.5. Dragonboat Multi-Raft (precedents)

| Метрика | Dragonboat | etcd/raft | Источник |
|---|---|---|---|
| Writes/s на группу (16 Б) | **1.25M** | ~44K–50K | Lei Ni benchmarks |
| Writes/s на узел (22 ядра) | **9M** | ~50K | Lei Ni benchmarks |
| P99 latency при 8M writes/s | **< 5 мс** | ~20–22 мс | Lei Ni benchmarks |
| Allocs/op (RawNode) | Контролируемо | 5–7 | etcd/raft issues |
| Multi-Raft | **Да, из коробки** | Нет | — |
| WAL в комплекте | **Да (Tan engine)** | Нет | D101 |
| Jepsen/Knossos | **Пройден** | Частично | — |
| Memory на 10000 groups | ~500 МБ (цель) | — | HLD v11.2 §6.7 |

### 3.6. Startup time

| Система | Startup на 10 GB | Источник |
|---|---|---|
| **Badger** | 1–5 сек (WAL replay) | Badger issues |
| **Pebble** | 1–3 сек | CockroachDB |
| **RocksDB** | 1–5 сек | RocksDB |
| **TephraKV (target)** | < 5 сек | BR-6 |

### 3.7. Вывод по standards

| Метрика | Обязаны встретить | Где выигрываем |
|---|---|---|
| RAM per key | ≤ ScyllaDB × 1.5 | ≤ Badger, ≤ Redis ÷ 30 |
| WA (средняя) | ≤ 5 | ≤ Pebble, ≤ RocksDB |
| Space amp | ≤ 1.5 | ≈ ScyllaDB |
| p999 GET T2 (L3) | ≤ Pebble × 0.01 | **100–200× лучше** |
| p999 GET T3 (RAM) | ≤ Pebble × 0.04 | **25–50× лучше** |
| Transport memory/conn (server mode) | ≤ TiKV × 1 | ≈ TiKV |
| **Raft throughput (на группу)** | **≥ Dragonboat × 0.4** | **Dragonboat (1.25M writes/s)** |
| **Raft throughput (на узел)** | **≥ etcd/raft × 20** | **Dragonboat (9M writes/s)** |
| Startup 10 GB | ≤ 5 сек | ≈ Badger |
| Durability per op | Redis не имеет | Уникально |
| Server mode allocs/op codec | ≤ 2 (TiKV protobuf) | CodecV2: ~0 (D86) |

---

## 4. Differentiation Budget

### 4.1. Где мы тратим больше

| Решение | Дополнительный ресурс | Что получаем | Обоснование |
|---|---|---|---|
| **Partition index** (v0.2) | +16 МБ на 1B blocks | Read amp p99 < 3 (v0.2), < 2 (v0.3+) | D10, D11 |
| **MinMax block index** (v0.2) | +0.5 Б/ключ | Scan skip | D11 |
| **Bloom 9.6 бит/ключ** | +1.2 Б/ключ/уровень | FP < 1% | D9 |
| **Tiered bloom** (v0.3) | +~300–400 МБ на 100M keys (vs 840 МБ) | Экономия памяти | D97 |
| **Arena + epoch** (v0.1) | +20–30% overhead на slab | Zero-alloc hot path | D85 |
| **VLog** (v0.3) | +space amp 1.3 | WA < 3, read amp p99 < 2 | ADR-022 |
| **MVCC версии** (v0.4) | +N Б/версия до watermark | Snapshot isolation | ADR-017 |
| **Columnar replica** (v0.6b) | +20% RAM, +50% disk | HTAP | ADR-024 |
| **CodecV2 + SharedBufferPool** (v0.6a) | +pool memory (bounded, ~1–5 МБ/conn) | allocs/op ≤ 2 на codec | D86 |
| **Titan-style WriteCallback** (v0.3) | +metadata per Blob file | GC-snapshot consistency | D87 |
| **Audit log** (v0.5, Compliance) | +disk (append-only, ~10% ops) | Compliance | D73 |
| **Encryption at rest** (v0.6b, Compliance) | +10–20% CPU (AES-NI) | Compliance | D74 |
| **PITR** (v0.5, Compliance) | +WAL archive (~30 дней) | Point-in-time recovery | D75 |
| **Dragonboat** (v0.6a) | ~500 МБ на 10000 groups | Multi-Raft из коробки | D78 |
| **Tan engine** (v0.6a) | ~1–5 МБ на 100 shards | Log-file based LogDB без space amp | D101 |
| **Кастомный LogDB** (v0.6a) | +buffer + callback state | Per-op durability | D102 |
| **Встроенный TCP Dragonboat** (v0.6a) | ~5–10 МБ/conn | Ближе к raw performance | D103 |

### 4.2. Где мы тратим меньше

| Решение | Экономия | Что теряем | Обоснование |
|---|---|---|---|
| **Нет per-key index в RAM** | ~50–100 Б/ключ vs Redis | Нет O(1) lookup — O(log n) skiplist | Embedded, не cache |
| **Offsets вместо указателей** | ~8 Б → 4 Б на узел skiplist | Чуть сложнее код | D85 |
| **Single-version MemTable** | ~50% памяти MemTable | Читатель видит только последнюю версию | D72 |
| **Shared WAL pool** | O(1) по файлам vs O(N_shards) | Сложнее recovery (фильтрация) | D53, D62 |
| **N Raft-групп на 1 gRPC stream (server mode)** | ~6–16 МБ/conn vs 625 МБ | Сложнее мультиплексирование | D63 |
| **Нет собственного QUIC** | 12× RAM vs gRPC | Нет 0-RTT | D51 |
| **sync.Pool запрещён в data plane** | Нет дрейфа пула | Свой arena — код дороже | D68, D85 |
| **Dragonboat вместо своего Raft** | 12–18 мес разработки | Не zero-alloc на Raft path | D78, D79 |
| **CodecV2, не protobuf** | ~300× меньше аллокаций на marshal | Свой wire format | D70, D86 |
| **Tan engine vs RocksDB LogDB** | Нет space amplification | Экспериментальный | D101 |
| **Встроенный TCP vs свой gRPC transport** | Не теряем оптимизации Dragonboat | Меньше контроля | D103, D106 |

### 4.3. Net differentiation

| Метрика | Мы vs Badger | Мы vs Pebble | Мы vs Redis | Мы vs TiKV |
|---|---|---|---|---|
| RAM per key | ≈ (оба ~2–4 Б) | ≈ (оба ~1–2 Б) | **−30× меньше** | ≈ |
| p999 GET T2 (L3) | **100–200× лучше** | **60–160× лучше** | **×100–200 лучше** | n/a |
| p999 GET T3 (RAM) | **25–50× лучше** | **15–40× лучше** | **×10–200 лучше** | n/a |
| WA | **2–3× лучше** | **2–5× лучше** | n/a | ≈ |
| Durability per op | Уникально | Уникально | Redis не имеет | Уникально |
| Read amp p99 | **< 3 / < 2** | ~3 | n/a | ~3 |
| **Raft throughput на узел** | n/a | n/a | n/a | **20× лучше** |
| **`AllocsPerDistributedWrite`** | n/a | n/a | n/a | **≤ 10 (честно)** |
| Server mode allocs/op codec | **≤ 2 (V2)** | ~2 (protobuf) | n/a | ~2 (protobuf) |

**Вывод:** дифференциация идёт за счёт **tiered p999 + zero-alloc + read amp + per-op durability + Dragonboat Multi-Raft + CodecV2**. RAM — на уровне ScyllaDB, что даёт право называться «экономным», но не является главным аргументом.

---

## 5. Economy Targets

### 5.1. Формула (из HLD v11.2 §4.6)

```
TCO_per_1M = (Infra_3y + Ops_3y + Migration) / (Useful_ops_3y / 1M)
```

### 5.2. Целевые TCO по сегментам (v1.0, 3 года, single-node)

| Сегмент | Infra | Ops | Migration | Useful ops | TCO / 1M |
|---|---|---|---|---|---|
| **Edge (RPi 5)** | $300 | $0 | $200 | 10^11 | **$0.005** |
| **Game servers** | $1800 | $600 | $400 | 5×10^11 | **$0.006** |
| **Embedded app** | $2400 | $800 | $500 | 10^12 | **$0.004** |
| **AI-infra** | $9600 | $1200 | $1000 | 10^13 | **$0.001** |
| **HFT** | $20000 | $5000 | $2000 | 10^13 | **$0.003** |
| **CDN** | $7200 | $1000 | $800 | 5×10^12 | **$0.002** |
| **Fintech (Compliance)** | $9600 | $3000 | $2000 | 10^12 | **$0.015** |
| **Distributed (5 nodes, Dragonboat)** | $60000 | $10000 | $5000 | 10^14 | **$0.0008** |

**Цель BR-8:** < $0.01 за 1M useful ops (solo, 3y). Fintech (Compliance) — $0.015 из-за audit/encryption/PITR.

### 5.3. RAM budget (per profile)

#### Profile Latency (single-node)

| Сегмент | Ключей | Bloom | Partition index | Arena | Skiplist | Tier | Итого RAM |
|---|---|---|---|---|---|---|---|
| Edge | 1M | 38 МБ | 16 КБ | 64 МБ | 8 МБ | T3 | **~110 МБ** |
| Game servers | 10M | 480 МБ | 1.6 МБ | 256 МБ | 80 МБ | T2/T3 | **~820 МБ** |
| Embedded | 10M | 480 МБ | 1.6 МБ | 256 МБ | 80 МБ | T2/T3 | **~820 МБ** |
| AI-infra | 100M | 5.8 ГБ | 16 МБ | 2 ГБ | 800 МБ | T3 | **~8.6 ГБ** |
| HFT | 100M | 5.8 ГБ | 16 МБ | 2 ГБ | 800 МБ | T1/T2 | **~8.6 ГБ** |
| CDN | 100M | 5.8 ГБ | 16 МБ | 2 ГБ | 800 МБ | T2/T3 | **~8.6 ГБ** |

**Ключевой tier-показатель:**

- T1 (L1/L2) ≤ 100K keys → ~1 МБ hot set.
- T2 (L3) ≤ 1M keys → ~10–30 МБ working set.
- T3 (RAM) ≤ 100M keys → ГБ.

**Tiered bloom (v0.3+, D97):** L0–L1 без bloom, L2+ с bloom. Экономия 840 МБ → ~300–400 МБ на 100M keys.

#### Profile Compliance (ниша 2)

| Сегмент | Ключей | Base (Latency) | MVCC overhead | Audit | Encryption | Итого |
|---|---|---|---|---|---|---|
| Fintech | 10M | 820 МБ | +50 МБ | +100 МБ | +80 МБ | **~1.05 ГБ** |
| Fintech | 100M | 8.6 ГБ | +500 МБ | +1 ГБ | +800 МБ | **~10.9 ГБ** |

#### Distributed (v0.6a+, Dragonboat)

На узел (8 vCPU, 64 GB), 100M keys/shard × 10 shards:

| Компонент | RAM |
|---|---|
| 10 shards × 8.6 ГБ | 86 ГБ |
| Dragonboat NodeHost (10000 groups) | ~500 МБ |
| Tan engine (Raft log storage) | ~1–5 МБ на 100 shards |
| Кастомный LogDB (per-op durability) | ~50–100 МБ |
| Встроенный TCP buffers (Dragonboat) | ~5–10 МБ/conn |
| **Итого** | **~87 ГБ** |

**Проверка BR-1:** Redis на 100M ключей × 64 Б value + 32 Б key + overhead ≈ 15–20 ГБ. TephraKV ≈ 8.6 ГБ. Укладываемся в ×1.5. ✓

**Проверка BR-2:** Edge 1M ключей ≈ 110 МБ RAM. Укладываемся в 4 ГБ. ✓

**Проверка BR-10:** gRPC server mode 100–250 streams × 65 535 Б ≈ 6–16 МБ. Укладываемся в < 20 МБ. ✓

**Проверка BR-21:** Dragonboat memory на 10000 groups ≈ 500 МБ. Укладываемся. ✓

### 5.4. Disk budget (Profile Latency, 100M keys × 1 КБ value)

| Компонент | 100M keys |
|---|---|
| Raw data | 100 ГБ |
| WAL shared pool (10 сегментов × 1 ГБ) | 10 ГБ |
| WAL active + retention | 2 ГБ |
| SSTable (WA 3×, v0.3+) | 300 ГБ |
| VLog (values > 256 Б) | 80 ГБ |
| VLog space amp (1.3×) | 104 ГБ |
| Manifest, temp | 1 ГБ |
| **Итого (LSM-only)** | **~200 ГБ** |
| **Итого (VLog)** | **~500 ГБ** |

**Проверка BR-5:** LSM-only 2× ✓; VLog 5× ✓.

### 5.5. Disk budget (Profile Compliance, 100M keys × 1 КБ)

| Компонент | 100M keys |
|---|---|
| Base (Latency, VLog) | 500 ГБ |
| Audit log (append-only) | 50 ГБ |
| PITR WAL archive (30 дней) | 100 ГБ |
| Encryption overhead (+5%) | 25 ГБ |
| **Итого** | **~675 ГБ** |

### 5.6. Disk budget (Distributed, Dragonboat, 10 shards)

| Компонент | 10 shards × 100M keys |
|---|---|
| 10 shards × 500 ГБ (VLog) | 5 ТБ |
| **Tan engine (Raft log)** | ~10 ГБ (не 10 ТБ preallocation!) |
| Custom LogDB buffers | ~1 ГБ |
| Snapshot storage | ~1 ТБ |
| **Итого** | **~6 ТБ** |

**Ключевое отличие от v2.0:** Tan engine не preallocates 1 ГБ на shard (как WAL pool). **Экономия:** ~10 ТБ → ~10 ГБ. Это критично для distributed.

### 5.7. CPU budget

| Сегмент | vCPU | Целевые ops/s | Цикл/op | Tier |
|---|---|---|---|---|
| Edge | 1–2 | 100K–500K | < 1500 GET | T3 |
| Game servers | 2–4 | 1M–4M | < 1500 GET | T2/T3 |
| Embedded | 2–4 | 1M–4M | < 1500 GET | T2/T3 |
| AI-infra | 4–8 | 5M–10M | < 1500 GET | T3 |
| HFT | 4–16 | 10M–50M | < 1500 GET | T1/T2 |
| CDN | 4–8 | 5M–10M | < 1500 GET | T2/T3 |
| Fintech (Compliance) | 4–8 | 1M–2M | < 2000 GET (encryption) | T3 |
| Distributed | 8–16 | 10M–50M | < 1500 GET | T3 |

**Проверка BR-4:** 4 vCPU × 3 GHz = 12 GHz; 1M ops/s × 1500 циклов = 1.5 GHz ≈ 12.5% CPU. ✓

**Tiered CPU:**

- T2 p999 < 500 нс → p50 < 100 нс → ~300 циклов @ 3 ГГц. p999 < 1500 циклов.
- T3 p999 < 2 мкс → ~6000 циклов @ 3 ГГц.

### 5.8. Transport memory

#### Server mode (gRPC, D63)

| Параметр | Значение |
|---|---|
| Streams/conn | 100–250 |
| INITIAL_WINDOW_SIZE | 65 535 Б |
| Memory/conn (flow control) | 6–16 МБ |
| CodecV2 SharedBufferPool | ~1–5 МБ/conn |
| **Итого/conn** | **< 20 МБ** |

#### Raft transport (встроенный TCP Dragonboat, D103)

| Параметр | Значение |
|---|---|
| Batch messages | ~1 МБ/conn |
| TLS (опционально) | +5% |
| Memory/conn | **~5–10 МБ** |

**Отличие от v2.0:** Raft transport = встроенный TCP Dragonboat (не gRPC). Это **меньше памяти** и ближе к raw performance (D103, D106).

#### CodecV2 бенчмарк (4 КБ message, D86)

| Метрика | V1 bridge | V2 | Улучшение |
|---|---|---|---|
| Unmarshal ns/op | 174 | 78 | 2.4× |
| Marshal ns/op | 728 | 268 | 2.7× |
| B/op (marshal) | 486 | ~1.6 | ~300× |

### 5.9. Arena performance (D85)

| Параметр | Arena (mmap bump) | Go heap | Улучшение |
|---|---|---|---|
| Alloc 64 Б | ~2.9 нс | ~40 нс | 13.8× |
| CAS-based fast path | < 5 нс/op | ~25 нс/op | 5× |
| GC pressure | 0 (off-heap) | Есть | — |

---

## 6. Tiered Capacity Model

### 6.1. Tier definitions

| Tier | Условие | Working set | Cache level | p999 target |
|---|---|---|---|---|
| **T1** | Hot keys ≤ 100K | ≤ 1 МБ | L1/L2 (32–64 КБ / 256 КБ – 1 МБ) | < 200 нс |
| **T2** | ≤ 1M keys | ≤ 30 МБ | L3 (8–64 МБ) | < 500 нс |
| **T3** | ≤ 100M keys | ГБ | RAM | < 2 мкс |
| **T4** | Cold SSTable | ТБ | NVMe | < 50 мкс |

### 6.2. Tier-соответствие по сегментам

| Сегмент | Keys | Working set | Ожидаемый tier | Профиль |
|---|---|---|---|---|
| Edge | 1M | ~30 МБ | T3 (частично T2) | Latency |
| Game servers | 10M | ~300 МБ | T2/T3 | Latency |
| Embedded | 10M | ~300 МБ | T2/T3 | Latency |
| AI-infra | 100M | ГБ | T3 | Latency |
| HFT | 100M | ГБ | T1/T2 (hot path) | Latency |
| CDN | 100M | ГБ | T2/T3 | Latency |
| Fintech | 100M | ГБ | T3 | Compliance |
| Distributed | 100M/shard | ГБ | T3 | Custom |

### 6.3. Tiered latency targets

| Tier | p50 | p99 | p999 | Cycles p50 @ 3 ГГц | Cycles p999 @ 3 ГГц |
|---|---|---|---|---|---|
| T1 (L1/L2) | < 50 нс | < 100 нс | < 200 нс | < 150 | < 600 |
| T2 (L3) | < 100 нс | < 300 нс | < 500 нс | < 300 | < 1500 |
| T3 (RAM) | < 300 нс | < 1 мкс | < 2 мкс | < 900 | < 6000 |
| T4 (NVMe) | < 5 мкс | < 20 мкс | < 50 мкс | — | — |

### 6.4. Tier distribution target

**BR-15:** ≥ 80% операций в L1/L2/L3 для 1M keys (T2).

**Митигация деградации T2:**

- Tier-aware compaction: возврат hot keys в L3.
- Prefetching top-of-skiplist.
- L0 cache residency.

### 6.5. Distributed tier (Dragonboat)

| Параметр | Значение |
|---|---|
| Single-node T2 p999 | < 500 нс |
| Distributed p999 | < 50 мс |
| **Разрыв** | **~10 000×** |
| Причина | Network RTT + Raft round (heartbeat + log append + commit) |

**Правило:** distributed — другая физика. Не маскировать разрыв. T1/T2 только single-node.

---

## 7. Capacity Model по версиям

### 7.1. v0.1 — Deterministic Foundation

**SP:** 98. **Tiered exit:** T2 p999 < 500 нс, T3 p999 < 2 мкс, tier distribution ≥ 80%.

| Ресурс | 1M keys (T2) | 10M keys (T3) | 100M keys (T4) |
|---|---|---|---|
| RAM | 30 МБ | 250 МБ | 2.5 ГБ |
| Disk | 1 ГБ | 10 ГБ | 100 ГБ |
| CPU (ops/s на 4 vCPU) | 2M | 2M | 2M |
| Bloom | 9.6 МБ | 96 МБ | 960 МБ |
| Arena | 16 МБ | 128 МБ | 1 ГБ |
| Shared WAL pool | 10 ГБ | 10 ГБ | 10 ГБ |
| Tier distribution | ≥ 80% L1/L2/L3 | ≥ 70% L3 | ≥ 50% RAM |

**Bottleneck v0.1:** MemTable arena (single-skiplist, single-threaded CAS). При >1M ops/s → contention.

### 7.2. v0.2 — Read-amp Compaction

| Ресурс | 10M keys | 100M keys |
|---|---|---|
| RAM | 250 МБ | 2.5 ГБ |
| Partition index | +160 КБ | +1.6 МБ |
| MinMax | +5 МБ | +50 МБ |
| Compaction I/O | +30% | +30% |
| **Tier-aware compaction** | T2 p999 +10% | T2 p999 +10% |

**Bottleneck v0.2:** compaction I/O. Митигация: bandwidth-aware admission control (D90), tier-aware compaction.

### 7.3. v0.3 — Value Log

| Ресурс | 10M keys × 1 КБ | 100M keys × 1 КБ |
|---|---|---|
| RAM | 250 МБ | 2.5 ГБ |
| Disk (LSM-only) | 20 ГБ | 200 ГБ |
| Disk (VLog) | 52 ГБ | 520 ГБ |
| Titan WriteCallback metadata | +1% | +1% |
| Tiered bloom (L0–L1 без) | −40% | −55% |
| **T2 p999 для append-heavy** | +10% | +10% |

**Bottleneck v0.3:** VLog GC при fragmentation > 50%.

### 7.4. v0.4 — MVCC + Shard-per-core

| Ресурс | 10M keys | 100M keys |
|---|---|---|
| RAM (base) | 250 МБ | 2.5 ГБ |
| Version overhead | +50 МБ | +500 МБ |
| Shard overhead (64 cores) | +64 МБ | +64 МБ |
| Watermark state | +1 МБ | +10 МБ |
| T2 p999 | ≤ 500 нс | ≤ 500 нс |

**Bottleneck v0.4:** GC версий.

### 7.5. v0.5 — TTL, PITR, Hardening

| Ресурс | 10M keys | 100M keys |
|---|---|---|
| RAM | 300 МБ | 3 ГБ |
| TTL metadata | +2% | +2% |
| PITR WAL archive | +30 ГБ | +100 ГБ |
| Audit log (Compliance) | +50 МБ | +1 ГБ |
| Prometheus exporter | +20 МБ | +50 МБ |

### 7.6. v0.6a — Distributed KV Core (Dragonboat)

На узел (8 vCPU, 64 GB):

| Ресурс | 100M keys/shard × 10 shards |
|---|---|
| RAM (10 shards) | 86 ГБ |
| Disk (10 shards, VLog) | 5 ТБ |
| CPU | 50M ops/s |
| **Shared WAL pool** | **N/A (single-node only)** |
| **Tan engine** | **~10 ГБ на 10 shards (не 10 ТБ!)** |
| **Кастомный LogDB** | **~50–100 МБ** |
| **Dragonboat NodeHost (10000 groups)** | **~500 МБ** |
| **Встроенный TCP buffers** | **~5–10 МБ/conn** |
| Snapshot storage | ~1 ТБ |
| Server mode (gRPC) | +1–2 ГБ |
| CodecV2 SharedBufferPool | +100 МБ |

**Ключевые выигрыши v0.6a:**

- **Tan engine:** log-file based LogDB, без space amplification. ~10 ГБ vs 10 ТБ preallocation (если был WAL per shard).
- **Dragonboat:** 1.25M writes/s на группу, 9M writes/s на 22 ядрах. Multi-Raft из коробки.
- **Встроенный TCP:** ближе к raw performance, не теряет оптимизации Dragonboat (D103, D106).
- **`AllocsPerDistributedWrite`:** ≤ 10 (честно).

**Bottleneck v0.6a:**

- Интеграция с Tan engine.
- Кастомный LogDB через `ILogDB` — нестабильный API (D102).
- Dragonboat throughput на 10000 groups (не измерено).
- Snapshot transfer больших данных.

### 7.7. v0.6b — SQL + Columnar Replica

| Ресурс | 100M keys/shard × 10 shards |
|---|---|
| RAM (columnar replica) | +20% |
| Disk (columnar) | +50% |
| CPU (SQL subset) | +10% |
| Encryption at rest | +10–20% CPU |

### 7.8. v0.7 — Columnar

| Ресурс | 100M keys |
|---|---|
| RAM (columnar replica) | +20% |
| Disk (columnar, compression 5×) | +50% |
| SIMD CPU | +10% |

### 7.9. v1.0 — Enterprise

| Ресурс | Multi-region (3 regions) |
|---|---|
| RAM | ×1.2 |
| Disk | ×1.1 |
| CPU | ×1.3 |
| Network | ×3 |
| Свой Raft (если команда 5+) | +5% CPU vs Dragonboat |

---

## 8. Bottleneck Analysis

### 8.1. Worst-case 10× burst (BR-7)

| Компонент | Normal | Burst 10× | Bottleneck? | Митигация |
|---|---|---|---|---|
| MemTable write | 1M ops/s | 10M ops/s | Yes | Backpressure (ADR-008) |
| WAL shared pool fsync | 1M ops/s | 10M ops/s | Yes | Group commit (100 мкс) |
| Compaction | 100 МБ/с | 1 ГБ/с | Yes | SLA-driven scheduler (D12, D90) |
| GC версий | 10K версий/с | 100K версий/с | Yes | Watermark stalls |
| RAM | стабильно | растёт | Yes | Arena pool exhaustion |
| **Dragonboat Raft** | **1M ops/s** | **10M ops/s** | **Yes** | **SyncPropose + backpressure** |
| **Tan engine** | **~100 МБ/с** | **~1 ГБ/с** | **Yes** | **Batch writes** |
| Server mode gRPC | 100–250 streams | 100–250 | No | MaxConcurrentStreams (D63) |
| CodecV2 pool | ~1–5 МБ | ~50 МБ | No | SharedBufferPool bounded (D86) |

### 8.2. 10× рост данных

| Ресурс | 100M → 1B keys | Bottleneck |
|---|---|---|
| RAM | 8.6 ГБ → 86 ГБ | Yes |
| Disk | 500 ГБ → 5 ТБ | Yes |
| Compaction | 1× → 3× | Yes |
| Recovery | 5 сек → 50 сек | Yes |
| **Dragonboat groups** | **10000 → 100000** | **Yes** |

### 8.3. CPU scaling

| Cores | Expected ops/s | Actual (прогноз) | Scaling |
|---|---|---|---|
| 1 | 1M | 1M | 1.0× |
| 2 | 2M | 1.9M | 0.95× |
| 4 | 4M | 3.7M | 0.93× |
| 8 | 8M | 7.0M | 0.88× |
| 16 | 16M | 12M | 0.75× |
| 64 | 64M | 30M | 0.47× |

### 8.4. Shared WAL pool recovery time (BR-11)

| Shards | WAL files | Recovery (прогноз) |
|---|---|---|
| 1 | 1 | ~100 мс |
| 100 | 1 | ~500 мс |
| 10000 | 1 | ~5 сек |
| 10000 (parallel, 8 concurrent) | 1 | ~1 сек |

### 8.5. Dragonboat Multi-Raft capacity (BR-16..BR-21)

| Параметр | Цель | Обоснование |
|---|---|---|
| Throughput на группу | ≥ 500K writes/s | Dragonboat: 1.25M (precedent) |
| Throughput на узел | ≥ 1M ops/s | Dragonboat: 9M (precedent) |
| Heartbeat RPC/s | ≤ 200 | Heartbeat batching O(nodes) |
| Memory на 10000 groups | ≤ 500 МБ | ~50 Б/group |
| Election time p99 | < 1 сек | Randomized timeout |
| `AllocsPerDistributedWrite` | ≤ 10 | Честно, Dragonboat не zero-alloc |
| Кастомный LogDB overhead | +5–10% latency | ILogDB callback |

---

## 9. Consequences

### 9.1. Положительные

- Design partners получают конкретные числа по сегментам и tier.
- BENCH-010 (TCO) и BENCH-016 (Tiered Latency) имеют формальную базу.
- Регрессии ловятся: рост RAM/disk/tier distribution → ADR review.
- Дифференциация явная: где тратим (CodecV2, Dragonboat, Tan engine), где экономим (Redis-like RAM, shared WAL).
- Профили (Latency / Compliance) имеют разные capacity-требования.
- **Dragonboat capacity формализована:** 500 МБ на 10000 groups, Tan engine ~10 ГБ vs 10 ТБ preallocation.
- **Tiered capacity:** T1/T2/T3/T4 с конкретными working set и cache levels.

### 9.2. Отрицательные

- Модель требует поддержки: обновление при каждом ADR.
- Часть цифр — прогноз, не измерение.
- Дополнительный документ в комплекте.
- **Dragonboat capacity не измерена** — только прогноз на основе precedent.

### 9.3. Нейтральные

- Модель не заменяет benchmark'и.
- При расхождении — BENCH прав, модель правится.

---

## 10. Alternatives

| Альтернатива | Почему отклонена |
|---|---|
| Не иметь capacity model | Design partners не получат ответов |
| Только BENCH-010, BENCH-016 | Нет бизнес-контекста, нет differentiation budget |
| Полная симуляция | Избыточно для solo |
| Копировать ScyllaDB sizing guide | Не учитывает embedded-нишу и Go-специфику |
| Ad-hoc расчёты в issues | Не воспроизводимо |
| Не разделять профили | Latency и Compliance имеют разные требования |
| **Не разделять tiered** | **Tiered — основа дифференциации (D107, D108)** |
| **Не учитывать Dragonboat** | **Distributed (v0.6a) без capacity — риск** |

---

## 11. Что дальше

1. **Утвердить ADR-035 v2.3** → статус `Accepted`.
2. **Проверить HLD v11.2 §6.3:** Bloom 840 МБ (уже исправлено).
3. **Запустить BENCH-010** после v0.2.
4. **Запустить BENCH-012** (server mode overhead) в v0.6a.
5. **Запустить BENCH-014** (Dragonboat Multi-Raft throughput) в v0.6a.
6. **Запустить BENCH-016** (Tiered Latency) в v0.1.
7. **Написать ADR-039** (Dragonboat dependency risk) — версия, пиннинг, план B.
8. **Написать ADR-040** (Migration v0.5 → v0.6a) — WAL pool → Tan engine.
9. **Обновлять модель** при каждом ADR.

---

## 12. Открытые вопросы (честно)

- **T2 достижимость (L3-resident, 1M keys, p999 < 500 нс):** не измерено. BENCH-016.
- **T3 достижимость (RAM-resident, 10M keys, p999 < 2 мкс):** не измерено. BENCH-016.
- **Tier distribution в реальной нагрузке:** не измерено. BENCH-016.
- **Dragonboat throughput на 10000 groups:** не измерено. BENCH-014.
- **Dragonboat heartbeat batching при 10000 groups × N nodes:** не измерено.
- **Dragonboat quiescing: сколько групп активно в реальной нагрузке:** не измерено.
- **Tan engine интеграция:** не измерено. Spike в начале v0.6a.
- **Кастомный LogDB (`ILogDB`) overhead:** не измерено. `ILogDB` не считается стабильным public API.
- **Встроенный TCP Dragonboat throughput:** не измерено.
- **`AllocsPerDistributedWrite`:** не измерено. BENCH-014.
- **Bloom-память:** 9.6 бит × 6–7 уровней ≈ 840 МБ. Tiered bloom → ~300–400 МБ. Не измерено.
- **VLog GC overhead:** 30% space amp — оценка.
- **Shard-per-core scaling:** 0.47× на 64 cores — прогноз.
- **Shared WAL recovery 10000 shards:** ~1 сек — прогноз.
- **CodecV2 миграция:** 2.4×/2.7× — из grpc-go benchmark.
- **Columnar replica RAM:** +20% — оценка.
- **Encryption CPU overhead:** 10–20% AES-NI — оценка.
- **TCO Ops:** $800 за 3 года solo — оценка.
- **Свой Raft (v1.0+):** +5% CPU vs Dragonboat — прогноз.

**Правило:** неизвестное документируется как неизвестное.

---

## 13. Ссылки

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v11.2 | Архитектура, §2.3, §4.2, §4.6, §5.4, §6.7, §6.8, §9 |
| TephraKV-PRD-001 v1.3 | ICP, use cases, TAM |
| TephraKV-GTM-001 v2.3 | Go-to-Market |
| TephraKV-PRICING-001 v1.3 | Pricing & Packaging |
| TephraKV-ROADMAP-001 v4.0 | §8 Capacity, §8.2 Временной бюджет |
| TephraKV-BACKLOG-001 v6.0 | CHORE-008, BENCH-016 |
| TephraKV-GLOSSARY-001 v3.4 | Capacity model, TCO, Profile, Tier |
| ADR-003 | Bloom filter, partition index |
| ADR-005 v5 | Transport: встроенный TCP Dragonboat (Raft) + gRPC (server mode) |
| ADR-006 v5 | Raft core: Dragonboat |
| ADR-009 v2 | Zero-alloc scope: framing vs codec |
| ADR-010 v3 | Durability semantics: per-op через кастомный LogDB |
| ADR-012 v6 | WAL shared pool (single-node). Tan engine для distributed. |
| ADR-021 | Compaction policy (SLA-driven) |
| ADR-022 | VLog (WiscKey) |
| ADR-023 | VLog GC |
| ADR-024 | Columnar storage |
| ADR-026 v5 | Dragonboat Multi-Raft: Tan engine, встроенный TCP |
| ADR-029 v2 | Wire format: CodecV2, не protobuf |
| ADR-036 v1 | Read path: ReadIndex + Lease Read + Tier-aware routing |
| ADR-037 | Profile semantics (Latency / Compliance) |
| ADR-035 v2.0 | Предыдущая версия |

---

## 14. Что изменилось против v2.0

**Синхронизация с HLD v11.2, ROADMAP v4.0, BACKLOG v6.0.**

Ключевые правки:

- **Tiered capacity model:** T1/T2/T3/T4 с конкретными working set, cache levels, latency targets.
- **BR-13..BR-21:** tiered p999, tier distribution, Dragonboat throughput, `AllocsPerDistributedWrite`, Tan engine, кастомный LogDB, heartbeat RPC, memory на 10000 groups.
- **§3.3 Latency:** tiered (T1–T4) вместо «p999 < 5 мс».
- **§3.5 Dragonboat Multi-Raft:** precedents (1.25M writes/s на группу, 9M на 22 ядрах).
- **§3.7 Standards:** tiered p999, Raft throughput.
- **§4.1:** добавлены Dragonboat, Tan engine, кастомный LogDB, встроенный TCP.
- **§4.2:** Dragonboat vs свой Raft, Tan engine vs RocksDB LogDB, встроенный TCP.
- **§4.3:** добавлен столбец «vs TiKV», `AllocsPerDistributedWrite`.
- **§5.3:** добавлен столбец Tier. Dragonboat NodeHost, Tan engine, кастомный LogDB RAM.
- **§5.6:** новый — disk budget distributed (Tan engine ~10 ГБ vs 10 ТБ).
- **§5.7 CPU:** добавлен столбец Tier. Tiered cycles.
- **§5.8:** server mode gRPC (100–250 streams) + встроенный TCP Dragonboat.
- **§6 Tiered Capacity Model:** новый раздел (tier definitions, сегменты, targets, distribution, distributed tier).
- **§7.6 v0.6a:** Dragonboat, Tan engine, кастомный LogDB. Distributed capacity.
- **§7.7 v0.6b:** SQL + columnar replica.
- **§8.5 Dragonboat Multi-Raft capacity:** новый раздел (BR-16..BR-21).
- **§11:** BENCH-014, BENCH-016, ADR-039, ADR-040.
- **§12:** открытые вопросы обновлены (T2/T3 достижимость, tier distribution, Dragonboat throughput, Tan engine, ILogDB).
- **§13:** ссылки обновлены (HLD v11.2, ROADMAP v4.0, BACKLOG v6.0, ADR-005 v5, ADR-006 v5, ADR-010 v3, ADR-012 v6, ADR-026 v5, ADR-029 v2, ADR-036 v1).

Структурные правки:

- §5.3: разделение на Profile Latency / Compliance / Distributed (Dragonboat).
- §5.8: server mode vs Raft transport.
- §6: новый раздел Tiered Capacity Model.
- §7.6: Dragonboat capacity (Tan engine, кастомный LogDB).
- §8.5: Dragonboat Multi-Raft capacity.
- §14: новая секция.

Исправленные ошибки:

- Bloom math: 840 МБ (подтверждено).
- BR-5: уточнено.
- **Замена «etcd/raft» → «Dragonboat»** во всех ссылках.
- **Замена «gRPC transport» → «встроенный TCP Dragonboat»** для Raft.
- **Tiered latency** вместо «p999 < 5 мс».

---

**Конец ADR-035 v2.3**
