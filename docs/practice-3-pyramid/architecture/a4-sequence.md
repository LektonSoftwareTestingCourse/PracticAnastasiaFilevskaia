# A4. Sequence — три сценария

`->>` — синхронный вызов (HTTP).
`-)` — асинхронная отправка (AMQP).

## Сценарий 1. Успешная авторизация (happy path)

```mermaid
%%{init: {'theme':'neutral','themeVariables':{'fontSize':'15px'},'flowchart':{'htmlLabels':true,'nodeSpacing':50,'rankSpacing':60}}}%%
sequenceDiagram
    autonumber
    participant T as Terminal
    participant G as Gateway
    participant S as Switch
    participant A as Authorization
    participant C as Card-Management
    participant M as Merchant-Acquirer
    participant Q as RabbitMQ
    participant L as Transaction-Logger
    participant D as Dashboard

    T->>G: POST /api/transactions (mti 0100)
    G->>G: валидация TransactionRequestValidator
    G->>S: POST /api/internal/route
    S->>S: RoutingService: BIN(6) → issuerId
    S->>A: POST /api/internal/authorize
    A->>C: GET /api/cards/{pan}
    C-->>A: карта (ACTIVE, баланс, срок)
    A->>A: статус → срок → баланс → upsert лимитов → RRN
    A->>C: POST /api/cards/{pan}/reserve
    C-->>A: 200 (резерв создан)
    A-->>S: 200 APPROVED (rrn, authCode, "00")
    S->>M: POST /api/simulator/merchant/fee
    M-->>S: комиссия
    S-)Q: publish smp.transactions / transaction.log
    S-->>G: 200 APPROVED
    G-->>T: 200 APPROVED
    Q-)L: queue transaction-log (позже, асинхронно)
    L->>L: TransactionService.store (идемпотентно по id)
    L-)D: WebSocket /ws/transactions
```

## Сценарий 2. Отказ: карта BLOCKED

```mermaid
%%{init: {'theme':'neutral','themeVariables':{'fontSize':'15px'},'flowchart':{'htmlLabels':true,'nodeSpacing':50,'rankSpacing':60}}}%%
sequenceDiagram
    autonumber
    participant T as Terminal
    participant G as Gateway
    participant S as Switch
    participant A as Authorization
    participant C as Card-Management
    participant Q as RabbitMQ
    participant L as Transaction-Logger

    T->>G: POST /api/transactions
    G->>S: POST /api/internal/route
    S->>A: POST /api/internal/authorize
    A->>C: GET /api/cards/{pan}
    C-->>A: карта, status = BLOCKED
    A->>A: DeclineOutcome.CARD_BLOCKED (код "05"), reserve НЕ вызывается
    A-->>S: HTTP 403, тело: DECLINED, "05"
    Note over S: клиент игнорирует не-2xx статус и читает тело (onStatus → пусто)
    S-)Q: publish (запись DECLINED тоже журналируется)
    S-->>G: 200 DECLINED, код "05"
    G-->>T: 200 DECLINED
    Q-)L: сохранить DECLINED-транзакцию
```

## Сценарий 3. Откат резерва (RabbitMQ недоступен)

```mermaid
%%{init: {'theme':'neutral','themeVariables':{'fontSize':'15px'},'flowchart':{'htmlLabels':true,'nodeSpacing':50,'rankSpacing':60}}}%%
sequenceDiagram
    autonumber
    participant G as Gateway
    participant S as Switch
    participant A as Authorization
    participant C as Card-Management
    participant Q as RabbitMQ

    G->>S: POST /api/internal/route
    S->>A: authorize
    A->>C: reserve
    C-->>A: 200
    A-->>S: 200 APPROVED (rrn)
    S-xQ: convertAndSend → AmqpException (брокер недоступен)
    Note over S: tryLog() вернул false, а статус APPROVED
    S->>A: POST /api/internal/rollback (rrn, pan, amount)
    A->>C: POST /api/cards/{pan}/rollback
    C-->>A: резерв снят, баланс возвращён
    A-->>S: 200 rollback APPROVED
    S-->>G: 200 DECLINED, код "96" (systemError)
```

| Утверждение | Код |
|---|---|
| Валидация на входе | [`TransactionRequestValidator`](../../../services/gateway/src/main/java/com/processing/gateway/validation/TransactionRequestValidator.java) |
| Оркестрация: BIN → authorize → fee → журнал → откат при `!logged && APPROVED` | [`RouteService`](../../../services/switch/src/main/java/com/processing/service/RouteService.java) |
| Не-2xx ответы Authorization не бросают исключение, тело читается (`onStatus(... {})`) | [`AuthorizationClient`](../../../services/switch/src/main/java/com/processing/service/AuthorizationClient.java) |
| Статус 403 для BLOCKED, код `05` | [`AuthControllerImpl`](../../../services/authorization/src/main/java/com/processing/authorization/controller/AuthControllerImpl.java), [`DeclineOutcome`](../../../services/authorization/src/main/java/com/processing/authorization/constants/DeclineOutcome.java) |
| Резерв и откат в Card-Management | [`CardServiceImpl.reserve/rollback`](../../../services/card-management/src/main/java/com/processing/cardmanagement/services/CardServiceImpl.java) |
| Публикация в брокер | [`LoggerClient`](../../../services/switch/src/main/java/com/processing/service/LoggerClient.java) |
| Приём и запись; WebSocket | [`TransactionLogListener`](../../../services/transaction-logger/src/main/java/com/processing/transactionlogger/listener/TransactionLogListener.java), [`TransactionService`](../../../services/transaction-logger/src/main/java/com/processing/transactionlogger/service/TransactionService.java), [`WebSocketManager`](../../../services/transaction-logger/src/main/java/com/processing/transactionlogger/websocket/WebSocketManager.java) |
| Существующий e2e-тест на откат (TC-23) | [`RabbitMQAsyncE2eTest`](../../../services/e2e-tests/src/test/java/com/processing/e2e/tests/RabbitMQAsyncE2eTest.java) |

## Чего не хватало в проекте и что сделано

- **Расхождение документации и кода (главное).** В [`docs/architecture.md`](../../../docs/architecture.md) сказано, что Switch ждёт publisher-confirm до 2 секунд и при его отсутствии шлёт откат сообщением `mti 0400`. В коде этого нет: `LoggerClient` вызывает обычный `rabbitTemplate.convertAndSend`, `false` возвращается только при `AmqpException`; ожидания confirm нет; откат идёт через REST `/api/internal/rollback`. Фабрика `AuthorizationRequest.forReversal` (mti `0400`) нигде не вызывается. На диаграмме нарисовано **то, что делает код**.
- Следствие: если брокер принял сообщение, но не подтвердил (или потерял) его, а исключения нет — резерв **не** откатится, а транзакция не попадёт в журнал. Этот риск закрывается только интеграционным тестом с реальным брокером; в тест-кейсах он описан отдельно.
- В документации не сказано, что отказы (`DECLINED`) тоже журналируются и что неизвестный BIN (код `14`) и недоступность Authorization (код `05`) возвращаются как `200 DECLINED` из Switch, а не как HTTP-ошибки.
