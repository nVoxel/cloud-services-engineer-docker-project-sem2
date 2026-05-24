# Momo Store

Магазин пельменей. Backend — Go REST API, frontend — Vue 3 SPA.

## Быстрый старт

```bash
# Скопировать конфиг и секрет
cp .env.example .env
cp secrets/app_secret.txt.example secrets/app_secret.txt

# Собрать и запустить (prod-режим)
docker compose -f docker-compose.yml up -d --build

# Открыть в браузере
# http://localhost/momo-store/
```

## Команды

| Команда | Описание |
|---------|----------|
| `docker compose up -d --build` | Запустить в dev-режиме (override применяется автоматически) |
| `docker compose -f docker-compose.yml up -d --build` | Запустить в prod-режиме |
| `docker compose down` | Остановить |
| `docker compose logs -f` | Смотреть логи |
| `docker compose ps` | Статус сервисов |
| `docker compose up -d --scale backend=3` | Горизонтальное масштабирование бэкенда |

## Архитектура

```
Prod-режим:
  Браузер → :80 (host) → frontend nginx :8080
               ├─ /momo-store/**  →  Vue SPA (статика)
               └─ /api/**         →  backend:8081 (proxy, round-robin)
                                       [backend replica 1]
                                       [backend replica 2]
                                       [backend replica N]

Dev-режим (override):
  Браузер → :80  → frontend nginx (статика + proxy)
  Браузер → :8081 → backend (прямой доступ для отладки)

Сети:
  frontend-net  — только frontend (публичная)
  backend-net   — frontend + backend (internal: true, нет доступа в интернет)
```

В prod backend **не публикует** порт на хост — доступен только через nginx proxy (`/api/`).
В dev override автоматически добавляет `ports: "8081:8081"` для прямого доступа.

## Образы

### Backend

| Стадия | Образ | Размер |
|--------|-------|--------|
| Builder | `golang:1.17-alpine` | ~300 MB (не входит в финальный образ) |
| Runtime | `alpine:3.19` | ~8 MB base |
| **Итого** | | **~20 MB** |

### Frontend

| Стадия | Образ | Размер |
|--------|-------|--------|
| Builder | `node:18-alpine` | ~170 MB (не входит в финальный образ) |
| Runtime | `nginxinc/nginx-unprivileged:stable-alpine` | ~25 MB base |
| **Итого** | | **~35 MB** |

## Переменные окружения

| Переменная | По умолчанию | Описание |
|-----------|-------------|---------|
| `FRONTEND_PORT` | `80` | Порт frontend на хосте |
| `BACKEND_PORT` | `8081` | Порт backend на хосте |
| `VUE_APP_API_URL` | `/api` | Базовый URL API, бакуется в SPA при сборке |

## Горизонтальное масштабирование

Backend stateless — масштабируется без изменений.

```bash
docker compose -f docker-compose.yml up -d --scale backend=3
```

nginx в frontend перерезолвит `backend:8081` через Docker DNS (`127.0.0.11`) и распределит запросы по всем репликам (round-robin). Директива `resolver 127.0.0.11 valid=10s` в nginx.conf обеспечивает динамическое обнаружение новых реплик.

В prod-compose backend не публикует хост-порт, поэтому конфликта при масштабировании нет. В dev-override порт 8081 добавляется только для первой реплики (для отладки).

После масштабирования нужно обновить DNS в nginx, чтобы он увидел новые реплики:

```bash
docker compose -f docker-compose.yml exec frontend nginx -s reload
```

## Безопасность

### Секреты

Секреты передаются через Docker Secrets (файловый механизм), а не через переменные окружения:

```yaml
secrets:
  app_secret:
    file: ./secrets/app_secret.txt
```

Секрет монтируется в контейнер как файл `/run/secrets/app_secret`. В production надо заменить `secrets/app_secret.txt` реальным секретом или использовать внешний провайдер (Vault, AWS Secrets Manager).

### Локальное сканирование с Trivy

```bash
# Установка (macOS)
brew install trivy

# Сканировать образы после сборки
trivy image momo-store-backend:latest
trivy image momo-store-frontend:latest

# Только CRITICAL
trivy image --severity CRITICAL momo-store-backend:latest

# Сохранить отчёт в JSON
trivy image --format json --output report.json momo-store-backend:latest
```

## Dev-режим

`docker-compose.override.yml` применяется автоматически при `docker compose up` и:
- Отключает `read_only` для удобства отладки
- Переключает `VUE_APP_API_URL` на прямой вызов backend (`http://localhost:8081`)
- Смягчает условие `depends_on` до `service_started`

```bash
# Dev (override применяется автоматически)
docker compose up -d --build

# Prod (только основной файл)
docker compose -f docker-compose.yml up -d --build
```
