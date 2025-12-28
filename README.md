# 🎵 VibeMusic — Микросервисная Музыкальная Платформа

[![Architecture](https://img.shields.io/badge/architecture-microservices-blue)]()
[![Status](https://img.shields.io/badge/status-in_development-yellow)]()

---

## 📖 Описание

**VibeMusic** — современная музыкальная платформа на **микросервисной архитектуре**.

### Ключевые Особенности:
- 🏗️ **9 целевых микросервисов** (Hybrid-first approach)
- 📡 **Event-Driven Architecture** (Internal bus → Kafka)
- 🗄️ **Schema per Service** (Phase 0), Database per Service (Phase 3)
- ☁️ **Cloud-Native** (Docker, Kubernetes)
- 🔍 **Полнотекстовый поиск** (Postgres FTS → Elasticsearch)

---

## 🏗️ Архитектура

### Target Microservices (Phase 3)

| # | Сервис | Порт | Описание |
|---|--------|------|----------|
| 1 | **gateway-service** | 8000 | API Gateway, JWT, routing |
| 2 | **auth-service** | 8001 | Аутентификация, пользователи |
| 3 | **content-service** | 8002 | Посты, контент |
| 4 | **artist-service** | 8003 | Артисты, альбомы, треки |
| 5 | **comment-service** | 8004 | Комментарии |
| 6 | **feedback-service** | 8005 | Обратная связь |
| 7 | **llm-chat-service** | 8006 | AI-чат |
| 8 | **search-service** | 8007 | Поиск |
| 9 | **frontend-service** | 3000 | UI |

### Диаграмма

```
┌─────────────────────────────────────────────────────────────┐
│                         NGINX                                │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                     API Gateway                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
    ┌──────────┬───────────┼───────────┬──────────┐
    │          │           │           │          │
┌───▼───┐  ┌───▼───┐  ┌────▼────┐  ┌───▼───┐  ┌───▼───┐
│Identity│  │Catalog│  │ Social  │  │Playlist│  │ Media │
│ :8001  │  │ :8002 │  │ :8003   │  │ :8004  │  │ :8005 │
└───┬────┘  └───┬───┘  └────┬────┘  └───┬────┘  └───┬───┘
    │           │           │           │           │
    └───────────┴───────────┴───────────┴───────────┘
                            │
                    ┌───────▼───────┐
                    │     Kafka     │
                    └───────────────┘
```

---

## 🛠️ Технологический Стек

### Backend
- **Framework:** FastAPI
- **Python:** 3.11+
- **ORM:** SQLAlchemy 2.0
- **Validation:** Pydantic v2

### Infrastructure
- **Databases:** PostgreSQL (per service)
- **Messaging:** Kafka
- **Cache:** Redis
- **Files:** MinIO (S3-compatible)
- **Search:** Elasticsearch

### DevOps
- **Containers:** Docker
- **Orchestration:** Kubernetes
- **CI/CD:** GitHub Actions

---

## 📁 Структура Проекта

```
vibemusic/
├── apps/
│   ├── gateway-service/
│   ├── auth-service/
│   ├── content-service/
│   ├── artist-service/
│   ├── comment-service/
│   ├── feedback-service/
│   ├── llm-chat-service/
│   ├── search-service/
│   └── frontend-service/
├── shared/
│   ├── contracts/          # Events, errors
│   └── libs/               # Utilities
├── infra/
│   ├── docker/
│   ├── nginx/
│   ├── kafka/
│   ├── monitoring/
│   └── k8s/
└── docs/
```

---

## 🚀 Быстрый Старт

### Docker Compose

```bash
# Клонирование
git clone https://github.com/Artemis1992/FastApi-VibeMusic.git
cd FastApi-VibeMusic

# Запуск
docker-compose up -d

# Проверка
curl http://localhost:8000/health
```

### Swagger UI

- **Gateway:** http://localhost:8000/docs
- **Auth:** http://localhost:8001/docs
- **Content:** http://localhost:8002/docs

---

## 🚨 Главные Правила

### No Direct Imports

```python
# ❌ ЗАПРЕЩЕНО
from identity_app.db.models import User

# ✅ РАЗРЕШЕНО
from catalog_app.integrations.http_clients import IdentityClient
```

### Database per Service

- Каждый сервис имеет свою БД
- Нет FK между сервисами
- Данные через HTTP/Kafka

---

## 📚 Документация

- **[📖 START HERE](docs/START_HERE.md)** — Путеводитель
- **[📋 Standards](docs/PROJECT_STANDARDS.md)** — Стандарты
- **[🏗️ Architecture](docs/ARCHITECTURE_OVERVIEW.md)** — Архитектура
- **[📊 Data Models](docs/DATA_MODELS.md)** — Модели данных
- **[🌐 API Guide](docs/API_GUIDE.md)** — Правила API
- **[📁 ADR](docs/ADR/)** — Архитектурные решения

---

## 📡 Kafka Events

```
vibemusic.identity.user       → user.registered, user.deleted
vibemusic.catalog.track       → track.created, track.deleted
vibemusic.social.like         → like.created
vibemusic.media.file          → file.uploaded
```

---

## 📊 Статус

```
Services:     [░░░░░░░░░░░░░░░░░░░░] 0/9
Documentation [████████████████████] 100%
Phase:        0 (Hybrid-first)
```

---

## 👨‍� Автор

- **Author:** Artemis1992
- **Architecture:** Microservices

---

**🎵 VibeMusic — музыка будущего!**
