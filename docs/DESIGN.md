# TephraKV — Design Doc

**Документ:** TephraKV-DESIGN-001
**Версия:** 1.2
**Статус:** Draft
**Связь:** [HLD-000](TephraDB-HLD-000.md), [PRD-001](adr/TephraKV-PRD-001%20v1.0.md), [ROADMAP](ROADMAP.md), [BACKLOG](BACKLOG.md), [NFR](NFR.md), [ADR Registry](adr/README.md)

---

## 0. Как читать этот документ

Design Doc отвечает на три вопроса для того, кто впервые открывает проект:

1. **Что строим** — какая система, для кого, с какими границами.
2. **Почему именно так** — какие были альтернативы и почему они не подошли.
3. **Чем платим** — осознанные компромиссы, а не случайные свойства.

Design Doc не заменяет HLD, ADR, NFR и BACKLOG — он связывает их. При противоречии прав тот документ, на который Design Doc ссылается.

---

## 1. Что строим

**TephraKV** — дисковая key-value СУБД на Go. Один движок, три режима:

| Режим | Что это | Версия |
|---|---|---|
| **Embedded** | In-process библиотека, Go API | v0.1 |
| **Server** | Standalone-процесс, gRPC + REST, 13+ языков | v0.6a |
| **Distributed** | Multi-Raft кластер на Dragonboat, без внешнего etcd | v0.6a |

| Характеристика | Значение |
|---|---|
| Язык | Go 1.24, без cgo |
| Runtime-зависимости | Ноль |
| Артефакт | Один статический бинарник |
| Платформы | Linux x86_64 / ARM64 |
| Hot path (embedded) | 0 аллокаций |
| Целевой tier | T2: 1M ключей в L3, p999 < 500 нс |
| Durability | Policy per operation |
| Лицензия | Apache 2.0 (ядро) |

**Профиль нагрузки:** 1–10M ops/s, working set 100K–100M ключей, значения от десятков байт до мегабайт. Ниши: HFT, game servers, AI-infra, edge, CDN, fintech.

---

## 2. От чего отталкиваемся

### 2.1. Что не так с существующими решениями

| Продукт | Проблема для нашей ниши |
|---|---|
| **BadgerDB** | GC-паузы до 18 мс на p99 — на 128-tick игровом сервере это три пропущенных тика |
| **Pebble** | Read amplification 3+, нет per-op durability |
| **RocksDB** | cgo: ~200 нс на каждый вызов → на 1M ops/s это 20% бюджета только на границу C/Go |
| **Redis** | Держит всё в RAM, не даёт durable по умолчанию |
| **PostgreSQL** | Не встраивается в приложение |

Примеры из реальных кейсов:

- **SpacetimeDB**: 131 КБ аллокаций на одно чтение → stuttering каждые 10 сек по 500–1000 мс.
- **LEGO (ACM)**: `map[int]*struct` даёт 28–32 мс на GC-цикл, `map[int]int` — 0.67–0.73 мс. Дело в количестве ссылок в куче.
- **Netlify**: CGO crash в проде на edge-инфраструктуре.
- **Instacart**: миграция feature store с Redis дала −70% кластера и −50% latency.

### 2.2. Ограничения проекта

| Ограничение | Причина |
|---|---|
| Solo-разработка до v0.6a | Найм distributed-инженера — к v0.6a |
| Без cgo | Zero-alloc контракт + операционная простота |
| Без runtime-зависимостей | Один бинарник, переносимость |
| Linux x86_64 / ARM64 в v0.1 | macOS/Windows — build support с v0.3 |
| Только NVMe/SSD | HDD даёт десятки мс — несовместимо с T4 |
| Нет multi-region до v1.0 | Требует joint consensus и SSI |

---

## 3. Решения

Формат каждого раздела: **что решили → альтернативы → почему → чем платим**. Полное обоснование — в ADR.

---

### 3.1. Язык: Go

**Решение.** Go 1.24+, без cgo.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Rust** | Целевые пользователи — Go-разработчики. Второй язык в стеке = барьер встраивания. Async/await «окрашивает» функции, `Send`/`Sync` добавляют когнитивную нагрузку |
| **C++** | Путь RocksDB с cgo-мостом (~200 нс/вызов). Операционный риск CGO crash |
| **Java / Kotlin** | JVM в embedded-сценарии — противоречит принципу «один бинарник» |

**Почему Go.** Все целевые сегменты — HFT-инфра, игровые серверы, AI-infra, edge — уже пишут на Go. Продукт встраивается без второго языка.

**Ключевое преимущество — горутины.** Не «удобный синтаксис», а фундаментальное свойство для архитектуры:

- **Per-shard worker** (v0.4): shard-per-core требует лёгкой задачи на shard. Горутина — естественная единица. В C++ это был бы thread pool с ручной балансировкой; в Rust — async-задача с ограничениями `Send`/`Sync`.
- **Per-connection goroutine** (server mode): 100–250 стримов на соединение, тысячи соединений. Без thread-per-connection и без callback-ада.
- **Background workers**: compaction, WAL rotation, flush, GC VLog, metrics — каждый в своей горутине, общение через channels.
- **Читатели и писатели skiplist**: lock-free структуры + горутины снимают вопрос «как это запускать параллельно».

**Почему в C++ и Rust это тяжелее.**

- **C++**: `std::thread` — дорогой (ОС-поток), нужен ручной пул, синхронизация через mutex/atomic вручную, гонки ловятся как UB. Для KV-движка с десятками задач это сложная инфраструктура.
- **Rust**: async/await zero-cost, но sync и async не смешиваются без перехода, borrow checker ограничивает паттерны в конкурентном коде.

**Важная оговорка.** Горутины решают «как запустить много задач», но **не решают contention за данные**. Десятки горутин на shared mutex дают contention не хуже десятков потоков. Решается архитектурно: shard-per-core, lock-free структуры, per-shard LSN, атомарные ссылки.

**Цена.** Go GC — фундаментальная проблема, обходится через arena. Scheduler не даёт hard real-time гарантий — hot path должен быть коротким и без вызовов планировщика.

---

### 3.2. Память: arena на mmap

**Решение.** Объекты движка (skiplist, WAL, ключи, значения) — в mmap-backed bump allocator. Освобождение — epoch-based reclamation.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **`sync.Pool`** | Возвращает объекты в кучу Go — GC их видит, сканирует, lifetime недетерминирован |
| **Heap + `runtime.KeepAlive`** | Не устраняет write barriers и не даёт предсказуемого размещения |
| **C-аллокатор через cgo** | Противоречит принципу «без cgo» |

**Почему arena.** mmap даёт полный контроль над страницами: `madvise`, освобождение целыми регионами, отсутствие фрагментации heap'а. Аллокация — сдвиг указателя вперёд: **~2.9 нс против ~40 нс** для heap.

**Цена.** Ручное управление lifetime. Ошибка даёт use-after-free, а не панику Go runtime.

**ADR:** [ADR-019](adr/ADR-019-arena-epoch.md).

---

### 3.3. Адресация: offsets вместо указателей

**Решение.** Узлы skiplist ссылаются смещениями (`uint64`) внутри арены.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Go-указатели** | Write barrier при записи; GC сканирует граф на каждой сборке; объекты «пинятся» к адресам |
| **`unsafe.Pointer`** | Не решает write barriers, добавляет риск |
| **Гибрид** | Указатели на верхних уровнях, offsets на нижних — сложнее, выгода неясна |

**Почему offsets.** `uint64` GC не видит вообще. Побочный эффект: сериализация графа при flush превращается в `memcpy`, а не в обход по узлам.

**Цена.** Ручная offset-арифметика. Ошибка — segfault вместо panic.

**ADR:** [ADR-001](adr/ADR-001-lockfree-skiplist.md).

---

### 3.4. Memtable: lock-free skiplist

**Решение.** Skiplist без мьютексов: поиск спускается по уровням, вставка фиксируется CAS.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Bw-tree** (lock-free B-tree) | Сложная перебалансировка через CAS, много памяти на split |
| **ART** | Хорош для точечных lookup, но range scan не естественен — а KV-хранилищу нужен эффективный scan |
| **Hash table** | Нет упорядоченности, нужной для LSM |
| **B-tree с RCU** | Сложная интеграция с Go memory model |

**Почему skiplist.** Вероятностная структура — lock-free реализация проще, диапазонные запросы ложатся напрямую.

**Цена.** Cache locality хуже B-tree: обход по уровням даёт больше промахов в кэш. Смягчается плотным размещением в arena.

**ADR:** [ADR-001](adr/ADR-001-lockfree-skiplist.md).

---

### 3.5. WAL: shared pool + group commit

**Решение.** Один WAL pool на узел (16–64 сегмента) с per-shard LSN namespace. Записи собираются в группу и коммитятся одним fsync.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **WAL per shard** | 10 000 shards → 10 000 дескрипторов, 10 000 очередей fsync, O(N) по ресурсам |
| **fsync на каждую запись** | Потолок в десятки тысяч ops/s (fsync — десятки мкс) |
| **Без WAL, только snapshot** | Потеря последних N секунд при краше — несовместимо с `SYNC_MASTER` |

**Почему shared pool.** O(1) по файлам, единый group commit, per-shard LSN сохраняет корректность recovery. Прецеденты: TiKV (один RocksDB WAL на узел для всех Region), Neon (фильтрация WAL на safekeeper).

**Цена.** Recovery сложнее: фильтрация записей по shard, параллельное восстановление с лимитом concurrent, чтобы не насытить I/O.

**ADR:** [ADR-012](adr/ADR-012-wal-equals-raft-log.md), [ADR-002](adr/ADR-002-wal-format.md).

---

### 3.6. Durability: policy per operation

**Решение.** Уровень гарантии — аргумент `Put`, не настройка инстанса. Пять политик: `NO_SYNC`, `SYNC_MASTER`, `SYNC_LEADER`, `SYNC_MAJORITY`, `SYNC_ALL`.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Глобальный `sync`** (RocksDB) | Один инстанс обслуживает и транзакции, и кэш. Глобальный sync заставляет выбирать худшее из двух |
| **Отдельные инстансы** для durable и fast | Два процесса, два хранилища — операционная сложность |
| **Асинхронный WAL без выбора** | Нет контроля риска |

**Почему per-op.** Приложение маршрутизирует гарантию туда, где она нужна. Кэш пишется без fsync, платёж — с fsync.

**Цена.** Смешанные батчи запрещены (`ErrMixedDurability`). Per-op state в WAL расширяет формат на несколько байт.

**ADR:** [ADR-010](adr/ADR-010-durability-semantics.md).

---

### 3.7. Consensus: WAL = Raft log

**Решение.** В distributed-режиме журнал записи используется напрямую как Raft log.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Отдельный Raft log** | Двойной fsync, двойное хранение, класс ошибок «recovery WAL ≠ recovery Raft log» |
| **Raft log в LSM** (RocksDB-style) | Space amplification, лишние компакции; Raft log — чисто sequential access |

**Почему унифицировано.** Raft log и WAL — оба append-only потоки. Могут быть одним файлом, если формат поддерживает term/index поля.

**Цена.** Формат WAL обязан включать term/index поля с первого дня, даже в single-node. Нулевые значения — контракт совместимости v0.4 → v0.6a.

**ADR:** [ADR-012](adr/ADR-012-wal-equals-raft-log.md).

---

### 3.8. Storage: KV separation (WiscKey)

**Решение.** Ключи и метаданные — в LSM, значения > 256 Б — в отдельном VLog.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Чистый LSM** | Значение копируется на каждый уровень при компакции. Для values 1 КБ+ это WA 10–30× |
| **Hash-index + heap files** | Теряется упорядоченность, range scan не эффективен |
| **Отдельная СУБД для больших значений** | Два хранилища, транзакционная сложность |

**Почему WiscKey.** Убирает значения из compaction-потока — компакция переписывает только ключи.

**Цена.** Дополнительный seek на чтение (offset из LSM → value из VLog). GC VLog с учётом snapshot. Деградация на update-heavy нагрузках — документированное ограничение.

**ADR:** [ADR-022](adr/ADR-022-vlog-gc.md), [ADR-023](adr/ADR-023-vlog-raft.md).

---

### 3.9. Wire: CodecV2 + SharedBufferPool

**Решение.** Собственный wire-формат, decode без копирования, пул буферов.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **protobuf + gRPC Codec V1** | Аллоцирует буфер на каждый message (`BufferSlice.Materialize`) — источник GC-нагрузки на 1M ops/s |
| **FlatBuffers / Cap'n Proto** | Zero-copy по дизайну, но сложнее и тяжелее в экосистеме Go |
| **JSON для REST** | Оставляем только для REST-части, не для data plane |

**Почему CodecV2.** API grpc-go 1.66+ позволяет decode напрямую в pooled buffer. Собственный формат даёт контроль над layout полей.

**Цена.** Нет protobuf-совместимости, свой формат нужно версионировать вручную с N-1 backward compat.

**ADR:** [ADR-029](adr/ADR-029-wire-vs-protobuf.md).

---

### 3.10. Distributed core: Dragonboat

**Решение.** Raft core — Dragonboat Multi-Raft.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **etcd/raft** | 5–7 allocs/op, ~44K writes/s, нет Multi-Raft. Библиотека для одного-двух KV, не для тысяч групп |
| **hashicorp/raft** | Простой, но без Multi-Raft |
| **Собственный Multi-Raft** | 12–18 месяцев (membership, снапшоты, лог GC, транспорт, фейл-детекция) — непосильно для solo |

**Почему Dragonboat.** Production-grade Multi-Raft на Go: **1.25M writes/s на группу, 9M writes/s на 22 ядрах**, до 10 000 групп на узел, пройден Jepsen.

**Цена.** Dragonboat — control plane: аллокации в нём допустимы, но контракт zero-alloc на Raft path не выполняется. Честно документируется через `AllocsPerDistributedWrite ≤ 10`.

**ADR:** [ADR-006](adr/ADR-006-multi-raft.md).

---

### 3.11. Raft log storage: Tan engine

**Решение.** Raft log — в Tan engine (log-file движок).

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **RocksDB LogDB** | Space amplification, лишние компакции для append-only паттерна |
| **Собственный log-file формат** | Дублирование работы, которой Tan engine уже занимается |
| **LSM для логов** | Raft log — sequential access; LSM создан для random access |

**Почему Tan engine.** Пишет ровно столько, сколько получает. Recovery — последовательное сканирование. Потребление диска: **~10 ГБ на 10 000 shards вместо ~10 ТБ** при WAL-per-shard preallocation.

**Цена.** Tan engine — экспериментальный компонент. Интеграция через `ILogDB` — документированный, но не стабильный public API Dragonboat. Нужен план B (ADR-039 планируется).

**ADR:** [ADR-023](adr/ADR-023-vlog-raft.md).

---

### 3.12. Read path: ReadIndex default

**Решение.** Distributed-чтение по умолчанию — ReadIndex, без часов.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Lease Read** | Опирается на монотонные часы и оценку drift. При рассинхроне — stale read, нарушение линеаризуемости |
| **Follower read без согласования** | Нарушает линеаризуемость |
| **Ожидание commit** | Избыточно, дороже ReadIndex |

**Почему ReadIndex.** Не использует часы — безопасен при любом clock skew. Прецедент: TiKV.

**Цена.** Дополнительный round-trip, батчится. Distributed p99 — миллисекунды, не микросекунды. При 1M reads/s и батчинге 100 reads/heartbeat — ~20 000 syscalls/s только на ReadIndex.

**ADR:** [ADR-011](adr/ADR-011-readindex-vs-lease.md).

---

### 3.13. Режимы: три в одном движке

**Решение.** Embedded, server, distributed — один движок, `Options.Mode`.

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Отдельные продукты** | Три кодовые базы, три формата на диске, три API |
| **Только embedded** | Теряем distributed-нишу (TiKV/CockroachDB) |
| **Только distributed** | Теряем embedded-нишу (HFT, game servers, edge) |

**Почему три в одном.** Общее ядро хранения (WAL, memtable, LSM, VLog, compaction). Профили — наборы параметров, не форки. Переход embedded → distributed не требует смены формата или API.

**Цена.** Сложность растёт нелинейно. Тестирование требует покрытия всех режимов.

**ADR:** [ADR-030](adr/ADR-030-server-vs-embedded.md), [ADR-034](adr/ADR-034-api-semantics.md).

---

### 3.14. Метрики: tiered latency

**Решение.** Публикуются p50/p99/p999 по кэш-уровням: T1 (L1/L2), T2 (L3), T3 (RAM), T4 (NVMe).

**Альтернативы.**

| Вариант | Почему не подходит |
|---|---|
| **Одно число p999** | Без указания working set — либо маркетинг, либо ошибка измерения |
| **Средняя задержка** | Сглаживает события, которые ломают SLA |
| **Throughput без latency** | Не отвечает на вопрос «какой хвост» |

**Почему tiered.** Разница между L1/L2, L3, RAM и NVMe — три порядка. Одна цифра без tier неинформативна. Публикация p999 без tier запрещена правилом проекта (D113).

**Цена.** Метрика сложнее для восприятия, требует понимания своего working set.

**ADR:** [ADR-020](adr/ADR-020-metrics-definitions.md).

---

## 4. Что не строим

Non-goals — не «не успели», а «решили не делать».

| Не строим | До | Причина |
|---|---|---|
| OLAP (кроме columnar replica) | v0.6b | Columnar — отдельная инфраструктура |
| Замена PostgreSQL с SQL | никогда | Разные ниши |
| Замена Redis для in-memory-only | никогда | Другая модель durability |
| Cross-key транзакции | v0.4 | Требует MVCC |
| Multi-region | v1.0 | Требует joint consensus и SSI |
| RBAC, multi-tenant | v1.0 | Enterprise-фича |
| Zero-alloc на server mode | никогда | Framing — control plane |
| Zero-alloc на Raft path | никогда | Dragonboat — control plane |
| Векторный поиск | v0.7+ | Отдельная инфраструктура |
| Работа на HDD | никогда | Физика диска несовместима с T4 |

Полный список — [BACKLOG §3](BACKLOG.md), [BACKLOG §11](BACKLOG.md).

---

## 5. Компромиссы

| Решение | Получаем | Платим |
|---|---|---|
| Go вместо Rust | Embedded в Go-экосистеме | GC — обходим через arena |
| Горутины | Лёгкая конкурентность, миллионы задач | Contention на shared state — обходим через lock-free и shard-per-core |
| Arena вместо heap | 0 allocs/op, нет GC-пауз | Ручной lifetime, use-after-free вместо panic |
| Offsets вместо указателей | Нет write barriers | Offset-арифметика, segfault при ошибке |
| Skiplist вместо B-tree | Lock-free, естественный range scan | Хуже cache locality |
| Shared WAL pool | O(1) по файлам | Сложнее recovery (фильтрация) |
| Per-op durability | Гибкость для приложения | Запрет mixed batch |
| WAL = Raft log | Единый recovery, единое хранилище | Формат с term/index полями с v0.1 |
| WiscKey | Меньше WA и read amp | Seek, GC, деградация на update-heavy |
| CodecV2 | Zero-alloc на wire | Нет protobuf-совместимости |
| Dragonboat | Multi-Raft из коробки | Аллокации в control plane |
| Tan engine | Предсказуемый диск | Экспериментальный, нужен план B |
| ReadIndex default | Безопасен при clock skew | Дополнительный round-trip |
| Три режима в одном движке | Общий код, шире TAM | Нелинейная сложность тестирования |
| Tiered latency | Честная метрика | Сложнее для восприятия |

---

## 6. Открытые вопросы

| Вопрос | Как проверяем |
|---|---|
| T2 p999 < 500 нс на 1M ключей достижим? | BENCH-016 в v0.1 |
| T3 p999 < 2 мкс на 10M ключей достижим? | BENCH-016 |
| Tier distribution ≥ 80% в L1/L2/L3 для 1M ключей? | BENCH-016 |
| Dragonboat даст ≥ 1M ops/s на узел на 8 vCPU? | BENCH-014 в v0.6a |
| `AllocsPerDistributedWrite ≤ 10` реалистично? | BENCH-014 |
| Интеграция Tan engine через `ILogDB` стабильна? | Spike в начале v0.6a |
| Кастомный LogDB не потеряет throughput Dragonboat? | BENCH-014 |
| Shared WAL recovery 10 000 shards уложится в ~1 сек? | BENCH-007 extended |

Полный список — [HLD §8.4](TephraDB-HLD-000.md), [ROADMAP §13](ROADMAP.md).

---

## 7. Что дальше

1. Утвердить Design Doc → статус `Accepted`.
2. Написать ADR-039 «Dragonboat dependency risk» — версия, пиннинг, план B.
3. Написать ADR-040 «Migration v0.5 → v0.6a» — WAL pool → Tan engine.
4. Написать ADR-011 v3 «Distributed consistency model».
5. Начать реализацию v0.1 — [BACKLOG §2](BACKLOG.md).

Правило: Design Doc обновляется при каждом ADR, влияющем на архитектурный ландшафт.

---

**Конец TephraKV-DESIGN-001 v1.2**

---
