# TephraKV — Data Format Specification

**Документ:** TephraKV-FORMAT-001
**Версия:** 1.0
**Статус:** Черновик
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v7.0, TephraKV-API-001 v1.0, TephraKV-ROADMAP-001 v3.0, TephraKV-BACKLOG-001 v5.0, TephraKV-GLOSSARY-001 v2.0, ADR-002, ADR-012 v3, ADR-029 v2, ADR-005 v4
**Классификация:** Внутренний / публичный

---

## 0. Назначение

FORMAT-001 — **полная спецификация форматов данных** TephraKV. Источник правды для:

- Реализации writer/reader для WAL, SSTable, Manifest, VLog.
- Wire format для gRPC message codec (CodecV2).
- Версионирования (N-1 backward compat).
- Recovery (как читать после сбоя).
- Debug (hex dump, анализ corrupted файлов).

HLD v7.0 §6.9 упоминает wire format. FORMAT-001 — детализация до уровня, достаточного для реализации FEAT-003 (WAL), FEAT-005a/b (SSTable), FEAT-043/044 (gRPC transport).

**Правило:** если формат не в FORMAT-001 — его нет. Если версия не описана — она не поддерживается.

---

## 1. Общие принципы

### 1.1. Endianness

**Little-endian** для всех числовых полей. Обоснование: x86_64 и ARM64 (LE) — целевые платформы (HLD v7.0 §5.1). Big-endian не поддерживается.

### 1.2. CRC

**CRC32C** (Castagnoli) для всех checksums. Обоснование: hardware-accelerated на x86_64 (SSE4.2) и ARM64 (CRC32 extension). ADR-002.

### 1.3. Версионирование

Каждый формат имеет **version byte** в заголовке.

- **N-1 backward compat:** reader v0.N читает формат v0.(N-1).
- **N+1 — reject:** reader v0.N отказывает формат v0.(N+1) с `ErrVersionMismatch`.
- **Writer пишет N:** writer v0.N пишет формат v0.N (не N-1).

### 1.4. Zero-alloc на hot path

Форматы, читаемые/пишущиеся на hot path (WAL write, SST read, message codec), работают на **preallocated буферах из arena**. Zero-alloc контракт (HLD v7.0 §3.2, D3).

### 1.5. Offsets vs pointers

Все ссылки внутри форматов — **offsets**, не pointers. Обоснование: arena-friendly, GC-friendly, переносимо между mmap-сегментами. HLD v7.0 §5.4 (принцип 3).

---

## 2. WAL Record Format

**Ссылки:** ADR-002, ADR-012 v3, HLD v7.0 §6.7, D53, D62.

### 2.1. Shared WAL pool

Один WAL pool на узел. Preallocated сегменты (1 ГБ). Rotation. **Per-shard LSN namespace:** LSN монотонен внутри shard, между shard не координируется.

**Формат сегмента:**

```
+------------------------------------------------------------+
| Segment Header (64 Б)                                       |
+------------------------------------------------------------+
| Record 1                                                    |
+------------------------------------------------------------+
| Record 2                                                    |
+------------------------------------------------------------+
| ...                                                         |
+------------------------------------------------------------+
| Padding до 1 ГБ                                             |
+------------------------------------------------------------+
| Segment Footer (32 Б)                                       |
+------------------------------------------------------------+
```

### 2.2. Segment Header

```
+--------+--------+--------+--------+--------+--------+--------+--------+
| Magic  | Version| Flags  | ShardID| SegID  | StartLSN|        | CRC32C |
| 4 Б    | 1 Б    | 1 Б    | 2 Б    | 4 Б    | 8 Б     | 12 Б   | 4 Б    |
+--------+--------+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| Magic | 4 Б | `0x54_4B_56_57` ("TKVW") |
| Version | 1 Б | 0x01 для v0.1 |
| Flags | 1 Б | bit 0: compressed, bit 1: encrypted, bit 2-7: reserved |
| ShardID | 2 Б | ID shard (0 для single-node v0.1) |
| SegID | 4 Б | ID сегмента (монотонный) |
| StartLSN | 8 Б | LSN первой записи в сегменте |
| Reserved | 12 Б | Нули (для будущего) |
| CRC32C | 4 Б | CRC заголовка |

**Итого:** 4 + 1 + 1 + 2 + 4 + 8 + 12 + 4 = **36 Б** + padding до 64 Б.

### 2.3. Record

```
+--------+--------+--------+--------+--------+--------+--------+--------+
| CRC32C | LSN    | Term   | Index  | Type   | KeyLen | ValLen | Flags  |
| 4 Б    | 8 Б    | 8 Б    | 8 Б    | 1 Б    | 2 Б    | 4 Б    | 1 Б    |
+--------+--------+--------+--------+--------+--------+--------+--------+
| Key (KeyLen Б)                                                     |
+--------------------------------------------------------------------+
| Value (ValLen Б)                                                   |
+--------------------------------------------------------------------+
```

| Поле | Размер | Описание |
|---|---|---|
| CRC32C | 4 Б | CRC всего record (header + key + value) |
| LSN | 8 Б | Log Sequence Number, монотонный per shard |
| Term | 8 Б | Raft term (0 в single-node v0.1) |
| Index | 8 Б | Raft index (0 в single-node v0.1) |
| Type | 1 Б | 0x01 PUT, 0x02 DELETE, 0x03 BATCH_START, 0x04 BATCH_END |
| KeyLen | 2 Б | Длина ключа (max 64) |
| ValLen | 4 Б | Длина значения (max 1 МБ) |
| Flags | 1 Б | bit 0: tombstone, bit 1: TTL present, bit 2-7: reserved |
| Key | KeyLen Б | Ключ |
| Value | ValLen Б | Значение |

**Итого header:** 4 + 8 + 8 + 8 + 1 + 2 + 4 + 1 = **36 Б**.

**Alignment:** не требуется (формат streaming).

**Term/Index поля:** включены для совместимости v0.1 → v0.6. В single-node v0.1 — нули. В distributed v0.6 — Raft term/index. HLD v7.0 §6.7.

### 2.4. Segment Footer

```
+--------+--------+--------+--------+--------+--------+--------+--------+
| Magic  | Count  | LastLSN|        |        |        |        | CRC32C |
| 4 Б    | 8 Б    | 8 Б    |        |        |        |        | 4 Б    |
+--------+--------+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| Magic | 4 Б | `0x54_4B_57_46` ("TKWF") |
| Count | 8 Б | Количество записей |
| LastLSN | 8 Б | LSN последней записи |
| Reserved | 8 Б | Нули |
| CRC32C | 4 Б | CRC заголовка |

**Итого:** 4 + 8 + 8 + 8 + 4 = **32 Б**.

### 2.5. Partial write

При сбое последняя запись может быть обрезана:

- Reader читает record header (36 Б).
- Если header не полностью (EOF) — обрезать.
- Если CRC header != expected — обрезать.
- Если key/value не полностью — обрезать.
- Если CRC record != expected — обрезать.

Всё после обрезанной записи — игнорируется (recovery восстанавливает до последнего fsync). BENCH-007.

### 2.6. Group commit

Group commit coordinator батчит до 64 записей или окно 100 мкс. Записи пишутся последовательно в сегмент. Один fsync на батч. ADR-002.

### 2.7. Rotation

Сегмент 1 ГБ. При заполнении — новый сегмент. Старые сегменты удаляются после flush в SSTable + retention. ADR-002.

---

## 3. SSTable Block Format

**Ссылки:** ADR-003, FEAT-005a/b, HLD v7.0 §5.4.

### 3.1. Структура файла

```
+------------------------------------------------------------+
| SSTable Header (128 Б)                                      |
+------------------------------------------------------------+
| Data Block 1                                                |
+------------------------------------------------------------+
| Data Block 2                                                |
+------------------------------------------------------------+
| ...                                                         |
+------------------------------------------------------------+
| Data Block N                                                |
+------------------------------------------------------------+
| Index Block                                                 |
+------------------------------------------------------------+
| Bloom Filter                                                |
+------------------------------------------------------------+
| Footer (64 Б)                                               |
+------------------------------------------------------------+
```

### 3.2. SSTable Header

```
+--------+--------+--------+--------+--------+--------+
| Magic  | Version| Level  | Flags  |        | CRC32C |
| 4 Б    | 1 Б    | 1 Б    | 1 Б    | 117 Б  | 4 Б    |
+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| Magic | 4 Б | `0x54_4B_53_54` ("TKST") |
| Version | 1 Б | 0x01 для v0.1 |
| Level | 1 Б | LSM level (0–7) |
| Flags | 1 Б | bit 0: compressed, bit 1: VLog pointers, bit 2-7: reserved |
| Reserved | 117 Б | Нули |
| CRC32C | 4 Б | CRC заголовка |

**Итого:** 4 + 1 + 1 + 1 + 117 + 4 = **128 Б**.

### 3.3. Data Block

```
+--------+--------+--------+--------+--------+
| Count  | Flags  |        | CRC32C | Entries|
| 2 Б    | 1 Б    | 1 Б    | 4 Б    | var    |
+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| Count | 2 Б | Количество entries (max 65535) |
| Flags | 1 Б | bit 0: compressed, bit 1: has TTL |
| Restart | 1 Б | Точка рестарта для binary search |
| CRC32C | 4 Б | CRC заголовка + entries |
| Entries | var | Последовательность entries |

**Block size:** 4 КБ (конфигурируемый). FEAT-005a.

### 3.4. Entry (v0.1–v0.2, value в LSM)

```
+--------+--------+--------+--------+--------+--------+
| KeyLen | Key    | ValLen | Value  | Flags  | TTL    |
| 2 Б    | var    | 4 Б    | var    | 1 Б    | 8 Б    |
+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| KeyLen | 2 Б | Длина ключа |
| Key | KeyLen Б | Ключ |
| ValLen | 4 Б | Длина значения |
| Value | ValLen Б | Значение |
| Flags | 1 Б | bit 0: tombstone, bit 1: TTL present |
| TTL | 8 Б | Unix timestamp (ns), если flags bit 1 |

### 3.5. Entry (v0.3+, VLog pointers)

```
+--------+--------+--------+--------+--------+--------+--------+
| KeyLen | Key    | VLogPtr| ValLen | Flags  | TTL    |        |
| 2 Б    | var    | 12 Б   | 4 Б    | 1 Б    | 8 Б    |        |
+--------+--------+--------+--------+--------+--------+--------+
```

**VLogPtr (12 Б):**

| Поле | Размер | Описание |
|---|---|---|
| SegID | 4 Б | ID сегмента VLog |
| Offset | 8 Б | Offset в сегменте |

**Итого VLogPtr:** 12 Б. Value хранится в VLog (ADR-022).

### 3.6. Index Block

Разреженный индекс: одна запись на каждые 1000 data block.

```
+--------+--------+--------+--------+--------+
| Count  |        |        | CRC32C | Entries|
| 4 Б    | 4 Б    | 4 Б    | 4 Б    | var    |
+--------+--------+--------+--------+--------+
```

**Index Entry:**

```
+--------+--------+--------+--------+
| KeyLen | Key    | Offset | Size   |
| 2 Б    | var    | 8 Б    | 4 Б    |
+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| KeyLen | 2 Б | Длина минимального ключа в block |
| Key | KeyLen Б | Минимальный ключ |
| Offset | 8 Б | Offset block в файле |
| Size | 4 Б | Размер block |

**Partition index (v0.2+):** ADR-003, D10. Min/max ключ для каждой партиции.

**MinMax block index (v0.2+):** ADR-003, D11. Per partition, не per block.

### 3.7. Bloom Filter

```
+--------+--------+--------+--------+--------+
| Magic  | Bits   | K      |        | CRC32C |
| 4 Б    | 4 Б    | 1 Б    | 3 Б    | 4 Б    |
+--------+--------+--------+--------+--------+
| Bit array (Bits/8 Б)                       |
+--------------------------------------------+
```

| Поле | Размер | Описание |
|---|---|---|
| Magic | 4 Б | `0x54_4B_42_46` ("TKBF") |
| Bits | 4 Б | Количество бит |
| K | 1 Б | Количество hash-функций |
| Reserved | 3 Б | Нули |
| CRC32C | 4 Б | CRC заголовка + bit array |
| Bit array | Bits/8 Б | Битовая карта |

**Bits/key:** 9.6 (эталон ScyllaDB). FP rate < 1%. ADR-003, D9.

**Память:** при 100M keys × 7 levels = **840 МБ** (не 8.4 ГБ — исправление арифметической ошибки HLD v5.1 §5.4).

### 3.8. Footer

```
+--------+--------+--------+--------+--------+--------+
| Magic  | Version| IndexO | BloomO |        | CRC32C |
| 4 Б    | 1 Б    | 8 Б    | 8 Б    | 39 Б   | 4 Б    |
+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| Magic | 4 Б | `0x54_4B_46_54` ("TKFT") |
| Version | 1 Б | 0x01 для v0.1 |
| IndexOffset | 8 Б | Offset index block |
| BloomOffset | 8 Б | Offset bloom filter |
| Reserved | 39 Б | Нули |
| CRC32C | 4 Б | CRC заголовка |

**Итого:** 4 + 1 + 8 + 8 + 39 + 4 = **64 Б**.

### 3.9. Versioning N-1

- Reader v0.N читает формат v0.(N-1).
- Reader v0.N отказывает формат v0.(N+1) с `ErrVersionMismatch`.
- REF-003: SST footer versioning.

---

## 4. Manifest Format

**Ссылки:** HLD v7.0 §5.1.

### 4.1. Append-only + snapshot compaction

Manifest — append-only файл с периодическим snapshot compaction. Atomic swap при компакции.

### 4.2. Manifest Entry

```
+--------+--------+--------+--------+--------+--------+
| CRC32C | Type   | Level  |        | SSTID  | Size   |
| 4 Б    | 1 Б    | 1 Б    | 2 Б    | 8 Б    | 8 Б    |
+--------+--------+--------+--------+--------+--------+
| MinKey (KeyLen Б)                                  |
+----------------------------------------------------+
| MaxKey (KeyLen Б)                                  |
+----------------------------------------------------+
```

| Поле | Размер | Описание |
|---|---|---|
| CRC32C | 4 Б | CRC entry |
| Type | 1 Б | 0x01 ADD, 0x02 REMOVE, 0x03 COMPACT |
| Level | 1 Б | LSM level |
| KeyLen | 2 Б | Длина min/max ключа |
| SSTID | 8 Б | ID SSTable |
| Size | 8 Б | Размер SSTable в байтах |
| MinKey | KeyLen Б | Минимальный ключ |
| MaxKey | KeyLen Б | Максимальный ключ |

### 4.3. Snapshot Compaction

При snapshot compaction:

1. Читать все entries.
2. Собрать текущее состояние (какие SSTable существуют).
3. Записать новый manifest во временный файл.
4. fsync.
5. Atomic rename (POSIX rename).
6. Удалить старый manifest.

**Atomic swap:** rename атомарен на POSIX. REF-005.

---

## 5. VLog Pointer Format

**Ссылки:** ADR-022, ADR-023, HLD v7.0 §6.5.

### 5.1. VLog Segment

```
+------------------------------------------------------------+
| VLog Segment Header (64 Б)                                  |
+------------------------------------------------------------+
| Record 1                                                    |
+------------------------------------------------------------+
| Record 2                                                    |
+------------------------------------------------------------+
| ...                                                         |
+------------------------------------------------------------+
```

### 5.2. VLog Segment Header

```
+--------+--------+--------+--------+--------+--------+
| Magic  | Version| Flags  | SegID  |        | CRC32C |
| 4 Б    | 1 Б    | 1 Б    | 4 Б    | 50 Б   | 4 Б    |
+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| Magic | 4 Б | `0x54_4B_56_4C` ("TKVL") |
| Version | 1 Б | 0x01 для v0.3 |
| Flags | 1 Б | bit 0: compressed, bit 1: encrypted |
| SegID | 4 Б | ID сегмента |
| Reserved | 50 Б | Нули |
| CRC32C | 4 Б | CRC заголовка |

**Итого:** 4 + 1 + 1 + 4 + 50 + 4 = **64 Б**.

### 5.3. VLog Record

```
+--------+--------+--------+--------+--------+--------+
| CRC32C | ValLen | Flags  |        | Value  |        |
| 4 Б    | 4 Б    | 1 Б    | 3 Б    | var    |        |
+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| CRC32C | 4 Б | CRC record |
| ValLen | 4 Б | Длина значения |
| Flags | 1 Б | bit 0: tombstone |
| Reserved | 3 Б | Нули |
| Value | ValLen Б | Значение |

### 5.4. Pointer в LSM

См. §3.5: SegID (4 Б) + Offset (8 Б) = 12 Б.

### 5.5. Titan-style WriteCallback

**Ссылки:** ADR-022, D87.

GC-compatible snapshots: WriteCallback пишет в Blob file, LSM обновляется атомарно. Titan controls write amplification through Blob file partitioning and targeted GC only on high-garbage-density files.

---

## 6. Wire Format (gRPC Message Codec)

**Ссылки:** ADR-029 v2, ADR-005 v4, HLD v7.0 §6.8, §6.9, D39, D70, D86.

### 6.1. Общие принципы

- **Кастомный gRPC codec, CodecV2 + SharedBufferPool, не protobuf.**
- Zero-alloc на message codec (payload) — hot path.
- Framing (HTTP/2 frames) — control plane.
- Версионирование с первого дня (N-1 backward compat).
- Encode/decode на preallocated буферах из arena (SharedBufferPool).

### 6.2. Message Header

```
+--------+--------+--------+--------+--------+--------+
| Magic  | Version| Type   | Flags  |        | CRC32C |
| 2 Б    | 1 Б    | 1 Б    | 1 Б    | 3 Б    | 4 Б    |
+--------+--------+--------+--------+--------+--------+
```

| Поле | Размер | Описание |
|---|---|---|
| Magic | 2 Б | `0x54_4B` ("TK") |
| Version | 1 Б | 0x01 для v0.6 |
| Type | 1 Б | Тип сообщения (см. §6.3) |
| Flags | 1 Б | bit 0: compressed, bit 1: encrypted |
| Reserved | 3 Б | Нули |
| CRC32C | 4 Б | CRC заголовка |

**Итого header:** 12 Б.

### 6.3. Message Types

| Type | Название | Raft | Описание |
|---|---|---|---|
| 0x01 | AppendEntries | Да | Log replication |
| 0x02 | AppendEntriesResponse | Да | Ответ |
| 0x03 | RequestVote | Да | Election |
| 0x04 | RequestVoteResponse | Да | Ответ |
| 0x05 | InstallSnapshot | Да | Snapshot transfer |
| 0x06 | InstallSnapshotResponse | Да | Ответ |
| 0x07 | Heartbeat | Да | Heartbeat (отдельный stream) |
| 0x08 | HeartbeatResponse | Да | Ответ |
| 0x09 | ReadIndex | Да | ReadIndex round |
| 0x0A | ReadIndexResponse | Да | Ответ |
| 0x0B | ClientPut | Нет | Клиентская операция |
| 0x0C | ClientPutResponse | Нет | Ответ |
| 0x0D | ClientGet | Нет | Клиентская операция |
| 0x0E | ClientGetResponse | Нет | Ответ |
| 0x0F | Error | Нет | Ошибка |

### 6.4. AppendEntries Payload

```
+--------+--------+--------+--------+--------+--------+
| Term   | LeaderID| PrevLogIdx| PrevLogTerm|        |
| 8 Б    | 8 Б    | 8 Б       | 8 Б        |        |
+--------+--------+--------+--------+--------+--------+
| LeaderCommit| EntryCount| Entries (var)         |
| 8 Б         | 4 Б       |                       |
+-------------+-----------+-----------------------+
```

**Entry (в AppendEntries):**

```
+--------+--------+--------+--------+--------+--------+
| Term   | Index  | Type   | KeyLen | ValLen | Flags  |
| 8 Б    | 8 Б    | 1 Б    | 2 Б    | 4 Б    | 1 Б    |
+--------+--------+--------+--------+--------+--------+
| Key (KeyLen Б)                                     |
+----------------------------------------------------+
| Value (ValLen Б)                                   |
+----------------------------------------------------+
```

### 6.5. RequestVote Payload

```
+--------+--------+--------+--------+--------+--------+
| Term   | CandidateID| LastLogIdx| LastLogTerm|      |
| 8 Б    | 8 Б       | 8 Б       | 8 Б        |      |
+--------+--------+--------+--------+--------+--------+
```

### 6.6. InstallSnapshot Payload

```
+--------+--------+--------+--------+--------+--------+
| Term   | LeaderID| LastIdx| LastTerm| Offset | Data  |
| 8 Б    | 8 Б    | 8 Б    | 8 Б     | 8 Б    | var   |
+--------+--------+--------+--------+--------+--------+
| Done (1 Б)                                         |
+----------------------------------------------------+
```

**Incremental, chunked, resume.** Snapshot immutable на время transfer. При смене лидера — новый snapshot, resume с начала. ADR-014.

### 6.7. CodecV2 API

```go
// CodecV2 — кастомный gRPC codec, не protobuf.
// Регистрируется через encoding.RegisterCodecV2.
//
// Marshal: encode в preallocated буфер из SharedBufferPool.
// Unmarshal: decode из transport buffer (MaterializeToBuffer — no copy).
```

**Бенчмарк (4 КБ message, D86):**

| Метрика | V1 bridge | V2 | Улучшение |
|---|---|---|---|
| Unmarshal ns/op | 174 | 78 | 2.4× быстрее |
| Marshal ns/op | 728 | 268 | 2.7× быстрее |
| B/op (marshal) | 486 | ~1.6 | ~300× меньше |

### 6.8. Streams

- **N Raft-групп на 1 stream** (100–250 streams на соединение, D63).
- Не 1 stream = 1 Raft group.
- Отдельный stream для snapshot transfer.
- Отдельный stream для heartbeat.
- MaxConcurrentStreams: 100–250.
- Memory per connection: < 20 МБ (BR-10, ADR-035 v2.0 §5.7).

### 6.9. Versioning N-1

- Reader v0.N читает формат v0.(N-1).
- Reader v0.N отказывает формат v0.(N+1) с `ErrVersionMismatch`.
- Writer пишет N.

---

## 7. Zero-alloc контракт по форматам

| Формат | Hot path? | Zero-alloc? | Примечание |
|---|---|---|---|
| WAL record (write) | Да | Да | Preallocated сегменты |
| WAL record (read) | Нет (recovery) | Нет | Control plane |
| SSTable block (write) | Нет (flush) | Нет | Control plane |
| SSTable block (read) | Да | Да | Partition index, Bloom в RAM |
| Manifest (write) | Нет | Нет | Control plane |
| Manifest (read) | Нет (recovery) | Нет | Control plane |
| VLog record (write) | Да | Да | Preallocated сегменты |
| VLog record (read) | Да | Да | По pointer из LSM |
| Wire format (encode) | Да | Да | CodecV2 + SharedBufferPool |
| Wire format (decode) | Да | Да | MaterializeToBuffer (no copy) |
| gRPC framing | Нет | Нет | Control plane |

**Гарантия:** CI-бенчмарк с `-benchmem` на каждой версии Go (D69).

---

## 8. Recovery

**Ссылки:** ADR-002, ADR-012 v3, HLD v7.0 §5.4 (принцип 9), BENCH-007.

### 8.1. Процесс

1. Читать manifest (snapshot).
2. Собрать список SSTable.
3. Читать WAL сегменты (все).
4. Фильтровать записи по shard (для shared WAL pool).
5. Применять записи к MemTable.
6. Flush MemTable в SSTable (если нужно).
7. Обновить manifest.

### 8.2. Partial write

См. §2.5. Последняя запись может быть обрезана. Recovery восстанавливает до последнего fsync.

### 8.3. Параллельный recovery per shard

Для shared WAL pool с 10000 shards:

- Recovery per shard параллельный.
- Лимит concurrent (не более N shard одновременно, чтобы не насытить I/O).
- BR-11: O(1) по файлам, не O(N_shards).
- Прогноз: 10000 shards, 8 concurrent — ~1 сек (ADR-035 v2.0 §7.4).

### 8.4. Corruption

| Corruption | Детекция | Действие |
|---|---|---|
| WAL record CRC mismatch | CRC32C | Обрезать последнюю запись |
| WAL header CRC mismatch | CRC32C | Обрезать сегмент |
| SSTable block CRC mismatch | CRC32C | Fail fast (ErrCorrupted) |
| SSTable footer CRC mismatch | CRC32C | Fail fast |
| Manifest entry CRC mismatch | CRC32C | Fail fast |
| VLog record CRC mismatch | CRC32C | Fail fast |

---

## 9. Примеры (hex dump)

### 9.1. WAL record (PUT)

```
00000000: 1a 2b 3c 4d 00 00 00 00 00 00 00 01 00 00 00 00  .+<M............
00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000020: 01 05 00 08 00 00 00 00 6b 65 79 3a 31 76 61 6c  ........key:1val
00000030: 75 65 3a 31                                         ue:1
```

Расшифровка:
- CRC32C: `1a 2b 3c 4d`
- LSN: `00 00 00 00 00 00 00 01`
- Term: `00 00 00 00 00 00 00 00` (single-node)
- Index: `00 00 00 00 00 00 00 00` (single-node)
- Type: `01` (PUT)
- KeyLen: `05 00` (5)
- ValLen: `08 00 00 00` (8)
- Flags: `00`
- Key: `6b 65 79 3a 31` ("key:1")
- Value: `76 61 6c 75 65 3a 31` ("value:1")

### 9.2. Wire format (AppendEntries)

```
00000000: 54 4b 01 01 00 00 00 00 00 00 00 00 54 4b 01 01  TK..........TK..
00000010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000020: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000030: 00 00 00 00 00 00 00 01 00 00 00 01              ............
```

Расшифровка:
- Magic: `54 4b` ("TK")
- Version: `01`
- Type: `01` (AppendEntries)
- Flags: `00`
- Reserved: `00 00 00`
- CRC32C: `00 00 00 00` (placeholder)
- Term: `00 00 00 00 00 00 00 01`
- LeaderID: `00 00 00 00 00 00 00 01`
- PrevLogIdx: `00 00 00 00 00 00 00 00`
- PrevLogTerm: `00 00 00 00 00 00 00 00`
- LeaderCommit: `00 00 00 00 00 00 00 01`
- EntryCount: `00 00 00 01`

---

## 10. Versioning (N-1 backward compat)

### 10.1. Таблица версий

| Формат | v0.1 | v0.2 | v0.3 | v0.4 | v0.5 | v0.6 | v0.7 | v1.0 |
|---|---|---|---|---|---|---|---|---|
| WAL | 0x01 | 0x01 | 0x01 | 0x01 | 0x01 | 0x02 | 0x02 | 0x03 |
| SSTable | 0x01 | 0x02 | 0x03 | 0x03 | 0x03 | 0x04 | 0x05 | 0x06 |
| Manifest | 0x01 | 0x01 | 0x01 | 0x01 | 0x01 | 0x02 | 0x02 | 0x03 |
| VLog | — | — | 0x01 | 0x01 | 0x01 | 0x01 | 0x01 | 0x02 |
| Wire | — | — | — | — | — | 0x01 | 0x01 | 0x02 |

### 10.2. Правила

- Reader v0.N читает формат v0.(N-1).
- Reader v0.N отказывает формат v0.(N+1) с `ErrVersionMismatch`.
- Writer v0.N пишет формат v0.N.
- Migration guide — в release notes.

### 10.3. Пример

- Reader v0.2 читает WAL 0x01 (v0.1).
- Reader v0.2 отказывает WAL 0x02 (v0.6) — `ErrVersionMismatch`.
- Reader v0.6 читает WAL 0x01 (v0.1) и 0x02 (v0.6).
- Reader v0.6 пишет WAL 0x02.

---

## 11. Open questions

- **WAL segment 1 ГБ:** для edge (eMMC 16 ГБ) — слишком большой. Вариант: конфигурируемый размер (256 МБ – 1 ГБ). Не решено.
- **SSTable block size 4 КБ:** оптимально для NVMe, но для eMMC может быть меньше. Вариант: конфигурируемый. Не решено.
- **Bloom 9.6 бит/ключ × 6–7 уровней:** ~840 МБ на 100M keys. Для embedded — много. Вариант: tiered bloom (L0–L1 без bloom, L2+ с bloom). Не решено.
- **Wire format для CodecV2:** как именно регистрировать? `encoding.RegisterCodecV2`. Не решено, достаточно ли.
- **Manifest atomic rename:** на Windows — не атомарен. v0.3 (Windows prod-ready). Не решено.
- **VLog SegID 4 Б:** max 4B сегментов. При 1 ГБ сегментах — 4 EB данных. Достаточно. Не решено.
- **WAL Term/Index 8 Б:** для Raft достаточно. Не решено.
- **CRC32C hardware acceleration:** SSE4.2 (x86_64) и CRC32 (ARM64). Fallback — software CRC32C. Не решено, приемлемо ли.
- **Compression в WAL:** per-segment или per-record? Не решено.
- **Encryption в WAL:** in-place encrypt (AES-256-GCM). Overhead 10–20% CPU (ADR-035 v2.0 §5.6). Не решено.

**Правило:** неизвестное документируется как неизвестное.

---

## 12. Связанные документы

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v7.0 | Архитектура, §5.1 (слои), §5.3 (модули), §6.7 (shared WAL), §6.8 (CodecV2), §6.9 (wire format) |
| TephraKV-API-001 v1.0 | Public API |
| TephraKV-ROADMAP-001 v3.0 | §5 (детализация версий) |
| TephraKV-BACKLOG-001 v5.0 | §2 (FEAT-003, FEAT-005a/b, FEAT-043/044) |
| TephraKV-GLOSSARY-001 v2.0 | Термины |
| ADR-002 | WAL format |
| ADR-003 | Bloom filter, partition index |
| ADR-005 v4 | Transport: gRPC → DRPC → свой TCP |
| ADR-009 v2 | Zero-alloc scope: framing vs codec |
| ADR-012 v3 | WAL shared pool с per-shard LSN namespace |
| ADR-014 | Snapshot transfer |
| ADR-022 | VLog (WiscKey) |
| ADR-023 | VLog GC |
| ADR-029 v2 | Wire format: кастомный codec, не protobuf |
| ADR-034 v2.0 | Engineering practices |
| ADR-035 v2.0 | Capacity model |

---

## 13. Что дальше

1. **Утвердить FORMAT-001 v1.0** → статус `Approved`.
2. **Синхронизировать с API-001 v1.0** — форматы WAL/SST под API.
3. **Написать godoc** — из API-001.
4. **Начать CHORE-001** — bootstrap.
5. **Начать FEAT-003 (WAL)** — используя FORMAT-001 §2.
6. **Начать FEAT-005a (SSTable)** — используя FORMAT-001 §3.

**Правило:** если формат не в FORMAT-001 — его нет.

---

**Конец TephraKV-FORMAT-001 v1.0**
