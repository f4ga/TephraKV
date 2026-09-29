# TephraKV — Glossary

**Документ:** TephraKV-GLOSSARY-001
**Версия:** 2.0
**Статус:** Active
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v7.0, TephraKV-PRD-001 v1.0, TephraKV-ROADMAP-001 v3.0, TephraKV-BACKLOG-001 v5.0, ADR-034, ADR-035
**Заменяет:** TephraKV-GLOSSARY-001 v1.1 (TephraDB)

---

## 0. Правила ведения

### 0.1. Что изменилось против v1.1

Синхронизация с HLD-000 v7.0, PRD-001 v1.0, ROADMAP-001 v3.0, BACKLOG-001 v5.0.

1. **Название:** TephraDB → TephraKV. Применено везде.
2. **Ссылки:** HLD-000 v5.1 → v7.0; ROADMAP v2.1 → v3.0; BACKLOG v4.1 → v5.0.
3. **Raft core:** etcd/raft для v0.6, свой Raft — v1.0+. Обновлены определения `Raft`, `Multi-Raft`, `Raft group`.
4. **Транспорт:** добавлены `CodecV2`, `SharedBufferPool`, `MaterializeToBuffer`. Обновлён `Message codec`.
5. **Профили:** добавлены `Profile Latency`, `Profile Compliance`, `Profile Custom` (HLD v7.0 §6.1).
6. **Модули:** добавлены `Audit log`, `Crypto (encryption at rest)`, `PITR`, `TTL`, `MVCC mode`.
7. **Новые термины:** `CodecV2`, `SharedBufferPool`, `Monotonic raw clock`, `Instant`, `SystemTime`, `Bandwidth-aware admission control`, `Titan-style WriteCallback`, `L0 partition`, `WiscKey update-heavy degradation`, `ReadIndex (clock-free)`, `Lease Read (bounded drift)`, `Two profiles one core`.
8. **Обновлены определения:** `Arena` (mmap-backed bump allocator, ~2.9 нс), `Hot path` (transport codec), `WAL` (shared pool с per-shard LSN namespace), `ReadIndex` (clock-free), `Lease Read` (monotonic raw clock), `Wire format` (CodecV2), `Durability policy` (5 политик + SYNC_LEADER).
9. **Удалены:** `CAN-005` (свой Raft для v0.6) — противоречие с D78.
10. **Добавлены BENCH:** BENCH-011..015.

### 0.2. Назначение

Глоссарий — **единственный источник правды о терминах** проекта. Если термин используется в HLD, PRD, ROADMAP, BACKLOG, ADR, коде, тестах, бенчмарках или коммитах — он должен быть здесь. Если термина нет — он либо не определён, либо определён в другом месте (что нарушает правило единого словаря).

### 0.3. Правила

1. **Один термин = одно определение.** Не «примерно так» — точно.
2. **Термины в алфавитном порядке** (латиница, затем кириллица).
3. **Каждый термин имеет категорию** (§0.4).
4. **Каждый термин имеет ссылку** на источник (ADR, HLD, BENCH).
5. **Перекрёстные ссылки** обязательны.
6. **Не дублировать определения из внешних источников.** Ссылка + краткое «как у нас».
7. **Обновляется при каждом новом ADR.**
8. **Язык:** русский с английскими терминами в скобках.
9. **Формат:** таблица или список. Единый.
10. **Версия и дата** — в шапке.

### 0.4. Категории

| Категория | Описание | Пример |
|---|---|---|
| `ARCH` | Архитектура | Bounded context, shard |
| `STORAGE` | Хранилище | LSM, MemTable, SSTable, WAL |
| `DIST` | Распределённые системы | Raft, gRPC, DRPC, ReadIndex |
| `CONSISTENCY` | Модели консистентности | Linearizability, SI |
| `PERF` | Производительность | Read amp, WA, p999, zero-alloc |
| `DURABILITY` | Долговечность | Durability policy, fsync |
| `OPS` | Эксплуатация | SLO, error budget, runbook |
| `PROCESS` | Процессы | ADR, DoR, DoD, SP |
| `DOMAIN` | Домен (DDD) | Ubiquitous language, bounded context |
| `BUSINESS` | Бизнес | TAM, ARR, acqui-hire |
| `SECURITY` | Безопасность | Threat model, failure model |

### 0.5. Формат термина

```markdown
### <Термин>

- **Категория:** <категория>
- **Определение:** <1–3 предложения>
- **Контекст TephraKV:** <как применяется у нас>
- **Ссылки:** <ADR-NNN, HLD §X.Y, BENCH-NNN>
- **Синонимы:** <если есть>
- **Антонимы:** <если есть>
- **Связанные:** <другие термины>
```

---

## 1. Термины (алфавитный порядок)

### A

#### Acquired (Acqui-hire)

- **Категория:** BUSINESS
- **Определение:** Покупка компании ради команды, а не ради продукта.
- **Контекст TephraKV:** Реалистичный сценарий выхода для solo-проекта — не $15B valuation, а покупка командой или технологией.
- **Ссылки:** HLD-000 v7.0 §1, PRD-001 v1.0 §12.
- **Синонимы:** team acquisition.
- **Связанные:** Open-core, Design partner.

---

#### ADR (Architecture Decision Record)

- **Категория:** PROCESS
- **Определение:** Документ, фиксирующий одно архитектурное решение: контекст, решение, альтернативы, последствия.
- **Контекст TephraKV:** ADR пишется **до** кода. Один ADR = одно решение. ADR immutable: передумал — новый ADR, старый `Superseded by ADR-XXX`. Внешний критик — open-source feedback (замена внутреннему ревью в solo-контексте).
- **Ссылки:** ADR-034, ENG-001 §1, HLD-000 v7.0 §10.
- **Синонимы:** —
- **Связанные:** Decision Log, Fitness function.

---

#### Arena

- **Категория:** STORAGE / PERF
- **Определение:** Bump allocator с фиксированными slabs, из которого аллоцируются объекты без `runtime.mallocgc`. Память освобождается через epoch-based reclamation.
- **Контекст TephraKV:** Фундамент zero-alloc storage. mmap-backed bump allocator: аллокация 64 Б за ~2.9 нс (vs ~40 нс для heap). Skiplist nodes, WAL буферы, Raft log, transport message buffers — всё аллоцируется из arena. `sync.Pool` **запрещён** в data plane.
- **Ссылки:** ADR-019, HLD-000 v7.0 §3.2, §5.4, D85.
- **Синонимы:** bump allocator, region-based allocator.
- **Антонимы:** `sync.Pool`, per-object malloc.
- **Связанные:** Epoch reclamation, Zero-alloc.

---

#### Audit log

- **Категория:** SECURITY / STORAGE
- **Определение:** Append-only, immutable лог операций для compliance.
- **Контекст TephraKV:** Модуль (v0.5). Не часть основного KV. Отдельный namespace в общем WAL. Export в S3/Glacier (v0.7). Для ниши 2 (fintech/audit).
- **Ссылки:** HLD-000 v7.0 §5.3, §6.6, D73.
- **Синонимы:** —
- **Связанные:** PITR, Compliance, Profile Compliance.

---

### B

#### Backpressure

- **Категория:** DIST / PERF
- **Определение:** Механизм, при котором перегруженный потребитель сигнализирует производителю о необходимости снизить темп.
- **Контекст TephraKV:** Явный backpressure между Raft apply и gRPC flow control (v0.6); между frozen MemTable и flush worker. Не drop, не unbounded queue.
- **Ссылки:** ADR-008, HLD-000 v7.0 §3.7, §4.2.
- **Синонимы:** flow control.
- **Связанные:** gRPC, Raft, Frozen MemTable.

---

#### Bandwidth-aware admission control

- **Категория:** PERF / OPS
- **Определение:** Механизм, при котором система отказывает в новых запросах, если пропускная способность I/O исчерпана, вместо накопления очереди.
- **Контекст TephraKV:** SLA-driven compaction scheduler (v0.2). RaKV-style: предотвращение write stalls при p99 > 2. Token bucket + cooldown 60 сек + гистерезис (старт при p99 > 2, стоп при p99 < 1.5).
- **Ссылки:** ADR-021, HLD-000 v7.0 §6.4, D90.
- **Синонимы:** admission control.
- **Связанные:** Compaction, Backpressure, SLA-driven scheduler.

---

#### BENCH (Benchmark)

- **Категория:** PERF / PROCESS
- **Определение:** Документ с методологией измерения и результатами. Каждый BENCH фиксирует hardware, Go version, commit, дату.
- **Контекст TephraKV:** BENCH-001..015. Публикуются в `docs/bench/`. Двухуровневые: внешние + внутренние метрики.
- **Ссылки:** ENG-001 §9, HLD-000 v7.0 §8, ROADMAP-001 v3.0 §11.
- **Синонимы:** —
- **Связанные:** Two-Level Metrics, Baseline.

---

#### Bloom filter

- **Категория:** STORAGE
- **Определение:** Вероятностная структура данных для проверки принадлежности элемента множеству. Может давать ложноположительные срабатывания, но не ложноотрицательные.
- **Контекст TephraKV:** 9.6 bits/key (эталон ScyllaDB), false positive rate < 1%. Per-SSTable. Память: при 100M keys × 7 levels = **840 МБ** (не 8.4 ГБ — исправление арифметической ошибки v5.1).
- **Ссылки:** ADR-003, HLD-000 v7.0 §3.5, §5.4.
- **Синонимы:** —
- **Связанные:** SSTable, Partition index, Read amplification.

---

#### Bounded context

- **Категория:** ARCH / DOMAIN
- **Определение:** Граница, внутри которой определённая доменная модель имеет единый смысл.
- **Контекст TephraKV:** Модули одного бинарника: `internal/arena`, `internal/memtable`, `internal/wal`, `internal/lsm`, `internal/raft`, `internal/transport`, `internal/profile`, `internal/audit`, `internal/crypto`, `internal/pitr`, `internal/ttl`, `internal/mvcc`, `internal/vlog`. Запрещены циклы между модулями (go-arch-lint).
- **Ссылки:** HLD-000 v7.0 §5.3, ENG-001 §4, ADR-034.
- **Синонимы:** модуль, компонент.
- **Связанные:** Ubiquitous language, Fitness function.

---

#### Bump allocator

- **Категория:** STORAGE / PERF
- **Определение:** Аллокатор, который выделяет память простым сдвигом указателя вперёд. Не поддерживает free per-object.
- **Контекст TephraKV:** Основа arena. mmap-backed. Free — через `Reset()` или epoch reclamation.
- **Ссылки:** ADR-019, D85.
- **Синонимы:** linear allocator, region allocator.
- **Связанные:** Arena, Epoch reclamation.

---

### C

#### Capacity model

- **Категория:** OPS / PERF
- **Определение:** Документ, фиксирующий: сколько RAM/CPU/Disk нужно на N ключей и M операций; где bottleneck при росте.
- **Контекст TephraKV:** ADR-035. Включает Bloom-память, partition index, VLog space, WAL shared pool. Differentiation budget: где тратим (partition index, arena), где экономим (Redis-like RAM).
- **Ссылки:** ADR-035, HLD-000 v7.0 §4.6, ROADMAP-001 v3.0 §8.
- **Синонимы:** capacity plan.
- **Связанные:** TCO, SLO.

---

#### CHANGELOG

- **Категория:** PROCESS
- **Определение:** Файл, фиксирующий изменения по каждому релизу: Added, Changed, Fixed, Removed, Deprecated, Security, Benchmarks.
- **Контекст TephraKV:** Обновляется по каждому PR. Формат — conventional commits.
- **Ссылки:** ENG-001 §5.3, ADR-034.
- **Синонимы:** —
- **Связанные:** Release notes, Semver.

---

#### Clock skew

- **Категория:** DIST / CONSISTENCY
- **Определение:** Разница между локальными часами узлов.
- **Контекст TephraKV:** ReadIndex — clock-free (устойчив к skew). Lease Read требует bounded clock drift < 1000 ppm и monotonic raw clock. Election timeout 10 сек, lease validity 9 сек. Randomized election timeout + skew < timeout/10.
- **Ссылки:** ADR-036, HLD-000 v7.0 §6.7, D64, D88.
- **Синонимы:** —
- **Связанные:** ReadIndex, Lease Read, Monotonic raw clock.

---

#### CodecV2

- **Категория:** DIST / PERF
- **Определение:** API gRPC Go (grpc-go 1.66+) для codec, который позволяет marshal/unmarshal напрямую в shared buffer pool без аллокаций.
- **Контекст TephraKV:** Обязателен для zero-alloc transport codec. V1 bridge аллоцирует `BufferSlice.Materialize()` на каждый message. CodecV2 + SharedBufferPool: 2.4× быстрее Unmarshal, 2.7× быстрее Marshal, ~300× меньше аллокаций.
- **Ссылки:** ADR-029 v2, ADR-005 v4, HLD-000 v7.0 §6.8, D86, D70.
- **Синонимы:** grpc.CodecV2.
- **Антонимы:** Codec V1 (bridge).
- **Связанные:** SharedBufferPool, Message codec, gRPC, Zero-alloc.

---

#### Compaction

- **Категория:** STORAGE
- **Определение:** Процесс слияния SSTable разных уровней LSM для удаления дубликатов и поддержания read amplification.
- **Контекст TephraKV:** SLA-driven policy: триггеры L0 = 4 SSTable, L1 = 256 МБ, read amp p99 > 2. Cooldown 60 сек, гистерезис (старт при p99 > 2, стоп при p99 < 1.5). Bandwidth-aware admission control. RaKV-style dynamically partitioned L0.
- **Ссылки:** ADR-021, HLD-000 v7.0 §6.4, D89, D90.
- **Синонимы:** merge.
- **Связанные:** LSM, Leveled compaction, Read amplification.

---

#### Compliance profile

- **Категория:** ARCH / BUSINESS
- **Определение:** Профиль конфигурации TephraKV для fintech/audit (ниша 2). Включает MVCC, PITR, audit log, encryption at rest.
- **Контекст TephraKV:** `ProfileCompliance`. Default: SYNC_MASTER, MVCC on, PITR on, audit on, encryption on, compression ZSTD, read amp target < 3. См. `ProfileLatency` для ниши 1.
- **Ссылки:** HLD-000 v7.0 §6.1, D71, D73–D77.
- **Синонимы:** —
- **Антонимы:** Profile Latency.
- **Связанные:** Profile Latency, Profile Custom, Audit log, PITR.

---

#### Consistency model

- **Категория:** CONSISTENCY
- **Определение:** Контракт между системой хранения и клиентами о том, какие порядки и видимость операций допустимы.
- **Контекст TephraKV:** v0.1–v0.5: single-node, linearizable для single-key. v0.4: snapshot isolation для транзакций. v0.6: ReadIndex (clock-free) для distributed linearizable read; Lease Read (опционально, bounded drift).
- **Ссылки:** ADR-011, ADR-017, ADR-036, HLD-000 v7.0 §3.7.
- **Синонимы:** —
- **Связанные:** Linearizability, Snapshot Isolation, ReadIndex, Lease Read.

---

### D

#### Decision Log

- **Категория:** PROCESS
- **Определение:** Реестр всех ADR с датами, статусами и ссылками.
- **Контекст TephraKV:** Хранится в `docs/DECISIONS.md`. Обновляется при каждом новом ADR. HLD v7.0 §10 содержит D71–D90 (v7.0 дополнения).
- **Ссылки:** ENG-001 §1.4, HLD-000 v7.0 §10.
- **Синонимы:** —
- **Связанные:** ADR.

---

#### Definition of Done (DoD)

- **Категория:** PROCESS
- **Определение:** Чек-лист, который задача должна пройти, чтобы считаться завершённой.
- **Контекст TephraKV:** Код написан, тесты проходят, бенчмарк не деградировал, ADR обновлён, CHANGELOG обновлён, escape analysis чист, allocs/op == 0.
- **Ссылки:** ADR-034, ENG-001 §3, BACKLOG-001 v5.0 §0.9.
- **Синонимы:** —
- **Связанные:** Definition of Ready.

---

#### Definition of Ready (DoR)

- **Категория:** PROCESS
- **Определение:** Чек-лист, который задача должна пройти, чтобы её можно было взять в работу.
- **Контекст TephraKV:** Проблема сформулирована, acceptance criteria записаны, ADR написан, BENCH определён, zero-alloc оценено, зависимости проверены.
- **Ссылки:** ADR-034, ENG-001 §2, BACKLOG-001 v5.0 §0.8.
- **Синонимы:** —
- **Связанные:** Definition of Done.

---

#### Design partner

- **Категория:** BUSINESS
- **Определение:** Компания, которая использует продукт на ранней стадии и даёт обратную связь в обмен на влияние на roadmap.
- **Контекст TephraKV:** Планируется до v0.5. Без имён — надежда, не план. Нужны LOI (Letter of Intent).
- **Ссылки:** PRD-001 v1.0 §10, HLD-000 v7.0 §3.3.
- **Синонимы:** early adopter.
- **Связанные:** Path to revenue, Open-core.

---

#### DRPC

- **Категория:** DIST
- **Определение:** Lightweight RPC-фреймворк, drop-in замена gRPC с меньшим числом аллокаций и большей пропускной способностью на коротких сообщениях.
- **Контекст TephraKV:** Рассматривается как транспорт v0.7+, если gRPC bottleneck. CockroachDB зафиксировал 12% улучшение QPS. Zero-alloc граница: framing — control plane, codec — hot path.
- **Ссылки:** ADR-005 v4, HLD-000 v7.0 §4.7, §6.8, D38.
- **Синонимы:** Storj DRPC.
- **Связанные:** gRPC, TCP, Transport.

---

#### Durability policy

- **Категория:** DURABILITY
- **Определение:** Политика, определяющая, когда операция записи считается подтверждённой и какие гарантии сохранности даёт.
- **Контекст TephraKV:** Per-operation: NO_SYNC, SYNC_MASTER, SYNC_LEADER (v0.1); +SYNC_MAJORITY, +SYNC_ALL (v0.6). SYNC_MASTER в distributed ≠ distributed durability. SYNC_LEADER — явная семантика fast-path без distributed durability. Mixed batch запрещён.
- **Ссылки:** ADR-010, HLD-000 v7.0 §4.4, §6.2, D4–D6, D54, D61.
- **Синонимы:** sync mode.
- **Связанные:** fsync, Group commit, WAL, SYNC_LEADER.

---

### E

#### Encryption at rest

- **Категория:** SECURITY / STORAGE
- **Определение:** Шифрование данных на диске (AES-256-GCM).
- **Контекст TephraKV:** Модуль (v0.6). In-place encrypt на hot path (zero-alloc). Не отдельный слой, а часть WAL/SSTable. KMS integration. Для ниши 2.
- **Ссылки:** HLD-000 v7.0 §5.3, §6.6, D74.
- **Синонимы:** disk encryption.
- **Связанные:** WAL, SSTable, Compliance profile.

---

#### Epoch reclamation

- **Категория:** STORAGE / PERF
- **Определение:** Механизм освобождения памяти, при котором объект удаляется только после того, как все активные epoch-и прошли.
- **Контекст TephraKV:** Epoch before free. Используется в arena и skiplist iterator.
- **Ссылки:** ADR-019, HLD-000 v7.0 §5.1, §5.4 (принцип 4).
- **Синонимы:** epoch-based reclamation.
- **Связанные:** Arena, Lock-free, Skiplist.

---

#### Error budget

- **Категория:** OPS
- **Определение:** Допустимое количество нарушений SLO за окно (например, 28 дней). Когда бюджет сгорел — новые фичи замораживаются, только надёжность.
- **Контекст TephraKV:** 0.1% нарушений SLO за 28 дней. Для solo — самодисциплина.
- **Ссылки:** ADR-034, ENG-001 §7.3.
- **Синонимы:** —
- **Связанные:** SLO, SLI.

---

#### Escape analysis

- **Категория:** PERF
- **Определение:** Анализ компилятора Go, определяющий, может ли объект быть размещён на стеке вместо кучи.
- **Контекст TephraKV:** Проверяется в CI (`-gcflags=-m`). Провал при новом heap escape на hot path. **Но контракт zero-alloc гарантируется CI-бенчмарком с `-benchmem`, не escape analysis** — escape analysis не стабилен между версиями Go (D69).
- **Ссылки:** ADR-009 v2, ENG-001 §5.1, HLD-000 v7.0 §3.2, D69.
- **Синонимы:** —
- **Связанные:** Zero-alloc, Heap escape, CI-бенчмарк.

---

### F

#### Failure model

- **Категория:** SECURITY / OPS
- **Определение:** Документ, фиксирующий: что считается сбоем, какие гарантии даёт система для каждого типа сбоя, что вне scope.
- **Контекст TephraKV:** `docs/security/failure-model-v0.1.md`. 9 сбоев: process crash, OOM, disk full, partial write, bit rot, filesystem corruption, clock skew, network partition, leader crash. HLD v7.0 §9.3.
- **Ссылки:** HLD-000 v7.0 §9.3, CHORE-007 (BACKLOG).
- **Синонимы:** crash model.
- **Связанные:** Threat model, Recovery, Durability policy.

---

#### False positive

- **Категория:** STORAGE
- **Определение:** Ложное срабатывание Bloom filter: фильтр говорит «возможно, есть», хотя элемента нет.
- **Контекст TephraKV:** < 1% при 9.6 bits/key. Измеряется в BENCH-008.
- **Ссылки:** ADR-003, HLD-000 v7.0 §3.5.
- **Синонимы:** false positive rate (FPR).
- **Связанные:** Bloom filter, Read amplification.

---

#### Fitness function

- **Категория:** ARCH / PROCESS
- **Определение:** Автоматическая проверка, что архитектура не деградирует. Например, запрет циклов между модулями.
- **Контекст TephraKV:** `go-arch-lint`, zero-alloc gate, escape analysis, benchmark regression. **Единственный архитектурный критик для solo-проекта.** Если гейт не автоматизирован — он не работает.
- **Ссылки:** ADR-034, ENG-001 §6, CHORE-006 (BACKLOG).
- **Синонимы:** architecture test.
- **Связанные:** Bounded context, CI-гейт.

---

#### Framing

- **Категория:** DIST / PERF
- **Определение:** Разбиение потока байт на сообщения (frames) на уровне транспортного протокола. В gRPC — HTTP/2 frames.
- **Контекст TephraKV:** gRPC framing — **control plane**, аллокации допустимы. Отделён от message codec (payload), который на hot path. Граница зафиксирована в ADR-009 v2. `sync.Pool` допустим в framing (control plane).
- **Ссылки:** ADR-009 v2, ADR-005 v4, HLD-000 v7.0 §3.2, §6.8, D55, D68.
- **Синонимы:** HTTP/2 framing.
- **Связанные:** Message codec, gRPC, Zero-alloc, Transport.

---

#### fsync

- **Категория:** DURABILITY
- **Определение:** Системный вызов, гарантирующий запись данных и метаданных файла на диск.
- **Контекст TephraKV:** Для append-only WAL нужен `fsync` (не `fdatasync`), потому что размер файла меняется. WAL preallocated (1 ГБ сегменты), поэтому metadata journal не триггерится.
- **Ссылки:** ADR-002, ADR-010, HLD-000 v7.0 §6.2.
- **Синонимы:** —
- **Связанные:** Durability policy, Group commit, WAL.

---

#### Frozen MemTable

- **Категория:** STORAGE
- **Определение:** MemTable, который достиг лимита и заморожен для записи в SSTable. Новые записи идут в активный MemTable.
- **Контекст TephraKV:** Пул frozen MemTable + single-threaded flush worker. Writers не блокируются.
- **Ссылки:** FEAT-002, HLD-000 v7.0 §5.4.
- **Синонимы:** immutable memtable.
- **Связанные:** MemTable, SSTable, Flush.

---

### G

#### gRPC

- **Категория:** DIST
- **Определение:** RPC-фреймворк на базе HTTP/2 с codegen, multiplexing streams, deadlines/cancellation.
- **Контекст TephraKV:** Транспорт v0.6 для intra-cluster Raft. Проверен в production: TiKV, etcd, YugabyteDB, Vitess. Framing — control plane, codec — hot path. **CodecV2 + SharedBufferPool** обязательны для zero-alloc.
- **Ссылки:** ADR-005 v4, HLD-000 v7.0 §4.7, §6.8, D31, D86.
- **Синонимы:** Google RPC.
- **Связанные:** DRPC, TCP, Transport, Framing, Message codec, CodecV2.

---

#### Group commit

- **Категория:** DURABILITY / PERF
- **Определение:** Батчинг нескольких записей в один fsync для амортизации стоимости.
- **Контекст TephraKV:** Окно 100 мкс или до 64 записей. Mixed batch: SYNC_MASTER и SYNC_LEADER ждут fsync, NO_SYNC — нет.
- **Ссылки:** ADR-002, FEAT-003.
- **Синонимы:** batched fsync.
- **Связанные:** WAL, fsync, Durability policy.

---

### H

#### Hot path

- **Категория:** PERF
- **Определение:** Код, выполняемый на каждой операции: Put, Get, Delete, Scan, WriteBatch, raft apply, transport message codec.
- **Контекст TephraKV:** Zero-alloc контракт. Запрещены `sync.Pool`, `fmt`, `reflect`, `interface{}`, замыкания. Transport framing — control plane. **Кастомный gRPC codec (CodecV2), не protobuf.**
- **Ссылки:** ADR-009 v2, HLD-000 v7.0 §3.2, §6.8, ENG-001 §4.2, D3, D68, D70.
- **Синонимы:** data plane.
- **Антонимы:** Control plane.
- **Связанные:** Zero-alloc, Escape analysis, Message codec, CodecV2.

---

### I

#### Incident

- **Категория:** OPS
- **Определение:** Событие, классифицируемое по severity (S1–S4), с постмортемом для S1/S2.
- **Контекст TephraKV:** S1 — потеря данных; S2 — падение, SLO нарушен. Постмортем в `docs/incidents/`.
- **Ссылки:** ADR-034, ENG-001 §13.
- **Синонимы:** —
- **Связанные:** SLO, Error budget, Runbook.

---

#### Instant (monotonic raw clock)

- **Категория:** DIST / CONSISTENCY
- **Определение:** Монотонные часы Go (`time.Instant`), не подверженные wall-clock drift и NTP step.
- **Контекст TephraKV:** Lease Read требует `Instant`, не `SystemTime`. Wall-clock drift breaks linearizability. Fix Raft lease clock: SystemTime → Instant for monotonic lease validity checks.
- **Ссылки:** ADR-036, HLD-000 v7.0 §6.7, D88.
- **Синонимы:** monotonic clock.
- **Антонимы:** SystemTime (wall-clock).
- **Связанные:** Lease Read, Clock skew, ReadIndex.

---

### L

#### L0 partition (dynamically partitioned)

- **Категория:** STORAGE
- **Определение:** Динамическое партиционирование L0 in-memory для снижения write stalls.
- **Контекст TephraKV:** RaKV-style. Parallel L0-L1 compaction mechanism для минимизации write amplification и write stalls. Bandwidth-aware admission control. HATS co-schedules read and compaction tasks, снижая P99 latencies на 58.6–59.9% в YCSB read-dominant workloads.
- **Ссылки:** ADR-021, HLD-000 v7.0 §6.4, D89.
- **Синонимы:** L0 partitioning.
- **Связанные:** Compaction, Leveled compaction, SLA-driven scheduler.

---

#### Lease Read

- **Категория:** CONSISTENCY / DIST
- **Определение:** Механизм linearizable read в Raft, использующий leader lease вместо heartbeat round.
- **Контекст TephraKV:** Опционально (не default). Требует bounded clock drift < 1000 ppm и monotonic raw clock (Instant, не SystemTime). Election timeout 10 сек, lease validity 9 сек. Если stale leader serves lease read past expiry due to clock skew — linearizability breaks. ReadIndex в этом случае не safe.
- **Ссылки:** ADR-036, HLD-000 v7.0 §6.7, D64, D88.
- **Синонимы:** lease-based read.
- **Антонимы:** ReadIndex.
- **Связанные:** Raft, Linearizability, Clock skew, Instant.

---

#### Leveled compaction

- **Категория:** STORAGE
- **Определение:** Стратегия компакции, при которой каждый уровень L1+ имеет ограниченный размер и переполнение вызывает merge в следующий уровень.
- **Контекст TephraKV:** L0 — tiered (≤ 4 SSTable), L1+ — leveled. Целевой read amp p99 < 3 (v0.2), < 2 (v0.3+).
- **Ссылки:** ADR-021, HLD-000 v7.0 §5.4.
- **Синонимы:** —
- **Связанные:** LSM, Compaction, Read amplification.

---

#### Linearizability

- **Категория:** CONSISTENCY
- **Определение:** Модель консистентности, при которой каждая операция выглядит атомарной и происходит в момент между вызовом и возвратом.
- **Контекст TephraKV:** v0.1–v0.5: single-node linearizable. v0.6: distributed linearizable через ReadIndex (clock-free).
- **Ссылки:** ADR-011, HLD-000 v7.0 §3.7.
- **Синонимы:** strict consistency, atomic consistency.
- **Связанные:** ReadIndex, Lease Read, Snapshot Isolation.

---

#### LSM (Log-Structured Merge tree)

- **Категория:** STORAGE
- **Определение:** Структура хранения, при которой записи буферизуются в MemTable, затем сбрасываются в SSTable, которые периодически сливаются (compaction).
- **Контекст TephraKV:** Read-amp-minimized LSM: L0 tiered, L1+ leveled с partition index. Bloom 9.6 bits/key. VLog (WiscKey) для values > threshold (v0.3+).
- **Ссылки:** HLD-000 v7.0 §5.1, §6.3.
- **Синонимы:** LSM-tree.
- **Связанные:** MemTable, SSTable, Compaction, VLog.

---

#### LSN (Log Sequence Number)

- **Категория:** STORAGE / DURABILITY
- **Определение:** Уникальный монотонно возрастающий номер записи в WAL. Используется для упорядочивания и recovery.
- **Контекст TephraKV:** LSN per op, монотонен внутри shard. В distributed — LSN per shard, между shard не координируется. Shared WAL pool с per-shard LSN namespace: O(1) по файлам, не O(N_shards).
- **Ссылки:** ADR-002, ADR-012 v3, HLD-000 v7.0 §5.1, §6.7, D53, D62.
- **Синонимы:** —
- **Связанные:** WAL, Group commit, Recovery, Shard, Shared WAL pool.

---

### M

#### Manifest

- **Категория:** STORAGE
- **Определение:** Append-only файл, фиксирующий метаданные LSM: какие SSTable существуют, на каких уровнях, границы ключей.
- **Контекст TephraKV:** Append-only + snapshot compaction. Atomic swap при компакции.
- **Ссылки:** HLD-000 v7.0 §5.1.
- **Синонимы:** —
- **Связанные:** SSTable, Compaction, Recovery.

---

#### MaterializeToBuffer

- **Категория:** DIST / PERF
- **Определение:** Метод CodecV2 в grpc-go, который в common single-buffer case только берёт reference на transport buffer, без копирования.
- **Контекст TephraKV:** Ключевой для zero-alloc decode. В V1 bridge — unconditional allocate-and-copy. В V2 — no copy.
- **Ссылки:** ADR-029 v2, HLD-000 v7.0 §6.8, D86.
- **Синонимы:** —
- **Антонимы:** BufferSlice.Materialize (V1 bridge).
- **Связанные:** CodecV2, SharedBufferPool, Message codec.

---

#### MemTable

- **Категория:** STORAGE
- **Определение:** In-memory структура для свежих записей до сброса в SSTable.
- **Контекст TephraKV:** Lock-free skiplist на offsets в arena. Single-version (MVCC версии — в SST и VLog). Режим MVCC — single-version vs multi-version (D72).
- **Ссылки:** ADR-001, ADR-017, HLD-000 v7.0 §5.4, D72.
- **Синонимы:** active memtable.
- **Связанные:** Skiplist, Arena, Frozen MemTable.

---

#### Message codec

- **Категория:** DIST / PERF
- **Определение:** Сериализация/десериализация payload сообщения (не framing). В TephraKV — свой binary wire format внутри gRPC message.
- **Контекст TephraKV:** **Hot path** (raft apply). Zero-alloc через собственный пул буферов. **Кастомный gRPC codec, CodecV2 + SharedBufferPool, не protobuf.** Граница с framing зафиксирована в ADR-009 v2.
- **Ссылки:** ADR-009 v2, ADR-029 v2, HLD-000 v7.0 §3.2, §6.8, §6.9, D39, D55, D70, D86.
- **Синонимы:** payload serialization.
- **Связанные:** Framing, Wire format, Zero-alloc, gRPC, CodecV2, SharedBufferPool.

---

#### MinMax block index

- **Категория:** STORAGE
- **Определение:** Индекс, хранящий min и max ключ для каждой партиции, ускоряющий scan с фильтром.
- **Контекст TephraKV:** Per partition, не per block. С v0.2.
- **Ссылки:** ADR-003, HLD-000 v7.0 §5.4, FEAT-009.
- **Синонимы:** —
- **Связанные:** Partition index, SSTable, Scan.

---

#### Monotonic raw clock

- **Категория:** DIST / CONSISTENCY
- **Определение:** См. `Instant (monotonic raw clock)`.

---

#### Multi-Raft

- **Категория:** DIST
- **Определение:** Архитектура, при которой каждый shard имеет свою Raft group. TiKV-style.
- **Контекст TephraKV:** 10000 Raft groups (по числу shards). Rebalance — перемещение лидерства, не изменение числа групп. 1 shard = 1 Raft group = 1 WAL namespace. **Raft core — etcd/raft для v0.6**, свой Raft — v1.0+ (если команда 5+).
- **Ссылки:** ADR-006 v2, ADR-026 v3, HLD-000 v7.0 §4.7, §5.2, §6.7, D32, D48, D56, D78, D79.
- **Синонимы:** —
- **Связанные:** Raft, Raft group, Shard, Snapshot transfer, etcd/raft.

---

#### MVCC (Multi-Version Concurrency Control)

- **Категория:** STORAGE / CONSISTENCY
- **Определение:** Механизм, при котором каждая версия данных сохраняется, позволяя читателям видеть консистентный snapshot.
- **Контекст TephraKV:** Режим MemTable (не модуль, D72). Single-version MemTable, версии — в SST и VLog. Snapshot isolation. Global watermark = min(active snapshot). Модуль v0.4.
- **Ссылки:** ADR-017, HLD-000 v7.0 §5.3, §6.6, D72.
- **Синонимы:** —
- **Связанные:** Snapshot Isolation, GC версий, Timestamp, Watermark.

---

### N

#### NO_SYNC

- **Категория:** DURABILITY
- **Определение:** Политика, при которой запись подтверждается после записи в WAL-буфер, fsync — в окне group commit.
- **Контекст TephraKV:** Потеря последних N мс при сбое. v0.1.
- **Ссылки:** ADR-010, HLD-000 v7.0 §4.4, §6.2.
- **Синонимы:** async write.
- **Антонимы:** SYNC_MASTER, SYNC_LEADER, SYNC_MAJORITY, SYNC_ALL.
- **Связанные:** Durability policy, WAL, Group commit.

---

### O

#### Open-core

- **Категория:** BUSINESS
- **Определение:** Бизнес-модель, при которой ядро продукта open-source, а enterprise-фичи — платные.
- **Контекст TephraKV:** Границы OSS/enterprise — ADR-004 (Proposed). Реалистичный путь к revenue. Community edition (Apache 2.0) — ядро v0.1–v0.5. Enterprise — audit, encryption, RBAC, SOC2, multi-region, support SLA.
- **Ссылки:** ADR-004, PRD-001 v1.0 §9, HLD-000 v7.0 §3.1.
- **Синонимы:** —
- **Связанные:** Design partner, Path to revenue.

---

### P

#### p99 / p999

- **Категория:** PERF
- **Определение:** 99-й и 99.9-й процентили распределения задержек.
- **Контекст TephraKV:** Метрика первого класса. p999 GET < 5 мс (in-memory, single-node). Distributed p99 — миллисекунды.
- **Ссылки:** HLD-000 v7.0 §2.2, §4.1, BENCH-003.
- **Синонимы:** tail latency.
- **Связанные:** SLO, Two-Level Metrics.

---

#### Partition index

- **Категория:** STORAGE
- **Определение:** Разреженный индекс в памяти, хранящий min/max ключ для каждой партиции SSTable.
- **Контекст TephraKV:** Каждые 1000 блоков. 1M записей × 16 Б = 16 МБ при 1B blocks. С v0.2.
- **Ссылки:** ADR-003, HLD-000 v7.0 §3.5, §5.4.
- **Синонимы:** —
- **Связанные:** MinMax block index, SSTable, Read amplification.

---

#### Path to revenue

- **Категория:** BUSINESS
- **Определение:** План достижения устойчивого дохода.
- **Контекст TephraKV:** Open-source + enterprise support + acqui-hire. Не $15B valuation. Design partners ($10–30K/год) → Standard ($50–100K/год) → Premium ($150–300K/год) → Enterprise ($300–500K/год).
- **Ссылки:** ADR-032, PRD-001 v1.0 §9, HLD-000 v7.0 §1.
- **Синонимы:** monetization.
- **Связанные:** Design partner, Open-core, TAM.

---

#### PITR (Point-in-Time Recovery)

- **Категория:** STORAGE / DURABILITY
- **Определение:** Восстановление состояния БД на произвольный момент в прошлом.
- **Контекст TephraKV:** Модуль (v0.5). Требует MVCC. WAL archive + replay API. Для ниши 2 (fintech/audit).
- **Ссылки:** HLD-000 v7.0 §5.3, §6.6, D75.
- **Синонимы:** —
- **Связанные:** MVCC, Audit log, Compliance profile.

---

#### Profile Custom

- **Категория:** ARCH
- **Определение:** Профиль конфигурации TephraKV для ручной настройки.
- **Контекст TephraKV:** `ProfileCustom`. Пользователь задаёт все параметры вручную. Не использует дефолты Latency/Compliance.
- **Ссылки:** HLD-000 v7.0 §6.1, D77.
- **Синонимы:** —
- **Связанные:** Profile Latency, Profile Compliance.

---

#### Profile Latency

- **Категория:** ARCH / BUSINESS
- **Определение:** Профиль конфигурации TephraKV для latency-critical приложений (ниша 1). Оптимизирован под p999 и zero-alloc.
- **Контекст TephraKV:** `ProfileLatency`. Default: NO_SYNC, TTL on, MVCC off, compression none, read amp target < 2. Для HFT, game servers, AI-infra, edge, CDN.
- **Ссылки:** HLD-000 v7.0 §6.1, D71, D76.
- **Синонимы:** —
- **Антонимы:** Profile Compliance.
- **Связанные:** Profile Compliance, Profile Custom, TTL.

---

### Q

#### QUIC

- **Категория:** DIST
- **Определение:** Транспортный протокол поверх UDP с встроенным TLS 1.3, multiplexing без head-of-line blocking, connection migration.
- **Контекст TephraKV:** Только cross-region (v1.0+), **не intra-cluster**. Причины отказа от intra-cluster: 66% throughput от TCP+TLS, 12× RAM, UDP блокируется в enterprise-сетях, ни одна production СУБД не использует QUIC для inter-node Raft. DEF-007.
- **Ссылки:** ADR-005 v4, HLD-000 v7.0 §4.7, D51, DEF-007.
- **Синонимы:** —
- **Связанные:** gRPC, DRPC, TCP, Transport.

---

### R

#### Raft

- **Категория:** DIST
- **Определение:** Протокол консенсуса для реплицированного лога. Обеспечивает leader election, log replication, safety.
- **Контекст TephraKV:** **etcd/raft для v0.6** (проверен Jepsen, 10 лет production, 10K+ stars). Свой Raft — v1.0+ (если команда 5+). ReadIndex (default) + Lease Read (опционально). Static membership до v1.0. Learner-based rolling replacement с v0.6.
- **Ссылки:** ADR-006 v2, ADR-011, ADR-013, ADR-015 v2, HLD-000 v7.0 §4.7, §6.7, D32, D78, D79.
- **Синонимы:** —
- **Связанные:** Multi-Raft, Raft group, ReadIndex, Lease Read, Snapshot transfer, etcd/raft.

---

#### Raft group

- **Категория:** DIST
- **Определение:** Один экземпляр Raft-консенсуса, обслуживающий один shard.
- **Контекст TephraKV:** 1 shard = 1 Raft group = 1 WAL namespace. 10000 Raft groups на кластер. Лидер группы обрабатывает записи в свой shard.
- **Ссылки:** ADR-026 v3, ADR-012 v3, HLD-000 v7.0 §5.2, §6.7, D56.
- **Синонимы:** Raft instance.
- **Связанные:** Multi-Raft, Shard, WAL, Shared WAL pool.

---

#### Read amplification

- **Категория:** PERF
- **Определение:** Количество дисковых чтений на один GET.
- **Контекст TephraKV:** v0.1–v0.2: p99 < 3 (LSM без KV separation). v0.3+: p99 < 2 (с KV separation / WiscKey). Memory reads (L0 в кэше) не считаются. Публикуется disk reads + memory reads отдельно.
- **Ссылки:** ADR-003, HLD-000 v7.0 §3.5, §4.5, BENCH-008, D8, D60.
- **Синонимы:** read amp.
- **Связанные:** LSM, Bloom filter, Partition index, VLog.

---

#### ReadIndex

- **Категория:** CONSISTENCY / DIST
- **Определение:** Механизм linearizable read в Raft: лидер подтверждает, что он всё ещё лидер, через heartbeat round к кворуму, затем обслуживает read.
- **Контекст TephraKV:** **Default** (не опционально). **Clock-free** — no skew or NTP step can make it return stale data. При 1M reads/s и батчинге 100 reads/heartbeat → 10 000 heartbeat/s, каждый heartbeat — 2 сетевых операции. ~20 000 syscalls/s только на ReadIndex. Distributed p99 — миллисекунды, не микросекунды. Жертвуем latency ради корректности при unbounded clock skew.
- **Ссылки:** ADR-011, ADR-036, HLD-000 v7.0 §6.7, D33, D64.
- **Синонимы:** ReadIndex read.
- **Антонимы:** Lease Read.
- **Связанные:** Raft, Linearizability, Clock skew, Lease Read.

---

#### Recovery

- **Категория:** STORAGE / DURABILITY
- **Определение:** Процесс восстановления состояния БД после сбоя: replay WAL + scan SSTable.
- **Контекст TephraKV:** Детерминированный. В distributed — per shard, параллельно, с лимитом concurrent (не более N shard одновременно, чтобы не насытить I/O). Восстанавливает все SYNC_MASTER/SYNC_LEADER записи. Корректно обрабатывает partial write (последняя запись обрезана).
- **Ссылки:** ADR-002, ADR-012 v3, HLD-000 v7.0 §5.4 (принцип 9), BENCH-007.
- **Синонимы:** crash recovery.
- **Связанные:** WAL, LSN, fsync, Failure model.

---

### S

#### Semver

- **Категория:** PROCESS
- **Определение:** Схема версионирования Major.Minor.Patch.
- **Контекст TephraKV:** Major — несовместимое изменение API/формата. Minor — новая фича. Patch — bugfix.
- **Ссылки:** ENG-001 §11.1, ADR-034.
- **Синонимы:** —
- **Связанные:** CHANGELOG, Deprecation.

---

#### Shard

- **Категория:** ARCH / DIST
- **Определение:** Логическая единица хранения, имеющая свой arena, MemTable, WAL namespace, Raft group.
- **Контекст TephraKV:** 1 shard = 1 Raft group = 1 WAL namespace = 1 MemTable (на лидере) = 1 набор SSTable (на лидере). Shard-per-core (v0.4) — execution unit. Multi-Raft (v0.6) — distribution unit. 10000 shards на кластер. Общий WAL pool (не WAL per shard), но per-shard LSN namespace.
- **Ссылки:** ADR-026 v3, ADR-012 v3, HLD-000 v7.0 §5.2, §5.3, §6.7, D53, D56, D62.
- **Синонимы:** partition (осторожно: не путать с partition index).
- **Связанные:** Multi-Raft, Raft group, Shard-per-core, Shared WAL pool.

---

#### Shard-per-core

- **Категория:** ARCH / PERF
- **Определение:** Модель исполнения, при которой каждый core обслуживает пул shards через event loop.
- **Контекст TephraKV:** v0.4. При 64 ядрах и 10000 shards — ~156 shards на ядро. Не путать с «1 core = 1 shard»: shards статические, распределяются по ядрам.
- **Ссылки:** ADR-026 v3, HLD-000 v7.0 §5.2, §5.3, FEAT-030.
- **Синонимы:** shard-per-core model.
- **Связанные:** Shard, Multi-Raft, Raft group.

---

#### SharedBufferPool

- **Категория:** DIST / PERF
- **Определение:** Пул буферов в grpc-go для encoding/decoding сообщений. Буферы возвращаются после передачи по сети для переиспользования.
- **Контекст TephraKV:** Часть CodecV2. Без него allocs/op ≤ 2 недостижимо. Marshal через generatedSizeVT + MarshalToSizedBufferVT в shared buffer pools.
- **Ссылки:** ADR-029 v2, HLD-000 v7.0 §6.8, D86.
- **Синонимы:** grpc SharedBufferPool.
- **Связанные:** CodecV2, Message codec, Zero-alloc, MaterializeToBuffer.

---

#### Shared WAL pool

- **Категория:** STORAGE / DIST
- **Определение:** Один WAL pool на узел с per-shard LSN namespace. Не WAL per shard.
- **Контекст TephraKV:** O(1) по числу открытых файлов, не O(N_shards). 10000 shards не создают 10000 WAL. Preallocated сегменты (1 ГБ). Rotation. Recovery: WAL replay с фильтрацией по shard, параллельный per shard с лимитом concurrent. Прецеденты: TiKV (один RocksDB WAL на узел для всех Region), Neon (фильтрация WAL на safekeeper).
- **Ссылки:** ADR-012 v3, HLD-000 v7.0 §5.1, §6.7, D53, D62.
- **Синонимы:** shared WAL.
- **Антонимы:** WAL per shard.
- **Связанные:** WAL, LSN, Shard, Raft group.

---

#### Skiplist

- **Категория:** STORAGE
- **Определение:** Вероятностная структура данных, альтернатива B-tree, с O(log n) поиском, вставкой, удалением.
- **Контекст TephraKV:** Lock-free skiplist на offsets в arena. Nodes — не указатели, а offsets. Iterator с epoch guard.
- **Ссылки:** ADR-001, HLD-000 v7.0 §5.4, FEAT-001.
- **Синонимы:** skip list.
- **Связанные:** MemTable, Arena, Lock-free.

---

#### SLA-driven compaction scheduler

- **Категория:** STORAGE / OPS
- **Определение:** Планировщик компакции, который ограничивает одновременно read tail latency (p99) и write-stall duration.
- **Контекст TephraKV:** v0.2. Предотвращает накопление L0 sublevels > 10 и write stall > 100 мс. Rate limiting I/O (token bucket), приоритезация пользовательских операций. Cooldown 60 сек, гистерезис (старт при p99 > 2, стоп при p99 < 1.5). Oscillation dampening — обязательное тестирование.
- **Ссылки:** ADR-021, HLD-000 v7.0 §6.4, D12, D90.
- **Синонимы:** compaction scheduler.
- **Связанные:** Compaction, Bandwidth-aware admission control, L0 partition.

---

#### SLI (Service Level Indicator)

- **Категория:** OPS
- **Определение:** Метрика, измеряющая уровень сервиса: latency, throughput, error rate.
- **Контекст TephraKV:** p99 GET, p999 GET, read amp, WA, allocs/op.
- **Ссылки:** ADR-034, ENG-001 §7.1.
- **Синонимы:** —
- **Связанные:** SLO, Error budget.

---

#### SLO (Service Level Objective)

- **Категория:** OPS
- **Определение:** Целевое значение SLI.
- **Контекст TephraKV:** p99 GET < 1 мс, p999 GET < 5 мс (in-memory, single-node), read amp p99 < 3 (v0.1–v0.2) / < 2 (v0.3+), allocs/op == 0.
- **Ссылки:** ADR-034, ENG-001 §7.2, HLD-000 v7.0 §2.2.
- **Синонимы:** —
- **Связанные:** SLI, Error budget.

---

#### Snapshot Isolation (SI)

- **Категория:** CONSISTENCY
- **Определение:** Уровень изоляции транзакций, при котором каждая транзакция видит консистентный snapshot данных на момент начала.
- **Контекст TephraKV:** v0.4. Single-version MemTable, версии в SST и VLog. Cross-shard — v0.6 с 2PC.
- **Ссылки:** ADR-017, ADR-018, HLD-000 v7.0 §6.6.
- **Синонимы:** SI.
- **Связанные:** MVCC, Timestamp, Watermark.

---

#### Snapshot transfer

- **Категория:** DIST
- **Определение:** Передача состояния Raft group новому или отстающему узлу.
- **Контекст TephraKV:** Incremental, chunked, resume. Snapshot immutable на время transfer. При смене лидера — новый snapshot, resume с начала. Отдельный stream в gRPC.
- **Ссылки:** ADR-014, HLD-000 v7.0 §3.7, §6.8.
- **Синонимы:** snapshot replication.
- **Связанные:** Raft, Raft group, Multi-Raft, Rebalance.

---

#### SSTable (Sorted String Table)

- **Категория:** STORAGE
- **Определение:** Immutable файл на диске с отсортированными key-value парами, индексом, Bloom filter, MinMax index.
- **Контекст TephraKV:** L0 tiered (≤ 4 SSTable), L1+ leveled. Partition index, Bloom 9.6 bits/key. Block size 4 КБ (конфигурируемый). Versioning N-1 backward compat.
- **Ссылки:** ADR-003, HLD-000 v7.0 §5.1, §5.4, FEAT-005.
- **Синонимы:** SST, sorted table.
- **Связанные:** LSM, MemTable, Compaction.

---

#### Stream

- **Категория:** DIST
- **Определение:** Логический канал мультиплексирования в transport (HTTP/2 stream в gRPC).
- **Контекст TephraKV:** **N Raft-групп на 1 stream** (100–250 streams на соединение). Не 1 stream = 1 Raft group. Отдельный stream для snapshot transfer. Отдельный stream для heartbeat. MaxConcurrentStreams: 100–250. Память: при 100–250 streams × INITIAL_WINDOW_SIZE 65 535 байт ≈ 6–16 МБ flow-control буферов на соединение (не 625 МБ при 10000 streams). Прецедент: TiKV `grpc-raft-conn-num` (default 4).
- **Ссылки:** ADR-005 v4, ADR-026 v3, HLD-000 v7.0 §6.8, D63.
- **Синонимы:** channel.
- **Связанные:** gRPC, DRPC, Transport, Raft group, MaxConcurrentStreams.

---

#### SYNC_ALL

- **Категория:** DURABILITY
- **Определение:** Политика, при которой подтверждение даётся после fsync на всех репликах.
- **Контекст TephraKV:** v0.6. Максимальная durability, максимальная latency.
- **Ссылки:** ADR-010, HLD-000 v7.0 §4.4.
- **Синонимы:** —
- **Антонимы:** NO_SYNC.
- **Связанные:** Durability policy, SYNC_MAJORITY.

---

#### SYNC_LEADER

- **Категория:** DURABILITY
- **Определение:** Политика, при которой запись подтверждается после локального fsync на лидере Raft group, **без ожидания кворума**.
- **Контекст TephraKV:** v0.6. В single-node эквивалентен SYNC_MASTER. В distributed — явная семантика fast-path без distributed durability: при смене лидера запись может быть потеряна. Введён в HLD v5.1, чтобы отделить local-only политику от SYNC_MASTER. Линтер `tephra-lint` предупреждает при использовании SYNC_LEADER в single-node.
- **Ссылки:** ADR-010, HLD-000 v7.0 §4.4, §6.2, D54.
- **Синонимы:** —
- **Антонимы:** SYNC_MAJORITY, SYNC_ALL.
- **Связанные:** Durability policy, SYNC_MASTER, Raft.

---

#### SYNC_MAJORITY

- **Категория:** DURABILITY
- **Определение:** Политика, при которой подтверждение даётся после fsync на большинстве реплик.
- **Контекст TephraKV:** v0.6. Устойчивость к отказу меньшинства. Distributed durability.
- **Ссылки:** ADR-010, HLD-000 v7.0 §4.4.
- **Синонимы:** quorum write.
- **Антонимы:** NO_SYNC, SYNC_MASTER, SYNC_LEADER.
- **Связанные:** Durability policy, Raft, Quorum.

---

#### SYNC_MASTER

- **Категория:** DURABILITY
- **Определение:** Политика, при которой подтверждение даётся после локального fsync. В distributed — **не** даёт distributed durability.
- **Контекст TephraKV:** v0.1. p99 PUT < 5 мс. Батчинг с другими SYNC_MASTER. В distributed линтер `tephra-lint` предупреждает при использовании — предпочтителен SYNC_LEADER.
- **Ссылки:** ADR-010, HLD-000 v7.0 §4.4, §6.2, D54.
- **Синонимы:** sync write.
- **Антонимы:** NO_SYNC.
- **Связанные:** Durability policy, fsync, Group commit, SYNC_LEADER.

---

### T

#### TAM (Total Addressable Market)

- **Категория:** BUSINESS
- **Определение:** Общий объём рынка для продукта.
- **Контекст TephraKV:** Целевой сегмент (latency-critical embedded KV): $50–150M, CAGR 15–20%. Общий рынок KV stores: $790–797M (2029–2030). Честное позиционирование: open-source + enterprise support + acqui-hire.
- **Ссылки:** HLD-000 v7.0 §1, PRD-001 v1.0 §12, ADR-032.
- **Синонимы:** —
- **Связанные:** Path to revenue, Design partner.

---

#### TCP (intra-cluster)

- **Категория:** DIST
- **Определение:** Собственный мультиплексированный транспорт поверх TCP с zero-alloc и zero-copy.
- **Контекст TephraKV:** Транспорт v1.0+, если DRPC bottleneck. Полный контроль над zero-alloc. 3–4 недели разработки. Framing — control plane, codec — hot path.
- **Ссылки:** ADR-005 v4, HLD-000 v7.0 §4.7, §6.8, D52.
- **Синонимы:** custom TCP.
- **Связанные:** gRPC, DRPC, Transport.

---

#### Threat model

- **Категория:** SECURITY
- **Определение:** Документ, фиксирующий: какие угрозы рассматриваются, какие митигации применяются, что вне scope.
- **Контекст TephraKV:** `docs/security/storage-threat-model-v0.1.md` (storage) и `docs/security/network-threat-model-v0.6.md` (network). 10 угроз для single-node embedded: corruption, partial write, malicious payload, OOM, path traversal, symlink, info leak, supply chain, TLS downgrade (v0.6), replay (v0.6). HLD v7.0 §9.
- **Ссылки:** HLD-000 v7.0 §9, CHORE-007 (BACKLOG), D57, D67.
- **Синонимы:** security model.
- **Связанные:** Failure model, TLS, Corruption.

---

#### Timestamp

- **Категория:** CONSISTENCY / STORAGE
- **Определение:** Монотонно возрастающее число, определяющее версию данных в MVCC.
- **Контекст TephraKV:** Snapshot timestamp + write set per transaction. Global watermark = min(active snapshot).
- **Ссылки:** ADR-017, HLD-000 v7.0 §6.6.
- **Синонимы:** version, ts.
- **Связанные:** MVCC, Snapshot Isolation, Watermark.

---

#### Titan-style WriteCallback

- **Категория:** STORAGE
- **Определение:** Механизм в Titan (RocksDB plugin), который обеспечивает GC-compatible snapshots через WriteCallback.
- **Контекст TephraKV:** Митигация проблемы WiscKey GC / snapshot consistency. В большинстве KV-separated LSM stores GC несовместим со snapshot function. Titan controls write amplification through Blob file partitioning and targeted GC only on high-garbage-density files. При GC trigger 25% (vs 50%): space amplification снижается на 29.7%, но write amplification растёт на 63.6%.
- **Ссылки:** ADR-022, ADR-023, HLD-000 v7.0 §6.5, D87.
- **Синонимы:** —
- **Связанные:** VLog, WiscKey, GC, Snapshot Isolation.

---

#### TTL (Time-To-Live)

- **Категория:** STORAGE
- **Определение:** Механизм автоматического удаления ключей по истечении заданного времени.
- **Контекст TephraKV:** Модуль (v0.5). Для ниши 1 (Profile Latency). TTL в metadata SST. Auto-GC expired.
- **Ссылки:** HLD-000 v7.0 §5.3, D76.
- **Синонимы:** expiration.
- **Связанные:** Profile Latency, GC.

---

#### Transport

- **Категория:** DIST
- **Определение:** Абстракция передачи сообщений между узлами. В TephraKV реализуется gRPC (v0.6), DRPC (v0.7+), своим TCP (v1.0+).
- **Контекст TephraKV:** N Raft-групп на 1 stream (100–250 streams/conn). Отдельные streams для snapshot и heartbeat. Граница zero-alloc: framing control plane, codec hot path (CodecV2 + SharedBufferPool).
- **Ссылки:** ADR-005 v4, HLD-000 v7.0 §4.7, §6.8.
- **Синонимы:** transport layer.
- **Связанные:** gRPC, DRPC, TCP, Stream, Framing, Message codec, CodecV2.

---

#### Two-Level Metrics

- **Категория:** PERF / PROCESS
- **Определение:** Методология, при которой каждая внешняя метрика публикуется вместе с соответствующей внутренней.
- **Контекст TephraKV:** Внешняя: p99/p999, throughput, cost. Внутренняя: cycles/op, allocs/op, read amp, WA. Обязательно в каждом релизе.
- **Ссылки:** ADR-020, ENG-001 §9, HLD-000 v7.0 §4.3, D7, D22.
- **Синонимы:** paired metrics.
- **Связанные:** BENCH, SLO, SLI.

---

#### Two profiles, one core

- **Категория:** ARCH
- **Определение:** Принцип: одна БД обслуживает две ниши (latency и compliance) через конфигурацию, не через форк.
- **Контекст TephraKV:** Ядро (WAL, MemTable, LSM, VLog, compaction) — общее. Профили (Latency / Compliance / Custom) — наборы параметров. Модули (MVCC, audit, encryption, PITR, TTL) — включаются. HLD v7.0 §6.1, D71.
- **Ссылки:** HLD-000 v7.0 §6.1, D71–D77.
- **Синонимы:** one DB, two profiles.
- **Связанные:** Profile Latency, Profile Compliance, Profile Custom.

---

### U

#### Ubiquitous language

- **Категория:** DOMAIN
- **Определение:** Единый словарь терминов, используемый в коде, документации, тестах, диаграммах и разговорах.
- **Контекст TephraKV:** Этот глоссарий. Если термин используется — он определён здесь. Если нет — он не существует.
- **Ссылки:** ADR-034, HLD-000 v7.0 §5.4.
- **Синонимы:** domain language, shared vocabulary.
- **Связанные:** Bounded context, Glossary.

---

### V

#### Value Log (VLog)

- **Категория:** STORAGE
- **Определение:** Отдельный append-only файл для больших значений, чтобы не раздувать LSM. WiscKey-подход.
- **Контекст TephraKV:** Threshold 256 Б (key + value, конфигурируемый). GC background, rate-limited. Space amp < 1.3 после GC. v0.3. Проблемы: update-heavy workload (GC дорогой), range-query workload (degradation), GC-snapshot inconsistency (митигация: Titan-style WriteCallback).
- **Ссылки:** ADR-022, ADR-023, HLD-000 v7.0 §6.5, D65, D87.
- **Синонимы:** WiscKey, blob store.
- **Связанные:** LSM, GC, Threshold, Titan-style WriteCallback.

---

#### ViscKey

- **Категория:** STORAGE
- **Определение:** См. `Value Log (VLog)`.

---

#### W

#### WAL (Write-Ahead Log)

- **Категория:** STORAGE / DURABILITY
- **Определение:** Append-only журнал, в который записываются все операции до применения к storage.
- **Контекст TephraKV:** = Raft log per shard (ADR-012 v3). **Shared WAL pool** с per-shard LSN namespace (не WAL per shard). O(1) по числу открытых файлов. Preallocated 1 ГБ сегменты. CRC32C, LSN. Group commit. Формат включает term/index поля (нулевые в single-node) для совместимости v0.4 → v0.6.
- **Ссылки:** ADR-002, ADR-012 v3, HLD-000 v7.0 §5.1, §6.7, D53, D62.
- **Синонимы:** journal, commit log.
- **Связанные:** LSN, fsync, Recovery, Group commit, Shard, Raft group, Shared WAL pool.

---

#### Watermark

- **Категория:** CONSISTENCY / STORAGE
- **Определение:** Минимальный timestamp среди всех активных снапшотов. Версии ниже watermark можно удалять.
- **Контекст TephraKV:** Global watermark = min(active snapshot timestamp). Per-shard watermark + координация через Raft (v0.6+).
- **Ссылки:** ADR-017, HLD-000 v7.0 §6.6.
- **Синонимы:** GC watermark.
- **Связанные:** MVCC, Snapshot Isolation, Timestamp, GC версий.

---

#### Wire format

- **Категория:** DIST
- **Определение:** Формат сериализации payload сообщений между узлами. Отделён от framing.
- **Контекст TephraKV:** Свой бинарный, внутри gRPC message (ADR-029 v2). Zero-copy decode где возможно. Versioning с первого дня (N-1 backward compat). Message codec — hot path. **CodecV2 + SharedBufferPool, не protobuf.**
- **Ссылки:** ADR-029 v2, ADR-009 v2, HLD-000 v7.0 §6.8, §6.9, D39, D70.
- **Синонимы:** protocol, serialization format.
- **Связанные:** gRPC, Message codec, Framing, Raft transport, CodecV2.

---

#### WiscKey update-heavy degradation

- **Категория:** STORAGE / PERF
- **Определение:** Известная проблема WiscKey: при update-heavy workload GC VLog становится дорогим, что приводит к higher space amplification.
- **Контекст TephraKV:** Явно документировано. Threshold конфигурируемый. Требуется отдельная оценка для update-heavy и range-heavy профилей (BENCH-006).
- **Ссылки:** ADR-022, HLD-000 v7.0 §6.5, D65.
- **Синонимы:** WiscKey degradation.
- **Связанные:** VLog, GC, Space amp.

---

#### Write amplification (WA)

- **Категория:** PERF
- **Определение:** Отношение байт, записанных на диск, к байтам, записанным пользователем.
- **Контекст TephraKV:** Цель < 5 (средняя), < 20 (p99). Публикуется средняя **и** p99. v0.3: WA < 3.
- **Ссылки:** HLD-000 v7.0 §2.2, §6.4, BENCH-005.
- **Синонимы:** WA.
- **Связанные:** LSM, Compaction, VLog.

---

### Z

#### Zero-alloc

- **Категория:** PERF
- **Определение:** Контракт: ноль аллокаций в куче на hot path.
- **Контекст TephraKV:** Put, Get, Delete, Scan, WriteBatch, raft apply, transport message codec (CodecV2 + SharedBufferPool). Проверяется `AllocsPerRun == 0`. Граница — control plane vs data plane; framing — control plane. Гарантия — **CI-бенчмарк с `-benchmem`**, не escape analysis (D69). Свой arena (mmap-backed bump allocator), не sync.Pool (D68, D85).
- **Ссылки:** ADR-009 v2, HLD-000 v7.0 §3.2, §6.8, ENG-001 §4.2, D3, D68, D69, D85, D86.
- **Синонимы:** zero allocations.
- **Антонимы:** GC pressure.
- **Связанные:** Arena, Escape analysis, sync.Pool, Framing, Message codec, CodecV2, CI-бенчмарк.

---

### А (кириллица)

#### Артефакт

- **Категория:** PROCESS
- **Определение:** Файл или набор файлов, создаваемых/изменяемых задачей.
- **Контекст TephraKV:** Перечисляется в BACKLOG для каждой задачи. Используется для DoD-проверки.
- **Ссылки:** BACKLOG-001 v5.0 §0.6.
- **Синонимы:** deliverable.
- **Связанные:** Definition of Done.

---

### К

#### Критический путь

- **Категория:** PROCESS
- **Определение:** Последовательность задач, задержка которых сдвигает весь релиз.
- **Контекст TephraKV:** Для v0.1: CHORE-001 → CHORE-002 → SPIKE-001 → FEAT-001 → FEAT-002 → FEAT-003 → FEAT-004 → FEAT-006 → TEST-001 → CHORE-005 → DOC-001.
- **Ссылки:** BACKLOG-001 v5.0 §0.10.
- **Синонимы:** critical path.
- **Связанные:** SP, Приоритет.

---

### С

#### Story Points (SP)

- **Категория:** PROCESS
- **Определение:** Относительная оценка сложности задачи. 1 SP ≈ 0.5 дня фокусной работы.
- **Контекст TephraKV:** Фиксированная шкала: 1, 2, 3, 5, 8. SP > 8 — разбить.
- **Ссылки:** BACKLOG-001 v5.0 §0.7.
- **Синонимы:** SP.
- **Связанные:** Backlog, Приоритет.

---

## 2. Индексы по категориям

### ARCH

- Bounded context, Profile Custom, Profile Latency, Profile Compliance, Shard, Shard-per-core, Two profiles one core, Fitness function.

### STORAGE

- Arena, Bloom filter, Compaction, Frozen MemTable, L0 partition, Leveled compaction, LSM, Manifest, MemTable, MinMax block index, MVCC, Partition index, Recovery, Skiplist, SSTable, Titan-style WriteCallback, TTL, Value Log, VLog, WAL, Watermark, Timestamp, WiscKey update-heavy degradation, Encryption at rest.

### DIST

- Backpressure, CodecV2, DRPC, Framing, gRPC, MaterializeToBuffer, Message codec, Multi-Raft, QUIC, Raft, Raft group, SharedBufferPool, Snapshot transfer, Stream, TCP, Transport, Wire format, Clock skew, Monotonic raw clock, Instant.

### CONSISTENCY

- Consistency model, Linearizability, ReadIndex, Lease Read, Snapshot Isolation, Timestamp, Watermark, Clock skew.

### PERF

- Arena, Bandwidth-aware admission control, BENCH, CodecV2, Escape analysis, Framing, Group commit, Hot path, MaterializeToBuffer, Message codec, p99/p999, Read amplification, SharedBufferPool, Two-Level Metrics, Write amplification, Zero-alloc.

### DURABILITY

- Durability policy, fsync, Group commit, LSN, NO_SYNC, Recovery, SYNC_ALL, SYNC_LEADER, SYNC_MAJORITY, SYNC_MASTER, WAL, PITR.

### OPS

- Capacity model, Error budget, Incident, SLI, SLO, SLA-driven compaction scheduler.

### PROCESS

- ADR, BENCH, CHANGELOG, Decision Log, Definition of Done, Definition of Ready, Fitness function, Semver, Story Points, Артефакт, Критический путь.

### DOMAIN

- Bounded context, Ubiquitous language.

### BUSINESS

- Acquired, Design partner, Open-core, Path to revenue, TAM.

### SECURITY

- Audit log, Encryption at rest, Failure model, Threat model.

---

## 3. Индексы по версиям

### v0.1

- Arena, Bloom filter, Durability policy, Escape analysis, Frozen MemTable, Group commit, Hot path, LSN, MemTable, NO_SYNC, Recovery, Skiplist, SSTable, SYNC_MASTER, SYNC_LEADER, WAL (shared pool), Zero-alloc, Failure model, Threat model, CI-бенчмарк.

### v0.2

- Compaction, Leveled compaction, MinMax block index, Partition index, Read amplification, SLA-driven compaction scheduler, Bandwidth-aware admission control, L0 partition.

### v0.3

- Value Log, VLog, Space amp, WiscKey update-heavy degradation, Titan-style WriteCallback.

### v0.4

- MVCC, Snapshot Isolation, Shard, Shard-per-core, Timestamp, Watermark.

### v0.5

- TTL, PITR, Prometheus exporter, Capacity model, Audit log.

### v0.6

- Backpressure, CodecV2, gRPC, Multi-Raft, Raft, Raft group, ReadIndex, SharedBufferPool, Snapshot transfer, Stream, SYNC_ALL, SYNC_MAJORITY, SYNC_LEADER, Wire format, Framing, Message codec, Transport, MaterializeToBuffer, Encryption at rest, Lease Read, Instant (monotonic raw clock).

### v0.7

- Columnar, SIMD, Late materialization, DRPC.

### v1.0

- SOC 2, SSI, Multi-region, RBAC, TCP (intra-cluster), Свой Raft (если команда 5+).

### v1.0+

- QUIC (cross-region only).

---

## 4. Что дальше

1. **Утвердить GLOSSARY v2.0** — статус `Active`.
2. **Синхронизировать** с ADR-034, ADR-035 (ссылки HLD v7.0).
3. **Синхронизировать** с BACKLOG v5.0 (обновить CAN-005).
4. **Обновлять** при каждом новом ADR.

**Правило:** если термин используется в коде или документации, но его нет в глоссарии — это баг.

---

## 5. Что изменилось против v1.1

**Синхронизация с HLD-000 v7.0, PRD-001 v1.0, ROADMAP-001 v3.0, BACKLOG-001 v5.0.**

Содержательные правки:

- **Название:** TephraDB → TephraKV.
- **Raft core v0.6:** etcd/raft (D78). Свой Raft — v1.0+ (D79). Обновлены `Raft`, `Multi-Raft`, `Raft group`.
- **Транспорт:** CodecV2 + SharedBufferPool (D86). Добавлены `CodecV2`, `SharedBufferPool`, `MaterializeToBuffer`.
- **Профили:** `Profile Latency`, `Profile Compliance`, `Profile Custom`, `Two profiles one core` (D71–D77).
- **Модули:** `Audit log`, `Encryption at rest`, `PITR`, `TTL` (HLD v7.0 §5.3).
- **Clock:** `Instant (monotonic raw clock)`, `Clock skew` (D88).
- **Compaction:** `Bandwidth-aware admission control`, `L0 partition`, `SLA-driven compaction scheduler` (D89, D90).
- **WiscKey:** `WiscKey update-heavy degradation`, `Titan-style WriteCallback` (D65, D87).
- **Read path:** `ReadIndex` (clock-free), `Lease Read` (bounded drift) (D64).
- **WAL:** `Shared WAL pool` (D53, D62).
- **Arena:** mmap-backed bump allocator (~2.9 нс) (D85).
- **Обновлены:** `Bloom filter` (840 МБ, не 8.4 ГБ), `Hot path` (transport codec), `Read amplification` (v0.1–v0.2 < 3, v0.3+ < 2), `Durability policy` (5 политик + SYNC_LEADER).
- **Удалены:** CAN-005 (свой Raft для v0.6).

Структурные правки:

- §0.1 «Что изменилось» — новая секция.
- §0.4: категория SECURITY расширена.
- §5 «Что изменилось» — новая секция.

Ссылки:

- HLD-000 v5.1 → v7.0.
- ROADMAP-001 v2.1 → v3.0.
- BACKLOG-001 v4.1 → v5.0.
- PRD-001 v1.0 — добавлен.
- ADR-034, ADR-035 — добавлены.

---

**Конец TephraKV-GLOSSARY-001 v2.0**