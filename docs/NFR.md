# TephraKV — Non-Functional Requirements

**Документ:** TephraKV-NFR-001
**Версия:** 1.0
**Статус:** Черновик
**Дата:** 29 сентября 2026
**Связь:** TephraKV-HLD-000 v11.3, PRD-001 v1.3, ROADMAP-001 v4.0, BACKLOG-001 v6.1, API-001 v1.3, FORMAT-001 v1.3, GLOSSARY-001 v3.5, ADR-005 v5, ADR-006 v5, ADR-009 v2, ADR-010 v3, ADR-012 v6, ADR-019, ADR-020, ADR-021, ADR-022, ADR-026 v5, ADR-029 v2, ADR-034 v2.2, ADR-035 v2.4, ADR-036 v1
**Классификация:** Внутренний / публичный

---

## 0. Область и правила

Документ фиксирует нефункциональные требования к TephraKV по всем версиям от v0.1 до v1.0. Каждое требование имеет приоритет (MUST / SHOULD / COULD / WON'T), целевую версию, конкретное числовое значение и метод верификации. Если требование не покрыто методом верификации — оно не подлежит приёмке.

Документ отвечает на вопрос «что требуется» и «как проверяем». HLD отвечает на «как устроено». BENCH отвечает на «как измеряем». ADR фиксирует решения.

Приоритеты:

- **MUST** — блокирует релиз версии, в которой заявлено.
- **SHOULD** — нарушение требует ADR с обоснованием.
- **COULD** — не блокирует, включается по возможности.
- **WON'T** — явно не делаем в этой версии, зафиксировано как non-goal.

Профиль TephraKV — детерминированная хвостовая задержка. Все latency-требования используют percentile (p50 / p99 / p999) и указывают tier (T1–T4) из HLD §2.3. Публикация p999 без указания tier запрещена (D113).

---

## 1. Performance

### 1.1. Определения

Все latency-требования в этом разделе измеряются:

- **Профиль:** YCSB C (100% read), key 16 байт, value 64 байта.
- **Concurrency:** 1 поток (single-thread), если не указано иное.
- **Cache state:** warm (данные прогреты, операционная система и процессорный кэш в стабильном состоянии).
- **Длительность:** 60 секунд warmup, 300 секунд измерения, минимум 5 прогонов, медиана.
- **Метрика:** перцентиль по всем операциям за период измерения.
- **Hardware:** фиксируется в отчёте (модель CPU, частота, размер L1/L2/L3, объем RAM, модель диска).
- **Источник:** BENCH-016.

### 1.2. Single-node latency (embedded)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-001 | p50 GET T1 (≤ 100K keys, L1/L2) | < 50 нс | MUST | v0.1 |
| PERF-002 | p99 GET T1 | < 100 нс | MUST | v0.1 |
| PERF-003 | p999 GET T1 | < 200 нс | MUST | v0.1 |
| PERF-004 | p50 GET T2 (≤ 1M keys, L3) | < 100 нс | MUST | v0.1 |
| PERF-005 | p99 GET T2 | < 300 нс | MUST | v0.1 |
| PERF-006 | p999 GET T2 | < 500 нс | MUST | v0.1 |
| PERF-007 | p50 GET T3 (≤ 100M keys, RAM) | < 300 нс | MUST | v0.1 |
| PERF-008 | p99 GET T3 | < 1 мкс | MUST | v0.1 |
| PERF-009 | p999 GET T3 | < 2 мкс | MUST | v0.1 |
| PERF-010 | p50 GET T4 (cold NVMe) | < 5 мкс | SHOULD | v0.1 |
| PERF-011 | p99 GET T4 | < 20 мкс | SHOULD | v0.1 |
| PERF-012 | p999 GET T4 | < 50 мкс | SHOULD | v0.1 |

**Уточнение по tier.** Tier определяется по рабочему набору (working set) и определяется промахами кэша, а не размером базы. Для T1 — весь набор помещается в L2 (типично 512 КБ – 1 МБ на ядро), для T2 — в L3 (типично 8–32 МБ), для T3 — в RAM, для T4 — читается с NVMe. Методика измерения — счётчики `L1-dcache-load-misses`, `LLC-load-misses` через perf.

**Уточнение по частоте.** Значения в наносекундах рассчитаны на CPU с частотой 3 ГГц и выше. Если CPU медленнее, значение в тактах остаётся целевым, но результат в наносекундах пересчитывается и публикуется с указанием частоты. Cycles/op фиксируются в PERF-031..034.

### 1.3. PUT latency

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-013 | p99 PUT (SYNC_MASTER, NVMe) | < 5 мс | MUST | v0.1 |
| PERF-014 | p999 PUT (SYNC_MASTER) | < 10 мс | MUST | v0.1 |
| PERF-015 | p99 PUT (NO_SYNC) | < 1 мкс | MUST | v0.1 |
| PERF-016 | p999 PUT (NO_SYNC) | < 5 мкс | MUST | v0.1 |
| PERF-017 | p99 PUT (SYNC_MASTER, SATA SSD) | < 15 мс | SHOULD | v0.1 |
| PERF-018 | p99 group commit window | < 100 мкс | MUST | v0.1 |

PERF-013 измеряется на NVMe с гарантированной queue depth ≥ 32. PERF-017 — допущение для SATA SSD (не целевой hardware, но должен работать без нарушения контракта). Значения для SYNC_MASTER учитывают fsync latency; если fsync на конкретном диске медленнее, требование не нарушено — это физическое ограничение диска.

### 1.4. Distributed latency (v0.6a)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-019 | p99 GET (ReadIndex, single-region) | < 10 мс | MUST | v0.6a |
| PERF-020 | p999 GET (ReadIndex) | < 50 мс | MUST | v0.6a |
| PERF-021 | p99 PUT (SYNC_MAJORITY) | < 20 мс | MUST | v0.6a |
| PERF-022 | p999 PUT (SYNC_MAJORITY) | < 100 мс | MUST | v0.6a |
| PERF-023 | Freshness p99 (OLTP → HTAP replica) | < 1 с | MUST | v0.6a |
| PERF-024 | Leader failover p99 | < 3 с | MUST | v0.6a |
| PERF-025 | Snapshot transfer throughput (per node) | ≥ 100 МБ/с | SHOULD | v0.6a |

Измерения проводятся на кластере 3 нод в одной зоне доступности, network RTT 0.5 мс (p99). PERF-019 и PERF-020 не применимы к embedded-режиму и не нарушают целей v0.1 — другая физика (сетевой round-trip + Raft commit).

### 1.5. Tier distribution

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-026 | Доля операций в L1/L2/L3 при 1M keys | ≥ 80% | MUST | v0.1 |
| PERF-027 | Доля операций в L3 при 10M keys | ≥ 70% | SHOULD | v0.1 |
| PERF-028 | Доля операций в RAM при 100M keys | ≥ 50% | SHOULD | v0.3 |

Значение PERF-026 измеряется как процент операций, попавших в кэш процессора без обращения к RAM или диску. Хотя бы одна операция из пяти может промахнуться — это физический предел распределения доступа. Митигация: tier-aware compaction (ADR-021).

### 1.6. Throughput (single-node)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-029 | GET: ops/s на 4 vCPU | ≥ 1M | MUST | v0.1 |
| PERF-030 | PUT (NO_SYNC, group commit): ops/s на 4 vCPU | ≥ 500K | MUST | v0.1 |
| PERF-031 | Mixed YCSB A (50/50): ops/s на 4 vCPU | ≥ 250K | MUST | v0.1 |
| PERF-032 | Mixed YCSB B (95/5): ops/s на 4 vCPU | ≥ 800K | MUST | v0.1 |
| PERF-033 | GET: ops/s на 64-core (in-memory) | ≥ 50M | SHOULD | v0.4 |

Измерение — YCSB с concurrency равной числу ядер. Значения агрегированные; первая цифра — общий throughput, не per-core.

### 1.7. Throughput (distributed, v0.6a)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-034 | Dragonboat: ops/s на узел | ≥ 1M | MUST | v0.6a |
| PERF-035 | Raft group: writes/s | ≥ 500K | SHOULD | v0.6a |
| PERF-036 | Heartbeat RPC/s при 10000 групп | ≤ 200 | MUST | v0.6a |
| PERF-037 | Memory на 10000 групп | ≤ 500 МБ | SHOULD | v0.6a |

Ориентир для PERF-034: опубликованные бенчмарки Dragonboat — 1.25M writes/s на одну группу, 9M writes/s на 22 ядрах. Наш целевой показатель (1M ops/s на узел на 8 vCPU) — консервативный; фактическое значение фиксируется в BENCH-014.

### 1.8. Allocations

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-038 | Allocs/op на hot path (Put, Get, Delete, Scan, WriteBatch) | 0 | MUST | v0.1 |
| PERF-039 | Allocs/op на transport codec (server mode) | ≤ 2 | MUST | v0.6a |
| PERF-040 | AllocsPerDistributedWrite | ≤ 10 | MUST | v0.6a |

PERF-038 проверяется CI-бенчмарком с `-benchmem` на каждой версии Go (D69). Escape analysis используется как диагностика, но не как контракт — она не стабильна между версиями Go.

### 1.9. Cycles per operation

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-041 | Cycles/op GET p50 (T2) | < 300 | MUST | v0.1 |
| PERF-042 | Cycles/op GET p99 (T2) | < 900 | MUST | v0.1 |
| PERF-043 | Cycles/op GET p999 (T2) | < 1500 | MUST | v0.1 |
| PERF-044 | Cycles/op PUT NO_SYNC p99 | < 500 | SHOULD | v0.1 |

Измерение — `rdtsc` на x86_64, `cntvct_el0` на ARM64. На ARM64 метрика публикуется как ticks, не cycles (D16). Overhead rdtsc (20–40 тактов) вычитается из результата: публикуются две цифры — raw и estimated. Для ARM64 частота System Counter указывается в отчёте (24 МГц на Apple M, ~1 ГГц на Graviton).

### 1.10. Amplification

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-045 | Read amp p99 (disk reads per GET) | < 3 | MUST | v0.1 |
| PERF-046 | Read amp p99 | < 2 | MUST | v0.3 |
| PERF-047 | Write amp (средняя) | < 5 | MUST | v0.1 |
| PERF-048 | Write amp (средняя) | < 3 | MUST | v0.3 |
| PERF-049 | Space amp (LSM-only) | < 1.5 | SHOULD | v0.2 |
| PERF-050 | Space amp (VLog, после GC) | < 1.3 | SHOULD | v0.3 |
| PERF-051 | Bloom false positive rate при 9.6 бит/ключ | < 1% | MUST | v0.1 |
| PERF-052 | Compaction 1 ГБ L0: время | < 30 с | SHOULD | v0.2 |
| PERF-053 | Read latency p99 во время compaction | < 2× baseline | MUST | v0.2 |
| PERF-054 | Флашей на 1M операций | < 100 | MUST | v0.1 |

Read amp — число физических чтений с диска (memory reads не считаются). Space amp — размер данных на диске, делённый на размер живых данных. WA измеряется как физически записанные байты к логически записанным.

### 1.11. VLog (v0.3+)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-055 | VLog read latency p99 (mmap, hot) | < 5 мкс | MUST | v0.3 |
| PERF-056 | VLog write throughput | ≥ 1.5× v0.2 | SHOULD | v0.3 |
| PERF-057 | GC overhead | < 5% CPU | SHOULD | v0.3 |
| PERF-058 | VLog fragmentation | < 20% | SHOULD | v0.3 |
| PERF-059 | GC 1 ГБ VLog: время | < 60 с | SHOULD | v0.3 |
| PERF-060 | T2 p999 деградация для append-heavy workload | < 10% | MUST | v0.3 |

PERF-060 — ключевое требование, отличающее v0.3 от v0.2: tiered-цель v0.1 не должна деградировать при введении VLog. Если деградация > 10%, tier-aware compaction не справляется, и v0.3 не принимается.

### 1.12. Resource footprint

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| PERF-061 | Аллокация 64 Б в arena | < 5 нс | SHOULD | v0.1 |
| PERF-062 | Размер бинарника ядра | < 5 МБ | SHOULD | v0.1 |
| PERF-063 | RAM per key (без Bloom) | ≤ 2 Б | SHOULD | v0.1 |
| PERF-064 | RAM per key (с Bloom 9.6 бит/ключ) | ≤ 4 Б | SHOULD | v0.1 |
| PERF-065 | RAM относительно Redis при durability | ≤ 1.5× | SHOULD | v0.5 |

PERF-065 — сравнение с Redis при эквивалентной durability (AOF everysec). Redis на 100M keys × 64 Б value + 32 Б key занимает ~15–20 ГБ. TephraKV — ~8.6 ГБ при том же наборе. Значение измеряется в BENCH-010.

---

## 2. Scalability

### 2.1. Vertical scaling

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| SCALE-001 | Scaling до 8 cores | ≥ 0.85× на ядро | MUST | v0.4 |
| SCALE-002 | Scaling до 16 cores | ≥ 0.75× на ядро | SHOULD | v0.4 |
| SCALE-003 | Scaling до 64 cores | ≥ 0.45× на ядро | SHOULD | v0.4 |

Scaling factor — отношение throughput при N ядрах к throughput при 1 ядре, делённое на N. Значения ниже 1 из-за contention на memory bus и NUMA. 0.45× на 64 cores — 28.8× относительно 1 ядра — ожидаемый результат для shared memory архитектуры без специальной NUMA-оптимизации.

### 2.2. Data volume

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| SCALE-004 | Single-node: 100M keys без деградации p99 | Да | MUST | v0.3 |
| SCALE-005 | 10× рост данных: p99 GET деградация | ≤ 2× | SHOULD | v0.3 |
| SCALE-006 | Single-node: до 5 ТБ данных | Да | SHOULD | v0.3 |
| SCALE-007 | Distributed: до 6 ТБ на узел (10 shards × 600 ГБ) | Да | SHOULD | v0.6a |

«Без деградации p99» означает: p99 GET T4 остаётся < 50 мкс при росте с 10M до 100M keys. Это требование к read amp — partition index и Bloom должны масштабироваться сублинейно.

### 2.3. Horizontal scaling (v0.6a+)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| SCALE-008 | Cluster size (v0.6a) | 3–5 нод | MUST | v0.6a |
| SCALE-009 | Cluster size (v1.0) | до 20 нод | SHOULD | v1.0 |
| SCALE-010 | Throughput: линейный scaling до 3 нод | ≥ 0.8× | SHOULD | v0.6a |
| SCALE-011 | Rebalance: перемещение лидерства | Да | MUST | v0.6a |
| SCALE-012 | Snapshot transfer: incremental, resume | Да | MUST | v0.6a |
| SCALE-013 | Raft groups per cluster | до 10000 | SHOULD | v0.6a |

### 2.4. Tier boundaries

Tier определяется рабочим набором, не размером базы. Границы фиксированы:

- T1: ≤ 100K keys (рабочий набор ≤ 6.4 МБ при key 16 Б + value 64 Б, с метаданными ~10 МБ).
- T2: ≤ 1M keys (рабочий набор ≤ 100 МБ).
- T3: ≤ 100M keys (рабочий набор ≤ 10 ГБ).
- T4: > 100M keys (любой рабочий набор, читается с NVMe).

Границы не жёсткие — при переходе через границу доля операций в целевом tier уменьшается, но tier-цель остаётся достижимой до 1.5× границы. За границей 1.5× tier меняется.

---

## 3. Availability and recovery

### 3.1. Startup и recovery (single-node)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| AVAIL-001 | Startup time на 10 ГБ данных | < 5 с | MUST | v0.1 |
| AVAIL-002 | Recovery time на 1 ГБ WAL | < 10 с | MUST | v0.1 |
| AVAIL-003 | Recovery 10000 shards (parallel, 8 concurrent) | < 5 с | SHOULD | v0.6a |
| AVAIL-004 | Детерминированный recovery (2 запуска — одинаковый результат) | Да | MUST | v0.1 |

### 3.2. Distributed availability (v0.6a+)

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| AVAIL-005 | Availability (3 ноды, single-region) | ≥ 99.9% | MUST | v0.6a |
| AVAIL-006 | Availability (5 нод, single-region) | ≥ 99.99% | SHOULD | v0.6a |
| AVAIL-007 | Availability (multi-region) | ≥ 99.99% | SHOULD | v1.0 |
| AVAIL-008 | Leader failover p99 | < 3 с | MUST | v0.6a |
| AVAIL-009 | No data loss on failover (SYNC_MAJORITY) | Да | MUST | v0.6a |
| AVAIL-010 | Network partition: no split-brain | Да | MUST | v0.6a |
| AVAIL-011 | Rolling restart без downtime | Да | MUST | v0.6a |

### 3.3. MTTR / MTBF

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| AVAIL-012 | MTTR (S1: data loss) | < 8 ч | SHOULD | v0.5 |
| AVAIL-013 | MTTR (S2: downtime) | < 24 ч | SHOULD | v0.5 |
| AVAIL-014 | MTTR (S3: деградация) | < 72 ч | COULD | v0.5 |
| AVAIL-015 | Change failure rate | < 10% | SHOULD | v0.5 |
| AVAIL-016 | Ops hours/month (single-node) | ≤ 4 ч | SHOULD | v0.5 |
| AVAIL-017 | Ops hours/month (5 nodes) | ≤ 20 ч | SHOULD | v0.6a |

### 3.4. RTO / RPO

| Сценарий | RTO | RPO | Версия |
|---|---|---|---|
| Single-node, 100 ГБ | < 60 с | 0 (SYNC_MASTER) | v0.5 |
| Distributed, 1 ТБ (3 ноды) | < 5 мин | 0 (SYNC_MAJORITY) | v0.6a |
| Multi-region (v1.0) | < 30 мин | 0 | v1.0 |

RTO — время от начала восстановления до готовности приёма операций. RPO — максимальная потеря подтверждённых данных. RPO = 0 для SYNC_MASTER и SYNC_MAJORITY; для NO_SYNC RPO равен окну group commit.

---

## 4. Durability

### 4.1. Гарантии по политикам

| Политика | Гарантия | Потеря при краше | Версия |
|---|---|---|---|
| NO_SYNC | Запись в WAL буфер, fsync в окне group commit | Последние N мс (N = окно) | v0.1 |
| SYNC_MASTER | fsync локально | 0 потерь | v0.1 |
| SYNC_LEADER | fsync локально на лидере | 0 потерь на лидере, возможна потеря при смене лидера | v0.6a |
| SYNC_MAJORITY | fsync на кворуме | 0 потерь при отказе меньшинства | v0.6a |
| SYNC_ALL | fsync на всех репликах | 0 потерь при отказе меньшинства | v0.6a |

Mixed batch (разные политики в одном батче) запрещён — возвращает `ErrMixedDurability` (ADR-010).

### 4.2. Crash recovery

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| DUR-001 | Crash-recovery harness | 10000/10000 успешных | MUST | v0.1 |
| DUR-002 | Partial write: recovery до последнего fsync | Да | MUST | v0.1 |
| DUR-003 | CRC mismatch: fail fast | Да | MUST | v0.1 |
| DUR-004 | Bit rot: детекция через CRC32C | Да | MUST | v0.1 |
| DUR-005 | Manifest corruption: fail fast | Да | MUST | v0.1 |

### 4.3. Backup и PITR

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| DUR-006 | Backup: копия директории | Да | MUST | v0.1 |
| DUR-007 | PITR: восстановление на произвольный момент | Да | MUST | v0.5 |
| DUR-008 | PITR: retention | 30 дней | SHOULD | v0.5 |
| DUR-009 | PITR: 100 ГБ данных работает | Да | MUST | v0.5 |
| DUR-010 | PITR: RTO на 100 ГБ | < 60 с | SHOULD | v0.5 |

---

## 5. Consistency

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| CONS-001 | Single-node: linearizable single-key | Да | MUST | v0.1 |
| CONS-002 | Scan: snapshot на момент начала | Да | MUST | v0.1 |
| CONS-003 | WriteBatch: atomic (all or nothing) | Да | MUST | v0.1 |
| CONS-004 | Read-your-writes в одной goroutine | Да | MUST | v0.1 |
| CONS-005 | Distributed read: linearizable (ReadIndex) | Да | MUST | v0.6a |
| CONS-006 | Distributed: консистентность при clock skew | Не нарушается | MUST | v0.6a |
| CONS-007 | Snapshot Isolation (v0.4) | Да | MUST | v0.4 |
| CONS-008 | Cross-shard транзакции: 2PC | Да | MUST | v0.6a |
| CONS-009 | Serializable (SSI) | Да | SHOULD | v1.0 |
| CONS-010 | Freshness (HTAP): p99 | < 1 с | MUST | v0.6a |

CONS-006 — ключевое отличие ReadIndex от Lease Read: первый не использует часы и не может выдать stale read при рассинхроне. Lease Read (опционально) требует bounded drift < 1000 ppm и монотонных часов.

---

## 6. Security

### 6.1. Storage threat model (v0.1)

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| SEC-001 | Файлы БД создаются с правами 0600 | MUST | v0.1 |
| SEC-002 | CRC32C на всех файлах (WAL, SSTable, Manifest, VLog) | MUST | v0.1 |
| SEC-003 | Bounds check на key/value при чтении | MUST | v0.1 |
| SEC-004 | Fuzz malformed WAL / SSTable / Manifest | MUST | v0.1 |
| SEC-005 | Zero runtime dependencies (снижает supply chain risk) | MUST | v0.1 |
| SEC-006 | `govulncheck` в CI, блокирует merge при CVE | MUST | v0.1 |
| SEC-007 | Подпись релизных артефактов (cosign) | SHOULD | v0.5 |

Модель угроз v0.1: локальный атакующий с доступом к файлам, malformed on-disk data, supply chain атаки через dev-зависимости. Вне scope: удалённый атакующий (нет server mode), side-channel, физический доступ.

### 6.2. Network threat model (v0.6a)

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| SEC-008 | TLS 1.3 (только) для server mode | MUST | v0.6a |
| SEC-009 | mTLS для inter-node Raft | SHOULD | v0.6a |
| SEC-010 | Rate limiting на server mode | MUST | v0.6a |
| SEC-011 | Метрики без keys и values | MUST | v0.6a |
| SEC-012 | Replay protection (Raft term/index + TLS session) | MUST | v0.6a |

### 6.3. Encryption and keys (v0.6b)

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| SEC-013 | Encryption at rest: AES-256-GCM | MUST | v0.6b |
| SEC-014 | Key rotation: 30 дней (configurable) | SHOULD | v0.6b |
| SEC-015 | Ключи через KMS или local file с правами 0600 | MUST | v0.6b |
| SEC-016 | Ключи не в логах и метриках | MUST | v0.6b |
| SEC-017 | Overhead encryption on hot path | ≤ 20% CPU | SHOULD | v0.6b |

### 6.4. RBAC и audit (v1.0)

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| SEC-018 | RBAC (роли, права) | MUST | v1.0 |
| SEC-019 | Audit log: append-only, immutable | MUST | v0.5 |
| SEC-020 | Audit log export (S3, Glacier) | SHOULD | v0.7 |
| SEC-021 | Audit retention ≥ 7 лет | SHOULD | v1.0 |

---

## 7. Observability

### 7.1. Metrics

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| OBS-001 | Two-Level Metrics (внешние + внутренние) | Да | MUST | v0.1 |
| OBS-002 | Latency p50/p99/p999 по операциям | Да | MUST | v0.1 |
| OBS-003 | Tier distribution в InternalMetrics | Да | MUST | v0.1 |
| OBS-004 | Throughput, cost per million ops | Да | MUST | v0.1 |
| OBS-005 | Allocs/op по операциям | Да | MUST | v0.1 |
| OBS-006 | Read amp, write amp, space amp | Да | MUST | v0.1 |
| OBS-007 | Compaction metrics (reasons, durations) | Да | MUST | v0.2 |
| OBS-008 | VLog metrics (fragmentation, GC) | Да | MUST | v0.3 |
| OBS-009 | Raft metrics (leader, term, lag) | Да | MUST | v0.6a |
| OBS-010 | Prometheus exporter (отдельный пакет) | Да | MUST | v0.5 |
| OBS-011 | Метрики доступны без внешних агентов | Да | MUST | v0.5 |
| OBS-012 | Метрики не содержат keys/values | Да | MUST | v0.1 |

### 7.2. Tracing и profiling

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| OBS-013 | `runtime/trace` (стандарт Go) | MUST | v0.1 |
| OBS-014 | OpenTelemetry | SHOULD | v0.6a |
| OBS-015 | `net/http/pprof` только в dev build | MUST | v0.1 |
| OBS-016 | eBPF observability | COULD | v0.8 |

---

## 8. Maintainability

### 8.1. Code quality gates

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| MAINT-001 | `go vet`, `golangci-lint`, `staticcheck` — CI зелёный | MUST | v0.1 |
| MAINT-002 | `go test -race` — все тесты проходят | MUST | v0.1 |
| MAINT-003 | Coverage ядра > 80% | SHOULD | v0.1 |
| MAINT-004 | Fuzz 1M ops без panic/race/corruption | MUST | v0.1 |
| MAINT-005 | `go-arch-lint`: нет циклов между модулями | MUST | v0.1 |
| MAINT-006 | Escape analysis: no new heap escapes на hot path | MUST | v0.1 |
| MAINT-007 | CodecV2 (не V1 bridge) в server mode | MUST | v0.6a |
| MAINT-008 | `sync.Pool` запрещён в data plane | MUST | v0.1 |

MAINT-003 — coverage считается только для `internal/*` и `pkg/*`. Тестовые пакеты исключены. Bench-пакеты исключены. Достижимое значение для storage-движка — 80–85% без формализма; для API-пакета — 90%.

### 8.2. Versioning

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| MAINT-009 | Semver | MUST | v0.1 |
| MAINT-010 | Формат данных: N-1 backward compat | MUST | v0.1 |
| MAINT-011 | Reader v0.N отказывает формат v0.(N+1) | MUST | v0.1 |
| MAINT-012 | Deprecation: 2 minor версии до удаления | MUST | v0.1 |
| MAINT-013 | CHANGELOG обновляется на каждый PR | MUST | v0.1 |

### 8.3. Documentation

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| MAINT-014 | Каждое архитектурное решение — ADR | MUST | v0.1 |
| MAINT-015 | Godoc на весь публичный API | MUST | v0.1 |
| MAINT-016 | README с quick start | MUST | v0.1 |
| MAINT-017 | Benchmark methodology публичная | MUST | v0.1 |
| MAINT-018 | Примеры компилируются и запускаются в CI | MUST | v0.1 |

### 8.4. Process

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| MAINT-019 | DoR/DoD на каждую задачу | MUST | v0.1 |
| MAINT-020 | Релиз + статья каждые 6–8 недель | SHOULD | v0.1 |
| MAINT-021 | WIP ≤ 3 | SHOULD | v0.1 |
| MAINT-022 | Cycle time < 3 дней | SHOULD | v0.1 |
| MAINT-023 | Baseline SP/неделя известен (по факту первых 4 недель) | SHOULD | v0.1 |

### 8.5. Benchmark regression

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| MAINT-024 | Регрессия p99 при merge | ≤ 5% | MUST | v0.1 |
| MAINT-025 | Регрессия throughput при merge | ≤ 10% | MUST | v0.1 |
| MAINT-026 | Baseline хранится в репозитории | Да | MUST | v0.1 |

Gate реализован через `benchstat` в CI. Baseline обновляется явным решением после значимых улучшений (не автоматически).

---

## 9. Portability

### 9.1. Платформы

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| PORT-001 | Linux x86_64 — production | MUST | v0.1 |
| PORT-002 | Linux ARM64 — production | SHOULD | v0.1 |
| PORT-003 | macOS — build support (не production) | SHOULD | v0.3 |
| PORT-004 | Windows — build support (не production) | SHOULD | v0.3 |
| PORT-005 | Windows — production | COULD | v0.4 |

Версия v0.1 официально поддерживает только Linux x86_64 и ARM64. macOS и Windows собираются, но не гарантируют производительность. Production-ready на этих платформах — с v0.3 (macOS) и v0.4 (Windows).

### 9.2. Go версии

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| PORT-006 | Go 1.24+ | MUST | v0.1 |
| PORT-007 | Zero-alloc gate на каждой поддерживаемой версии Go | MUST | v0.1 |
| PORT-008 | Benchmark regression на каждой поддерживаемой версии Go | MUST | v0.1 |
| PORT-009 | Поддержка N и N-1 версии Go | SHOULD | v0.1 |

### 9.3. Зависимости

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| PORT-010 | Zero runtime dependencies в core | MUST | v0.1 |
| PORT-011 | Dev-зависимости в build tag `tools` | MUST | v0.1 |
| PORT-012 | cgo запрещён | MUST | v0.1 |
| PORT-013 | `sync.Pool` запрещён в data plane | MUST | v0.1 |
| PORT-014 | `fmt`, `reflect`, `interface{}` запрещены на hot path | MUST | v0.1 |

### 9.4. mmap

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| PORT-015 | mmap на Linux x86_64 | MUST | v0.1 |
| PORT-016 | mmap на Linux ARM64 | SHOULD | v0.1 |
| PORT-017 | Fallback на `make([]byte, size)` на других платформах | MUST | v0.1 |

Fallback работает, но не даёт тот же performance (нет mmap-madvise, аллокации в heap). Документируется как известное ограничение.

---

## 10. Developer experience

### 10.1. API

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| UX-001 | Godoc на весь публичный API | MUST | v0.1 |
| UX-002 | Примеры для ProfileLatency и ProfileCompliance | MUST | v0.1 |
| UX-003 | Sentinel errors, совместимые с `errors.Is` | MUST | v0.1 |
| UX-004 | Обёртка ошибок через `%w` | SHOULD | v0.1 |
| UX-005 | `Get` возвращает `(value, release)` — контракт в типе | MUST | v0.1 |
| UX-006 | Bounds-ошибки с указанием лимита | MUST | v0.1 |

### 10.2. Onboarding

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| UX-007 | Quick start: от `go get` до первого Put ≤ 5 минут | SHOULD | v0.1 |
| UX-008 | Все примеры в docs компилируются | MUST | v0.1 |
| UX-009 | Docker-образ для benchmark | SHOULD | v0.1 |
| UX-010 | Миграционный гайд (Badger/Pebble → TephraKV) | SHOULD | v0.3 |

### 10.3. Error messages

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| UX-011 | Ошибки на английском | MUST | v0.1 |
| UX-012 | Ошибки содержат actionable info (что делать) | SHOULD | v0.1 |
| UX-013 | Ошибки не содержат keys/values | MUST | v0.1 |
| UX-014 | Ошибки не содержат путей файлов, кроме корневой директории БД | SHOULD | v0.1 |

---

## 11. Compliance

### 11.1. Сертификации

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| COMP-001 | SOC 2 Type II | MUST | v1.0 |
| COMP-002 | GDPR (encryption at rest, audit log, PITR) | MUST | v0.6b |
| COMP-003 | ISO 27001 | COULD | v1.0+ |
| COMP-004 | HIPAA | WON'T | — |
| COMP-005 | PCI DSS | WON'T | — |

HIPAA и PCI DSS вне scope — целевые сегменты (HFT, game servers, AI-infra, edge, CDN, fintech/audit) не требуют этих сертификаций в горизонте v1.0.

### 11.2. Аудит

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| COMP-006 | Audit log: append-only, immutable (WORM) | MUST | v0.5 |
| COMP-007 | Audit log export (S3, Glacier) | SHOULD | v0.7 |
| COMP-008 | Audit retention ≥ 7 лет | SHOULD | v1.0 |

### 11.3. Лицензии

| ID | Требование | Приоритет | Версия |
|---|---|---|---|
| COMP-009 | Community: Apache 2.0 | MUST | v0.1 |
| COMP-010 | Enterprise: commercial license | MUST | v0.6 |
| COMP-011 | SBOM (SPDX) | SHOULD | v1.0 |
| COMP-012 | SLSA Level 3 | SHOULD | v1.0 |

---

## 12. Cost efficiency

### 12.1. TCO

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| COST-001 | TCO per 1M ops (solo, 3 года) | < $0.01 | MUST | v0.5 |
| COST-002 | TCO per 1M ops (embedded app) | < $0.005 | SHOULD | v0.5 |
| COST-003 | TCO per 1M ops (HFT, high throughput) | < $0.001 | SHOULD | v0.6a |
| COST-004 | TCO per 1M ops (fintech Compliance) | < $0.015 | SHOULD | v0.6a |

TCO считается по формуле из HLD §4.6: `(Infra_3y + Ops_3y + Migration) / (Useful_ops_3y / 1M)`. Fintech дороже из-за encryption overhead, audit log, PITR archive.

### 12.2. Ресурсы

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| COST-005 | Disk overhead (LSM-only) | < 2× | MUST | v0.1 |
| COST-006 | Disk overhead (VLog) | < 5× | SHOULD | v0.3 |
| COST-007 | CPU: 1M ops/s на ≤ 4 vCPU | Да | MUST | v0.1 |
| COST-008 | RAM относительно Redis (durability) | ≤ 1.5× | SHOULD | v0.5 |

### 12.3. Operational

| ID | Требование | Значение | Приоритет | Версия |
|---|---|---|---|---|
| COST-009 | Ops hours/month (single-node) | ≤ 4 ч | SHOULD | v0.5 |
| COST-010 | Ops hours/month (5 nodes) | ≤ 20 ч | SHOULD | v0.6a |
| COST-011 | Миграция с Badger/Pebble | ≤ 1 инженеро-месяц | SHOULD | v0.5 |
| COST-012 | Миграция с SQLite | ≤ 1 инженеро-месяц | SHOULD | v0.5 |

COST-011 и COST-012 применимы для key-value workloads без SQL. Для SQL-workloads миграция возможна только после v0.6b (SQL subset).

---

## 13. Решённые вопросы

Раздел фиксирует решения по вопросам, которые в предыдущих версиях документа были открытыми.

**Размер WAL сегмента.** 256 МБ по умолчанию, конфигурируемый диапазон 64 МБ – 1 ГБ. Для edge-пресета — 64 МБ (eMMC). Для server-пресета — 1 ГБ (NVMe). Обоснование: 256 МБ — компромисс между частотой rotation и временем recovery. При 1 ГБ recovery составляет ~10 с, что близко к NFR-AVAIL-002.

**Bloom bits/key.** 9.6 для ProfileLatency (эталон ScyllaDB, FP < 1%), 12 для ProfileCompliance (FP < 0.1%). Конфигурируемый диапазон 4–16. Обоснование: 9.6 — минимум для FP < 1%, каждый дополнительный бит снижает FP экспоненциально.

**SST block size.** 4 КБ по умолчанию, конфигурируемый 1–64 КБ. Для edge — 2 КБ. Обоснование: 4 КБ совпадает с размером страницы и даёт хорошую пропускную способность NVMe при чтении. Меньшие блоки выигрывают для точечных чтений, но проигрывают на сканировании.

**Compression в WAL.** По умолчанию нет (CPU-стоимость сжатия выше выигрыша на I/O для типичных write-паттернов). Опция в `Options.Compression` для ProfileCompliance: LZ4. Обоснование: WAL записи обычно небольшие, сжатие даёт минимальный эффект, но добавляет CPU.

**Encryption в WAL.** In-place AES-256-GCM на уровне блока, только для ProfileCompliance (SEC-013). Overhead 10–20% CPU с AES-NI (SEC-017). Обоснование: hardware AES-NI делает шифрование приемлемым по стоимости; без AES-NI overhead неприемлем, документируется как требование к hardware.

**T2 достижимость.** Целевое значение (PERF-006) достижимо при условии CPU с частотой ≥ 3 ГГц и L3 ≥ 16 МБ. Если целевое значение не достигнуто на BENCH-016, публикуется фактическое с указанием hardware и частоты. Требование не ослабляется — оно фиксирует цель, а не обещание; если цель не достигнута, это становится известным до релиза.

**Dragonboat throughput.** Ориентир для PERF-034 — опубликованные бенчмарки Dragonboat (1.25M writes/s на группу, 9M на 22 ядрах). Наш целевой показатель консервативен (1M ops/s на 8 vCPU). Фактическое значение фиксируется в BENCH-014. Если Dragonboat не даёт 1M ops/s на 8 vCPU, требование пересматривается через ADR.

**Benchmark regression gate.** 5% для p99 и 10% для throughput. Обоснование: 5% — порог, ниже которого шум измерения сравним с сигналом; выше 5% регрессия видна в продакшене. Baseline обновляется явным решением, не автоматически.

**Coverage ядра.** > 80% для `internal/*`. API-пакет — 90%. Тестовые и bench-пакеты исключены. Обоснование: 80% — достижимый уровень для storage-движка без формализма; 90% для API — контракт с пользователем, ошибки критичны.

**SOC 2 timeline.** Аудит начинается при v0.6b, завершается при v1.0. Окно аудита — 6 месяцев наблюдения. Обоснование: SOC 2 требует операционной истории; начать раньше v0.6b — нет стабильного продукта; закончить позже v1.0 — блокирует enterprise-продажи.

**MTTR.** S1 (data loss) — 8 часов, S2 (downtime) — 24 часа, S3 (degradation) — 72 часа. Обоснование: solo-разработка не имеет дежурной смены; 8 часов — время бодрствования, за которое можно диагностировать и починить; 24 часа — верхняя граница для потери данных в критичных сегментах.

**Ops hours.** Single-node — 4 ч/мес (примерно 1 ч/нед), 5 nodes — 20 ч/мес. Обоснование: embedded-режим не требует постоянного внимания; distributed требует мониторинга, обновлений, разбора инцидентов.

**Миграция с Badger/Pebble.** ≤ 1 инженеро-месяц для key-value workloads. Обоснование: API совместим на уровне операций (Put/Get/Delete/Scan), но формат данных разный — миграция требует re-import. Для SQL-workloads — 3 инженеро-месяца, только после v0.6b.

**Fallback mmap на macOS/Windows.** Fallback через `make([]byte, size)` работает, но не даёт того же performance (нет mmap-madvise, аллокации в heap). Документируется как известное ограничение. Production-ready на macOS — v0.3, Windows — v0.4.

**Шифрование в WAL при отсутствии AES-NI.** Если CPU не поддерживает AES-NI, включать encryption at rest не рекомендуется. Документируется в README. Fallback на software AES даёт 3–5× overhead, что нарушает SEC-017 и PERF-013.

**ARM64 ticks vs cycles.** На ARM64 публикуются ticks (System Counter), не cycles. Частота System Counter фиксируется в отчёте: 24 МГц на Apple M, ~1 ГГц на Graviton. Конвертация в наносекунды делается через калибровку. Сравнение x86_64 и ARM64 напрямую запрещено (D16).

**Read amp при VLog.** Read amp < 2 (PERF-046) достижим при условии, что value в VLog читается одним последовательным чтением. Для values размером > 1 МБ read amp растёт до 3 из-за разбиения на блоки. Для таких workloads read amp публикуется отдельно.

**Freshness < 1 с (PERF-023).** Достижимо при Raft log replication в single-region. Multi-region freshness — v1.0, целевое значение публикуется отдельно. Обоснование: single-region RTT ≤ 1 мс, HTAP replica apply — секунды.

**Ops hours для solo.** 4 ч/мес — реалистично при условии, что релизы каждые 6–8 недель и нет S1/S2 инцидентов. При инцидентах — до 20 ч/мес. Обоснование: solo-разработчик тратит на инфраструктуру ~10% времени; остальное — разработка.

**Coverage на bench-пакетах.** Исключены из подсчёта. Обоснование: bench-пакеты измеряют, не содержат логики; их покрытие не имеет смысла.

**Compression для SSTable.** По умолчанию None для ProfileLatency (CPU-стоимость сжатия выше выигрыша на I/O для hot path). ZSTD для ProfileCompliance (ниже нагрузка на диск, меньше объём). Конфигурируется в Options.

---

## 14. Верификация

Каждое требование имеет источник верификации. Сводная таблица по категориям:

| Категория | Метод | Артефакт |
|---|---|---|
| Performance (latency, throughput) | Benchmark | BENCH-003, 004, 016 |
| Performance (allocs) | CI gate | BENCH-002 |
| Performance (read/write/space amp) | Benchmark | BENCH-005, 008 |
| Performance (VLog) | Benchmark | BENCH-006 |
| Performance (distributed) | Benchmark | BENCH-011, 014 |
| Scalability | Benchmark | BENCH-004, 014 |
| Availability (startup, recovery) | Harness | BENCH-007 |
| Availability (failover) | Jepsen-like | BENCH-013 |
| Durability | Crash-recovery harness | BENCH-007 |
| Durability (distributed) | Jepsen-like | BENCH-013 |
| Consistency | Property-based tests | `internal/txn/property_test.go` |
| Security (storage) | Fuzz, `govulncheck` | BENCH-007 |
| Security (network) | Integration tests | `internal/transport/security_test.go` |
| Security (encryption) | Unit + integration | `internal/crypto/` |
| Observability | Unit tests | `internal/metrics/` |
| Maintainability (code) | CI gates | `.github/workflows/` |
| Maintainability (format) | N-1 compatibility tests | `internal/format/compat_test.go` |
| Portability | CI matrix | `.github/workflows/portability.yml` |
| Developer experience | Godoc + examples | `docs/examples/` |
| Compliance | External audit | v1.0 |
| Cost | Benchmark + model | BENCH-010 |

---

## 15. Приёмка версий

### 15.1. v0.1 — Deterministic Foundation

MUST NFR, обязательные для приёмки:

- PERF-001..009, 013..016, 018
- PERF-026, 029..032
- PERF-038, 041..043
- PERF-045, 047, 051, 054
- AVAIL-001..004
- DUR-001..005
- CONS-001..004
- SEC-001..006
- OBS-001..006, 012..013, 015
- MAINT-001..002, 004..006, 008..018
- MAINT-024..026
- PORT-001, 006..017 (кроме PERF-016 ARM64 — SHOULD)
- UX-001..005, 008, 011, 013
- COST-005, 007

### 15.2. v0.2

Дополнительно:

- PERF-045 (read amp p99 < 3 подтверждён на реальных данных)
- PERF-049, 052, 053
- OBS-007

### 15.3. v0.3

Дополнительно:

- PERF-028, 046, 048, 050, 055..060
- SCALE-004, 005, 006
- PORT-003, 004
- UX-010

### 15.4. v0.4

Дополнительно:

- PERF-033
- SCALE-001, 002, 003
- CONS-007
- PORT-005

### 15.5. v0.5

Дополнительно:

- PERF-065
- AVAIL-012..014, 016
- DUR-007..010
- SEC-007, 019
- OBS-010, 011
- COMP-006
- COST-001, 002, 008, 009, 011, 012

### 15.6. v0.6a

Дополнительно:

- PERF-019..025, 034..037, 039, 040
- SCALE-008, 010..013
- AVAIL-005, 006, 008..011, 017
- DUR-001..005 (distributed), DUR-006 (расширено)
- CONS-005, 006, 008, 010
- SEC-008..012
- OBS-009, 014
- COST-003, 004, 010
- AVAIL-007

### 15.7. v0.6b

Дополнительно:

- SEC-013..017
- COMP-002

### 15.8. v0.7

Дополнительно:

- PERF-055 (для больших values)
- OBS-016
- SEC-020
- COMP-007

### 15.9. v1.0

Дополнительно:

- AVAIL-007
- CONS-009
- SEC-018, 021
- COMP-001, 003, 008, 011, 012
- SCALE-009

---

## 16. Что не покрыто и почему

**Latency на SATA SSD.** PERF-017 — допущение, не обязательство. SATA SSD физически медленнее NVMe на fsync (5–15 мс против 1–5 мс). Если пользователь работает на SATA, latency PUT (SYNC_MASTER) будет выше PERF-014; это документируется, не нарушает контракт.

**Zero-alloc на server mode.** Невозможно принципиально (ADR-030). Server mode допускает аллокации на framing (control plane), но не на message codec (data plane). PERF-039 фиксирует ≤ 2 allocs/op на codec, что достижимо.

**Zero-alloc на Raft path (v0.6a).** Dragonboat аллоцирует на Raft core (control plane). Это осознанное решение (D80), не нарушение. PERF-040 фиксирует честно: ≤ 10 allocs/op на distributed write.

**Zero-alloc на Metrics, Compact, Open, Close.** Эти операции относятся к control plane, не к hot path. Нет требований по аллокациям.

**Jepsen (Clojure).** Отклонён (CAN-007). Используется Knockbox (Go) — Jepsen-подобный harness на Go. BENCH-013 проверяет linearizability под partition, leader failover, clock skew.

**TLA+ refinement mapping.** Отклонён (CAN-008). Используются property-based тесты + модель в ADR-015. Этого достаточно для solo-проекта.

**Multi-region freshness.** v1.0. Целевое значение < 5 с (p99). Обоснование: cross-region RTT 50–150 мс, raft replication через регионы добавляет задержку.

**RBAC в v0.1–v0.5.** Не применимо (нет multi-user). RBAC — v1.0 (SEC-018).

**Multi-tenant isolation.** v1.0. Требует RBAC и resource quotas. Не входит в v0.1–v0.7.

**Работа на HDD.** Отклонено. Требования по latency (PERF-010..012) предполагают NVMe или SATA SSD. HDD даёт latency в десятки миллисекунд, что нарушает PERF-012 (p999 T4 < 50 мкс).

---

## 17. Ссылки

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v11.3 | Архитектура, позиционирование, Decision Log |
| TephraKV-PRD-001 v1.3 | ICP, use cases, TAM |
| TephraKV-ROADMAP-001 v4.0 | OKR, timeline, success metrics |
| TephraKV-BACKLOG-001 v6.1 | Задачи, включая BENCH |
| TephraKV-API-001 v1.3 | Public API |
| TephraKV-FORMAT-001 v1.3 | Data formats |
| TephraKV-GLOSSARY-001 v3.5 | Термины |
| ADR-005 v5 | Transport: встроенный TCP Dragonboat + gRPC |
| ADR-006 v5 | Raft core: Dragonboat |
| ADR-009 v2 | Zero-alloc scope |
| ADR-010 v3 | Durability semantics |
| ADR-012 v6 | Shared WAL pool |
| ADR-019 | Arena + epoch |
| ADR-020 | Metrics definitions |
| ADR-021 | Compaction policy |
| ADR-022 | VLog |
| ADR-026 v5 | Multi-Raft: Tan engine, встроенный TCP |
| ADR-029 v2 | Wire format: CodecV2 |
| ADR-034 v2.2 | Engineering practices |
| ADR-035 v2.4 | Capacity model |
| ADR-036 v1 | Read path: ReadIndex + Lease Read |

---

## 18. Что дальше

1. Утвердить NFR-001 v1.0 → статус `Approved`.
2. Проверить покрытие: каждое MUST NFR для v0.1 из §15.1 должно иметь задачу в BACKLOG v6.1.
3. Обновить BENCH-016 — ссылки на PERF-001..012.
4. Обновить BENCH-002, 003 — ссылки на PERF-038, 041..043.
5. Обновить BENCH-007 — ссылки на DUR-001..005, AVAIL-001..004.
6. Обновить BENCH-014 — ссылки на PERF-034..037, 040.
7. Создать checklist приёмки релиза — на основе §15.
8. Начать CHORE-001 — bootstrap.

Правило приёмки: если NFR не имеет источника верификации — он не подлежит приёмке. Если MUST NFR не покрыт задачей — релиз не принимается.

---

**Конец документа TephraKV-NFR-001 v1.0**