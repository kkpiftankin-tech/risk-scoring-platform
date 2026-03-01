# C4 Level 3: Components — Interface Layer

## Область

Этот документ описывает компоненты интерфейсного слоя Risk Scoring Platform в терминах C4 Level 3 (Components) для контейнеров:

- `Decision API` (внешний API для фронт-офисных систем)
- `Integration Layer API` (внутренний API для интеграций и batch-процессов)

Цель — зафиксировать, какие компоненты отвечают за приём запросов, аутентификацию, валидацию, маршрутизацию и адаптацию запросов к внутренним сервисам. [web:595][web:612]

---

## Контейнер: Decision API

`Decision API` — фасад/входная точка для фронт-офисных систем, реализующий REST API для онлайн-скоринга и связанных операций. Он может быть реализован как API Gateway или BFF (Backend for Frontend) для фронт-офиса. [web:600][web:601][web:602][web:610]

### Компоненты

1. **HTTP API Controller**

   - Ответственность:
     - Обработка входящих HTTP-запросов от фронт-офисной системы.
     - Маршрутизация по endpoint’ам (`/score`, `/applications/{id}`, `/health`).
     - Формирование HTTP-ответов (коды, заголовки). [web:602][web:606]
   - Сотрудничество:
     - Вызывает `Authentication & Authorization`.
     - Передаёт валидированные данные в `Request Validation & Mapping` и далее в `Risk Request Orchestrator`.

2. **Authentication & Authorization**

   - Ответственность:
     - Проверка подлинности клиента (OAuth2/JWT/MTLS, в зависимости от выбранной схемы).
     - Проверка прав доступа к операциям скоринга. [web:602][web:603][web:607]
   - Сотрудничество:
     - Используется `HTTP API Controller` перед выполнением бизнес-операций.
     - Обращается к внешнему IdP/Keycloak/интеграции авторизации (при наличии).

3. **Request Validation & Mapping**

   - Ответственность:
     - Синтаксическая и семантическая валидация входных DTO (обязательные поля, диапазоны, форматы).
     - Маппинг внешних контрактов API в внутренние доменные объекты (`ScoreRequest`, `CustomerProfileRequest`). [web:609][web:611]
   - Сотрудничество:
     - Принимает данные от `HTTP API Controller`.
     - Передаёт доменные объекты в `Risk Request Orchestrator`.
     - При ошибках возвращает структурированные сообщения об ошибках в `HTTP API Controller`.

4. **Risk Request Orchestrator**

   - Ответственность:
     - Оркестрация процесса онлайн-скоринга:
       - вызов `Customer Data Adapter` для получения клиентских данных;
       - вызов `External Bureau Adapter` при необходимости;
       - вызов `Risk Engine Client` для расчёта скорингового балла;
       - агрегирование результатов в единый ответ. [web:600][web:610][web:605]
   - Сотрудничество:
     - Вызывается `Request Validation & Mapping`.
     - Использует компоненты интеграционного слоя/клиентские SDK для доступа к Core Banking и внешним бюро.
     - Возвращает агрегированный результат обратно в `HTTP API Controller`.

5. **Response Mapping & Error Handling**

   - Ответственность:
     - Преобразование доменного результата (`ScoreResult`, `DecisionResult`) в внешние DTO ответа API.
     - Единый формат ошибок (error codes, correlation id). [web:609][web:611]
   - Сотрудничество:
     - Получает данные от `Risk Request Orchestrator`.
     - Возвращает DTO в `HTTP API Controller` для отправки клиенту.

6. **Logging & Audit Adapter**

   - Ответственность:
     - Логирование ключевых событий: входящий запрос, результаты скоринга, ошибки.
     - Формирование аудиторского следа для последующего анализа и регуляторной отчётности. [web:600][web:607][web:610]
   - Сотрудничество:
     - Вызывается `HTTP API Controller`, `Risk Request Orchestrator` и `Authentication & Authorization`.
     - Пишет логи в централизованную систему (ELK, Loki, Splunk).

7. **Rate Limiting & Throttling (опционально)**

   - Ответственность:
     - Ограничение числа запросов от конкретного клиента/канала.
     - Защита внутреннего контура от перегрузки. [web:600][web:604][web:610]
   - Сотрудничество:
     - Встраивается в `HTTP API Controller` или перед ним (middleware/фильтр в API Gateway).

---

## Контейнер: Integration Layer API

`Integration Layer API` предоставляет интерфейсы для внутренних систем (batch-загрузки, пересчёты портфеля, ручная инициация процессов) и может отличаться контрактами и политиками от `Decision API`. [web:600][web:602][web:607]

### Компоненты

1. **Integration HTTP/API Controller**

   - Ответственность:
     - Обработка запросов от внутренних систем (ETL, batch-процессы, аналитические приложения).
     - Поддержка синхронных REST/HTTP и, при необходимости, gRPC/async-endpoints. [web:602]
   - Сотрудничество:
     - Использует `Integration Auth` (если отдельная схема auth для внутренних систем).
     - Вызывает `Batch Orchestrator` или `Portfolio Recalculation Orchestrator`.

2. **Integration Auth (Service-to-Service Authentication)**

   - Ответственность:
     - Аутентификация и авторизация внутренних сервисов (mTLS, service accounts, client credentials). [web:602][web:607]
   - Сотрудничество:
     - Вызывается `Integration HTTP/API Controller` перед выполнением операций.

3. **Batch Request Validation & Mapping**

   - Ответственность:
     - Валидация и маппинг batch-запросов (например, файлы/массивы заявок) в внутренние форматы.
   - Сотрудничество:
     - Получает данные от `Integration HTTP/API Controller`.
     - Передаёт данные в `Batch Orchestrator`.

4. **Batch Orchestrator**

   - Ответственность:
     - Управление batch-процессами:
       - массовый пересчёт скоринговых баллов;
       - обработка файловых загрузок (портфели заявок/клиентов). [web:605]
   - Сотрудничество:
     - Использует `Risk Engine Client` и адаптеры доступа к данным.
     - Публикует статусы выполнения в шину событий/мониторинг.

5. **Portfolio Recalculation Orchestrator (опционально)**

   - Ответственность:
     - Запуск сценариев пересчёта риска по портфелю (по расписанию или по событию).
   - Сотрудничество:
     - Использует адаптеры к хранилищам и `Risk Engine`.

6. **Monitoring & Metrics Adapter**

   - Ответственность:
     - Экспорт метрик интерфейсного и интеграционного слоя (RPS, latency, error rate) в Prometheus/другие системы мониторинга. [web:600][web:607]
   - Сотрудничество:
     - Интегрируется с контроллерами и оркестраторами.

---

## Связь с требованиями и другими C4-артефактами

- Требования:
  - `docs/requirements/system/srs-functional.md`
  - `docs/requirements/system/srs-non-functional.md`
- Архитектура:
  - `docs/architecture/containers.md`
  - `docs/architecture/components-risk-engine.md`
- Диаграммы:
  - `docs/architecture/diagrams/c4/containers.mmd`
  - `docs/architecture/diagrams/c4/components-interfaces.mmd` (будет добавлена)
- ADR:
  - `docs/adr/0001-use-c4-model.md`
  - ADR по выбору паттерна API Gateway/BFF и схем аутентификации (будут добавлены). [web:600][web:607][web:589][web:594]
# C4 Level 3: Components — Interface Layer

## Область

Этот документ описывает компоненты интерфейсного слоя Risk Scoring Platform в терминах C4 Level 3 (Components) для контейнеров:

- `Decision API` (внешний API для фронт-офисных систем)
- `Integration Layer API` (внутренний API для интеграций и batch-процессов)

Цель — зафиксировать, какие компоненты отвечают за приём запросов, аутентификацию, валидацию, маршрутизацию и адаптацию запросов к внутренним сервисам. [web:595][web:612]

---

## Диаграмма компонентов интерфейсного слоя (Mermaid)

```mermaid
C4Component
    title Risk Scoring Platform - Interface Layer Components

    Container_Boundary(c_decision_api, "Decision API") {
        Component(http_controller, "HTTP API Controller", "REST Controller", "Обрабатывает HTTP-запросы от фронт-офисной системы, маршрутизирует по endpoint'ам")
        Component(authz, "Authentication & Authorization", "Auth Component", "Проверяет подлинность клиентов и права доступа")
        Component(req_validation, "Request Validation & Mapping", "Validator/Mapper", "Валидирует внешние DTO и маппит в доменные объекты")
        Component(risk_orchestrator, "Risk Request Orchestrator", "Orchestrator", "Оркестрирует процесс онлайн-скоринга")
        Component(resp_mapping, "Response Mapping & Error Handling", "Mapper", "Формирует DTO ответа и единый формат ошибок")
        Component(logging, "Logging & Audit Adapter", "Adapter", "Логирует события и пишет аудиторский след")
        Component(rate_limit, "Rate Limiting & Throttling", "Middleware", "Ограничивает частоту запросов (RPS) для защиты системы")
    }

    Container_Boundary(c_integration_api, "Integration Layer API") {
        Component(int_http_controller, "Integration HTTP/API Controller", "REST/gRPC Controller", "Обрабатывает запросы от внутренних систем и batch-процессов")
        Component(int_auth, "Integration Auth", "Auth Component", "Аутентифицирует и авторизует внутренние сервисы")
        Component(batch_validation, "Batch Request Validation & Mapping", "Validator/Mapper", "Валидирует batch-запросы и маппит в внутренние форматы")
        Component(batch_orchestrator, "Batch Orchestrator", "Orchestrator", "Управляет batch-процессами и массовыми пересчётами")
        Component(portfolio_orchestrator, "Portfolio Recalculation Orchestrator", "Orchestrator", "Запускает сценарии пересчёта портфеля (опционально)")
        Component(metrics, "Monitoring & Metrics Adapter", "Adapter", "Публикует метрики интерфейсного слоя в систему мониторинга")
    }

    System_Boundary(c_internal, "Internal Systems") {
        System(risk_engine, "Risk Engine", "Service", "Выполняет расчёт скорингового балла")
        System(core_banking, "Core Banking System", "External System", "Хранит учетные данные клиентов и счетов")
        System(monitoring, "Monitoring System", "External System", "Собирает метрики и алерты (Prometheus/и т.п.)")
    }

    Rel(front_office_system, http_controller, "Отправляет запросы скоринга", "HTTPS/REST")
    Rel(http_controller, authz, "Проверка токена/прав доступа")
    Rel(http_controller, req_validation, "Передаёт данные для валидации и маппинга")
    Rel(req_validation, risk_orchestrator, "Передаёт доменные объекты ScoreRequest")
    Rel(risk_orchestrator, risk_engine, "Запрашивает расчёт скоринга", "Sync/Async API")
    Rel(risk_orchestrator, logging, "Логирует результат скоринга")
    Rel(risk_orchestrator, resp_mapping, "Передаёт доменный результат")
    Rel(resp_mapping, http_controller, "Возвращает DTO ответа клиенту")
    Rel(rate_limit, http_controller, "Ограничивает частоту вызовов", "Middleware/Filter")

    Rel(int_http_controller, int_auth, "Проверка service-to-service auth")
    Rel(int_http_controller, batch_validation, "Передаёт batch-запросы для валидации")
    Rel(batch_validation, batch_orchestrator, "Передаёт нормализованные batch-задачи")
    Rel(batch_orchestrator, risk_engine, "Массово вызывает Risk Engine")
    Rel(portfolio_orchestrator, risk_engine, "Инициирует пересчёт портфеля")
    Rel(metrics, http_controller, "Собирает метрики запросов")
    Rel(metrics, int_http_controller, "Собирает метрики интеграционного API")
    Rel(metrics, monitoring, "Публикует метрики", "Prometheus/PushGateway")
```

