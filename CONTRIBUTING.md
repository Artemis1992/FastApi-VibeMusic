# 📖 Руководство по Контрибуции (CONTRIBUTING)

**Версия:** 1.0  
**Дата:** 2025-12-28

---

## 🚀 Быстрый Старт

### 1. Клонирование и Настройка

```bash
# Клонирование
git clone https://github.com/Artemis1992/FastApi-VibeMusic.git
cd FastApi-VibeMusic

# Виртуальное окружение
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Зависимости
pip install -r requirements.txt
```

### 2. Запуск Инфраструктуры

Инфраструктура описана в `infra/` (Docker/K8s/Kafka/NGINX/monitoring) и в документации:
- [docs/LOCAL_DEV.md](docs/LOCAL_DEV.md)

> В текущем состоянии репозитория `docker-compose.yml` ещё не добавлен — PR’ы на инфраструктурный стенд приветствуются.

### 3. Запуск Приложения

```bash
# Локальный запуск одного сервиса (после того как сервис содержит FastAPI entrypoint)
# Пример (ожидаемая структура): apps/catalog-service/catalog_app/main.py
cd apps/catalog-service
uvicorn catalog_app.main:app --reload --port 8002
```

### 4. Тесты

```bash
# Все тесты
pytest

# Конкретный сервис
pytest apps/catalog-service/tests/

# С покрытием
pytest --cov=apps --cov-report=html
```

---

## 📁 Структура Проекта

См. [docs/PROJECT_STANDARDS.md](docs/PROJECT_STANDARDS.md) для полной структуры.

Краткая версия:
```
apps/
├── gateway-service/      # API Gateway (edge routing/auth)
├── auth-service/         # Аутентификация
├── content-service/      # Контент (посты)
├── artist-service/       # Артисты, альбомы, треки
├── comment-service/      # Комментарии
├── feedback-service/     # Обратная связь
├── llm-chat-service/     # AI-чат
├── search-service/       # Поиск
└── frontend-service/     # UI

shared/
├── contracts/            # Контракты событий
└── libs/                 # Утилиты

infra/
├── docker/               # Docker assets (compose, Dockerfiles, etc.)
├── nginx/                # NGINX конфиги
├── kafka/                # Kafka/Redpanda assets
├── monitoring/           # Prometheus/Grafana assets
└── k8s/                  # Kubernetes manifests
```

---

## 🔀 Workflow Разработки

### Git Flow

```
main (production)
  └── develop (staging)
        └── feature/XXX-description
        └── fix/XXX-description
        └── refactor/XXX-description
```

### Именование Веток

```
feature/123-add-track-likes     # Новая функциональность
fix/456-fix-auth-token          # Исправление бага
refactor/789-optimize-search    # Рефакторинг
docs/update-api-guide           # Документация
```

### Процесс

1. **Создай ветку** от `develop`
   ```bash
   git checkout develop
   git pull
   git checkout -b feature/123-add-track-likes
   ```

2. **Разработай** с частыми коммитами
   ```bash
   git commit -m "feat(catalog): add like endpoint"
   ```

3. **Проверь** перед PR
   ```bash
   # Линтер
   ruff check .
   
   # Форматирование
   ruff format .
   
   # Тесты
   pytest
   ```

4. **Создай Pull Request** в `develop`

---

## 📝 Правила Коммитов

### Формат

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Типы

| Тип | Описание |
|-----|----------|
| `feat` | Новая функциональность |
| `fix` | Исправление бага |
| `refactor` | Рефакторинг (без изменения поведения) |
| `docs` | Документация |
| `test` | Тесты |
| `chore` | Настройка, зависимости |
| `style` | Форматирование |

### Scope (опционально)

```
catalog, identity, social, playlists, media, search, shared, infra
```

### Примеры

```bash
feat(catalog): add track creation endpoint
fix(identity): fix JWT token expiration
refactor(social): extract like service
docs: update API guide
test(catalog): add track service tests
chore: update dependencies
```

---

## 🧪 Тестирование

### Структура Тестов

```
tests/
├── conftest.py           # Фикстуры
├── test_api/             # API тесты
│   └── test_tracks.py
├── test_service/         # Service тесты
│   └── test_track_service.py
└── test_repo/            # Repo тесты
    └── test_track_repo.py
```

### Правила

1. **Каждый слой тестируется отдельно**
   - `test_api/` — HTTP endpoints (используй `TestClient`)
   - `test_service/` — бизнес-логика (мокай repo)
   - `test_repo/` — база данных (используй test DB)

2. **Покрытие минимум 70%**

3. **Naming convention**
   ```python
   def test_create_track_success():
   def test_create_track_without_album_fails():
   def test_get_track_not_found_returns_404():
   ```

### Пример

```python
# tests/test_api/test_tracks.py

import pytest
from fastapi.testclient import TestClient

class TestTrackAPI:
    def test_create_track_success(self, client: TestClient, auth_headers: dict):
        response = client.post(
            "/api/v1/catalog/tracks",
            json={"title": "Test Track", "album_id": "...", "duration": 180},
            headers=auth_headers,
        )
        assert response.status_code == 201
        assert response.json()["title"] == "Test Track"
    
    def test_create_track_without_auth_returns_401(self, client: TestClient):
        response = client.post("/api/v1/catalog/tracks", json={...})
        assert response.status_code == 401
```

---

## 🎨 Стиль Кода

### Python

- **Formatter:** Ruff (или Black)
- **Linter:** Ruff
- **Type Hints:** Обязательны
- **Docstrings:** Google style

### Настройки Ruff

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W", "UP", "B"]
```

### Примеры

```python
# ✅ ПРАВИЛЬНО
async def get_track_by_id(self, track_id: UUID) -> Track | None:
    """Получить трек по ID.
    
    Args:
        track_id: UUID трека
        
    Returns:
        Track или None если не найден
    """
    ...

# ❌ НЕПРАВИЛЬНО
async def get_track(id):  # Нет типов!
    ...
```

---

## 📋 Checklist для PR

Перед созданием PR убедись:

### Код
- [ ] Соблюдён ZERO-BYPASS (нет прямых импортов между сервисами)
- [ ] Type hints везде
- [ ] Docstrings для публичных функций
- [ ] Нет TODO/FIXME без тикета

### Тесты
- [ ] Тесты написаны
- [ ] Тесты проходят (`pytest`)
- [ ] Покрытие не упало

### Качество
- [ ] Линтер пройден (`ruff check .`)
- [ ] Форматирование (`ruff format .`)
- [ ] Нет секретов в коде

### БД
- [ ] Миграция создана (если менялась схема)
- [ ] Миграция обратима

### Документация
- [ ] OpenAPI обновлён
- [ ] README/docs обновлены (если нужно)

---

## 🔍 Code Review

### Что Проверяем

1. **Архитектура**
   - ZERO-BYPASS соблюдён?
   - Слои не нарушены?

2. **Код**
   - Понятные названия?
   - Нет дублирования?
   - Обработка ошибок?

3. **Тесты**
   - Покрытие достаточное?
   - Edge cases?

4. **Безопасность**
   - Нет SQL injection?
   - Нет утечек данных?

### Как Давать Фидбек

```markdown
# Обязательно исправить
🔴 BLOCKER: Прямой импорт модели из другого сервиса

# Рекомендация
🟡 SUGGESTION: Можно вынести в отдельную функцию

# Вопрос/уточнение
❓ QUESTION: Почему выбран этот подход?

# Похвала
💚 NICE: Отличное решение!
```

---

## 🚨 Что Делать Если

### Тесты Падают

```bash
# Проверь логи
pytest -v --tb=long

# Запусти конкретный тест
pytest tests/test_api/test_tracks.py::TestTrackAPI::test_create_track_success -v
```

### Линтер Ругается

```bash
# Автофикс
ruff check . --fix

# Форматирование
ruff format .
```

### Миграция Не Работает

```bash
# Откат
alembic downgrade -1

# Проверь SQL
alembic upgrade head --sql
```

---

## 📞 Контакты

- **GitHub Issues:** для багов и фич
- **Pull Requests:** для code review

---

**Спасибо за вклад в проект!** 🎵
