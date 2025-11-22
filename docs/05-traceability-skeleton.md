### Test Scenarios (наступний крок)

На основі ERD invariants та DFD error paths:

- **TS-001:** Happy path створення бронювання (capacity 0→1)
  - INSERT INTO bookings, перевірка COUNT < 3
  - Генерація confirmation_code UNIQUE
  - booking_status = 'requested'
- **TS-002:** Створення при 2/3 capacity (2→3)
  - COUNT = 2, INSERT дозволено
  - Перевірка INDEX (timeslot_id, booking_status)
- **TS-003:** Відмова при 3/3 capacity (slot_unavailable)
  - COUNT = 3, INSERT заблоковано
  - Error code: slot_unavailable, HTTP 409
- **TS-004:** Concurrent booking requests (race condition)
  - Два одночасних INSERT для останнього місця
  - FOR UPDATE захист, один SUCCESS один 409
- **TS-005:** Валідація invalid_email
  - Перевірка customer_email RFC 5322
  - Error code: invalid_email, HTTP 400
- **TS-006:** Скасування та звільнення capacity (3→2)
  - UPDATE booking_status = 'cancelled'
  - cancelled_at = NOW()
  - COUNT зменшується, слот знову доступний
- **TS-007:** Cache hit performance (≤200ms)
  - D4 cache_or_view перевірка
  - INDEX scan на timeslots + bookings
- **TS-008:** Privacy check (masked email in notifications)

  - customer_email → masked_email в notification_events
  - u**_@e_**.com формат

- **TS-009:** FK integrity check (timeslot_not_found)

  - INSERT booking з неіснуючим timeslot_id
  - Error code: reference_not_found, HTTP 404

- **TS-010:** Status transition validation

  - cancelled → confirmed заборонено
  - Error code: invalid_status_transition, HTTP 409

- **TS-011:** Unique confirmation_code

  - Collision handling (малоймовірно)
  - Retry generation при duplicate

- **TS-012:** Time range constraint
  - start_at >= end_at заборонено
  - DB CHECK constraint, error: invalid_time_range

---

**Кінець оновленого Traceability Matrix v2.0**

---

## ERD → DFD Mapping

| ERD Entity          | DFD Data Store         | Processes (Read)                  | Processes (Write)           |
| ------------------- | ---------------------- | --------------------------------- | --------------------------- |
| meeting_rooms       | D1 meeting_rooms       | P1 (read_rooms)                   | P2 (write_rooms)            |
| timeslots           | D2 timeslots           | P1 (read_timeslots), P3.2 (check) | P2 (write_timeslots)        |
| bookings            | D3 bookings            | P3.3 (COUNT), P4 (read)           | P3.4 (persist), P4 (update) |
| notification_events | D5 notification_outbox | P5 (read for processing)          | P3.6, P4 (emit events)      |
| users               | -                      | P2 (authentication context)       | -                           |

**Cache layer:**

- D4 cache_or_view - не є окремою таблицею ERD
- Проекція даних з timeslots + bookings COUNT
- Інвалідується P2, P3, P4

---

## ERD Indexes → DFD Query Patterns

| Index                                          | Query Pattern                                          | DFD Process           | Performance Target       |
| ---------------------------------------------- | ------------------------------------------------------ | --------------------- | ------------------------ |
| (meeting_room_id, start_at) on timeslots       | WHERE meeting_room_id = ? AND start_at BETWEEN ? AND ? | P1 discover           | ≤ 200ms (with cache)     |
| (timeslot_id, booking_status) on bookings      | WHERE timeslot_id = ? AND booking_status IN (?)        | P3.3 capacity check   | ≤ 50ms (index-only scan) |
| (customer_email, created_at DESC) on bookings  | WHERE customer_email = ? ORDER BY created_at DESC      | P4 list user bookings | ≤ 100ms                  |
| (processed, created_at) on notification_events | WHERE processed = false ORDER BY created_at            | P5 dequeue events     | ≤ 50ms                   |
| UNIQUE confirmation_code on bookings           | WHERE confirmation_code = ?                            | P4 lookup by code     | ≤ 10ms (unique index)    |

---

## ERD Invariants → DFD Validation Points

| Invariant                | ERD Level             | DFD Process                      | Error Code                  |
| ------------------------ | --------------------- | -------------------------------- | --------------------------- |
| start_at < end_at        | DB CHECK (timeslots)  | P2 validate_timeslot             | invalid_time_range          |
| booking_status enum      | DB CHECK (bookings)   | P3.1 validate_request            | invalid_enum_value          |
| cancelled_at logic       | DB CHECK (bookings)   | P4 cancel_booking                | invariant_violation         |
| email format             | APP validation        | P3.1 validate_request            | invalid_email               |
| **Capacity ≤ 3**         | **APP-level**         | **P3.3.1 validate_booking_data** | **slot_unavailable**        |
| Status transitions       | APP state machine     | P4 update_status                 | invalid_status_transition   |
| confirmation_code unique | DB UNIQUE (bookings)  | P3.4 persist (retry on conflict) | duplicate_confirmation_code |
| timeslot uniqueness      | DB UNIQUE (composite) | P2 create_timeslot               | duplicate_timeslot          |

---

## ERD → Code Sets Mapping

| ERD Column                     | Code Set                                                 | DFD Usage                                 |
| ------------------------------ | -------------------------------------------------------- | ----------------------------------------- |
| bookings.booking_status        | booking_status {requested, confirmed, cancelled}         | P3 (create='requested'), P4 (transitions) |
| bookings.cancel_reason_code    | cancel_reason_code {user_request, room_unavailable, ...} | P4 (set on cancellation)                  |
| notification_events.event_type | event_type {booking_created, booking_cancelled}          | P3.6, P4 (emit events)                    |
| users.role                     | role {space_manager, admin}                              | P2 (authorization check)                  |

---

## Примітки для майбутнього розвитку

### API Endpoints (наступний крок)

На основі ERD entities та DFD processes:

| Endpoint                         | HTTP Method | ERD Entities                       | DFD Process |
| -------------------------------- | ----------- | ---------------------------------- | ----------- |
| `/api/v1/timeslots`              | GET         | meeting_rooms, timeslots, bookings | P1          |
| `/api/v1/bookings`               | POST        | bookings, timeslots                | P3          |
| `/api/v1/bookings/{id}`          | GET         | bookings                           | P4          |
| `/api/v1/bookings/{id}`          | DELETE      | bookings                           | P4          |
| `/api/v1/bookings?email={email}` | GET         | bookings                           | P4          |
| `/api/v1/rooms`                  | POST        | meeting_rooms                      | P2          |
| `/api/v1/rooms/{id}/timeslots`   | POST        | timeslots                          | P2          |

### Test Scenarios (наступний крок)

На основі ERD invariants та DFD error paths:

- **TS-001:** Happy path створення бронювання (capacity 0→1)
  - INSERT INTO bookings, перевірка COUNT# Traceability Matrix - CoworkSpace Booking System

**Дата оновлення:** 2025-01-15  
**Версія:** 2.0 (додано DFD процеси)

---

## User Stories → Architecture Components (ПОВНА ВЕРСІЯ)

| User Story                                    | DFD Process                                                             | ERD Entities                                                           | **API Endpoints**                                                                                                                                                                             | **Error Codes**                                                                                                                                                                                 | Test Scenarios         |
| --------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| **US-001: Переглядати доступні часові слоти** | **P1** discover_services_and_timeslots                                  | • meeting_rooms (read)<br>• timeslots (read)<br>• bookings (COUNT)     | **GET /v1/meeting_rooms/{id}/availability**<br>Query: from, to<br>Response: 200 OK<br><br>**GET /v1/timeslots**<br>Query: filter[meeting_room_id], filter[only_available]<br>Response: 200 OK | • `200` - Success<br>• `400` - invalid_format (date)<br>• `401` - authentication_required<br>• `404` - resource_not_found                                                                       | TS-001                 |
| **US-002: Створювати нове бронювання**        | **P3** create_booking<br>P3.1-P3.8 (Level 1)<br>P3.3.1-P3.3.5 (Level 2) | • bookings (INSERT)<br>• timeslots (FK check)<br>• notification_events | **POST /v1/bookings**<br>Headers: Idempotency-Key<br>Body: timeslot_id, customer_name, customer_email<br>Response: 201 Created + Location                                                     | • `201` - Created<br>• `409` - **slot_unavailable** (capacity 3/3)<br>• `422` - validation_error (email, name)<br>• `422` - reference_not_found (timeslot)<br>• `401` - authentication_required | TS-002, TS-003, TS-004 |
| **US-003: Переглядати мої бронювання**        | **P4** modify_or_cancel (частково)                                      | • bookings (SELECT)<br>WHERE customer_email = ?                        | **GET /v1/bookings**<br>Query: filter[customer_email]<br>Response: 200 OK + pagination                                                                                                        | • `200` - Success<br>• `401` - authentication_required                                                                                                                                          | TS-005                 |
| **US-006: Підтверджувати бронювання**         | **P4** modify_or_cancel                                                 | • bookings (UPDATE)<br>SET status='confirmed'                          | **PATCH /v1/bookings/{id}**<br>Body: {"booking_status": "confirmed"}<br>Response: 200 OK                                                                                                      | • `200` - Success<br>• `404` - resource_not_found<br>• `409` - invalid_status_transition<br>• `401`, `403`                                                                                      | TS-006                 |
| **US-007: Скасовувати бронювання**            | **P4** modify_or_cancel                                                 | • bookings (UPDATE)<br>SET status='cancelled',<br>cancelled_at=NOW()   | **PATCH /v1/bookings/{id}**<br>Body: {"booking_status": "cancelled", "cancel_reason_code": "user_request"}<br>Response: 200 OK                                                                | • `200` - Success<br>• `404` - resource_not_found<br>• `409` - invalid_status_transition (cancelled→confirmed)<br>• `422` - invariant_violation                                                 | TS-007                 |

---

## API Endpoints Summary

| Endpoint                           | Method | DFD Process | Success | Errors                         |
| ---------------------------------- | ------ | ----------- | ------- | ------------------------------ |
| `/meeting_rooms`                   | GET    | P1          | 200     | 401                            |
| `/meeting_rooms/{id}/availability` | GET    | P1          | 200     | 400, 401, 404                  |
| `/timeslots`                       | GET    | P1          | 200     | 400, 401                       |
| `/bookings`                        | GET    | P4          | 200     | 401                            |
| `/bookings`                        | POST   | P3          | 201     | 409 slot_unavailable, 422, 401 |
| `/bookings/{id}`                   | GET    | P4          | 200     | 404, 401                       |
| `/bookings/{id}`                   | PATCH  | P4          | 200     | 409, 422, 404, 401             |

---

## Error Codes → HTTP Status Mapping

| Error Code                  | HTTP Status | DFD Origin        | API Usage                                 |
| --------------------------- | ----------- | ----------------- | ----------------------------------------- |
| `authentication_required`   | 401         | -                 | Всі endpoints без токену                  |
| `permission_denied`         | 403         | -                 | Недостатньо scopes                        |
| `resource_not_found`        | 404         | P4 (read)         | GET/PATCH /bookings/{id}                  |
| `reference_not_found`       | 422         | P3.2 check        | POST /bookings (невалідний timeslot_id)   |
| `validation_error`          | 422         | P3.1 validate     | POST /bookings (invalid email/name)       |
| **`slot_unavailable`**      | **409**     | **P3.3 capacity** | **POST /bookings (3/3 capacity)**         |
| `invalid_status_transition` | 409         | P4 update         | PATCH /bookings (cancelled→confirmed)     |
| `invariant_violation`       | 422         | DB CHECK          | PATCH /bookings (cancelled_at без status) |
| `invalid_format`            | 400         | Validation        | Query params (date format)                |
| `rate_limit_exceeded`       | 429         | Rate limiter      | Всі endpoints (100/hour)                  |

---

## Idempotency Mapping

| Endpoint             | Idempotency-Key        | Behavior                                                  |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| POST /bookings       | Required (recommended) | Повторний запит з тим самим ключем → той самий booking_id |
| PATCH /bookings/{id} | Optional               | Ідемпотентно за замовчуванням (PUT-like)                  |
| GET endpoints        | N/A                    | Природно ідемпотентні                                     |

---

**Кінець оновленого Traceability Matrix v3.0 (з API)**
|------------|-------------|--------------|---------------|----------------|
| **US-001: Переглядати доступні часові слоти** | **P1** discover_services_and_timeslots<br>**Потоки:**<br>• availability_query (вхід)<br>• available_timeslots (вихід)<br>• read_rooms (D1)<br>• read_timeslots (D2)<br>• query_cache / cached_data (D4) | **Entities:**<br>• meeting_rooms (read)<br>• timeslots (read)<br>• bookings (COUNT aggregate)<br>**Indexes:**<br>• (meeting_room_id, start_at)<br>• (timeslot_id, booking_status)<br>**Computed:** available_spots = 3 - COUNT(bookings) | → (TBD) | → (TBD) |
| **US-002: Створювати нове бронювання** | **P3** create_booking<br>**Level 1:**<br>• P3.1 validate_request<br>• P3.2 check_timeslot_existence<br>• P3.3 check_capacity<br>• P3.4 persist_booking<br>• P3.5 generate_confirmation<br>• P3.6 emit_domain_event<br>• P3.7 mask_personal_data<br>• P3.8 reject_with_errors<br>**Level 2 (P3.3):**<br>• P3.3.1 validate_booking_data<br>• P3.3.2 write_to_bookings<br>• P3.3.3 update_cache<br>• P3.3.4 prepare_confirmation<br>• P3.3.5 forward_to_logging<br>**Потоки:**<br>• booking_request (вхід)<br>• booking_confirmation (вихід)<br>• slot_unavailable (помилка)<br>• timeslot_reference (D2)<br>• booking_created (D3)<br>• invalidate_cache (D4) | **Entities:**<br>• bookings (INSERT)<br> - booking_id (PK, UUID)<br> - timeslot_id (FK)<br> - customer_email (PII)<br> - confirmation_code (UNIQUE)<br> - booking_status = 'requested'<br>• timeslots (FK check)<br>• notification_events (INSERT)<br>**Indexes:**<br>• UNIQUE confirmation_code<br>• (timeslot_id, booking_status)<br>**Invariants:**<br>• COUNT ≤ 3 (app-level)<br>• email validation (RFC 5322)<br>• confirmation_code [A-Z0-9]{8} | → (TBD) | → (TBD) |
| **US-003: Переглядати мої бронювання** | **P4** modify_or_cancel_booking<br>(частково - перегляд)<br>**Потоки:**<br>• lookup_by_code (від E3)<br>• booking_details (вихід)<br>• booking_data (D3 read) | **Entities:**<br>• bookings (SELECT)<br> WHERE customer_email = ?<br> OR confirmation_code = ?<br>**Indexes:**<br>• (customer_email, created_at DESC)<br>• UNIQUE confirmation_code<br>**PII:** маскування email в логах | → (TBD) | → (TBD) |
| **US-004: Управляти переговорними кімнатами** | **P2** manage_availability<br>**Потоки:**<br>• shift_creation (вхід від E2)<br>• new_shift_data (вихід)<br>• write_rooms (D1)<br>• invalidate_cache (D4) | **Entities:**<br>• meeting_rooms (INSERT/UPDATE)<br> - meeting_room_id (PK)<br> - room_name (UNIQUE)<br> - capacity_persons<br> - equipment (JSONB)<br>**Indexes:**<br>• UNIQUE room_name<br>**Invariants:**<br>• capacity_persons > 0 | → (TBD) | → (TBD) |
| **US-005: Налаштовувати часові слоти** | **P2** manage_availability<br>**Потоки:**<br>• timeslot_data (вхід від E2)<br>• operation_result (вихід)<br>• write_timeslots (D2)<br>• timeslot_change → P5<br>• invalidate_cache (D4) | **Entities:**<br>• timeslots (INSERT/UPDATE)<br> - timeslot_id (PK)<br> - meeting_room_id (FK)<br> - start_at, end_at<br> - max_concurrent_bookings = 3<br>**Indexes:**<br>• UNIQUE (meeting_room_id, start_at, end_at)<br>**Invariants:**<br>• start_at < end_at<br>• не перекриваються | → (TBD) | → (TBD) |
| **US-006: Підтверджувати бронювання** | **P4** modify_or_cancel_booking<br>(частково - зміна status)<br>**Потоки:**<br>• update_booking (D3 write)<br>• booking_data (D3 read)<br>• cancel_event → P5 | **Entities:**<br>• bookings (UPDATE)<br> SET booking_status = 'confirmed'<br> WHERE booking_id = ?<br>**State transition:**<br>• requested → confirmed<br>**Actor:** space_manager | → (TBD) | → (TBD) |
| **US-007: Скасовувати бронювання** | **P4** modify_or_cancel_booking<br>**Потоки:**<br>• cancel_request (вхід від E1)<br>• booking_cancelled (вихід)<br>• update_booking (D3)<br>• cancel_event → P5<br>• invalidate_cache (D4) | **Entities:**<br>• bookings (UPDATE)<br> SET booking_status = 'cancelled',<br> cancelled_at = NOW(),<br> cancel_reason_code = ?<br>**State transitions:**<br>• requested → cancelled<br>• confirmed → cancelled<br>**Indexes:**<br>• (timeslot_id, booking_status) для capacity update | → (TBD) | → (TBD) |
| **US-008: Переглядати деталі бронювання** | **P4** modify_or_cancel_booking<br>**Потоки:**<br>• lookup_by_code (від E3)<br>• booking_details (вихід)<br>• booking_data (D3 read) | **Entities:**<br>• bookings (SELECT)<br> WHERE confirmation_code = ?<br>**Indexes:**<br>• UNIQUE confirmation_code<br>**Use case:** учасник зустрічі знає код | → (TBD) | → (TBD) |
| **US-009: Отримувати аналітику використання** | **(Не деталізовано в Level 0)**<br>Можливий процес:<br>**P5** view_logs_and_stats<br>**Потоки:**<br>• log_request (вхід від E3 admin)<br>• log_report (вихід)<br>• log_data (D4 logs read) | **Entities:**<br>• bookings (aggregate queries)<br>• timeslots (JOIN для звітів)<br>• meeting_rooms (JOIN)<br>**Queries:**<br>• GROUP BY booking_status<br>• GROUP BY cancel_reason_code<br>• COUNT per meeting_room | → (TBD) | → (TBD) |
| **US-010: Шукати кімнати за критеріями** | **P1** discover_services_and_timeslots<br>(розширена версія)<br>**Потоки:**<br>• availability_query + filters<br>• available_timeslots (filtered)<br>• read_rooms (D1 з фільтрами) | **Entities:**<br>• meeting_rooms<br> WHERE capacity_persons >= ?<br> AND equipment @> ?::jsonb<br> AND is_active = true<br>**Indexes:**<br>• is_active<br>• GIN index на equipment (JSONB) | → (TBD) | → (TBD) |

---

## Детальне трасування ключових User Stories

### US-001: Переглядати доступні часові слоти

**Acceptance Criteria → DFD:**

| AC                                                                                                                              | DFD Процес                         | Потік даних                                                                                                                  | Сховище                                              |
| ------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Given обрана meeting_room і діапазон дат<br>When я запитую доступні timeslots                                                   | P1 discover_services_and_timeslots | availability_query:<br>• date_from<br>• date_to<br>• meeting_room_id (optional)                                              | D1 meeting_rooms<br>D2 timeslots<br>D4 cache_or_view |
| Then бачу лише вільні слоти у форматі ISO 8601 (UTC)                                                                            | P1 → E1                            | available_timeslots:<br>• Array of timeslots<br>• start_time, end_time (ISO 8601)<br>• current_bookings<br>• available_spots | D2 timeslots JOIN<br>D3 bookings COUNT               |
| And слоти відсортовані за часом                                                                                                 | P1 (логіка)                        | ORDER BY start_time ASC                                                                                                      | D2 (query)                                           |
| And показано поточну кількість бронювань для кожного слоту                                                                      | P1 (логіка)                        | current_bookings field:<br>COUNT(D3) WHERE status != 'cancelled'                                                             | D3 bookings<br>(агрегація)                           |
| Given переговорна кімната з capacity=3<br>When я переглядаю timeslot з 2 існуючими бронюваннями                                 | P1 (фільтрація)                    | Slot включений у available_timeslots                                                                                         | D3 COUNT = 2                                         |
| Then слот показується як доступний (1 місце вільне)<br>And відображається "2/3 зайнято"                                         | P1 → E1                            | available_timeslots item:<br>• current_bookings: 2<br>• available_spots: 1                                                   | Розрахунок:<br>3 - COUNT(D3)                         |
| Given timeslot повністю зайнятий (3/3)<br>When я переглядаю список доступних слотів<br>Then цей слот не відображається у списку | P1 (фільтрація)                    | WHERE current_bookings < 3                                                                                                   | D3 COUNT >= 3<br>→ виключити                         |

**Performance вимоги (NFR):**

- P1 спочатку перевіряє D4 cache_or_view
- Якщо кеш валідний (expires_at > NOW()) → повертає cached_data
- Відповідь за ≤ 200ms
- Пагінація: 20 елементів по замовчуванню

---

### US-002: Створювати нове бронювання

**Acceptance Criteria → DFD:**

| AC                                                                                                                              | DFD Процес                                                                                                         | Потік даних                                                                                                                                                                | Сховище                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Given вільний timeslot з доступною capacity<br>When я надсилаю customer_name + валідний customer_email + timeslot_id            | **P3** create_booking<br>**P3.1** validate_request<br>**P3.2** check_timeslot_existence<br>**P3.3** check_capacity | booking_request:<br>• customer_name (2-100 chars)<br>• customer_email (validated)<br>• timeslot_id (UUID)<br><br>→ validated_request<br>→ timeslot_exists<br>→ capacity_ok | D2 timeslots (read)<br>D3 bookings (read COUNT)                                        |
| Then отримую 201 Created і booking зі status='requested'                                                                        | **P3.4** persist_booking<br>**P3.5** generate_confirmation<br>**P3.7** mask_personal_data                          | booking_persisted<br>→ booking_confirmation:<br>• booking_id (UUID)<br>• confirmation_code (8 chars)<br>• status='requested'<br>• created_at (ISO 8601)                    | D3 bookings (write INSERT)                                                             |
| And booking містить booking_id та confirmation_code                                                                             | P3.5                                                                                                               | booking_confirmation structure                                                                                                                                             | Generated in P3.4                                                                      |
| Given timeslot з 2 існуючими бронюваннями (capacity=3)<br>When я створюю нове бронювання<br>Then бронювання успішно створюється | **P3.3** check_capacity<br>**P3.3.1** validate_booking_data<br>**P3.3.2** write_to_bookings                        | Level 2:<br>• COUNT query: 2 < 3 ✓<br>• validated_booking<br>→ booking_record                                                                                              | D3 bookings:<br>SELECT COUNT(\*)<br>WHERE timeslot_id = ?<br>AND status != 'cancelled' |
| And capacity оновлюється до 3/3                                                                                                 | **P3.3.3** update_cache<br>**P3.4** persist_booking                                                                | invalidate_cache<br>→ D4 cache_or_view                                                                                                                                     | D4 (invalidate)<br>Next P1 call recalculates                                           |
| Given timeslot повністю зайнятий<br>When я намагаюся створити бронювання<br>Then отримую помилку з кодом 'slot_unavailable'     | **P3.3** check_capacity<br>**P3.3.1** validate_booking_data<br>**P3.8** reject_with_errors                         | COUNT = 3 >= 3<br>→ slot_unavailable<br>→ error_response:<br>• error_code: "slot_unavailable"<br>• HTTP 409 Conflict                                                       | D3 bookings (read COUNT)<br>No write                                                   |

**Детальна декомпозиція P3:**

1. **P3.1 validate_request**

   - Перевірка обов'язкових полів
   - Email regex validation
   - customer_name length (2-100)
   - Помилки: `invalid_email`, `missing_field`

2. **P3.2 check_timeslot_existence**

   - D2 read: SELECT \* WHERE timeslot_id = ?
   - Перевірка start_time > NOW()
   - Помилка: `timeslot_not_found` (404)

3. **P3.3 check_capacity** (Level 2 декомпозиція)

   - **P3.3.1** validate_booking_data:
     - D3 read: COUNT(\*) WHERE timeslot_id = ? AND status != 'cancelled'
     - IF count < 3 → validated_booking
     - ELSE → slot_unavailable
   - **P3.3.2** write_to_bookings:
     - D3 write: INSERT booking record
     - Generate booking_id (UUID)
     - Generate confirmation_code (8 chars)
   - **P3.3.3** update_cache:
     - D4 write: invalidate cache for timeslot_id
   - **P3.3.4** prepare_confirmation
   - **P3.3.5** forward_to_logging:
     - D4 logs write

4. **P3.4 persist_booking**

   - Final INSERT to D3
   - Return booking_persisted

5. **P3.5 generate_confirmation**

   - Format response structure
   - HTTP 201 Created

6. **P3.6 emit_domain_event**

   - D5 write: notification_outbox
   - Event: 'booking_created'

7. **P3.7 mask_personal_data** (Privacy-first)

   - Mask email: user@example.com → u**_@e_**.com
   - D5 write: masked_booking_data

8. **P3.8 reject_with_errors**
   - Standard error structure
   - Machine-readable error codes

**Privacy-first compliance:**

- Повний email зберігається тільки в D3 bookings
- D5 notification_outbox містить лише замасковані дані

**Performance-first compliance:**

- P3.4 інвалідує D4 cache
- P3 completion time ≤ 300ms

---

### US-007: Скасовувати бронювання

**Acceptance Criteria → DFD:**

| AC                                                                                            | DFD Процес                      | Потік даних                                                                             | Сховище                                  |
| --------------------------------------------------------------------------------------------- | ------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------- |
| Given активне бронювання<br>When я надсилаю cancel_request з booking_id або confirmation_code | **P4** modify_or_cancel_booking | cancel_request:<br>• booking_id (UUID)<br>OR<br>• confirmation_code (string)            | D3 bookings (read)                       |
| Then бронювання змінює status на 'cancelled'                                                  | P4 (логіка)                     | update_booking:<br>• status = 'cancelled'<br>• cancelled_at = NOW()                     | D3 bookings (write UPDATE)               |
| And я отримую підтвердження скасування                                                        | P4 → E1                         | booking_cancelled:<br>• booking_id<br>• status='cancelled'<br>• cancelled_at (ISO 8601) | Return from D3                           |
| And capacity слоту звільняється (2/3 → 1/3)                                                   | P4 → D4                         | invalidate_cache                                                                        | D4 cache_or_view<br>Next COUNT will be 2 |
| And учасники отримують notification                                                           | P4 → **P5** notify_parties      | cancel_event:<br>→ D5 notification_outbox                                               | D5 (write)<br>P5 processes async         |

**Потоки P4:**

- Вхід: cancel_request (від E1 meeting_organizer)
- D3 read: знайти booking by booking_id OR confirmation_code
- D3 write: UPDATE status='cancelled', cancelled_at=NOW()
- D4 write: invalidate_cache для timeslot_id
- D5 write: emit cancel_event
- Вихід: booking_cancelled (до E1)

---

## DFD Data Stores Mapping

| Сховище                    | Відповідає за                  | Використовується в US                  | Ключові процеси                                     |
| -------------------------- | ------------------------------ | -------------------------------------- | --------------------------------------------------- |
| **D1 meeting_rooms**       | Переговорні кімнати коворкінгу | US-001, US-004, US-010                 | P1 (read), P2 (write)                               |
| **D2 timeslots**           | Часові слоти для бронювання    | US-001, US-002, US-005                 | P1 (read), P2 (write), P3.2 (read)                  |
| **D3 bookings**            | Бронювання користувачів        | US-002, US-003, US-006, US-007, US-008 | P3 (write), P4 (read/write), P3.3 (read COUNT)      |
| **D4 cache_or_view**       | Кеш доступності (performance)  | US-001, US-002, US-007                 | P1 (read/write), P3.4 (invalidate), P4 (invalidate) |
| **D5 notification_outbox** | Черга повідомлень (privacy)    | US-002, US-007                         | P3.6 (write), P4 (write), P5 (read/process)         |
| **D4 logs**                | Аудит та аналітика             | US-009                                 | P3.3.5 (write), P5 view_logs (read)                 |

---

## DFD External Entities Mapping

| Зовнішня сутність           | Роль у системі                             | Взаємодіє з процесами | User Stories                                 |
| --------------------------- | ------------------------------------------ | --------------------- | -------------------------------------------- |
| **E1 meeting_organizer**    | Організатор зустрічей, основний користувач | P1, P3, P4            | US-001, US-002, US-003, US-007               |
| **E2 space_manager**        | Менеджер коворкінгу, адміністратор         | P2                    | US-004, US-005, US-006                       |
| **E3 participant**          | Учасник зустрічі, перегляд деталей         | P4                    | US-008                                       |
| **E4 notification_gateway** | Зовнішній сервіс відправки повідомлень     | P5                    | (Out of scope v1.0, але архітектурно готово) |

---

## Capacity Logic Traceability

**Variant:** P=1 (непарне) → capacity = 3

| Requirement                       | DFD реалізація                                      | Level              | Data Store                            |
| --------------------------------- | --------------------------------------------------- | ------------------ | ------------------------------------- |
| До 3 одночасних бронювань на слот | P3.3 check_capacity<br>P3.3.1 validate_booking_data | Level 1<br>Level 2 | D3 bookings<br>Constraint: COUNT <= 3 |
| Показ "2/3 зайнято"               | P1 discover_services                                | Level 0            | D3 (COUNT aggregation)                |
| Відмова при 3/3                   | P3.3.1 → slot_unavailable → P3.8                    | Level 2 → Level 1  | D3 COUNT >= 3                         |
| Звільнення при cancel             | P4 → status='cancelled'<br>→ invalidate_cache       | Level 0            | D3 UPDATE<br>D4 invalidate            |

**SQL логіка:**

```sql
-- P3.3.1 check
SELECT COUNT(*) as current_count
FROM bookings
WHERE timeslot_id = ?
  AND booking_status != 'cancelled';

-- Condition
IF current_count < 3 THEN
  -- proceed to P3.3.2
ELSE
  -- reject with slot_unavailable
END IF;
```

---

## NFR Compliance Traceability

### Performance-first

| NFR вимога                 | DFD реалізація  | Процес                    | Сховище                                                                         |
| -------------------------- | --------------- | ------------------------- | ------------------------------------------------------------------------------- |
| GET /timeslots за ≤ 200ms  | P1 з кешуванням | P1 → D4 cache check first | D4 cache_or_view                                                                |
| POST /bookings за ≤ 300ms  | P3 оптимізовано | P3 (всі підпроцеси)       | Indexed queries on D3                                                           |
| Кеш TTL = 30 секунд        | D4 expires_at   | P1 перевіряє expires_at   | D4.expires_at                                                                   |
| Розмір відповіді ≤ 50KB    | P1 пагінація    | P1 LIMIT 20               | D2/D3 query                                                                     |
| Індекси на критичних полях | DB schema       | -                         | D3: (timeslot_id, status)<br>D3: (customer_email)<br>D2: (start_time, end_time) |

### Privacy-first

| NFR вимога                        | DFD реалізація          | Процес    | Сховище             |
| --------------------------------- | ----------------------- | --------- | ------------------- |
| Маскування email                  | P3.7 mask_personal_data | P3.7      | D5 (masked only)    |
| Повний email тільки в bookings    | Архітектурне обмеження  | P3.4 → D3 | D3 bookings         |
| Не передавати PII в notifications | P5 використовує D5      | P5 → E4   | D5 payload (masked) |

### Стабільні коди помилок

| Код помилки          | Генерується в                                       | Процес      | HTTP Status     |
| -------------------- | --------------------------------------------------- | ----------- | --------------- |
| `invalid_email`      | P3.1 validate_request                               | P3.1 → P3.8 | 400 Bad Request |
| `missing_field`      | P3.1 validate_request                               | P3.1 → P3.8 | 400 Bad Request |
| `timeslot_not_found` | P3.2 check_timeslot                                 | P3.2 → P3.8 | 404 Not Found   |
| `slot_unavailable`   | P3.3 check_capacity<br>P3.3.1 validate_booking_data | P3.3 → P3.8 | 409 Conflict    |
| `booking_not_found`  | P4 modify_or_cancel                                 | P4 → E1     | 404 Not Found   |

---

## Balancing Verification

### Level 0 → Level 1 (P3 декомпозиція)

| Level 0 потік                      | Level 1 еквівалент      | Збалансовано |
| ---------------------------------- | ----------------------- | ------------ |
| booking_request (вхід від E1)      | → P3.1 validate_request | ✅           |
| booking_confirmation (вихід до E1) | P3.5 → P3.7 → E1        | ✅           |
| timeslot_reference (D2 read)       | P3.2 ↔ D2               | ✅           |
| booking_created (D3 write)         | P3.4 → D3               | ✅           |
| booking_event (до P5)              | P3.6 → P5               | ✅           |
| invalidate_cache (D4 write)        | P3.4 → D4               | ✅           |

### Level 1 → Level 2 (P3.3 декомпозиція)

| Level 1 потік                   | Level 2 еквівалент             | Збалансовано |
| ------------------------------- | ------------------------------ | ------------ |
| timeslot_exists (вхід від P3.2) | → P3.3.1 validate_booking_data | ✅           |
| capacity_ok (вихід до P3.4)     | P3.3.3 → output                | ✅           |
| slot_unavailable (до P3.8)      | P3.3.1 → output (error path)   | ✅           |
| count_bookings (D3 read)        | P3.3.1 ↔ D3                    | ✅           |
| booking_record (D3 write)       | P3.3.2 → D3                    | ✅           |
| cache_entry (D4 write)          | P3.3.3 → D4                    | ✅           |
| log_entry (D4 logs write)       | P3.3.5 → D4                    | ✅           |

**Висновок:** Всі потоки збалансовані між рівнями декомпозиції.

---

## Примітки для майбутнього розвитку

### ERD Entities (наступний крок)

На основі DFD сховищ, основні сутності ERD:

- `meeting_rooms` (від D1)
- `timeslots` (від D2)
- `bookings` (від D3)
- `cache_entries` (від D4)
- `notification_events` (від D5)
- `audit_logs` (від D4 logs)

Зв'язки:

- timeslots N:1 meeting_rooms
- bookings N:1 timeslots
- notification_events N:1 bookings

### API Endpoints (наступний крок)

- `GET /api/v1/timeslots` → P1
- `POST /api/v1/bookings` → P3
- `GET /api/v1/bookings/{id}` → P4
- `DELETE /api/v1/bookings/{id}` → P4
- `GET /api/v1/bookings?email={email}` → P4 (або новий процес)
- `POST /api/v1/timeslots` → P2 (space_manager)
- `GET /api/v1/rooms` → P1 (варіант)

### Test Scenarios (наступний крок)

- **TS-001:** Happy path створення бронювання (capacity 0→1)
- **TS-002:** Створення при 2/3 capacity (2→3)
- **TS-003:** Відмова при 3/3 capacity (slot_unavailable)
- **TS-004:** Concurrent booking requests (race condition)
- **TS-005:** Валідація invalid_email
- **TS-006:** Скасування та звільнення capacity (3→2)
- **TS-007:** Cache hit performance (≤200ms)
- **TS-008:** Privacy check (masked email in D5)

---

**Кінець оновленого Traceability Matrix**
