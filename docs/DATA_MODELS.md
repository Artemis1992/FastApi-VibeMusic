# 🗄️ Модели Данных VobeMusic

## 🎯 Принципы Проектирования Моделей

### 1. Database per Service
- Каждый сервис владеет своими таблицами
- NO FOREIGN KEYS между базами разных сервисов
- Связи через ID (например, `user_id`, `post_id`)

### 2. Eventual Consistency
- Данные синхронизируются через Kafka события
- Допустима небольшая задержка (< 1 секунды)

### 3. Defensive Programming
- Валидация существования связанных сущностей через API
- Soft Delete для предотвращения потери данных

---

## 📊 Модели по Сервисам
 
---

### 1️⃣ Auth Service (База: `vibe_auth_db`)

#### Таблица: `users`
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    is_superuser BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_login TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
```

**SQLAlchemy Model (app/models/user.py):**
```python
from sqlalchemy import Column, String, Boolean, DateTime
from sqlalchemy.dialects.postgresql import UUID
import uuid

class User(Base):
    __tablename__ = "users"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    email = Column(String(255), unique=True, nullable=False, index=True)
    username = Column(String(50), unique=True, nullable=False, index=True)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    is_superuser = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    last_login = Column(DateTime, nullable=True)
```

#### Таблица: `refresh_tokens`
```sql
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,  -- NO FK! Связь через API
    token VARCHAR(500) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
```

---

### 2️⃣ Content Service (База: `vibe_content_db`)

#### Таблица: `posts`
```sql
CREATE TABLE posts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    content TEXT NOT NULL,
    author_id UUID NOT NULL,        -- ID из Auth Service (NO FK!)
    primary_artist_id UUID,         -- ID из Artist Service (NO FK!)
    cover_image_url VARCHAR(500),
    slug VARCHAR(255) UNIQUE NOT NULL,
    status VARCHAR(20) DEFAULT 'draft',  -- draft, published, archived
    views_count INTEGER DEFAULT 0,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP
);

CREATE INDEX idx_posts_author_id ON posts(author_id);
CREATE INDEX idx_posts_artist_id ON posts(primary_artist_id);
CREATE INDEX idx_posts_slug ON posts(slug);
CREATE INDEX idx_posts_status ON posts(status);
CREATE INDEX idx_posts_published_at ON posts(published_at DESC);
```

**SQLAlchemy Model:**
```python
class Post(Base):
    __tablename__ = "posts"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    title = Column(String(255), nullable=False)
    description = Column(Text, nullable=True)
    content = Column(Text, nullable=False)
    author_id = Column(UUID(as_uuid=True), nullable=False, index=True)  # NO FK!
    primary_artist_id = Column(UUID(as_uuid=True), nullable=True, index=True)  # NO FK!
    cover_image_url = Column(String(500), nullable=True)
    slug = Column(String(255), unique=True, nullable=False, index=True)
    status = Column(String(20), default="draft", index=True)
    views_count = Column(Integer, default=0)
    is_deleted = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    published_at = Column(DateTime, nullable=True, index=True)
```

#### Таблица: `post_artists` (Many-to-Many)
```sql
CREATE TABLE post_artists (
    post_id UUID NOT NULL,
    artist_id UUID NOT NULL,        -- ID из Artist Service
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (post_id, artist_id)
);

CREATE INDEX idx_post_artists_artist_id ON post_artists(artist_id);
```

**Комментарий:** Эта таблица связывает посты с несколькими артистами (featured artists).

---

### 3️⃣ Artist Service (База: `vibe_artist_db`)

#### Таблица: `artists`
```sql
CREATE TABLE artists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    bio TEXT,
    photo_url VARCHAR(500),
    country VARCHAR(100),
    genres JSONB,                   -- ["Rock", "Pop"]
    social_links JSONB,             -- {"spotify": "...", "instagram": "..."}
    verified BOOLEAN DEFAULT FALSE,
    followers_count INTEGER DEFAULT 0,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_artists_name ON artists(name);
CREATE INDEX idx_artists_genres ON artists USING GIN(genres);
```

#### Таблица: `albums`
```sql
CREATE TABLE albums (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artist_id UUID NOT NULL,        -- FK внутри своей БД - РАЗРЕШЕНО!
    title VARCHAR(255) NOT NULL,
    description TEXT,
    cover_url VARCHAR(500),
    release_date DATE,
    album_type VARCHAR(20),         -- album, single, EP
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (artist_id) REFERENCES artists(id) ON DELETE CASCADE
);

CREATE INDEX idx_albums_artist_id ON albums(artist_id);
CREATE INDEX idx_albums_release_date ON albums(release_date DESC);
```

#### Таблица: `tracks` (Музыкальные произведения)
```sql
CREATE TABLE tracks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    album_id UUID NOT NULL,         -- FK внутри своей БД - РАЗРЕШЕНО!
    title VARCHAR(255) NOT NULL,
    duration INTEGER,               -- Секунды
    track_number INTEGER,
    audio_url VARCHAR(500),         -- URL на S3/CDN
    lyrics TEXT,
    plays_count INTEGER DEFAULT 0,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (album_id) REFERENCES albums(id) ON DELETE CASCADE
);

CREATE INDEX idx_tracks_album_id ON tracks(album_id);
```

**SQLAlchemy Models:**
```python
class Artist(Base):
    __tablename__ = "artists"
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False, index=True)
    # ...
    albums = relationship("Album", back_populates="artist")  # OK внутри сервиса!

class Album(Base):
    __tablename__ = "albums"
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    artist_id = Column(UUID(as_uuid=True), ForeignKey("artists.id"), nullable=False)
    # ...
    artist = relationship("Artist", back_populates="albums")
    tracks = relationship("Track", back_populates="album")

class Track(Base):
    __tablename__ = "tracks"
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    album_id = Column(UUID(as_uuid=True), ForeignKey("albums.id"), nullable=False)
    # ...
    album = relationship("Album", back_populates="tracks")
```

---

### 4️⃣ Comment Service (База: `vibe_comment_db`)

#### Таблица: `comments`
```sql
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    post_id UUID NOT NULL,          -- ID из Content Service (NO FK!)
    user_id UUID NOT NULL,          -- ID из Auth Service (NO FK!)
    parent_id UUID,                 -- Для вложенных комментариев (FK внутри БД)
    content TEXT NOT NULL,
    is_edited BOOLEAN DEFAULT FALSE,
    is_deleted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (parent_id) REFERENCES comments(id) ON DELETE CASCADE
);

CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
CREATE INDEX idx_comments_parent_id ON comments(parent_id);
CREATE INDEX idx_comments_created_at ON comments(created_at DESC);
```

**SQLAlchemy Model:**
```python
class Comment(Base):
    __tablename__ = "comments"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    post_id = Column(UUID(as_uuid=True), nullable=False, index=True)  # NO FK!
    user_id = Column(UUID(as_uuid=True), nullable=False, index=True)  # NO FK!
    parent_id = Column(UUID(as_uuid=True), ForeignKey("comments.id"), nullable=True)
    content = Column(Text, nullable=False)
    is_edited = Column(Boolean, default=False)
    is_deleted = Column(Boolean, default=False)
    created_at = Column(DateTime, default=datetime.utcnow, index=True)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    # Relationship внутри сервиса - OK!
    replies = relationship("Comment", backref=backref("parent", remote_side=[id]))
```

---

### 5️⃣ Feedback Service (База: `vibe_feedback_db`)

#### Таблица: `feedback_tickets`
```sql
CREATE TABLE feedback_tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,          -- ID из Auth Service (NO FK!)
    subject VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    category VARCHAR(50),           -- bug, feature_request, question
    status VARCHAR(20) DEFAULT 'open',  -- open, in_progress, resolved, closed
    priority VARCHAR(20) DEFAULT 'medium',
    admin_notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    resolved_at TIMESTAMP
);

CREATE INDEX idx_feedback_user_id ON feedback_tickets(user_id);
CREATE INDEX idx_feedback_status ON feedback_tickets(status);
CREATE INDEX idx_feedback_created_at ON feedback_tickets(created_at DESC);
```

---

### 6️⃣ LLM Chat Service (База: `vibe_chat_db`)

#### Таблица: `chat_sessions`
```sql
CREATE TABLE chat_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,          -- ID из Auth Service (NO FK!)
    title VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_chat_sessions_user_id ON chat_sessions(user_id);
```

#### Таблица: `chat_messages`
```sql
CREATE TABLE chat_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL,
    role VARCHAR(20) NOT NULL,      -- user, assistant, system
    content TEXT NOT NULL,
    tokens_used INTEGER,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (session_id) REFERENCES chat_sessions(id) ON DELETE CASCADE
);

CREATE INDEX idx_chat_messages_session_id ON chat_messages(session_id);
CREATE INDEX idx_chat_messages_created_at ON chat_messages(created_at);
```

---

### 7️⃣ Search Service (База: `Elasticsearch`)

#### Index: `posts_index`
```json
{
  "mappings": {
    "properties": {
      "post_id": { "type": "keyword" },
      "title": { "type": "text", "analyzer": "standard" },
      "description": { "type": "text" },
      "content": { "type": "text" },
      "author_username": { "type": "keyword" },
      "artist_names": { "type": "text" },
      "status": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "published_at": { "type": "date" },
      "created_at": { "type": "date" }
    }
  }
}
```

#### Index: `artists_index`
```json
{
  "mappings": {
    "properties": {
      "artist_id": { "type": "keyword" },
      "name": { "type": "text" },
      "bio": { "type": "text" },
      "genres": { "type": "keyword" },
      "country": { "type": "keyword" },
      "verified": { "type": "boolean" }
    }
  }
}
```

**Комментарий:** Данные в Elasticsearch дублируются из основных БД через Kafka события.

---

## 🔄 Синхронизация Данных через Kafka

### Пример: Создание Поста

#### 1. Content Service создает пост:
```python
# content-service/app/core/services.py
async def create_post(post_data: PostCreate):
    post = Post(**post_data.dict())
    db.add(post)
    await db.commit()
    
    # Публикуем событие в Kafka
    await event_producer.publish(
        topic="post.created",
        event={
            "event_id": str(uuid.uuid4()),
            "event_type": "post.created",
            "timestamp": datetime.utcnow().isoformat(),
            "data": {
                "post_id": str(post.id),
                "title": post.title,
                "author_id": str(post.author_id),
                "artist_id": str(post.primary_artist_id),
                "slug": post.slug,
                "published_at": post.published_at.isoformat()
            }
        }
    )
    return post
```

#### 2. Search Service слушает событие:
```python
# search-service/app/events/consumer.py
async def handle_post_created(event: dict):
    post_data = event["data"]
    
    # Получаем дополнительные данные через API
    artist_name = await artist_client.get_artist_name(post_data["artist_id"])
    author_username = await auth_client.get_username(post_data["author_id"])
    
    # Индексируем в Elasticsearch
    await es.index(
        index="posts_index",
        id=post_data["post_id"],
        document={
            "post_id": post_data["post_id"],
            "title": post_data["title"],
            "artist_names": [artist_name],
            "author_username": author_username,
            # ...
        }
    )
```

---

## 🚫 Что ЗАПРЕЩЕНО

### ❌ НЕЛЬЗЯ: Foreign Keys между БД
```sql
-- ПЛОХО! Content Service пытается сослаться на Auth Service
CREATE TABLE posts (
    id UUID PRIMARY KEY,
    author_id UUID NOT NULL,
    FOREIGN KEY (author_id) REFERENCES vibe_auth_db.users(id)  -- КАТАСТРОФА!
);
```

### ❌ НЕЛЬЗЯ: JOIN между сервисами
```python
# ПЛОХО! Content Service импортирует модель из Artist Service
from artist_service.models import Artist

posts = session.query(Post).join(Artist).all()  # ЗАПРЕЩЕНО!
```

### ✅ ПРАВИЛЬНО: Получение через API
```python
# ХОРОШО! Content Service запрашивает через HTTP
posts = await post_repo.get_all()
for post in posts:
    artist = await artist_client.get_artist(post.primary_artist_id)
    post.artist_name = artist["name"]
```

---

## 📋 Pydantic Schemas (DTO)

### Пример для Content Service:

```python
# app/schemas/post.py
from pydantic import BaseModel, UUID4, Field
from datetime import datetime
from typing import Optional

class PostBase(BaseModel):
    title: str = Field(..., max_length=255)
    description: Optional[str] = None
    content: str
    primary_artist_id: Optional[UUID4] = None

class PostCreate(PostBase):
    pass

class PostUpdate(BaseModel):
    title: Optional[str] = None
    content: Optional[str] = None
    status: Optional[str] = None

class PostResponse(PostBase):
    id: UUID4
    author_id: UUID4
    slug: str
    status: str
    views_count: int
    created_at: datetime
    published_at: Optional[datetime]
    
    class Config:
        from_attributes = True  # Для SQLAlchemy 2.0

class PostDetailResponse(PostResponse):
    """Агрегированный ответ с данными из других сервисов"""
    author_username: str           # Из Auth Service
    artist_name: Optional[str]     # Из Artist Service
    comments_count: int            # Из Comment Service
```

---

## 🎯 Следующие Шаги

1. **Урок 2:** Создашь базовую инфраструктуру (PostgreSQL, Kafka)
2. **Урок 3:** Создашь структуру директорий
3. **Урок 4:** Реализуешь первые модели в Auth Service
4. **Урок 5:** Напишешь миграции Alembic

---

## 📚 Полезные Ссылки

- [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/)
- [Pydantic V2 Documentation](https://docs.pydantic.dev/)
- [Database per Service Pattern](https://microservices.io/patterns/data/database-per-service.html)

---

**Дата:** 2025-11-30  
**Версия:** 1.0

---

> 💡 **Ключевая мысль:**  
> "Модели - это контракт между сервисом и его базой данных. Никто снаружи не должен знать о внутренностях."

