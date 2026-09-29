# TephraKV — Roadmap

**Документ:** TephraKV-ROADMAP-001
**Версия:** 4.0
**Статус:** Active
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v11.2, TephraKV-PRD-001 v1.3, TephraKV-GTM-001 v2.3, TephraKV-PRICING-001 v1.3, TephraKV-BACKLOG-001 v6.4, TephraKV-GLOSSARY-001 v3.4, TephraKV-ENG-001 v1.2, ADR-005 v5, ADR-006 v5, ADR-010 v3, ADR-012 v6, ADR-026 v5, ADR-034 v2.2, ADR-035 v2.3, ADR-036 v1
**Заменяет:** TephraKV-ROADMAP-001 v3.0

---

## 0. Назначение и правила

### 0.1. Что изменилось против v3.0

**Синхронизация с HLD v11.2. Ключевое: tiered-цели по хвостовой задержке (наносекунды), Dragonboat вместо etcd/raft, разбиение v0.6 на v0.6a + v0.6b.**

1. **Tiered latency goals:** p999 GET переформулирован в cache-hierarchy terms. T1 (L1/L2) < 200 нс, T2 (L3) < 500 нс, T3 (RAM) < 2 мкс, T4 (NVMe) < 50 мкс.
2. **Raft core v0.6a:** Dragonboat (не etcd/raft). v3.0 содержал противоречие с HLD v11.x.
3. **Raft transport v0.6a:** встроенный TCP Dragonboat (не gRPC). gRPC — только server mode.
4. **v0.6 разбит:** v0.6a (78 SP, KV core) + v0.6b (28 SP, SQL + columnar).
5. **SP:** v0.1 98 (было 102), итого 448 (было 452). +3 SP на BENCH-016 (Tiered Latency).
6. **OKR:** переформулированы в tiered-терминах.
7. **Non-goals:** добавлены Dragonboat-only, Tan engine, встроенный TCP.
8. **Риски:** недостижение T2/T3, деградация T2 при cache miss, интеграция с Tan engine, кастомный LogDB, `ILogDB` нестабильность.
9. **Unknowns:** tier distribution, достижимость T2/T3, Dragonboat throughput на 10000 groups.
10. **Новые BENCH:** BENCH-016 (Tiered Latency).
11. **Ссылки:** HLD v7.0 → v11.2, PRD v1.0 → v1.3, BACKLOG v4.1 → v6.4, GLOSSARY v1.1 → v3.4.

### 0.2. Что это

ROADMAP — стратегический документ. Отвечает на вопросы:

- Куда идёт проект (vision, objectives).
- Почему именно так (обоснование последовательности).
- Когда что выходит (timeline: base + risk-adjusted).
- Как измеряем успех (OKR, exit criteria).
- Чего не делаем (non-goals).
- Что неизвестно (честно).

ROADMAP не отвечает на вопросы «как именно» и «кто делает задачу N» — это BACKLOG.

### 0.3. Разделение с BACKLOG

| Вопрос | ROADMAP | BACKLOG |
|---|---|---|
| Куда идём? | yes | — |
| Почему так? | yes | — |
| Когда релиз? | yes | — |
| Какой теме посвящён релиз? | yes | — |
| Как измеряем успех релиза? | yes | — |
| Какие задачи в релизе? | — | yes |
| Кто владелец задачи? | — | yes |
| Какой DoR/DoD? | — | yes |
| Критический путь? | yes (сводно) | yes (детально) |

Правило: ROADMAP меняется редко (раз в квартал или при значимом ADR). BACKLOG меняется ежедневно.

### 0.4. Источник чисел

Все SP и количество задач берутся из BACKLOG-001 v6.4. Если числа расходятся — BACKLOG прав, ROADMAP правится.

| Версия | Задач | SP |
|---|---|---|
| v0.1 | 35 | 98 |
| v0.2 | 16 | 47 |
| v0.3 | 11 | 25 |
| v0.4 | 13 | 40 |
| v0.5 | 10 | 26 |
| **v0.6a** | **—** | **78** |
| **v0.6b** | **—** | **28** |
| v0.7 | 8 | 30 |
| v1.0 | 8 | 76 |
| **Итого** | **—** | **448** |

### 0.5. Стандарты планирования

1. Outcome over output — каждая версия имеет измеримый результат (OKR).
2. Now / Next / Later — горизонты разной точности: v0.1–v0.2 детально, v0.3–v0.5 средне, v0.6–v1.0 концептуально.
3. Theme per version — одна тема. Не смешивать.
4. Appetite (Shape Up) — время фиксировано, объём переменный. Не укладываемся — режем scope.
5. Dependencies explicit — критический путь виден.
6. Risk-adjusted — два плана: base и risk-adjusted.
7. Explicit non-goals — что не делаем, до какой версии.
8. Capacity visible — SP, время, бюджет видны.
9. Re-planning triggers — при каких условиях roadmap пересматривается.
10. Honest unknowns — неизвестное помечено как неизвестное.

### 0.6. Горизонты точности

| Горизонт | Версии | Точность | Пересмотр |
|---|---|---|---|
| Now | v0.1, v0.2 | Высокая. Задачи в BACKLOG, SP оценены. | Раз в неделю |
| Next | v0.3, v0.4, v0.5 | Средняя. Эпики, SP, но не задачи. | Раз в месяц |
| Later | v0.6a, v0.6b, v0.7, v1.0 | Концептуальная. Темы, OKR, ключевые решения. | Раз в квартал |

### 0.7. Re-planning triggers

- Закрыта очередная версия.
- Benchmark деградировал > 5%.
- ADR меняет архитектуру.
- SP-оценка v0.N отличается от реальной > 30%.
- Найм (или отсутствие) меняет темп.
- Design partner запрашивает фичу вне roadmap.
- Квартальный review.
- **T2 (L3-resident, p999 < 500 нс) не достигнут на BENCH-016.**

---

## 1. Vision и позиционирование

### 1.1. Vision (горизонт 20–24 месяца)

TephraKV — встраиваемое KV-хранилище на Go, которое даёт **детерминированную хвостовую задержку в tiered-терминах** (p999 < 500 нс для L3-resident working set) без GC-пауз, с per-operation durability, и работает как single-node, так и в распределённом режиме на **Dragonboat Multi-Raft** поверх встроенного TCP (v0.6a) → DRPC (v0.7+) → собственного TCP (v1.0+).

### 1.2. Outcomes (к концу v1.0)

1. **Технически:** работающий продукт с измеримыми преимуществами в tiered p999, read amp, zero-alloc.
2. **Репутационно:** узнаваемость в Go-сообществе (10K+ stars), упоминание в статьях и докладах.
3. **Коммерчески:** первый платящий клиент (v0.5), $1M ARR (v1.0), реалистичный путь к acqui-hire или enterprise-support.
4. **Инженерно:** дисциплинированный процесс (ADR, DoR/DoD, CI-гейты, SLO), сохраняющий качество.

### 1.3. Позиционирование

**Embedded Go KV with deterministic tail latency. Zero allocations on hot path. Per-operation durability. Dragonboat Multi-Raft. p999 GET < 500 нс (T2: L3-resident, 1M keys, single-node).**

Три слова: **deterministic**, **zero-allocation**, **per-operation**.

| От | Чем отличаемся |
|---|---|
| Badger/Pebble | Zero-alloc hot path, per-operation durability, tiered p999, read amp p99 < 2 (v0.3+) |
| Redis | Embedded, persistent, per-op durability |
| ScyllaDB | Embedded, Go, без LSM-хвостов в p999 |
| Aerospike | Embedded, без RAM-стоимости индекса |
| TiKV/CockroachDB | Embedded-first, tiered p999, Dragonboat Multi-Raft |

### 1.4. Целевые сегменты (HLD v11.2 §3.1)

| Сегмент | ICP | Бюджет | Триггер |
|---|---|---|---|
| **HFT / trading infra** | CTO / VP Eng, 20–200 инженеров | $50–200K/год | Инцидент с p999 |
| **Game servers** | Tech Lead, 10–100 инженеров | $10–50K/год | Жалобы игроков на лаги |
| **AI-infra (feature stores)** | ML Infra Eng / CTO, 20–100 инженеров | $20–100K/год | Рост latency при масштабировании |
| **Edge / IoT** | Embedded Eng / CTO, 10–50 инженеров | $5–20K/год | Рост объёма данных |
| **CDN / edge compute** | Infra Eng / CTO, 50–200 инженеров | $20–100K/год | Проблемы с деплоем |
| **Fintech / audit (Compliance)** | CTO / Head of Eng, 50–300 инженеров | $100–500K/год | Аудит / регуляторное требование |

### 1.5. Рыночный контекст

| Рынок | Размер | CAGR |
|---|---|---|
| Key-value stores | $436–486M (2023) → $790–797M (2029–2030) | 8.6–8.9% |
| Embedded database systems | $11.7B (2025) → $23.55B (2034) | ~8% |
| **Целевой сегмент (latency-critical embedded KV)** | **$50–150M** | **15–20%** |

### 1.6. Масштаб (честно)

- Не $15B valuation. Не server-mode SaaS.
- Да open-source + enterprise-support + acqui-hire.
- Реалистичный горизонт: работающий продукт с измеримыми преимуществами и понятным путём к revenue.

---

## 2. Objectives и Key Results (OKR)

### 2.1. Формат

Каждая версия имеет:

- **Objective** — качественная цель.
- **Key Results** — 3–7 измеримых результатов.
- **Anti-KR** — что НЕ должно произойти.

### 2.2. OKR по версиям

#### v0.1 — Deterministic Foundation

**Objective:** доказать, что детерминированная задержка и zero-alloc достижимы в Go на single-node, в tiered-терминах.

**Key Results:**
- KR1: **p999 GET < 500 нс (T2: L3-resident, 1M keys, single-node).**
- KR2: **p999 GET < 2 мкс (T3: RAM-resident, 10M keys, single-node).**
- KR3: **Tier distribution ≥ 80% операций в L1/L2/L3 для 1M keys.**
- KR4: Allocs/op == 0 на Put/Get/Delete/Scan/WriteBatch.
- KR5: Escape analysis чист.
- KR6: CI-гейт блокирует merge при регрессии.
- KR7: Threat model v0.1 и Failure model v0.1 опубликованы.
- KR8: BENCH-016 (Tiered Latency) опубликован.
- KR9: 200+ stars.

**Anti-KR:**
- Нет деградации T2 p999 > 1 мкс под нагрузкой 1M ops/s.
- Нет аллокаций в hot path даже при burst 10×.
- Нет незадокументированных failure modes.
- **T3 p999 не > 5 мкс.**

#### v0.2 — Read-Amplification-Minimized Compaction

**Objective:** доказать, что read amp p99 < 3 достижим на реальных объёмах при сохранении tiered p999.

**Key Results:**
- KR1: Read amp p99 < 3 (disk reads per GET) на 10M ключей.
- KR2: WA < 5 (средняя).
- KR3: < 100 флашей на 1M ops.
- KR4: **Tier-aware compaction** — T2 p999 не деградирует > 10%.
- KR5: BENCH-005, BENCH-008 опубликованы.
- KR6: 500+ stars.

**Anti-KR:**
- Нет осцилляции компакции.
- Нет деградации T2 p999 > 1 мкс.
- Нет вытеснения hot keys из L3 при компакции.

#### v0.3 — Value Log

**Objective:** снизить write amplification для больших values без потери tiered p999.

**Key Results:**
- KR1: WA < 3 (средняя).
- KR2: Space amp < 1.3 (после GC, steady-state).
- KR3: Read amp p99 < 2.
- KR4: GC-snapshot совместимость (Titan-style WriteCallback, D87).
- KR5: BENCH-006 опубликован.
- KR6: 1000+ stars.

**Anti-KR:**
- Нет write stall под нагрузкой.
- Нет деградации read amp p99 > 2.
- Нет GC-snapshot несовместимости.
- **T2 p999 не деградирует > 10% для append-heavy workload.**

#### v0.4 — MVCC + Shard-per-core

**Objective:** доказать, что транзакции и шардинг работают без потери детерминизма.

**Key Results:**
- KR1: 50M GET/s на 64-core (in-memory, single-shard).
- KR2: Snapshot Isolation корректна (property-тесты).
- KR3: Single-shard транзакции без потери latency.
- KR4: Profile Compliance activation.
- KR5: 2000+ stars.

**Anti-KR:**
- Нет деградации T2 p999 > 500 нс.
- Нет утечек версий (GC работает).

#### v0.5 — TTL, PITR, Hardening

**Objective:** продакшн-готовность, первый платящий клиент.

**Key Results:**
- KR1: Первый платящий пользователь.
- KR2: PITR работает на 100 ГБ данных.
- KR3: Prometheus exporter + Grafana dashboard.
- KR4: Capacity model (ADR-035 v2.3) опубликован.
- KR5: Audit log.
- KR6: 3000+ stars.

**Anti-KR:**
- Нет регрессии SLO.
- Нет незадокументированных edge cases.

#### v0.6a — Distributed KV Core

**Objective:** доказать, что distributed KV на Dragonboat Multi-Raft поверх встроенного TCP работает.

**Key Results:**
- KR1: Freshness p99 < 1 с.
- KR2: **Dragonboat Multi-Raft throughput ≥ 1M ops/s на узел.**
- KR3: **`AllocsPerDistributedWrite` ≤ 10.**
- KR4: **Tan engine как Raft log storage.**
- KR5: **Кастомный LogDB для per-operation durability.**
- KR6: **Встроенный TCP Dragonboat для Raft.**
- KR7: 3–5 нод в проде, BENCH-011..014 пройдены.
- KR8: $100K ARR.
- KR9: 5000+ stars.

**Anti-KR:**
- Нет потери подтверждённых записей при partition (SYNC_MAJORITY).
- Нет clock skew-индуцированных split-brain.
- **Нет потери оптимизаций Dragonboat из-за кастомного transport.**
- **Нет регрессии single-node T2 p999 > 500 нс при distributed.**

#### v0.6b — SQL + Columnar Replica

**Objective:** добавить SQL subset и columnar replica поверх distributed KV core.

**Key Results:**
- KR1: SQL subset + PostgreSQL wire.
- KR2: Columnar replica (Raft Learner).
- KR3: TPC-C пройден.
- KR4: Resource isolation (cgroups).
- KR5: Encryption at rest (модуль).

**Anti-KR:**
- Нет деградации OLTP при OLAP > 20%.
- Нет двух executor'ов.

#### v0.7 — Columnar

**Objective:** добавить columnar storage для аналитических запросов.

**Key Results:**
- KR1: Compression ratio ≥ 5×.
- KR2: Column pruning ≥ 90% savings.
- KR3: SIMD speedup ≥ 4×.
- KR4: BENCH-015 (Lease Read vs ReadIndex) опубликован.

**Anti-KR:**
- Нет деградации OLTP.
- Нет двух executor'ов.

#### v1.0 — Enterprise

**Objective:** enterprise-готовность, SOC2, SLA, multi-region.

**Key Results:**
- KR1: SOC 2 Type II.
- KR2: SLA 99.99%.
- KR3: $1M ARR.
- KR4: 10000+ stars.
- KR5: Team 5+.
- KR6: Joint consensus.

**Anti-KR:**
- Нет снижения tiered p999.
- Нет незакрытых S1/S2 инцидентов.

### 2.3. Сводная таблица OKR

| Версия | Objective | Главный KR |
|---|---|---|
| v0.1 | Deterministic Foundation | **p999 T2 < 500 нс** |
| v0.2 | Read-amp Compaction | Read amp p99 < 3 |
| v0.3 | Value Log | WA < 3, read amp p99 < 2 |
| v0.4 | MVCC + Shard | 50M GET/s |
| v0.5 | Production | Первый платящий |
| **v0.6a** | **Distributed KV Core** | **Freshness < 1 с, 1M ops/s** |
| **v0.6b** | **SQL + Columnar** | **TPC-C** |
| v0.7 | Columnar | SIMD ≥ 4× |
| v1.0 | Enterprise | SOC2, $1M ARR |

---

## 3. Non-goals (что НЕ делаем)

### 3.1. По версиям

| Не-цель | До версии |
|---|---|
| OLAP / аналитика (кроме columnar replica) | v0.6b |
| Более 100 млн ключей | v0.3 |
| Cross-key транзакции | v0.4 |
| Enterprise SLA 99.99% | v1.0 |
| Windows / macOS production | v0.3 |
| SQL / PostgreSQL wire | v0.6b |
| Векторный поиск | v0.7+ |
| Joint consensus | v1.0 |
| Multi-region | v1.0 |
| Server mode под zero-alloc контрактом | никогда |
| $15B valuation | никогда |
| Собственный QUIC для intra-cluster | v1.0+ (cross-region only) |
| Encryption at rest | v0.6b (модуль) |
| RBAC | v1.0 |
| Audit log | v0.5 (модуль) |
| **`etcd/raft` для v0.6a** | **никогда (Dragonboat)** |
| **Свой Multi-Raft core** | **никогда (Dragonboat)** |
| **Свой gRPC transport для Raft** | **никогда (встроенный TCP Dragonboat)** |
| **p999 без tier** | **никогда** |

### 3.2. Постоянные non-goals

- Не заменяем PostgreSQL. Разные ниши.
- Не distributed для всего. Только где нужно.
- Не жертвуем tiered p999 ради средней скорости.
- Не используем `sync.Pool` в data plane, `fmt`, `reflect`, `interface{}` на hot path.
- Не добавляем зависимости без ADR.
- Не публикуем утверждения из запрещённого списка (HLD v11.2 §12.B).
- Не обещаем zero-alloc на Raft path (Dragonboat аллоцирует, честно).
- **Не публикуем p999 без указания tier.**

### 3.3. Cancelled

| ID | Что | Почему |
|---|---|---|
| CAN-001 | Cross-shard без 2PC | Теоретически невозможно (ADR-018) |
| CAN-002 | TPC-C в v0.4 | Требует cross-shard (перенесено в v0.6b) |
| CAN-003 | sync.Pool на hot path | Нарушает zero-alloc (ADR-009 v2) |
| CAN-004 | $15B valuation | Не цель проекта (HLD v11.2 §1) |
| CAN-005 | etcd/raft для v0.6a | Отменено. Dragonboat (D78) |
| CAN-006 | Собственный QUIC для intra-cluster | 66% throughput, 12× RAM (ADR-005 v5) |
| CAN-007 | Jepsen (Clojure) | Knockbox (Go) достаточно (ADR-016) |
| CAN-008 | TLA+ refinement mapping | Модель + property-тесты достаточно (ADR-015 v2) |
| CAN-009 | gRPC Codec V1 | Аллоцирует на каждый message. CodecV2 + SharedBufferPool (D86) |
| CAN-010 | TephraDB (название) | Переименовано в TephraKV |
| **CAN-011** | **Свой Raft для v0.6a** | **Отменено. Dragonboat (D78, D79)** |
| **CAN-012** | **Свой gRPC transport для Raft** | **Отменено. Встроенный TCP Dragonboat (D106)** |
| **CAN-013** | **p999 < 5 мс (in-memory)** | **Отменено. Tiered-цели (D107)** |

### 3.4. Deferred

| ID | Что | Пересмотр |
|---|---|---|
| DEF-001 | Векторный поиск | 2027-Q2 |
| DEF-002 | Joint consensus | v1.0 |
| DEF-003 | Multi-region | v1.0 |
| DEF-004 | Server mode zero-alloc | никогда |
| DEF-005 | Shared Bloom (per-key) | v0.5 |
| DEF-006 | Calvin-style deterministic txn | после v1.0 |
| DEF-007 | QUIC для cross-region | v1.0+ |
| DEF-008 | Свой Raft | v1.0+ (если команда 5+) |

---

## 4. Timeline

### 4.1. Два плана

- **Base plan** — без buffer, оптимистичный.
- **Risk-adjusted plan** — с buffer на риски.

Правило: для внешних обязательств — только risk-adjusted.

### 4.2. Base plan

| Версия | Фокус | Срок | Недели | SP | Ключевая метрика |
|---|---|---|---|---|---|
| v0.1 | Deterministic Foundation | 6 нед | W0–W6 | 98 | **p999 T2 < 500 нс** |
| v0.2 | Read-amp Compaction | +6 нед | W7–W12 | 47 | Read amp p99 < 3 |
| v0.3 | Value Log | +6 нед | W13–W18 | 25 | WA < 3, read amp p99 < 2 |
| v0.4 | MVCC + Shard | +8 нед | W19–W26 | 40 | 50M GET/s |
| v0.5 | TTL + PITR | +8 нед | W27–W34 | 26 | Первый платящий |
| **v0.6a** | **Distributed KV Core** | **+12 нед** | **W35–W46** | **78** | **Freshness < 1 с** |
| **v0.6b** | **SQL + Columnar** | **+6 нед** | **W47–W52** | **28** | **TPC-C** |
| v0.7 | Columnar | +8 нед | W53–W60 | 30 | SIMD ≥ 4× |
| v1.0 | Enterprise | +24 нед | W61–W84 | 76 | SOC2, $1M ARR |

**Итого base:** 84 недели (~19.5 месяцев). **SP:** 448.

### 4.3. Risk-adjusted plan

| Версия | Base | Buffer | Итого | Недели |
|---|---|---|---|---|
| v0.1 | 6 | +2 | 8 | W0–W8 |
| v0.2 | 6 | +1 | 7 | W9–W15 |
| v0.3 | 6 | +1 | 7 | W16–W22 |
| v0.4 | 8 | +1.5 | 9.5 | W23–W32 |
| v0.5 | 8 | +1.5 | 9.5 | W33–W42 |
| **v0.6a** | **12** | **+2** | **14** | **W43–W56** |
| **v0.6b** | **6** | **+1** | **7** | **W57–W63** |
| v0.7 | 8 | +1 | 9 | W64–W72 |
| v1.0 | 24 | +4 | 28 | W73–W100 |

**Итого risk-adjusted:** 100 недель (~23 месяца). **SP:** 448.

### 4.4. Вехи (milestones)

| Веха | Base | Risk-adj | Что значит |
|---|---|---|---|
| M1: Первый zero-alloc GET | W1 | W1 | Гипотеза работает |
| M2: Первый T2 p999 < 500 нс | W2 | W3 | **Tiered-цель подтверждена** |
| M3: Первый WAL fsync | W3 | W4 | Durability работает |
| M4: Первый recovery после kill -9 | W4 | W5 | Надёжность работает |
| M5: v0.1 release | W6 | W8 | Публичный релиз |
| M6: v0.2 release | W12 | W15 | Read amp подтверждён |
| M7: v0.3 release | W18 | W22 | WA снижен |
| M8: v0.4 release | W26 | W32 | Транзакции работают |
| M9: Первый платящий | W34 | W42 | Revenue начался |
| M10: v0.6a release | W46 | W56 | **Distributed работает (Dragonboat)** |
| M11: v0.6b release | W52 | W63 | SQL + columnar |
| M12: v1.0 release | W84 | W100 | Enterprise-готовность |

### 4.5. Релизный каденс

- Минорные (v0.X): 6–8 недель (base), 7–10 недель (risk-adjusted).
- v0.6 разбит: v0.6a (12/14 нед) + v0.6b (6/7 нед).
- Патчи (v0.X.Y): по необходимости, в течение 48 часов.
- Major (v1.0): после закрытия всех exit criteria.

---

## 5. Детализация версий

### 5.1. v0.1 — Deterministic Foundation

**Theme:** доказать, что zero-alloc и tiered p999 достижимы.
**Горизонт:** Now. **Точность:** высокая.
**Срок:** 6 недель (base) / 8 недель (risk-adjusted). **SP:** 98. **Задач:** 35.

**Состав (из BACKLOG v6.4):**

| Эпик | Задач | SP | Критпуть |
|---|---|---|---|
| E01: Bootstrap | 5 | 9 | yes |
| E02: Arena | 3 | 7 | yes |
| E03: MemTable | 5 | 18 | yes |
| E04: WAL | 5 | 16 | yes |
| E05: Durability | 3 | 11 | yes |
| E06: SSTable | 5 | 17 | no |
| E07: Metrics | 3 | 12 | no |
| E08: CI | 3 | 7 | yes |
| E09: Benchmarks (incl. BENCH-016) | 3 | 8 | no |
| **Итого** | **35** | **98** | |

**Критический путь:**

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
                                                            BENCH-016 → DOC-001
```

**Exit criteria:**
- [ ] **p999 GET < 500 нс (T2: L3-resident, 1M keys, single-node).**
- [ ] **p999 GET < 2 мкс (T3: RAM-resident, 10M keys, single-node).**
- [ ] **Tier distribution ≥ 80% операций в L1/L2/L3 для 1M keys.**
- [ ] Allocs/op == 0 на hot path.
- [ ] Cycles/op GET: p50 < 300, p99 < 900, p999 < 1500 (T2).
- [ ] Read amp p99 < 3 (эмуляция 2 уровней).
- [ ] CI-гейт работает.
- [ ] BENCH-001, 002, 003, 007, **016** опубликованы.
- [ ] Threat model v0.1 и Failure model v0.1 опубликованы.
- [ ] 200+ stars.

**Ключевые ADR:** 001, 002, 009 v2, 010 v3, 012 v6, 019, 020, **036 v1**.

**Capacity:** 1 инженер (solo).

---

### 5.2. v0.2 — Read-Amplification-Minimized Compaction

**Theme:** доказать, что read amp p99 < 3 достижим при сохранении tiered p999.
**Горизонт:** Now. **Точность:** высокая.
**Срок:** +6 недель (base) / +7 недель (risk-adjusted). **SP:** 47.

**Состав:**

| Эпик | Задач | SP |
|---|---|---|
| E01: Partition index | 3 | 12 |
| E02: MinMax index | 2 | 5 |
| E03: Leveled compaction | 3 | 10 |
| E04: Read-amp trigger | 3 | 7 |
| E05: Auto-flush | 2 | 3 |
| E06: Benchmarks | 3 | 10 |
| **Итого** | **16** | **47** |

**Exit criteria:**
- [ ] Read amp p99 < 3 на 10M ключей (YCSB C).
- [ ] WA < 5 (средняя).
- [ ] < 100 флашей на 1M ops.
- [ ] Feedback loop стабилен (нет осцилляции).
- [ ] **Tier-aware compaction** — T2 p999 не деградирует > 10%.
- [ ] 500+ stars.

**Ключевые ADR:** 003, 021.

---

### 5.3. v0.3 — Value Log

**Theme:** снизить WA для больших values без потери tiered p999.
**Горизонт:** Next. **Точность:** средняя.
**Срок:** +6 недель (base) / +7 недель (risk-adjusted). **SP:** 25.

**Состав:**

| Эпик | Задач | SP |
|---|---|---|
| E01: VLog writer/reader | 3 | 10 |
| E02: VLog GC | 3 | 10 |
| E03: Benchmarks | 2 | 5 |
| E04: Windows/macOS prod-ready | 3 | TBD |
| **Итого** | **11** | **25** |

**Exit criteria:**
- [ ] WA < 3 (средняя).
- [ ] Space amp < 1.3 (после GC).
- [ ] Read amp p99 < 2.
- [ ] Write stall отсутствует.
- [ ] GC-snapshot совместимость (Titan-style WriteCallback, D87).
- [ ] **T2 p999 не деградирует > 10% для append-heavy workload.**
- [ ] 1000+ stars.

**Ключевые ADR:** 022, 023.

**Ограничения (явно):** update-heavy и range-heavy workload'ы требуют отдельной оценки (BENCH-006).

---

### 5.4. v0.4 — MVCC + Shard-per-core

**Theme:** транзакции и шардинг без потери детерминизма.
**Горизонт:** Next. **Точность:** средняя.
**Срок:** +8 недель (base) / +9.5 недель (risk-adjusted). **SP:** 40.

**Состав:**

| Эпик | Задач | SP |
|---|---|---|
| E01: MVCC | 4 | 13 |
| E02: Транзакции (single-shard) | 4 | 15 |
| E03: Shard-per-core | 3 | 10 |
| E04: Resource groups (концепт) | 1 | 2 |
| **Итого** | **13** | **40** |

**Exit criteria:**
- [ ] 50M GET/s на 64-core (in-memory, single-shard).
- [ ] Snapshot Isolation корректна (property-тесты).
- [ ] Single-shard транзакции без потери latency.
- [ ] **T2 p999 не деградирует > 500 нс.**
- [ ] Profile Compliance activation.
- [ ] 2000+ stars.

**Убрано из v0.4:**
- TPC-C (перенесено в v0.6b, CAN-002).
- Cross-shard без 2PC (CAN-001, ADR-018).

**Примечание:** shard-per-core (v0.4) — execution unit. Отображение 1 shard = 1 Raft group появляется в v0.6a (ADR-026 v5).

**Ключевые ADR:** 017, 018.

---

### 5.5. v0.5 — TTL, PITR, Hardening

**Theme:** продакшн-готовность, первый платящий клиент.
**Горизонт:** Next. **Точность:** средняя.
**Срок:** +8 недель (base) / +9.5 недель (risk-adjusted). **SP:** 26.

**Состав:**

| Эпик | Задач | SP |
|---|---|---|
| E01: TTL | 3 | 7 |
| E02: PITR | 3 | 10 |
| E03: Prometheus exporter | 2 | 5 |
| E04: Rate limiting | 1 | 2 |
| E05: Capacity model | 1 | 2 |
| **Итого** | **10** | **26** |

**Exit criteria:**
- [ ] Первый платящий пользователь.
- [ ] PITR работает на 100 ГБ данных.
- [ ] Prometheus exporter + Grafana dashboard.
- [ ] Capacity model (ADR-035 v2.3) опубликован.
- [ ] Audit log.
- [ ] 3000+ stars.

**Ключевые ADR:** 035 v2.3.

---

### 5.6. v0.6a — Distributed KV Core

**Theme:** distributed KV на Dragonboat Multi-Raft поверх встроенного TCP.
**Горизонт:** Later. **Точность:** концептуальная.
**Срок:** +12 недель (base) / +14 недель (risk-adjusted). **SP:** 78.

**Состав:**

| Эпик | SP |
|---|---|
| E01: Dragonboat Multi-Raft + Tan engine | 21 |
| E02: Custom LogDB (per-op durability) | 12 |
| E03: Snapshot + membership + learner | 11 |
| E04: 2PC | 8 |
| E05: Server mode (gRPC + REST) + CodecV2 | 10 |
| E06: Benchmarks (BENCH-011..014) | 10 |
| E07: Cluster 3–5 нод + integration | 6 |
| **Итого** | **78** |

**Ключевые решения (из HLD v11.2):**

| Компонент | Решение | Обоснование |
|---|---|---|
| Raft core | **Dragonboat** | 1.25M writes/s, Multi-Raft из коробки, Jepsen (D78) |
| Raft log storage | **Tan engine** | Log-file based LogDB, без space amplification (D101) |
| Per-op durability | **Кастомный LogDB** (`ILogDB`) | Документированный интерфейс Dragonboat (D102) |
| Raft transport | **Встроенный TCP Dragonboat** | Ближе к raw performance. Gitaly использует (D103) |
| Server mode | gRPC + REST | Внешний API, не Raft path |
| Codec | **CodecV2 + SharedBufferPool** | V1 bridge аллоцирует (D86) |
| Read path | ReadIndex (default) + Lease Read (опц.) | TiKV-прецедент (D64) |
| Lease Read clock | Monotonic raw clock (Instant) | Wall-clock drift breaks linearizability (D88) |
| Zero-alloc | Single-node hot path + LSM apply | Dragonboat — control plane, аллокации допустимы (D80) |
| `AllocsPerDistributedWrite` | ≤ 10 | Честная метрика (D104) |
| Wire format | Свой binary, кастомный codec | Zero-alloc на payload (D39, D70) |

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
- [ ] $100K ARR.
- [ ] 5000+ stars.

**Anti-KR:**
- Нет потери подтверждённых записей при partition (SYNC_MAJORITY).
- **Нет потери оптимизаций Dragonboat из-за кастомного transport.**
- **Нет регрессии single-node T2 p999 > 500 нс при distributed.**

**Capacity:** 2 инженера (найм 1 distributed-инженера до старта).

**Ключевые ADR:** 005 v5, 006 v5, 009 v2, 010 v3, 011, 012 v6, 013, 014, 015 v2, 016, 018, 026 v5, 027, 036 v1.

---

### 5.7. v0.6b — SQL + Columnar Replica

**Theme:** SQL subset + columnar replica поверх distributed KV core.
**Горизонт:** Later. **Точность:** концептуальная.
**Срок:** +6 недель (base) / +7 недель (risk-adjusted). **SP:** 28.

**Состав:**

| Эпик | SP |
|---|---|
| E01: SQL subset + PostgreSQL wire | 12 |
| E02: Columnar replica (Raft Learner) | 8 |
| E03: TPC-C | 4 |
| E04: Resource isolation (cgroups) | 4 |
| **Итого** | **28** |

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

**Ключевые ADR:** 024, 025, 027.

---

### 5.8. v0.7 — Columnar

**Theme:** columnar storage + SIMD executor.
**Горизонт:** Later. **Точность:** концептуальная.
**Срок:** +8 недель (base) / +9 недель (risk-adjusted). **SP:** 30.

**Состав:**

| Эпик | Задач | SP |
|---|---|---|
| E01: Columnar format | 5 | 15 |
| E02: SIMD AVX-512 + fallback | 2 | 10 |
| E03: Late materialization | 1 | 5 |
| **Итого** | **8** | **30** |

**Exit criteria:**
- [ ] Compression ratio ≥ 5×.
- [ ] Column pruning ≥ 90% savings.
- [ ] SIMD speedup ≥ 4× (AVX-512).
- [ ] Fallback на AVX2/NEON работает.
- [ ] BENCH-015 опубликован.

**Опционально:** миграция встроенный TCP → DRPC (ADR-005 v5), если bottleneck.

**Capacity:** 2 инженера.

**Ключевые ADR:** 024, 025.

---

### 5.9. v1.0 — Enterprise

**Theme:** enterprise-готовность, SOC2, SLA, multi-region.
**Горизонт:** Later. **Точность:** концептуальная.
**Срок:** +24 недели (base) / +28 недель (risk-adjusted). **SP:** 76.

**Состав:**

| Эпик | Задач | SP |
|---|---|---|
| E01: Security | 4 | 21 |
| E02: Distribution (multi-region + SSI) | 2 | 26 |
| E03: Compliance (SOC2, SLA) | 2 | 29 |
| **Итого** | **8** | **76** |

**Exit criteria:**
- [ ] SOC 2 Type II.
- [ ] SLA 99.99%.
- [ ] Multi-region.
- [ ] SSI.
- [ ] Joint consensus.
- [ ] $1M ARR.
- [ ] 10000+ stars.
- [ ] Team 5+.

**Опционально:** миграция DRPC → свой TCP (ADR-005 v5), если DRPC bottleneck.

**Capacity:** 5+ инженеров.

**Ключевые ADR:** 004, 028, 029 v2, 030, 031, 032, 033.

---

## 6. Критический путь и зависимости

### 6.1. Межверсионные зависимости

```
v0.1 (Foundation) ──► v0.2 (Compaction) ──► v0.3 (VLog)
                          │
                          ▼
                       v0.4 (MVCC) ──► v0.5 (Production)
                          │
                          ▼
                       v0.6a (Distributed KV) ──► v0.6b (SQL + Columnar)
                          │
                          ▼
                       v0.7 (Columnar) ──► v1.0 (Enterprise)
```

Правило: версия не стартует, пока предыдущая не закрыта.

### 6.2. Критический путь v0.1

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
                                                            BENCH-016 → DOC-001
```

### 6.3. Ключевые ADR по версиям

| Версия | ADR |
|---|---|
| v0.1 | 001, 002, 009 v2, 010 v3, 012 v6, 019, 020, **036 v1** |
| v0.2 | 003, 021 |
| v0.3 | 022, 023 |
| v0.4 | 017, 018 |
| v0.5 | 035 v2.3 |
| **v0.6a** | **005 v5, 006 v5, 009 v2, 010 v3, 011, 012 v6, 013, 014, 015 v2, 016, 018, 026 v5, 027, 036 v1** |
| **v0.6b** | **024, 025, 027** |
| v0.7 | 024, 025 |
| v1.0 | 004, 028, 029 v2, 030, 031, 032, 033 |

---

## 7. Risk-adjusted roadmap

### 7.1. Основные риски и их влияние на сроки

| Риск | Версия | Base | Buffer | Risk-adj |
|---|---|---|---|---|
| **Недостижение T2 (L3-resident, p999 < 500 нс)** | **v0.1** | **6 нед** | **+2** | **8 нед** |
| **Недостижение T3 (RAM-resident, p999 < 2 мкс)** | **v0.1** | **—** | **включено** | **—** |
| **Деградация T2 при cache miss** | **v0.1–v0.2** | **—** | **включено** | **—** |
| Lock-free баги в skiplist | v0.1 | — | включено | — |
| Замаскированные аллокации | v0.1 | — | включено | — |
| Read amp > 3 под нагрузкой | v0.2 | 6 нед | +1 | 7 нед |
| VLog write stall | v0.3 | 6 нед | +1 | 7 нед |
| WiscKey GC / snapshot consistency | v0.3 | — | включено | — |
| MVCC GC не успевает | v0.4 | 8 нед | +1.5 | 9.5 нед |
| Нет revenue до v0.5 | v0.5 | 8 нед | +1.5 | 9.5 нед |
| **Dragonboat: интеграция с Tan engine** | **v0.6a** | **12 нед** | **+2** | **14 нед** |
| **Dragonboat: кастомный LogDB** | **v0.6a** | **—** | **включено** | **—** |
| **Dragonboat: ILogDB нестабильность** | **v0.6a** | **—** | **включено** | **—** |
| **Dragonboat: throughput на 10000 groups** | **v0.6a** | **—** | **включено** | **—** |
| **Dragonboat: heartbeat batching при 10000 groups** | **v0.6a** | **—** | **включено** | **—** |
| **Dragonboat: встроенный TCP throughput** | **v0.6a** | **—** | **включено** | **—** |
| **`AllocsPerDistributedWrite` > 10** | **v0.6a** | **—** | **включено** | **—** |
| Snapshot transfer больших данных | v0.6a | — | включено | — |
| TLS AEAD overhead на p999 | v0.6a | — | включено | — |
| Lease Read clock drift | v0.6a | — | включено | — |
| Найм distributed-инженера | v0.6a | — | включено | — |
| AVX-512 не на всех CPU | v0.7 | 8 нед | +1 | 9 нед |
| SOC2 долгий | v1.0 | 24 нед | +4 | 28 нед |

### 7.2. План B (если задержки критические)

Если v0.6a затягивается > 16 недель:

- **Plan B1:** урезать SQL subset до минимума (только CRUD), отложить columnar replica на v0.7.
- **Plan B2:** урезать Multi-Raft до 100 шардов (не 10000), отложить rebalance.
- **Plan B3:** убрать server mode (gRPC/REST), оставить только embedded.
- **Plan B4:** fallback на RocksDB LogDB в Dragonboat, если Tan engine нестабилен. Per-op durability теряется.

Правило: режем scope, не сдвигаем дату.

### 7.3. Что делать, если риск реализовался

1. Остановить работу над версией.
2. Открыть incident (если S1/S2).
3. Постмортем в `docs/incidents/`.
4. ADR с решением.
5. Пересмотр ROADMAP.
6. Обновление BACKLOG.

---

## 8. Capacity и бюджет

### 8.1. Инженерный capacity

| Период | Инженеров | Base SP/нед | Risk-adj SP/нед | Комментарий |
|---|---|---|---|---|
| v0.1–v0.3 | 1 | ~17 | ~14 | Solo |
| v0.4–v0.5 | 1 | ~15 | ~12 | Solo |
| v0.6a | 2 | ~25 | ~20 | Найм 1 distributed-инженера |
| v0.6b | 2 | ~25 | ~20 | |
| v0.7 | 2 | ~25 | ~20 | |
| v1.0 | 5+ | ~50 | ~40 | Найм 3+ |

### 8.2. Временной бюджет

| Активность | % времени |
|---|---|
| Разработка кода | 50% |
| Тесты, бенчмарки | 15% |
| Документация (ADR, HLD, BENCH) | 15% |
| Review, отчёты | 10% |
| Техдолг, рефакторинг | 10% |

Правило: если документация > 20% — процесс сломан.

### 8.3. Финансовый бюджет (минимальный)

| Статья | v0.1–v0.5 | v0.6a/b | v1.0 |
|---|---|---|---|
| Инфраструктура | $200/мес | $500/мес | $2000/мес |
| CI | $50/мес | $100/мес | $300/мес |
| Домен, SSL | $20/год | $20/год | $20/год |
| Юридические | $0 | $500 | $5000 |
| SOC2 audit | — | — | $50000 |
| **Итого/год** | **~$3000** | **~$8000** | **~$80000** |

---

## 9. Success metrics

### 9.1. Технические метрики

| Метрика | v0.1 | v0.3 | v0.6a | v1.0 |
|---|---|---|---|---|
| **p999 GET (T1: L1/L2, 100K keys)** | **< 200 нс** | < 200 нс | < 200 нс | < 200 нс |
| **p999 GET (T2: L3, 1M keys)** | **< 500 нс** | < 500 нс | < 500 нс | < 500 нс |
| **p999 GET (T3: RAM, 10M keys)** | **< 2 мкс** | < 2 мкс | < 2 мкс | < 2 мкс |
| **p999 GET (T4: NVMe, 100M keys)** | **< 50 мкс** | < 50 мкс | < 50 мкс | < 50 мкс |
| p99 GET (distributed) | — | — | < 10 мс | < 10 мс |
| Read amp p99 | < 3 | < 2 | < 2 | < 2 |
| WA (средняя) | < 5 | < 3 | < 3 | < 3 |
| Allocs/op hot path | 0 | 0 | 0 | 0 |
| Allocs/op transport codec | — | — | ≤ 2 | ≤ 1 |
| **`AllocsPerDistributedWrite`** | — | — | **≤ 10** | ≤ 5 |
| Cycles/op GET (p50, T2) | < 300 | < 300 | < 300 | < 300 |
| QPS на узел | 1M | 10M | 50M | 100M |
| Freshness p99 | — | — | < 1 с | < 1 с |
| Degradation | — | — | < 20% | < 20% |
| **Tier distribution (1M keys)** | **≥ 80% L1/L2/L3** | ≥ 80% | ≥ 80% | ≥ 80% |

### 9.2. Продуктовые метрики

| Метрика | v0.1 | v0.3 | v0.5 | v0.6a | v1.0 |
|---|---|---|---|---|---|
| GitHub stars | 200+ | 1000+ | 3000+ | 5000+ | 10000+ |
| Платящих | 0 | 0 | 1+ | 5+ | 20+ |
| ARR | $0 | $0 | $10K | $100K | $1M+ |
| Design partners | 0 | 0 | 3+ | 5+ | 10+ |
| Contributors | 0 | 2+ | 5+ | 10+ | 20+ |

### 9.3. Процессные метрики (DORA-like)

| Метрика | Цель |
|---|---|
| Deployment frequency | 6–8 нед |
| Lead time | < 5 дней |
| Change failure rate | < 10% |
| MTTR (S1/S2) | < 24 ч |
| Cycle time | < 3 дня |
| WIP | ≤ 3 |
| Throughput | 12–15 SP/нед (solo) |

---

## 10. Exit criteria и гейты

### 10.1. Гейты между версиями

| Переход | Обязательные условия |
|---|---|
| v0.1 → v0.2 | Zero-alloc CI зелёный; BENCH-001..003, 007, **016**; **T2 p999 < 500 нс**; threat model + failure model |
| v0.2 → v0.3 | Read amp p99 < 3 (BENCH-008); WA < 5; tier-aware compaction работает |
| v0.3 → v0.4 | WA < 3; space amp < 1.3; read amp p99 < 2; GC-snapshot совместимость |
| v0.4 → v0.5 | 50M GET/s на 64-core; property-тесты |
| v0.5 → v0.6a | Первый платящий; PITR на 100 ГБ; capacity model |
| v0.6a → v0.6b | Freshness p99 < 1 с; BENCH-011..014; 3–5 нод в проде; **Dragonboat ≥ 1M ops/s**; **`AllocsPerDistributedWrite` ≤ 10**; **Tan engine**; **кастомный LogDB** |
| v0.6b → v0.7 | TPC-C пройден; columnar replica работает |
| v0.7 → v1.0 | SIMD ≥ 4×; compression ≥ 5×; columnar формат стабилен |

### 10.2. Публикационные гейты (каждый релиз)

- [ ] Внешние метрики опубликованы (p50/p99/p999 по tier).
- [ ] Внутренние метрики опубликованы.
- [ ] Tier distribution опубликован.
- [ ] BENCH-файлы в `docs/bench/`.
- [ ] Release notes в `docs/releases/`.
- [ ] Decision Log обновлён.
- [ ] CHANGELOG обновлён.
- [ ] Миграционный гайд (если API менялся).
- [ ] Анонс (Twitter, Reddit, HN) — с честными цифрами.

---

## 11. Cross-cutting треки

Параллельно всем версиям.

| Трек | Ответственность | Артефакты | Частота |
|---|---|---|---|
| Документация | HLD, ENG, ADR, GLOSSARY | `docs/` | При каждом ADR |
| Бенчмарки | BENCH-001..016 | `docs/bench/` | Каждый релиз |
| CI/CD | Zero-alloc, race, escape, bench regression | `.github/workflows/` | Каждый PR |
| Безопасность | Threat model, failure model | `docs/security/` | Обновляется |
| Сообщество | GitHub, Twitter, блог | Внешние | Еженедельно |
| Design partners | 3–5 партнёров до v0.5 | Обратная связь | Ежемесячно |
| Open-core | Границы OSS/enterprise | ADR-004 | До v0.4 |
| Найм | 1 инженер до v0.6a | План в v0.5 | v0.5–v0.6a |
| SLO/error budget | Мониторинг | `docs/SLO.md` | Постоянно |
| Capacity model | RAM/CPU/Disk | ADR-035 v2.3 | Обновляется |
| **Dragonboat upstream** | **Мониторинг версий, PR, форк** | **ADR-039 (planned)** | **Постоянно** |

---

## 12. Re-planning

### 12.1. Обязательный пересмотр

- Каждый релиз.
- Каждый квартал.
- При критическом риске.
- **Если T2 p999 < 500 нс не достигнут на BENCH-016.**

### 12.2. Триггеры

См. §0.7.

### 12.3. Процесс пересмотра

1. Анализ: что сделано, что нет, почему.
2. Переоценка SP для следующих версий.
3. Обновление OKR.
4. Обновление non-goals.
5. Обновление timeline.
6. Обновление BACKLOG.
7. ADR (если решение значимое).
8. Запись в `docs/reports/`.

---

## 13. What's explicitly unknown (честно)

Из HLD v11.2 §9.4:

- **Достижимость T2 (L3-resident, p999 < 500 нс) на 1M keys — не знаем. BENCH-016.**
- **Достижимость T3 (RAM-resident, p999 < 2 мкс) на 10M keys — не знаем. BENCH-016.**
- **Tier distribution в реальной нагрузке — не знаем.**
- Worst-case burst 10× — измеряем, не знаем.
- Clock skew > timeout — тестируем, не знаем.
- Snapshot transfer больших данных — не знаем.
- Columnar apply 10M ops/s — не знаем.
- **Dragonboat throughput на 10000 groups — не знаем. BENCH-014.**
- **Dragonboat heartbeat batching эффективность при 10000 groups × N nodes — не знаем.**
- **Dragonboat quiescing: сколько групп активно в реальной нагрузке — не знаем.**
- **Dragonboat интеграция с Tan engine — не знаем.**
- **Dragonboat кастомный LogDB для per-op durability — не знаем.**
- **Встроенный TCP Dragonboat: throughput при 10000 groups — не знаем.**
- **`AllocsPerDistributedWrite` — не знаем. BENCH-014.**
- TLS AEAD overhead при 1M ops/s — не знаем.
- Lease Read при bounded drift в production — не знаем.
- CodecV2 миграция — не знаем.
- Path to revenue — не знаем.
- Moat против Google — не знаем.

Правило: неизвестное документируется как неизвестное.

---

## 14. Что дальше

1. Согласовать формат — с Екатериной.
2. Заполнить DoR для P0-задач v0.1.
3. Синхронизировать с BACKLOG v6.4 (448 SP).
4. Обновить ADR-034, ADR-035, ADR-036 ссылки на HLD v11.2.
5. Написать **ADR-039 «Dragonboat dependency risk»** (версия, пиннинг, политика апгрейда, план B).
6. Написать **ADR-040 «Migration v0.5 → v0.6a»** (WAL pool → Tan engine).
7. Написать **ADR-011 v3 «Distributed consistency model»** (Linearizable? маппинг `ReadOptions.Consistency`).
8. Обновить **ADR-035 до v2.3** (capacity model с учётом Dragonboat: RAM групп, TCP буферы, Tan overhead).
9. Начать v0.1 — CHORE-001.
10. Обновлять при каждом релизе.

Правило: ROADMAP — стратегия. BACKLOG — тактика. HLD — архитектура. ENG — правила. GLOSSARY — язык.

---

## 15. Что изменилось против v3.0

**Синхронизация с HLD v11.2: tiered p999, Dragonboat, v0.6a/b, SP 448.**

Содержательные правки:

- §0.1: tiered latency goals (T1/T2/T3/T4).
- §0.1: Dragonboat вместо etcd/raft.
- §0.1: встроенный TCP Dragonboat вместо gRPC для Raft.
- §0.1: v0.6 разбит на v0.6a (78 SP) + v0.6b (28 SP).
- §0.1: SP 448 (было 452). v0.1 98 (было 102). +3 SP BENCH-016.
- §1.1 Vision: Dragonboat Multi-Raft + встроенный TCP.
- §1.3 Позиционирование: tiered p999 < 500 нс (T2: L3-resident).
- §2.2 OKR: все KR переформулированы в tiered-терминах. v0.6a/b разбиты.
- §3.1 Non-goals: добавлены «etcd/raft для v0.6a — никогда», «Свой Multi-Raft core — никогда», «Свой gRPC transport для Raft — никогда», «p999 без tier — никогда».
- §3.3 CAN-005: etcd/raft → Dragonboat. CAN-011..013 новые.
- §3.4 DEF-008: свой Raft → v1.0+.
- §4.2 Base plan: v0.6a/b. Итого 84 недели.
- §4.3 Risk-adjusted: v0.6a/b. Итого 100 недель.
- §4.4 Вехи: M2 (первый T2 p999 < 500 нс), M10 (v0.6a), M11 (v0.6b).
- §5.1 v0.1: SP 98. Exit criteria — tiered p999 + tier distribution.
- §5.6 v0.6a: Dragonboat, Tan engine, кастомный LogDB, встроенный TCP. SP 78.
- §5.7 v0.6b: SQL + columnar replica. SP 28.
- §6.1 Зависимости: v0.6a → v0.6b.
- §6.3 Ключевые ADR: 005 v5, 006 v5, 012 v6, 026 v5, 036 v1.
- §7.1 Риски: недостижение T2/T3, интеграция с Tan engine, кастомный LogDB, ILogDB нестабильность, throughput Dragonboat.
- §9.1 Технические метрики: tiered p999, `AllocsPerDistributedWrite`, tier distribution.
- §10.1 Гейты: v0.6a → v0.6b требует Dragonboat ≥ 1M ops/s.
- §13 Unknowns: tier distribution, достижимость T2/T3, Dragonboat throughput, Tan engine.
- §14 Что дальше: ADR-039, ADR-040, ADR-011 v3, ADR-035 v2.3.

Структурные правки:
- §0.1 «Что изменилось» — новая секция.
- §15 «Что изменилось» — новая секция.

Ссылки:
- HLD-000 v7.0 → v11.2.
- PRD-001 v1.0 → v1.3.
- GTM-001 v1.0 → v2.3.
- PRICING-001 — добавлен.
- BACKLOG-001 v4.1 → v6.4.
- GLOSSARY-001 v1.1 → v3.4.
- ENG-001 v1.1 → v1.2.
- ADR-005 v4 → v5.
- ADR-006 v2 → v5.
- ADR-010 v2 → v3.
- ADR-012 v3 → v6.
- ADR-026 v3 → v5.
- ADR-036 v1 — добавлен.

---

**Конец TephraKV-ROADMAP-001 v4.0**
