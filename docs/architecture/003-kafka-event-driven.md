# ADR-003: Kafka для Event-Driven Architecture

**Статус:** ✅ Принято  
**Дата:** 2025-11-30  
**Автор:** Student + Cursor AI Mentor  
**Связано с:** ADR-001 (Microservices), ADR-002 (Database per Service)

---

## Контекст

В микросервисной архитектуре с Database per Service возникают вопросы:

1. Как синхронизировать данные между сервисами?
2. Как избежать длинных цепочек синхронных HTTP вызовов?
3. Как обеспечить отказоустойчивость при обновлении данных?
4. Как масштабировать обработку событий?

### Проблемы синхронного взаимодействия:

```python
# Синхронная цепочка (анти-паттерн)
Gateway -> Content Service -> Artist Service -> Notification Service
   200ms      + 50ms              + 80ms            + 100ms
= 430ms total latency!

# Если Notification Service упадет - весь запрос провалится
```

---

## Решение

**Используем Apache Kafka** как Message Broker для асинхронного взаимодействия между сервисами (Event-Driven Architecture).

### Принцип работы:

```
Publisher (Producer)           Kafka Broker           Subscribers (Consumers)
     │                              │                         │
     │    1. Публикация события     │                         │
     ├─────────────────────────────>│                         │
     │       "post.created"          │                         │
     │                               │    2. Доставка события │
     │                               ├────────────────────────>│ Search Service
     │                               │                         │ (индексирует пост)
     │                               │                         │
     │                               ├────────────────────────>│ Notification Service
     │                               │                         │ (отправляет уведомление)
     │                               │                         │
     │    3. Ответ сразу             │                         │
     │<──────────────────────────────┘                         │
     │    (не ждем обработки)        │                         │
```

### Kafka Topics:

| Topic | Publisher | Subscribers | Описание |
|-------|-----------|-------------|----------|
| `user.registered` | Auth Service | Chat Service, Notification | Регистрация пользователя |
| `post.created` | Content Service | Search Service, Notification | Создание поста |
| `post.updated` | Content Service | Search Service | Обновление поста |
| `post.deleted` | Content Service | Search, Comment Service | Удаление поста |
| `comment.created` | Comment Service | Notification Service | Новый комментарий |
| `artist.created` | Artist Service | Search Service | Новый артист |
| `artist.updated` | Artist Service | Content Service (cache), Search | Обновление артиста |

---

## Обоснование

### ✅ Преимущества

#### 1. Decoupling (Разделение)

**Без Kafka:**
```python
# Content Service знает обо всех подписчиках
async def create_post(post_data):
    post = await db.save(post_data)
    
    # Tight coupling!
    await search_service.index_post(post)  # Если упадет - весь запрос упадет
    await notification_service.notify(post)  # Еще одна зависимость
    
    return post
```

**С Kafka:**
```python
# Content Service НЕ знает о подписчиках
async def create_post(post_data):
    post = await db.save(post_data)
    
    # Publish and forget
    await kafka_producer.send("post.created", {
        "post_id": str(post.id),
        "title": post.title,
        "author_id": str(post.author_id)
    })
    
    return post  # Ответ сразу!
```

**Результат:** Добавление нового подписчика не требует изменений в Content Service.

---

#### 2. Resilience (Отказоустойчивость)

**Проблема:** Search Service упал во время индексации.

**Решение:** Kafka хранит события, пока они не будут обработаны.

```python
# Search Service восстанавливается и обрабатывает пропущенные события
@kafka_consumer("post.created", group_id="search-indexer")
async def index_post(event):
    try:
        await elasticsearch.index("posts", event)
        # Kafka автоматически коммитит offset
    except Exception as e:
        logger.error(f"Indexing failed: {e}")
        # Kafka повторит попытку (retry)
```

**Гарантии Kafka:**
- **At-least-once delivery:** Событие будет доставлено хотя бы один раз
- **Retention:** События хранятся N дней (по умолчанию 7)
- **Replication:** Данные реплицируются на N брокеров

---

#### 3. Scalability (Масштабируемость)

**Partitioning:**
```python
# События распределяются по партициям (по ключу)
await producer.send(
    "post.created",
    key=str(post.author_id),  # Все посты одного автора в одну партицию
    value=event_data
)
```

**Consumer Groups:**
```
Topic: post.created (3 partitions)
Consumer Group: search-indexer (3 consumers)

Partition 0 -> Consumer 1
Partition 1 -> Consumer 2
Partition 2 -> Consumer 3

= Параллельная обработка!
```

---

#### 4. Audit Log (Журнал событий)

```python
# Kafka - это immutable log всех событий
# Можно воспроизвести состояние системы

# Восстановить Search Index с нуля
await consumer.seek_to_beginning()
async for event in consumer:
    await elasticsearch.index(event)
```

---

#### 5. Temporal Decoupling

**Без Kafka:**
```python
# Notification Service должен быть доступен прямо сейчас
await notification_service.send_email(user_email)  # Может упасть
```

**С Kafka:**
```python
# Notification Service обработает, когда сможет
await producer.send("notification.requested", email_data)
# Даже если сервис сейчас недоступен - событие в очереди
```

---

### ⚠️ Недостатки и Митигация

#### 1. Eventual Consistency

**Проблема:** Данные обновляются с задержкой.

```python
# 1. Content Service создает пост (200ms)
await content_service.create_post()

# 2. Kafka доставляет событие (50ms)
# 3. Search Service индексирует (100ms)

# Итого: пост появится в поиске через ~350ms
```

**Митигация:**
- Для критичных данных: синхронные HTTP запросы
- Для остального: eventual consistency приемлема
- Показывать пользователю индикатор "обработка..."

---

#### 2. Message Ordering

**Проблема:** События могут прийти не в порядке отправки.

```python
# Отправлено:
1. post.created (id=123)
2. post.updated (id=123, title="New Title")

# Получено (из-за retry):
1. post.updated (title="New Title")
2. post.created (title="Old Title")  # Перезаписывает новое!
```

**Митигация:**
- Использовать **партиции с ключом** (одинаковый post_id → одна партиция)
- Добавлять `version` или `timestamp` в события
- Проверять версию перед обновлением

```python
# В событии
{
    "post_id": "123",
    "version": 2,  # Игнорировать события с version < текущей
    "title": "New Title"
}
```

---

#### 3. Duplicate Messages

**Проблема:** At-least-once может привести к дублям.

```python
# Событие обработано, но коммит упал
# Kafka отправит снова
```

**Митигация:** **Idempotency** (идемпотентность)

```python
@kafka_consumer("post.created")
async def index_post(event):
    post_id = event["post_id"]
    
    # Проверяем, не обработано ли уже
    if await redis.exists(f"processed:{post_id}"):
        logger.info(f"Post {post_id} already indexed, skip")
        return
    
    # Обрабатываем
    await elasticsearch.index("posts", event)
    
    # Помечаем как обработанное
    await redis.setex(f"processed:{post_id}", 86400, "1")  # TTL 24h
```

---

#### 4. Debugging

**Проблема:** Сложно отследить цепочку событий.

**Митигация:**
- **Correlation ID:** пробрасывать через все события
- **Structured Logging:** логировать каждое событие

```python
# В событии
{
    "event_id": "uuid-1234",
    "correlation_id": "request-5678",  # Из исходного HTTP запроса
    "timestamp": "2025-11-30T10:00:00Z",
    "event_type": "post.created",
    "data": { ... }
}

# В логах
[INFO] correlation_id=request-5678 event=post.created status=published
[INFO] correlation_id=request-5678 event=post.created status=indexed
```

---

## Альтернативы

### 1. RabbitMQ

**Плюсы:**
- Проще настроить
- Routing по правилам (exchanges)

**Минусы:**
- Меньше throughput, чем Kafka
- Не подходит для Event Sourcing (нет long-term storage)

**Почему отклонено:** Kafka лучше для high-throughput и audit log.

---

### 2. AWS SQS/SNS

**Плюсы:**
- Managed service (нет управления)
- Хорошая интеграция с AWS

**Минусы:**
- Vendor lock-in
- Дороже Kafka на больших объемах

**Почему отклонено:** Хотим избежать привязки к AWS.

---

### 3. Redis Pub/Sub

**Плюсы:**
- Очень быстрый
- Уже используем Redis

**Минусы:**
- Нет persistence (сообщения не хранятся)
- Нет гарантий доставки

**Почему отклонено:** Не подходит для критичных событий.

---

### 4. HTTP Webhooks

**Плюсы:**
- Простота (обычные HTTP запросы)

**Минусы:**
- Publisher должен знать всех подписчиков (tight coupling)
- Нет retry из коробки
- Сложно масштабировать

**Почему отклонено:** Нарушает decoupling.

---

## Паттерны Использования

### 1. Event Notification

**Когда:** Уведомить другие сервисы об изменении.

```python
# Content Service
await producer.send("post.created", {
    "post_id": str(post.id),
    "title": post.title
})

# Search Service просто знает, что пост создан
```

---

### 2. Event-Carried State Transfer

**Когда:** Передать полные данные в событии.

```python
# Content Service отправляет ВСЕ данные поста
await producer.send("post.created", {
    "post_id": str(post.id),
    "title": post.title,
    "content": post.content,
    "author_id": str(post.author_id),
    "artist_id": str(post.artist_id),
    "created_at": post.created_at.isoformat()
})

# Search Service не делает доп. HTTP запросов
```

**Плюсы:** Меньше сетевых вызовов  
**Минусы:** Большие сообщения

---

### 3. Event Sourcing

**Когда:** Хранить все изменения как события.

```python
# Вместо UPDATE в БД
events = [
    {"type": "post.created", "data": {...}},
    {"type": "post.title.updated", "data": {"title": "New"}},
    {"type": "post.published", "data": {}}
]

# Состояние восстанавливается из событий
```

**Плюсы:** Полная история, audit trail  
**Минусы:** Сложная реализация

**Решение:** Гибрид (БД + ключевые события в Kafka).

---

## Технические Детали

### Setup (Development):

Используем **Redpanda** вместо Kafka (совместимый, но проще):

```yaml
# docker-compose.yml
services:
  redpanda:
    image: vectorized/redpanda:latest
    ports:
      - "9092:9092"  # Kafka API
      - "8081:8081"  # Schema Registry
```

### Producer (Python):

```python
# app/events/producer.py
from kafka import KafkaProducer
import json

class EventProducer:
    def __init__(self, bootstrap_servers: str):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',  # Ждать подтверждения от всех реплик
            retries=3
        )
    
    async def publish(self, topic: str, event: dict):
        self.producer.send(topic, value=event)
        self.producer.flush()
```

### Consumer (Python):

```python
# app/events/consumer.py
from kafka import KafkaConsumer
import json

class EventConsumer:
    def __init__(self, topics: list[str], group_id: str):
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers='localhost:9092',
            group_id=group_id,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            enable_auto_commit=False  # Ручной коммит после обработки
        )
    
    async def process_events(self):
        for message in self.consumer:
            try:
                await self.handle_event(message.value)
                self.consumer.commit()  # Успешно обработано
            except Exception as e:
                logger.error(f"Failed to process: {e}")
                # Не коммитим - Kafka повторит
```

---

## Схема Событий

### Базовая структура:

```python
from pydantic import BaseModel
from datetime import datetime
from uuid import UUID

class BaseEvent(BaseModel):
    event_id: UUID
    event_type: str
    correlation_id: UUID
    timestamp: datetime
    data: dict
```

### Пример события:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "post.created",
  "correlation_id": "request-123",
  "timestamp": "2025-11-30T10:30:00Z",
  "data": {
    "post_id": "123e4567-e89b-12d3-a456-426614174000",
    "title": "Best Rock Song",
    "author_id": "user-456",
    "artist_id": "artist-789",
    "slug": "best-rock-song"
  }
}
```

---

## Мониторинг

### Метрики:
- Kafka lag (задержка обработки)
- Messages per second
- Failed message rate
- Consumer uptime

### Dashboards (Grafana):
```
1. Topic Throughput (сообщений/сек)
2. Consumer Lag (сколько сообщений не обработано)
3. Error Rate (% упавших обработок)
```

---

## Последствия

### Что нужно реализовать:

1. **Infrastructure:**
   - [ ] Redpanda в docker-compose
   - [ ] Kafka в Kubernetes

2. **Code:**
   - [ ] Базовые классы Producer/Consumer
   - [ ] Схемы событий (Pydantic)
   - [ ] Idempotency logic

3. **Observability:**
   - [ ] Correlation ID middleware
   - [ ] Event logging
   - [ ] Kafka lag monitoring

---

## Правила Разработки

### ✅ Используй Kafka для:
- Синхронизация данных (Search index)
- Уведомления (email, push)
- Audit log
- Decoupling сервисов

### ❌ НЕ используй Kafka для:
- Запросов (queries) - используй HTTP
- Критичных данных, нужных немедленно
- Request-Response паттерна

---

## Критерии Успеха

1. ✅ Latency < 100ms для публикации события
2. ✅ Consumer lag < 1 секунды (95 percentile)
3. ✅ 0 потерянных событий (at-least-once)
4. ✅ Падение одного consumer не ломает других

---

## Связанные ADR

- ADR-001: Microservices Architecture
- ADR-002: Database per Service

---

## Ссылки

1. [Martin Fowler - Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
2. [Kafka Documentation](https://kafka.apache.org/documentation/)
3. [Designing Events-First Microservices](https://www.confluent.io/blog/event-driven-microservices/)

---

**Статус:** ✅ Принято  
**Следующий пересмотр:** После реализации первых событий (Урок 5)

