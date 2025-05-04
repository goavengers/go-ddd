# Go-DDD: Domain Driven Design, SAGA, Event-Sourcing & CQRS

## Немного теории

Современная разработка ПО – это почти всегда про распределенные системы. Микросервисы, облака, глобальный охват – все это стало нормой. Но за красивыми диаграммами и модными словами скрывается фундаментальная сложность. Как заставить кучу разрозненных компонентов работать вместе надежно? Как гарантировать, что данные, размазанные по сети, останутся корректными и доступными? Эта головная боль знакома любому, кто проектировал системы сложнее калькулятора, будь то в требовательном финтехе, динамичном e-commerce или где-либо еще.

И вот тут на помощь (или, скорее, для обозначения поля боя) приходят три понятия: ACID, BASE и теорема CAP. Может показаться, что это сухая теория, но игнорировать их – все равно что выходить в море без компаса и карты. Эти концепции описывают фундаментальные компромиссы, с которыми приходится иметь дело каждому архитектору. Понимание их – не гарантия успеха, но его необходимое условие.

### Незыблемый ACID: Классика жанра (и ее ограничения)

Начнем со столпа, на котором десятилетиями держались базы данных и монолитные приложения. **Что же обещает нам ACID? По сути, это четыре столпа надежности транзакций, фундамент, на котором строилась уверенность в классических системах:**

- **Атомарность (Atomicity):** Все просто – либо вся операция проходит успешно, либо откатывается так, словно ее и не затевали. Представьте перевод денег: списали тут, зачислили там. Нельзя застрять посередине. ACID гарантирует вот это "все или ничего".
- **Согласованность (Consistency):** Тут важно не путать с 'C' из CAP! В ACID это значит, что транзакция не нарушит установленных вами правил игры – всяких там ограничений целостности, уникальности. Баланс не уйдет в минус, если нельзя. Данные остаются логически корректными после каждой транзакции.
- **Изолированность (Isolation):** Мир многопоточный, транзакции летят пачками. Изоляция – это щит, который не дает им наступать друг другу на пятки и видеть промежуточный 'беспорядок' друг друга. В идеале (уровень Serializable) все выглядит так, будто они идут строго по очереди. На деле мы часто идем на компромиссы (используем уровни изоляции попроще) ради скорости, но цель – не дать им смешать карты друг другу.
- **Долговечность (Durability):** Сказано – сделано, и точка. Если база отчиталась об успехе, ваши данные переживут перезагрузку, сбой питания – что угодно, кроме совсем уж апокалипсиса вроде пожара в серверной. Запись надежна.

ACID – это прекрасно. Это как сейф для данных. Но попробуйте растянуть этот сейф на несколько комнат (серверов), соединенных коридорами (сетью). Обеспечить ту же непробиваемость между комнатами становится чертовски сложно. Протоколы вроде двухфазного коммита (2PC), которые пытаются это сделать, требуют координатора и блокировок во всех участвующих "комнатах". Если одна комната задумалась или коридор к ней перекрыли – весь процесс встает. Это бьет по производительности (задержки растут) и, главное, по доступности системы. Цена за строгий распределенный ACID часто оказывается слишком высокой.

### Теорема CAP: Жесткая дилемма распределенки

Столкнувшись с проблемами распределенного ACID, индустрия обратилась к теореме CAP. Она стала чем-то вроде закона сохранения энергии для распределенных систем. И нет, это не "выбери два из трех", как часто говорят. Все немного хитрее. Теорема оперирует тремя свойствами:

- Тут **Согласованность (Consistency)** – это про другое, это хардкор. Это когда любое чтение данных откуда угодно выдает самый свежий, только что записанный результат. Как будто источник данных реально один на всех, и все видят его изменения мгновенно.
- **Доступность (Availability)** – значит, система на связи. Вы стучитесь в любой живой узел – он отвечает по существу, а не ошибкой "попробуйте позже". Он может отдать не самые последние данные, но он отвечает.
- **Устойчивость к разделению (Partition Tolerance)** – это способность системы пережить раскол сети, когда узлы теряют связь друг с другом. И вот тут загвоздка, самый важный вывод из CAP: этот раскол сети (P) – не гипотетическая страшилка, а неизбежная реальность любой распределенной системы. Сеть будет "моргать", и с этим надо жить.

И вот что говорит теорема на самом деле: **когда происходит сетевое разделение (P), система должна пожертвовать либо Согласованностью (C), либо Доступностью (A)**. Нельзя иметь и то, и другое **одновременно** в момент разрыва связи.

- **Выбираете CP (Consistency над Availability):** Ваша система превыше всего ценит согласованность данных. Если узел из-за раздела сети не может гарантировать, что видит самую свежую версию данных или что его запись увидят другие, он предпочтет вернуть ошибку или не отвечать вовсе, лишь бы не нарушить линеаризуемость. Вы получаете гарантию C ценой потенциальной недоступности части системы во время P.
- **Выбираете AP (Availability над Consistency):** Ваша система превыше всего ценит возможность ответить пользователю. Даже если узел изолирован разделом, он продолжит обрабатывать запросы (возможно, на основе локальных, потенциально устаревших данных) и принимать новые записи (которые потом придется как-то синхронизировать). Вы получаете гарантию A ценой временной (или даже постоянной, если конфликты не разрешить) рассогласованности данных между частями системы во время P.

А что же системы CA (Consistency + Availability)? Они возможны лишь в гипотетическом мире без сетевых проблем. Как только случается P, любая CA-система вынуждена будет деградировать либо до CP, либо до AP.

### BASE: Прагматизм в мире несовершенства

Итак, если вы столкнулись с реальностью P и выбрали A (доступность), вы естественным образом приходите к принципам, описываемым акронимом BASE. Это не строгий стандарт, а скорее философия проектирования систем, которые оптимизированы для работы в условиях частичных отказов и нестрогой согласованности:

- **Basically Available (Базовая доступность):** Система делает все возможное, чтобы оставаться доступной для запросов, как и предполагает выбор 'A' в CAP. Может быть, не все функции работают идеально, но система "жива".
- **Soft state (Гибкое состояние):** Состояние системы может меняться со временем даже без прямого внешнего воздействия. Это происходит из-за фоновых процессов синхронизации, когда узлы обмениваются информацией и пытаются прийти к единому состоянию. Представьте кэши, которые периодически инвалидируются или обновляются.
- **Eventually consistent (Согласованность в конечном счете):** Самый известный и часто неправильно понимаемый принцип BASE. Система не гарантирует, что сразу после записи все узлы увидят новое значение. Но она гарантирует, что если новых записей в этот конкретный фрагмент данных больше не поступает, то рано или поздно все реплики этого фрагмента сойдутся к последнему записанному значению. "Рано или поздно" – ключевой момент, который может означать миллисекунды, секунды или даже минуты, и это время должно быть как-то ограничено или хотя бы наблюдаемо.

BASE – это признание того, что во многих случаях абсолютная немедленная согласованность не нужна и ее достижение слишком дорого обходится с точки зрения доступности и производительности. Лента новостей, рекомендации товаров, количество просмотров – здесь задержка в синхронизации часто приемлема. Это оптимистичный подход: принимаем изменения, а потом разбираемся. Но важно помнить: работать с eventually consistent данными сложнее, нужно учитывать возможные аномалии чтения (прочитать старые данные после новых, например).

### Архитектура как искусство компромисса: Применяем ACID/BASE/CAP

Понимание этих трех концепций – это только начало. Настоящая работа архитектора – применять их для принятия конкретных решений:

1. **Требования – во главе угла:** Всегда начинайте с вопроса "Что нужно бизнесу и пользователю?". Насколько критична потеря или рассогласованность этих конкретных данных? Какова цена ошибки? Каковы ожидания по времени отклика? Ответы на эти вопросы помогут определить, нужна ли вам крепость ACID или гибкость BASE для данной части системы. И умение перевести технические ограничения CAP на язык бизнес-рисков – важный навык архитектора.
2. **Выбор технологий:** Решение о СУБД, брокере сообщений, кэше – это во многом решение о модели согласованности/доступности.
   - **Реляционные СУБД (Postgres, etc.):** Обычно ваш выбор для данных, требующих строгого ACID. В кластере часто ориентированы на CP.
   - **NoSQL:** Здесь царит разнообразие. Cassandra и Riak славятся своей AP-ориентацией. MongoDB и Couchbase предлагают гибкую настройку согласованности. Key-value хранилища типа Redis часто используют для кэширования (AP). Важно читать документацию и понимать настройки – одна и та же СУБД может вести себя по-разному. Например, настройка кворумов чтения/записи в Cassandra напрямую влияет на баланс C и A.
3. **Архитектурные паттерны:** Многие популярные паттерны – это, по сути, способы обойти ограничения или реализовать нужную модель поведения:
   - **Saga:** Позволяет оркестрировать сложные бизнес-операции, разбивая их на локальные ACID-транзакции с компенсациями. Это способ достичь бизнес-атомарности там, где распределенный ACID (2PC) непрактичен из-за влияния на доступность.
   - **CQRS:** Разделение путей для команд (запись) и запросов (чтение). Позволяет иметь более строгую модель для записи и оптимизированную, возможно, eventually consistent модель для чтения. Это прямой ответ на то, что требования к согласованности для записи и чтения часто различаются.
   - **Event Sourcing:** Запись истории изменений как событий. Отлично ложится на CQRS и облегчает построение eventually consistent проекций, а также анализ и отладку.
   - **Очереди сообщений (Kafka, RabbitMQ):** Сердце асинхронной, слабосвязанной архитектуры (AP/BASE). Позволяют сервисам общаться надежно, но без прямых блокирующих вызовов, сглаживая пики нагрузки и обеспечивая механизм для eventual consistency. Важно также думать об идемпотентности обработчиков сообщений.
4. **Жизнь с Partition Tolerance:** Нужно не только выбрать CP/AP, но и спроектировать систему так, чтобы она могла пережить P. Это включает:
   - Надежный мониторинг состояния узлов и сети.
   - Стратегии разрешения конфликтов для AP-систем (кто побеждает при одновременной записи?).
   - Мониторинг задержек репликации в BASE-системах, чтобы понимать, насколько "старыми" могут быть данные.
   - Четкие алерты при обнаружении разделов сети или аномально больших задержек.

### Осознанный выбор

**ACID, BASE и CAP** – это не догмы, а инструменты мышления. Они подсвечивают неизбежные компромиссы в сложном мире распределенных систем. Нет "серебряной пули". Так что работа архитектора – это постоянный поиск баланса. Идеальных решений тут нет, есть только выбор наиболее подходящего размена для конкретной задачи, с учетом всех "хотелок" бизнеса, технических реалий и того, что нужно пользователю в данный момент. Это всегда компромисс, и наша задача – сделать его осознанным и управляемым.

## Эволюция архитектуры распределенных систем

## Введение

Все, кто занимается разработкой микросервисов, так или иначе решают для себя вопрос: как обеспечить согласованность бизнес-транзакций, в которых участвуют данные нескольких сервисов. Разумеется, лучшее решение - отсутствие такой необходимости. Но не всегда это возможно.

При переходе от монолита к микросервисам ключевой проблемой становятся распределенные транзакции. В этом документе мы проследим эволюцию архитектурных подходов:

1. Базовая реализация SAGA
2. Улучшенная Event-Driven SAGA
3. Переход к CQRS
4. Введение Event Sourcing
5. Комбинированный подход ES+CQRS

## 1. Введение в проблематику

Распределенные транзакции в микросервисной архитектуре требуют особых подходов. 
Рассмотрим эволюцию решения от базовой SAGA до комплексной Event-Driven архитектуры.

## 2. Базовая реализация SAGA

### Проблематика
В микросервисной архитектуре традиционные ACID-транзакции невозможны. SAGA предлагает решение через последовательность компенсируемых операций.


### 2.1. Архитектурный обзор

<img src="./static/diagrams/saga_0.svg" alt="saga_0_overview">

```graph
graph TD
    A[Client] --> B[Order Service]
    B --> C[Catalog Service]
    B --> D[Payment Service]
```

### 2.2. Диаграмма последовательности SAGA

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

### 2.3. Критические компоненты

#### 2.3.1. Order Service

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

#### 2.3.2. Catalog Service

Роль: Управляет остатками товаров.

#### 2.3.3. Payment Service

Роль: Обрабатывает платежи.

### 2.4. Диаграмма последовательнсти

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

### 2.5. Проблемы базовой реализации:

- Нет защиты от **повторных запросов**
- **Каскадные ошибки** при недоступности сервисов
- **Потеря событий** при сбоях
- **Отсутствие истории** изменений

### 2.6. Что можно улучшить

- Добавить Idempotency Key для избежания дублирования операций.
- Внедрить Circuit Breaker для устойчивости к ошибкам.
- Использовать Outbox Pattern для надежной доставки событий.

### 2.7. Вывод

- ✅ SAGA решает проблему распределенных транзакций
- ✅ Каждый сервис управляет своей частью данных
- ✅ Компенсационные транзакции обеспечивают согласованность
- ⚠ Не подходит для высоконагруженных систем (лучше Event Sourcing + CQRS)

### 2.8. Глосарий

> ##### Компенсационные транзакции (SAGA Rollback)
> Если на любом этапе происходит ошибка, система выполняет обратные операции в обратном порядке:
> 1. **Платеж не прошел** → Отмена резерва товаров (CatalogService.CancelReservation).
> 2. **Резерв не удался** → Заказ отклоняется без списания денег.


## 3. Улучшенная Event-Driven SAGA

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

### 3.1. Idempotency Key (Идемпотентность)

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

### 3.2. Circuit Breaker (Автоматический переключатель)

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

### 3.3. Outbox Pattern (Надежная доставка событий)

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

### 3.4. Архитектурная диаграмма

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

### 3.5. Диаграмма последовательнсти

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

## 4. 📊 Сравнение подходов

| Подход              | Достоинства | Недостатки  |
|---------------------|---|---|
| **Базовая SAGA**    | Простота реализации  | Нет защиты от дублирования  |
| **С оптимизациями** | Отказоустойчивость, надежность | Сложность возрастает  |

## 5. Переход к CQRS и Event Sourcing

### 5.1. CQRS: разделение ответственности

#### 5.1.1. Архитектура

<img src="./static/diagrams/cqrs.svg" alt="cqrs_overview">

```graph
graph TB
    A[Command] --> B[Command Handler]
    B --> C[Event Store]
    C --> D[Read Model]
    E[Query] --> F[Read Model]
```

#### 5.1.2. Пример кода

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

### 5.2. Глосарий

> Принцип разделения: 
> 
> **Commands** - изменение состояния
> 
> **Queries** - чтение данных

Queries - чтение данных

## 6. Event Sourcing: принципиально новый подход

### 6.1. Основная концепция

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

### 6.2. Реализация агрегата

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

## 7. Комбинированный подход: Event Sourcing + CQRS

### 7.1. Полная архитектура

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

### 7.2. Пример кода

```go
package main

// todo
```

### 7.3. Диаграмма последовательности

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

### 7.4. Текстовая версия диаграммы:

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

### 7.5. Ключевые особенности потока:

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

### 7.6. Типовые задержки:

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

## 8. Сравнительный анализ

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

## 9. Рекомендации по внедрению

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