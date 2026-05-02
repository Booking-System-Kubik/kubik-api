# API Booking System

Официальное описание REST API сервиса бронирования рабочих мест и переговорных для интеграции сторонних клиентов.

| Параметр | Значение |
|----------|----------|
| Версия приложения | `0.0.1-SNAPSHOT` (Maven artifact `com.t1:officebooking`) |
| Формат обмена | JSON (`application/json`), кроме отчётов CSV |
| Кодировка | UTF-8 |
| Базовый URL | `{HOST}` — хост развёрнутого сервиса (локально по умолчанию порт **8080**, см. `server.port` в конфигурации) |
| Префикс API бизнес-логики | `/api/...` |

Все пути ниже указаны относительно `{HOST}` (например: `https://api.example.com/api/auth/login`).

---

## 1. Общие правила

### 1.1. Заголовки

| Заголовок | Обязательность | Описание |
|-----------|------------------|----------|
| `Content-Type` | Для тел запросов с JSON | `application/json` |
| `Authorization` | Для защищённых методов | `Bearer <access_token>` |

Аутентификация выполняется по **JWT access token** в заголовке `Authorization` (схема Bearer). Токен выдаётся при входе и обновляется через refresh token.

### 1.2. Форматы даты и времени

- В телах запросов и ответах используются типы в стиле ISO-8601:
  - `LocalDateTime` — как строка `YYYY-MM-DDTHH:mm:ss` (часто без смещения зоны; сервер интерпретирует моменты бронирования в связке с часовым поясом локации).
  - `LocalDate` — `YYYY-MM-DD`.
  - `LocalTime` — `HH:mm:ss`.
- Поля `expiresIn` и `refreshExpiresIn` в ответе логина — **секунды** до истечения.

### 1.3. JWT и время жизни токенов

Параметры задаются конфигурацией приложения (`jwt.access-expiration-min`, `jwt.refresh-expiration-days`, `jwt.secret`). Конкретные числа зависят от окружения.

В полезной нагрузке access token содержатся утверждения (claims), в том числе `userId` и `roles`.

### 1.4. Роли пользователей

В системе используются значения перечисления `UserRole`:

| Значение | Описание |
|----------|----------|
| `ROLE_USER` | Обычный пользователь |
| `ROLE_ADMIN_PROJECT` | Администратор проекта / локации (ограничение действий своей площадкой в бизнес-логике) |
| `ROLE_ADMIN_WORKSPACE` | Администратор workspace (организация) |

Маршруты Spring Security:

- **`/api/admin/work-space/**`** — только роль **ADMIN_WORKSPACE** (в конфигурации `hasRole("ADMIN_WORKSPACE")`, что соответствует authority `ROLE_ADMIN_WORKSPACE`).
- **`/api/admin/**`** (остальное, включая отчёты) — **ADMIN_PROJECT** или **ADMIN_WORKSPACE**.
- Остальные запросы под префиксом `/api`, не перечисленные как публичные, требуют аутентификации.

### 1.5. Публичные эндпоинты (без JWT)

Без заголовка `Authorization` доступны:

- все методы **`/api/auth/**`**;
- пути, совпадающие с **`/api/organizations/**`** (список организаций и локаций по организации).

**Важно:** маршруты вида `/api/locations/...` **не** входят в исключение и требуют JWT.

### 1.6. Мониторинг (Spring Boot Actuator)

Подключены эндпоинты (см. `management.endpoints.web.exposure.include`):

| Метод | Путь | Назначение |
|-------|------|------------|
| GET | `/actuator/health` | Проверка работоспособности |
| GET | `/actuator/info` | Информация о приложении |
| GET | `/actuator/prometheus` | Метрики Prometheus |

Доступ к actuator в типичной конфигурации Spring Security может требовать аутентификации или возвращать `403` — зависит от развёртывания. Уточняйте у администратора окружения.

---

## 2. Ошибки и коды ответов

### 2.1. Ошибки валидации тела запроса

**HTTP 400 Bad Request**, тело:

```json
{
  "message": "Validation failed",
  "errors": {
    "email": "Email should be valid",
    "password": "Password must be between 6 and 100 characters"
  }
}
```

Ключи в `errors` — имена полей или объекты валидации.

### 2.2. Прочие типичные ответы

| Код | Когда |
|-----|--------|
| 200 | Успешное выполнение |
| 201 | Ресурс создан |
| 202 | Запрос принят, обработка асинхронная / ожидание (регистрация в режиме заявки) |
| 400 | Некорректное бронирование и др. (`IncorrectBookingException`) — текст сообщения в теле |
| 401 | Невалидный или истёкший JWT, неверный пароль при логине и т.п. |
| 403 | Недостаточно прав (`AdminAuthorityAbusingException`, `RefactoringForeignBookingsException`, `RoleAssignmentViolationException`) |
| 404 | Сущность не найдена (`EntityNotFoundException`) |
| 409 | Конфликт: слот занят, дубликат и т.д. (`SlotAlreadyBookedException`, `EntityExistsException`, часть `IllegalStateException`) |
| 503 | Проблемы доступа к данным (`DataAccessResourceFailureException`) |

Часто тело ошибки — **простая строка** (текст сообщения), не JSON-объект.

---

## 3. Модели данных (справочник)

### 3.1. `Organization`

| Поле | Тип | Описание |
|------|-----|----------|
| id | long | Идентификатор |
| name | string | Уникальное имя (до 150 символов) |
| isActive | boolean | Активность |

### 3.2. `Location`

| Поле | Тип | Описание |
|------|-----|----------|
| id | long | Идентификатор |
| name | string | Название |
| city | string | Город |
| address | string | Адрес |
| isActive | boolean | Активность |
| workDayStart | LocalTime | Начало рабочего дня |
| workDayEnd | LocalTime | Конец рабочего дня |
| timeZone | string | Идентификатор часового пояса (например `Europe/Moscow`) |
| organization | object | Связь с организацией (может присутствовать в JSON при сериализации) |

### 3.3. `Space` (сущность ответа при создании помещения)

В ответах создания возвращается JPA-сущность; типичные поля:

| Поле | Тип | Описание |
|------|-----|----------|
| id | long | Идентификатор |
| capacity | int | Вместимость |
| bookable | boolean | Доступно для бронирования |
| bounds | object | Координаты на плане (`x`, `y`, `width`, `height`) при наличии |
| location | object | Локация |
| spaceType | object | Тип помещения |
| floor | object | Этаж |

Точный состав вложенных объектов зависит от загрузки связей Hibernate и настроек сериализации.

### 3.4. `SpaceResponse` (упрощённое представление помещения)

```json
{
  "id": 1,
  "locationId": 2,
  "spaceTypeId": 3,
  "spaceType": "MEETING_ROOM",
  "capacity": 8,
  "floor": {
    "id": 10,
    "floorNumber": 3,
    "polygon": [{ "x": 0, "y": 0 }]
  },
  "bookable": true,
  "bounds": { "x": 10, "y": 20, "width": 100, "height": 80 }
}
```

### 3.5. `BookingResponse`

| Поле | Тип | Описание |
|------|-----|----------|
| id | long | Идентификатор бронирования |
| userEmail | string | Email пользователя |
| locationName | string | Название локации |
| locationId | long | Идентификатор локации |
| spaceName | string | Имя/метка помещения |
| spaceId | long | Идентификатор помещения |
| start | LocalDateTime | Начало |
| end | LocalDateTime | Окончание |
| bookingType | string | Тип бронирования |
| status | string | Статус (например `CONFIRMED`, `CANCELLED`) |

### 3.6. `UserDataResponse`

| Поле | Тип |
|------|-----|
| email | string |
| fullName | string |
| locationId | long \| null |
| locationName | string \| null |
| organizationId | long \| null |
| organizationName | string \| null |
| roles | массив `UserRole` |

### 3.7. `LoginResponse`

```json
{
  "jwtResponse": {
    "accessToken": "...",
    "refreshToken": "...",
    "expiresIn": 900,
    "refreshExpiresIn": 604800
  },
  "role": ["ROLE_USER"]
}
```

Поле `role` в JSON сериализуется как `role` (имя геттера record-компонента в Java).

### 3.8. `PendingRegistrationResponse`

| Поле | Тип |
|------|-----|
| id | long |
| email | string |
| fullName | string |
| position | string |
| organizationId | long \| null |
| locationId | long \| null |
| status | string |
| createdAt | LocalDateTime |

### 3.9. `TimeSlotResponse`

| Поле | Тип | Описание |
|------|-----|----------|
| offset | string | Смещение часового пояса для слота |
| start | LocalDateTime | Начало интервала |
| end | LocalDateTime | Конец интервала |
| status | string | `available` или `booked` |
| availableDurations | массив string \| null | Допустимые длительности в формате `Duration` ISO (например `PT1H`) для свободных слотов |

### 3.10. Тип бронирования (`type` в запросах брони)

Поле передаётся строкой (`bookingType` в БД до 20 символов). Должно быть согласовано с типом помещения и правилами сервера (в коде встречаются типы помещений вроде `MEETING_ROOM`, `WORKSPACE`). Уточняйте допустимые значения для вашей инсталляции у администратора данных.

---

## 4. Эндпоинты: аутентификация

Базовый путь: **`/api/auth`**

### 4.1. Регистрация

**`POST /api/auth/register`**

Тело: `RegistrationRequest`

| Поле | Обязательность | Описание |
|------|----------------|----------|
| email | да | Email |
| password | да | 6–100 символов |
| fullName | да | 5–100 символов |
| position | да | Должность, до 100 символов |
| location | нет | Идентификатор локации (`long`) |
| organizationId | нет | Существующая организация |
| organizationName | нет | Имя новой организации (создание организации при регистрации) |

**Логика ответа:**

- Если пользователь регистрируется в **уже существующей** организации (без создания новой организации в этой заявке) — создаётся **заявка** на регистрацию, пользователь в БД не создаётся сразу.
  - **HTTP 202 Accepted**, тело пустое.
- Если создаётся **новая** организация (через `organizationName`) — пользователь создаётся сразу (в т.ч. с ролью администратора workspace).
  - **HTTP 200 OK**, тело пустое.

Конфликт: email уже занят — **409** (`IllegalStateException`: сообщение о существующем email).

---

### 4.2. Вход

**`POST /api/auth/login`**

Тело: `LoginRequest` — `email`, `password`.

**Ответ 200 OK:** `LoginResponse` (токены и набор ролей).

Неверный пароль — **401** (текст сообщения).

---

### 4.3. Обновление токена

**`POST /api/auth/refresh`**

Тело: `RefreshTokenRequest`

```json
{ "refreshToken": "<refresh_token>" }
```

**Ответ 200 OK:** `JwtResponse` (новая пара access/refresh и сроки жизни).

---

### 4.4. Выход (отзыв refresh token)

**`POST /api/auth/logout`**

Тело: `RefreshTokenRequest` с полем `refreshToken`.

**Ответ:** без тела (успешное завершение — обычно **200**).

---

### 4.5. Проверка текущей сессии

**`GET /api/auth/check-auth`**

**Требуется:** Bearer access token.

**Ответ 200 OK:** `UserDataResponse` для текущего пользователя.

---

## 5. Эндпоинты: справочники (организации и локации)

Базовый путь: **`/api`**

### 5.1. Список организаций

**`GET /api/organizations`**

**Доступ:** публичный.

**Ответ 200 OK:** массив `Organization`.

---

### 5.2. Локации организации

**`GET /api/organizations/{organizationId}/locations`**

**Доступ:** публичный.

**Ответ 200 OK:** массив `Location`.

---

### 5.3. Помещения этажа локации

**`GET /api/locations/{locationId}/spaces?floorNumber={n}`**

| Параметр | Тип | Обязательность |
|----------|-----|----------------|
| floorNumber | integer | да (query) |

**Доступ:** JWT.

**Ответ 200 OK:** `FloorSpacesResponse`

```json
{
  "floor": {
    "id": 1,
    "floorNumber": 2,
    "polygon": [{ "x": 0, "y": 0 }]
  },
  "spaces": [ { ...SpaceResponse } ]
}
```

Поле `floor` может быть `null`, если список помещений пуст.

---

### 5.4. Типы помещений локации

**`GET /api/locations/{locationId}/spacetypes`**

**Доступ:** JWT.

**Ответ 200 OK:** список типов помещений (`SpaceType` и т.п. — фактическая структура соответствует сериализации сущности).

---

## 6. Эндпоинты: бронирование (пользователь)

Базовый путь: **`/api/booking`**

Все методы ниже требуют **JWT** (роль пользователя или администратора — см. бизнес-правила).

### 6.1. Фильтрация помещений

**`POST /api/booking/space-filter`**

Тело: `FilteringSpacesRequest`

| Поле | Обязательность | Описание |
|------|----------------|----------|
| locationId | да | Локация |
| spaceTypeId | да | Тип помещения |
| floorNumber | нет | Номер этажа |

**Ответ 200 OK:** массив `SpaceResponse`.

---

### 6.2. Доступные интервалы на день

**`POST /api/booking/time-intervals`**

Тело: `TimeIntervalsRequest`

| Поле | Обязательность |
|------|----------------|
| date | да (`LocalDate`) |
| spaceId | да |

**Ответ 200 OK:** массив `TimeSlotResponse`.

---

### 6.3. Создать бронирование

**`POST /api/booking/book`**

Тело: `BookingRequest`

| Поле | Обязательность |
|------|----------------|
| spaceId | да |
| type | да (строка типа брони) |
| start | да (`LocalDateTime`) |
| end | да (`LocalDateTime`) |

**Ответ 201 Created:** `BookingResponse`.

Ошибки: занят слот — **409**; нарушение правил времени/офиса — **400**; и др.

---

### 6.4. Отменить своё бронирование

**`POST /api/booking/cancel/{id}`**

| Параметр | Описание |
|----------|----------|
| id | Идентификатор бронирования (path) |

**Требуется:** JWT пользователя-владельца брони.

**Ответ:** без тела (**200**). Чужая бронь — **403**.

---

### 6.5. Активные бронирования текущего пользователя

**`GET /api/booking/active-bookings`**

**Ответ 200 OK:** массив `BookingResponse`.

---

### 6.6. Все бронирования текущего пользователя

**`GET /api/booking/all-bookings`**

**Ответ 200 OK:** массив `BookingResponse` (включая завершённые/отменённые — по логике репозитория).

---

## 7. Эндпоинты: администрирование (общее)

Базовый путь: **`/api/admin`**

**Доступ:** роли **ADMIN_PROJECT** или **ADMIN_WORKSPACE** (кроме раздела work-space, см. ниже).

Ограничения по организации/локации для «администратора проекта» vs «администратора workspace» реализованы в сервисах: администратор проекта привязан к своей локации, администратор workspace — к организации.

### 7.1. Отмена бронирования администратором

**`POST /api/admin/cancel/{id}`**

**Ответ:** без тела. Недостаточно полномочий — **403**.

---

### 7.2. Активные бронирования пользователя по email

**`GET /api/admin/users/active-bookings?email={email}`**

**Параметр query:** `email` — обязателен, формат email.

**Ответ 200 OK:** массив `BookingResponse`.

---

### 7.3. Все бронирования пользователя по email

**`GET /api/admin/users/bookings?email={email}`**

**Ответ 200 OK:** массив `BookingResponse`.

---

### 7.4. Активные бронирования в зоне ответственности администратора

**`GET /api/admin/active-bookings`**

Для администратора проекта — по своей локации; для администратора workspace — по организации.

**Ответ 200 OK:** массив `BookingResponse`.

---

### 7.5. Список пользователей

**`GET /api/admin/users`**

**Ответ 200 OK:** массив `UserDataResponse`.

---

### 7.6. Назначить роль

**`POST /api/admin/assign-role`**

Тело: `ChangingRoleRequest`

| Поле | Описание |
|------|----------|
| email | Пользователь |
| role | Одно из `UserRole` |

**Ответ:** без тела. Нарушение правил назначения — **403**.

---

### 7.7. Отозвать роль

**`POST /api/admin/revoke-role`**

Тело: `ChangingRoleRequest` (те же поля).

---

### 7.8. Бронирование от имени пользователя

**`POST /api/admin/book`**

Тело: `BookingByAdminRequest`

| Поле | Описание |
|------|----------|
| userEmail | Email пользователя, для которого создаётся бронь |
| spaceId | Помещение |
| type | Тип бронирования |
| start | Начало |
| end | Конец |

**Ответ 201 Created:** `BookingResponse`.

---

### 7.9. Удалить пользователя из организации

**`DELETE /api/admin/users?email={email}`**

**Ответ:** без тела.

---

## 8. Эндпоинты: администрирование workspace

Базовый путь: **`/api/admin/work-space`**

**Доступ:** только **ADMIN_WORKSPACE** (`ROLE_ADMIN_WORKSPACE`).

### 8.1. Создать локацию

**`POST /api/admin/work-space/create-location`**

Тело: `CreatingLocationRequest`

| Поле | Описание |
|------|----------|
| name | Название |
| city | Город |
| address | Адрес |
| isActive | Активность |
| workDayStart | Начало рабочего дня |
| workDayEnd | Конец рабочего дня |
| timeZone | Часовой пояс |
| organizationId | Организация (должна совпадать с организацией администратора) |

**Ответ 201 Created:** тело — `long`, идентификатор созданной локации.

Попытка указать чужую организацию — **403**.

---

### 8.2. Создать помещение

**`POST /api/admin/work-space/create-space`**

Тело: `CreatingSpaceRequest`

| Поле | Описание |
|------|----------|
| locationId | Локация (принадлежит организации администратора) |
| spaceTypeId | Тип помещения |
| capacity | ≥ 1 |
| floorNumber | Номер этажа |
| x, y, width, height | Опционально — позиция на плане |

**Ответ 201 Created:** сущность `Space`.

---

### 8.3. Создать тип помещения

**`POST /api/admin/work-space/create-spacetype`**

Тело: `CreatingSpaceTypeRequest`

| Поле | Описание |
|------|----------|
| type | Строковый тип (имя) |
| locationId | Локация |
| allowedDurations | Список строк длительностей (формат `Duration`, например `PT30M`, `PT1H`) |

**Ответ 201 Created:** без тела.

---

### 8.4. Массовое создание помещений этажа

**`POST /api/admin/work-space/create-floor-spaces`**

Тело: `CreatingFloorSpacesRequest`

| Поле | Описание |
|------|----------|
| locationId | Локация |
| floorNumber | Этаж |
| polygon | Массив `PointRequest` (`x`, `y`) — контур этажа |
| spaces | Массив вложенных `CreatingSpaceRequest` |

**Ответ 201 Created:** массив `Space`.

---

### 8.5. Активные бронирования по локации

**`GET /api/admin/work-space/location/{locationId}/bookings`**

**Ответ 200 OK:** массив `BookingResponse`.

---

### 8.6. Пользователи локации

**`GET /api/admin/work-space/location/{locationId}/users`**

**Ответ 200 OK:** массив `UserDataResponse`.

---

### 8.7. Заявки на регистрацию

**`GET /api/admin/work-space/registration-requests`**

Список заявок по организации текущего администратора.

**Ответ 200 OK:** массив `PendingRegistrationResponse`.

---

### 8.8. Одобрить заявку

**`POST /api/admin/work-space/registration-requests/{id}/approve`**

**Ответ 200 OK:** без тела.

---

### 8.9. Отклонить заявку

**`POST /api/admin/work-space/registration-requests/{id}/reject`**

**Ответ 200 OK:** без тела.

---

## 9. Эндпоинты: отчёты (CSV)

Базовый путь: **`/api/admin/reports`**

**Доступ:** **ADMIN_PROJECT** или **ADMIN_WORKSPACE**.

Формат ответа у всех методов ниже:

- **Content-Type:** `text/csv`
- **Content-Disposition:** `attachment; filename=analytics_report.csv`
- Тело: бинарное содержимое файла CSV (`byte[]`).

Тело запроса для POST — **`ReportRequest`**:

| Поле | Описание |
|------|----------|
| start | Начало периода (`LocalDate`) |
| end | Конец периода (`LocalDate`) |
| timeZone | Идентификатор часового пояса для отчёта |

Для маршрутов с `{locationId}` в пути при генерации отчёта используется указанная локация; для администратора workspace без привязки к локации в некоторых методах параметры масштаба задаются иначе (см. реализацию: «project admin» vs «workspace admin» определяется наличием роли `ROLE_ADMIN_WORKSPACE`).

### 9.1. Базовый аналитический отчёт (вся доступная область)

**`POST /api/admin/reports/analytics/csv`**

---

### 9.2. Базовый отчёт по локации

**`POST /api/admin/reports/analytics/location/{locationId}/csv`**

---

### 9.3. Отчёт загрузки помещений

**`POST /api/admin/reports/analytics/space-load/csv`**

---

### 9.4. Загрузка помещений по локации

**`POST /api/admin/reports/analytics/location/{locationId}/space-load/csv`**

---

### 9.5. Отчёт загрузки локаций

**`POST /api/admin/reports/analytics/location-load/csv`**

---

### 9.6. Загрузка локаций по одной локации

**`POST /api/admin/reports/analytics/location/{locationId}/location-load/csv`**

---

## 10. Рекомендации для интеграции

1. Сохраняйте `refreshToken` в защищённом хранилище; перед истечением access token вызывайте **`POST /api/auth/refresh`**.
2. Для выхода вызывайте **`POST /api/auth/logout`**, передавая refresh token — это инвалидирует сессию обновления.
3. Учитывайте **202 Accepted** при регистрации в существующей организации — вход возможен только после одобрения заявки администратором workspace.
4. Планируйте повтор запросов при **409** при бронировании (гонка за слот).
5. Используйте **IAM-часовой пояс** локации при отображении слотов пользователю; сервер возвращает слоты с учётом рабочего дня и занятости.

---

*Документ сформирован по исходному коду Spring Boot-приложения в репозитории. При расхождении поведения с данным текстом приоритет имеет фактическая реализация и конфигурация развёрнутого окружения.*
