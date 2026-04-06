# Практическое задание 7
## Шишков А.Д. ЭФМО-02-25
## Тема
Написание Dockerfile и сборка контейнера.

## Цель
Освоить контейнеризацию backend-приложения на Go с помощью Docker, научиться писать Dockerfile, собирать Docker-образ и запускать контейнеризированный сервис в воспроизводимой среде.

---

## 1. Структура проекта

```
services/tasks/
  cmd/tasks/main.go           — точка входа, HTTP-сервер
  internal/http/handler.go    — обработчики (CRUD + /health)
  internal/service/task.go    — бизнес-логика
  internal/repository/        — PostgreSQL-репозиторий
  Dockerfile                  — multi-stage сборка
  .dockerignore               — исключения для build context
deploy/
  docker-compose.yml          — оркестрация tasks + PostgreSQL
```

Эндпоинт `/health` добавлен для проверки работоспособности контейнера:

```go
func (h *Handler) handleHealth(w http.ResponseWriter, r *http.Request) {
    h.respondJSON(w, http.StatusOK, map[string]string{
        "status":  "ok",
        "service": "tasks",
    })
}
```

---

## 2. Dockerfile (multi-stage build)

Файл `services/tasks/Dockerfile`:

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /app/bin/tasks ./services/tasks/cmd/tasks

FROM alpine:3.20

RUN apk --no-cache add ca-certificates

WORKDIR /app

COPY --from=builder /app/bin/tasks /app/tasks

EXPOSE 8082

ENV TASKS_PORT=8082

CMD ["/app/tasks"]
```

### Описание этапов

**Этап 1 — builder (`golang:1.23`)**:
- Копирует `go.mod` и `go.sum`, загружает зависимости (кэшируется Docker)
- Копирует весь исходный код
- Компилирует статический бинарник (`CGO_ENABLED=0`) для Linux/amd64

**Этап 2 — runtime (`alpine:3.20`)**:
- Минимальный образ (~7 МБ) без компилятора и исходников
- Копирует только готовый бинарник из builder
- Объявляет порт 8082 и задаёт переменные окружения
- Запускает приложение

**Преимущества multi-stage build:**
- Финальный образ ~15–20 МБ вместо ~1 ГБ (golang-образ)
- Нет исходного кода и инструментов сборки в production-образе
- Уменьшенная поверхность атаки

---

## 3. .dockerignore

Файл `services/tasks/.dockerignore`:

```
.git
bin
tmp
*.log
.idea
.vscode
Dockerfile.old
.env
```

Корневой `.dockerignore`:

```
.git
.idea
.vscode
*.exe
*.log
.env
deploy/tls/certs/
docs/
```

**Назначение:** исключает из build context файлы, не нужные для сборки — уменьшает размер контекста и ускоряет передачу в Docker daemon.

---

## 4. Docker Compose

Файл `deploy/docker-compose.yml`:

```yaml
version: "3.9"

services:
  db:
    image: postgres:15-alpine
    container_name: tasks_db
    environment:
      POSTGRES_USER: tasks
      POSTGRES_PASSWORD: tasks
      POSTGRES_DB: tasks
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tasks"]
      interval: 3s
      timeout: 3s
      retries: 10

  tasks:
    build:
      context: ..
      dockerfile: services/tasks/Dockerfile
    container_name: tasks_service
    ports:
      - "8082:8082"
    environment:
      TASKS_PORT: "8082"
      DATABASE_URL: "postgres://tasks:tasks@db:5432/tasks?sslmode=disable"
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

**Описание:**
- **db** — PostgreSQL 15, healthcheck через `pg_isready`, данные в named volume `pgdata`
- **tasks** — собирается из Dockerfile, стартует после готовности БД
- Порт 8082 пробрасывается на хост, переменные окружения задают конфигурацию

---

## 5. Сборка и запуск

### Сборка образа вручную

```bash
cd deploy
docker compose build tasks
```

Или напрямую:

```bash
docker build -t techip-tasks:0.1 -f services/tasks/Dockerfile .
```

Проверить образ:

```bash
docker images | grep techip-tasks
```

### Запуск одного контейнера

```bash
docker run --rm -p 8082:8082 -e TASKS_PORT=8082 techip-tasks:0.1
```

### Запуск через Docker Compose

```bash
cd deploy
docker compose up -d --build
```

Проверить статус:

```bash
docker compose ps
```

Логи:

```bash
docker compose logs -f
```

### Проверка работоспособности

```bash
curl http://localhost:8082/health
```

Ожидаемый ответ:

```json
{"status":"ok","service":"tasks"}
```

### Остановка

```bash
cd deploy
docker compose down
```

---

## 6. Переменные окружения

| Переменная | Значение по умолчанию | Описание |
|---|---|---|
| `TASKS_PORT` | `8082` | Порт HTTP-сервера |
| `DATABASE_URL` | `postgres://tasks:tasks@localhost:5432/tasks?sslmode=disable` | DSN подключения к PostgreSQL |
| `AUTH_MODE` | `http` | Режим авторизации (http/grpc) |
| `AUTH_BASE_URL` | `http://localhost:8081` | URL Auth-сервиса |

Порт задаётся через env, а не хардкодится:

```go
port := os.Getenv("TASKS_PORT")
if port == "" {
    port = "8082"
}
```

В Dockerfile `EXPOSE 8082` лишь документирует порт — фактический порт определяется переменной `TASKS_PORT` и флагом `-p` при запуске.

---

## 7. Демонстрация

Проверка работоспособности контейнера через эндпоинт `/health`:

<img width="478" height="150" alt="image" src="https://github.com/user-attachments/assets/accac27f-efb1-4e67-844c-83d38215ed9b" /> 



