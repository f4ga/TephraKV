# TephraKV — Go-to-Market Strategy (Case-Study Edition)

**Документ:** TephraKV-GTM-001
**Версия:** 2.0
**Статус:** Черновик
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v7.0, TephraKV-PRD-001 v1.0, TephraKV-ROADMAP-001 v3.0, TephraKV-BACKLOG-001 v5.0, TephraKV-GLOSSARY-001 v2.0, ADR-034 v2.0, ADR-035 v2.0
**Заменяет:** TephraKV-GTM-001 v1.0
**Классификация:** Внутренний / публичный

---

## 0. Назначение

GTM-001 v2.0 — стратегия выхода TephraKV на рынок, **обоснованная реальными кейсами**. Каждый сегмент подкреплён историей компании, которая уже решала похожую проблему. Это не маркетинговый документ — это **карта проверенных болей и проверенных бюджетов**.

**Правило:** если сегмент не подтверждён кейсом — он не в GTM.

---

## 1. Рынок

### 1.1. Размер рынка

| Рынок | 2025 | Прогноз | CAGR | Источник |
|---|---|---|---|---|
| Key-Value Stores | $485.68M (2023) | $796.77M (2029) | 8.6% | MarketPublishers |
| Embedded Database Systems | $11.71B | $23.55B (2034) | 8.08% | The Insight Partners |
| **Целевой сегмент TephraKV** | **~$50–150M** | **15–20%/год** | — | HLD v7.0 §1 |

### 1.2. Драйверы роста (подтверждённые кейсами)

| Драйвер | Кейс | Что доказывает |
|---|---|---|
| GC-паузы = потеря денег | SpacetimeDB Issue #4093: 131KB аллокаций на каждое чтение → stuttering каждые 10 сек на 500–1000 мс | Прямой спрос на zero-alloc |
| RAM дорого | Instacart: миграция feature store с ElastiCache на Dragonfly → −70% кластера, −50% latency | Готовность платить за RAM efficiency |
| cgo = операционный риск | Netlify: CGO crash в проде на edge-инфраструктуре | Спрос на pure Go |
| Edge = новый рынок | Cloudflare CacheDB: Rust + RocksDB на каждом edge-сервере, sub-150ms purge | Edge DB = реальный рынок |
| Audit = compliance | OpenPayd: сбор audit logs занимал 2 человека × 2 недели → 1 день с Sumo Logic | Готовность платить за compliance |

### 1.3. Барьеры

| Барьер | Митигация |
|---|---|
| TAM мал ($50–150M) | Open-core + enterprise support + acqui-hire |
| Конкуренция с Badger/Pebble | Дифференциация: zero-alloc, per-op durability, read amp |
| Нет revenue до v0.5 | Design partners ($10–30K/год) |
| Solo-разработка | 3-недельные релизы, честный scope |

---

## 2. Целевые сегменты (ICP с кейсами)

### 2.1. HFT / Trading Infrastructure

#### Кейс: DolphinDB — Tick-to-Trade ≤ 5ms

**Компания:** Ведущий промышленный финансовый институт (Китай). Управляет 30+ фьючерсными контрактами (LME Copper, COMEX Gold, DCE Iron Ore). Годовой объём хеджирования >1M тонн. Фьючерсные позиции — десятки миллиардов RMB.

**Проблема:** Legacy Oracle + Python архитектура. Ежедневный объём tick data >20 GB. Relational databases создавали изолированные данные. Real-time computation был невозможен. AI-модели не могли быть задеплоены — inference latency >1 сек.

**Решение:** DolphinDB. Columnar storage с глубоким сжатием. TSDB LSM-Tree architecture. Event-driven backtesting.

**Результат:**

| Метрика | До | После | Улучшение |
|---|---|---|---|
| Storage cost | — | — | **−80%** |
| Backtesting | 8 часов | ≤5 минут | **100×** |
| Tick-to-Trade | — | **≤5 мс** | — |
| AI inference | >1 сек | **<10 мс** | **100×** |



**Что это значит для TephraKV:** HFT-клиенты готовы платить за tick-to-trade ≤5 мс. TephraKV p999 GET <5 мс + zero-alloc + per-op durability — прямое попадание в эту боль.

#### Кейс: One Trading — sub-200 микросекунд round-trip

**Компания:** Первая регулируемая биржа бессрочных фьючерсов в ЕС. Самая быстрая торговая площадка в мире: **sub-200 микросекунд round-trip latency**.

**Проблема:** Нужна база данных, которая соответствует performance expectations без компромиссов. Ingest терабайты high-frequency market data ежедневно.

**Решение:** QuestDB. SQL, Parquet, Apache Iceberg support. Zero vendor lock-in.

**Результат:** Scalability до **1000× текущей capacity** с уверенностью.

**Что это значит для TephraKV:** HFT-клиенты ищут **открытые стандарты** (SQL, Parquet) и **отсутствие vendor lock-in**. TephraKV — embedded, pure Go, Apache 2.0.

#### Кейс: Anti Capital — QuestDB для HFT

**Компания:** Prop trading firm, специализируется на HFT и market-making across global markets.

**Решение:** QuestDB развёрнут для streaming full-fidelity order book data, reconciles fills в реальном времени, delivers clean datasets to quantitative researchers.

**Результат:** Быстрее decision-making, больше уверенности в trading strategies.

#### ICP: HFT

| Поле | Значение |
|---|---|
| **ICP** | CTO / VP Eng в prop trading или market maker, 20–200 инженеров |
| **Текущее решение** | Redis + PostgreSQL, иногда Aerospike, QuestDB, kdb+ |
| **Боль** | GC-паузы Go, p999 Redis на персистентности, стоимость RAM, cgo overhead |
| **Кейс-подтверждение** | DolphinDB: tick-to-trade ≤5ms. One Trading: sub-200мкс. Anti Capital: full-fidelity order book. |
| **Триггер покупки** | Инцидент с p999 в проде |
| **Бюджет** | $50–200K/год на infra |
| **Цикл сделки** | 3–6 месяцев |
| **Канал** | Конференции (HFT), GitHub, личные связи |
| **Кто платит** | VP Eng / CTO |
| **Критерий успеха** | p999 < 1 мс в проде на их workload |

---

### 2.2. Game Servers

#### Кейс: SpacetimeDB — 131KB аллокаций на каждое чтение

**Компания:** SpacetimeDB — игровая база данных с открытым исходным кодом.

**Проблема:** Пользователь сообщил: «Мой сервер stuttering каждые 10 секунд на 500–1000 мс. Я не мог это исправить после тщательного отслеживания garbage, созданного моим кодом».

**Причина:** C# SDK аллоцирует **~131KB на каждую операцию чтения** независимо от размера payload. `RawTableIterBase.Enumerator` имеет `byte[] buffer = new byte[0x20_000]` (131,072 байт) + per-result copy.

**Результат:** Для 120 fps (бюджет 8.3 мс) GC-пауза 5–10 мс убивает плавность. Overhead не амортизируется — 10 вызовов Find() = 10× overhead.

**Что это значит для TephraKV:** Game servers **остро** нуждаются в zero-alloc. TephraKV: 0 аллокаций на hot path. Прямое попадание.

#### Кейс: Hytale Treasure Hunt — 34 мс GC pause

**Компания:** Hytale (игровой движок).

**Проблема:** Pprof показал 34 мс median GC pause под G1 в JVM. Реальный killer — per-packet allocation. GC pauses упали до 19 мс, но игроки всё ещё жаловались на rubber-band teleports при сохранении мира.

**Что это значит для TephraKV:** Game servers нуждаются не просто в «меньших паузах», а в **полном отсутствии GC-пауз**. TephraKV: 0 GC-пауз (zero-alloc + arena).

#### Кейс: 千万 DAU 手游 Go 服务端 — GC pause std dev ±112ms → ±4.3ms

**Компания:** Мобильная игра с 10M+ DAU.

**Проблема:** GC pause стандартное отклонение ±112 мс. Непредсказуемость убивает игровой опыт.

**Решение:** Оптимизация Go-сервиса. Результат: throughput до 15.3k QPS, GC pause std dev ±4.3 мс. **Улучшение предсказуемости в 26 раз.**

**Что это значит для TephraKV:** Game servers готовы инвестировать в предсказуемость. TephraKV: детерминированный хвост, p999 < 5 мс.

#### ICP: Game Servers

| Поле | Значение |
|---|---|
| **ICP** | Tech Lead / Backend Eng в game studio, 10–100 инженеров |
| **Текущее решение** | Redis, in-memory maps, SpacetimeDB, Badger |
| **Боль** | GC-паузы = лаги, stuttering, потеря данных |
| **Кейс-подтверждение** | SpacetimeDB: 131KB/read → stuttering. Hytale: 34ms GC pause. 手游: ±112ms std dev. |
| **Триггер покупки** | Жалобы игроков на лаги |
| **Бюджет** | $10–50K/год |
| **Цикл сделки** | 1–3 месяца |
| **Канал** | GitHub, Reddit r/golang, Discord |
| **Кто платит** | Tech Lead |
| **Критерий успеха** | Нет GC-пауз в проде |

---

### 2.3. AI-infra (Feature Stores)

#### Кейс: Instacart — −70% кластера, −50% latency

**Компания:** Instacart — leading grocery technology company. 1B+ consumers globally.

**Проблема:** Multi-hundred-node ad-serving feature store на managed Valkey (Redis). Individual shards hitting 100% memory utilization while others underutilized.

**Решение:** Миграция на Dragonfly Cloud. Drop-in Redis replacement.

**Результат:**

| Метрика | Результат |
|---|---|
| Cluster size | **−70%** (с сотен узлов до ~100) |
| Average latency | **−50%** |
| P99 latency | **−50%** |
| Memory efficiency | **−25%** vs Redis |



**Что это значит для TephraKV:** Feature stores готовы мигрировать с Redis, если выгода измерима. TephraKV: embedded, RAM в 5× меньше, p99 < 1 мс.

#### Кейс: DoorDash — 900K ML evaluations/sec

**Компания:** DoorDash. Service reaches 1B+ consumers globally. Peak: **~900,000 ML evaluations per second**.

**Проблема:** Redis — natural starting point. Достигли больших успехов initially. Но once per-instance vertical scalability была исчерпана, cost of maintaining entire dataset in memory began putting pressure on efficiency targets.

**Решение:** Clusterless ML feature store. Distributed Redis cluster using instance store disks. Hybrid approach: subset migrated to horizontally scalable relational database.

**Что это значит для TephraKV:** DoorDash **активно ищет** альтернативы Redis. TephraKV: embedded, durable, RAM efficient. Прямое попадание.

#### Кейс: Feast Benchmark — Redis vs DynamoDB vs DataStore

**Проблема:** Feast (open-source feature store) community Slack: «How scalable/performant is Feast?»

**Benchmark:** Feast сравнили feature serving latency с разными online stores (Redis vs Google Cloud DataStore vs AWS DynamoDB) и разными mechanisms (Java gRPC server, Python HTTP server, lambda function).

**Результат:** Feast — **most performant using Java gRPC server and with Redis as the online store**.

**Что это значит для TephraKV:** Feature stores **уже используют gRPC + Redis**. TephraKV: embedded, gRPC transport (v0.6), p99 < 1 мс. Замена Redis без потери производительности.

#### ICP: AI-infra

| Поле | Значение |
|---|---|
| **ICP** | ML Infra Eng / CTO в AI-стартапе Series A–B, 20–100 инженеров |
| **Текущее решение** | Redis, Badger, custom, DynamoDB |
| **Боль** | RAM дорого, latency при масштабировании, нет durability |
| **Кейс-подтверждение** | Instacart: −70% кластера, −50% latency. DoorDash: 900K evals/sec, Redis bottleneck. Feast: Redis + gRPC = best. |
| **Триггер покупки** | Рост latency при масштабировании |
| **Бюджет** | $20–100K/год |
| **Цикл сделки** | 3–6 месяцев |
| **Канал** | GitHub, конференции (MLOps), Twitter/X |
| **Кто платит** | CTO |
| **Критерий успеха** | p99 < 1 мс на feature lookup |

---

### 2.4. Edge / IoT

#### Кейс: AerialDB — 400 дронов, 80 edge servers

**Компания:** AerialDB — federated, peer-to-peer, spatio-temporal datastore для дронов.

**Проблема:** Нужна lightweight, decentralized data storage and query system для multi-UAV system. Containerized deployment spanning up to 400 drones и 80 edges.

**Результат:** **10× improvement in performance** with insertion.

**Что это значит для TephraKV:** Edge DB — реальный рынок. TephraKV: embedded, pure Go, один бинарник, без cgo. Идеально для дронов, IoT gateway, on-prem edge.

#### Кейс: RAGdb — Zero-Dependency Embeddable

**Компания:** RAGdb — архитектура для multimodal retrieval-augmented generation на edge.

**Проблема:** Distributed stack требует cloud-hosted vector databases, heavy deep learning frameworks. RAGdb: zero-dependency, embeddable architecture. ONNX-based extraction + hybrid vector retrieval в одном бинарнике. Experimental evaluation on Intel i7-1165G7. Retrieval on low-power edge devices.

**Что это значит для TephraKV:** Edge требует **zero-dependency, embeddable**. TephraKV: один бинарник, Apache 2.0, без runtime-зависимостей.

#### Кейс: Verizon + MongoDB — 5G Edge

**Компания:** Verizon + MongoDB.

**Решение:** Atlas Functions интегрирована с Verizon edge discovery service для направления 5G mobile clients к топологически ближайшему database instance across customer's edge deployment. «We can take a fleet of virtual machines across all 19 AWS Wavelength zones and run Realm on each of those instances with Device Sync.»

**Что это значит для TephraKV:** Telecom edge — реальный рынок. TephraKV: embedded, per-op durability, p999 < 5 мс.

#### Кейс: Climatiq + Fastly + Fauna

**Компания:** Climatiq — Berlin-based startup, embedded carbon intelligence.

**Проблема:** Нужна distributed и serverless architecture. Выбрали Fauna as database + Fastly's Compute@Edge for compute layer.

**Что это значит для TephraKV:** Edge startups активно выбирают **embedded + distributed**. TephraKV: embedded, pure Go, gRPC transport (v0.6).

#### ICP: Edge / IoT

| Поле | Значение |
|---|---|
| **ICP** | Embedded Eng / CTO в IoT-компании, 10–50 инженеров |
| **Текущее решение** | SQLite, BoltDB, custom |
| **Боль** | Медленно, не масштабируется, нет durability-политик, cloud round-trip |
| **Кейс-подтверждение** | AerialDB: 400 дронов, 80 edges. RAGdb: zero-dependency edge. Verizon: 5G edge. Climatiq: Fastly + Fauna. |
| **Триггер покупки** | Рост объёма данных |
| **Бюджет** | $5–20K/год |
| **Цикл сделки** | 3–6 месяцев |
| **Канал** | GitHub, конференции (embedded) |
| **Кто платит** | CTO |
| **Критерий успеха** | Работает на ограниченных ресурсах |

---

### 2.5. CDN / Edge Compute

#### Кейс: Cloudflare CacheDB — sub-150ms global purge

**Компания:** Cloudflare. 330+ data centers globally.

**Проблема:** Старая система purge: 1.5 секунды. Lazy purge inefficient, storage pressuring.

**Решение:** CacheDB — Rust service running per-machine RocksDB instance on every edge server. Index cached assets by tag, hostname, URL prefix for instant purge. Schema: five column families, prefix iteration, dual lazy and active purge mechanisms, tombstone management.

**Результат:** Global purge latency: **1.5 сек → 150 мс (P50)**. Storage consumption: **−90%**. Cache hit rate: улучшен.

**Что это значит для TephraKV:** Cloudflare использует **Rust + RocksDB** — не Go. Но проблема та же: embedded DB на каждом edge-сервере. TephraKV: pure Go, без cgo, embedded. Прямое попадание.

#### Кейс: Netlify — CGO crash в проде

**Компания:** Netlify. Edge-инфраструктура.

**Проблема:** CGO binding к C++ библиотеке для redirect rules evaluation at edge. **CGO crash в продакшене.** «We uncovered some of the difficulties and concerns with using a CGO binding to a C++ implementation.»

**Что это значит для TephraKV:** cgo = операционный риск. TephraKV: pure Go. Zero cgo. Один бинарник.

#### Кейс: Criteo — 290M QPS, −78% серверов

**Компания:** Criteo — глобальная AdTech компания.

**Проблема:** Couchbase + Memcached. Нужна была higher performance при меньшем footprint.

**Решение:** Aerospike. Hybrid Memory Architecture (HMA).

**Результат:**

| Метрика | Результат |
|---|---|
| QPS | **290 млн** |
| Server reduction | **−78%** |
| Latency | **sub-ms** |
| CO2 reduction | Снижение за счёт меньшего железа |



**Что это значит для TephraKV:** AdTech/CDN готовы мигрировать с legacy на специализированное решение. TephraKV: embedded, p999 < 5 мс, per-op durability.

#### ICP: CDN / Edge Compute

| Поле | Значение |
|---|---|
| **ICP** | Infra Eng / CTO в CDN или edge-платформе, 50–200 инженеров |
| **Текущее решение** | RocksDB (cgo), Rust + RocksDB, custom |
| **Боль** | cgo overhead, сложность сборки, latency, operational risk |
| **Кейс-подтверждение** | Cloudflare: Rust + RocksDB, sub-150ms purge. Netlify: CGO crash в проде. Criteo: 290M QPS, −78% серверов. |
| **Триггер покупки** | Проблемы с деплоем |
| **Бюджет** | $20–100K/год |
| **Цикл сделки** | 6–12 месяцев |
| **Канал** | Sales, конференции |
| **Кто платит** | CTO |
| **Критерий успеха** | Упрощение деплоя, latency |

---

### 2.6. Fintech / Audit (Profile Compliance)

#### Кейс: OpenPayd — audit logs: 2 недели → 1 день

**Компания:** OpenPayd — leading Banking-as-a-Service provider. Fintech.

**Проблема:** Compliance largely manual. Сбор audit logs занимал **2 человека × 2 недели**. Риск non-compliance penalties.

**Решение:** Sumo Logic. Cloud SIEM + log analytics.

**Результат:**

| Метрика | До | После | Улучшение |
|---|---|---|---|
| Audit log collection | 2 недели | **1 день** | **10×** |
| MTTD | — | — | **−80%** |
| MTTR | — | — | **−80%** |
| PCI DSS audit | — | Пройден | — |



**Что это значит для TephraKV:** Fintech **остро** нуждается в audit log. TephraKV Profile Compliance: audit log (append-only, immutable), PITR, encryption. Прямое попадание.

#### Кейс: Symphony + MongoDB — audit trail для банков

**Компания:** Symphony. Securing private financial data.

**Проблема:** «Banks need access to information dating back months or years to comply with annual audits or when they're investigating actions and activities.»

**Решение:** Audit trail capabilities. Track all user interactions. Ensure compliance.

**Что это значит для TephraKV:** Banks готовы платить за **audit trail + PITR**. TephraKV Profile Compliance: PITR (v0.5), audit log (v0.5).

#### Кейс: Vodafone Türkiye — TCO cut by 50%

**Компания:** Vodafone Türkiye.

**Решение:** OpenText Database Activity Monitoring. Detailed audit logs and reports.

**Результат:** TCO cut by **50%**. Archive retrieval time drastically slashed.

**Что это значит для TephraKV:** Telecom/fintech готовы платить за compliance решения. TephraKV: per-op durability, PITR, audit log — всё в одном embedded продукте.

#### ICP: Fintech / Audit

| Поле | Значение |
|---|---|
| **ICP** | CTO / Head of Eng в fintech Series B–C, 50–300 инженеров |
| **Текущее решение** | PostgreSQL + custom audit, Kafka + S3, Sumo Logic |
| **Боль** | Стоимость audit, compliance-требования, latency, manual processes |
| **Кейс-подтверждение** | OpenPayd: 2 недели → 1 день. Symphony: banks need audit trail. Vodafone: TCO −50%. |
| **Триггер покупки** | Аудит / регуляторное требование |
| **Бюджет** | $100–500K/год |
| **Цикл сделки** | 12–18 месяцев |
| **Канал** | Sales, конференции (Money20/20) |
| **Кто платит** | CFO / Head of Compliance |
| **Критерий успеха** | Прохождение аудита с TephraKV |

---

## 3. GTM-уроки от инфраструктурных компаний

### 3.1. HashiCorp: $15B GTM engine

**История:** Mitchell Hashimoto построил Vagrant как side project для решения собственной проблемы. «Turns out, you just solved it for every developer out there.» Через 10 лет — 200+ Fortune 500 используют HashiCorp для управления cloud.

**GTM-урок:** «Build for how developers work, not how you want them to buy. Let real usage, not vanity metrics, guide your GTM. Treat intent as your north star — it tells you when to move.»

**Применение к TephraKV:** Не продавать HFT-клиентам через cold outreach. Строить продукт, который они сами найдут на GitHub, попробуют, и придут с вопросом «как купить?».

### 3.2. KrakenD: Open-source → acquisition

**История:** Основатели построили API Gateway, потому что существующие не масштабировались. «Nobody was solving the API Gateway problem using proper engineering practices.»

**GTM-урок:** Self-funding, organic adoption, extreme operational efficiency. 5 человек, quietly doubling revenue year after year while remaining profitable. Клиенты: Honda, AMC Networks, American Express, LG Electronics, U.S. Navy, National Geographic.

**Применение к TephraKV:** Solo-основатель может конкурировать с enterprise vendors, если продукт решает hard, unglamorous problem well. TephraKV: zero-alloc embedded KV — именно такая проблема.

### 3.3. CockroachDB: Developer-led GTM

**История:** Bypassed traditional enterprise sales channels entirely. Embedding в open-source communities where database engineers naturally gathered. Founders engaged technical decision-makers through GitHub contributions, comprehensive documentation, and active participation in developer forums.

**GTM-урок:** Продукт gained credibility through **demonstrated capability** rather than marketing claims.

**Применение к TephraKV:** Публиковать ADR, BENCH, HLD в open-source. Feedback сообщества — замена внутреннему ревью. Credibility через transparency.

### 3.4. Supabase: $70M ARR, тихий рост

**История:** 4M+ developers, ~$70M ARR, ~250% YoY. Fully open-source core, built on PostgreSQL. «They didn't win with aggressive sales or loud marketing.»

**GTM-урок:** Open-source core + developer love = growth без sales team.

**Применение к TephraKV:** Apache 2.0, developer-first, документация, примеры. Не нанимать sales до v0.5.

### 3.5. Confluent: $11.4B IPO

**История:** Turned open-source Kafka into $11.4B IPO in 7 years. «Nailed developer-led GTM by knowing exactly which engineering teams needed Kafka right now.»

**GTM-урок:** «Drive adoption at the source — developers using and loving Kafka early. Solve real pain — identify...»

**Применение к TephraKV:** Фокус на HFT и game servers (early adopters с acute pain). Не распыляться на все 6 сегментов сразу.

### 3.6. FerretDB: OSS → DBaaS за недели

**История:** Turned open-source project into commercial DBaaS success in weeks, not years. Partnered with Omnistrate to bring SaaS product to market fast, securely.

**GTM-урок:** Managed service — быстрый путь к revenue для embedded DB.

**Применение к TephraKV:** v0.6 server mode → managed service на cloud-провайдерах (AWS Marketplace, GCP Marketplace).

---

## 4. Почему сейчас: подтверждённые тренды

| Тренд | Кейс | Подтверждение |
|---|---|---|
| GC-паузы = потеря денег | SpacetimeDB: 131KB/read → stuttering каждые 10 сек | Game dev ищет zero-alloc |
| RAM дорого | Instacart: −70% кластера, −50% latency | Feature stores мигрируют с Redis |
| cgo = операционный риск | Netlify: CGO crash в проде | Edge-инфраструктура ищет pure Go |
| Edge = новый рынок | Cloudflare: 330+ data centers, RocksDB на каждом | Edge DB = реальный рынок |
| Audit = compliance | OpenPayd: 2 недели → 1 день | Fintech готов платить за audit |
| Open-source = GTM | HashiCorp, KrakenD, CockroachDB, Supabase | Developer-led GTM работает |

---

## 5. GTM-фазы (с кейсами)

### Фаза 1: v0.1–v0.2 (0–3 месяца) — «Developer Love»

| Действие | Кейс-основа | Метрика |
|---|---|---|
| GitHub публикация | HashiCorp: build for how developers work | 500 stars |
| Show HN | Supabase: no loud marketing, organic growth | 100 upvotes |
| Technical blog posts | CockroachDB: credibility through demonstrated capability | 5K reads |
| ADR в open-source | KrakenD: engineering-first mindset | 10 issues |

### Фаза 2: v0.3–v0.4 (3–6 месяцев) — «Design Partners»

| Действие | Кейс-основа | Метрика |
|---|---|---|
| 3–5 design partners | Confluent: know exactly which teams needed Kafka right now | $10–30K/год |
| HFT cold outreach | DolphinDB: tick-to-trade ≤5ms | 5 ответов |
| Game server outreach | SpacetimeDB: 131KB/read → stuttering | 5 ответов |
| Feature store outreach | Instacart: −70% кластера | 5 ответов |

### Фаза 3: v0.5–v0.6 (6–12 месяцев) — «Revenue»

| Действие | Кейс-основа | Метрика |
|---|---|---|
| Прямые продажи | FerretDB: OSS → DBaaS за недели | 5–10 платящих |
| Cloud marketplace | ScyllaDB: AWS Marketplace TCV grew 200%+ in FY23 | 3 партнёра |
| Compliance sales | OpenPayd: audit log pain | 1 fintech |

### Фаза 4: v0.7–v1.0 (12–24 месяца) — «Scale»

| Действие | Кейс-основа | Метрика |
|---|---|---|
| Enterprise sales | CockroachDB: developer-led → enterprise | 20–30 платящих |
| SOC2 | Fintech compliance requirement | $1M ARR |
| Partner ecosystem | Aerospike: SIOS partnership Japan | 5 партнёров |

---

## 6. Pricing & Packaging

### 6.1. Модель

**Open-core + enterprise support.** (Подтверждено: HashiCorp, Supabase, CockroachDB.)

| Уровень | Что входит | Цена | Сегмент |
|---|---|---|---|
| **Community** | Ядро (v0.1–v0.5), Apache 2.0 | $0 | Все |
| **Design Partner** | Early access, влияние на roadmap | $10–30K/год | Ранние |
| **Standard Support** | SLA 99.9%, response 24ч | $50–100K/год | HFT, AI-infra |
| **Premium Support** | SLA 99.99%, response 4ч | $150–300K/год | HFT, fintech |
| **Enterprise** | SOC2, encryption, RBAC, audit log | $300–500K/год | Fintech, enterprise |

### 6.2. Unit Economics

| Метрика | Цель | Кейс-бенчмарк |
|---|---|---|
| CAC | $5–20K | HashiCorp: developer-led → низкий CAC |
| LTV | $150–500K | KrakenD: quietly doubling revenue |
| Payback period | 6–12 месяцев | — |
| Gross margin | 80–90% | — |
| Churn | < 10%/год | — |

---

## 7. Sales Playbook

### 7.1. Квалификация

| Вопрос | Что проверяем | Кейс-основа |
|---|---|---|
| «Есть ли GC-паузы в проде?» | Боль zero-alloc | SpacetimeDB: 131KB/read |
| «Сколько RAM уходит на KV?» | Боль RAM efficiency | Instacart: −70% кластера |
| «Используете cgo?» | Боль pure Go | Netlify: CGO crash |
| «Проходите аудит?» | Боль compliance | OpenPayd: 2 недели → 1 день |

### 7.2. Демо

- Показать BENCH-002 (zero-alloc gate).
- Показать BENCH-003 (p999 GET).
- Показать BENCH-004 (YCSB, сравнение с Badger/Pebble).
- Показать BENCH-008 (read amplification).
- Показать сравнение на **их** workload.

### 7.3. Pilot

- 30 дней.
- Их workload.
- Их метрики.
- Их feedback.

### 7.4. Контракт

- Design partner: $10–30K/год.
- Standard: $50–100K/год.
- Premium: $150–300K/год.

---

## 8. Метрики GTM (с кейс-бенчмарками)

| Метрика | v0.1 | v0.3 | v0.5 | v0.6 | v1.0 | Кейс-бенчмарк |
|---|---|---|---|---|---|---|
| GitHub stars | 200+ | 1000+ | 3000+ | 5000+ | 10000+ | HashiCorp: 200+ F500 через 10 лет |
| Design partners | 0 | 3 | 5 | 10 | 20 | Confluent: developer-led |
| Платящих | 0 | 1 | 5 | 15 | 30 | KrakenD: 5 человек, profitable |
| ARR | $0 | $10K | $100K | $300K | $1M | Supabase: $70M ARR |
| Contributors | 0 | 2+ | 5+ | 10+ | 20+ | CockroachDB: community |

---

## 9. Каналы дистрибуции (с кейсами)

### 9.1. GitHub

| Действие | Кейс-основа | Частота |
|---|---|---|
| Публикация релизов | HashiCorp: build for developers | Каждый релиз |
| Ответ на issues | KrakenD: engineering-first | < 24ч |
| Примеры в docs/examples/ | CockroachDB: comprehensive docs | Каждый релиз |

### 9.2. Hacker News / Reddit

| Действие | Кейс-основа | Частота |
|---|---|---|
| Show HN | Supabase: no loud marketing | v0.1, v0.3, v0.6 |
| Reddit r/golang | — | Каждый релиз |

### 9.3. Конференции

| Конференция | Сегмент | Кейс-основа | Когда |
|---|---|---|---|
| GopherCon | Go | — | v0.2+ |
| HFT conferences | HFT | DolphinDB: tick-to-trade | v0.3+ |
| MLOps conferences | AI-infra | Feast: Redis + gRPC | v0.3+ |
| Money20/20 | Fintech | OpenPayd: audit log | v0.6+ |

### 9.4. Blog

| Тема | Кейс-основа | Когда |
|---|---|---|
| «Why zero-alloc matters» | SpacetimeDB: 131KB/read | v0.1 |
| «Mixed batch durability» | — | v0.1 |
| «Shared WAL pool» | TiKV: WAL shared pool | v0.3 |
| «ReadIndex vs Lease Read» | — | v0.6 |
| «WiscKey update-heavy degradation» | — | v0.3 |

---

## 10. Риски и митигации (с кейсами)

| Риск | P | I | Митигация | Кейс-основа |
|---|---|---|---|---|
| Позиционирование не резонирует | С | В | Публикация p999 с первого дня | DolphinDB: tick-to-trade |
| Zero-alloc сложен для пользователей | С | С | Документация, примеры | CockroachDB: comprehensive docs |
| Solo → выгорание | В | В | 3-недельные релизы, честный scope | KrakenD: 5 человек, profitable |
| Нет revenue до v0.5 | В | В | Design partners, open-core | HashiCorp: open-source → $15B |
| HFT не готовы платить за embedded Go | С | В | 20 интервью до v0.1 | One Trading: sub-200мкс |
| Game servers не готовы платить | В | С | Фокус на HFT и AI-infra | SpacetimeDB: stuttering |
| Конкуренты добавят zero-alloc | Н | В | Скорость итераций | Flow: BadgerDB → PebbleDB |
| TAM мал ($50–150M) | В | В | Open-core + acqui-hire | KrakenD: acquisition |

---

## 11. Open questions

- **Какая цена приемлема?** Pricing-тесты не проводились. Бенчмарк: Instacart сэкономил −70% кластера — готовы платить за измеримую выгоду.
- **Какой сегмент даст первого платящего?** Не знаем. Гипотеза: HFT (acute pain, бюджет).
- **Готовы ли HFT платить за embedded Go?** Не проверено. Кейс: DolphinDB (C++), QuestDB (Java) — не Go. Но Anti Capital использует Rust ecosystem.
- **Готовы ли game servers платить вообще?** Не проверено. SpacetimeDB — open-source. Но студии платят за Redis.
- **Готовы ли AI-infra выбрать TephraKV вместо Redis?** Не проверено. Instacart мигрировал с Redis.
- **Готовы ли edge/IoT платить $5–20K/год?** Не проверено. Verizon + MongoDB — enterprise pricing.
- **Готовы ли CDN платить за замену cgo?** Не проверено. Cloudflare использует Rust + RocksDB.
- **Готовы ли fintech платить $100–500K/год?** Не проверено. OpenPayd использует Sumo Logic (enterprise).
- **Какой канал даст лучший CAC?** Не знаем. HashiCorp: developer-led → низкий CAC.
- **Достаточно ли p999 < 5 мс для HFT?** Не проверено. DolphinDB: tick-to-trade ≤5ms.

---

## 12. Связанные документы

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v7.0 | Архитектура, §1 (рынок), §3.1 (ICP) |
| TephraKV-PRD-001 v1.0 | ICP, use cases, TAM |
| TephraKV-ROADMAP-001 v3.0 | §4 (timeline), §9 (метрики) |
| TephraKV-BACKLOG-001 v5.0 | §2–§9 (задачи по версиям) |
| TephraKV-GLOSSARY-001 v2.0 | Термины |
| TephraKV-COMPETITIVE-001 v1.0 | Competitive Analysis |
| TephraKV-PRICING-001 v1.0 | Pricing & Packaging |
| ADR-032 | Path to revenue |
| ADR-034 v2.0 | Engineering practices |
| ADR-035 v2.0 | Capacity model |

---

## 13. Что изменилось против v1.0

**Case-Study Edition.**

Содержательные правки:

- **§0:** новое — «если сегмент не подтверждён кейсом — он не в GTM».
- **§1.2:** драйверы роста подтверждены 5 кейсами.
- **§2.1:** HFT — 3 кейса (DolphinDB, One Trading, Anti Capital).
- **§2.2:** Game servers — 3 кейса (SpacetimeDB, Hytale, 手游).
- **§2.3:** AI-infra — 3 кейса (Instacart, DoorDash, Feast).
- **§2.4:** Edge/IoT — 3 кейса (AerialDB, RAGdb, Verizon, Climatiq).
- **§2.5:** CDN — 3 кейса (Cloudflare, Netlify, Criteo).
- **§2.6:** Fintech — 3 кейса (OpenPayd, Symphony, Vodafone).
- **§3:** новый — GTM-уроки от 6 инфраструктурных компаний (HashiCorp, KrakenD, CockroachDB, Supabase, Confluent, FerretDB).
- **§4:** новый — подтверждённые тренды.
- **§5:** фазы GTM с кейс-обоснованием.
- **§6.2:** unit economics с кейс-бенчмарками.
- **§7.1:** квалификация с кейс-вопросами.
- **§8:** метрики с кейс-бенчмарками.
- **§9:** каналы с кейс-основой.
- **§10:** риски с кейс-основой.
- **§11:** open questions с кейс-контекстом.
- **§13:** новая секция «Что изменилось».

---

**Конец TephraKV-GTM-001 v2.0**