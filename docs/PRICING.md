# TephraKV — Pricing & Packaging

**Документ:** TephraKV-PRICING-001
**Версия:** 1.0
**Статус:** Черновик
**Дата:** 22 сентября 2026
**Связь:** TephraKV-HLD-000 v7.0, TephraKV-GTM-001 v2.0, TephraKV-PRD-001 v1.0, TephraKV-ROADMAP-001 v3.0, TephraKV-BACKLOG-001 v5.0, TephraKV-GLOSSARY-001 v2.0, ADR-032, ADR-034 v2.0, ADR-035 v2.0
**Классификация:** Внутренний / публичный

---

## 0. Назначение

PRICING-001 — ценовая модель TephraKV. Источник правды для:

- Какую ценность мы продаём (value metric).
- Как упаковываем продукт (open-core split).
- Сколько стоит каждый уровень (tiers).
- Почему именно столько (обоснование через value ratio).
- Как масштабируется revenue (unit economics).
- Как меняется pricing при росте (evolution).

**Правило:** если цена не обоснована value ratio — она не в PRICING-001.

---

## 1. Принципы ценообразования

### 1.1. Value-based, not cost-based

**Принцип:** цена основана на ценности для покупателя, не на стоимости разработки.

**Обоснование:** Технические основатели склонны привязывать цену к затратам («нам стоит $X, значит берём $2X»). Enterprise-покупатели привязывают цену к ценности («это экономит нам $200K, значит $45K дёшево»). **Разница — это маржа.** 

**Value Ratio:**

```
Value Ratio = Customer's alternative cost / Your price

> 10× → массово недооценён
5–10× → недооценён (большинство стартапов здесь)
3–5× → здоровое ценообразование
< 3× → приближение к потолку
< 2× → дорого (нужна сильная дифференциация)
```



**Как считаем alternative cost:**
- Часы ручного процесса × ставка × частота.
- Стоимость in-house разработки (инженеры × месяцы × loaded cost).
- Стоимость существующего инструмента + switching cost.
- Стоимость **не решения проблемы** (инциденты, downtime, churn). 

### 1.2. Open-core как основа

**Принцип:** ядро — open-source (Apache 2.0), enterprise-фичи — платные.

**Обоснование:** Open-core — самая популярная модель для современных developer tools. Free-версия — это permanent trial, который никогда не заканчивается. Пользователи интегрируют продукт, видят ценность и апгрейдятся, когда упираются в конкретную боль, которую решает Pro-тир. 

**Конверсия free → paid:** 2–10%. Для инфраструктуры выше — 5–10%, для general tools ниже — 2–5%. 

### 1.3. Pricing model precedes feature design

**Принцип:** модель ценообразования определяет дизайн продукта, не наоборот.

**Обоснование:** Ramanujam (Monetizing Innovation): «design the product around the price, not the price around the product». PRD должен фиксировать выбранную модель и объяснять, почему другие отклонены. 

### 1.4. Три модели (и когда каждая ломается)

| Модель | Когда работает | Когда ломается |
|---|---|---|
| **Tier (Good/Better/Best)** | Предсказуемая ценность, чёткие границы упаковки, B2B sales-led | Cliffs на границах тиров; клиенты downgrade вместо upgrade |
| **Usage-based** | Ценность масштабируется с использованием (API calls, GB, events) | Непредсказуемость revenue; billing surprises; churn от sticker shock |
| **Hybrid (platform fee + usage)** | Multi-product, предсказуемая platform value + variable feature value | Сложность коммуникации; клиенты путаются в expected cost |
| **Per-seat** | Collaboration product, ценность линейна по пользователям | Отговаривает от adoption; declining в 2025–2026 |
| **Outcome-based** | Vendor уверен в измеримом outcome | Сложно определить; риск спора об измерении |



**Anti-patterns:**
- Per-seat для AI-продукта: AI-ценность не масштабируется с seats. 
- Usage-based без floor: клиенты могут упасть до $0 в медленный месяц. 
- Tier с одной differentiating feature: клиенты обходят differentiator. 
- Публичный enterprise pricing: оставляет переговорное пространство. 

### 1.5. Enterprise deals drive revenue

**Принцип:** enterprise-сделки приносят большинство revenue в open-core компаниях.

**Обоснование:** Average Contract Value (Enterprise) = $10K–$100K/год. Enterprise deals drive most revenue. 

---

## 2. Анализ конкурентов

### 2.1. Прямые конкуренты (ценовые модели)

| Конкурент | Модель | Цена | Источник |
|---|---|---|---|
| **Badger** | Open-source | $0 (Apache 2.0) | GitHub |
| **Pebble** | Open-source | $0 (BSD-3) | GitHub |
| **RocksDB** | Open-source | $0 (Apache 2.0) | GitHub |
| **Redis** | Open-core + managed cloud | Essentials: $0.007/час (~$5/мес). Pro: $0.014/час, min $200/мес. Enterprise: $10K–$15K/год (small) → $50K–$150K/год |  |
| **Aerospike** | Enterprise + managed cloud | Licensed by volume of unique production data. Median buyer: $149,735/год. Range: $34,500–$170,700 |  |
| **ScyllaDB** | Open-source + enterprise + managed | Free tier до 10TB. Enterprise: от $0.045/час. Managed: subscription или flex credits |  |

### 2.2. Косвенные конкуренты (ценовые модели)

| Конкурент | Модель | Цена | Источник |
|---|---|---|---|
| **HashiCorp (Terraform)** | Open-core + managed (HCP) | HCP: $0.10–$0.99/resource/мес. Terraform Enterprise: self-managed |  |
| **CockroachDB** | Open-core + cloud (usage-based) | Basic: $0.20/1M RU, $0.50/GiB. Standard: $0.18/vCPU/час. Annual: $25K–$200K+ |  |
| **Supabase** | Open-core + managed | Free: $0. Pro: $25/мес. Team: $599/мес. Enterprise: custom. Overages: $0.125/GB disk, $0.09/GB egress |  |
| **KrakenD** | Open-core + enterprise | Flat-tiered plans. Enterprise: от $13,188/год. All features included at every tier |  |
| **Redpanda** | Source-available + enterprise + cloud | Community: free. Enterprise: license key. Cloud: usage-based, от $15,000/год |  |
| **TigerBeetle** | Open-source + fully managed | Apache 2.0: $0. Fully managed: contact sales |  |
| **FerretDB** | Open-source + DBaaS | Open-source: free. DBaaS: via Omnistrate | — |

### 2.3. Выводы из конкурентного анализа

| Инсайт | Применение к TephraKV |
|---|---|
| Open-source ядро — стандарт | Badger, Pebble, RocksDB, ScyllaDB, CockroachDB, Supabase, KrakenD, Redpanda, TigerBeetle |
| Enterprise ACV $10K–$200K/год | Redis, Aerospike, CockroachDB |
| Flat-tiered pricing (KrakenD) — альтернатива usage-based | Проще для клиента, предсказуемее revenue |
| Usage-based (CockroachDB) — для cloud | Подходит для v0.6+ server mode |
| Free tier до 10TB (ScyllaDB) — агрессивный | Для TephraKV: free tier до 100M keys (single-node) |
| Managed service — быстрый путь к revenue (FerretDB) | v0.6+ server mode → cloud marketplace |
| Median enterprise ACV ~$150K (Aerospike) | TephraKV target: $50–200K/год |

---

## 3. Ценностная метрика

### 3.1. Выбор метрики

**Value metric** — единица, по которой масштабируется цена. Должна коррелировать с ценностью для клиента.

| Сегмент | Value metric | Почему |
|---|---|---|
| **HFT** | Nodes / cores | Ценность растёт с количеством узлов и throughput |
| **Game servers** | Nodes / CCU (concurrent users) | Ценность растёт с масштабом игры |
| **AI-infra** | Nodes / feature lookups | Ценность растёт с объёмом feature serving |
| **Edge/IoT** | Devices / nodes | Ценность растёт с количеством edge-устройств |
| **CDN** | Nodes / edge locations | Ценность растёт с географией |
| **Fintech (Compliance)** | Data volume / audit retention | Ценность растёт с объёмом compliance-данных |

### 3.2. Решение: flat-tiered + nodes

**Для v0.1–v0.5 (embedded, single-node):**

Flat-tiered pricing с метрикой **nodes** (количество инстансов TephraKV). Не usage-based (нет server mode, нет cloud).

| Tier | Nodes | Цена | Обоснование |
|---|---|---|---|
| Community | Unlimited | $0 | Дистрибуция |
| Design Partner | Unlimited | $10–30K/год | Early access, влияние на roadmap |
| Standard Support | Unlimited | $50–100K/год | SLA 99.9%, response 24ч |
| Premium Support | Unlimited | $150–300K/год | SLA 99.99%, response 4ч |

**Для v0.6+ (distributed, server mode):**

Hybrid: platform fee + usage-based (nodes + data volume).

| Tier | Platform fee | Nodes | Data volume | Цена |
|---|---|---|---|---|
| Community | $0 | Unlimited | Unlimited | $0 |
| Standard | $10K/год | до 5 | до 100 ГБ | $50–100K/год |
| Premium | $30K/год | до 20 | до 1 ТБ | $150–300K/год |
| Enterprise | Custom | Unlimited | Unlimited | $300–500K/год |

**Обоснование выбора flat-tiered:**

- KrakenD: flat-tiered plans. Клиенты платят одинаково, независимо от traffic spike. 
- Проще для клиента: предсказуемый bill.
- Проще для solo-основателя: не нужно строить metering infrastructure.
- Usage-based — anti-pattern без floor (клиенты могут упасть до $0). 

### 3.3. Альтернативы (отклонены)

| Альтернатива | Почему отклонена |
|---|---|
| Per-seat | TephraKV — embedded KV, не collaboration tool. Ценность не масштабируется с seats.  |
| Pure usage-based | Нет floor → volatile cash flow.  |
| Outcome-based | Сложно измерить outcome для embedded KV. |
| Публичный enterprise pricing | Оставляет переговорное пространство.  |

---

## 4. Модель монетизации: open-core split

### 4.1. Ядро (Community, Apache 2.0)

| Компонент | Версия | Обоснование |
|---|---|---|
| WAL shared pool | v0.1 | Ядро storage |
| MemTable (lock-free skiplist) | v0.1 | Ядро storage |
| SSTable (block format, bloom) | v0.1 | Ядро storage |
| Zero-alloc hot path | v0.1 | Ядро производительности |
| Per-op durability (NO_SYNC, SYNC_MASTER, SYNC_LEADER) | v0.1 | Ядро durability |
| Two-Level Metrics | v0.1 | Ядро observability |
| Read-amp compaction (partition index, MinMax) | v0.2 | Ядро storage |
| SLA-driven compaction scheduler | v0.2 | Ядро storage |
| VLog (WiscKey) | v0.3 | Ядро storage |
| MVCC (single-shard) | v0.4 | Ядро transactions |
| Shard-per-core | v0.4 | Ядро scaling |
| TTL | v0.5 | Ядро storage |
| PITR | v0.5 | Ядро durability |
| Distributed (Multi-Raft, gRPC) | v0.6 | Ядро distributed |
| Server mode (gRPC/REST) | v0.6 | Ядро network |
| Columnar storage | v0.7 | Ядро analytics |

### 4.2. Enterprise-фичи (платные)

| Фича | Версия | Почему платная | Сегмент |
|---|---|---|---|
| **Encryption at rest** | v0.6 | Compliance requirement | Fintech |
| **RBAC** | v1.0 | Enterprise security | Fintech, enterprise |
| **Audit log** (append-only, immutable, export) | v0.5 | Compliance requirement | Fintech |
| **SOC 2 Type II** | v1.0 | Enterprise procurement | Fintech, enterprise |
| **Multi-region** | v1.0 | Enterprise scaling | Fintech, enterprise |
| **SSI** (Serializable Snapshot Isolation) | v1.0 | Enterprise transactions | Fintech |
| **Joint consensus** | v1.0 | Enterprise membership | Enterprise |
| **Support SLA** | v0.3+ | Enterprise support | Все enterprise |
| **Dedicated engineer** | v0.5+ | Premium support | HFT, fintech |

**Обоснование open-core split:**

- Ядро — то, что нужно **всем** пользователям (HLD v7.0 §6.1). Open-source для дистрибуции.
- Enterprise-фичи — то, за что **платят** enterprise-клиенты: security, compliance, management, support. 
- **Правило:** paid-фичи живут в отдельном репозитории (`tephrakv-enterprise`), не в OSS-репо. Это защита от scope drift и случайного контрибьюта. 

### 4.3. Managed cloud (v0.6+)

| Уровень | Модель | Цена | Сегмент |
|---|---|---|---|
| **Cloud Basic** | Usage-based | $0.20/1M RU, $0.50/GiB | AI-infra, edge |
| **Cloud Standard** | Provisioned + usage | $0.18/vCPU/час + storage | Game servers, CDN |
| **Cloud Premium** | Provisioned + usage | Custom | HFT, fintech |

**Обоснование:** Managed service — быстрый путь к revenue для embedded DB (FerretDB precedent). Cloud marketplace (AWS, GCP) — дистрибуция. 

---

## 5. Упаковка (Packaging)

### 5.1. Tier definition

| План | Persona | Цена | Core value | Included | Limits |
|---|---|---|---|---|---|
| **Community** | Solo dev, OSS project | $0 | Embedded KV с zero-alloc | Ядро (v0.1–v0.5), Apache 2.0 | Unlimited |
| **Design Partner** | Early adopter, startup Series A–B | $10–30K/год | Early access, влияние на roadmap | Ядро + приоритетные багфиксы + ADR feedback | Unlimited |
| **Standard Support** | Production team, startup Series B–C | $50–100K/год | SLA 99.9%, production support | Ядро + Standard SLA + response 24ч + Prometheus exporter | Unlimited |
| **Premium Support** | HFT, AI-infra (latency-critical) | $150–300K/год | SLA 99.99%, dedicated engineer | Ядро + Premium SLA + response 4ч + dedicated engineer + audit log | Unlimited |
| **Enterprise** | Fintech, large enterprise | $300–500K/год | Compliance, security, multi-region | Ядро + всё выше + encryption at rest + RBAC + SOC2 + multi-region + SSI + joint consensus | Unlimited |

### 5.2. Feature matrix

| Фича | Community | Design Partner | Standard | Premium | Enterprise |
|---|---|---|---|---|---|
| Ядро (WAL, LSM, VLog) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Zero-alloc hot path | ✅ | ✅ | ✅ | ✅ | ✅ |
| Per-op durability | ✅ | ✅ | ✅ | ✅ | ✅ |
| Distributed (v0.6+) | ✅ | ✅ | ✅ | ✅ | ✅ |
| TTL (v0.5+) | ✅ | ✅ | ✅ | ✅ | ✅ |
| PITR (v0.5+) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Encryption at rest (v0.6+) | ❌ | ❌ | ❌ | ✅ | ✅ |
| Audit log (v0.5+) | ❌ | ❌ | ❌ | ✅ | ✅ |
| RBAC (v1.0+) | ❌ | ❌ | ❌ | ❌ | ✅ |
| SOC 2 (v1.0+) | ❌ | ❌ | ❌ | ❌ | ✅ |
| Multi-region (v1.0+) | ❌ | ❌ | ❌ | ❌ | ✅ |
| SSI (v1.0+) | ❌ | ❌ | ❌ | ❌ | ✅ |
| Support SLA | ❌ | Best effort | 99.9% | 99.99% | 99.99% |
| Response time | ❌ | 72ч | 24ч | 4ч | 1ч |
| Dedicated engineer | ❌ | ❌ | ❌ | ✅ | ✅ |

---

## 6. Ценообразование по сегментам

### 6.1. HFT / Trading Infrastructure

| Поле | Значение |
|---|---|
| **Tier** | Premium Support ($150–300K/год) |
| **Value metric** | Nodes |
| **Value ratio** | Tick-to-trade ≤5 мс (DolphinDB). Alternative: in-house C++ решение ($500K+/год). Value ratio: 500/200 = **2.5×**. Healthy pricing. |
| **Willingness-to-pay** | Высокая (p999 = деньги) |
| **Бюджет** | $50–200K/год на infra |
| **Кейс-подтверждение** | DolphinDB: tick-to-trade ≤5ms. One Trading: sub-200мкс. Anti Capital: full-fidelity order book. |

### 6.2. Game Servers

| Поле | Значение |
|---|---|
| **Tier** | Standard Support ($50–100K/год) |
| **Value metric** | Nodes / CCU |
| **Value ratio** | SpacetimeDB: 131KB/read → stuttering. Alternative: in-house Go-оптимизация ($200K+/год). Value ratio: 200/75 = **2.7×**. |
| **Willingness-to-pay** | Средняя (GC-паузы = лаги) |
| **Бюджет** | $10–50K/год |
| **Кейс-подтверждение** | SpacetimeDB: 131KB/read. Hytale: 34ms GC pause. 手游: ±112ms std dev. |

### 6.3. AI-infra (Feature Stores)

| Поле | Значение |
|---|---|
| **Tier** | Standard Support ($50–100K/год) |
| **Value metric** | Nodes / feature lookups |
| **Value ratio** | Instacart: −70% кластера, −50% latency. Alternative: ElastiCache ($500K+/год). Value ratio: 500/75 = **6.7×**. Undervalued. |
| **Willingness-to-pay** | Средняя (RAM efficiency) |
| **Бюджет** | $20–100K/год |
| **Кейс-подтверждение** | Instacart: −70% кластера. DoorDash: 900K evals/sec. Feast: Redis + gRPC = best. |

### 6.4. Edge / IoT

| Поле | Значение |
|---|---|
| **Tier** | Community ($0) или Standard ($50–100K/год) |
| **Value metric** | Devices / nodes |
| **Value ratio** | AerialDB: 400 дронов. Alternative: cloud DB ($100K+/год). Value ratio: 100/50 = **2×**. Expensive. Need strong differentiation. |
| **Willingness-to-pay** | Низкая (цена важнее) |
| **Бюджет** | $5–20K/год |
| **Кейс-подтверждение** | AerialDB: 400 дронов. RAGdb: zero-dependency edge. Verizon: 5G edge. |

### 6.5. CDN / Edge Compute

| Поле | Значение |
|---|---|
| **Tier** | Standard Support ($50–100K/год) |
| **Value metric** | Nodes / edge locations |
| **Value ratio** | Cloudflare: sub-150ms purge. Alternative: RocksDB + Rust ($1M+/год). Value ratio: 1000/75 = **13.3×**. Massively undervalued. |
| **Willingness-to-pay** | Средняя (cgo overhead) |
| **Бюджет** | $20–100K/год |
| **Кейс-подтверждение** | Cloudflare: Rust + RocksDB. Netlify: CGO crash. Criteo: 290M QPS, −78% серверов. |

### 6.6. Fintech / Audit (Profile Compliance)

| Поле | Значение |
|---|---|
| **Tier** | Enterprise ($300–500K/год) |
| **Value metric** | Data volume / audit retention |
| **Value ratio** | OpenPayd: 2 недели → 1 день. Alternative: Sumo Logic ($500K+/год). Value ratio: 500/400 = **1.25×**. Approaching ceiling. Need strong differentiation. |
| **Willingness-to-pay** | Высокая (compliance) |
| **Бюджет** | $100–500K/год |
| **Кейс-подтверждение** | OpenPayd: 2 недели → 1 день. Symphony: banks need audit trail. Vodafone: TCO −50%. |

---

## 7. Unit Economics

### 7.1. CAC (Customer Acquisition Cost)

| Сегмент | Канал | CAC | Обоснование |
|---|---|---|---|
| HFT | Cold outreach + конференции | $15–20K | Высокий touch, долгий цикл |
| Game servers | GitHub + Reddit | $5–10K | Developer-led, низкий CAC |
| AI-infra | GitHub + конференции | $10–15K | Developer-led |
| Edge/IoT | GitHub | $5–10K | Developer-led |
| CDN | Sales | $15–20K | Enterprise sales |
| Fintech | Sales + конференции | $20K | Долгий цикл (12–18 мес) |

**Кейс-бенчмарк:** HashiCorp: developer-led → низкий CAC. 

### 7.2. LTV (Lifetime Value)

| Сегмент | ARPU | Churn | LTV | LTV/CAC |
|---|---|---|---|---|
| HFT | $200K/год | 5% | $4M | **200×** |
| Game servers | $50K/год | 10% | $500K | **50×** |
| AI-infra | $75K/год | 8% | $937K | **62×** |
| Edge/IoT | $15K/год | 15% | $100K | **10×** |
| CDN | $75K/год | 8% | $937K | **47×** |
| Fintech | $400K/год | 5% | $8M | **200×** |

**Кейс-бенчмарк:** KrakenD: quietly doubling revenue. 

### 7.3. Payback period

| Сегмент | CAC | ARPU/мес | Payback |
|---|---|---|---|
| HFT | $20K | $16.7K | **1.2 мес** |
| Game servers | $10K | $4.2K | **2.4 мес** |
| AI-infra | $15K | $6.25K | **2.4 мес** |
| Edge/IoT | $10K | $1.25K | **8 мес** |
| CDN | $20K | $6.25K | **3.2 мес** |
| Fintech | $20K | $33.3K | **0.6 мес** |

### 7.4. Gross margin

| Компонент | Cost | % от revenue |
|---|---|---|
| Support (SRE-часы) | 10% | |
| Infrastructure (CI, тесты) | 5% | |
| **Gross margin** | **85%** | |

**Кейс-бенчмарк:** Open-core companies: 80–90% gross margin. 

---

## 8. Ценовая политика

### 8.1. Grandfathering

**Принцип:** существующие клиенты сохраняют старую цену при повышении.

**Обоснование:** Price increase — sensitive. Grandfathering снижает churn. 

**Правило:** при повышении цен — grandfathering на 12 месяцев. После — новая цена.

### 8.2. Скидки

| Условие | Скидка | Обоснование |
|---|---|---|
| Annual prepay | 15–20% | Cash flow |
| Multi-year (2–3 года) | 25–30% | Predictable revenue |
| Startup (<$10M revenue) | 50% | CockroachDB precedent: free for <$10M revenue  |
| OSS project | 100% | Дистрибуция |

### 8.3. Regional pricing

| Регион | Множитель | Обоснование |
|---|---|---|
| US / EU | 1.0× | Base |
| India / SEA | 0.5× | PPP |
| LATAM | 0.6× | PPP |
| Eastern Europe | 0.7× | PPP |

### 8.4. Rollback criteria

| Метрика | Порог | Действие |
|---|---|---|
| Churn increase | >5% за 3 мес | Пересмотр цен |
| Conversion drop | >20% за 3 мес | Пересмотр упаковки |
| NPS drop | >10 пунктов | Пересмотр value proposition |
| Customer complaints | >10/мес | Пересмотр communication |

---

## 9. Evolution pricing по версиям

| Версия | Модель | Цена | Обоснование |
|---|---|---|---|
| v0.1–v0.2 | Community only | $0 | Дистрибуция, adoption |
| v0.3–v0.4 | Community + Design Partner | $0 / $10–30K/год | Early access, feedback |
| v0.5 | + Standard Support | $50–100K/год | Production support, первый платящий |
| v0.6 | + Premium Support + Cloud | $150–300K/год + usage | Distributed, HTAP |
| v0.7 | + Columnar tier | Custom | Analytics |
| v1.0 | + Enterprise | $300–500K/год | Compliance, security, SOC2 |

**Правило:** не вводить платные тиры до v0.3. Сначала — adoption, потом — monetization.

---

## 10. Риски и митигации

| Риск | P | I | Митигация |
|---|---|---|---|
| Underpricing (Value Ratio >5×) | В | В | Ежегодный pricing review |
| Overpricing (Value Ratio <3×) | С | В | 20 интервью до v0.1 |
| TAM мал ($50–150M) | В | В | Open-core + enterprise support + acqui-hire |
| Конкуренты демпингуют | С | С | Дифференциация: zero-alloc, per-op durability, read amp |
| Нет revenue до v0.5 | В | В | Design partners ($10–30K/год) |
| Grandfathering съедает revenue | С | С | Ограничить 12 месяцами |
| Churn от price increase | С | В | Value-based pricing, grandfathering |
| Enterprise sales долгий | В | С | Developer-led GTM, open-source credibility |

---

## 11. Open questions

- **Какая цена приемлема?** Pricing-тесты не проводились.
- **Какой сегмент даст первого платящего?** Гипотеза: HFT (acute pain, бюджет).
- **Готовы ли HFT платить $150–300K/год?** Не проверено.
- **Готовы ли game servers платить $50–100K/год?** Не проверено.
- **Готовы ли AI-infra платить $50–100K/год?** Не проверено.
- **Готовы ли edge/IoT платить $5–20K/год?** Не проверено.
- **Готовы ли CDN платить $50–100K/год?** Не проверено.
- **Готовы ли fintech платить $300–500K/год?** Не проверено.
- **Какая модель лучше: flat-tiered или usage-based?** Не проверено.
- **Достаточно ли Value Ratio 2–3× для win?** Не проверено.

**Правило:** неизвестное документируется как неизвестное.

---

## 12. Связанные документы

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v7.0 | Архитектура, §1 (рынок), §3.1 (ICP) |
| TephraKV-GTM-001 v2.0 | Go-to-Market Strategy (Case-Study Edition) |
| TephraKV-PRD-001 v1.0 | ICP, use cases, TAM |
| TephraKV-ROADMAP-001 v3.0 | §4 (timeline), §9 (метрики) |
| TephraKV-BACKLOG-001 v5.0 | §2–§9 (задачи по версиям) |
| TephraKV-GLOSSARY-001 v2.0 | Термины |
| ADR-032 | Path to revenue |
| ADR-034 v2.0 | Engineering practices |
| ADR-035 v2.0 | Capacity model |

---

## 13. Что дальше

1. **Утвердить PRICING-001 v1.0** → статус `Approved`.
2. **Написать COMPETITIVE-001** — детальный конкурентный анализ.
3. **Начать 20 интервью с ICP** — Van Westendorp Price Sensitivity Meter (4 вопроса). 
4. **Запустить pricing-тесты** после v0.3 (когда есть design partners).
5. **Начать CHORE-001** — bootstrap.

**Правило:** pricing меняется при каждом релизе + по результатам интервью.

---

**Конец TephraKV-PRICING-001 v1.0**

> **Замечание Тэфри:** PRICING-001 v1.0 — ценовая модель TephraKV. Ключевые решения: open-core (Apache 2.0 ядро + enterprise-фичи); flat-tiered pricing (nodes) для v0.1–v0.5, hybrid для v0.6+; 5 тиров (Community → Enterprise); Value Ratio 2–13× по сегментам; CAC $5–20K; LTV $100K–$8M; LTV/CAC 10–200×; gross margin 85%. Конкурентный анализ: Redis ($10K–$150K/год), Aerospike ($149K median), CockroachDB ($25K–$200K+), KrakenD (flat-tiered, $13K/год), Supabase ($25–599/мес). Action items: 20 интервью (Van Westendorp); pricing-тесты после v0.3; начать CHORE-001.

**Следующий шаг:**
1. Утвердить PRICING-001 v1.0.
2. Написать COMPETITIVE-001.
3. Начать 20 интервью с ICP (Van Westendorp).
4. Начать CHORE-001 — bootstrap.