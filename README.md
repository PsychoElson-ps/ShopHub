# ShopHub.Microservices — КТ-4

Учебная реализация КТ-4 для архитектуры ShopHub из КТ-2 и КТ-3.

## Что реализовано

- **CatalogService** — CRUD товаров, PostgreSQL, Redis-кэш карточки товара, Swagger, `/health`.
- **OrderService** — создание и чтение заказов, HTTP-вызов CatalogService, Polly retry + circuit breaker, PostgreSQL.
- **NotificationService** — RabbitMQ consumer `OrderCreated`, идемпотентная обработка через `eventId`, журнал уведомлений PostgreSQL, Fake Email в dev и SMTP через MailKit.
- **Transactional Outbox** — заказ и событие сохраняются в одной транзакции PostgreSQL, фоновый publisher публикует событие в RabbitMQ после COMMIT.
- **Docker Compose** — три сервиса, три PostgreSQL, Redis и RabbitMQ поднимаются одной командой.
- **Health Checks** — `/health` у каждого сервиса и healthchecks у инфраструктурных контейнеров.

## Архитектурный поток

```text
Client
  |
  | POST /api/orders
  v
OrderService ---- HTTP GET ----> CatalogService ----> PostgreSQL
     |
     | save Order + Outbox (same transaction)
     v
 PostgreSQL
     |
     | OutboxPublisher
     v
 RabbitMQ
     |
     v
NotificationService ----> notification_log / Fake SMTP
```

## Требования

- Docker Desktop с Docker Compose v2
- .NET 8 SDK, если запускать сервисы вне Docker

## Запуск через Docker Compose

```bash
cp .env.example .env
# Windows PowerShell:
# Copy-Item .env.example .env

docker compose up --build
```

Проверить:

- Catalog Swagger: http://localhost:5002/swagger
- Order Swagger: http://localhost:5004/swagger
- Catalog health: http://localhost:5002/health
- Order health: http://localhost:5004/health
- Notification health: http://localhost:5006/health
- RabbitMQ Management: http://localhost:15672

RabbitMQ: `admin` / пароль из `.env`.

## Сквозной тест

1. В `CatalogService` вызови `POST /api/products` или используй тестовый товар `iPhone 15 Pro`.
2. В `OrderService` вызови `POST /api/orders` и передай заголовок `X-User-Id`.
3. В ответе должен быть `201 Created`.
4. `OrderService` создаёт заказ и запись Outbox.
5. Фоновый publisher отправляет `OrderCreated` в RabbitMQ.
6. `NotificationService` принимает событие и пишет в лог `📧 FAKE EMAIL`.
7. В PostgreSQL NotificationService появляется строка в `notification_log`.

Готовый набор запросов: `requests.http`.
PowerShell smoke test: `Tests/smoke-test.ps1`.

## Проверка ошибок

### Каталог недоступен

Останови `catalog-service` и повтори создание заказа. OrderService не должен зависнуть: после ограниченных retry/circuit breaker вернётся ошибка интеграции.

### RabbitMQ недоступен

Заказ уже может быть сохранён в `orders` вместе с `outbox_messages`. После восстановления RabbitMQ OutboxPublisher повторит публикацию.

### Повторная доставка события

`NotificationService` проверяет `eventId`. Если уведомление уже имеет статус `SENT`, повторная доставка не приводит к повторной отправке.

## Почему именно такие решения

.NET 8 контейнеры поддерживают non-root пользователя `app` и используют порт 8080 по умолчанию. Поэтому Dockerfile использует `USER app` и `EXPOSE 8080`.  Health checks публикуются через `MapHealthChecks`, а Compose запускает зависимые сервисы после прохождения `service_healthy`.  Polly используется напрямую для retry/circuit breaker в HTTP-клиенте.

## Соответствие критериям КТ-4

| Критерий | Реализация |
|---|---|
| CatalogService CRUD + БД | Да: PostgreSQL + CRUD |
| OrderService + HTTP | Да: OrderService → CatalogService |
| RabbitMQ | Да: OrderCreated → NotificationService |
| docker-compose | Да: 3 сервиса + БД/Redis/RabbitMQ |
| `/health` | Да: у всех 3 сервисов |
| Retry/Circuit Breaker | Да: Polly в CatalogServiceClient |
| Документация/summary | Да: README + XML summary |

## Важное замечание

В КТ-4 используется учебный `X-User-Id` вместо полноценного JWT/API Gateway из КТ-2/КТ-3, потому что обязательные критерии КТ-4 не требуют реализации IdentityService и YARP. Это не заменяет JWT в целевой архитектуре, а только упрощает запуск и демонстрацию трёх сервисов.
