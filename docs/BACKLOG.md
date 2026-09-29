# TephraKV — Backlog

**Документ:** TephraKV-BACKLOG-001
**Версия:** 6.0
**Статус:** Active
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v11.2, TephraKV-PRD-001 v1.3, TephraKV-ROADMAP-001 v4.0, TephraKV-GLOSSARY-001 v3.4, TephraKV-ENG-001 v1.2, ADR-005 v5, ADR-006 v5, ADR-010 v3, ADR-012 v6, ADR-026 v5, ADR-034 v2.2, ADR-035 v2.3, ADR-036 v1
**Заменяет:** TephraKV-BACKLOG-001 v5.0

---

## 0. Правила ведения

### 0.1. Что изменилось против v5.0

**Синхронизация с HLD v11.2 и ROADMAP v4.0: tiered latency, Dragonboat, v0.6a/b, SP 448.**

1. **Tiered latency goals:** v0.1 получил tiered p999 (T1 < 200 нс, T2 < 500 нс, T3 < 2 мкс, T4 < 50 мкс). `TierDistribution` в InternalMetrics.
2. **BENCH-016 (Tiered Latency):** добавлен (DOC-011, v0.1, 3 SP).
3. **Raft core v0.6a:** Dragonboat (не etcd/raft). Tan engine, кастомный LogDB (`ILogDB`), встроенный TCP Dragonboat.
4. **v0.6 разбит:** v0.6a (78 SP, Distributed KV Core) + v0.6b (28 SP, SQL + Columnar Replica).
5. **SP:** v0.1 98 (было 103), итого 448 (было 453).
6. **Задач:** 125 (было 126). v0.1 — 34 (было 35).
7. **REF-001 (Arena free-list):** перенесён в Deferred (P2, не критичен для v0.1).
8. **REF-003 (SST footer versioning):** перенесён в v0.2 (P2).
9. **FEAT-007 (`Options.Profile`):** сохранён в v0.1, но SP 1 (не 2). Tiered-конфигурация включена в `Options`.
10. **FEAT-003:** shared WAL pool, формат WAL включает term/index поля. Совместимость v0.4 → v0.6a.
11. **ADR-012 v6:** shared WAL pool с per-shard LSN namespace. Tan engine в distributed.
12. **CAN-005:** etcd/raft → Dragonboat. CAN-011..014 новые.
13. **Все ссылки:** HLD v7.0 → v11.2, ROADMAP v3.0 → v4.0, GLOSSARY v2.0 → v3.4.
14. **D101–D114:** решения по интеграции Dragonboat, tiered latency, `AllocsPerDistributedWrite`.

### 0.2. Источник правды

BACKLOG — единственный источник правды о работе. Нет задачи в BACKLOG — нет работы. Задача противоречит HLD/PRD/ROADMAP/ADR — задачи нет.

### 0.3. Десять правил

1. Одна задача = один формат. Шаблон из §15 обязателен.
2. Одна задача = один статус. Не «почти готово».
3. Одна задача = один PR. Не смешивать.
4. DoR до старта. Не начинать без чек-листа.
5. DoD до закрытия. Не закрывать без чек-листа.
6. Одна задача = одна версия. Переезд — через ADR.
7. Оценка только в SP. 1 SP = 0.5 дня фокусной работы.
8. Ссылки обязательны. HLD/PRD/ADR/ENG/BENCH.
9. Никаких TODO в коде без задачи в BACKLOG.
10. Отменённые задачи не удаляются. Статус `cancelled` + причина.

### 0.4. Статусы

| Статус | Значение |
|---|---|
| `todo` | В бэклоге, готова к DoR |
| `dor` | DoR проверяется |
| `ready` | DoR выполнен, готова к старту |
| `in-progress` | В работе |
| `blocked` | Заблокирована. Причина и ответственный указаны. |
| `review` | Код готов, идёт self-review / внешнее ревью |
| `done` | DoD выполнен, merge в main |
| `cancelled` | Отменена. Причина и ADR указаны. |
| `deferred` | Отложена. Дата пересмотра указана. |

### 0.5. Приоритеты

| Приоритет | Значение | Правило |
|---|---|---|
| `P0` | Блокирует релиз | Без неё версия не выходит. Только P0 в критическом пути. |
| `P1` | Нужна для версии | Может переехать на +1 версию, если P0 не готов. |
| `P2` | Желательна | Переезжает свободно. |
| `P3` | Идея | Не привязана к версии. В `Deferred`. |

Правило P0: не более 3 P0-задач в статусе `in-progress`.

### 0.6. Типы задач

| Тип | Префикс | SP | Пример |
|---|---|---|---|
| Feature | `FEAT` | 1–8 | Новая функциональность |
| Bug | `BUG` | 1–8 | Исправление |
| Refactor | `REF` | 2–8 | Без смены семантики |
| Performance | `PERF` | 2–8 | Оптимизация |
| Test | `TEST` | 1–5 | Тесты, fuzz, бенчмарки |
| Docs | `DOC` | 1–5 | ADR, HLD, BENCH, отчёты |
| Chore | `CHORE` | 1–3 | CI, bootstrap, tooling |
| Spike | `SPIKE` | 2–5 | Исследование, PoC |
| ADR | `ADR` | 1–3 | Architecture Decision Record |

### 0.7. SP-шкала

| SP | Дни | Что это |
|---|---|---|
| 1 | 0.5 | Тривиально. Один файл, без тестов. |
| 2 | 1 | Мелкая задача. Один файл + тест. |
| 3 | 1.5 | Средняя. Несколько файлов + тесты. |
| 5 | 2.5 | Крупная. Модуль + тесты + бенчмарк. |
| 8 | 4 | Очень крупная. Несколько модулей. Разбить, если можно. |

Правило: SP > 8 — разбить. Не допускается SP 13 в одной задаче.

### 0.8. DoR (Definition of Ready)

Задача переходит в `ready`, только если все пункты выполнены:

- [ ] Проблема сформулирована. Не «сделать X», а «сейчас Y, нужно Z, потому что W».
- [ ] Acceptance criteria записаны. Конкретные, проверяемые, измеримые.
- [ ] ADR написан (если решение значимое).
- [ ] BENCH определён (если perf-critical).
- [ ] Zero-alloc оценено. Есть ответ: hot path или нет.
- [ ] Влияние на формат данных оценено (если storage / wire).
- [ ] Зависимости проверены. Нет циклов, нет новых внешних либ без ADR.
- [ ] SP оценён. Не > 8.
- [ ] Приоритет присвоен. P0/P1/P2/P3.
- [ ] Версия указана.
- [ ] Артефакты перечислены.

### 0.9. DoD (Definition of Done)

Задача переходит в `done`, только если все пункты выполнены:

- [ ] Код написан и проходит CI (build, vet, test -race, lint).
- [ ] Тесты: unit + race + fuzz (если применимо).
- [ ] Бенчмарк обновлён и не деградировал > 5% (p99).
- [ ] AllocsPerRun == 0 для hot path (или документированное исключение в ADR).
- [ ] Escape analysis чист. Нет новых heap escapes на hot path.
- [ ] ADR обновлён (статус `Accepted`), если применимо.
- [ ] CHANGELOG.md обновлён.
- [ ] Документация обновлена: godoc на публичный API, README если API менялся.
- [ ] Нет TODO без issue-ссылки.
- [ ] Комментарии объясняют «почему», не «что».
- [ ] Коммит с понятным сообщением (conventional commits).
- [ ] Self-review выполнен (для задач SP ≥ 5).

### 0.10. Критический путь (v0.1)

```
CHORE-001 → CHORE-002 → SPIKE-001 → ADR-019
                                       ↓
                                    FEAT-001 → ADR-001 → TEST-002
                                       ↓
                                    FEAT-002 → FEAT-003 → ADR-002 → TEST-003
                                                                   ↓
                                                            FEAT-004 → ADR-010
                                                                   ↓
                                                            FEAT-006 → ADR-020
                                                                   ↓
                                                            TEST-001 → CHORE-005
                                                                   ↓
                                                            DOC-011 (BENCH-016) → DOC-001
```

Правило: критический путь отслеживается ежедневно. Если задача на критическом пути в статусе `blocked` > 1 день — эскалация.

### 0.11. Метрики бэклога

| Метрика | Цель | Как измеряется |
|---|---|---|
| WIP | ≤ 3 | Кол-во задач в `in-progress` |
| Cycle time | ≤ 3 дня | От `in-progress` до `done` |
| Lead time | ≤ 5 дней | От `ready` до `done` |
| Throughput | 1–3 задачи/день | `done` за день |
| Blocked ratio | < 10% | `blocked` / всего в работе |
| P0 blocked | 0 | Кол-во P0 в `blocked` |
| SP выполнено за неделю | ≥ 12 | Сумма SP `done` за 5 дней |
| Baseline SP/неделя | по факту первых 4 недель | Solo target |

---

## 1. Обзор по версиям

### 1.1. Прогресс

| Версия | Задач | Done | In-progress | Ready | Todo | Blocked | SP всего | SP done | Прогресс |
|---|---|---|---|---|---|---|---|---|---|
| v0.1 | 34 | 0 | 0 | 0 | 34 | 0 | 98 | 0 | 0% |
| v0.2 | 16 | 0 | 0 | 0 | 16 | 0 | 47 | 0 | 0% |
| v0.3 | 11 | 0 | 0 | 0 | 11 | 0 | 25 | 0 | 0% |
| v0.4 | 13 | 0 | 0 | 0 | 13 | 0 | 40 | 0 | 0% |
| v0.5 | 10 | 0 | 0 | 0 | 10 | 0 | 26 | 0 | 0% |
| v0.6a | 18 | 0 | 0 | 0 | 18 | 0 | 78 | 0 | 0% |
| v0.6b | 7 | 0 | 0 | 0 | 7 | 0 | 28 | 0 | 0% |
| v0.7 | 8 | 0 | 0 | 0 | 8 | 0 | 30 | 0 | 0% |
| v1.0 | 8 | 0 | 0 | 0 | 8 | 0 | 76 | 0 | 0% |
| **Всего** | **125** | **0** | **0** | **0** | **125** | **0** | **448** | **0** | **0%** |

### 1.2. Сверка с ROADMAP

ROADMAP-001 v4.0 фиксирует: 448 SP.
BACKLOG-001 v6.0 уточняет: 125 задач, 448 SP.

Числа синхронизированы.

### 1.3. Ближайший релиз

**v0.1 — Deterministic Foundation**
Старт: 2026-09-22. Base-срок: 6 недель. Risk-adjusted: 8 недель.
Критерий: **p999 GET < 500 нс (T2: L3-resident, 1M keys, single-node)**, zero-alloc на hot path.

### 1.4. Критический путь

См. §0.10.

---

## 2. v0.1 — Deterministic Foundation

**Цель:** детерминированная задержка на одном узле в tiered-терминах, zero-alloc hot path.
**Non-goals:** компакции, VLog, транзакции, репликация, Windows/macOS prod, >100M ключей.
**Base-срок:** 6 недель. **Risk-adjusted:** 8 недель. **SP:** 98. **Задач:** 34.

### 2.1. Эпики v0.1

| Эпик | Задач | SP | P0 | P1 | P2 | Критпуть |
|---|---|---|---|---|---|---|
| E01: Bootstrap | 5 | 8 | 2 | 3 | 0 | yes |
| E02: Arena | 2 | 4 | 2 | 0 | 0 | yes |
| E03: MemTable | 5 | 17 | 4 | 0 | 1 | yes |
| E04: WAL | 5 | 16 | 3 | 2 | 0 | yes |
| E05: Durability + Profile | 4 | 11 | 3 | 1 | 0 | yes |
| E06: SSTable | 4 | 15 | 1 | 2 | 1 | no |
| E07: Metrics + Tier | 3 | 12 | 2 | 1 | 0 | no |
| E08: CI | 3 | 7 | 2 | 1 | 0 | yes |
| E09: Benchmarks | 3 | 8 | 2 | 1 | 0 | no |
| **Итого** | **34** | **98** | **21** | **11** | **2** | |

### 2.2. E01: Bootstrap

#### CHORE-001: Bootstrap repository

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | — |
| ADR | — |
| BENCH | — |
| Критпуть | yes |

**Контекст:** репозиторий пустой. Нужны базовые файлы проекта.

**Acceptance criteria:**
- [ ] `go.mod`: module path `github.com/<user>/tephrakv`, Go 1.24.
- [ ] `.gitignore` покрывает: binaries, coverage, IDE, OS, bench, dist, tmp.
- [ ] `Makefile` с целями: `all`, `build`, `test`, `test-race`, `lint`, `fmt`, `vet`, `bench`, `bench-regression`, `clean`.
- [ ] `README.md` ≤ 40 строк, без запрещённых утверждений (HLD v11.2 §12.B).
- [ ] `LICENSE` placeholder со ссылкой на ADR-004.
- [ ] `.golangci.yml` с минимальным набором линтеров.
- [ ] Первый коммит в `master`.

**Артефакты:** `go.mod`, `.gitignore`, `Makefile`, `README.md`, `LICENSE`, `.golangci.yml`.

---

#### CHORE-002: CI skeleton

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 1 |
| Владелец | @ekaterina |
| Зависимости | CHORE-001 |
| ADR | — |
| BENCH | — |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] `.github/workflows/ci.yml` создан.
- [ ] Steps: `go build ./...`, `go vet ./...`, `go test -race -count=1 ./...`, `golangci-lint run`.
- [ ] Триггер: push в `master`/`main` и PR.
- [ ] CI зелёный на пустом проекте.
- [ ] Badge в README.

**Артефакты:** `.github/workflows/ci.yml`.

---

#### CHORE-003: ADR template + README

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 1 |
| Владелец | @ekaterina |
| Зависимости | CHORE-001 |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Acceptance criteria:**
- [ ] `docs/adr/template.md` создан.
- [ ] `docs/adr/README.md` создан: реестр всех ADR со статусами.
- [ ] Реестр синхронизирован с HLD v11.2 §10 (D1–D114).

**Артефакты:** `docs/adr/template.md`, `docs/adr/README.md`.

---

#### CHORE-004: Branch protection + git hooks

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | CHORE-002 |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Acceptance criteria:**
- [ ] GitHub branch protection на `master`: required status checks (CI), no force push, no deletion.
- [ ] Pre-commit hook: `gofmt`, `go vet`, `golangci-lint` на staged files.
- [ ] Pre-push hook: `go test ./...`.
- [ ] `.githooks/README.md`.

**Артефакты:** `.githooks/pre-commit`, `.githooks/pre-push`, `.githooks/README.md`.

---

#### CHORE-007: Threat model v0.1 + Failure model v0.1

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | CHORE-001 |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Контекст:** HLD v11.2 §9.5, §9.6 фиксируют failure model и threat model (разделённые: storage v0.1, network v0.6a).

**Acceptance criteria:**
- [ ] `docs/security/failure-model-v0.1.md`: 10 сбоев из HLD v11.2 §9.5, гарантии v0.1.
- [ ] `docs/security/storage-threat-model-v0.1.md`: 6 угроз из HLD v11.2 §9.6, митигации v0.1.
- [ ] `docs/security/network-threat-model-v0.6a.md`: 5 угроз из HLD v11.2 §9.6, митигации v0.6a (заготовка).
- [ ] Секция «вне scope v0.1» явная.
- [ ] Ссылки на ADR-002, ADR-010 v3, ADR-009 v2.
- [ ] Ревью: соответствие HLD v11.2 §9.

**Артефакты:** `docs/security/failure-model-v0.1.md`, `docs/security/storage-threat-model-v0.1.md`, `docs/security/network-threat-model-v0.6a.md`.

---

### 2.3. E02: Arena

#### SPIKE-001: Arena + epoch spike

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 3 |
| Владелец | @ekaterina |
| Зависимости | CHORE-001 |
| ADR | ADR-019 |
| BENCH | BENCH-003 (черновик) |
| Критпуть | yes |

**Контекст:** arena — фундамент zero-alloc storage. mmap-backed bump allocator. Аллокация 64 Б за ~2.9 нс (vs ~40 нс для heap) — D85.

**Acceptance criteria:**
- [ ] `internal/arena/arena.go`: API `New(slabSize int)`, `Alloc(n int) []byte`, `Reset()`, `Stats() Stats`.
- [ ] `internal/arena/arena_test.go`: ≥ 6 тестов (Basic, Zero, TooLarge, Reset, Stats, NoOverlap).
- [ ] `internal/arena/arena_bench_test.go`: 2 бенчмарка (Small 64B, Medium 1KB).
- [ ] Все тесты проходят с `-race`.
- [ ] Benchmark: `allocs/op == 0`.
- [ ] Escape analysis чист.
- [ ] mmap-backed (Linux x86_64 + ARM64).
- [ ] ADR-019 в статусе `Accepted`.

**Артефакты:** `internal/arena/{arena,arena_test,arena_bench_test}.go`, `docs/adr/ADR-019-arena-epoch.md`.

---

#### ADR-019: Arena + epoch reclamation

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 1 |
| Владелец | @ekaterina |
| Зависимости | — |
| ADR | ADR-019 |
| BENCH | — |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Шаблон ADR соблюдён.
- [ ] Decision: bump allocator + fixed-size slabs + epoch-based reclamation. mmap-backed.
- [ ] Alternatives: `sync.Pool`, malloc per object, mmap per slab.
- [ ] Явно: `sync.Pool` запрещён в data plane (D68).
- [ ] Ссылки: HLD v11.2 §4.2, D85.

**Артефакты:** `docs/adr/ADR-019-arena-epoch.md`.

---

### 2.4. E03: MemTable (skiplist)

#### FEAT-001: Lock-free skiplist MemTable

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 8 |
| Владелец | @ekaterina |
| Зависимости | SPIKE-001, ADR-001 |
| ADR | ADR-001 |
| BENCH | BENCH-002, BENCH-003, BENCH-016 |
| Критпуть | yes |

**Контекст:** MemTable — lock-free skiplist на offsets в arena. Single-version (D72).

**Acceptance criteria:**
- [ ] Skiplist nodes — offsets в arena, не указатели.
- [ ] Insert/Get/Delete — lock-free (CAS на next pointers).
- [ ] Iterator с epoch guard.
- [ ] Fuzz: 1M ops, 4 writers, 4 readers, без race.
- [ ] Race-тест проходит.
- [ ] Benchmark: p99 GET < 1 мкс, allocs/op == 0.
- [ ] Escape analysis чист.
- [ ] **T2 (L3-resident, 1M keys): p999 < 500 нс.**
- [ ] ADR-001 в статусе `Accepted`.

**Артефакты:** `internal/memtable/{skiplist,skiplist_test,skiplist_bench_test,fuzz_test}.go`, `docs/adr/ADR-001-lockfree-skiplist.md`.

---

#### ADR-001: Lock-free skiplist vs B-tree

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 1 |
| Владелец | @ekaterina |
| Зависимости | — |
| ADR | ADR-001 |
| BENCH | — |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Alternatives: B-tree, ART, hash table.
- [ ] Decision: lock-free skiplist на offsets.
- [ ] Consequences: cache locality хуже B-tree, но lock-free проще. Tiered p999 зависит от locality.
- [ ] References: Pugh, Herlihy.

**Артефакты:** `docs/adr/ADR-001-lockfree-skiplist.md`.

---

#### REF-002: Skiplist — level height tuning

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P2 |
| Статус | todo |
| SP | 1 |
| Владелец | @ekaterina |
| Зависимости | FEAT-001 |
| ADR | ADR-001 |
| BENCH | BENCH-003, BENCH-016 |
| Критпуть | no |

**Acceptance criteria:**
- [ ] Benchmark с разными `p` (0.25, 0.5) и `maxHeight` (16, 20, 24).
- [ ] Метрики: p50/p99/p999 по tier.
- [ ] Результаты в BENCH-003, BENCH-016.
- [ ] ADR-001 обновлён с обоснованием.

**Артефакты:** `internal/memtable/skiplist_config.go`, обновлённый ADR-001.

---

#### FEAT-002: Frozen MemTable → SST flush

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 5 |
| Владелец | @ekaterina |
| Зависимости | FEAT-001, FEAT-005a |
| ADR | — |
| BENCH | BENCH-005 (черновик) |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Frozen MemTable pool.
- [ ] Flush worker (single-threaded).
- [ ] Writers не блокируются на flush.
- [ ] Backpressure при переполнении пула (ADR-008).
- [ ] Тесты: flush под 1M concurrent writes.

**Артефакты:** `internal/memtable/frozen.go`, `internal/memtable/frozen_test.go`.

---

#### TEST-002: Skiplist fuzz 1M ops

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | FEAT-001 |
| ADR | ADR-001 |
| BENCH | — |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] `FuzzSkiplist` с 1M ops за прогон.
- [ ] 4 writers, 4 readers, random ops.
- [ ] Прогон 1 час без race, без panic, без corruption.
- [ ] Интеграция в CI (раз в сутки).

**Артефакты:** `internal/memtable/fuzz_test.go`, `.github/workflows/fuzz.yml`.

---

### 2.5. E04: WAL

#### FEAT-003: WAL writer + group commit (shared pool)

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 8 |
| Владелец | @ekaterina |
| Зависимости | CHORE-001, ADR-002, ADR-012 v6 |
| ADR | ADR-002, ADR-012 v6 |
| BENCH | BENCH-007, BENCH-009 |
| Критпуть | yes |

**Контекст:** HLD v11.2 §5.1 фиксирует shared WAL pool с per-shard LSN namespace. v0.1 — single-node, но структура готова к distributed.

**Acceptance criteria:**
- [ ] WAL segment: mmap, CRC32C, LSN.
- [ ] **Shared WAL pool** (не WAL per shard). Per-shard LSN namespace. `hash(shard_id) mod N`.
- [ ] Group commit coordinator: batching до 64 записей, окно 100 мкс.
- [ ] WAL replay при recovery.
- [ ] Corruption detection + partial recovery.
- [ ] WAL preallocated (1 ГБ сегменты). Rotation.
- [ ] Формат WAL включает term/index поля (нулевые в single-node) — совместимость с v0.6a.
- [ ] Benchmark: p99 fsync < 100 мкс (NVMe).
- [ ] Zero-alloc на write path (NO_SYNC).

**Артефакты:** `internal/wal/{wal,shared_pool,group_commit,wal_test,wal_bench_test,recovery_test}.go`, `docs/adr/ADR-002-wal-format.md`, `docs/adr/ADR-012-wal-shared-pool.md`.

---

#### ADR-002: WAL format

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 1 |
| Владелец | @ekaterina |
| Зависимости | — |
| ADR | ADR-002 |
| BENCH | — |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Секции: формат записи, CRC32C, LSN, segment size, rotation.
- [ ] Alternatives: per-op fsync vs group commit; fixed vs variable record size.
- [ ] Формат включает term/index поля.
- [ ] Ссылки: HLD v11.2 §4.4, §5.1.

**Артефакты:** `docs/adr/ADR-002-wal-format.md`.

---

#### ADR-012 v6: WAL shared pool с per-shard LSN namespace

| Поле | Значение |
|---|---|
| Версия | v0.1 (design), v0.6a (implementation) |
| Приоритет | P0 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | ADR-002 |
| ADR | ADR-012 v6 |
| BENCH | — |
| Критпуть | yes |

**Контекст:** HLD v11.2 §5.1 фиксирует shared WAL pool. O(1) по файлам, не O(N_shards). Прецеденты: TiKV (один RocksDB WAL на узел для всех Region), Neon (фильтрация WAL на safekeeper). D53, D62. В distributed — Tan engine (D101).

**Acceptance criteria:**
- [ ] Decision: **shared WAL pool** с per-shard LSN namespace (single-node). Tan engine для distributed.
- [ ] WAL entry включает Raft term + index.
- [ ] Single-node: term/index поля присутствуют с нулевыми значениями (совместимость v0.4 → v0.6a).
- [ ] Distributed: Raft log хранится в Tan engine, WAL pool не участвует в Raft path.
- [ ] Recovery: WAL replay с фильтрацией по shard; параллельный per shard, с лимитом concurrent.
- [ ] Alternatives: WAL per shard; отдельный Raft log; no-Raft для single-node.
- [ ] Ссылки: HLD v11.2 §5.1, §5.2, D53, D62, D101.

**Артефакты:** `docs/adr/ADR-012-wal-shared-pool.md` (v6).

---

#### TEST-003: WAL recovery test (kill -9)

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 3 |
| Владелец | @ekaterina |
| Зависимости | FEAT-003 |
| ADR | ADR-002 |
| BENCH | BENCH-007 |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Тест с N = 10000 записей, kill -9 через random интервал.
- [ ] Recovery восстанавливает все SYNC_MASTER записи.
- [ ] Recovery корректно обрабатывает partial write.
- [ ] 100 прогонов без corruption.

**Артефакты:** `internal/wal/recovery_test.go`, `docs/bench/BENCH-007-crash-recovery.md`.

---

#### PERF-001: WAL fsync latency measurement

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | FEAT-003 |
| ADR | ADR-002 |
| BENCH | BENCH-009 |
| Критпуть | no |

**Acceptance criteria:**
- [ ] Бенчмарк fsync с 1M прогонов.
- [ ] Метрики: p50, p99, p999, p9999, max.
- [ ] Hardware зафиксирован.

**Артефакты:** `internal/wal/fsync_bench_test.go`, `docs/bench/BENCH-009-durability-policy.md`.

---

### 2.6. E05: Durability + Profile

#### FEAT-004: DurabilityPolicy per operation

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 6 |
| Владелец | @ekaterina |
| Зависимости | FEAT-003 |
| ADR | ADR-010 v3 |
| BENCH | BENCH-009 |
| Критпуть | yes |

**Контекст:** HLD v11.2 §4.4 вводит SYNC_LEADER для distributed. v0.1 реализует NO_SYNC, SYNC_MASTER, SYNC_LEADER (эквивалентен SYNC_MASTER в single-node).

**Acceptance criteria:**
- [ ] `DurabilityPolicy` enum: NO_SYNC, SYNC_MASTER, SYNC_LEADER, SYNC_MAJORITY, SYNC_ALL.
- [ ] **Zero value = безопасный default: `SYNC_MASTER = iota` (0).** D92.
- [ ] v0.1: SYNC_MAJORITY/SYNC_ALL возвращают `ErrNotImplemented`.
- [ ] `WriteOptions{Durability}`.
- [ ] `Put`, `Get`, `Delete` с variadic opts (D30).
- [ ] **`Get` возвращает `(value, release, err)`** (D93).
- [ ] NO_SYNC: запись в буфер, возврат, fsync в окне group commit.
- [ ] SYNC_MASTER: fsync, возврат.
- [ ] SYNC_LEADER в single-node эквивалентен SYNC_MASTER (документировано в godoc).
- [ ] Mixed batch: LSN per op, recovery восстанавливает до последнего fsync.
- [ ] Метрики: `writes_by_durability_policy{policy}`, `fsync_latency_by_policy{policy}`.

**Артефакты:** `tephrakv/{options,db,db_test}.go`.

---

#### FEAT-007: Options.Profile с ProfileLatency default

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 1 |
| Владелец | @ekaterina |
| Зависимости | FEAT-004 |
| ADR | ADR-037 (proposed) |
| BENCH | — |
| Критпуть | no |

**Контекст:** HLD v11.2 §6.1 вводит `Profile` (Latency / Compliance) как конфигурацию, не форк. v0.1 реализует только `ProfileLatency` default.

**Acceptance criteria:**
- [ ] `Profile` enum: `ProfileLatency`, `ProfileCompliance`.
- [ ] `Options.Profile` field.
- [ ] v0.1: `ProfileLatency` заполняет дефолты (DefaultDurability: SYNC_MASTER, ReadAmpTarget: 3, ValueLogThreshold: 256, BloomBitsPerKey: 9.6).
- [ ] v0.1: `ProfileCompliance` возвращает `ErrNotImplemented` (модули MVCC/PITR/audit не готовы).
- [ ] Валидация: `Open` возвращает `ErrInvalidProfile` при неизвестном значении.
- [ ] ADR-037 (Profile semantics) в статусе `Accepted`.

**Артефакты:** `tephrakv/profile.go`, `tephrakv/profile_test.go`, `docs/adr/ADR-037-profile.md`.

---

#### ADR-010 v3: Durability semantics

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | ADR-002 |
| ADR | ADR-010 v3 |
| BENCH | — |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Таблица политик (5 штук, включая v0.6a).
- [ ] Определение: что гарантирует каждая, что нет.
- [ ] Явно: SYNC_MASTER в distributed не даёт distributed durability.
- [ ] Явно: SYNC_LEADER добавлен для явной семантики fast-path без distributed durability.
- [ ] **Mixed batch запрещён.** D61.
- [ ] **Per-operation durability в distributed — через кастомный LogDB** (`ILogDB`). D102.
- [ ] Линтер `tephra-lint` предупреждает при SYNC_MASTER в distributed (v0.6a+).
- [ ] Ссылки: HLD v11.2 §4.4, D54, D61, D102.

**Артефакты:** `docs/adr/ADR-010-durability-semantics.md` (v3).

---

#### TEST-004: Durability policy matrix

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | FEAT-004 |
| ADR | ADR-010 v3 |
| BENCH | BENCH-009 |
| Критпуть | no |

**Acceptance criteria:**
- [ ] Матрица в BENCH-009.
- [ ] 5 политик: NO_SYNC, SYNC_MASTER, SYNC_LEADER, SYNC_MAJORITY, SYNC_ALL.
- [ ] SYNC_MAJORITY/SYNC_ALL: проверка `ErrNotImplemented` в v0.1.
- [ ] Профили: 100% write, 50/50, 80/20.
- [ ] 1M ops per профиль.

**Артефакты:** `docs/bench/BENCH-009-durability-policy.md`.

---

### 2.7. E06: SSTable (minimal)

#### FEAT-005a: SST block format + writer

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 5 |
| Владелец | @ekaterina |
| Зависимости | FEAT-001, FEAT-003 |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Acceptance criteria:**
- [ ] Block format: data block, index block, footer.
- [ ] SST writer (single-threaded).
- [ ] CRC32C per block.
- [ ] Block size 4 КБ (конфигурируемый, 1 КБ – 64 КБ).
- [ ] Versioning с первого дня (N-1 backward compat).
- [ ] Тесты: write + read back, block boundaries, empty SST.
- [ ] Benchmark: write throughput.

**Артефакты:** `internal/sstable/{format,writer,writer_test,format_test}.go`.

---

#### FEAT-005b: SST Bloom + reader + recovery

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 6 |
| Владелец | @ekaterina |
| Зависимости | FEAT-005a |
| ADR | — |
| BENCH | BENCH-008 (черновик), BENCH-016 |
| Критпуть | no |

**Acceptance criteria:**
- [ ] Bloom filter 9.6 bits/key, FP < 1%.
- [ ] SST reader + iterator.
- [ ] Recovery: собрать состояние из WAL + SST.
- [ ] Тесты: recovery после kill -9.
- [ ] Benchmark: FP rate измерен и < 1%.
- [ ] **Tiered bloom (L0–L1 без bloom, L2+ с bloom)** — задел на v0.3, D97.

**Артефакты:** `internal/sstable/{bloom,reader,reader_test,recovery_test}.go`.

---

#### TEST-005: SST Bloom false positive rate

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | FEAT-005b |
| ADR | — |
| BENCH | BENCH-008 |
| Критпуть | no |

**Acceptance criteria:**
- [ ] Тест на 10M ключей, 1M negative lookups.
- [ ] FP rate < 1%.
- [ ] Память на Bloom измерена (9.6 бит × keys).

**Артефакты:** `internal/sstable/bloom_test.go`, `docs/bench/BENCH-008-read-amp.md`.

---

#### DOC-002: SST format spec

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P2 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | FEAT-005a, FEAT-005b |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Acceptance criteria:**
- [ ] `docs/spec/sst-format.md` создан.
- [ ] Секции: block layout, footer, index, bloom.
- [ ] Диаграмма ASCII.
- [ ] Синхронизирован с `docs/FORMAT-001.md` (когда написан).

**Артефакты:** `docs/spec/sst-format.md`.

---

### 2.8. E07: Metrics + Tier

#### FEAT-006: Two-Level Metrics Collector + Tier distribution

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 8 |
| Владелец | @ekaterina |
| Зависимости | FEAT-004, FEAT-005b |
| ADR | ADR-020 |
| BENCH | BENCH-001, BENCH-016 |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] `ExternalMetrics`: LatencyP50/P99/P999, ThroughputOpsPerSec, CostPerMillionOps.
- [ ] `InternalMetrics`: CyclesPerOp, BytesRead/WritePerOp, AllocsPerOp, FlushesPerMillion, ReadAmp, WriteAmp.
- [ ] **`TierDistribution map[Tier]float64`** — % операций по tier (L1/L2, L3, RAM, NVMe). D110.
- [ ] HDR histogram (без аллокаций на hot path).
- [ ] Build tag `tephra_metrics`.
- [ ] `db.Metrics()` работает.

**Артефакты:** `internal/metrics/{collector,external,internal,histogram,tier,*_test}.go`.

---

#### ADR-020: Metrics definitions

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | — |
| ADR | ADR-020 |
| BENCH | — |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Таблица: метрика, определение, единица, как измеряется.
- [ ] CyclesPerOp: rdtsc / cntvct_el0.
- [ ] AllocsPerOp: testing.AllocsPerRun + MemStats delta.
- [ ] ReadAmp: disk reads per GET.
- [ ] WriteAmp: bytes written / bytes user-written.
- [ ] **TierDistribution: % операций в L1/L2, L3, RAM, NVMe.**
- [ ] Ссылки: HLD v11.2 §4.3, D7, D22, D27, D110.

**Артефакты:** `docs/adr/ADR-020-metrics-definitions.md`.

---

#### TEST-006: Metrics correctness

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | FEAT-006 |
| ADR | ADR-020 |
| BENCH | BENCH-001 |
| Критпуть | no |

**Acceptance criteria:**
- [ ] Controlled workload: известный latency distribution.
- [ ] Проверка: p50/p99/p999 в пределах 5% от ожидаемого.
- [ ] AllocsPerOp совпадает с testing.AllocsPerRun.
- [ ] TierDistribution суммируется к 100%.

**Артефакты:** `internal/metrics/correctness_test.go`, `docs/bench/BENCH-001-methodology.md`.

---

### 2.9. E08: CI / Quality gates

#### TEST-001: Zero-alloc gate в CI

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 3 |
| Владелец | @ekaterina |
| Зависимости | FEAT-001, FEAT-003, FEAT-004, CHORE-002 |
| ADR | ADR-009 v2 |
| BENCH | BENCH-002 |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] Benchmark suite с AllocsPerRun.
- [ ] Escape analysis report (`-gcflags=-m`) — диагностика.
- [ ] **CI-бенчмарк с `-benchmem` на каждой версии Go** — контракт (D69).
- [ ] GitHub Action блокирует merge при провале.
- [ ] Документация: как локально прогнать.

**Артефакты:** `bench/zero_alloc_test.go`, `.github/workflows/zero-alloc.yml`, `docs/bench/BENCH-002-zero-alloc.md`.

---

#### CHORE-005: Benchmark regression gate

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | CHORE-002, TEST-001 |
| ADR | — |
| BENCH | BENCH-001 |
| Критпуть | yes |

**Acceptance criteria:**
- [ ] `bench/baseline.txt` создан.
- [ ] `bench/current.txt` генерируется на каждый PR.
- [ ] `benchstat` сравнивает, провал при +5% p99.
- [ ] GitHub Action блокирует merge.

**Артефакты:** `bench/baseline.txt`, `.github/workflows/bench-regression.yml`.

---

#### CHORE-006: go-arch-lint integration

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | CHORE-002, FEAT-001 |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Acceptance criteria:**
- [ ] `.go-arch-lint.yml` создан.
- [ ] `go-arch-lint check` в CI.
- [ ] Провал при цикле.
- [ ] Модули из HLD v11.2 §5.3.

**Артефакты:** `.go-arch-lint.yml`, обновлённый `.github/workflows/ci.yml`.

---

### 2.10. E09: Benchmarks

#### DOC-001: BENCH-001, 002, 003, 007

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 3 |
| Владелец | @ekaterina |
| Зависимости | TEST-001, TEST-003 |
| ADR | — |
| BENCH | BENCH-001, 002, 003, 007 |
| Критпуть | no |

**Acceptance criteria:**
- [ ] BENCH-001: Methodology + TCO формула.
- [ ] BENCH-002: Zero-Alloc Gate с результатами.
- [ ] BENCH-003: Cycles and Ticks с результатами.
- [ ] BENCH-007: Crash-Recovery с результатами.
- [ ] Публикация в `docs/bench/releases/v0.1.md`.

**Артефакты:** `docs/bench/BENCH-{001,002,003,007}-*.md`, `docs/bench/releases/v0.1.md`.

---

#### DOC-011: BENCH-016 Tiered Latency

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 3 |
| Владелец | @ekaterina |
| Зависимости | FEAT-001, FEAT-006, TEST-001 |
| ADR | ADR-020 |
| BENCH | BENCH-016 |
| Критпуть | yes |

**Контекст:** HLD v11.2 §8.4 вводит BENCH-016 — измерение p50/p99/p999 по tier (L1/L2, L3, RAM, NVMe). Главная differentiation-цель — T2 (L3-resident, p999 < 500 нс). D111.

**Acceptance criteria:**
- [ ] Методология: controlled working set (100K, 1M, 10M, 100M keys).
- [ ] Key size 16 Б, value size 64 Б. YCSB C.
- [ ] Измерение через `rdtsc` (x86_64) / `cntvct_el0` (ARM64).
- [ ] Tier определяется по cache miss (perf counters: `LLC-load-misses`, `L1-dcache-load-misses`).
- [ ] Публикуется **Tier distribution**.
- [ ] Целевые: T1 < 200 нс, T2 < 500 нс, T3 < 2 мкс, T4 < 50 мкс.
- [ ] Прецеденты: Badger in-memory (~50–100 мкс p999), Pebble in-memory (~30–80 мкс), RocksDB in-memory (~20–50 мкс).
- [ ] Результаты в `docs/bench/BENCH-016-tiered-latency.md`.

**Артефакты:** `bench/tiered_latency_test.go`, `docs/bench/BENCH-016-tiered-latency.md`.

---

#### DOC-012: Публичный README для GitHub

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P1 |
| Статус | in-progress |
| SP | 3 |
| Владелец | @ekaterina |
| Зависимости | CHORE-001 |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Контекст:** GTM-001 фаза 1 (GitHub-публикация) требует полноценного публичного README в корне репозитория. CHORE-001 создаёт bootstrap-README ≤ 40 строк; DOC-012 — его публичная версия: язык — русский, структура с оглавлением, badges, разделы «Возможности», «Производительность», «Режимы», «Быстрый старт», «Архитектура», «Durability», «Roadmap», «Бенчмарки», «Лицензия».

**Acceptance criteria:**
- [ ] Корневой `README.md` создан, на русском языке, с оглавлением и гиперссылками.
- [ ] Использованы только разрешённые утверждения из HLD v11.3 §10.1; запрещённые (§10.2) отсутствуют.
- [ ] Badges (shields.io): Go, лицензия, статус, платформа, zero-alloc, latency target.
- [ ] Все целевые показатели помечены как целевые (не выданные за измеренные).
- [ ] Ссылки на публичные документы: HLD, PRD, ROADMAP, BENCH.
- [ ] Отсутствуют непроверенные цифры и сравнения без раскрытия профиля.

**Артефакты:** `README.md`.

---

#### DOC-003: v0.1 release notes

| Поле | Значение |
|---|---|
| Версия | v0.1 |
| Приоритет | P0 |
| Статус | todo |
| SP | 2 |
| Владелец | @ekaterina |
| Зависимости | DOC-001, DOC-011 |
| ADR | — |
| BENCH | — |
| Критпуть | no |

**Acceptance criteria:**
- [ ] `docs/releases/v0.1.md` создан.
- [ ] Секции: highlights, benchmarks (включая tiered), breaking changes, known issues, next.
- [ ] GitHub Release создан.
- [ ] CHANGELOG обновлён.

**Артефакты:** `docs/releases/v0.1.md`, обновлённый `CHANGELOG.md`.

---

### 2.11. Сводка v0.1

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: Bootstrap | 5 | 8 | 2 | 3 | 0 |
| E02: Arena | 2 | 4 | 2 | 0 | 0 |
| E03: MemTable | 5 | 17 | 4 | 0 | 1 |
| E04: WAL | 5 | 16 | 3 | 2 | 0 |
| E05: Durability + Profile | 4 | 11 | 3 | 1 | 0 |
| E06: SSTable | 4 | 15 | 1 | 2 | 1 |
| E07: Metrics + Tier | 3 | 12 | 2 | 1 | 0 |
| E08: CI | 3 | 7 | 2 | 1 | 0 |
| E09: Benchmarks | 3 | 8 | 2 | 1 | 0 |
| **Итого** | **34** | **98** | **21** | **11** | **2** |

Критический путь: 21 P0-задача, base 6 недель, risk-adjusted 8 недель.

**Exit criteria v0.1:**
- [ ] **p999 GET < 500 нс (T2: L3-resident, 1M keys, single-node).**
- [ ] **p999 GET < 2 мкс (T3: RAM-resident, 10M keys, single-node).**
- [ ] **Tier distribution ≥ 80% операций в L1/L2/L3 для 1M keys.**
- [ ] Allocs/op == 0 на hot path.
- [ ] Cycles/op GET: p50 < 300, p99 < 900, p999 < 1500 (T2).
- [ ] CI-гейт работает.
- [ ] BENCH-001, 002, 003, 007, **016** опубликованы.
- [ ] Threat model v0.1 и Failure model v0.1 опубликованы.
- [ ] 200+ stars.

---

## 3. v0.2 — Read-Amplification-Minimized Compaction

**Цель:** read amp p99 < 3 на реальных объёмах при сохранении tiered p999.
**Non-goals:** VLog, MVCC, TTL, distributed.
**Base-срок:** +6 недель. **Risk-adjusted:** +7 недель. **SP:** 47. **Задач:** 16.

### 3.1. Эпики v0.2

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: Partition index | 3 | 12 | 3 | 0 | 0 |
| E02: MinMax index | 2 | 5 | 2 | 0 | 0 |
| E03: Leveled compaction | 3 | 10 | 2 | 1 | 0 |
| E04: Read-amp trigger (SLA-driven) | 3 | 7 | 2 | 1 | 0 |
| E05: Auto-flush + tier-aware | 3 | 5 | 1 | 1 | 1 |
| E06: Benchmarks | 2 | 8 | 2 | 0 | 0 |
| **Итого** | **16** | **47** | **12** | **3** | **1** |

**Ожидаемые задачи:**
- FEAT-008: Partition index L1+ (5 SP)
- FEAT-009: Partition-based SST reader (5 SP)
- REF-004: Partition index persistence (2 SP)
- FEAT-010: MinMax per partition (2 SP)
- FEAT-011: Block skipping iterator (3 SP)
- FEAT-012: Leveled compaction planner (5 SP)
- REF-005: Atomic manifest swap (3 SP)
- PERF-002: Rate limiting I/O (token bucket) (2 SP)
- FEAT-013: Read amp tracker (3 SP)
- FEAT-014: SLA-driven trigger + cooldown + гистерезис (2 SP)
- FEAT-015: **Tier-aware compaction** (возврат hot keys в L3) (2 SP)
- FEAT-016: Backpressure на write path (2 SP)
- REF-003: SST footer versioning (перенесён из v0.1) (2 SP)
- REF-001: Arena free-list (перенесён из v0.1) (3 SP)
- TEST-007: Compaction trigger reasons (2 SP)
- DOC-004: BENCH-005 Compaction + BENCH-008 Read Amplification (4 SP)

**Ключевые ADR:** 003, 021.

**Решения HLD v11.2:** SLA-driven scheduler (D12, D90), bandwidth-aware admission control (D90), L0 partition (D89), tier-aware compaction.

**Exit criteria:**
- [ ] Read amp p99 < 3 на 10M ключей (YCSB C).
- [ ] WA < 5 (средняя).
- [ ] < 100 флашей на 1M ops.
- [ ] **Tier-aware compaction — T2 p999 не деградирует > 10%.**
- [ ] 500+ stars.

---

## 4. v0.3 — Value Log

**Цель:** разгрузить LSM от больших values, снизить WA, достичь read amp p99 < 2.
**Non-goals:** MVCC, транзакции, distributed.
**Base-срок:** +6 недель. **Risk-adjusted:** +7 недель. **SP:** 25. **Задач:** 11.

### 4.1. Эпики v0.3

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: VLog writer/reader | 3 | 10 | 3 | 0 | 0 |
| E02: VLog GC | 3 | 10 | 2 | 1 | 0 |
| E03: Benchmarks | 2 | 5 | 1 | 1 | 0 |
| E04: Windows/macOS prod-ready | 3 | TBD | 0 | 2 | 1 |
| **Итого** | **11** | **25** | **6** | **4** | **1** |

**Ожидаемые задачи:**
- FEAT-017: VLog segment (mmap) (5 SP)
- FEAT-018: VLog pointer format (2 SP)
- FEAT-019: VLog reader (3 SP)
- FEAT-020: Fragmentation tracker (3 SP)
- FEAT-021: VLog GC worker с **Titan-style WriteCallback** (5 SP) — D87
- TEST-009: VLog fragmentation + GC-snapshot consistency (2 SP)
- DOC-007: BENCH-006 VLog (update-heavy, range-heavy) (3 SP)
- DOC-008: v0.3 release notes (2 SP)
- FEAT-022: Windows build support (TBD) — Manifest `ReplaceFile` (D99)
- FEAT-023: macOS build support (TBD)
- TEST-010: Cross-platform CI (TBD)

**Ключевые ADR:** 022, 023.

**Решения HLD v11.2:** Titan-style WriteCallback (D87), WiscKey update-heavy degradation (D65), tiered bloom (D97), WAL segment конфигурируемый (D98).

**Exit criteria:**
- [ ] WA < 3 (средняя).
- [ ] Space amp < 1.3 (после GC).
- [ ] Read amp p99 < 2.
- [ ] Write stall отсутствует.
- [ ] GC-snapshot совместимость.
- [ ] **T2 p999 не деградирует > 10% для append-heavy workload.**
- [ ] 1000+ stars.

**Ограничения (явно):** update-heavy и range-heavy workload'ы требуют отдельной оценки (BENCH-006).

---

## 5. v0.4 — MVCC + Shard-per-core

**Цель:** транзакции, snapshot isolation, масштабирование на ядра.
**Non-goals:** distributed, SQL, HTAP.
**Base-срок:** +8 недель. **Risk-adjusted:** +9.5 недель. **SP:** 40. **Задач:** 13.

### 5.1. Эпики v0.4

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: MVCC | 4 | 13 | 4 | 0 | 0 |
| E02: Транзакции (single-shard) | 4 | 15 | 3 | 1 | 0 |
| E03: Shard-per-core | 3 | 10 | 3 | 0 | 0 |
| E04: Resource groups (концепт) | 1 | 2 | 0 | 1 | 0 |
| E05: Profile Compliance activation | 1 | TBD | 0 | 1 | 0 |
| **Итого** | **13** | **40** | **10** | **3** | **0** |

**Ожидаемые задачи:**
- FEAT-024: Version chain в arena (5 SP)
- FEAT-025: Timestamp oracle (2 SP)
- FEAT-026: Snapshot API (3 SP)
- FEAT-027: Auto-GC версий (3 SP)
- FEAT-028: Tx context + write set (5 SP)
- FEAT-029: Commit protocol (single-shard) (3 SP)
- FEAT-030: Rollback (2 SP)
- FEAT-031: Shard = {arena, memtable, WAL namespace} (5 SP)
- FEAT-032: Hash-based routing (3 SP)
- FEAT-033: Shard-local metrics (2 SP)
- ADR-038: Resource groups (концепт) (2 SP)
- FEAT-034: Profile Compliance activation — MVCC mode (TBD)

**Примечание:** shard-per-core (v0.4) — execution unit. Отображение 1 shard = 1 Raft group появляется в v0.6a (ADR-026 v5).

**Ключевые ADR:** 017, 018.

**Exit criteria:**
- [ ] 50M GET/s на 64-core (in-memory, single-shard).
- [ ] Snapshot Isolation корректна (property-тесты).
- [ ] Single-shard транзакции без потери latency.
- [ ] **T2 p999 не деградирует > 500 нс.**
- [ ] 2000+ stars.

---

## 6. v0.5 — TTL, PITR, Hardening

**Цель:** продакшн-готовность, первый платящий клиент.
**Non-goals:** distributed, SQL, HTAP.
**Base-срок:** +8 недель. **Risk-adjusted:** +9.5 недель. **SP:** 26. **Задач:** 10.

### 6.1. Эпики v0.5

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: TTL | 3 | 7 | 2 | 1 | 0 |
| E02: PITR | 3 | 10 | 2 | 1 | 0 |
| E03: Prometheus exporter | 2 | 5 | 1 | 1 | 0 |
| E04: Rate limiting | 1 | 2 | 0 | 1 | 0 |
| E05: Capacity model | 1 | 2 | 0 | 1 | 0 |
| **Итого** | **10** | **26** | **5** | **5** | **0** |

**Ожидаемые задачи:**
- FEAT-035: TTL в metadata SST (3 SP)
- FEAT-036: Auto-GC expired (3 SP)
- TEST-011: TTL metrics (1 SP)
- FEAT-037: WAL archive (3 SP)
- FEAT-038: Snapshot + replay API (5 SP) — `RestoreTo`
- TEST-012: PITR benchmark (2 SP)
- FEAT-039: Prometheus exporter (3 SP)
- DOC-010: Grafana dashboard (2 SP)
- FEAT-040: Token bucket per shard (2 SP)
- CHORE-008: **ADR-035 v2.3 Capacity model** (2 SP)

**Ключевые ADR:** 035 v2.3.

**Exit criteria:**
- [ ] Первый платящий пользователь.
- [ ] PITR работает на 100 ГБ данных.
- [ ] Prometheus exporter + Grafana dashboard.
- [ ] Capacity model (ADR-035 v2.3) опубликован.
- [ ] 3000+ stars.

---

## 7. v0.6a — Distributed KV Core

**Цель:** distributed KV на Dragonboat Multi-Raft поверх встроенного TCP.
**Non-goals:** SQL, columnar replica, multi-region, SSI, SOC2.
**Base-срок:** +12 недель. **Risk-adjusted:** +14 недель. **SP:** 78. **Задач:** 18.

### 7.1. Эпики v0.6a

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: Dragonboat Multi-Raft + Tan engine | 4 | 21 | 4 | 0 | 0 |
| E02: Custom LogDB (per-op durability) | 2 | 10 | 2 | 0 | 0 |
| E03: Snapshot + membership + learner | 3 | 11 | 2 | 1 | 0 |
| E04: 2PC | 2 | 8 | 2 | 0 | 0 |
| E05: Server mode (gRPC + REST) + CodecV2 | 2 | 10 | 1 | 1 | 0 |
| E06: Cluster 3–5 нод + integration | 2 | 6 | 1 | 1 | 0 |
| E07: Benchmarks | 3 | 12 | 2 | 1 | 0 |
| **Итого** | **18** | **78** | **14** | **4** | **0** |

### 7.2. Ключевые решения (из HLD v11.2)

| Компонент | Решение | Обоснование |
|---|---|---|
| Raft core | **Dragonboat** | 1.25M writes/s, Multi-Raft из коробки, Jepsen (D78) |
| Raft log storage | **Tan engine** | Log-file based LogDB, без space amplification (D101) |
| Per-op durability | **Кастомный LogDB** (`ILogDB`) | Документированный интерфейс Dragonboat (D102) |
| Raft transport | **Встроенный TCP Dragonboat** | Ближе к raw performance (D103) |
| Server mode | gRPC + REST | Внешний API, не Raft path |
| Codec | **CodecV2 + SharedBufferPool** | V1 bridge аллоцирует (D86) |
| Read path | ReadIndex (default) + Lease Read (опц.) | TiKV-прецедент (D64) |
| Lease Read clock | Monotonic raw clock (Instant) | Wall-clock drift breaks linearizability (D88) |
| Zero-alloc | Single-node hot path + LSM apply | Dragonboat — control plane, аллокации допустимы (D80) |
| `AllocsPerDistributedWrite` | ≤ 10 | Честная метрика (D104) |
| Wire format | Свой binary, кастомный codec | Zero-alloc на payload (D39, D70) |

### 7.3. Ожидаемые задачи

**E01: Dragonboat Multi-Raft + Tan engine:**
- FEAT-041: Dragonboat integration + NodeHost + 10000 Raft groups (8 SP)
- FEAT-042: Tan engine as LogDB (5 SP)
- FEAT-043: ReadIndex (default, clock-free) (5 SP)
- ADR-006 v5: Raft core = Dragonboat для v0.6a (3 SP)

**E02: Custom LogDB (per-op durability):**
- FEAT-044: Кастомный LogDB через `ILogDB` для per-operation durability (7 SP)
- ADR-010 v3: Per-op durability через кастомный LogDB (3 SP)

**E03: Snapshot + membership + learner:**
- FEAT-045: Incremental snapshot (chunked, resume) (5 SP)
- FEAT-046: Learner-based rolling replacement (3 SP)
- ADR-013 v2: Membership — static до v1.0, learner-based rolling (3 SP)

**E04: 2PC:**
- FEAT-047: 2PC coordinator + participant (5 SP)
- ADR-018 v2: Cross-shard 2PC (3 SP)

**E05: Server mode (gRPC + REST) + CodecV2:**
- FEAT-048: gRPC server + connection pooling + TLS 1.3 + interceptors (5 SP). **Acceptance:** framing control plane, TLS handshake control plane, CodecV2 registered, MaxConcurrentStreams: 250.
- FEAT-049: CodecV2 + SharedBufferPool + свой wire format (5 SP). **Acceptance:** codec hot path (zero-alloc), framing control plane, N Raft-групп на 1 stream.

**E06: Cluster 3–5 нод + integration:**
- FEAT-050: Cluster coordinator (static membership) (3 SP)
- FEAT-051: End-to-end integration test 3–5 nodes (3 SP)

**E07: Benchmarks:**
- DOC-012: BENCH-011 H-Score (3 SP)
- DOC-013: BENCH-012 Server Mode Overhead (3 SP)
- DOC-014: BENCH-013 Raft Under Partition + BENCH-014 Dragonboat Multi-Raft Throughput (6 SP). **Acceptance:** `AllocsPerDistributedWrite` измерен, Tan engine, кастомный LogDB.

**Ключевые ADR:** 005 v5, 006 v5, 009 v2, 010 v3, 011, 012 v6, 013 v2, 014, 015 v2, 016, 018 v2, 026 v5, 027, 036 v1.

**Exit criteria:**
- [ ] Freshness p99 < 1 с.
- [ ] **Dragonboat Multi-Raft throughput ≥ 1M ops/s на узел.**
- [ ] **`AllocsPerDistributedWrite` ≤ 10.**
- [ ] **Tan engine как Raft log storage.**
- [ ] **Кастомный LogDB для per-operation durability.**
- [ ] **Встроенный TCP Dragonboat для Raft.**
- [ ] 3–5 нод в проде.
- [ ] BENCH-011..014 пройдены.
- [ ] Transport overhead server mode: allocs/op ≤ 2 на codec (BENCH-012).
- [ ] SYNC_LEADER, SYNC_MAJORITY, SYNC_ALL работают.
- [ ] $100K ARR.
- [ ] 5000+ stars.

**Anti-KR:**
- Нет потери подтверждённых записей при partition (SYNC_MAJORITY).
- Нет потери оптимизаций Dragonboat из-за кастомного transport.
- Нет регрессии single-node T2 p999 > 500 нс при distributed.

**Capacity:** 2 инженера (найм 1 distributed-инженера до старта).

---

## 8. v0.6b — SQL + Columnar Replica

**Цель:** SQL subset + columnar replica поверх distributed KV core.
**Non-goals:** multi-region, SSI, SOC2.
**Base-срок:** +6 недель. **Risk-adjusted:** +7 недель. **SP:** 28. **Задач:** 7.

### 8.1. Эпики v0.6b

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: SQL subset + PostgreSQL wire | 2 | 12 | 1 | 1 | 0 |
| E02: Columnar replica (Raft Learner) | 2 | 8 | 1 | 1 | 0 |
| E03: TPC-C | 1 | 4 | 1 | 0 | 0 |
| E04: Resource isolation (cgroups) | 1 | 4 | 1 | 0 | 0 |
| E05: Encryption at rest (модуль) | 1 | TBD | 0 | 1 | 0 |
| **Итого** | **7** | **28** | **4** | **3** | **0** |

**Ожидаемые задачи:**
- FEAT-052: SQL subset parser + planner (8 SP)
- FEAT-053: PostgreSQL wire protocol (4 SP)
- FEAT-054: Columnar replica (Raft Learner) (5 SP)
- FEAT-055: Late materialization executor (3 SP)
- FEAT-056: TPC-C benchmark (4 SP)
- FEAT-057: Resource isolation (cgroups) (4 SP)
- FEAT-058: Encryption at rest — модуль (TBD)

**Ключевые ADR:** 024, 025, 027.

**Exit criteria:**
- [ ] SQL subset + PostgreSQL wire.
- [ ] Columnar replica (Raft Learner).
- [ ] TPC-C пройден.
- [ ] Resource isolation (cgroups).
- [ ] Encryption at rest (модуль).

**Anti-KR:**
- Нет деградации OLTP при OLAP > 20%.
- Нет двух executor'ов.

**Capacity:** 2 инженера.

---

## 9. v0.7 — Columnar

**Цель:** columnar storage + SIMD executor.
**Non-goals:** vector search, distributed columnar replica.
**Base-срок:** +8 недель. **Risk-adjusted:** +9 недель. **SP:** 30. **Задач:** 8.

### 9.1. Эпики v0.7

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: Columnar format | 5 | 15 | 4 | 1 | 0 |
| E02: SIMD AVX-512 + fallback | 2 | 10 | 2 | 0 | 0 |
| E03: Late materialization | 1 | 5 | 1 | 0 | 0 |
| **Итого** | **8** | **30** | **7** | **1** | **0** |

**Опционально:** миграция встроенный TCP → DRPC (ADR-005 v5), если bottleneck.

**Ключевые ADR:** 024, 025.

**Exit criteria:**
- [ ] Compression ratio ≥ 5×.
- [ ] Column pruning ≥ 90% savings.
- [ ] SIMD speedup ≥ 4× (AVX-512).
- [ ] Fallback на AVX2/NEON работает.
- [ ] BENCH-015 (Lease Read vs ReadIndex) опубликован.

---

## 10. v1.0 — Enterprise

**Цель:** enterprise-готовность, SOC2, multi-region, SLA.
**Base-срок:** +24 недели. **Risk-adjusted:** +28 недель. **SP:** 76. **Задач:** 8.

### 10.1. Эпики v1.0

| Эпик | Задач | SP | P0 | P1 | P2 |
|---|---|---|---|---|---|
| E01: Security | 4 | 21 | 3 | 1 | 0 |
| E02: Distribution (multi-region + SSI) | 2 | 26 | 2 | 0 | 0 |
| E03: Compliance (SOC2, SLA) | 2 | 29 | 2 | 0 | 0 |
| **Итого** | **8** | **76** | **7** | **1** | **0** |

**Опционально:** миграция DRPC → свой TCP (ADR-005 v5), если DRPC bottleneck.

**Ключевые ADR:** 004, 028, 029 v2, 030, 031, 032, 033.

**Exit criteria:**
- [ ] SOC 2 Type II.
- [ ] SLA 99.99%.
- [ ] Multi-region.
- [ ] SSI.
- [ ] Joint consensus.
- [ ] $1M ARR.
- [ ] 10000+ stars.
- [ ] Team 5+.

---

## 11. Deferred

| ID | Название | Причина | Дата пересмотра |
|---|---|---|---|
| DEF-001 | Векторный поиск | Не цель до v1.0 | 2027-Q2 |
| DEF-002 | Joint consensus | Static до v1.0 | v1.0 |
| DEF-003 | Multi-region | Требует distributed | v1.0 |
| DEF-004 | Server mode под zero-alloc | Принципиально невозможно (ADR-030) | никогда |
| DEF-005 | Shared Bloom (per-key) | Отложено, per-SSTable достаточно | v0.5 |
| DEF-006 | Calvin-style deterministic txn | Сложно, не нужно | после v1.0 |
| DEF-007 | QUIC для cross-region | Не для intra-cluster | v1.0+ |
| DEF-008 | Свой Raft | v1.0+, если команда 5+ (D79) | v1.0+ |
| DEF-009 | RBAC | Enterprise-фича | v1.0 |
| DEF-010 | Multi-tenant isolation | Enterprise-фича | v1.0 |

---

## 12. Cancelled

| ID | Название | Причина |
|---|---|---|
| CAN-001 | Cross-shard без 2PC | Теоретически невозможно (ADR-018) |
| CAN-002 | TPC-C в v0.4 | Требует cross-shard (перенесено в v0.6b) |
| CAN-003 | sync.Pool на hot path | Нарушает zero-alloc (ADR-009 v2, D68) |
| CAN-004 | $15B valuation | Не цель проекта (HLD v11.2 §1) |
| CAN-005 | etcd/raft для v0.6a | Отменено. Dragonboat (D78) |
| CAN-006 | Собственный QUIC для intra-cluster | 66% throughput, 12× RAM (ADR-005 v5) |
| CAN-007 | Jepsen (Clojure) | Knockbox (Go) достаточно (ADR-016) |
| CAN-008 | TLA+ refinement mapping | Модель + property-тесты достаточно (ADR-015 v2) |
| CAN-009 | gRPC Codec V1 | Аллоцирует на каждый message. CodecV2 + SharedBufferPool (D86) |
| CAN-010 | TephraDB (название) | Переименовано в TephraKV |
| CAN-011 | WAL per shard | Shared WAL pool с per-shard LSN namespace (D53, D62) |
| CAN-012 | Свой gRPC transport для Raft | Отменено. Встроенный TCP Dragonboat (D106) |
| CAN-013 | Собственный Multi-Raft core | Отменено. Dragonboat (D78, D79) |
| CAN-014 | p999 < 5 мс (in-memory) | Отменено. Tiered-цели (D107) |

---

## 13. Метрики бэклога

| Метрика | Цель | Текущее | Статус |
|---|---|---|---|
| WIP | ≤ 3 | 0 | ok |
| Cycle time | ≤ 3 дня | — | — |
| Lead time | ≤ 5 дней | — | — |
| Throughput | 1–3 задачи/день | 0 | — |
| Blocked ratio | < 10% | 0% | ok |
| P0 blocked | 0 | 0 | ok |
| SP выполнено за неделю | ≥ 12 | 0 | — |
| Задач в BACKLOG без DoR | 0 | 125 | fail |
| Задач SP > 8 | 0 | 0 | ok |

**Action items:**
- Заполнить DoR для всех P0-задач v0.1 (21 задача) — до старта недели.
- Написать API-001, FORMAT-001 (критично до кода).
- Обновить ADR-034, ADR-035, ADR-036 ссылки на HLD v11.2.
- Написать **ADR-039 «Dragonboat dependency risk»** (версия, пиннинг, план B).
- Написать **ADR-040 «Migration v0.5 → v0.6a»** (WAL pool → Tan engine).
- Написать **ADR-011 v3 «Distributed consistency model»**.

---

## 14. Что дальше

1. Согласовать формат — с Екатериной.
2. Заполнить DoR для P0-задач v0.1 (21 задача).
3. Написать API-001, FORMAT-001.
4. Обновить ADR-034, ADR-035, ADR-036 ссылки на HLD v11.2.
5. Начать CHORE-001 — bootstrap.

Правило: BACKLOG обновляется ежедневно.

---

## 15. Шаблон задачи

```markdown
#### <TYPE>-<NNN>: <Название>

| Поле | Значение |
|---|---|
| Версия | vX.Y |
| Приоритет | P0 / P1 / P2 / P3 |
| Статус | todo / dor / ready / in-progress / blocked / review / done / cancelled / deferred |
| SP | N |
| Владелец | @user |
| Зависимости | <TYPE>-NNN, ADR-NNN |
| ADR | ADR-NNN |
| BENCH | BENCH-NNN |
| Критпуть | yes / no |

Контекст: 1–3 предложения.

Acceptance criteria:
- [ ] Критерий 1
- [ ] Критерий 2

Артефакты:
- path/to/file.go
- docs/adr/ADR-NNN.md

Заметки:
- Всё, что не влезло.
```

DoR и DoD не дублируются в задаче. Чек-листы — в §0.8 и §0.9.

---

**Конец TephraKV-BACKLOG-001 v6.0**

> **Замечание Тэфри:** v6.0 синхронизирован с HLD v11.2 и ROADMAP v4.0. Ключевые правки: tiered p999 для v0.1 (T2 < 500 нс), BENCH-016 (DOC-011), Dragonboat вместо etcd/raft, Tan engine, кастомный LogDB, встроенный TCP, разбиение v0.6 на v0.6a (78 SP) + v0.6b (28 SP), SP 448 (было 453), задач 125 (было 126). REF-001 перенесён в v0.2. `TierDistribution` в InternalMetrics. `SYNC_MASTER = iota` (0) — безопасный default. `Get` возвращает `(value, release)`.