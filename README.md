# Проектная работа дисциплины «Docker-контейнеризация и хранение данных»

Приложение состоит из двух частей:

- **backend** — REST API на Go (momo-store);
- **frontend** — SPA на Vue.js, раздаётся через nginx.

Обе части упакованы в Docker-образы, оркестрация выполняется через Docker Compose, сборка и проверка автоматизированы в GitHub Actions.

## Запуск проекта

### Требования

- Docker Engine 24+
- Docker Compose v2
- Свободные порты: `80` (фронтенд), `8081` (бэкенд)

### CI/CD (GitHub Actions)

Настроен воркфлоу для GitHub Actions при коммите. Пайплайн .github/workflows/deploy.yaml запускается на push в main и состоит из двух job'ов:

1. build_and_push_to_docker_hub
Сборка образов бэкенда и фронтенда.

Пуш в DockerHub с двумя тегами: latest и SHA коммита (для трассировки и отката).

Сканирование обоих образов через Trivy. Пайплайн падает при обнаружении CRITICAL или HIGH уязвимостей с доступным фиксом.

2. run-with-docker-compose

Запуск стека через docker compose up -d --build.

Проверка статуса контейнеров (docker compose ps).

Проверка здоровья бэкенда (/health).

Проверка доступности фронтенда (curl http://localhost:80).

Логи контейнеров при падении.

Корректная остановка (docker compose down -v).


## Docker-образы
### Backend (backend/Dockerfile)
**Multi-stage сборка:**

Этап сборки: golang:1.25-alpine - компиляция статического бинарника с флагами -ldflags="-s -w" для уменьшения размера.

Этап выполнения: alpine:3.22 - минимальный рантайм, только бинарник и сертификаты.

Итоговый размер: 10.34 MB

### Frontend (frontend/Dockerfile)
**Multi-stage сборка:**

Этап сборки: node:20-alpine - установка зависимостей и сборка Vue-приложения.

Этап выполнения: nginx:alpine - раздача статики, минимальный образ.

Итоговый размер: 28.27 MB

## Переменные окружения

Данные переменные передаются через GitHub Secrets

```
DOCKER_USER
DOCKER_PASSWORD
```

## Безопасность

### Trivy

Добавлен этап проверки безопасности посредством Trivy. Проверки уровня CRITICAL и HIGH пройдены на момент разработки

### Контейнеры
Непривилегированный пользователь в бэкенде (appuser, UID 1001).

cap_drop: ALL с добавлением только необходимых capabilities:

backend: NET_BIND_SERVICE

frontend: CHOWN, SETUID, SETGID, NET_BIND_SERVICE

read_only: true - файловая система контейнера только для чтения.

tmpfs для временных данных:

backend: /tmp

frontend: /var/cache/nginx, /var/run, /tmp

### Ограничения ресурсов:

backend: 0.50 CPU, 256 МБ памяти

frontend: 0.25 CPU, 128 МБ памяти

Политика перезапуска: unless-stopped.
