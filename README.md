# Go-DDD: Domain Driven Design, SAGA, Event-Sourcing & CQRS

## Эволюция архитектуры распределенных систем

### 1. Введение в проблематику

Распределенные транзакции в микросервисной архитектуре требуют особых подходов. 
Рассмотрим эволюцию решения от базовой SAGA до комплексной Event-Driven архитектуры.

### 2. Базовая реализация SAGA

#### 2.1. Архитектурный обзор

<img src="./static/diagrams/saga_0.svg" alt="saga_0_overview">

```graph
graph TD
    A[Client] --> B[Order Service]
    B --> C[Catalog Service]
    B --> D[Payment Service]
```

##### 2.2. Диаграмма последовательности SAGA

<img src="./static/diagrams/saga_1.svg" alt="saga_first_overview">

```sequence
sequenceDiagram
    participant Client
    participant OrderService
    participant CatalogService
    participant PaymentService

    Client->>OrderService: Создать заказ (POST /orders)
    OrderService->>CatalogService: Резервировать товары (PUT /catalog/reserve)
    CatalogService-->>OrderService: Товары зарезервированы
    OrderService->>PaymentService: Списать средства (POST /payment/charge)
    PaymentService-->>OrderService: Платеж успешен
    OrderService->>Client: Заказ подтвержден
```

#### 2.3. Критические компоненты

##### 1. Order Service

Роль: Оркестратор SAGA, управляет процессом заказа.

**Пример кода:**

```go
func (s *OrderService) CreateOrder() error {
    // 1. Резервируем товары
    if err := s.catalog.ReserveItems(); err != nil {
        return err
    }
    
    // 2. Списываем средства
    if err := s.payment.Charge(); err != nil {
        s.catalog.CancelReservation() // Компенсационная транзакция
        return err
    }
    
    return nil
}
```

##### 2. Catalog Service

Роль: Управляет остатками товаров.

##### 3. Payment Service

Роль: Обрабатывает платежи.

#### 2.4. Диаграмма последовательнсти

<img src="./static/diagrams/saga_2.svg" alt="saga_second_overview">

```sequence
sequenceDiagram
    participant OrderService
    participant Kafka
    participant CatalogService
    participant PaymentService

    OrderService->>Kafka: OrderCreated (SAGA_START)
    Kafka->>CatalogService: ReserveItems
    CatalogService->>Kafka: ItemsReserved / ReserveFailed
    Kafka->>PaymentService: ChargePayment (если ReserveSuccess)
    PaymentService->>Kafka: PaymentProcessed / PaymentFailed
    Kafka->>OrderService: SAGA_COMPLETE / SAGA_ROLLBACK
```

#### Проблемы базовой реализации:

- Нет защиты от **повторных запросов**
- **Каскадные ошибки** при недоступности сервисов
- **Потеря событий** при сбоях
- **Отсутствие истории** изменений

#### Что можно улучшить

- Добавить Idempotency Key для избежания дублирования операций.
- Внедрить Circuit Breaker для устойчивости к ошибкам.
- Использовать Outbox Pattern для надежной доставки событий.

#### Вывод

- ✅ SAGA решает проблему распределенных транзакций
- ✅ Каждый сервис управляет своей частью данных
- ✅ Компенсационные транзакции обеспечивают согласованность
- ⚠ Не подходит для высоконагруженных систем (лучше Event Sourcing + CQRS)

### Глосарий

> ##### Компенсационные транзакции (SAGA Rollback)
> Если на любом этапе происходит ошибка, система выполняет обратные операции в обратном порядке:
> 1. **Платеж не прошел** → Отмена резерва товаров (CatalogService.CancelReservation).
> 2. **Резерв не удался** → Заказ отклоняется без списания денег.


### 3. Улучшенная Event-Driven SAGA

<img src="./static/diagrams/saga_6.svg" alt="saga_6_overview">

```graph
graph LR
    A[Order] -->|Events| B[Kafka]
    B --> C[Catalog]
    B --> D[Payment]
    C --> B
    D --> B
```

> Можно реализовать через Kafka или RabbitMQ:

<img src="./static/diagrams/saga_3.svg" alt="saga_third_overview">

```sequence
sequenceDiagram
    participant Client
    participant OrderService
    participant CatalogService
    participant PaymentService
    participant Kafka
    participant OutboxTable

    Client->>OrderService: POST /orders (idempotency-key: XYZ123)
    OrderService->>OutboxTable: Сохранить OrderCreatedEvent
    OrderService->>Kafka: OrderCreated (SAGA_START)
    Kafka->>CatalogService: ReserveItems (через CB)
    CatalogService->>OutboxTable: Сохранить ItemsReservedEvent
    CatalogService->>Kafka: ItemsReserved
    Kafka->>PaymentService: ChargePayment (через CB)
    PaymentService->>OutboxTable: Сохранить PaymentProcessedEvent
    PaymentService->>Kafka: PaymentProcessed
    Kafka->>OrderService: SAGA_COMPLETE
```

#### 3.1. Idempotency Key (Идемпотентность)

**Задача:** Предотвращение дублирования операций при повторных запросах.
**Пример кода:**

```go
// Middleware для проверки ключа идемпотентности
func IdempotencyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        key := r.Header.Get("Idempotency-Key")
        if cached := cache.Get(key); cached != nil {
            respondWithCached(w, cached)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

#### 3.2. Circuit Breaker (Автоматический переключатель)

**Задача:** Защита от каскадных ошибок при недоступности сервисов.
**Пример кода:**

```go
// Настройка для Catalog Service
cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:    "CatalogService",
    Timeout: 30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})
```

#### 3.3. Outbox Pattern (Надежная доставка событий)

**Задача:** Гарантированная доставка событий даже при падении сервиса.
**Пример кода:**

```go
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    saga_id VARCHAR(255),
    event_type VARCHAR(100),
    payload JSONB,
    created_at TIMESTAMP,
    processed BOOLEAN DEFAULT false
);
```

#### 3.4. Архитектурная диаграмма

<img src="./static/diagrams/saga_4.svg" alt="saga_4_overview">

```graph
graph TD
    A[Client] --> B[Order Service]
    B --> C[Catalog Service]
    B --> D[Payment Service]
    B --> E[(Outbox DB)]
    C --> F[(Catalog DB)]
    D --> G[(Payment DB)]
    B --> H[Kafka]
    C --> H
    D --> H
    H --> I[SAGA Orchestrator]
    I --> B
```

#### 3.5. Диаграмма последовательнсти

<img src="./static/diagrams/saga_5.svg" alt="saga_5_overview">

```sequence
sequenceDiagram
    participant Client
    participant OrderService
    participant CatalogService
    participant PaymentService
    participant Kafka
    participant OutboxProcessor

    Client->>OrderService: POST /orders
    OrderService->>OrderService: Генерирует Idempotency Key
    OrderService->>OrderService: Сохраняет заказ в БД
    OrderService->>OutboxProcessor: OrderCreated (Outbox)
    OutboxProcessor->>Kafka: OrderCreated
    Kafka->>SagaOrchestrator: OrderCreated
    SagaOrchestrator->>CatalogService: ReserveItems
    CatalogService->>CatalogService: Проверяет наличие
    CatalogService->>CatalogService: Резервирует товары
    CatalogService->>OutboxProcessor: ItemsReserved (Outbox)
    OutboxProcessor->>Kafka: ItemsReserved
    Kafka->>SagaOrchestrator: ItemsReserved
    SagaOrchestrator->>PaymentService: ProcessPayment
    PaymentService->>PaymentService: Обрабатывает платеж
    PaymentService->>OutboxProcessor: PaymentProcessed (Outbox)
    OutboxProcessor->>Kafka: PaymentProcessed
    Kafka->>SagaOrchestrator: PaymentProcessed
    SagaOrchestrator->>OrderService: CompleteOrder
    OrderService->>OrderService: Обновляет статус заказа
    OrderService->>Client: 200 OK
```

### 📊 Сравнение подходов

| Подход              | Достоинства | Недостатки  |
|---------------------|---|---|
| **Базовая SAGA**    | Простота реализации  | Нет защиты от дублирования  |
| **С оптимизациями** | Отказоустойчивость, надежность | Сложность возрастает  |

### 4. Переход к CQRS и Event Sourcing

#### 4.1. CQRS: разделение ответственности

##### 4.1.1. Архитектура

<img src="./static/diagrams/cqrs.svg" alt="cqrs_overview">

```graph
graph TB
    A[Command] --> B[Command Handler]
    B --> C[Event Store]
    C --> D[Read Model]
    E[Query] --> F[Read Model]
```

##### 4.1.2. Пример кода

```go
package main

// Command Side
type CreateOrderCommand struct {
    UserID uuid.UUID
    Items  []OrderItem
}

type OrderCommandHandler struct {
    eventStore EventStore
}

func (h *OrderCommandHandler) Handle(cmd CreateOrderCommand) error {
    events := []Event{
        NewEvent("OrderCreated", cmd),
    }
    return h.eventStore.Append(events)
}

// Query Side
type OrderReadModel struct {
    db *sql.DB
}

func (r *OrderReadModel) GetByID(id uuid.UUID) (OrderView, error) {
    // Query from optimized read storage
}
```

#### 4.2.Event Sourcing: принципиально новый подход

##### 4.2.1. Основная концепция

**Состояние системы** = Последовательность событий

```go
type EventStore interface {
    Append(event Event) error
    GetStream(aggregateID string) ([]Event, error)
}

type Event struct {
    ID          uuid.UUID
    Type        string
    AggregateID string
    Data        []byte
    Version     uint64
    Timestamp   time.Time
}
```

##### 4.2.2. Реализация агрегата

```go
type OrderAggregate struct {
    ID      uuid.UUID
    Version uint64
    State   OrderState
}

func (a *OrderAggregate) Apply(event Event) error {
    switch event.Type {
    case "OrderCreated":
        var data OrderCreatedData
        json.Unmarshal(event.Data, &data)
        a.State = OrderState{Status: "created"}
    case "PaymentProcessed":
        a.State.Status = "paid"
    }
    a.Version = event.Version
    return nil
}
```

#### 4.3. Комбинированный подход: Event Sourcing + CQRS

##### 4.3.1. Полная архитектура

<img src="./static/diagrams/es-cqrs.svg" alt="es_cqrs_overview">

```graph
graph LR
    A[Client] --> B[Command]
    B --> C[Command Handler]
    C --> D[Event Store]
    D --> E[Event Processor]
    E --> F[Read Model]
    A --> G[Query]
    G --> F
    E --> H[Projections]
    H --> I[Analytics DB]
```

##### 4.3.2. Пример кода

```go
package main

// todo
```

#### 4.3.3. Диаграмма последовательности

<img src="./static/diagrams/es-cqrs-2.svg" alt="es_cqrs_2_overview">

```sequence
sequenceDiagram
    participant Client
    participant CommandAPI
    participant CommandHandler
    participant EventStore
    participant EventProcessor
    participant ReadModel
    participant QueryAPI

    Client->>CommandAPI: POST /orders (Command)
    CommandAPI->>CommandHandler: CreateOrderCommand
    CommandHandler->>EventStore: Append Events[OrderCreated]
    EventStore-->>CommandHandler: Success
    CommandHandler-->>CommandAPI: 202 Accepted
    CommandAPI-->>Client: 202 Accepted

    loop Async Processing
        EventProcessor->>EventStore: Poll New Events
        EventStore-->>EventProcessor: OrderCreatedEvent
        EventProcessor->>ReadModel: Update Projection
        EventProcessor->>ReadModel: Update MaterializedView
    end

    Client->>QueryAPI: GET /orders/{id} (Query)
    QueryAPI->>ReadModel: GetOrderView
    ReadModel-->>QueryAPI: OrderView
    QueryAPI-->>Client: 200 OK (DTO)
```

#### 4.3.4. Текстовая версия диаграммы:

> 1. Клиент отправляет команду:
>
>    `Client → CommandAPI: POST /orders {items: [...]}`
> 2. Command API перенаправляет команду обработчику:
>
>    `CommandAPI → CommandHandler: CreateOrderCommand`
> 3. Обработчик сохраняет события:
>
>    `CommandHandler → EventStore: Append [OrderCreatedEvent]`
> 4. EventStore подтверждает запись:
>
>    `EventStore → CommandHandler: Success`
> 5. Клиент получает подтверждение:
>
>    `CommandHandler → CommandAPI → Client: 202 Accepted`
> 6. Фоновый процесс обработки событий:
>    - **EventProcessor** периодически опрашивает **EventStore**
>    - Получает новые события (`OrderCreatedEvent`)
>    - Обновляет проекции в **ReadModel**
> 7. Клиент запрашивает данные:
>
>    `Client → QueryAPI: GET /orders/123`
> 8. Query API получает данные из ReadModel:
>
>    `QueryAPI → ReadModel: GetOrderView(123)`
> 9. Данные возвращаются клиенту:
>
>    `ReadModel → QueryAPI → Client: OrderViewDTO`

#### 4.3.5. Ключевые особенности потока:

1. **Разделение путей записи и чтения:**
    - Запись идет через Command Stack (левая часть)
    - Чтение через Query Stack (правая часть)

2. **Асинхронная обработка:**
    - Обновление ReadModel происходит после фиксации событий
    - Задержка между записью и консистентностью чтения (Eventual Consistency)
3. **Компоненты:**
    - `EventStore` - хранилище событий (например, Kafka+PostgreSQL)
    - `EventProcessor` - подписчик на события (Consumer)
    - `ReadModel` - оптимизированное хранилище для чтения (MongoDB/Elasticsearch)

#### 4.3.6. Типовые задержки:

<img src="./static/diagrams/es-cqrs-latency.svg" alt="es_cqrs_3_overview">

```timeline
timeline
    title Временная диаграмма согласованности
    section Запись
    Command : 0 ms
    Event Store : 50 ms
    ReadModel Update : 100-500 ms
    section Чтение
    Query : После 500 ms (strong consistency)
```

### 5. Сравнительный анализ

| Критерий               | SAGA                | Event Sourcing      | CQRS                | ES+CQRS             |
|------------------------|---------------------|---------------------|---------------------|---------------------|
| **Сложность**          | Низкая              | Высокая             | Средняя             | Очень высокая       |
| **Производительность** |                     |                     |                     |                     |
| - Запись               | Средняя             | Низкая              | Средняя             | Средняя             |
| - Чтение               | Средняя             | Низкая              | Очень высокая       | Оптимальная         |
| **Масштабируемость**   | Ограничена          | Отличная            | Отличная            | Наилучшая           |
| **Гибкость**           | Ограничена          | Максимальная        | Высокая             | Максимальная        |
| **История изменений**  | Нет                 | Полная              | Частичная           | Полная              |
| **Согласованность**    | Eventual            | Strong              | Strong              | Strong              |
| **Латентность**        | Низкая              | Высокая             | Низкая (чтение)     | Средняя             |
| **Объем кода**         | Минимальный         | Большой             | Умеренный           | Максимальный        |
| **Использование**      | Простые workflows   | Аудит/аналитика     | Отчеты              | Комплексные системы |

### 6. Рекомендации по внедрению

1. Стартапы: SAGA + Circuit Breaker
2. Средние проекты: Добавить Event Sourcing для ключевых агрегатов
3. Сложные системы: Полный ES+CQRS с оптимизированными проекциями

<img src="./static/diagrams/recommendations.svg" alt="es_cqrs_overview">

```graph
graph TD
    A[Выбор архитектуры] --> B{Критична ли полная история?}
    B -->|Да| C{Требуется ли высокая производительность чтения?}
    C -->|Да| D[ES+CQRS]
    C -->|Нет| E[Чистый Event Sourcing]
    B -->|Нет| F[Оптимизированная SAGA]
```

## Заключение

Эволюция архитектуры требует постепенного усложнения:

1. Начните с **SAGA** для базовой согласованности
2. Добавьте **Event Sourcing** для ключевых доменов
3. Внедрите **CQRS** для проблемных запросов
4. Комбинируйте **ES+CQRS** в сложных системах

> **Главный принцип:** Выбирайте архитектуру, соответствующую вашим реальным потребностям, а не трендам.