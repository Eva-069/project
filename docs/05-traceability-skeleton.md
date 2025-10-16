# Traceability Matrix - CoworkSpace Booking System

**Дата оновлення:** 2025-01-15  
**Версія:** 2.0 (додано DFD процеси)

---

## User Stories → Architecture Components

| User Story                                    | DFD Process                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | ERD Entities | API Endpoints | Test Scenarios |
| --------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | ------------- | -------------- |
| **US-001: Переглядати доступні часові слоти** | **P1** discover_services_and_timeslots<br>**Потоки:**<br>• availability_query (вхід)<br>• available_timeslots (вихід)<br>• read_rooms (D1)<br>• read_timeslots (D2)<br>• query_cache / cached_data (D4)                                                                                                                                                                                                                                                                                                                                                                                                                                                     | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-002: Створювати нове бронювання**        | **P3** create_booking<br>**Level 1:**<br>• P3.1 validate_request<br>• P3.2 check_timeslot_existence<br>• P3.3 check_capacity<br>• P3.4 persist_booking<br>• P3.5 generate_confirmation<br>• P3.6 emit_domain_event<br>• P3.7 mask_personal_data<br>• P3.8 reject_with_errors<br>**Level 2 (P3.3):**<br>• P3.3.1 validate_booking_data<br>• P3.3.2 write_to_bookings<br>• P3.3.3 update_cache<br>• P3.3.4 prepare_confirmation<br>• P3.3.5 forward_to_logging<br>**Потоки:**<br>• booking_request (вхід)<br>• booking_confirmation (вихід)<br>• slot_unavailable (помилка)<br>• timeslot_reference (D2)<br>• booking_created (D3)<br>• invalidate_cache (D4) | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-003: Переглядати мої бронювання**        | **P4** modify_or_cancel_booking<br>(частково - перегляд)<br>**Потоки:**<br>• lookup_by_code (від E3)<br>• booking_details (вихід)<br>• booking_data (D3 read)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-004: Управляти переговорними кімнатами** | **P2** manage_availability<br>**Потоки:**<br>• shift_creation (вхід від E2)<br>• new_shift_data (вихід)<br>• write_rooms (D1)<br>• invalidate_cache (D4)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-005: Налаштовувати часові слоти**        | **P2** manage_availability<br>**Потоки:**<br>• timeslot_data (вхід від E2)<br>• operation_result (вихід)<br>• write_timeslots (D2)<br>• timeslot_change → P5<br>• invalidate_cache (D4)                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-006: Підтверджувати бронювання**         | **P4** modify_or_cancel_booking<br>(частково - зміна status)<br>**Потоки:**<br>• update_booking (D3 write)<br>• booking_data (D3 read)<br>• cancel_event → P5                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-007: Скасовувати бронювання**            | **P4** modify_or_cancel_booking<br>**Потоки:**<br>• cancel_request (вхід від E1)<br>• booking_cancelled (вихід)<br>• update_booking (D3)<br>• cancel_event → P5<br>• invalidate_cache (D4)                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-008: Переглядати деталі бронювання**     | **P4** modify_or_cancel_booking<br>**Потоки:**<br>• lookup_by_code (від E3)<br>• booking_details (вихід)<br>• booking_data (D3 read)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-009: Отримувати аналітику використання** | **(Не деталізовано в Level 0)**<br>Можливий процес:<br>**P5** view_logs_and_stats<br>**Потоки:**<br>• log_request (вхід від E3 admin)<br>• log_report (вихід)<br>• log_data (D4 logs read)                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | → (TBD)      | → (TBD)       | → (TBD)        |
| **US-010: Шукати кімнати за критеріями**      | **P1** discover_services_and_timeslots<br>(розширена версія)<br>**Потоки:**<br>• availability_query + filters<br>• available_timeslots (filtered)<br>• read_rooms (D1 з фільтрами)                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | → (TBD)      | → (TBD)       | → (TBD)        |

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
