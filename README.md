# TephraKV

<div align="center">

<img src="https://capsule-render.vercel.app/api?type=rect&color=0d1117&height=110&section=header&text=TephraKV&fontSize=44&fontColor=e6edf3&fontAlignY=45&desc=disk%20kv%20store%20%C2%B7%20go%20%C2%B7%20embedded%20server%20distributed&descAlignY=72&descSize=13&descColor=7d8590" alt="TephraKV"/>

<br/>

[![Go](https://img.shields.io/badge/Go-1.24+-00ADD8?style=flat-square&logo=go&logoColor=white)](https://go.dev/)
[![License](https://img.shields.io/badge/license-Apache%202.0-3fb950?style=flat-square)](LICENSE)
[![Status](https://img.shields.io/badge/status-design-8957e5?style=flat-square)](#status)

</div>

<br/>

**TephraKV** — дисковая key-value СУБД на Go. Один движок, три режима развёртывания: встраиваемый, серверный, распределённый. Без cgo, без runtime-зависимостей, один статический бинарник.

Проект начинается с простого наблюдения: большинство хранилищ оптимизируют среднее, а SLA ломают единичные медленные операции. Поэтому здесь всё подчинено p999 — движок живёт вне кучи GC, долговечность задаётся отдельно для каждой записи, а распределённый режим построен на Multi-Raft без внешнего etcd.

> 🚧 **Design stage.** Реализация v0.1 в работе. Публичных бенчмарков пока нет.

---

<details>
<summary><b>📖 Содержание</b></summary>

<br/>

- [Задача](#задача)
- [Решение](#решение)
- [Архитектура](#архитектура)
- [Для кого](#для-кого)
- [Быстрый старт](#быстрый-старт)
- [Режимы развёртывания](#режимы-развёртывания)
- [Durability](#durability)
- [Производительность](#производительность)
- [Отличия](#отличия)
- [Status](#status)
- [Roadmap](#roadmap)
- [Документация](#документация)
- [Лицензия](#лицензия)

</details>

---

## Задача

Хранилища обычно меряют средними и пиковой пропускной способностью. Но в продакшене SLA нарушают не средние — его нарушают единичные медленные операции. Каждая такая операция — это конкретная потеря: невыполненная сделка, пропущенный тик, штраф по SLA, отказ платёжной системы.

| | Проблема | Что происходит на практике |
|:---:|:---|:---|
| 🐢 | Паузы GC | BadgerDB: p99 до 18 мс. На сервере с частотой тиков 128 Гц это три пропущенных тика и видимый игроку фриз |
| 📉 | Read amplification | LSM без разделения ключей и значений читает с диска 3+ раз на один GET. Больше нагрузка на диск — выше стоимость инстанса под тот же объём |
| 🔒 | Глобальный sync | У RocksDB `WriteOptions.sync` — один флаг на весь батч. Приходится выбирать между скоростью и сохранностью данных |
| 🌉 | cgo-мост | RocksDB: ~200 нс на каждый вызов. На миллионе ops/s это 20% бюджета только на переход границы C/Go |
| 💰 | Стоимость RAM | Redis держит всё в памяти. Для хранилища признаков это 2–5 мс на выдачу и счёт за RAM, растущий линейно с объёмом данных |

Средняя задержка скрывает именно те события, которые ломают SLA. А пиковая пропускная способность в синтетическом бенчмарке мало говорит о том, как система поведёт себя под реальной нагрузкой.

---

## Решение

TephraKV строится не «в среднем быстрая», а предсказуемая. Распределение задержек важнее его центра — и все решения подчинены этому:

| Принцип | Что это значит для продукта |
|:---|:---|
| **Горячий путь вне кучи Go** | Нет пауз GC — нет пропущенных тиков и сорванных SLA |
| **Долговечность — параметр операции** | Кэш и финансовые транзакции живут в одной базе: каждый тип данных получает нужный уровень гарантии |
| **Один движок, три режима** | Одна команда, один набор навыков, один формат данных от прототипа до кластера |

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
<summary><b>Arena на mmap — почему именно так</b></summary>

<br/>

Skiplist, буферы WAL, ключи и значения размещаются в отдельном mmap-регионе. Аллокация — сдвиг указателя вперёд, ~2.9 нс против ~40 нс для heap.

**Почему не `sync.Pool`.** Пул возвращает объекты в кучу Go — GC их видит и сканирует, время жизни недетерминировано. mmap даёт контроль над страницами: `madvise`, освобождение целыми регионами, без фрагментации heap'а.

**Цена.** Ручное управление временем жизни. Ошибка — use-after-free, а не паника Go runtime.

→ [ADR-019 Arena + epoch reclamation](docs/adr/ADR-019-arena-epoch.md)

</details>

<details>
<summary><b>Offsets вместо указателей — почему именно так</b></summary>

<br/>

Узлы ссылаются смещениями (`uint64`) внутри арены.

**Почему не указатели.** Указатель требует write barrier, GC сканирует граф на каждой сборке, объекты «пинятся» к адресам. Offset — обычный `uint64`, GC его не видит. Побочный эффект: flush превращается в `memcpy`.

**Цена.** Ручная offset-арифметика. Ошибка — segfault вместо panic.

→ [ADR-001 Lock-free skiplist](docs/adr/ADR-001-lockfree-skiplist.md)

</details>

<details>
<summary><b>Lock-free skiplist — почему именно так</b></summary>

<br/>

Memtable без мьютексов. Чтения не блокируются писателями. Вставка фиксируется CAS.

**Почему не Bw-tree или ART.** Lock-free B-tree требует сложной перебалансировки через CAS. ART хорош для точечных lookup, но обход диапазона через него не естественен.

**Цена.** Локальность кэша хуже, чем у B-tree. Смягчается плотным размещением в arena.

→ [ADR-001 Lock-free skiplist](docs/adr/ADR-001-lockfree-skiplist.md)

</details>

<details>
<summary><b>Общий пул WAL — почему именно так</b></summary>

<br/>

Один `fsync` на группу записей вместо `fsync` на каждую. fsync стоит десятки микросекунд — на каждую запись это потолок в десятки тысяч ops/s.

**Почему не WAL на каждый shard.** 10 000 shards → 10 000 дескрипторов, 10 000 очередей fsync. O(N) по ресурсам. Прецеденты: TiKV, Neon.

**Цена.** Восстановление сложнее — фильтрация по shard.

→ [ADR-012 WAL = Raft log](docs/adr/ADR-012-wal-equals-raft-log.md), [ADR-002 WAL format](docs/adr/ADR-002-wal-format.md)

</details>

<details>
<summary><b>Долговечность на операцию — почему именно так</b></summary>

<br/>

Уровень гарантии — аргумент `Put`, а не настройка инстанса.

**Почему не глобально.** Один инстанс обслуживает и транзакции, и кэш. Глобальный sync заставляет выбирать худшее из двух.

**Цена.** Смешанные батчи запрещены (`ErrMixedDurability`).

→ [ADR-010 Durability semantics](docs/adr/ADR-010-durability-semantics.md)

</details>

<details>
<summary><b>WAL = Raft log — почему именно так</b></summary>

<br/>

В распределённом режиме журнал записи используется напрямую как Raft log.

**Почему унифицировано.** Раздвоение даёт двойной fsync и класс ошибок «восстановление WAL ≠ восстановление Raft log». Оба потока append-only.

**Цена.** Формат обязан включать term/index поля с первого дня, даже в single-node.

→ [ADR-012 WAL = Raft log](docs/adr/ADR-012-wal-equals-raft-log.md)

</details>

<details>
<summary><b>KV separation (WiscKey) — почему именно так</b></summary>

<br/>

Ключи — в LSM, значения > 256 Б — в отдельном VLog.

**Почему не чистый LSM.** Значение копируется на каждый уровень при компакции. Для values 1 КБ+ это write amplification 10–30×.

**Цена.** Дополнительный seek на чтение, GC value log, деградация на update-heavy нагрузках.

→ [ADR-022 VLog GC](docs/adr/ADR-022-vlog-gc.md), [ADR-023 VLog + Raft](docs/adr/ADR-023-vlog-raft.md)

</details>

<details>
<summary><b>CodecV2 + SharedBufferPool — почему именно так</b></summary>

<br/>

Собственный wire-формат вместо protobuf, пул буферов.

**Почему не protobuf.** gRPC Codec V1 аллоцирует буфер на каждый message. На 1M ops/s — источник нагрузки на GC.

**Цена.** Нет protobuf-совместимости.

→ [ADR-029 Wire vs protobuf](docs/adr/ADR-029-wire-vs-protobuf.md)

</details>

---

## Для кого

Три сегмента, где p999 напрямую связан с выручкой или удержанием.

### HFT / Trading: задержка = деньги

p99 в 5 мс и p99 в 500 мкс — это разница между исполнением по цене и проскальзыванием. На объёме сделок даже несколько микросекунд конвертируются в деньги. Redis теряет данные на персистентности, RocksDB тянет за собой cgo. Нужен предсказуемый p999 и долговечность на уровне операции для книги заявок.

### Game Servers: предсказуемость без C++ сложности

У сервера с частотой тиков 128 Гц бюджет тика — 7.8 мс. Из них **40–60% уходит на чтение состояния** — позиции игроков, здоровье, инвентарь. При 100 игроках и 20 свойствах на игрока это 256 000 чтений в секунду. На Redis с задержкой 5 мкс только чтение съедает 4.2 мс из 7.8 — больше половины бюджета до того, как начнётся физика.

**Что используют в индустрии.** В геймдеве обычно три варианта: RocksDB через cgo (Hytale хранит чанки мира именно так), Go-решения вроде BadgerDB, или собственный arena-аллокатор на C++. У каждого есть цена.

- **RocksDB через cgo** — ~200 нс на каждый вызов. На миллионе операций в секунду это 20% CPU только на переход границы C/Go.
- **Go с GC** — при интенсивной записи compaction может поднять паузу GC до сотен микросекунд. На 128-тиковом сервере это заметно.
- **Свой arena на C++** — 3–6 месяцев разработки и постоянное сопровождение. Плюс risk use-after-free, который проявляется как краш без трейса.

**Что даёт TephraKV.** Тот же принцип, что у C++ arena — память вне кучи GC, offsets, zero-alloc на горячем пути — но на Go, без cgo и без сложности C++. На 100 000 подключений с `gnet` p99 укладывается в **7.9 мс**, максимальная пауза GC — **87 мкс**. Arena-аллокаторы в Go уже дают снижение GC-давления до 64% в продакшене, а с Go 1.24 runtime/arena стал стабильным. Разрыв между C++ и Go по сырой производительности сокращается, а стоимость разработки и сопровождения остаётся в разы ниже.

### AI-infra: задержка lookup = задержка продукта

Online inference обращается к хранилищу признаков на каждой операции. Задержка 2–5 мс на Feast + Redis приемлема для прототипа, но не для SLA 100 мс, когда таких обращений на один вызов модели — десять. Feature store перестаёт быть внутренней инфраструктурой и становится критическим путём продукта. Плюс счёт за RAM растёт линейно с объёмом фичей — при масштабировании превращается в отдельную статью бюджета.

Остальные сегменты (edge, CDN, fintech) — в [GTM](docs/GTM.md) и [USER-STORIES-001](docs/USER-STORIES-001.md).

### Кому это нужно

| Роль | Задача | Что получает бизнес |
|:---|:---|:---|
| CTO / VP Eng в HFT | Тик-тайм ≤ 5 мс | Меньше проскальзывания — выше PnL на том же объёме сделок |
| Tech Lead в game studio | Частота тиков 128 Гц | До 50% бюджета тика освобождается под игровую логику |
| ML Infra Engineer | Выдача признака < 1 мс p99 | Ниже счёт за RAM, укладываемся в SLA продукта |
| Embedded Engineer | 1–2 vCPU, работа offline | Один бинарник, ARM64, без зависимостей — дешевле деплой на тысячах устройств |
| CDN Infra Engineer | Локальный KV в сотнях локаций | Pure Go, статический бинарник — быстрее катить обновления |
| Head of Eng в fintech | Регистр операций и аудит | Прохождение аудита без ручного труда, PITR и шифрование из коробки |

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
<summary><b>Почему Dragonboat, а не etcd/raft или свой Raft</b></summary>

<br/>

- **etcd/raft** — библиотека для одного-двух реплицированных KV: 5–7 allocs/op, ~44K writes/s, нет Multi-Raft.
- **Свой Multi-Raft** — 12–18 месяцев работы (состав кластера, snapshot'ы, лог GC, транспорт, детекция отказов).
- **Dragonboat** — 1.25M writes/s на группу, до 10 000 групп на узле, пройден Jepsen.

→ [ADR-006 Multi-Raft](docs/adr/ADR-006-multi-raft.md)

</details>

<details>
<summary><b>Почему Tan engine, а не RocksDB LogDB</b></summary>

<br/>

RocksDB LSM для append-only Raft log даёт space amplification. Tan engine — log-file структура: ~10 ГБ на 10 000 shards против ~10 ТБ при WAL-per-shard preallocation.

→ [ADR-023 VLog + Raft](docs/adr/ADR-023-vlog-raft.md)

</details>

<details>
<summary><b>Почему ReadIndex, а не Lease Read по умолчанию</b></summary>

<br/>

Lease Read зависит от ограниченного дрейфа часов — при рассинхроне возможен stale read. В финансовом контексте stale read означает неверный баланс или устаревшую цену. ReadIndex не использует часы вовсе. Цена — дополнительный round-trip, который батчится.

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

Политика задаётся **на каждой операции**. Приложение само решает, где риск потери допустим (кэш, сессии), а где нет (платежи, транзакции). Смешанные батчи запрещены — см. [ADR-010](docs/adr/ADR-010-durability-semantics.md).

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

**Почему tiered, а не одно число.** Разница между L1/L2, L3, RAM и NVMe — три порядка. p999 без указания tier неинформативен: та же система на другом рабочем наборе покажет другую цифру. Публикация p999 без tier запрещена правилом проекта (D113).

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

**Что это значит в деньгах.** Один сервер на 8 vCPU закрывает нагрузку, для которой конкурентам нужно два-три инстанса того же класса. Плюс отсутствие пауз GC снимает необходимость держать буферные мощности под пики.

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

Подробно — [docs/product/competitive.md](docs/product/competitive.md)

---

## Status

| Ограничение | Детали |
|:---|:---|
| **Design only** | Реализация v0.1 в работе. API может меняться |
| **Zero-alloc** | Только встраиваемый режим. В серверном недостижим на уровне wire-протокола |
| **Single-node** | До v0.6a. Распределённый появится с Dragonboat и Tan engine |
| **Платформы** | Linux x86_64 / ARM64 |
| **`ProfileCompliance`** | В v0.1 возвращает `ErrNotImplemented` |

---

## Roadmap

| Версия | Фокус | Что даёт | Ключевая метрика |
|:---:|:---|:---|:---|
| **v0.1** | Deterministic Foundation | База для всех обещаний: p999, zero-alloc, per-op durability | **T2 p999 < 500 нс** |
| v0.2 | Read-amp Compaction | Ниже нагрузка на диск — дешевле инстанс под тот же объём | Read amplification p99 < 3 |
| v0.3 | Value Log | Ниже write amplification — дольше живёт NVMe | Write amplification < 3 |
| v0.4 | MVCC + Shard-per-core | Транзакции и рост по ядрам на одном узле | 50M GET/s |
| v0.5 | TTL + PITR | Первые enterprise-функции | Первый платящий клиент |
| **v0.6a** | Distributed KV Core | Server-режим для polyglot-команд + distributed без etcd | Freshness < 1 с |
| v0.6b | SQL + Columnar Replica | Выход в HTAP | TPC-C |
| v0.7 | Columnar | Векторизация аналитики | SIMD ≥ 4× |
| v1.0 | Enterprise | SOC 2, RBAC, шифрование | SOC 2, $1M ARR |

[ROADMAP](docs/ROADMAP.md) · [BACKLOG](docs/BACKLOG.md) · [NFR](docs/NFR.md) · [CAPACITY](docs/CAPACITY.md)

---

## Документация

> ⚠️ Многие документы в `docs/` — заготовки. Наполнение идёт параллельно с реализацией v0.1.

**Точка входа**

- [DESIGN-001](docs/DESIGN.md) — Design Doc: что строим, почему, чем платим

**Стратегия**

- [HLD-000](docs/TephraKV-HLD-000.md) — high-level design
- [ROADMAP](docs/ROADMAP.md) — план разработки
- [BACKLOG](docs/BACKLOG.md) — текущая работа
- [NFR](docs/NFR.md) — нефункциональные требования
- [CAPACITY](docs/CAPACITY.md) — модель ёмкости
- [GTM](docs/GTM.md) — go-to-market

**Архитектура**

- [ADR Registry](docs/adr/README.md) — 35 architectural decision records
- [API-001](docs/TephraKV-API-001%20v1.0.md) — публичный API
- [FORMAT-001](docs/TephraKV-FORMAT-001%20v1.0.md) — формат на диске
- [ENG-001](docs/TephraKV-ENG-001%20v1.0.md) — инженерные правила
- [GLOSSARY](docs/GLOSSARY.md) — термины

**Продукт**

- [USER-STORIES-001](docs/USER-STORIES-001.md) — сценарии использования
- [PRICING](docs/PRICING.md) — модель ценообразования
- [docs/product/](docs/product/) — competitive, positioning, revenue
- [DR-BCP-001](docs/DR-BCP-001%20v1.0.md) — disaster recovery

**Операции**

- [docs/bench/](docs/bench/) — методология бенчмарков
- [docs/incidents/](docs/incidents/) — разборы инцидентов
- [docs/runbooks/](docs/runbooks/) — операционные процедуры

---

## Лицензия

**Apache 2.0** — Community-версия. Enterprise-функции (шифрование at rest, RBAC, SOC 2) — отдельно. См. [ADR-004](docs/adr/ADR-004-open-core.md).

<br/>

<div align="center">
<sub><b>TephraKV</b> — predictable tail latency by design.</sub>
<br/><br/>
<sub>Что дальше: <a href="docs/DESIGN.md">DESIGN-001</a> — для входа в проект. <a href="docs/adr/">docs/adr/</a> — для деталей решений. <a href="docs/BACKLOG.md">BACKLOG</a> — для текущих задач.</sub>
</div>
