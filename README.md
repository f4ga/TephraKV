# TephraKV

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=rect&color=0d1117&height=110&section=header&text=TephraKV&fontSize=44&fontColor=e6edf3&fontAlignY=45&desc=disk%20kv%20store%20%C2%B7%20go%20%C2%B7%20embedded%20server%20distributed&descAlignY=72&descSize=13&descColor=7d8590" alt="TephraKV"/>

<br/>

[![Go](https://img.shields.io/badge/Go-1.24+-00ADD8?style=flat-square&logo=go&logoColor=white)](https://go.dev/)
[![License](https://img.shields.io/badge/license-Apache%202.0-3fb950?style=flat-square)](LICENSE)
[![Status](https://img.shields.io/badge/status-early%20development-8957e5?style=flat-square)](#status)

</div>

<br/>

**TephraKV** — дисковая key-value СУБД на Go. Один движок, три режима развёртывания: встраиваемый, серверный, распределённый. Без cgo, без runtime-зависимостей, один статический бинарник.

Проект начинается с простого наблюдения: большинство хранилищ оптимизируют среднее, а SLA ломают единичные медленные операции. Поэтому здесь всё подчинено p999 — движок живёт вне кучи GC, долговечность задаётся отдельно для каждой записи, а распределённый режим построен на Multi-Raft без внешнего etcd.

> ⚠️ **Early development.** Реализация v0.1 в работе. API может меняться.

---

<details>
<summary><b>📖 Содержание</b></summary>

<br/>

- [Задача](#задача)
- [Решение](#решение)
- [Архитектура](#архитектура)
- [Быстрый старт](#быстрый-старт)
- [Режимы развёртывания](#режимы-развёртывания)
- [Durability](#durability)
- [Производительность](#производительность)
- [Отличия](#отличия)
- [Status](#status)
- [Roadmap](#roadmap)
- [Документация](#документация)
- [Участие](#участие)
- [Лицензия](#лицензия)

</details>

---

## Задача

Хранилища обычно меряют средними и пиковой пропускной способностью. Но в продакшене SLA нарушают не средние — его нарушают единичные медленные операции. Каждая такая операция — это пропущенный тик, отклонённый запрос, нарушенный контракт.

| | Проблема | Что происходит на практике |
|:---:|:---|:---|
| 🐢 | Паузы GC | BadgerDB: p99 до 18 мс. На сервере с частотой тиков 128 Гц это три пропущенных тика |
| 📉 | Read amplification | LSM без разделения ключей и значений читает с диска 3+ раз на один GET. Больше нагрузка на диск — выше стоимость инстанса под тот же объём |
| 🔒 | Глобальный sync | У RocksDB `WriteOptions.sync` — один флаг на весь батч. Приходится выбирать между скоростью и сохранностью данных |
| 🌉 | cgo-мост | RocksDB: ~200 нс на каждый вызов. На миллионе ops/s это 20% бюджета только на переход границы C/Go |
| 💰 | Стоимость RAM | Redis держит всё в памяти. Для хранилища признаков это 2–5 мс на выдачу и счёт за RAM, растущий линейно с объёмом данных |

Средняя задержка скрывает именно те события, которые ломают SLA. А пиковая пропускная способность в синтетическом бенчмарке мало говорит о том, как система поведёт себя под реальной нагрузкой.

---

## Решение

TephraKV строится не «в среднем быстрая», а предсказуемая. Распределение задержек важнее его центра — и все решения подчинены этому:

| Принцип | Что это значит |
|:---|:---|
| **Горячий путь вне кучи Go** | Нет пауз GC — задержки определяются алгоритмом, а не планировщиком |
| **Долговечность — параметр операции** | Кэш и транзакции живут в одной базе: каждый тип данных получает нужный уровень гарантии |
| **Один движок, три режима** | Один формат данных и один API от прототипа до кластера |

**Целевой профиль:** миллион ключей в L3-кэше, p999 < 500 нс, долговечность на уровне операции, ноль аллокаций на горячем пути во встраиваемом режиме.

---

## Архитектура

Все компоненты хранилища живут **вне кучи Go**. GC не отслеживает объекты движка и не вносит паузы в горячий путь.

| Компонент | Решение | Эффект |
|:---|:---|:---|
| Память | Arena на mmap | Аллокация вне GC, 0 аллокаций на горячем пути |
| Адресация | Offsets вместо указателей | Нет write barriers, плотное размещение |
| Memtable | Lock-free skiplist | Нет contention на чтениях |
| WAL | Общий пул + group commit | Долговечность на операцию без потери пропускной способности |
| Долговечность | Политика на операцию | Выбор уровня гарантии для каждой записи |
| Консенсус | WAL = Raft log | Единое хранилище для встраиваемого и распределённого режимов |
| Хранилище | KV separation (WiscKey) | Снижение read amplification и write amplification |
| Wire | CodecV2 + SharedBufferPool | Серверный режим без нагрузки на GC |

<details>
<summary><b>Память под контролем: почему arena на mmap</b></summary>

<br/>

Каждая аллокация в heap Go — это работа для GC: чем больше живых объектов, тем чаще и дольше паузы. Если убрать аллокации с горячего пути, GC перестаёт вмешиваться в работу базы. Именно это и делает arena: skiplist, буферы WAL, ключи и значения размещаются в отдельном mmap-регионе, который GC вообще не видит.

Аллокация в arena — просто сдвиг указателя вперёд: **~2.9 нс против ~40 нс** для heap. Освобождение — epoch-based reclamation, без блокировок: память возвращается в оборот только когда все читатели текущей эпохи закончили работу.

**Почему не `sync.Pool`.** Пул возвращает объекты в кучу Go — GC их видит и сканирует, время жизни недетерминировано. Это ровно та проблема, от которой уходим. mmap даёт контроль над страницами: `madvise`, освобождение целыми регионами, без фрагментации heap'а.

**Чем платим.** Ручное управление временем жизни — ошибка даёт use-after-free, а не панику Go runtime с трейсом.

→ [ADR-019 Arena + epoch reclamation](docs/adr/ADR-019-arena-epoch.md)

</details>

<details>
<summary><b>Offsets вместо указателей: граф, который GC не сканирует</b></summary>

<br/>

Go-указатель — это не просто адрес. Запись через указатель требует write barrier, GC обязан сканировать граф объектов на каждой сборке, а сами объекты «пинятся» к своим адресам. Всё это лишняя работа на горячем пути.

Мы заменили указатели на **смещения (`uint64`) внутри арены**. Логический граф «узел → следующий узел» выражен числом байтов от начала арены. Offset — обычный `uint64`, GC его не видит и не сканирует. Побочный эффект: при flush в SSTable граф копируется как единый блок памяти, а не обходится узел за узлом.

**Чем платим.** Ручная offset-арифметика — ошибка даёт segfault вместо panic с понятным стеком.

→ [ADR-001 Lock-free skiplist](docs/adr/ADR-001-lockfree-skiplist.md)

</details>

<details>
<summary><b>Skiplist вместо B-tree: почему lock-free важнее локальности</b></summary>

<br/>

Memtable принимает все записи и обслуживает все чтения. Мьютекс здесь означает, что даже чистые чтения контендят друг с другом и с писателями. При росте числа ядер contention растёт квадратично — а мы целимся в 64 ядра.

Skiplist позволяет убрать мьютексы полностью: поиск спускается по уровням без блокировок, вставка фиксируется CAS. Чтения не блокируются писателями и **не имеют «хвостов ожидания»** — задержка чтения не зависит от того, кто держит блокировку.

**Почему не Bw-tree или ART.** Lock-free B-tree (Bw-tree) требует сложной перебалансировки через CAS и большого объёма памяти на split. ART хорош для точечных lookup, но обход диапазона через него не естественен — а KV-хранилищу нужен эффективный scan.

**Чем платим.** Локальность кэша хуже, чем у B-tree — обход по уровням даёт больше промахов. Смягчается плотным размещением в arena.

→ [ADR-001 Lock-free skiplist](docs/adr/ADR-001-lockfree-skiplist.md)

</details>

<details>
<summary><b>Один fsync на группу: как работает общий пул WAL</b></summary>

<br/>

fsync — операция на десятки микросекунд. Делать её на каждую запись означает потолок в десятки тысяч ops/s. Group commit собирает записи в группу и выполняет **один fsync на всю группу** — durability сохраняется, а пропускная способность растёт кратно.

WAL — общий пул из 16–64 сегментов с per-shard LSN namespace. Записи из разных шардов попадают в один буфер и коммитятся вместе.

**Почему не WAL на каждый shard.** При 10 000 shards это 10 000 открытых дескрипторов, 10 000 очередей fsync и 10 000 задач rotation. O(N) по ресурсам — не масштабируется. Прецеденты: TiKV (один RocksDB WAL на узел для всех Region), Neon (фильтрация WAL на safekeeper).

**Чем платим.** Восстановление сложнее — нужно фильтровать записи по shard при recovery.

→ [ADR-012 WAL = Raft log](docs/adr/ADR-012-wal-equals-raft-log.md), [ADR-002 WAL format](docs/adr/ADR-002-wal-format.md)

</details>

<details>
<summary><b>Durability на операцию: почему не один флаг на инстанс</b></summary>

<br/>

Один инстанс БД обслуживает разные типы данных: транзакции, где нужен fsync на каждую операцию, и кэш, где он не нужен. Глобальная настройка sync заставляет выбирать худшее из двух — либо всё медленно, либо всё с риском потери.

Мы сделали durability **аргументом `Put`**: приложение само решает, где риск потери допустим, а где нет. Кэш пишется без fsync, критичная запись — с fsync. Пять политик: `NO_SYNC`, `SYNC_MASTER`, `SYNC_LEADER`, `SYNC_MAJORITY`, `SYNC_ALL`.

**Чем платим.** Смешанные батчи запрещены (`ErrMixedDurability`) — правило, которое нужно объяснить пользователю. Зато двусмысленности не остаётся.

→ [ADR-010 Durability semantics](docs/adr/ADR-010-durability-semantics.md)

</details>

<details>
<summary><b>WAL как Raft log: почему единое хранилище для локального и распределённого</b></summary>

<br/>

Raft log и WAL — оба append-only потоки. В распределённом режиме можно использовать один файл вместо двух, и мы так и делаем.

**Почему не раздельно.** Раздвоение даёт двойной fsync, двойное хранение одних и тех же записей и класс ошибок «восстановление из WAL не совпадает с восстановлением из Raft log». Один поток — один recovery, одно место для ошибок.

**Чем платим.** Формат WAL обязан включать term/index поля с первого дня, даже в single-node. Нулевые значения — не «мёртвый код», а контракт совместимости на будущее.

→ [ADR-012 WAL = Raft log](docs/adr/ADR-012-wal-equals-raft-log.md)

</details>

<details>
<summary><b>KV separation: почему большие значения вынесены из LSM</b></summary>

<br/>

В классическом LSM значение копируется на каждый уровень при компакции. Для values 1 КБ и выше это доминирующая стоимость записи — write amplification 10–30×. Компакция перезаписывает одни и те же значения снова и снова.

KV separation (WiscKey) выносит значения больше 256 Б в отдельный VLog. Компакция в LSM работает только с ключами — маленькими и лёгкими. Значения пишутся один раз и живут до следующего обновления.

**Чем платим.** Дополнительный seek на чтение (offset из LSM → value из VLog), GC value log с учётом snapshot-совместимости, и деградация на update-heavy нагрузках — документированное ограничение.

→ [ADR-022 VLog GC](docs/adr/ADR-022-vlog-gc.md), [ADR-023 VLog + Raft](docs/adr/ADR-023-vlog-raft.md)

</details>

<details>
<summary><b>CodecV2: почему собственный wire-формат вместо protobuf</b></summary>

<br/>

gRPC Codec V1 с protobuf аллоцирует буфер на каждый message через `BufferSlice.Materialize`. Для control plane это допустимо, для data plane на миллионе ops/s — источник GC-нагрузки. Ровно та проблема, которую встраиваемый режим решает ареной.

CodecV2 — API grpc-go 1.66+, позволяющий decode напрямую в pooled buffer без копирования. Собственный wire-формат даёт контроль над layout полей: fixed-size поля на фиксированных смещениях, variable-length — по длине в заголовке.

**Результат:** 2.4× быстрее Unmarshal, 2.7× быстрее Marshal, **~300× меньше аллокаций**.

**Чем платим.** Нет protobuf-совместимости, свой формат нужно версионировать и поддерживать N-1 backward compat вручную.

→ [ADR-029 Wire vs protobuf](docs/adr/ADR-029-wire-vs-protobuf.md)

</details>

---

## Быстрый старт

> ⚠️ **v0.1 в разработке.** Модуль появится в `go get` после публичного релиза. API — целевой, см. [API-001](docs/TephraKV-API-001%20v1.0.md).

```bash
go get github.com/tephrakv/tephrakv@v0.1.0
```

```go
package main

import (
    "fmt"
    "log"

    "github.com/tephrakv/tephrakv"
)

func main() {
    // Открываем базу: встраиваемый режим, безопасный default по долговечности.
    db, err := tephrakv.Open("/var/lib/tephrakv", tephrakv.Options{
        Mode:              tephrakv.ModeEmbedded,
        DefaultDurability: tephrakv.SYNC_MASTER, // 0 потерь при краше
    })
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Запись: 0 аллокаций на горячем пути, fsync через group commit.
    if err := db.Put([]byte("player:42:score"), []byte("12800")); err != nil {
        log.Fatal(err)
    }

    // Чтение: значение + release-функция — без копий и аллокаций.
    // release() обязателен: он отпускает эпоху арены.
    value, release, err := db.Get([]byte("player:42:score"))
    if err != nil {
        log.Fatal(err)
    }
    defer release()
    fmt.Printf("score = %s\n", value)

    // Обход диапазона: колбэк без аллокаций на вызов.
    _ = db.Scan([]byte("player:"), []byte("player;"), func(k, v []byte) error {
        fmt.Printf("%s = %s\n", k, v)
        return nil
    })

    // Батч: атомарная группа записей с общей политикой долговечности.
    batch := db.NewBatch()
    batch.Put([]byte("a"), []byte("1"))
    batch.Delete([]byte("b"))
    _ = db.WriteBatch(batch)

    // Метрики: внешние (p50/p99/p999) + внутренние (тиры).
    external, internal := db.Metrics()
    _, _ = external, internal
}
```

### Публичный API (v0.1)

```go
func Open(path string, opts Options) (*DB, error)
func (db *DB) Close() error

func (db *DB) Put(key, value []byte, opts ...WriteOption) error
func (db *DB) Get(key []byte, opts ...ReadOption) (value []byte, release func(), err error)
func (db *DB) Delete(key []byte, opts ...WriteOption) error
func (db *DB) Scan(start, end []byte, fn func(k, v []byte) error) error
func (db *DB) WriteBatch(batch *Batch, opts ...WriteOption) error

func (db *DB) Metrics() (ExternalMetrics, InternalMetrics)
```

> 💡 **Почему `Get` возвращает `(value, release)`.** Значение не копируется: читается память арены, а `release()` говорит движку, когда её можно вернуть в оборот. Zero-alloc контракт виден прямо в типе.

---

## Режимы развёртывания

|  | Встраиваемый | Серверный | Распределённый |
|:---|:---:|:---:|:---:|
| **Форма** | In-process библиотека | Standalone процесс | Multi-Raft кластер |
| **Клиенты** | Go | gRPC + REST | Go + gRPC |
| **Консенсус** | — | — | Dragonboat |
| **Внешний etcd** | — | — | Не требуется |
| **Долговечность** | `NO_SYNC`, `SYNC_MASTER` | + `SYNC_LEADER` | + `SYNC_MAJORITY`, `SYNC_ALL` |
| **Zero-alloc** | ✅ | ❌ | ❌ |

**Модель данных:** `1 shard = 1 Raft group = 1 namespace`. Переход из встраиваемого режима в распределённый не требует смены формата или API.

<details>
<summary><b>Multi-Raft без изобретения колеса: почему Dragonboat</b></summary>

<br/>

Свой Multi-Raft — это не «пара функций», а годы работы: состав кластера, snapshot'ы, лог GC, транспорт, детекция отказов. Для команды это зарплаты нескольких инженеров на полтора года, потраченные не на продукт.

Dragonboat — production-grade библиотека: 1.25M writes/s на группу, до 10 000 групп на узел, пройден Jepsen. **Почему не etcd/raft:** 5–7 allocs/op, ~44K writes/s, нет Multi-Raft — библиотека для одного-двух реплицированных KV, не для тысяч групп.

→ [ADR-006 Multi-Raft](docs/adr/ADR-006-multi-raft.md)

</details>

<details>
<summary><b>Tan engine вместо RocksDB LogDB: как сэкономить порядок на дисках</b></summary>

<br/>

Raft log — чисто append-only поток. RocksDB LSM для него даёт space amplification: записи размазаны по уровням, лишние компакции. Tan engine — log-file структура, пишет ровно столько, сколько получает.

**Экономия:** ~10 ГБ на 10 000 shards против ~10 ТБ при WAL-per-shard preallocation. Разница в стоимости дисков — на порядок.

→ [ADR-023 VLog + Raft](docs/adr/ADR-023-vlog-raft.md)

</details>

<details>
<summary><b>ReadIndex по умолчанию: почему безопаснее Lease Read</b></summary>

<br/>

Lease Read быстрее — но зависит от bounded clock drift. При рассинхроне часов можно прочитать устаревшее значение. В финансовом контексте stale read означает неверный баланс или устаревшую цену.

ReadIndex не использует часы вовсе — безопасен при любом clock skew. Цена — дополнительный round-trip, который батчится.

→ [ADR-011 ReadIndex vs Lease](docs/adr/ADR-011-readindex-vs-lease.md)

</details>

---

## Durability

| Политика | Встраиваемый | Серверный | Распределённый | Гарантия |
|:---|:---:|:---:|:---:|:---|
| `NO_SYNC` | ✅ | ✅ | ✅ | Отсутствует |
| `SYNC_MASTER` | ✅ | ✅ | ✅ | Запись на локальный диск |
| `SYNC_LEADER` | — | ✅ | ✅ | Подтверждение от лидера |
| `SYNC_MAJORITY` | — | — | ✅ | Большинство реплик |
| `SYNC_ALL` | — | — | ✅ | Все реплики |

Политика задаётся **на каждой операции**. Смешанные батчи запрещены — см. [ADR-010](docs/adr/ADR-010-durability-semantics.md).

---

## Производительность

**Tiered latency, single-node.** Нагрузка: YCSB C, key 16 Б, value 64 Б. Методология — [docs/bench/](docs/bench/).

| Tier | Рабочий набор | Кэш | p50 | p99 | **p999** |
|:---:|:---|:---:|:---:|:---:|:---:|
| **T1** | ≤ 100K ключей | L1/L2 | < 50 нс | < 100 нс | **< 200 нс** |
| **T2** | ≤ 1M ключей | L3 | < 100 нс | < 300 нс | **< 500 нс** |
| **T3** | ≤ 100M ключей | RAM | < 300 нс | < 1 мкс | **< 2 мкс** |
| **T4** | Холодный SSTable | NVMe | < 5 мкс | < 20 мкс | **< 50 мкс** |

> 🎯 **Целевой tier — T2:** 1M ключей в L3, p999 < 500 нс.

**Почему tiered, а не одно число.** Разница между L1/L2, L3, RAM и NVMe — три порядка. p999 без указания tier неинформативен: та же система на другом рабочем наборе покажет другую цифру.

> ⚠️ **Все значения — целевые.** Публичные замеры появятся после v0.1.

**Распределённый режим (v0.6a):**

| Метрика | Цель |
|:---|---:|
| p99 GET | < 10 мс |
| p999 GET | < 50 мс |
| Raft groups на узел | до 10 000 |
| Пропускная способность (Dragonboat) | 1.25M writes/s на группу |
| `AllocsPerDistributedWrite` | ≤ 10 |

Распределённый путь через Dragonboat не является zero-alloc — значение `≤ 10` публикуется как есть.

---

## Отличия

| | Redis | RocksDB | BadgerDB | PostgreSQL | **TephraKV** |
|:---|:---:|:---:|:---:|:---:|:---:|
| Персистентность | Ограниченная | ✅ | ✅ | ✅ | **✅** |
| Без cgo | — | — | ✅ | — | **✅** |
| Zero-alloc hot path | ❌ | — | ❌ | — | **✅** (встраиваемый) |
| Долговечность на операцию | ❌ | Частично | Частично | ❌ | **✅** |
| Multi-Raft из коробки | — | — | — | — | **✅** |
| Без внешнего etcd | — | — | — | — | **✅** |
| Встраиваемость | — | ✅ | ✅ | — | **✅** |
| Один бинарник | — | — | — | — | **✅** |

---

## Status

| Ограничение | Детали |
|:---|:---|
| **Early development** | Реализация v0.1 в работе. API может меняться |
| **Zero-alloc** | Только встраиваемый режим. В серверном недостижим на уровне wire-протокола |
| **Single-node** | До v0.6a. Распределённый появится с Dragonboat и Tan engine |
| **Платформы** | Linux x86_64 / ARM64 |

---

## Roadmap

| Версия | Фокус | Ключевая метрика |
|:---:|:---|:---|
| **v0.1** | Deterministic Foundation | **T2 p999 < 500 нс** |
| v0.2 | Read-amp Compaction | Read amplification p99 < 3 |
| v0.3 | Value Log | Write amplification < 3, read amplification < 2 |
| v0.4 | MVCC + Shard-per-core | 50M GET/s |
| v0.5 | TTL + PITR | Первый production-ready релиз |
| **v0.6a** | Distributed KV Core | Freshness < 1 с |
| v0.6b | SQL + Columnar Replica | TPC-C |
| v0.7 | Columnar | SIMD ≥ 4× |
| v1.0 | Enterprise | SOC 2, multi-region |

---

## Документация

- [Architecture overview](docs/architecture.md) — слои движка и принятые решения
- [API reference](docs/api.md) — публичный Go API
- [Format specification](docs/format.md) — WAL, SSTable, Manifest, VLog, Wire
- [Design decisions](docs/adr/) — ADR-реестр с обоснованием каждого выбора
- [Benchmarks](docs/bench/) — методология и результаты

---

## Участие

Проект в ранней разработке. Если хотите помочь — откройте issue с описанием проблемы или предложением. PR приветствуются после обсуждения в issue.

Перед PR:
- `go test ./... -race`
- `golangci-lint run`
- Изменения в публичном API — с обновлением godoc

---

## Лицензия

[Apache 2.0](LICENSE).

<br/>

<div align="center">
<sub><b>TephraKV</b> — predictable tail latency by design.</sub>
</div>

---
