# Проектная работа дисциплины «Docker-контейнеризация и хранение данных»

Приложение состоит из двух частей:

- **backend** — REST API на Go (momo-store);
- **frontend** — SPA на Vue.js, раздаётся через nginx.

Обе части упакованы в Docker-образы, оркестрация выполняется через Docker Compose, сборка и проверка автоматизированы в GitHub Actions.

## Архитектура
Клиент идёт на proxy:80 (nginx-unprivileged)

┌────┴─────┐

frontend × N backend × N

Наружу опубликован **только proxy** (порт 80). Backend и frontend доступны
исключительно внутри изолированной сети `app-network`.

### Запуск проекта

### Требования

- Docker Engine 24+
- Docker Compose v2
- Свободные порты: `80` (proxy)

### Быстрый старт

```bash
# 1. Создать файл секрета (не коммитится в Git)
mkdir -p secrets
echo "demo-api-key-12345" > secrets/api_key.txt
chmod 600 secrets/api_key.txt

# 2. Запустить стек с репликами
docker compose up -d --scale backend=2 --scale frontend=2 --build

# 3. Проверить статус
docker compose ps
```

После запуска приложение доступно на http://localhost/

### Остановка

```bash
# Остановить, сохранив volumes
docker compose down

# Остановить и удалить volumes
docker compose down -v
```

### CI/CD (GitHub Actions)

Настроен воркфлоу для GitHub Actions при коммите. Пайплайн .github/workflows/deploy.yaml запускается на push в main и состоит из двух job'ов:

`1. build_and_push_to_docker_hub`
Сборка образов бэкенда и фронтенда.

Пуш в DockerHub с двумя тегами: latest и SHA коммита (для трассировки и отката).

Сканирование обоих образов через Trivy. Пайплайн падает при обнаружении CRITICAL или HIGH уязвимостей с доступным фиксом.

`2. run-with-docker-compose`

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
API_KEY_SECRET
```

## Проверка балансировки

```
# 10 запросов к /health через proxy
for i in {1..10}; do
  curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost/health
done

# Распределение по репликам
docker logs cloud-services-engineer-docker-project-sem2-backend-1 2>&1 | grep -c "/health"
docker logs cloud-services-engineer-docker-project-sem2-backend-2 2>&1 | grep -c "/health"
```

## Volumes

Приложение stateless — persistent data не требуется. Настроены named volumes для логов:

| Volume | Mount point | Назначение |
|--------|-------------|------------|
| backend-logs |	/app/logs | Логи бэкенда |
| frontend-logs	| /var/log/nginx | Логи nginx |


## Secrets
Проект использует Docker Secrets для управления конфиденциальными данными.

| Имя | Назначение | Путь в контейнере |
|-----|------------|-------------------|
| api_key_secret | Демонстрационный API-ключ | /run/secrets/api_key_secret |

Ключ передаётся через GitHub Secret `API_KEY_SECRET`

## Healthchecks
Для всех сервисов настроены healthcheck'и:

```
backend: wget --spider http://127.0.0.1:8081/health

frontend: wget --spider http://127.0.0.1:8080/

proxy: wget --spider http://127.0.0.1:8080/health
```


## Непривилегированные пользователи

Для работы backend используется пользователь appuser, для фронтенде nginx

## Trivy

Добавлен этап проверки безопасности посредством Trivy. Проверки уровня CRITICAL и HIGH пройдены на момент разработки

## Контейнеры
Непривилегированный пользователь в бэкенде (appuser, UID 1001).

cap_drop: ALL с добавлением только необходимых capabilities:

backend: NET_BIND_SERVICE

frontend: CHOWN, SETUID, SETGID, NET_BIND_SERVICE

read_only: true - файловая система контейнера только для чтения.

tmpfs для временных данных:

backend: /tmp

frontend: /var/cache/nginx, /var/run, /tmp

## Ограничения ресурсов:

backend: 0.50 CPU, 256 МБ памяти

frontend: 0.25 CPU, 128 МБ памяти

Политика перезапуска: unless-stopped.

