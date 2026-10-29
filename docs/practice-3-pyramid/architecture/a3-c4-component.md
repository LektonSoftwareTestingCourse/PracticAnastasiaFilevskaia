# A3. C4 Component — Authorization и Card-Management

Сплошная стрелка — синхронный вызов.
Пунктир — асинхронный (через брокер).

## 3.1 Authorization

```mermaid
%%{init: {'theme':'neutral','themeVariables':{'fontSize':'15px'},'flowchart':{'htmlLabels':true,'nodeSpacing':50,'rankSpacing':60}}}%%
flowchart TB
    SW["switch (внешний вызов)"]
    subgraph AUTH["authorization"]
        C["AuthControllerImpl<br/>POST /api/internal/authorize<br/>POST /api/internal/rollback"]
        S["AuthServiceImpl<br/>порядок проверок, RRN, authCode"]
        CMC["CardManagementClientImpl<br/>getCard / reserve / rollback"]
        BLC["BinLookupClient<br/>(только лог)"]
        LUR["LimitUsageRepository<br/>native SQL upsert<br/>дневной/месячный лимит"]
        DO["DeclineOutcome<br/>причина → код ответа"]
    end
    CMS["card-management"]
    BIN["bin-lookup"]
    PG[("PostgreSQL<br/>limit_usage, rrn_seq")]

    SW --> C --> S
    S --> CMC --> CMS
    S --> BLC --> BIN
    S --> LUR --> PG
    S --> DO
```

**Порядок проверок в `AuthServiceImpl.authorize`** (определяет, на каком уровне что тестировать):
`getCard` → Bin Lookup (только лог) → статус карты → срок действия → баланс → `upsertLimitUsage` (суточный/месячный лимит) → RRN из `rrn_seq` (формат `%1d%03d%08d`) → `reserve` в Card-Management → authCode → `approved`.

| Утверждение | Код |
|---|---|
| Контроллер переводит причину отказа в HTTP-статус (CARD_NOT_FOUND→404, INSUFFICIENT_FUNDS→422, EXPIRED/BLOCKED/INACTIVE→403, SERVICE_UNAVAILABLE/RESERVATION_FAILED→503, прочее→400) | [`AuthControllerImpl`](../../../services/authorization/src/main/java/com/processing/authorization/controller/AuthControllerImpl.java) |
| Порядок проверок и вызовы клиента | [`AuthServiceImpl`](../../../services/authorization/src/main/java/com/processing/authorization/services/AuthServiceImpl.java) |
| Лимиты считаются атомарно в SQL (`INSERT … ON CONFLICT … DO UPDATE` с проверкой дневного/месячного) | [`LimitUsageRepository`](../../../services/authorization/src/main/java/com/processing/authorization/repositories/LimitUsageRepository.java) |
| Коды ответа (`05/14/51/54/61/96`) | [`DeclineOutcome`](../../../services/authorization/src/main/java/com/processing/authorization/constants/DeclineOutcome.java) |
| Card-Management вызывается по REST: `GET /api/cards/{pan}`, `POST …/reserve`, `POST …/rollback` | [`CardManagementClientImpl`](../../../services/authorization/src/main/java/com/processing/authorization/client/CardManagementClientImpl.java) |
| Bin Lookup: сбой не ломает авторизацию | [`BinLookupClient`](../../../services/authorization/src/main/java/com/processing/authorization/client/BinLookupClient.java) |

## 3.2 Card-Management

```mermaid
%%{init: {'theme':'neutral','themeVariables':{'fontSize':'15px'},'flowchart':{'htmlLabels':true,'nodeSpacing':50,'rankSpacing':60}}}%%
flowchart TB
    CLIENT["gateway / authorization (внешний вызов)"]
    subgraph CMS["card-management"]
        CC["CardController<br/>/api/cards, /{pan}, /reserve, /rollback, /bulk-update"]
        GEH["GlobalExceptionHandler<br/>исключение → HTTP 400/402/404/409/413"]
        SVC["CardServiceImpl<br/>createCard, patchCard, deleteCard,<br/>reserve, rollback, bulkUpdateStatus"]
        MODEL["Card (record)<br/>startReservation, withData, deleted,<br/>withReservation, withRollback"]
        BIS["BinIssuerService<br/>BIN → issuerId"]
        REPO["CardRepository / ReservationRepository<br/>(JPA, SELECT … FOR UPDATE)"]
        EVN["CardEventNotifier"]
        OBX["Outbox (таблица)"]
        PROC["OutboxEventProcessorImpl<br/>каждые 1000 мс, retry ≤ 3"]
    end
    PG[("PostgreSQL")]
    MQ{{"RabbitMQ<br/>smp.card-events"}}

    CLIENT --> CC --> SVC
    CC -.-> GEH
    SVC --> MODEL
    SVC --> BIS
    SVC --> REPO --> PG
    SVC --> EVN --> OBX --> PG
    PROC --> OBX
    PROC -. "AMQP card.&lt;Event&gt;" .-> MQ
```

| Утверждение | Код |
|---|---|
| Эндпоинты `/api/cards` (POST, GET с `limit/offset/status/bin/issuerId/startDate/endDate`, GET/PATCH/DELETE `/{pan}`, `/reserve`, `/rollback`, `/bulk-update`) | [`CardController`](../../../services/card-management/src/main/java/com/processing/cardmanagement/controllers/CardController.java) |
| Бизнес-правила: резерв только для ACTIVE и при достаточном балансе; отрицательные лимиты и «месячный < дневного» → `IllegalArgumentException` | [`Card`](../../../services/card-management/src/main/java/com/processing/cardmanagement/models/Card.java) |
| Резерв идемпотентен по паре (rrn, pan), строки карты блокируются `FOR UPDATE` | [`CardServiceImpl.reserve/rollback`](../../../services/card-management/src/main/java/com/processing/cardmanagement/services/CardServiceImpl.java) |
| Событие кладётся в outbox вместе с изменением карты, отправляется отдельным процессом | [`CardEventNotifierImpl`](../../../services/card-management/src/main/java/com/processing/cardmanagement/events/CardEventNotifierImpl.java), [`OutboxEventProcessorImpl`](../../../services/card-management/src/main/java/com/processing/cardmanagement/services/OutboxEventProcessorImpl.java) |
| Состояния outbox: PENDING / PROCESSED / FAILED; FAILED после `retryCount >= max-retry-count (3)` | [`OutboxEventDataStatus`](../../../services/card-management/src/main/java/com/processing/cardmanagement/models/OutboxEventDataStatus.java) |
| Сопоставление исключений и HTTP-статусов | [`GlobalExceptionHandler`](../../../services/card-management/src/main/java/com/processing/cardmanagement/configuration/GlobalExceptionHandler.java) |

## Чего не хватало в проекте и что сделано

- Нигде не было описано **порядка проверок** в авторизации — а он определяет приоритет отказов (например, заблокированная карта с нулевым балансом получит `05`, а не `51`). Порядок восстановлен по коду и вынесен в диаграмму; от него зависят ожидаемые результаты тест-кейсов.
- Нигде не сказано, что проверка «дневной лимит ≤ месячный» есть только в `Card.withData` (PATCH и bulk-update), а при создании карты (`createCard` → `Card.fromDraft`) её нет. Это пробел в валидации, обнаруженный при чтении кода: тест-кейс на него — в `test-cases.md`.
- Логика лимитов вынесена в SQL: проверить её юнит-тестом с моком нельзя, нужен реальный PostgreSQL (интеграционный уровень).
