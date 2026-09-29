# TephraKV — API Specification

**Документ:** TephraKV-API-001
**Версия:** 1.0
**Статус:** Черновик
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v7.0, TephraKV-PRD-001 v1.0, TephraKV-ROADMAP-001 v3.0, TephraKV-BACKLOG-001 v5.0, TephraKV-GLOSSARY-001 v2.0, ADR-034 v2.0, ADR-035 v2.0, ADR-010, ADR-020
**Классификация:** Внутренний / публичный

---

## 0. Назначение

API-001 — **полная спецификация** публичного Go API TephraKV. Источник правды для:

- Реализации (какие методы, какие опции).
- Семантики (что гарантирует каждая операция).
- Ошибок (все, с описанием).
- Версионирования (N-1 backward compat).
- Примеров для обоих профилей (Latency, Compliance).

HLD v7.0 §6.1 показывает API **фрагментарно**. API-001 — детализация до уровня, достаточного для реализации FEAT-004, FEAT-007, FEAT-001, FEAT-006.

**Правило:** если метод не в API-001 — его нет. Если семантика не описана — она не гарантирована.

---

## 1. Package

```go
package tephrakv

// Module path: github.com/<user>/tephrakv
// Go version: 1.24
```

---

## 2. Types

### 2.1. DurabilityPolicy

```go
type DurabilityPolicy int

const (
    // NO_SYNC — запись в WAL-буфер, возврат, fsync в окне group commit.
    // Потеря последних N мс (окно group commit) при сбое.
    // v0.1.
    NO_SYNC DurabilityPolicy = iota

    // SYNC_MASTER — запись в WAL, fsync локально, возврат.
    // В distributed (v0.6+) — local durability, НЕ distributed.
    // v0.1.
    SYNC_MASTER

    // SYNC_LEADER — запись в Raft log, fsync локально на лидере,
    // возврат без ожидания кворума. В single-node эквивалентен SYNC_MASTER.
    // В distributed (v0.6+) — fast-path без distributed durability:
    // при смене лидера запись может быть потеряна.
    // v0.1 (single-node), v0.6 (distributed).
    SYNC_LEADER

    // SYNC_MAJORITY — fsync локально + подтверждение кворума через Raft.
    // Distributed durability. v0.6+.
    SYNC_MAJORITY

    // SYNC_ALL — fsync на всех репликах. Максимальная durability,
    // максимальная latency. v0.6+.
    SYNC_ALL
)

func (p DurabilityPolicy) String() string
```

**Семантика (см. ADR-010):**

| Политика | fsync | Кворум | Recovery | Потеря при сбое | Версия |
|---|---|---|---|---|---|
| NO_SYNC | Нет | Нет | WAL replay из буфера | Последние N мс | v0.1 |
| SYNC_MASTER | Да, локально | Нет | WAL replay | Нет (single-node) | v0.1 |
| SYNC_LEADER | Да, локально на лидере | Нет | WAL replay на лидере | Нет (single-node) | v0.1 |
| SYNC_MAJORITY | Да, на кворуме | Да | Raft log | Нет | v0.6 |
| SYNC_ALL | Да, на всех | Да | Raft log | Нет | v0.6 |

**Mixed batch запрещён (ADR-010):**
- Если в батче есть хотя бы одна операция с sync-политикой — весь батч синхронный.
- `WriteBatch` с `WriteOptions{Durability: NO_SYNC}`, содержащий операции с разными политиками, возвращает `ErrMixedDurability`.

### 2.2. Profile

```go
type Profile int

const (
    // ProfileLatency — ниша 1 (HFT, game servers, AI-infra, edge, CDN).
    // Default: NO_SYNC, TTL off, MVCC off, compression none,
    // ReadAmpTarget 3 (v0.1–v0.2) / 2 (v0.3+), ValueLogThreshold 256 Б.
    ProfileLatency Profile = iota

    // ProfileCompliance — ниша 2 (fintech, audit).
    // Default: SYNC_MASTER, MVCC on, PITR on, audit on, encryption on,
    // compression ZSTD, ReadAmpTarget 3.
    // v0.1: возвращает ErrNotImplemented (модули не готовы).
    // v0.4+: MVCC mode. v0.5+: PITR, audit. v0.6+: encryption.
    ProfileCompliance

    // ProfileCustom — ручная настройка. Все поля Options доступны.
    ProfileCustom
)

func (p Profile) String() string
```

**Дефолты по профилям:**

| Параметр | ProfileLatency | ProfileCompliance |
|---|---|---|
| DefaultDurability | NO_SYNC | SYNC_MASTER |
| MemTableSize | 64 МБ | 256 МБ |
| L0CompactionThreshold | 4 | 8 |
| ValueLogThreshold | 256 Б | 64 Б |
| BloomBitsPerKey | 9.6 | 12 |
| Compression | None | ZSTD |
| GroupCommitWindow | 100 мкс | 1 мс |
| ReadAmpTarget | 3 (v0.1–v0.2) / 2 (v0.3+) | 3 |
| WriteAmpTarget | 5 | 3 |
| EnableMVCC | false | true |
| EnablePITR | false | true |
| EnableTTL | false | false |
| EnableAudit | false | true |
| EnableEncryption | false | true |

### 2.3. CompressionType

```go
type CompressionType int

const (
    CompressionNone CompressionType = iota // default ProfileLatency
    CompressionSnappy                       // fast, low ratio
    CompressionZSTD                         // slow, high ratio, default ProfileCompliance
    CompressionLZ4                          // balanced
)

func (c CompressionType) String() string
```

### 2.4. Consistency

```go
type Consistency int

const (
    // ConsistencyReadIndex — ReadIndex (default, clock-free).
    // Heartbeat round к кворуму перед каждым read (с батчингом).
    // Clock skew не влияет. Distributed p99 — миллисекунды.
    // v0.6+.
    ConsistencyReadIndex Consistency = iota

    // ConsistencyLeaseRead — Lease Read (опционально).
    // Лидер записывает timestamp при heartbeat, lease валиден до
    // start + election_timeout / clock_drift_bound.
    // Требует bounded clock drift < 1000 ppm + monotonic raw clock.
    // v0.6+.
    ConsistencyLeaseRead
)
```

### 2.5. WriteOptions

```go
type WriteOptions struct {
    // Durability — политика долговечности для этой операции.
    // Если zero value (NO_SYNC) — используется Options.DefaultDurability.
    Durability DurabilityPolicy

    // TTL — время жизни ключа (v0.5+, ProfileCompliance или EnableTTL).
    // 0 — без TTL.
    TTL time.Duration
}
```

### 2.6. ReadOptions

```go
type ReadOptions struct {
    // Consistency — модель консистентности для distributed read (v0.6+).
    // Single-node: игнорируется.
    // v0.6+.
    Consistency Consistency
}
```

### 2.7. Options

```go
type Options struct {
    // Profile — профиль конфигурации. Заполняет дефолты.
    // ProfileLatency / ProfileCompliance / ProfileCustom.
    Profile Profile

    // --- Core ---

    // MemTableSize — размер MemTable до flush в SSTable.
    // Default: 64 МБ (Latency), 256 МБ (Compliance).
    MemTableSize int64

    // L0CompactionThreshold — количество SSTable в L0 до compaction.
    // Default: 4 (Latency), 8 (Compliance).
    L0CompactionThreshold int

    // ValueLogThreshold — values > threshold → VLog (v0.3+).
    // Default: 256 Б (Latency), 64 Б (Compliance).
    ValueLogThreshold int64

    // BloomBitsPerKey — бит на ключ для Bloom filter.
    // Default: 9.6 (Latency, эталон ScyllaDB), 12 (Compliance).
    BloomBitsPerKey int

    // Compression — тип сжатия SSTable.
    // Default: None (Latency), ZSTD (Compliance).
    Compression CompressionType

    // --- Policy ---

    // DefaultDurability — политика по умолчанию.
    // Default: NO_SYNC (Latency), SYNC_MASTER (Compliance).
    DefaultDurability DurabilityPolicy

    // GroupCommitWindow — окно group commit для NO_SYNC.
    // Default: 100 мкс (Latency), 1 мс (Compliance).
    GroupCommitWindow time.Duration

    // ReadAmpTarget — целевой read amp p99 (disk reads per GET).
    // Default: 3 (v0.1–v0.2), 2 (v0.3+).
    ReadAmpTarget float64

    // WriteAmpTarget — целевой write amp (средняя).
    // Default: 5 (Latency), 3 (Compliance).
    WriteAmpTarget float64

    // --- Modules ---

    // EnableMVCC — multi-version concurrency control (v0.4+).
    // Default: false (Latency), true (Compliance).
    EnableMVCC bool

    // EnablePITR — point-in-time recovery (v0.5+). Требует EnableMVCC.
    // Default: false (Latency), true (Compliance).
    EnablePITR bool

    // EnableTTL — time-to-live (v0.5+).
    // Default: false (Latency), false (Compliance).
    EnableTTL bool

    // EnableAudit — append-only audit log (v0.5+, Compliance).
    // Default: false (Latency), true (Compliance).
    EnableAudit bool

    // EnableEncryption — encryption at rest (v0.6+, Compliance).
    // Default: false (Latency), true (Compliance).
    EnableEncryption bool

    // EncryptionConfig — конфигурация encryption.
    // Требуется, если EnableEncryption == true.
    EncryptionConfig *EncryptionConfig

    // PITRConfig — конфигурация PITR.
    // Требуется, если EnablePITR == true.
    PITRConfig *PITRConfig

    // AuditConfig — конфигурация audit log.
    // Требуется, если EnableAudit == true.
    AuditConfig *AuditConfig

    // TTLConfig — конфигурация TTL.
    // Требуется, если EnableTTL == true.
    TTLConfig *TTLConfig
}
```

### 2.8. Module configs

```go
type EncryptionConfig struct {
    // KeyProvider — источник ключа (KMS, local file).
    KeyProvider KeyProvider

    // KeyRotationInterval — интервал ротации ключа.
    // Default: 30 дней.
    KeyRotationInterval time.Duration
}

type KeyProvider interface {
    // GetKey возвращает текущий ключ шифрования.
    GetKey(ctx context.Context) ([]byte, error)

    // RotateKey инициирует ротацию ключа.
    RotateKey(ctx context.Context) error
}

type PITRConfig struct {
    // Retention — срок хранения WAL archive.
    // Default: 30 дней.
    Retention time.Duration

    // ArchivePath — путь к WAL archive.
    ArchivePath string

    // Compression — сжатие archive.
    // Default: ZSTD.
    Compression CompressionType
}

type AuditConfig struct {
    // LogPath — путь к audit log.
    LogPath string

    // ExportPath — путь экспорта (S3, Glacier).
    // v0.7+.
    ExportPath string

    // Immutable — WORM-режим.
    // Default: true.
    Immutable bool
}

type TTLConfig struct {
    // DefaultTTL — TTL по умолчанию.
    // Default: 0 (без TTL).
    DefaultTTL time.Duration

    // GCInterval — интервал auto-GC expired.
    // Default: 1 минута.
    GCInterval time.Duration
}
```

### 2.9. Metrics

```go
type ExternalMetrics struct {
    // LatencyP50 — 50-й процентиль задержки.
    LatencyP50 time.Duration

    // LatencyP99 — 99-й процентиль.
    LatencyP99 time.Duration

    // LatencyP999 — 99.9-й процентиль.
    LatencyP999 time.Duration

    // ThroughputOpsPerSec — операций в секунду.
    ThroughputOpsPerSec float64

    // CostPerMillionOps — TCO на 1M операций (формула HLD v7.0 §4.6).
    CostPerMillionOps float64

    // FreshnessP99 — от commit на лидере до видимости на columnar replica.
    // v0.6+.
    FreshnessP99 time.Duration
}

type InternalMetrics struct {
    // CyclesPerOp — cycles per operation (rdtsc / cntvct_el0).
    CyclesPerOp uint64

    // BytesReadPerOp — байт прочитано на операцию.
    BytesReadPerOp uint64

    // BytesWrittenPerOp — байт записано на операцию.
    BytesWrittenPerOp uint64

    // AllocsPerOp — аллокаций на операцию.
    AllocsPerOp uint64

    // FlushesPerMillion — флашей на 1M операций.
    FlushesPerMillion uint64

    // CompactionsPerHour — компакций в час.
    CompactionsPerHour uint64

    // ReadAmplification — disk reads per GET.
    ReadAmplification float64

    // WriteAmplification — bytes written / bytes user-written.
    WriteAmplification float64
}
```

### 2.10. Batch

```go
type Batch struct {
    // internal fields (arena-based, offsets)
    // ...
}

// NewBatch создаёт новый Batch.
// Batch переиспользует arena. Reset() для повторного использования.
func NewBatch() *Batch

// Put добавляет операцию PUT в батч.
func (b *Batch) Put(key, value []byte) error

// Delete добавляет операцию DELETE в батч.
func (b *Batch) Delete(key []byte) error

// Reset очищает батч для повторного использования.
func (b *Batch) Reset()

// Len возвращает количество операций в батче.
func (b *Batch) Len() int

// Close освобождает ресурсы батча.
func (b *Batch) Close() error
```

---

## 3. Functions

### 3.1. Open / Close

```go
// Open открывает или создаёт БД по пути path с опциями opts.
//
// Ошибки:
//   ErrInvalidProfile — неизвестный Profile
//   ErrInvalidConfig — некорректная конфигурация (например,
//                     EnablePITR == true, но EnableMVCC == false)
//   ErrNotImplemented — ProfileCompliance в v0.1 (модули не готовы)
//   ErrPathNotFound — путь не существует и не может быть создан
//   ErrPermissionDenied — нет прав на путь
//   ErrCorrupted — WAL/SSTable corrupted, recovery невозможен
//   ErrVersionMismatch — формат данных N+1 (не поддерживается)
func Open(path string, opts Options) (*DB, error)

// Close закрывает БД, flush MemTable, fsync WAL.
// После Close все операции возвращают ErrClosed.
func (db *DB) Close() error
```

### 3.2. PutWithOptions

```go
// PutWithOptions записывает key-value с указанными WriteOptions.
//
// Гарантии:
//   NO_SYNC: запись в WAL-буфер, возврат, fsync в окне group commit.
//   SYNC_MASTER: запись в WAL, fsync локально, возврат.
//   SYNC_LEADER: single-node — эквивалент SYNC_MASTER;
//                distributed — fsync локально на лидере, возврат.
//   SYNC_MAJORITY / SYNC_ALL: v0.6+.
//
// Zero-alloc на hot path (data plane).
//
// Ошибки:
//   ErrClosed — БД закрыта
//   ErrKeyTooLarge — key > 64 Б
//   ErrValueTooLarge — value > 1 МБ
//   ErrNotImplemented — SYNC_MAJORITY / SYNC_ALL в v0.1
//   ErrWriteStall — backpressure, WAL pool exhausted
func (db *DB) PutWithOptions(key, value []byte, opts WriteOptions) error
```

**Ограничения:**
- Key: 16–64 Б (рекомендуется). Max 64 Б.
- Value: 100 Б – 1 МБ (рекомендуется). Max 1 МБ.

### 3.3. Get

```go
// Get читает значение по ключу.
//
// Гарантии:
//   Single-node: linearizable.
//   Distributed (v0.6+): зависит от ReadOptions.Consistency.
//     ConsistencyReadIndex (default): linearizable, clock-free.
//     ConsistencyLeaseRead: linearizable, bounded clock drift.
//
// Zero-alloc на hot path (data plane).
//
// Ошибки:
//   ErrNotFound — ключ не найден
//   ErrClosed — БД закрыта
//   ErrKeyTooLarge — key > 64 Б
//   ErrCorrupted — CRC mismatch при чтении
func (db *DB) Get(key []byte) ([]byte, error)

// GetWithOptions — Get с ReadOptions.
func (db *DB) GetWithOptions(key []byte, opts ReadOptions) ([]byte, error)
```

**Важно:** возвращаемый `[]byte` указывает на внутренний буфер. Не модифицировать. Если нужно сохранить — копировать.

### 3.4. DeleteWithOptions

```go
// DeleteWithOptions удаляет ключ с указанными WriteOptions.
// Семантика durability — как у PutWithOptions.
func (db *DB) DeleteWithOptions(key []byte, opts WriteOptions) error
```

### 3.5. Scan

```go
// Scan обходит ключи в диапазоне [start, end).
//
// fn вызывается для каждой пары key-value. Возврат ошибки из fn
// останавливает Scan и возвращает эту ошибку.
//
// Zero-alloc на hot path (data plane).
//
// Ошибки:
//   ErrClosed — БД закрыта
//   ErrCorrupted — CRC mismatch при чтении
//   любая ошибка из fn
func (db *DB) Scan(start, end []byte, fn func(k, v []byte) error) error
```

### 3.6. WriteBatch

```go
// WriteBatch атомарно применяет все операции батча с указанными WriteOptions.
//
// Гарантии:
//   Если все операции NO_SYNC — асинхронный, fsync в окне group commit.
//   Если есть хотя бы одна sync-политика — весь батч синхронный.
//
// Mixed batch запрещён:
//   WriteBatch с WriteOptions{Durability: NO_SYNC}, содержащий операции
//   с разными политиками, возвращает ErrMixedDurability.
//
// Zero-alloc на hot path (data plane).
//
// Ошибки:
//   ErrMixedDurability — mixed batch
//   ErrClosed — БД закрыта
//   ErrNotImplemented — SYNC_MAJORITY / SYNC_ALL в v0.1
//   ErrWriteStall — backpressure
func (db *DB) WriteBatch(batch *Batch, opts WriteOptions) error
```

### 3.7. Metrics

```go
// Metrics возвращает внешние и внутренние метрики.
//
// Внешние: p50/p99/p999, throughput, cost per million ops, freshness.
// Внутренние: cycles/op, bytes/op, allocs/op, read amp, write amp.
//
// Two-Level Metrics — обязательно в каждом релизе (ADR-020, D7, D22).
//
// Zero-alloc: не гарантируется (control plane).
func (db *DB) Metrics() (ExternalMetrics, InternalMetrics)
```

### 3.8. Compact

```go
// Compact запускает ручную компакцию.
//
// SLA-driven scheduler (ADR-021):
//   Cooldown 60 сек.
//   Гистерезис: старт при read amp p99 > 2, стоп при p99 < 1.5.
//   Bandwidth-aware admission control.
//
// Ошибки:
//   ErrClosed — БД закрыта
//   ErrCompactionInProgress — уже идёт
func (db *DB) Compact() error
```

### 3.9. PITR (v0.5+)

```go
// RestoreTo восстанавливает БД на момент timestamp.
// Требует EnablePITR.
//
// Ошибки:
//   ErrNotImplemented — EnablePITR == false
//   ErrPITRRange — timestamp вне retention
//   ErrClosed — БД закрыта
func (db *DB) RestoreTo(timestamp time.Time) error
```

---

## 4. Ошибки

```go
var (
    // ErrNotFound — ключ не найден.
    ErrNotFound = errors.New("tephrakv: key not found")

    // ErrClosed — БД закрыта.
    ErrClosed = errors.New("tephrakv: db closed")

    // ErrKeyTooLarge — key > 64 Б.
    ErrKeyTooLarge = errors.New("tephrakv: key too large")

    // ErrValueTooLarge — value > 1 МБ.
    ErrValueTooLarge = errors.New("tephrakv: value too large")

    // ErrMixedDurability — mixed batch запрещён (ADR-010).
    ErrMixedDurability = errors.New("tephrakv: mixed durability in batch")

    // ErrInvalidProfile — неизвестный Profile.
    ErrInvalidProfile = errors.New("tephrakv: invalid profile")

    // ErrInvalidConfig — некорректная конфигурация.
    ErrInvalidConfig = errors.New("tephrakv: invalid config")

    // ErrNotImplemented — фича не реализована в этой версии.
    ErrNotImplemented = errors.New("tephrakv: not implemented")

    // ErrPathNotFound — путь не существует.
    ErrPathNotFound = errors.New("tephrakv: path not found")

    // ErrPermissionDenied — нет прав на путь.
    ErrPermissionDenied = errors.New("tephrakv: permission denied")

    // ErrCorrupted — WAL/SSTable corrupted.
    ErrCorrupted = errors.New("tephrakv: corrupted data")

    // ErrVersionMismatch — формат данных N+1.
    ErrVersionMismatch = errors.New("tephrakv: version mismatch")

    // ErrWriteStall — backpressure, WAL pool exhausted.
    ErrWriteStall = errors.New("tephrakv: write stall")

    // ErrCompactionInProgress — компакция уже идёт.
    ErrCompactionInProgress = errors.New("tephrakv: compaction in progress")

    // ErrPITRRange — timestamp вне retention.
    ErrPITRRange = errors.New("tephrakv: timestamp out of PITR range")
)
```

**Правило:** ошибки — sentinel values. `errors.Is` работает. Обёртки через `fmt.Errorf("...: %w", err)`.

---

## 5. Семантика операций (ADR-034 reference)

| Операция | Durability | Consistency | Isolation | Zero-alloc |
|---|---|---|---|---|
| PutWithOptions | per-op (WriteOptions) | — | — | Да |
| Get | — | single-node: linearizable; distributed: ReadIndex/LeaseRead | — | Да |
| DeleteWithOptions | per-op | — | — | Да |
| Scan | — | snapshot на момент начала | — | Да |
| WriteBatch | per-op (whole batch) | — | atomic | Да |
| Metrics | — | — | — | Нет (control plane) |
| Compact | — | — | — | Нет (control plane) |
| RestoreTo | — | — | — | Нет (control plane) |

**ADR-034 reference:** ADR-034 v2.0 §2.11 (Zero-alloc scope), §2.12 (Shared WAL pool).

---

## 6. Версионирование API

### 6.1. Semver

- **Major** — несовместимое изменение API/формата.
- **Minor** — новая фича (обратно совместимая).
- **Patch** — bugfix.

### 6.2. N-1 backward compat

Формат данных (WAL, SSTable, manifest, wire) — N-1 backward compat:

- Reader v0.N читает формат v0.(N-1).
- Reader v0.N отказывает формат v0.(N+1) с `ErrVersionMismatch`.
- Writer v0.N пишет формат v0.N (не v0.(N-1)).

### 6.3. Deprecation

- Deprecated поле/метод — помечается в godoc.
- Удаляется через 2 minor версии.
- CHANGELOG фиксирует deprecation.

---

## 7. Примеры

### 7.1. Ниша 1: latency-critical (HFT, game servers)

```go
package main

import (
    "fmt"
    "log"

    "github.com/<user>/tephrakv"
)

func main() {
    db, err := tephrakv.Open("/data", tephrakv.Options{
        Profile: tephrakv.ProfileLatency,
        // Defaults:
        //   DefaultDurability: NO_SYNC
        //   MemTableSize: 64 МБ
        //   ValueLogThreshold: 256 Б
        //   BloomBitsPerKey: 9.6
        //   Compression: None
        //   ReadAmpTarget: 3 (v0.1–v0.2) / 2 (v0.3+)
    })
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // NO_SYNC: максимальная скорость, потеря последних N мс при сбое.
    err = db.PutWithOptions(
        []byte("order:12345"),
        []byte("BUY 100 AAPL @ 150.00"),
        tephrakv.WriteOptions{Durability: tephrakv.NO_SYNC},
    )
    if err != nil {
        log.Fatal(err)
    }

    // SYNC_MASTER: durability, p99 PUT < 5 мс.
    err = db.PutWithOptions(
        []byte("trade:67890"),
        []byte("EXECUTED"),
        tephrakv.WriteOptions{Durability: tephrakv.SYNC_MASTER},
    )
    if err != nil {
        log.Fatal(err)
    }

    val, err := db.Get([]byte("order:12345"))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(string(val))

    ext, int := db.Metrics()
    fmt.Printf("p999 GET: %v\n", ext.LatencyP999)
    fmt.Printf("allocs/op: %d\n", int.AllocsPerOp)
}
```

### 7.2. Ниша 2: fintech / audit (v0.4+)

```go
package main

import (
    "context"
    "log"
    "time"

    "github.com/<user>/tephrakv"
)

func main() {
    db, err := tephrakv.Open("/data", tephrakv.Options{
        Profile: tephrakv.ProfileCompliance,
        // Defaults:
        //   DefaultDurability: SYNC_MASTER
        //   EnableMVCC: true
        //   EnablePITR: true
        //   EnableAudit: true
        //   EnableEncryption: true
        //   Compression: ZSTD

        EncryptionConfig: &tephrakv.EncryptionConfig{
            KeyProvider:         kms.NewAWSKMS("arn:aws:kms:..."),
            KeyRotationInterval: 30 * 24 * time.Hour,
        },
        PITRConfig: &tephrakv.PITRConfig{
            Retention:   30 * 24 * time.Hour,
            ArchivePath: "/data/wal-archive",
            Compression: tephrakv.CompressionZSTD,
        },
        AuditConfig: &tephrakv.AuditConfig{
            LogPath:   "/data/audit",
            Immutable: true,
        },
    })
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // SYNC_MASTER: durability, audit log.
    err = db.PutWithOptions(
        []byte("ledger:2026-09-22:entry:001"),
        []byte(`{"amount": 1000.00, "currency": "USD", "from": "...", "to": "..."}`),
        tephrakv.WriteOptions{Durability: tephrakv.SYNC_MASTER},
    )
    if err != nil {
        log.Fatal(err)
    }

    // PITR: восстановление на момент в прошлом.
    if err := db.RestoreTo(time.Date(2026, 9, 21, 12, 0, 0, 0, time.UTC)); err != nil {
        log.Fatal(err)
    }
}
```

### 7.3. Custom profile (ручная настройка)

```go
db, err := tephrakv.Open("/data", tephrakv.Options{
    Profile:              tephrakv.ProfileCustom,
    DefaultDurability:    tephrakv.SYNC_MASTER,
    MemTableSize:         128 * 1024 * 1024, // 128 МБ
    ValueLogThreshold:    512,                // 512 Б
    BloomBitsPerKey:      10,
    Compression:          tephrakv.CompressionLZ4,
    ReadAmpTarget:        2,
    WriteAmpTarget:       4,
    GroupCommitWindow:    500 * time.Microsecond,
    EnableMVCC:           true,
    EnableTTL:            true,
    TTLConfig: &tephrakv.TTLConfig{
        DefaultTTL: 24 * time.Hour,
        GCInterval: time.Minute,
    },
})
```

### 7.4. Batch

```go
batch := tephrakv.NewBatch()
defer batch.Close()

for i := 0; i < 1000; i++ {
    key := fmt.Sprintf("key:%06d", i)
    val := fmt.Sprintf("value:%d", i)
    if err := batch.Put([]byte(key), []byte(val)); err != nil {
        log.Fatal(err)
    }
}

// Все операции NO_SYNC → асинхронный батч.
if err := db.WriteBatch(batch, tephrakv.WriteOptions{
    Durability: tephrakv.NO_SYNC,
}); err != nil {
    log.Fatal(err)
}
```

### 7.5. Mixed batch — запрещён

```go
// Это вернёт ErrMixedDurability:
err := db.WriteBatch(batch, tephrakv.WriteOptions{
    Durability: tephrakv.NO_SYNC, // но в батче есть SYNC_MASTER операции
})
// err == tephrakv.ErrMixedDurability
```

### 7.6. Scan

```go
err := db.Scan([]byte("order:"), []byte("order;"), func(k, v []byte) error {
    fmt.Printf("%s = %s\n", k, v)
    return nil
})
if err != nil {
    log.Fatal(err)
}
```

---

## 8. Zero-alloc контракт (детально)

**См. HLD v7.0 §3.2, ADR-009 v2, D3, D68, D69.**

| Метод | Zero-alloc? | Граница | Примечание |
|---|---|---|---|
| PutWithOptions | Да | data plane | WAL write, MemTable insert |
| Get | Да | data plane | MemTable lookup, SST read |
| DeleteWithOptions | Да | data plane | tombstones |
| Scan | Да | data plane | iterator |
| WriteBatch | Да | data plane | batch apply |
| NewBatch | Да | data plane | arena-based |
| Batch.Put | Да | data plane | offsets |
| Batch.Delete | Да | data plane | offsets |
| Batch.Reset | Да | data plane | arena reset |
| Metrics | Нет | control plane | HDR histogram |
| Compact | Нет | control plane | compaction worker |
| Open | Нет | control plane | mmap, manifest |
| Close | Нет | control plane | flush, fsync |
| RestoreTo | Нет | control plane | PITR |

**Гарантия:** CI-бенчмарк с `-benchmem` на каждой версии Go (D69).

---

## 9. Транспорт (v0.6+)

### 9.1. gRPC codec (CodecV2)

```go
// CodecV2 — кастомный gRPC codec, не protobuf (D70, D86).
// Регистрируется при старте server mode.
//
// Zero-alloc на message codec (payload) через SharedBufferPool.
// Framing — control plane, аллокации допустимы.
//
// Бенчмарк (4 КБ message):
//   Unmarshal ns/op: V1 bridge 174, V2 78 (2.4× быстрее).
//   Marshal ns/op: V1 bridge 728, V2 268 (2.7× быстрее).
//   B/op (marshal): V1 486, V2 ~1.6 (~300× меньше).
```

### 9.2. Transport N Raft-групп на 1 stream

```go
// Streams per connection: 100–250 (не default 100, не 10000, D63).
// MaxConcurrentStreams: 100–250.
// Memory per connection: < 20 МБ (BR-10).
//
// N Raft-групп на 1 stream. Не 1 stream = 1 Raft group.
// Отдельный stream для snapshot transfer.
// Отдельный stream для heartbeat.
```

---

## 10. Open questions

- **Get возвращает `[]byte` из внутреннего буфера.** Как это документировать, чтобы пользователи не модифицировали? Вариант: возвращать `const` или явно предупреждать в godoc.
- **Batch.Reset.** Как гарантировать, что пользователь не использует Batch после Reset? Вариант: `Closed` flag.
- **TTL в WriteOptions.** Как обрабатывать TTL для ProfileCompliance (EnableTTL == false)? Вариант: `ErrNotImplemented`.
- **Consistency в ReadOptions.** Игнорируется в single-node. Ошибка или silent? Вариант: silent, документировать.
- **Metrics в hot path.** `db.Metrics()` не zero-alloc. Как гарантировать, что пользователь не вызывает его в hot path? Вариант: документировать, benchmark warning.
- **RestoreTo в running DB.** Блокирует ли DB? Вариант: требует Close → Open.
- **CodecV2 gate.** Как именно проверять, что используется V2, а не V1 bridge? Reflection + benchmark. Не решено.
- **ProfileCompliance в v0.1.** Возвращает `ErrNotImplemented`. Как тестировать? Не решено.

**Правило:** неизвестное документируется как неизвестное.

---

## 11. Связанные документы

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v7.0 | Архитектура, §6.1 (API фрагмент), §3.2 (zero-alloc), §6.8 (CodecV2), §6.7 (shared WAL) |
| TephraKV-PRD-001 v1.0 | ICP, use cases |
| TephraKV-ROADMAP-001 v3.0 | §5 (детализация версий) |
| TephraKV-BACKLOG-001 v5.0 | §2 (FEAT-004, FEAT-007, FEAT-001, FEAT-006) |
| TephraKV-GLOSSARY-001 v2.0 | Термины |
| ADR-009 v2 | Zero-alloc scope: framing vs codec |
| ADR-010 | Durability semantics |
| ADR-020 | Metrics definitions |
| ADR-034 v2.0 | Engineering practices |
| ADR-035 v2.0 | Capacity model |
| ADR-036 | Read path: ReadIndex + Lease Read |
| ADR-037 (proposed) | Profile semantics |
| FORMAT-001 | Data Format Specification (критично до кода) |

---

## 12. Что дальше

1. **Утвердить API-001 v1.0** → статус `Approved`.
2. **Написать FORMAT-001** — критично до кода.
3. **Синхронизировать с ADR-034 v2.0** — API semantics per operation.
4. **Синхронизировать с ADR-037 (proposed)** — Profile semantics.
5. **Написать godoc** — из API-001.
6. **Начать CHORE-001** — bootstrap.

**Правило:** если метод не в API-001 — его нет.

---

**Конец TephraKV-API-001 v1.0**