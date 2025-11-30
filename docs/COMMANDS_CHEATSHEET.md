# 📋 Шпаргалка по Командам VobeMusic

Коллекция полезных команд для работы с проектом.

---

## 📁 Навигация по Проекту

```bash
# Корень проекта
cd D:\projects\VobeMusic

# Перейти к конкретному сервису
cd apps/auth-service
cd apps/content-service
cd apps/artist-service

# Документация
cd docs/learning

# Инфраструктура
cd infrastructure/k8s
```

---

## 🐍 Python & Poetry

### Установка Poetry (Windows):
```powershell
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | python -
```

### Работа с зависимостями:
```bash
# Инициализация нового сервиса
poetry init

# Установка зависимостей
poetry install

# Добавить пакет
poetry add fastapi sqlalchemy

# Добавить dev-зависимость
poetry add --group dev pytest

# Активация виртуального окружения
poetry shell

# Запуск команды в venv
poetry run python main.py
poetry run uvicorn app.main:app --reload
```

---

## 🐳 Docker & Docker Compose

### Базовые команды:
```bash
# Проверка версии
docker --version
docker-compose --version

# Запуск всех сервисов
docker-compose up

# Запуск в фоне
docker-compose up -d

# Остановка
docker-compose down

# Пересборка образов
docker-compose build

# Просмотр логов
docker-compose logs -f [service_name]

# Просмотр запущенных контейнеров
docker ps

# Зайти внутрь контейнера
docker exec -it <container_id> bash
```

### Полезные команды для очистки:
```bash
# Удалить все остановленные контейнеры
docker container prune

# Удалить неиспользуемые образы
docker image prune

# Удалить неиспользуемые volumes
docker volume prune

# Полная очистка
docker system prune -a --volumes
```

---

## 🗄️ PostgreSQL

### Подключение к БД:
```bash
# Через Docker
docker exec -it <postgres_container> psql -U postgres

# Через psql (если установлен локально)
psql -h localhost -U postgres -d vibe_auth_db
```

### Базовые SQL команды:
```sql
-- Список баз данных
\l

-- Подключиться к базе
\c vibe_auth_db

-- Список таблиц
\dt

-- Описание таблицы
\d users

-- Выход
\q
```

### Создание баз для всех сервисов:
```sql
CREATE DATABASE vibe_auth_db;
CREATE DATABASE vibe_content_db;
CREATE DATABASE vibe_artist_db;
CREATE DATABASE vibe_comment_db;
CREATE DATABASE vibe_feedback_db;
CREATE DATABASE vibe_chat_db;
```

---

## 🔄 Alembic (Миграции)

### Инициализация:
```bash
# Внутри сервиса
cd apps/auth-service
alembic init migrations
```

### Работа с миграциями:
```bash
# Создать новую миграцию (автогенерация)
alembic revision --autogenerate -m "Create users table"

# Создать пустую миграцию
alembic revision -m "Add index to users"

# Применить миграции
alembic upgrade head

# Откатить на 1 шаг назад
alembic downgrade -1

# Откатить все
alembic downgrade base

# Посмотреть текущую версию
alembic current

# История миграций
alembic history
```

---

## ☸️ Kubernetes (K8s)

### Minikube:
```bash
# Старт кластера
minikube start

# Остановка
minikube stop

# Удаление
minikube delete

# Статус
minikube status

# Открыть Dashboard
minikube dashboard
```

### kubectl:
```bash
# Информация о кластере
kubectl cluster-info

# Список всех pods
kubectl get pods

# Список services
kubectl get services

# Список deployments
kubectl get deployments

# Детальная информация
kubectl describe pod <pod_name>

# Логи pod
kubectl logs <pod_name>

# Логи с follow
kubectl logs -f <pod_name>

# Зайти внутрь pod
kubectl exec -it <pod_name> -- bash

# Применить манифест
kubectl apply -f infrastructure/k8s/auth-deployment.yaml

# Удалить ресурс
kubectl delete -f infrastructure/k8s/auth-deployment.yaml

# Список всех ресурсов
kubectl get all

# Проброс портов
kubectl port-forward service/auth-service 8001:8001
```

---

## 📡 Kafka

### Kafka CLI (внутри контейнера):
```bash
# Зайти в контейнер Kafka
docker exec -it <kafka_container> bash

# Создать топик
kafka-topics --create --topic post.created \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# Список топиков
kafka-topics --list --bootstrap-server localhost:9092

# Описание топика
kafka-topics --describe --topic post.created \
  --bootstrap-server localhost:9092

# Consumer (читать сообщения)
kafka-console-consumer --topic post.created \
  --from-beginning \
  --bootstrap-server localhost:9092

# Producer (отправить сообщение)
kafka-console-producer --topic post.created \
  --bootstrap-server localhost:9092
```

---

## 🔍 Elasticsearch

### Подключение:
```bash
# Через curl
curl -X GET "localhost:9200"

# Список индексов
curl -X GET "localhost:9200/_cat/indices?v"

# Поиск
curl -X GET "localhost:9200/posts_index/_search?q=title:music"
```

### Python клиент:
```python
from elasticsearch import Elasticsearch

es = Elasticsearch(["http://localhost:9200"])

# Индексация документа
es.index(index="posts_index", id=1, document={"title": "My Post"})

# Поиск
es.search(index="posts_index", query={"match": {"title": "music"}})
```

---

## 🧪 Pytest

### Запуск тестов:
```bash
# Все тесты
pytest

# Конкретный файл
pytest tests/test_auth.py

# Конкретный тест
pytest tests/test_auth.py::test_login

# С выводом print
pytest -s

# С coverage
pytest --cov=app tests/

# Генерация HTML отчета
pytest --cov=app --cov-report=html tests/
```

---

## 🚀 Запуск Сервисов

### Локально (без Docker):
```bash
# Auth Service
cd apps/auth-service
poetry run uvicorn app.main:app --reload --port 8001

# Content Service
cd apps/content-service
poetry run uvicorn app.main:app --reload --port 8002

# Gateway Service
cd apps/gateway-service
poetry run uvicorn app.main:app --reload --port 8000
```

### Через Docker Compose:
```bash
# Все сервисы
docker-compose up

# Конкретный сервис
docker-compose up auth-service

# Пересобрать и запустить
docker-compose up --build
```

---

## 📊 Мониторинг

### Prometheus:
```bash
# Открыть Prometheus UI
http://localhost:9090

# Targets (проверка статуса сервисов)
http://localhost:9090/targets
```

### Grafana:
```bash
# Открыть Grafana UI
http://localhost:3001

# Логин по умолчанию: admin/admin
```

---

## 🔧 Git

### Базовые команды:
```bash
# Статус
git status

# Добавить файлы
git add .
git add docs/learning/01_*.md

# Коммит
git commit -m "feat: Add Auth Service"

# Пуш
git push origin main

# Создать ветку
git checkout -b feature/auth-service

# Переключиться на ветку
git checkout main

# Слияние
git merge feature/auth-service

# Логи
git log --oneline

# Откатить изменения
git restore <file>
```

### Conventional Commits:
```bash
# Feature
git commit -m "feat: Add user registration"

# Fix
git commit -m "fix: Resolve JWT token validation"

# Documentation
git commit -m "docs: Update architecture guide"

# Refactoring
git commit -m "refactor: Improve database session handling"

# Tests
git commit -m "test: Add tests for auth endpoints"

# Chore
git commit -m "chore: Update dependencies"
```

---

## 🛠️ Полезные Скрипты (будем создавать)

### Windows PowerShell:
```powershell
# Запустить все миграции
./scripts/migrate_all.ps1

# Seed данных
python scripts/seed_data.py

# Проверка здоровья всех сервисов
python scripts/health_check.py

# Создать новый сервис
./scripts/create_service.ps1 -name "notification-service"
```

---

## 🔎 Отладка

### Логи FastAPI:
```bash
# Запуск с debug режимом
uvicorn app.main:app --reload --log-level debug

# Просмотр логов Docker контейнера
docker logs -f <container_id>

# Логи с фильтром
docker logs <container_id> 2>&1 | grep "ERROR"
```

### Python Debugger:
```python
# В коде
import pdb; pdb.set_trace()

# Или
breakpoint()
```

---

## 📦 Makefile (будем создавать)

Создадим удобные алиасы:

```makefile
.PHONY: dev prod test clean

dev:
	docker-compose up

dev-build:
	docker-compose up --build

prod:
	docker-compose -f docker-compose.prod.yml up -d

test:
	pytest tests/ -v

clean:
	docker-compose down -v
	docker system prune -f

migrate:
	./scripts/migrate_all.sh

seed:
	python scripts/seed_data.py
```

Использование:
```bash
make dev        # Запуск в dev режиме
make test       # Запуск тестов
make clean      # Очистка
```

---

## 📝 Быстрые Ссылки

### Локальные сервисы (после запуска):
- **Gateway:** http://localhost:8000
- **Auth Service:** http://localhost:8001
- **Content Service:** http://localhost:8002
- **Artist Service:** http://localhost:8003
- **Frontend:** http://localhost:3000

### Swagger UI (API Docs):
- **Gateway:** http://localhost:8000/docs
- **Auth:** http://localhost:8001/docs
- **Content:** http://localhost:8002/docs

### Инфраструктура:
- **PostgreSQL:** localhost:5432
- **Redis:** localhost:6379
- **Kafka:** localhost:9092
- **Elasticsearch:** localhost:9200
- **Prometheus:** localhost:9090
- **Grafana:** localhost:3001

---

## 🆘 Частые Проблемы

### Порт занят:
```bash
# Windows: Найти процесс на порту
netstat -ano | findstr :8000

# Убить процесс
taskkill /PID <PID> /F
```

### Docker проблемы:
```bash
# Перезапуск Docker Desktop
# Через GUI или:
net stop com.docker.service
net start com.docker.service
```

### Poetry проблемы:
```bash
# Пересоздать venv
poetry env remove python
poetry install
```

---

## 📚 Дополнительные Ресурсы

- [FastAPI Docs](https://fastapi.tiangolo.com/)
- [Docker Docs](https://docs.docker.com/)
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [Poetry Docs](https://python-poetry.org/docs/)

---

**Дата:** 2025-11-30  
**Версия:** 1.0

---

> 💡 **Совет:**  
> Добавь эту страницу в закладки — она понадобится на протяжении всего проекта!

