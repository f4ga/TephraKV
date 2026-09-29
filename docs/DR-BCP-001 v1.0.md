# TephraKV — Disaster Recovery & Business Continuity Plan

**Документ:** TephraKV-DR-BCP-001
**Версия:** 1.0
**Статус:** Черновик
**Дата:** 29 сентября 2026
**Связь:** TephraKV-HLD-000 v11.3, NFR-001 v1.0, BACKLOG-001 v6.1, ROADMAP-001 v4.0, API-001 v1.3, FORMAT-001 v1.3, ADR-010 v3, ADR-012 v6, ADR-022, ADR-026 v5
**Классификация:** Внутренний / публичный
**Стандарты:** ISO 22301 (Business Continuity), NIST SP 800-34 (Contingency Planning), BSI C5 (SIM-05/06)

---

## 0. Назначение и область применения

Настоящий документ определяет стратегии и процедуры для поддержания непрерывности работы (Business Continuity) и восстановления после катастрофических сбоев (Disaster Recovery) для TephraKV — дисковой KV-СУБД на Go с тремя режимами развёртывания: embedded, server, distributed.

### 0.1. Область применения

| Компонент | Покрытие | Приоритет |
|---|---|---|
| TephraKV Core (embedded) | Да | Критический |
| TephraKV Server (gRPC/REST) | Да | Критический |
| TephraKV Distributed (Dragonboat) | Да | Критический |
| WAL shared pool | Да | Критический |
| LSM / SSTable / Manifest | Да | Критический |
| VLog (WiscKey, v0.3+) | Да | Высокий |
| Dragonboat / Tan engine | Да | Высокий |
| Audit log (v0.5+) | Да | Высокий |
| PITR (v0.5+) | Да | Средний |
| Client SDK | Нет | Низкий |
| Документация | Нет | Низкий |

### 0.2. Аудитория

Документ предназначен для: SRE / DevOps, отвечающих за эксплуатацию TephraKV; инженеров, обеспечивающих восстановление после сбоев; product-менеджеров, оценивающих влияние сбоев на бизнес; аудиторов, проверяющих соответствие требованиям.

---

## 1. Business Impact Analysis (BIA)

### 1.1. Критические бизнес-процессы

| Процесс | Описание | RTO | RPO | Приоритет |
|---|---|---|---|---|
| KV-операции (embedded) | Put/Get/Delete/Scan | 5 мин | 0 (SYNC_MASTER) / N мс (NO_SYNC) | P1 |
| Server mode (gRPC) | Клиентские запросы по сети | 15 мин | 0 (SYNC_MASTER) | P1 |
| Distributed KV | Multi-Raft консенсус | 30 мин | 0 (SYNC_MAJORITY) | P1 |
| Durability / WAL | fsync, group commit | 5 мин | 0 | P1 |
| Compaction | Read-amp поддержание | 4 ч | 15 мин | P2 |
| VLog GC (v0.3+) | Фрагментация, space amp | 4 ч | 30 мин | P2 |
| PITR (v0.5+) | Point-in-time recovery | 2 ч | 0 (WAL archive) | P2 |
| Audit log (v0.5+) | Compliance-запись | 2 ч | 0 | P1 |
| Monitoring / Metrics | Наблюдаемость | 4 ч | 1 ч | P3 |

### 1.2. Recovery Time Objective (RTO)

RTO — максимально допустимое время восстановления сервиса после сбоя. Определения соответствуют стандартной практике.

| Уровень | RTO | Описание | Компоненты |
|---|---|---|---|
| Tier 1 | < 5 мин | Критические | Embedded KV, WAL, durability |
| Tier 2 | < 30 мин | Важные | Server mode, distributed, audit |
| Tier 3 | < 4 ч | Поддерживающие | Compaction, VLog GC, PITR |
| Tier 4 | < 24 ч | Некритические | Monitoring, docs |

### 1.3. Recovery Point Objective (RPO)

RPO — максимально допустимый объём потери данных, измеряемый во времени.

| Уровень | RPO | Допустимая потеря | Режим / политика |
|---|---|---|---|
| Tier 1 | 0 | Нет потерь | SYNC_MASTER (single-node), SYNC_MAJORITY (distributed) |
| Tier 2 | 5 с | Последние 5 секунд | Group commit (SYNC_MASTER с батчингом) |
| Tier 3 | 100 мкс – 1 мс | Окно group commit | NO_SYNC (group commit window) |
| Tier 4 | 15 мин | Конфигурация, метрики | Best-effort |

**Специфика TephraKV.** RPO зависит от выбранной политики durability (ADR-010): `NO_SYNC` даёт RPO = окно group commit (100 мкс – 1 мс); `SYNC_MASTER` даёт RPO = 0; `SYNC_MAJORITY` в distributed даёт RPO = 0 при отказе меньшинства.

### 1.4. Service Level Targets

| Метрика | Цель | Измерение |
|---|---|---|
| Annual availability (single-node) | 99.9% | ~8.8 ч простоя/год |
| Annual availability (distributed, 5 nodes) | 99.99% | ~52 мин простоя/год |
| MTTD (Mean Time to Detect) | < 2 мин | Monitoring alerts |
| MTTR (S1: data loss) | < 8 ч | Incident resolution |
| MTTR (S2: downtime) | < 24 ч | Incident resolution |

### 1.5. Maximum Tolerable Downtime (MTD)

| Система | MTD | Последствие превышения |
|---|---|---|
| Embedded KV | 4 ч | Остановка приложения |
| Server mode | 4 ч | Потеря клиентских запросов |
| Distributed KV | 2 ч | Потеря кворума, деградация |
| Audit log | 8 ч | Compliance violation |

---

## 2. Disaster Classification

### 2.1. Уровни серьёзности

| Уровень | Название | Определение | Время реакции | Примеры |
|---|---|---|---|---|
| SEV-1 | Critical | Полная остановка, риск потери данных | < 15 мин | Corruption WAL/SSTable, потеря кворума |
| SEV-2 | High | Серьёзная деградация | < 30 мин | Single-node down, degraded mode |
| SEV-3 | Medium | Частичная деградация | < 2 ч | Медленные компакции, высокая latency |
| SEV-4 | Low | Незначительные проблемы | < 24 ч | Метрики, не критично |

### 2.2. Категории сбоев

| Категория | Описание | Первичная реакция |
|---|---|---|
| L1 | Отказ одного компонента | Auto-recovery, restart |
| L2 | Отказ нескольких компонентов | Manual intervention |
| L3 | Отказ зоны / региона | Failover на standby |
| L4 | Полный отказ кластера | Full DR activation |
| L5 | Corruption / потеря данных | PITR, restore from backup |
| L6 | Security incident | Containment, forensics |

---

## 3. Backup Strategy

### 3.1. Типы резервных копий

| Тип | Частота | Метод | RPO | Retention |
|---|---|---|---|---|
| Continuous replication | Real-time | Raft log replication (distributed) | < 1 с | — |
| WAL archiving | Continuous | WAL segment archive | 0 (SYNC_MASTER) | 30 дней |
| Incremental snapshot | Каждые 15 мин | SSTable + WAL archive | 15 мин | 24 ч |
| Full backup | Ежедневно, 02:00 UTC | Копия директории БД | 24 ч | 30 дней |
| Long-term archive | Еженедельно | Compressed full | 7 дней | 1 год |

### 3.2. Хранилище резервных копий

| Локация | Провайдер | Регион | Шифрование | Lifecycle |
|---|---|---|---|---|
| Primary | S3-compatible | Основной + 1 | AES-256-GCM | Auto-retention |
| Secondary | S3-compatible | Гео-резервный | AES-256-GCM | Mirror from primary |
| Cold archive | Glacier | — | AES-256-GCM | 7 лет |

### 3.3. Верификация резервных копий

После каждой резервной копии выполняется автоматическая верификация: проверка целостности (CRC32C), проверка консистентности (Manifest + SSTable), sample-restore в изолированное окружение. Верификация интегрирована в CI/BENCH-007 (Crash-Recovery Harness).

### 3.4. Backup verification (для TephraKV)

```bash
# Полная резервная копия директории БД
tar -czf tephrakv-backup-$(date +%Y%m%d).tar.gz /var/lib/tephrakv

# Верификация: recovery в изолированное окружение
tephrakv-cli open --dir /tmp/restore-test --verify

# Проверка: все SYNC_MASTER записи восстановлены
tephrakv-cli verify --expected-count <N>
```

---

## 4. Recovery Procedures

### 4.1. Процедура восстановления: single-node (embedded)

**Шаги:**

1. **Обнаружение** (0–2 мин). Monitoring алерт или ручное сообщение.
2. **Классификация** (2–5 мин). Определить категорию: L1–L6.
3. **Изоляция** (5–10 мин). Остановить приложение, чтобы избежать дальнейшей записи.
4. **Восстановление** (10 мин – 4 ч):
   - **L1 (crash):** `Open()` выполняет recovery (WAL replay + SSTable load). Ожидаемое время: < 5 с на 10 ГБ.
   - **L5 (corruption):** Restore from backup + WAL replay до последнего валидного LSN. PITR на момент до corruption.
5. **Верификация** (после восстановления): проверка CRC, проверка count записей, sample Get.
6. **Возврат в эксплуатацию** (после верификации).

**Критерии успеха:** все `SYNC_MASTER` записи восстановлены; recovery time < RTO; данные не повреждены (CRC pass).

### 4.2. Процедура восстановления: server mode (gRPC)

**Шаги:**

1. **Обнаружение.** gRPC health check fail.
2. **Классификация.** Server down (L1–L2), network issue (L2), data corruption (L5).
3. **Failover.** Переключение на standby instance (если есть).
4. **Восстановление.** Restart server с recovery.
5. **Клиенты.** Reconnect, retry.

**Критерии успеха:** p99 GET < 10 мс после восстановления; все клиенты подключены.

### 4.3. Процедура восстановления: distributed (Dragonboat)

**Шаги:**

1. **Обнаружение.** Raft leader election fail, потеря кворума.
2. **Классификация.** Single-node failure (L1), leader failure (L2), partition (L3), cluster failure (L4).
3. **Failover.** Raft leader election (p99 < 3 с). ReadIndex protocol.
4. **Восстановление.** Замена ноды, snapshot transfer с resume.
5. **Верификация.** Freshness p99 < 1 с; leader failover p99 < 3 с.

**Критерии успеха:** кластер восстановил кворум; `SYNC_MAJORITY` записи не потеряны; snapshot transfer завершён.

### 4.4. Процедура восстановления: corruption / потеря данных (L5)

**Шаги:**

1. **Остановка.** Остановить все записи в БД.
2. **Определение точки восстановления.** PITR: выбрать timestamp до corruption.
3. **Restore.** Восстановить из full backup.
4. **WAL replay.** Replay WAL archive до выбранного timestamp.
5. **Верификация.** Проверка CRC, проверка целостности.
6. **Возврат.** Возобновить операции.

**Критерии успеха:** RTO < 4 ч; RPO = 0 (если WAL archive полный).

---

## 5. Disaster Scenarios

| Сценарий | Категория | RTO (цель) | RPO (цель) | Процедура |
|---|---|---|---|---|
| Process crash (kill -9) | L1 | < 5 с | 0 (SYNC_MASTER) | Auto recovery на `Open()` |
| OOM kill | L1 | < 5 с | 0 (SYNC_MASTER) | Auto recovery |
| Disk full | L2 | < 15 мин | 0 | Backpressure, cleanup, expand |
| Partial write / CRC mismatch | L2 | < 5 мин | 0 (до последнего fsync) | WAL replay stop на corruption |
| SSTable corruption | L2 | < 1 ч | 0 | Restore from backup |
| Manifest corruption | L2 | < 1 ч | 0 | Restore from backup |
| WAL corruption | L2 | < 1 ч | 0 | Restore from backup |
| Single-node failure (distributed) | L2 | < 30 мин | 0 (SYNC_MAJORITY) | Raft leader election |
| Leader failure (distributed) | L2 | < 3 с | 0 (SYNC_MAJORITY) | Raft election |
| Network partition | L3 | < 5 мин | 0 (кворум) | ReadIndex, degraded mode |
| Data center outage | L3 | < 30 мин | 5 с | Failover на standby |
| Regional disaster | L4 | < 2 ч | 1 мин | Full DR activation |
| Complete data loss | L5 | < 4 ч | 15 мин | Restore from backup + WAL replay |
| Ransomware | L6 | < 6 ч | 1 ч | Isolate, restore from immutable backup |

---

## 6. Testing & Validation

### 6.1. Типы тестов

| Тип | Частота | Что проверяет | Артефакт |
|---|---|---|---|
| Crash-recovery harness | CI (каждый PR) | 10000/10000 recovery | BENCH-007 |
| Backup verification | После каждой backup | Integrity, consistency | Auto script |
| DR drill (tabletop) | Квартально | Процедуры, роли, коммуникация | Отчёт |
| DR drill (full failover) | Полугодие | Реальный failover | Отчёт, метрики RTO/RPO |
| Chaos testing | Ежеквартально | Поведение под нагрузкой | BENCH-013 |

### 6.2. Crash-recovery harness

Выполняется 10000 итераций с `kill -9` в случайный момент. Проверяет: все подтверждённые записи восстановлены, нет corruption, recovery детерминирован. Артефакт — BENCH-007.

### 6.3. DR drill

**Tabletop:** обсуждение сценариев, проверка контактов, ролей, процедур. Без реального восстановления.

**Full failover:** реальный failover на standby, измерение RTO/RPO. После — разбор, обновление runbooks.

### 6.4. Backup verification

Ежедневно после backup: проверка целостности, sample-restore в изолированное окружение, проверка count записей.

---

## 7. Communication Plan

### 7.1. Escalation path

| Уровень | Роль | Контакт | Время реакции |
|---|---|---|---|
| Primary | On-call Engineer | PagerDuty | < 15 мин (SEV-1) |
| Secondary | Engineering Lead | Phone | < 30 мин |
| Tertiary | VP Engineering | Phone | < 1 ч |
| Executive | CTO / CEO | Phone | SEV-1 only |

### 7.2. Уведомления по уровням

| Уровень | Кого уведомлять | Метод |
|---|---|---|
| SEV-1 | Все stakeholders, customers | Status page, email |
| SEV-2 | Ops, business leads | Slack, email |
| SEV-3 | Ops | Slack |
| SEV-4 | Ops | Log |

### 7.3. Status page

Внешний status page обновляется при SEV-1/SEV-2. Внутренний — в Slack #incidents.

---

## 8. Roles & Responsibilities

| Роль | Ответственность |
|---|---|
| Incident Commander | Координация восстановления, принятие решений |
| SRE / DevOps | Выполнение процедур восстановления |
| Engineering Lead | Техническая экспертиза, эскалация |
| Product Manager | Оценка бизнес-влияния, коммуникация с клиентами |
| CTO | Executive decisions, budget |

---

## 9. Постмортем

После каждого SEV-1/SEV-2:

1. Timeline восстановления.
2. Root cause analysis.
3. Что сработало, что нет.
4. Action items с owner и deadline.
5. Обновление runbooks.

Документ хранится в `docs/incidents/`.

---

## 10. Связанные документы

| Документ | Тема |
|---|---|
| TephraKV-HLD-000 v11.3 | Архитектура, failure model |
| TephraKV-NFR-001 v1.0 | RTO/RPO, availability, durability |
| TephraKV-BACKLOG-001 v6.1 | BENCH-007, BENCH-013 |
| TephraKV-FORMAT-001 v1.3 | WAL, SSTable, Manifest formats |
| ADR-010 v3 | Durability semantics |
| ADR-012 v6 | Shared WAL pool |
| ADR-022 | VLog |
| ADR-026 v5 | Multi-Raft, Tan engine |
| ISO 22301 | Business Continuity |
| NIST SP 800-34 | Contingency Planning |

---

## 11. Что дальше

1. Утвердить DR-BCP-001 v1.0 → статус `Approved`.
2. Заполнить контакты и конкретные роли.
3. Настроить backup scripts и verification в CI.
4. Провести первый tabletop drill.
5. Обновлять при каждом релизе (новые компоненты: VLog, MVCC, distributed).
6. Интегрировать с BENCH-007 (crash-recovery) и BENCH-013 (Raft under partition).

---

**Конец документа TephraKV-DR-BCP-001 v1.0**