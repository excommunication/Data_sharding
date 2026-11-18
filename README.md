# Data sharding

## О проекте

Учебный магазин из нескольких сервисов. **store** — REST API и веб-интерфейс магазина: клиенты, поставщики, товары, изображения и адреса хранятся в PostgreSQL. Авторизацией занимается отдельный сервис **auth-service**, который store вызывает по gRPC.

## Стек

| Компонент | Технологии |
|---|---|
| store | Java 21, Spring Boot 4.1.1, Gradle (wrapper 9.6.0), Spring Web MVC, Spring Data JPA, Flyway, gRPC-клиент, MapStruct 1.6.3, Lombok, springdoc-openapi 3.1.1 |
| auth-service | Java 21, Spring Boot 4.1.1, Maven, gRPC-сервер (spring-grpc), Spring Security, Spring Data JPA, Flyway, JWT (jjwt), Protobuf |
| Контракты | Protobuf (`src/contracts/proto`) |
| Инфраструктура | PostgreSQL 17, Docker, Docker Compose |

## Быстрый старт

```powershell
docker compose up --build
```

Порядок запуска: postgres (healthcheck) → auth-service → store. При первом запуске postgres создаёт базы `store_db` и `auth_db`, а Flyway в каждом сервисе создаёт свои таблицы.

| Что | Адрес |
|---|---|
| Веб-интерфейс магазина | http://localhost:8080/ |
| Swagger UI | http://localhost:8080/swagger/index.html |
| OpenAPI JSON | http://localhost:8080/api-docs |
| auth-service (gRPC) | `localhost:9090` |
| PostgreSQL | `localhost:5432` (admin / admin) |

Остановка: `docker compose down` (с удалением данных: `docker compose down -v`).

## Настройки

Переменные окружения заданы в `docker-compose.yml`. Для входа через Google нужны настоящие `CLIENT_ID` и `CLIENT_SECRET`. Их можно передать через файл `.env` в корне проекта (он не попадает в git):

```
CLIENT_ID=ваш-client-id
CLIENT_SECRET=ваш-client-secret
REDIRECT_URI=http://localhost:8080/api/v1/auth/login/google/callback
SECRET=длинная-строка-не-короче-32-байт
EXPIRATION_TIME=900000
```

Без них сервисы запускаются с заглушками, а вход через Google работать не будет.

## Проверка функционала

1. **Веб-интерфейс.** Откройте http://localhost:8080/ — отображаются товары из тестовых данных миграции `V2`.

2. **REST API** (тестовые данные создаёт миграция `V2`):

```powershell
curl http://localhost:8080/api/v1/products
curl http://localhost:8080/api/v1/clients
curl http://localhost:8080/api/v1/suppliers
```

3. **Авторизация.** Регистрация и вход выполняются через store, который обращается к auth-service по gRPC:

```powershell
curl -X POST http://localhost:8080/api/v1/auth/register -H "Content-Type: application/json" -d "{\"email\":\"test@test.ru\",\"name\":\"Ivan\",\"surname\":\"Petrov\",\"number\":\"89123456789\",\"password\":\"Password123-\"}"
curl -X POST http://localhost:8080/api/v1/auth/login -H "Content-Type: application/json" -d "{\"email\":\"test@test.ru\",\"password\":\"Password123-\"}"
```

4. **Логи.**

```powershell
docker compose logs -f store
docker compose logs -f auth-service
```

## REST API store

| Группа | Маршруты |
|---|---|
| `/api/v1/auth` | `POST /login`, `POST /register`, `POST /reset`, `POST /validate`, `POST /change-password`, `GET /login/google` |
| `/api/v1/clients` | `GET`, `GET /{id}`, `GET /search`, `POST`, `DELETE /{id}`, `PATCH /{id}/address` |
| `/api/v1/suppliers` | `GET`, `GET /{id}`, `POST`, `DELETE /{id}`, `PATCH /{id}/address` |
| `/api/v1/products` | `GET`, `GET /{id}`, `POST`, `DELETE /{id}`, `PATCH /{id}/stock` |
| `/api/v1/images` | `GET /product/{productId}`, `GET /{id}`, `POST`, `PUT /{id}`, `DELETE /{id}` |

Схемы запросов — в Swagger UI.

## Структура проекта

```
.
├── docker-compose.yml              # postgres, auth-service, store
├── .dockerignore
├── docker
│   └── postgres/init               # создание базы auth_db
└── src
    ├── store                       # REST API, веб-интерфейс, клиент auth-service
    │   ├── Dockerfile
    │   └── src/main/java/com/shastage/store
    │       ├── web                 # контроллеры, DTO, мапперы
    │       ├── domain              # модели, команды, сервисы, интерфейсы репозиториев
    │       ├── datasource          # JPA-сущности, адаптеры репозиториев
    │       ├── infrastructure      # клиент gRPC, конфигурация
    │       └── config              # OpenAPI, WebMVC
    ├── auth-service/service        # gRPC-сервис авторизации (подробности в readme.md сервиса)
    │   ├── Dockerfile
    │   └── src/main/java/app
    ├── contracts/proto             # общие Protobuf-контракты
    └── logs                        # логи
```

Поток данных: `браузер / клиент → store → PostgreSQL (store_db)`, а для авторизации `store → auth-service (gRPC) → PostgreSQL (auth_db)`.

## Особенности и ограничения

- Вход через Google требует реальных `CLIENT_ID` и `CLIENT_SECRET`; регистрация и вход по email работают с заглушками.
- Тестовые данные (клиенты, поставщики, товары) создаются миграцией `V2`.
- В `src/store/package.json` объявлен фронтенд-тест на Playwright, но файла `tests/frontend.test.mjs` в репозитории нет.
