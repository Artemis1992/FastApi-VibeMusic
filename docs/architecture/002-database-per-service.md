# ADR-002: Database per Service Pattern

**Статус:** ✅ Принято  
**Дата:** 2025-11-30  
**Автор:** Student + Cursor AI Mentor  
**Связано с:** ADR-001 (Microservices Architecture)

---

## Контекст

В микросервисной архитектуре возникает вопрос: как организовать хранение данных? Должны ли сервисы делить одну базу данных или каждый должен иметь свою?

### Проблемы Shared Database:

1. **Tight Coupling:** Изменение схемы БД требует координации всех сервисов.
2. **Scalability:** Невозможно масштабировать БД для одного сервиса.
3. **Technology Lock-in:** Все сервисы привязаны к одной СУБД.
4. **Data Ownership:** Неясно, кто владеет данными.
5. **Failure Propagation:** Проблемы с БД влияют на все сервисы.

---

## Решение

**Каждый сервис владеет своей базой данных** (Database per Service Pattern).

### Структура БД:

```
PostgreSQL Server:
├── vibe_auth_db        (Auth Service)
├── vibe_content_db     (Content Service)
├── vibe_artist_db      (Artist Service)
├── vibe_comment_db     (Comment Service)
├── vibe_feedback_db    (Feedback Service)
└── vibe_chat_db        (LLM Chat Service)

Elasticsearch Cluster:
└── vibe_search_index   (Search Service)

Redis:
└── vibe_cache          (кэш для всех сервисов)
```

### Правила:

1. ❌ **Запрещено:** JOIN между таблицами разных сервисов
2. ❌ **Запрещено:** FOREIGN KEY на таблицы других сервисов
3. ❌ **Запрещено:** Прямое чтение БД другого сервиса
4. ✅ **Разрешено:** JOIN внутри своей БД
5. ✅ **Разрешено:** Получение данных через API другого сервиса
6. ✅ **Разрешено:** Подписка на Kafka события для синхронизации

---

## Обоснование

### ✅ Преимущества

#### 1. Loose Coupling (Слабая связность)
```python
# ❌ ПЛОХО: Content Service читает таблицу users из Auth DB
user = db.query(User).filter(User.id == post.author_id).first()

# ✅ ХОРОШО: Content Service запрашивает через API
user_data = await auth_client.get_user(post.author_id)
```

**Результат:** Content Service может работать, даже если Auth DB недоступна (с кэшем).

#### 2. Independent Scaling
- Auth DB: небольшая, много запросов → оптимизация для reads (read replicas)
- Content DB: много данных → sharding по user_id
- Artist DB: медленные запросы → separate indexes

#### 3. Technology Diversity
```yaml
Auth Service: PostgreSQL (ACID транзакции)
Search Service: Elasticsearch (полнотекстовый поиск)
Chat Service: PostgreSQL + Vector DB (embeddings)
```

#### 4. Failure Isolation
- Падение Content DB не влияет на аутентификацию
- Проблемы с индексацией Search не ломают создание постов

#### 5. Schema Evolution
- Content Service может изменить схему `posts` без согласования с Artist Service
- Миграции запускаются независимо

---

### ⚠️ Недостатки и Митигация

#### 1. Нет ACID транзакций между сервисами

**Проблема:**
```python
# Как гарантировать атомарность?
create_post(post_data)           # Content Service
create_notification(user_id)     # Notification Service
```

**Митигация:**
- **Saga Pattern** (через Temporal или Kafka)
- **Idempotency:** каждый запрос имеет уникальный ID
- **Eventual Consistency:** данные в итоге станут консистентными

**Пример Saga:**
```
1. Content Service создает пост (status=pending)
2. Публикует событие "post.created"
3. Search Service индексирует → публикует "post.indexed"
4. Content Service обновляет status=published
Если ошибка → компенсирующая транзакция (удаление поста)
```

#### 2. Нет JOIN между сервисами

**Проблема:**
```sql
-- Так нельзя!
SELECT posts.*, artists.name 
FROM posts 
JOIN artists ON posts.artist_id = artists.id;
```

**Митигация:**

**A) API Composition (в Gateway):**
```python
# Gateway Service
posts = await content_client.get_posts()
for post in posts:
    artist = await artist_client.get_artist(post.artist_id)
    post.artist_name = artist.name
```

**B) CQRS + Read Model (для сложных запросов):**
```python
# Search Service хранит денормализованные данные
{
  "post_id": "123",
  "title": "Best Song",
  "artist_name": "John Doe",  # Дублируется из Artist Service
  "author_username": "admin"   # Дублируется из Auth Service
}
```

**C) Kafka Event Sourcing:**
```
1. Artist Service публикует "artist.updated"
2. Content Service слушает событие и обновляет локальный кэш
```

#### 3. Data Duplication

**Проблема:** Search Service хранит копии данных из Content и Artist.

**Митигация:**
- Считаем это trade-off (производительность > дублирование)
- Single Source of Truth: Content Service владеет постами
- Kafka гарантирует синхронизацию (at-least-once delivery)

#### 4. Distributed Queries

**Проблема:** Пагинация и сортировка по полям из разных сервисов.

**Пример:**
```
Получить посты, отсортированные по artist.popularity
```

**Митигация:**
- Выполнять в Search Service (денормализованные данные)
- Или: получить IDs из одного сервиса, затем обогатить данными из других

---

## Альтернативы

### 1. Shared Database
**Описание:** Все сервисы используют одну БД.

**Плюсы:**
- JOIN между таблицами
- ACID транзакции
- Нет дублирования

**Минусы:**
- Tight Coupling (главная проблема!)
- Невозможность независимого масштабирования
- Единая точка отказа

**Почему отклонено:** Нарушает главный принцип микросервисов.

---

### 2. Shared Database с Private Tables
**Описание:** Одна БД, но каждый сервис использует свои таблицы.

**Плюсы:**
- Проще настроить
- Можно делать JOIN (но не рекомендуется)

**Минусы:**
- Все еще Tight Coupling
- Сложно масштабировать
- Риск случайного доступа к чужим таблицам

**Почему отклонено:** Полумера, не решает проблему.

---

### 3. Event Sourcing для всего
**Описание:** Все данные хранятся как события в Kafka.

**Плюсы:**
- Полная история изменений
- Легко воспроизвести состояние

**Минусы:**
- Очень сложная реализация
- Сложные запросы (нужны проекции)
- Большой объем данных

**Почему отклонено:** Overkill для нашего проекта. Используем гибрид (БД + Kafka для синхронизации).

---

## Паттерны Взаимодействия

### 1. Synchronous (HTTP)
**Когда:** Чтение данных, критичных для ответа.

```python
# Gateway агрегирует данные
post = await content_service.get_post(post_id)
artist = await artist_service.get_artist(post.artist_id)
comments = await comment_service.get_comments(post_id)

return {
    "post": post,
    "artist": artist,
    "comments": comments
}
```

**Проблемы:**
- Latency (3 последовательных запроса)
- Cascading failures

**Решения:**
- Кэширование (Redis)
- Circuit Breaker
- Параллельные запросы (asyncio.gather)

---

### 2. Asynchronous (Kafka Events)
**Когда:** Синхронизация данных, не критичная для ответа.

```python
# Content Service публикует событие
await kafka_producer.send("post.created", {
    "post_id": str(post.id),
    "title": post.title,
    "artist_id": str(post.artist_id)
})

# Search Service слушает и индексирует
@kafka_consumer("post.created")
async def index_post(event):
    await elasticsearch.index("posts", event)
```

**Плюсы:**
- Отказоустойчивость (retry из Kafka)
- Decoupling (сервисы не знают друг о друге)

---

### 3. Data Replication
**Когда:** Часто запрашиваемые данные.

```python
# Content Service кэширует artist names
class Post:
    artist_id: UUID
    _cached_artist_name: str | None  # Локальный кэш
    
    async def get_artist_name(self):
        if not self._cached_artist_name:
            artist = await artist_client.get(self.artist_id)
            self._cached_artist_name = artist["name"]
        return self._cached_artist_name
```

---

## Последствия

### Что нужно реализовать:

1. **Миграции:**
   - Alembic для каждого сервиса
   - Скрипт `migrate_all.sh` для запуска всех миграций

2. **API Clients:**
   - HTTP клиенты с retry и circuit breaker
   - Кэширование частых запросов

3. **Event System:**
   - Kafka Producer в каждом сервисе
   - Kafka Consumers для синхронизации

4. **Data Consistency:**
   - Idempotency keys
   - Saga Pattern для критичных операций

5. **Monitoring:**
   - Метрики консистентности данных
   - Алерты на расхождения

---

## Правила Разработки

### ✅ МОЖНО:
```python
# 1. JOIN внутри своей БД
posts = db.query(Post).join(PostMetadata).all()

# 2. Запрос через API
artist = await artist_client.get_artist(artist_id)

# 3. Кэширование
cached = await redis.get(f"artist:{artist_id}")

# 4. События Kafka
await producer.send("post.created", post_data)
```

### ❌ НЕЛЬЗЯ:
```python
# 1. Import моделей другого сервиса
from artist_service.models import Artist  # ЗАПРЕЩЕНО!

# 2. JOIN между БД
db.query(Post).join(Artist).all()  # ЗАПРЕЩЕНО!

# 3. Foreign Key на другую БД
artist_id = Column(UUID, ForeignKey("vibe_artist_db.artists.id"))  # ЗАПРЕЩЕНО!
```

---

## Критерии Успеха

1. ✅ Каждый сервис может деплоиться независимо
2. ✅ Падение БД одного сервиса не влияет на других (с кэшем)
3. ✅ Время ответа Gateway < 200ms (95 percentile)
4. ✅ Нет прямых обращений между БД разных сервисов (проверяется code review)

---

## Связанные ADR

- ADR-001: Microservices Architecture
- ADR-003: Kafka для Event-Driven Architecture

---

## Ссылки

1. [Database per Service Pattern](https://microservices.io/patterns/data/database-per-service.html)
2. [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
3. [Saga Pattern](https://microservices.io/patterns/data/saga.html)

---

**Статус:** ✅ Принято  
**Следующий пересмотр:** После реализации первых 3 сервисов

