# TephraKV — ADR-034: Engineering Practices (Solo Context)

**Документ:** TephraKV-ADR-034
**Версия:** 2.0
**Статус:** Proposed
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v7.0, TephraKV-PRD-001 v1.0, TephraKV-ROADMAP-001 v3.0, TephraKV-BACKLOG-001 v5.0, TephraKV-GLOSSARY-001 v2.0, ADR-035 v2.0
**Заменяет:** TephraKV-ADR-034 v1.0 (TephraDB)
**Тип:** Architecture Decision Record

---

## 1. Контекст

TephraKV — solo-проект. ROADMAP v3.0 §8.1 фиксирует: 1 инженер до v0.6, найм 1 distributed-инженера к v0.6, команда 5+ к v1.0. Значит, практики нужно отбирать по критерию: **работают ли они на одного человека, или превращаются в ритуал ради ритуала**.

Ключевое ограничение solo: **нет внешнего архитектурного критика**. Менеджер и архитектор — один человек. ADR пишет тот же, кто пишет код. Плюс: нет разрыва. Минус: нет независимой проверки. Это меняет роль fitness functions — они становятся **единственным автоматическим критиком**.

HLD v7.0 добавляет новые технические требования, которые влияют на engineering practices:

- **Zero-alloc контракт** распространяется на transport message codec (CodecV2 + SharedBufferPool, D86).
- **`sync.Pool` запрещён в data plane**, допустим в control plane (D68).
- **Arena** — mmap-backed bump allocator, не `sync.Pool` (D85).
- **Zero-alloc гарантируется CI-бенчмарком с `-benchmem`**, не escape analysis (D69).
- **Shared WAL pool** с per-shard LSN namespace (D53, D62).
- **etcd/raft для v0.6**, свой Raft — v1.0+ (D78, D79).
- **Профили** Latency / Compliance / Custom (D71–D77).
- **Два threat model** (storage v0.1, network v0.6), разделены (D57, D67).

Изучен документ о корпоративных инженерных практиках (орг-масштаб: stream-aligned teams, RACI, Conway's Law, platform team, 180-дневный rollout, бюджет 15–20% на архитектуру). Документ сильный, но написан для организации, не для solo-инженера.

---

## 2. Решение

### 2.1. Что берём (работает на solo)

| Практика | Адаптация | Артефакт |
|---|---|---|
| **ADR** | Один ADR = одно решение. Immutable: передумал — новый ADR, старый `Superseded by`. | `docs/adr/`, реестр в `docs/adr/README.md` |
| **Bounded contexts** | Модули одного бинарника с явным API, запрет циклов. Модули из HLD v7.0 §5.3: arena, memtable, wal, sstable, lsm, vlog, mvcc, txn, shard, raft, transport, metrics, manifest, profile, audit, crypto, pitr, ttl. | `internal/*`, `go-arch-lint` |
| **API-first** | Семантика операций (durability, consistency, isolation) фиксируется в ADR до реализации. API-001 обязателен до кода. | ADR-034 (API semantics — отдельно), API-001 |
| **Format-first** | Формат данных (WAL, SST, manifest, wire) фиксируется в FORMAT-001 до реализации. | FORMAT-001 |
| **DoR / DoD** | Короткие чек-листы. Защита от «начал, не додумал, переписал». | BACKLOG-001 v5.0 §0.8, §0.9 |
| **Fitness functions** | Автоматические CI-гейты как единственный архитектурный критик. | см. §2.2 |
| **SLO / SLI / Error budget** | Самодисциплина: нарушил p999 — не добавляешь фичи, чинишь. | ENG-001 §7, `docs/SLO.md` |
| **Capacity model** | Формальная модель ресурсов с бизнес-контекстом. | ADR-035 v2.0 |
| **DORA-метрики** | Для solo — способ видеть, где буксуешь. | BACKLOG-001 v5.0 §0.11 |
| **Two-Level Metrics** | Каждая внешняя метрика сопровождается внутренней. Обязательно. | ADR-020, HLD v7.0 §4.3 |

### 2.2. Fitness functions (CI-гейты)

Обязательные гейты на каждый PR:

| Гейт | Инструмент | Что проверяет | Провал |
|---|---|---|---|
| Zero-alloc | `testing.AllocsPerRun` + build tag | Allocs/op == 0 на hot path (Put, Get, Delete, Scan, WriteBatch, raft apply, transport codec) | Block merge |
| CI-бенчмарк `-benchmem` | `go test -bench -benchmem` на каждой версии Go | Allocs/op == 0. **Единственная гарантия zero-alloc** (D69) | Block merge |
| Escape analysis | `-gcflags=-m` | Нет новых heap escapes на hot path. **Диагностика, не контракт** (D69) | Block merge |
| Race | `go test -race` | Data races | Block merge |
| Архитектура | `go-arch-lint` | Нет циклов между модулями (18 модулей из HLD v7.0 §5.3) | Block merge |
| Benchmark regression | `benchstat` | p99 не хуже baseline > 5% | Block merge |
| Линтеры | `golangci-lint` + `staticcheck` | Стиль, баги, антипаттерны | Block merge |
| Запрет `sync.Pool` (data plane) | `depguard` + `forbidigo` | `sync.Pool` только в `internal/transport/framing`, `internal/transport/handshake` | Block merge |
| Запрет `fmt`, `reflect`, `interface{}` | `depguard` + `forbidigo` | Только в `internal/arena`, `internal/memtable`, hot path | Block merge |
| Уязвимости | `govulncheck` | CVE в зависимостях | Block merge |
| Тесты | `go test ./...` | Unit, race, fuzz | Block merge |
| CodecV2 | `go test -bench` | gRPC codec = CodecV2 (не V1 bridge) | Block merge |

Правило: **fitness functions — единственный архитектурный критик solo-проекта**. Если гейт не автоматизирован — он не работает (самопроверка не считается).

**Почему CI-бенчмарк, а не escape analysis:**

Escape analysis не стабилен между версиями Go. Новый pass может «break code that happened to work before». **CI-бенчмарк с `-benchmem` на каждой версии Go — единственная гарантия zero-alloc** (D69). Escape analysis — диагностика, не контракт.

### 2.3. Что не берём (преждевременно для solo)

| Практика | Почему не сейчас | Когда вернуться |
|---|---|---|
| Conway's Law, stream-aligned teams | Нет команд | При 3–5 инженерах |
| Platform team, enabling team | Нет людей | При 5+ инженерах |
| Architecture Guild | Нет гильдии | При 10+ инженерах |
| RACI | Solo — всё R, A, C, I | При 3+ инженерах |
| Двухтрековый Discovery/Delivery | Solo не тянет два трека параллельно | При 4+ инженерах |
| Бюджет 15–20% на архитектуру | Solo — весь бюджет | При 5+ инженерах |
| 180-дневный rollout с ролями | Для орг-внедрения, не для solo | При 10+ инженерах |
| Feature flags, canary, trunk-based | Только для server mode / SaaS | При server mode (v0.6+) |
| Дизайн-ревью комитет | Нет комитета | При 5+ инженерах |
| Multi-tenant isolation | Enterprise-фича | v1.0 |

Это не «плохо». Это **другой масштаб**. Документ-источник остаётся в силе при росте команды.

### 2.4. Роль ADR в solo-контексте

ADR — главный инструмент. Правила:

1. **ADR до кода.** Не «потом напишем».
2. **Один ADR = одно решение.** Не «ADR-про-всё».
3. **ADR immutable.** Передумал — новый ADR, старый помечается `Superseded by ADR-XXX`.
4. **Шаблон:** контекст → решение → альтернативы → последствия → статус → дата.
5. **Внешний критик:** публикация ADR в open-source. Feedback сообщества — замена внутреннему ревью.
6. **Ссылки обязательны:** HLD, PRD, ROADMAP, BACKLOG, BENCH.
7. **D-номера:** ADR синхронизирован с Decision Log в HLD v7.0 §10 (D71–D90).

**Критические ADR до кода (v0.1):**

| ADR | Тема | Критпуть |
|---|---|---|
| ADR-001 | Lock-free skiplist vs B-tree | yes |
| ADR-002 | WAL format | yes |
| ADR-009 v2 | Zero-alloc scope: framing vs codec; sync.Pool | yes |
| ADR-010 | Durability semantics (5 политик) | yes |
| ADR-012 v3 | WAL shared pool с per-shard LSN namespace | yes |
| ADR-019 | Arena + epoch reclamation | yes |
| ADR-020 | Metrics definitions | yes |
| ADR-037 | Profile semantics (Latency / Compliance / Custom) | no |

### 2.5. DoR / DoD (краткая форма)

**DoR (Definition of Ready):**
- Проблема сформулирована (Y → Z, потому что W).
- Acceptance criteria записаны (конкретные, проверяемые).
- ADR написан (если решение значимое).
- BENCH определён (если perf-critical).
- Zero-alloc оценено (hot path или нет).
- Влияние на формат данных оценено (storage / wire).
- Зависимости проверены.
- SP ≤ 8.

**DoD (Definition of Done):**
- Код проходит CI (build, vet, race, lint, zero-alloc, arch-lint, bench regression, CodecV2).
- Тесты: unit + race + fuzz (если применимо).
- Бенчмарк не деградировал > 5% (p99).
- AllocsPerRun == 0 для hot path (или документированное исключение в ADR).
- ADR обновлён (если применимо).
- CHANGELOG обновлён.
- Документация обновлена (godoc, README при смене API).

Полные версии — в BACKLOG-001 v5.0 §0.8, §0.9.

### 2.6. SLO / SLI / Error budget

| SLI | SLO | Error budget (28 дней) |
|---|---|---|
| p99 GET | < 1 мс (in-memory, single-node) | 0.1% |
| p999 GET | < 5 мс (in-memory, single-node) | 0.1% |
| Read amp p99 | < 3 (v0.1–v0.2) / < 2 (v0.3+) | 0.1% |
| WA (средняя) | < 5 (v0.1–v0.2) / < 3 (v0.3+) | 0.1% |
| Allocs/op hot path | 0 | 0% (нарушение = баг) |
| Allocs/op transport codec | ≤ 2 (v0.6) / ≤ 1 (v0.7+) | 0.1% |
| p99 GET (distributed, v0.6+) | < 10 мс | 0.1% |
| Freshness p99 (v0.6+) | < 1 с | 0.1% |

Правило: **error budget сгорел — фичи замораживаются, только надёжность**. Для solo — самодисциплина, не отчётность.

### 2.7. Capacity model

ADR-035 v2.0 фиксирует: RAM/CPU/Disk на N ключей по сегментам (HFT, game servers, AI-infra, edge, CDN, fintech), differentiation budget (где тратим, где экономим), TCO per unit, transport memory (gRPC 100–250 streams/conn, < 20 МБ), arena performance (2.9 нс vs 40 нс), shared WAL pool O(1), CodecV2 бенчмарк (2.4×/2.7×/300×).

Обновляется при каждом ADR, влияющем на ресурсы (Bloom, VLog, MVCC, columnar, audit, encryption, CodecV2).

### 2.8. DORA-метрики для solo

| Метрика | Что показывает | Где хранится |
|---|---|---|
| Release cadence | Deployment frequency | ROADMAP v3.0 §4.5 |
| Lead time (ADR → merge) | Скорость решений | BACKLOG v5.0 §0.11 |
| Change failure rate | % релизов с hotfix | `docs/reports/` |
| MTTR (S1/S2) | Время от бага до фикса | `docs/incidents/` |
| WIP | Задачи в `in-progress` | BACKLOG v5.0 §0.11 |
| Throughput | SP/нед | BACKLOG v5.0 §0.11 |

Это не для отчётности — это для **самопроверки**. Видеть, где буксуешь.

### 2.9. Two-Level Metrics как обязательный контракт

Каждая внешняя метрика сопровождается внутренней (HLD v7.0 §4.3, ADR-020). Обязательно в каждом релизе.

| Внешний уровень | Внутренний уровень |
|---|---|
| p50/p99/p999 задержки | Cycles per operation |
| Throughput | Байт прочитано / записано на операцию |
| TCO на млн операций | Аллокаций на операцию |
| Freshness (v0.6+) | Флашей и компакций на 1M операций |
| Transport p999 (v0.6+) | Allocs/op на codec |

### 2.10. Профили как engineering practice

HLD v7.0 §6.1 вводит профили (Latency / Compliance / Custom). Это **конфигурация, не форк**.

| Профиль | Default durability | MVCC | PITR | Audit | Encryption | Compression |
|---|---|---|---|---|---|---|
| Latency | NO_SYNC | off | off | off | off | none |
| Compliance | SYNC_MASTER | on | on | on | on | ZSTD |
| Custom | user | user | user | user | user | user |

**Правило:** ядро не знает о профилях. Профили — наборы параметров. Модули подключаются через интерфейсы. `ProfileCompliance` в v0.1 возвращает `ErrNotImplemented` (модули не готовы).

### 2.11. Zero-alloc scope: контроль

| Слой | Hot path? | Zero-alloc? | sync.Pool? |
|---|---|---|---|
| TLS handshake | Нет | Нет | Допустим (control plane) |
| TLS AEAD (data) | Да | Да (in-place) | Запрещён |
| gRPC framing (HTTP/2) | Нет | Нет | Допустим (control plane) |
| gRPC message codec (payload) | Да | Да (CodecV2 + SharedBufferPool) | Запрещён (свой arena) |
| netpoller epoll event | Да | Да (preallocated) | Запрещён |
| arena | Да | Да (mmap bump) | — |
| memtable | Да | Да (offsets) | Запрещён |
| Raft core (etcd/raft) | Нет (control plane) | Нет | Допустим |
| Raft log/apply | Да | Да (свой) | Запрещён |
| Server mode (v0.6+) | Нет | Нет (ADR-030) | Допустим |

Правило: **граница zero-alloc проходит по data plane vs control plane**. Нарушение — block merge.

### 2.12. Shared WAL pool: engineering practice

Shared WAL pool с per-shard LSN namespace (D53, D62) — не только архитектурное, но и engineering-решение:

- O(1) по числу открытых файлов, не O(N_shards).
- 10000 shards не создают 10000 WAL.
- Recovery: WAL replay с фильтрацией по shard, параллельный per shard, с лимитом concurrent (не более N shard одновременно, чтобы не насытить I/O).
- Формат WAL включает term/index поля (нулевые в single-node) — совместимость v0.4 → v0.6.

**Engineering implication:** recovery test (TEST-003) должен покрывать сценарий 10000 shards в shared WAL, с лимитом concurrent.

---

## 3. Последствия

### 3.1. Положительные

- 4 артефакта (ADR, fitness functions, DoR/DoD, capacity model) дают solo-проекту архитектурную дисциплину без корпоративной обвязки.
- Fitness functions автоматизируют роль внешнего критика.
- **12 CI-гейтов** (было 8 в v1.0) покрывают новые требования HLD v7.0: CodecV2, sync.Pool data plane, arch-lint 18 модулей.
- **CI-бенчмарк с `-benchmem`** — единственная гарантия zero-alloc (D69).
- SLO/error budget защищают от «добавим фичу, потом починим».
- ADR в open-source даёт внешний feedback без найма.
- Профили (Latency / Compliance) — engineering practice, не форк.
- Shared WAL pool — O(1) по файлам, engineering test coverage.

### 3.2. Отрицательные

- Документация (ADR + HLD + PRD + ROADMAP + BACKLOG + GLOSSARY + BENCH + API + FORMAT) занимает время. Контроль: не > 20% (ROADMAP §8.2).
- Нет внешнего критика, кроме CI и сообщества. Риск: слепые зоны.
- DoR/DoD для solo звучит как ритуал. Реальность: защита от «начал, не додумал».
- 12 CI-гейтов — больше времени на настройку. Но каждое нарушение = блок merge, что дешевле, чем баг в проде.

### 3.3. Нейтральные

- При росте команды — возврат к орг-практикам (RACI, Conway, platform team). Документ-источник сохраняется.

---

## 4. Alternatives

| Альтернатива | Почему отклонена |
|---|---|
| Применить орг-практики полностью | Solo не тянет RACI, Guild, platform team — ритуал без пользы |
| Не применять ничего | Solo-проекты без ADR и fitness functions деградируют быстрее |
| Только ADR, без fitness functions | Нет автоматического критика — ADR не проверяется кодом |
| Только fitness functions, без ADR | Нет фиксации решений — «я всё помню в голове» |
| Escape analysis как контракт (вместо CI-бенчмарка) | Escape analysis не стабилен между версиями Go (D69) |
| sync.Pool в data plane | Нарушает детерминированный lifetime (D68) |
| Zero-alloc на server mode | Принципиально невозможно (ADR-030) |
| Zero-alloc на Raft core (etcd/raft) | etcd/raft не проектировался под zero-alloc. Control plane, аллокации допустимы (D80) |

---

## 5. Ссылки

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v7.0 | Архитектура, §3.2 (zero-alloc), §4.4 (принцип 21: CI-гейты), §5.3 (18 модулей), §6.8 (CodecV2), §6.7 (shared WAL), §10 (D71–D90) |
| TephraKV-PRD-001 v1.0 | ICP, use cases, GTM, pricing |
| TephraKV-ROADMAP-001 v3.0 | §8.1 Capacity, §8.2 Временной бюджет, §11 Cross-cutting |
| TephraKV-BACKLOG-001 v5.0 | §0.8 DoR, §0.9 DoD, §0.11 Метрики, §2–§9 задачи |
| TephraKV-GLOSSARY-001 v2.0 | ADR, Fitness function, SLO, SLI, Error budget, CodecV2, Shared WAL pool, Profile |
| ADR-009 v2 | Zero-alloc scope: framing vs codec; sync.Pool |
| ADR-012 v3 | WAL shared pool с per-shard LSN namespace |
| ADR-020 | Metrics definitions |
| ADR-035 v2.0 | Capacity model |
| ADR-036 | Read path: ReadIndex + Lease Read |
| ADR-037 (proposed) | Profile semantics |
| API-001 | API Specification (критично до кода) |
| FORMAT-001 | Data Format Specification (критично до кода) |

---

## 6. Что дальше

1. **Утвердить ADR-034 v2.0** → статус `Accepted`.
2. **Синхронизировать ENG-001:** добавить ссылку на ADR-034 v2.0 в §1 (ADR) и §6 (Fitness functions).
3. **Добавить `docs/SLO.md`** — формализация §2.6.
4. **Добавить `docs/incidents/`** — шаблон постмортема для MTTR.
5. **Проверить CI:** все **12 гейтов** из §2.2 реализованы в CHORE-002 (CI skeleton) и CHORE-006 (go-arch-lint).
6. **Написать API-001, FORMAT-001** — критично до кода.
7. **Обновить ADR-035 v2.0** — ссылка на ADR-034 v2.0.

---

## 7. Открытые вопросы (честно)

- **Внешний feedback:** публикация ADR в open-source — гипотеза. Не измерено, придёт ли feedback.
- **Fitness functions coverage:** 12 гейтов — достаточно? Не знаем. Может потребоваться больше (например, mutation testing).
- **DoR/DoD для solo:** работает ли чек-лист на одного, или превращается в ритуал? Не знаем. Проверяется на первых 4 неделях v0.1.
- **DORA-метрики solo:** имеют ли смысл без команды? Гипотеза — да, для самопроверки. Не измерено.
- **CodecV2 gate:** как именно проверять, что используется V2, а не V1 bridge? Не решено. Вариант: reflection + benchmark.
- **sync.Pool gate:** как именно проверять, что `sync.Pool` не используется в data plane? `depguard` + `forbidigo`. Не решено, достаточно ли.
- **Shared WAL recovery test:** покрывать ли 10000 shards в CI? Дорого. Может быть, только 1000. Не решено.
- **Профили в v0.1:** `ProfileCompliance` возвращает `ErrNotImplemented`. Как это тестировать? Не решено.

**Правило:** неизвестное документируется как неизвестное.

---

## 8. Что изменилось против v1.0

**Синхронизация с HLD v7.0, PRD v1.0, ROADMAP v3.0, BACKLOG v5.0, GLOSSARY v2.0, ADR-035 v2.0.**

Содержательные правки:

- **Название:** TephraDB → TephraKV.
- **Ссылки:** HLD-000 v5.1 → v7.0; BACKLOG v4.1 → v5.0; ROADMAP v2.2 → v3.0; GLOSSARY v1.1 → v2.0; ADR-035 v1.0 → v2.0.
- **§2.1:** добавлены API-first, Format-first, Two-Level Metrics. Модули: 18 из HLD v7.0 §5.3 (добавлены profile, audit, crypto, pitr, ttl, mvcc, vlog).
- **§2.2:** CI-гейтов 12 (было 8). Добавлены: CI-бенчмарк `-benchmem` (D69), запрет `sync.Pool` (data plane, D68), запрет `fmt`/`reflect`/`interface{}`, CodecV2.
- **§2.4:** ADR до кода — 8 критических ADR для v0.1 (было 6). Добавлены ADR-009 v2, ADR-012 v3.
- **§2.6:** SLO обновлены: p999 < 5 мс (не мкс), read amp < 3 / < 2, WA < 5 / < 3, transport codec allocs/op ≤ 2, distributed p99 < 10 мс, freshness < 1 с.
- **§2.7:** Capacity model — ссылка на ADR-035 v2.0. Добавлены transport memory, arena performance, shared WAL pool, CodecV2.
- **§2.10:** новый — Профили как engineering practice (Latency / Compliance / Custom).
- **§2.11:** новый — Zero-alloc scope: контроль (таблица).
- **§2.12:** новый — Shared WAL pool: engineering practice.
- **§7:** открытые вопросы обновлены (CodecV2 gate, sync.Pool gate, shared WAL recovery test, профили в v0.1).
- **§8:** новая секция «Что изменилось».

Структурные правки:

- §2.10, §2.11, §2.12 — новые секции.
- §8 — новая секция.

Ссылки:

- HLD-000 v5.1 → v7.0.
- PRD-001 v1.0 — добавлен.
- ROADMAP-001 v2.2 → v3.0.
- BACKLOG-001 v4.1 → v5.0.
- GLOSSARY-001 v1.1 → v2.0.
- ADR-035 v1.0 → v2.0.
- ADR-036, ADR-037 — добавлены.
- API-001, FORMAT-001 — добавлены.

---

**Конец ADR-034 v2.0**
