# TephraKV

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=rect&color=0d1117&height=110&section=header&text=TephraKV&fontSize=44&fontColor=e6edf3&fontAlignY=45&desc=disk%20kv%20store%20%C2%B7%20go%20%C2%B7%20embedded%20server%20distributed&descAlignY=72&descSize=13&descColor=7d8590" alt="TephraKV"/>

<br/>

[![Go](https://img.shields.io/badge/Go-1.24+-00ADD8?style=flat-square&logo=go&logoColor=white)](https://go.dev/)
[![License](https://img.shields.io/badge/license-Apache%202.0-3fb950?style=flat-square)](LICENSE)
[![Status](https://img.shields.io/badge/status-design-8957e5?style=flat-square)](#status)

</div>

<br/>

Дисковая key-value СУБД на Go. Встраивается в приложение, разворачивается как standalone-сервер или как Multi-Raft-кластер. Один движок, три режима. Без cgo, без runtime-зависимостей, один статический бинарник.

> **Design project.** Реализация v0.1 в работе. Архитектура зафиксирована в документах (пока много пустых файлов только те, что в корне /docs); публичных бенчмарков нет.

---

## Содержание

- [Задача](#задача)
- [Решение](#решение)
- [Для кого](#для-кого)
- [Архитектура](#архитектура)
- [Режимы развёртывания](#режимы-развёртывания)
- [Durability](#durability)
- [Производительность](#производительность)
- [Отличия](#отличия)
- [Status](#status)
- [Roadmap](#roadmap)
- [Документация](#документация)
- [Лицензия](#лицензия)

---

## Задача

Существующие хранилища оптимизируют средние значения и throughput. На практике критичны другие метрики:

- **p99 и p999** — именно они определяют SLA, а не среднее;
- **стабильность под нагрузкой** — распределение задержек, а не его центр;
- **per-operation durability** — гарантия записи на конкретной операции, а не глобальная настройка инстанса;
- **предсказуемость памяти** — отсутствие пауз GC в горячем пути.

Средняя задержка сглаживает события, которые ломают SLA. Пиковый throughput в бенчмарке не отражает поведение под реальной нагрузкой. Redis быстр, но теряет данные. RocksDB персистентен, но требует cgo. BadgerDB использует Go, но даёт GC-паузы. PostgreSQL надёжен, но не встраивается.

---

## Решение

TephraKV проектируется от требований к p999 к остальным решениям. Не «средняя скорость в вакууме», а распределение задержек, устойчивое к планировщику, GC, contention и I/O.

Три принципа:

1. **Hot path вне кучи Go.** Arena, offsets, lock-free структуры.
2. **Durability — параметр операции.** Не глобальный флаг инстанса.
3. **Один движок, три режима.** Embedded, server, distributed — общее ядро хранения.

Целевой профиль: миллион ключей в L3-кэше, p999 < 500 нс, per-operation durability, ноль аллокаций на hot path в embedded-режиме.

---

## Для кого

| Сегмент | Задача | Почему TephraKV |
|:---|:---|:---|
| **HFT / Trading** | Order book, tick data, tick-to-trade SLA | Низкие p999, per-op durability, zero-alloc |
| **Game Servers** | Состояние сессий при 128-tick бюджете | Нет GC-пауз, быстрый recovery |
| **AI Infra** | Feature store с lookup < 1 мс p99 | RAM × 1.5 vs Redis, gRPC из Python |
| **Edge / IoT** | Embedded KV на 1–2 vCPU, offline | Один бинарник, ARM64, без зависимостей |
| **CDN / Edge Compute** | Локальный KV в сотнях локаций | Pure Go, без cgo, статический бинарник |
| **Fintech / Audit** | Ledger и audit log с retention | Per-op durability, PITR, encryption at rest |

Подробные сценарии — [USER-STORIES-001](docs/USER-STORIES-001.md). Сегментация и GTM — [GTM.md](docs/GTM.md).

---

## Архитектура

Все компоненты хранилища размещены вне кучи Go. GC не отслеживает объекты движка и не вносит паузы в горячий путь. Распределение задержек определяется алгоритмом, а не сборщиком мусора.

| Компонент | Решение | Эффект |
|:---|:---|:---|
| **Память** | Arena на mmap | Аллокация вне GC, 0 allocs/op на hot path |
| **Адресация** | Offsets вместо указателей | Нет write barriers, плотное размещение данных |
| **Memtable** | Lock-free skiplist | Нет contention на чтениях |
| **WAL** | Shared pool + group commit | Per-op durability без потери throughput |
| **Durability** | Policy per operation | Выбор уровня гарантии для каждой операции |
| **Consensus** | WAL = Raft log | Единое хранилище для embedded и distributed |
| **Storage** | KV separation (WiscKey) | Снижение write и read amplification |
| **Wire** | CodecV2 + SharedBufferPool | Server mode без GC-нагрузки |

<details>
<summary><b>Arena на mmap</b></summary>

<br/>

Skiplist, буферы WAL, ключи и значения размещаются в отдельном mmap-регионе. GC не отслеживает эти объекты и не вносит паузы. Аллокация — сдвиг указателя вперёд. Освобождение — epoch-based reclamation без блокировок.

**Почему mmap, а не `sync.Pool` или обычный heap.** `sync.Pool` возвращает объекты в кучу Go — GC их видит и сканирует, lifetime недетерминирован. Bump allocator на куче всё равно платит за write barriers. mmap даёт полный контроль над страницами: `madvise`, освобождение целыми регионами, отсутствие фрагментации heap'а.

Цена: ручное управление жизненным циклом памяти. Выгода: предсказуемое распределение задержек.

</details>

<details>
<summary><b>Offsets вместо указателей</b></summary>

<br/>

Узлы ссылаются смещениями внутри арены, а не Go-указателями. Граф объектов не сканируется GC, flush выполняется побайтовым копированием, данные размещаются плотно. Плотное размещение улучшает попадания в L2/L3.

**Почему offsets, а не указатели.** Go-указатель — это не просто адрес: запись через него требует write barrier, GC обязан сканировать граф при каждой сборке, объекты «пинятся» к адресам. Offset — обычный `uint64`, GC его не видит вообще. Побочный эффект: сериализация графа превращается в `memcpy`, а не в обход по одному узлу.

</details>

<details>
<summary><b>Lock-free skiplist</b></summary>

<br/>

Memtable без мьютексов. Чтения не блокируются писателями. Вставка фиксируется CAS. При росте числа ядер contention не растёт квадратично.

**Почему skiplist, а не lock-free B-tree или ART.** Lock-free B-tree (Bw-tree) требует сложной перебалансировки через CAS и большого объёма памяти на split. ART хорош для in-memory точечных lookup, но range scan через него не естественен. Skiplist — вероятностная структура, где lock-free реализация проще, а диапазонные запросы ложатся на неё напрямую.

</details>

<details>
<summary><b>Shared WAL pool + group commit</b></summary>

<br/>

Один `fsync` на группу записей вместо `fsync` на каждую операцию. `fsync` стоит десятки микросекунд; выполнять его на каждую запись — потолок в десятки тысяч ops/s. Записи собираются в группу и коммитятся одним системным вызовом.

**Почему shared pool, а не WAL на каждый shard.** При 10 000 shards WAL-per-shard означает 10 000 открытых файловых дескрипторов, 10 000 очередей fsync и 10 000 задач rotation. Shared pool даёт O(1) по файлам, единый group commit, а per-shard LSN namespace сохраняет корректность recovery.

</details>

<details>
<summary><b>Durability как параметр операции</b></summary>

<br/>

Уровень гарантии задаётся аргументом `Put`, а не настройкой инстанса. Приложение выбирает политику для каждой операции: от `NO_SYNC` до `SYNC_ALL`.

**Почему per-op, а не глобально.** Один инстанс БД обслуживает разные типы данных: финансовые транзакции, где нужен fsync, и кэш, где он не нужен. Глобальная настройка sync заставляет выбирать худшее из двух. Per-op durability позволяет приложению маршрутизировать гарантию туда, где она действительно требуется.

</details>

<details>
<summary><b>WAL = Raft log</b></summary>

<br/>

В distributed-режиме журнал записи используется напрямую как Raft log. Отдельная подсистема консенсуса не требуется. Recovery, потребление диска и консистентность между локальным и распределённым состоянием упрощаются.

**Почему унифицировано.** Раздвоение WAL и Raft log даёт двойной fsync, двойное хранение одних и тех же записей и класс ошибок «recovery из WAL не совпадает с recovery из Raft log». Raft log — это append-only поток; WAL — тоже append-only поток. Они могут быть одним файлом.

</details>

<details>
<summary><b>KV separation (WiscKey)</b></summary>

<br/>

Ключи и метаданные остаются в LSM, значения выше порога уходят в отдельный value log. Это снижает write amplification на update-heavy нагрузках и уменьшает read amplification.

**Почему WiscKey, а не чистый LSM.** В классическом LSM значение копируется на каждый уровень при компакции. Для values 1 КБ и выше это доминирующая стоимость записи. KV separation убирает значения из compaction-потока — компакция переписывает только ключи. Цена: дополнительный seek на чтение, GC value log и необходимость согласованности со snapshot.

</details>

<details>
<summary><b>CodecV2 + SharedBufferPool</b></summary>

<br/>

Собственный wire-формат вместо protobuf, пул переиспользуемых буферов. Server mode получает предсказуемую нагрузку на GC вместо аллокации на каждый message.

**Почему не protobuf.** gRPC Codec V1 с protobuf аллоцирует буфер на каждый message (`BufferSlice.Materialize`). Для control plane это допустимо, для data plane на 1M ops/s — источник GC-нагрузки. CodecV2 использует фиксированные смещения полей, zero-copy decode и pooled buffers, сохраняя zero-alloc контракт на всём пути.

</details>

---

## Режимы развёртывания

|  | Embedded | Server | Distributed |
|:---|:---:|:---:|:---:|
| **Форма** | In-process библиотека | Standalone процесс | Multi-Raft кластер |
| **Клиенты** | Go | gRPC + REST | Go + gRPC |
| **Consensus** | — | — | Dragonboat |
| **Внешний etcd** | — | — | Не требуется |
| **Durability** | `NO_SYNC`, `SYNC_MASTER` | + `SYNC_LEADER` | + `SYNC_MAJORITY`, `SYNC_ALL` |
| **Zero-alloc** | Да | Нет | Нет |

Модель данных: `1 shard = 1 Raft group = 1 namespace`.

Все три режима используют один движок хранения. Переход от embedded к distributed не требует смены формата на диске или API.

**Почему Dragonboat, а не etcd/raft или собственный Raft.** etcd/raft — библиотека для одного-двух реплицированных KV на процесс: 5–7 allocs/op, ~44K writes/s, без управления тысячами групп. Собственный Multi-Raft — 12–18 месяцев работы (membership, снапшоты, лог GC, транспорт, фейл-детекция). Dragonboat даёт production-grade Multi-Raft с 1.25M writes/s на группу и до 10 000 групп на узел.

**Почему Tan engine, а не RocksDB LogDB.** RocksDB LSM для append-only Raft log даёт space amplification и лишние компакции. Tan engine — log-file структура, пишет ровно столько, сколько получает, recovery — последовательное сканирование сегментов.

**Почему ReadIndex, а не Lease Read по умолчанию.** Lease Read опирается на монотонные часы и оценку drift: при рассинхроне можно прочитать stale данные. ReadIndex не использует часы вообще — безопасен при любом clock skew. Цена — дополнительный round-trip, который батчится.

---

## Durability

| Политика | Embedded | Server | Distributed | Гарантия |
|:---|:---:|:---:|:---:|:---|
| `NO_SYNC` | ✓ | ✓ | ✓ | Отсутствует |
| `SYNC_MASTER` | ✓ | ✓ | ✓ | Запись на локальный диск |
| `SYNC_LEADER` | — | ✓ | ✓ | Подтверждение от лидера |
| `SYNC_MAJORITY` | — | — | ✓ | Большинство реплик |
| `SYNC_ALL` | — | — | ✓ | Все реплики |

Политика задаётся на каждой операции. Приложение выбирает уровень гарантии для каждой записи независимо.

---

## Производительность

Tiered latency, single-node. Workload: YCSB C, key 16 Б, value 64 Б. Методология — в [docs/bench/](docs/bench/).

| Tier | Working set | Cache | p50 | p99 | p999 |
|:---:|:---|:---:|---:|---:|---:|
| **T1** | ≤ 100K ключей | L1/L2 | < 50 нс | < 100 нс | **< 200 нс** |
| **T2** | ≤ 1M ключей | L3 | < 100 нс | < 300 нс | **< 500 нс** |
| **T3** | ≤ 100M ключей | RAM | < 300 нс | < 1 мкс | **< 2 мкс** |
| **T4** | Холодный SSTable | NVMe | < 5 мкс | < 20 мкс | **< 50 мкс** |

Разница между L1/L2, L3, RAM и NVMe составляет три порядка величины. Задержка одной и той же системы зависит от working set и попадания в кэш. Значения p999 без указания tier не информативны.

Целевой tier — **T2**: 1M ключей в L3, p999 < 500 нс.

**Почему tiered, а не одно число p999.** Одна цифра «p999 = X» без указания working set либо маркетинг, либо ошибка измерения. Tiered-таблица честно разделяет состояния: попадание в L1/L2, в L3, в RAM, на NVMe — это разные физические процессы с разной ценой. Публикация p999 без tier запрещена правилом проекта (D113).

> **Все значения — целевые.** Публичные замеры появятся после v0.1.

<br/>

**Distributed (v0.6a)**

| Метрика | Цель |
|:---|---:|
| p99 GET | < 10 мс |
| p999 GET | < 50 мс |
| Raft groups на узел | до 10 000 |
| Throughput (Dragonboat) | 1.25M writes/s на группу |
| `AllocsPerDistributedWrite` | ≤ 10 |

Distributed-путь через Dragonboat не является zero-alloc. Значение `≤ 10` отражает фактическое поведение и публикуется как есть.

---

## Отличия

| Решение | Redis | RocksDB | BadgerDB | PostgreSQL | TephraKV |
|:---|:---:|:---:|:---:|:---:|:---:|
| Персистентность | Ограниченная | Да | Да | Да | Да |
| Без cgo | — | — | Да | — | **Да** |
| Zero-alloc hot path | Нет | — | Нет | — | **Да (embedded)** |
| Durability per operation | Нет | Частично | Частично | Нет | **Да** |
| Multi-Raft из коробки | — | — | — | — | **Да** |
| Без внешнего etcd | — | — | — | — | **Да** |
| Встраиваемость | — | Да | Да | — | **Да** |
| Один бинарник | — | — | — | — | **Да** |

Подробный конкурентный анализ — [docs/product/competitive.md](docs/product/).

---

## Status

| Ограничение | Детали |
|:---|:---|
| **Design only** | Реализация v0.1 в работе. API может меняться |
| **Zero-alloc** | Только embedded. В server-режиме недостижим на уровне wire-протокола |
| **Single-node** | До v0.6a. Distributed появится с Dragonboat и Tan engine |
| **Платформы** | Linux x86_64 / ARM64 |
| **`ProfileCompliance`** | В v0.1 возвращает `ErrNotImplemented` |

---

## Roadmap

| Версия | Фокус | Метрика |
|:---:|:---|:---|
| **v0.1** | Deterministic Foundation | T2 p999 < 500 нс |
| v0.2 | Read-amp Compaction | Read amp p99 < 3 |
| v0.3 | Value Log | WA < 3, read amp < 2 |
| v0.4 | MVCC + Shard-per-core | 50M GET/s |
| v0.5 | TTL + PITR | Первый платящий клиент |
| **v0.6a** | Distributed KV Core | Freshness < 1 с |
| v0.6b | SQL + Columnar Replica | TPC-C |
| v0.7 | Columnar | SIMD ≥ 4× |
| v1.0 | Enterprise | SOC 2, $1M ARR |

Полный план — [ROADMAP.md](docs/ROADMAP.md). Источник правды о текущей работе — [BACKLOG.md](docs/BACKLOG.md). Модель ёмкости — [CAPACITY.md](docs/CAPACITY.md). Нефункциональные требования — [NFR.md](docs/NFR.md).

---

## Документация

**Стратегия и обзор**

- [TephraDB-HLD-000](docs/TephraDB-HLD-000.md) — high-level design
- [ROADMAP.md](docs/ROADMAP.md) — план разработки
- [BACKLOG.md](docs/BACKLOG.md) — текущая работа
- [NFR.md](docs/NFR.md) — нефункциональные требования
- [CAPACITY.md](docs/CAPACITY.md) — модель ёмкости
- [GTM.md](docs/GTM.md) — go-to-market

**Продукт**

- [USER-STORIES-001](docs/USER-STORIES-001.md) — сценарии использования
- [PRICING.md](docs/PRICING.md) — модель ценообразования
- [DR-BCP-001](docs/DR-BCP-001%20v1.0.md) — disaster recovery

**Архитектура и API**

- [TephraKV-API-001](docs/TephraKV-API-001%20v1.0.md) — публичный API
- [TephraKV-FORMAT-001](docs/TephraKV-FORMAT-001%20v1.0.md) — формат на диске
- [TephraDB-ENG-001](docs/TephraDB-ENG-001%20v1.0.md) — инженерные правила
- [GLOSSARY.md](docs/GLOSSARY.md) — термины


## Лицензия

[Apache 2.0](LICENSE). Community-версия. Enterprise-функции (encryption at rest, RBAC, SOC 2) — отдельно.

<br/>

<div align="center">
<sub>TephraKV — predictable tail latency by design.</sub>
</div>
