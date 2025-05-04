# go-ddd
Domain Driven Design, SAGA, Event-Sourcing &amp; CQRS

## Эволюция архитектуры шаг за шагом

### Архитектура SAGA

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
