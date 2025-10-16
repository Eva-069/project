# DFD Dictionary - CoworkSpace Booking System

## Нотація: Yourdon-DeMarco

**Дата створення:** 2025-01-15  
**Capacity:** 3 одночасних бронювання на timeslot  
**NFR акцент:** Performance-first (кешування, оптимізація)

---

## 1. Зовнішні сутності (External Entities)

### E1 meeting_organizer

**Опис:** Користувач, який організовує зустрічі та створює бронювання переговорних кімнат.

**Вхідні потоки:**

- `available_timeslots` (від P1)
- `booking_confirmation` (від P3)
- `booking_cancelled` (від P4)
- `booking_details` (від P4)
- `error_response` (від P3.8)

**Вихідні потоки:**

- `availability_query` (до P1)
- `booking_request` (до P3)
- `cancel_request` (до P4)

---

### E2 space_manager

**Опис:** Менеджер коворкінгу, який керує переговорними кімнатами та часовими слотами.

**Вхідні потоки:**

- `operation_result` (від P2)
- `new_shift_data` (від P2)

**Вихідні потоки:**

- `shift_creation` (до P2)
- `timeslot_data` (до P2)

---

### E3 participant

**Опис:** Учасник зустрічі, який може переглянути деталі бронювання за кодом підтвердження.

**Вхідні потоки:**

- `booking_details` (від P4)

**Вихідні потоки:**

- `lookup_by_code` (до P4)

---

### E4 notification_gateway

**Опис:** Зовнішній сервіс для відправки повідомлень (email/SMS).

**Вхідні потоки:**

- `notification_payload` (від P5)

**Вихідні потоки:** Немає

---

## 2. Процеси Level 0

### P1 discover_services_and_timeslots

**Опис:** Пошук та відображення доступних переговорних кімнат і часових слотів.

**Вхідні потоки:**

- `availability_query` (від E1): Містить date_from, date_to, optional meeting_room_id

**Вихідні потоки:**

- `available_timeslots` (до E1): Список вільних слотів з інформацією про capacity

**Взаємодія зі сховищами:**

- Читає: D1 (meeting_rooms), D2 (timeslots), D4 (cache_or_view)
- Пише: D4 (update_cache)

**Бізнес-логіка:**

- Перевіряє кеш на наявність актуальних даних
- Фільтрує слоти де current_bookings < 3
- Повертає відсортовані за часом результати

---

### P2 manage_availability

**Опис:** Управління переговорними кімнатами та створення/редагування часових слотів.

**Вхідні потоки:**

- `shift_creation` (від E2): Дані про нові кімнати або слоти
- `timeslot_data` (від E2): Зміни в існуючих слотах

**Вихідні потоки:**

- `operation_result` (до E2): Результат операції (success/error)
- `new_shift_data` (до E2): Створені дані
- `timeslot_change` (до P5): Подія для нотифікації

**Взаємодія зі сховищами:**

- Читає/Пише: D1 (meeting_rooms), D2 (timeslots)
- Пише: D4 (invalidate_cache)

**Бізнес-логіка:**

- Валідує дані кімнат і слотів
- Перевіряє унікальність timeslot (meeting_room_id + start_time)
- Інвалідує кеш після змін

---

### P3 create_booking

**Опис:** Створення нового бронювання з перевіркою capacity та валідацією.

**Вхідні потоки:**

- `booking_request` (від E1): customer_name, customer_email, timeslot_id

**Вихідні потоки:**

- `booking_confirmation` (до E1): booking_id, confirmation_code, status='requested'
- `booking_event` (до P5): Подія для нотифікації
- `error_response` (до E1): Код помилки при невдачі

**Взаємодія зі сховищами:**

- Читає: D2 (timeslots) - перевірка існування
- Читає/Пише: D3 (bookings) - підрахунок та створення
- Пише: D5 (notification_outbox)

**Бізнес-логіка:**

- Валідація email формату
- Перевірка що timeslot існує
- **Capacity check:** COUNT(bookings WHERE timeslot_id = ? AND status != 'cancelled') < 3
- Генерація confirmation_code (8 символів alphanumeric)

**Деталізація:** Див. DFD Level 1

---

### P4 modify_or_cancel_booking

**Опис:** Скасування існуючих бронювань та перегляд деталей.

**Вхідні потоки:**

- `cancel_request` (від E1): booking_id або confirmation_code
- `lookup_by_code` (від E3): confirmation_code

**Вихідні потоки:**

- `cancellation_result` (до E1): Результат скасування
- `booking_details` (до E1/E3): Інформація про бронювання
- `cancel_event` (до P5): Подія для нотифікації

**Взаємодія зі сховищами:**

- Читає/Пише: D3 (bookings) - оновлення status на 'cancelled'
- Пише: D4 (invalidate_cache)

**Бізнес-логіка:**

- Перевірка що бронювання існує
- Оновлення status='cancelled', cancelled_at=NOW()
- Інвалідація кешу доступності
- Емісія події для сповіщень

---

### P5 notify_parties

**Опис:** Відправка повідомлень учасникам через notification_gateway.

**Вхідні потоки:**

- `booking_event` (від P3): Нове бронювання
- `cancel_event` (від P4): Скасування
- `timeslot_change` (від P2): Зміни в розкладі

**Вихідні потоки:**

- `notification_payload` (до E4): Замасковані дані для відправки

**Взаємодія зі сховищами:**

- Читає/Пише: D5 (notification_outbox) - черга повідомлень

**Бізнес-логіка:**

- Читає події з D5
- Маскує персональні дані (email: u**_@e_**.com)
- Формує payload для зовнішнього gateway
- Privacy-first: не передає повні PII

---

## 3. Процеси Level 1 (декомпозиція P3)

### P3.1 validate_request

**Опис:** Валідація вхідних даних booking_request.

**Вхідні потоки:**

- `booking_request` (від E1)

**Вихідні потоки:**

- `validated_request` (до P3.2): Перевірені дані
- `validation_errors` (до P3.8): Помилки валідації

**Бізнес-логіка:**

- Перевірка обов'язкових полів: customer_name, customer_email, timeslot_id
- Валідація email regex: `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
- Перевірка довжини customer_name (2-100 символів)
- Коди помилок: `invalid_email`, `missing_field`, `invalid_format`

---

### P3.2 check_timeslot_existence

**Опис:** Перевірка що timeslot існує в системі.

**Вхідні потоки:**

- `validated_request` (від P3.1)

**Вихідні потоки:**

- `timeslot_exists` (до P3.3): Підтверджені дані
- `timeslot_not_found` (до P3.8): Слот не знайдено

**Взаємодія зі сховищами:**

- Читає: D2 (timeslots) - SELECT \* WHERE timeslot_id = ?

**Бізнес-логіка:**

- Запит до D2 за timeslot_id
- Перевірка що start_time > NOW() (не минулий)
- Код помилки: `timeslot_not_found` (404)

---

### P3.3 check_capacity

**Опис:** Перевірка що кількість бронювань < 3 для даного слоту.

**Вхідні потоки:**

- `timeslot_exists` (від P3.2)

**Вихідні потоки:**

- `capacity_ok` (до P3.4): Є місце
- `slot_unavailable` (до P3.8): Слот повний

**Взаємодія зі сховищами:**

- Читає: D3 (bookings) - COUNT запит

**Бізнес-логіка:**

- SELECT COUNT(\*) FROM bookings WHERE timeslot_id = ? AND status != 'cancelled'
- Якщо count < 3 → capacity_ok
- Якщо count >= 3 → slot_unavailable (код помилки: 409 Conflict)

**Деталізація:** Див. DFD Level 2

---

### P3.4 persist_booking

**Опис:** Збереження запису бронювання в базу даних.

**Вхідні потоки:**

- `capacity_ok` (від P3.3)

**Вихідні потоки:**

- `booking_persisted` (до P3.5): Створений запис
- `booking_created` (до P3.6): Подія для логування

**Взаємодія зі сховищами:**

- Пише: D3 (bookings) - INSERT запит
- Пише: D4 (cache_or_view) - invalidate_cache

**Бізнес-логіка:**

- Генерація booking_id (UUID)
- Генерація confirmation_code (8 символів)
- status = 'requested', created_at = NOW()
- Інвалідація кешу доступності

---

### P3.5 generate_confirmation

**Опис:** Формування підтвердження для користувача.

**Вхідні потоки:**

- `booking_persisted` (від P3.4)

**Вихідні потоки:**

- `booking_confirmation` (до E1): Фінальна відповідь

**Бізнес-логіка:**

- Формує response: booking_id, confirmation_code, status, created_at
- HTTP 201 Created
- Структура: `{"booking_id": "...", "confirmation_code": "ABC12345", "status": "requested"}`

---

### P3.6 emit_domain_event

**Опис:** Емісія події про створення бронювання.

**Вхідні потоки:**

- `booking_created` (від P3.4)

**Вихідні потоки:**

- Немає прямих (event в D5)

**Взаємодія зі сховищами:**

- Пише: D5 (notification_outbox) - log_event_data

**Бізнес-логіка:**

- Записує подію: event_type='booking_created', payload=booking_data
- Асинхронна обробка через P5 notify_parties

---

### P3.7 mask_personal_data

**Опис:** Маскування персональних даних перед відправкою в notification_outbox (privacy-first).

**Вхідні потоки:**

- `confirmation_data` (від P3.5)

**Вихідні потоки:**

- `booking_confirmation` (до E1): Повні дані для користувача
- `masked_booking_data` (до D5): Замасковані дані

**Взаємодія зі сховищами:**

- Пише: D5 (notification_outbox) - masked_booking_data

**Бізнес-логіка:**

- Маскує email: `user@example.com` → `u***@e***.com`
- Зберігає: booking_id, confirmation_code, masked_email
- **Privacy-first:** повний email залишається тільки в D3

---

### P3.8 reject_with_errors

**Опис:** Обробка помилок та формування error response.

**Вхідні потоки:**

- `validation_errors` (від P3.1)
- `timeslot_not_found` (від P3.2)
- `slot_unavailable` (від P3.3)

**Вихідні потоки:**

- `error_response` (до E1): Машинний код помилки

**Бізнес-логіка:**

- Формує стандартизовану структуру помилки
- Формат: `{"error_code": "slot_unavailable", "message": "...", "details": {...}}`
- HTTP статус коди:
  - `invalid_email` → 400 Bad Request
  - `timeslot_not_found` → 404 Not Found
  - `slot_unavailable` → 409 Conflict

---

## 4. Процеси Level 2 (декомпозиція P3.3)

### P3.3.1 validate_booking_data

**Опис:** Перевірка доступності слоту через підрахунок активних бронювань.

**Вхідні потоки:**

- `timeslot_exists` (від P3.2 через P3.3)

**Вихідні потоки:**

- `validated_booking` (до P3.3.2): Дозвіл на створення
- `slot_unavailable` (до P3.8): Немає місць

**Взаємодія зі сховищами:**

- Читає: D2 (bookings) - COUNT запит

**Бізнес-логіка:**

- Запит: `SELECT COUNT(*) FROM bookings WHERE timeslot_id = ? AND status != 'cancelled'`
- Перевірка: `count < 3`
- **Capacity = 3:** максимум 3 одночасних бронювання
- Race condition protection: використання транзакцій або optimistic locking

---

### P3.3.2 write_to_bookings

**Опис:** Фізичний запис бронювання в сховище D2.

**Вхідні потоки:**

- `validated_booking` (від P3.3.1)

**Вихідні потоки:**

- `booking_summary` (до P3.3.3): Дані для кешу
- `confirmation_data` (до P3.3.4): Дані для підтвердження
- `log_event_data` (до P3.3.5): Дані для логування

**Взаємодія зі сховищами:**

- Пише: D2 (bookings) - INSERT query
- Читає: D2 (bookings) - повертає створений запис

**Бізнес-логіка:**

- INSERT INTO bookings (booking_id, customer_name, customer_email, timeslot_id, confirmation_code, status, created_at)
- Генерує booking_id (UUID v4)
- Генерує confirmation_code (8 символів: uppercase + digits)

---

### P3.3.3 update_cache

**Опис:** Оновлення кешу доступності після створення бронювання.

**Вхідні потоки:**

- `booking_summary` (від P3.3.2)

**Вихідні потоки:**

- `capacity_ok` (до P3.4): Фінальне підтвердження

**Взаємодія зі сховищами:**

- Пише: D5 (cache_or_view) - cache_entry

**Бізнес-логіка:**

- Інвалідує кеш для timeslot_id
- Оновлює projection доступності
- **Performance-first:** забезпечує актуальність кешу

---

### P3.3.4 prepare_confirmation

**Опис:** Підготовка даних підтвердження для користувача.

**Вхідні потоки:**

- `confirmation_data` (від P3.3.2)

**Вихідні потоки:**

- Передає дані далі по ланцюгу до P3.5

**Бізнес-логіка:**

- Формує структуру відповіді з усіма необхідними полями
- Додає метадані: created_at, status

---

### P3.3.5 forward_to_logging

**Опис:** Логування події створення бронювання.

**Вхідні потоки:**

- `log_event_data` (від P3.3.2)

**Вихідні потоки:**

- Немає прямих

**Взаємодія зі сховищами:**

- Пише: D4 (logs) - log_entry

**Бізнес-логіка:**

- Записує: timestamp, event_type='booking_created', booking_id, user_context
- Для аналітики та аудиту

---

## 5. Сховища даних (Data Stores)

### D1 meeting_rooms

**Опис:** Переговорні кімнати коворкінгу.

**Ключові атрибути:**

- `meeting_room_id` (PK, UUID)
- `room_name` (string, unique)
- `capacity_persons` (integer) - місткість людей
- `equipment` (JSON array) - обладнання
- `is_active` (boolean)
- `created_at`, `updated_at` (timestamp)

**Унікальність:** room_name

**Інваріанти:**

- capacity_persons > 0
- is_active по замовчуванню = true

**Індекси:**

- PRIMARY KEY (meeting_room_id)
- UNIQUE INDEX (room_name)
- INDEX (is_active)

---

### D2 timeslots

**Опис:** Часові слоти для бронювання переговорних кімнат.

**Ключові атрибути:**

- `timeslot_id` (PK, UUID)
- `meeting_room_id` (FK → D1.meeting_room_id)
- `start_time` (timestamp, ISO 8601 UTC)
- `end_time` (timestamp, ISO 8601 UTC)
- `max_concurrent_bookings` (integer) - завжди = 3
- `created_at`, `updated_at` (timestamp)

**Унікальність:** (meeting_room_id, start_time)

**Інваріанти:**

- end_time > start_time
- max_concurrent_bookings = 3 (capacity з варіанту)
- start_time >= NOW() (не минулі слоти)

**Індекси:**

- PRIMARY KEY (timeslot_id)
- UNIQUE INDEX (meeting_room_id, start_time)
- INDEX (start_time, end_time) - для пошуку доступності
- INDEX (meeting_room_id)

---

### D3 bookings

**Опис:** Бронювання переговорних кімнат користувачами.

**Ключові атрибути:**

- `booking_id` (PK, UUID)
- `timeslot_id` (FK → D2.timeslot_id)
- `customer_name` (string, 2-100 символів)
- `customer_email` (string, validated email)
- `confirmation_code` (string, 8 символів, unique)
- `booking_status` (enum: 'requested', 'confirmed', 'cancelled')
- `created_at` (timestamp)
- `confirmed_at` (timestamp, nullable)
- `cancelled_at` (timestamp, nullable)

**Унікальність:**

- booking_id (PK)
- confirmation_code (для lookup)

**Інваріанти:**

- Для одного timeslot_id може бути максимум 3 активних бронювання (status != 'cancelled')
- customer_email має бути валідним email форматом
- confirmation_code генерується один раз при створенні

**Індекси:**

- PRIMARY KEY (booking_id)
- UNIQUE INDEX (confirmation_code)
- INDEX (timeslot_id, booking_status) - для capacity check
- INDEX (customer_email) - для пошуку бронювань користувача
- INDEX (created_at DESC) - для сортування

**Constraint:**

```sql
CHECK (
  (SELECT COUNT(*) FROM bookings
   WHERE timeslot_id = NEW.timeslot_id
   AND booking_status != 'cancelled') <= 3
)
```

---

### D4 cache_or_view (performance-first)

**Опис:** Кеш або матеріалізоване представлення доступних слотів для швидкого пошуку.

**Ключові атрибути:**

- `cache_key` (string, PK) - напр. "availability:2025-01-15:room-123"
- `cached_data` (JSON) - список доступних слотів
- `current_bookings_count` (integer) - для швидкого показу "2/3 зайнято"
- `expires_at` (timestamp) - TTL = 30 секунд
- `created_at` (timestamp)

**Унікальність:** cache_key

**Інваріанти:**

- TTL = 30 секунд (з NFR checklist)
- Інвалідується при будь-яких змінах в D3 bookings або D2 timeslots

**Використання:**

- P1 спочатку перевіряє кеш
- Якщо expires_at > NOW() → повертає cached_data
- Якщо немає або застарів → запит до D2 + D3, оновлення кешу

---

### D5 notification_outbox

**Опис:** Черга подій для асинхронної відправки повідомлень (outbox pattern).

**Ключові атрибути:**

- `event_id` (PK, UUID)
- `event_type` (enum: 'booking_created', 'booking_cancelled', 'timeslot_changed')
- `aggregate_id` (UUID) - booking_id або timeslot_id
- `payload` (JSON) - замасковані дані для notification_gateway
- `processed` (boolean) - чи оброблено
- `processed_at` (timestamp, nullable)
- `created_at` (timestamp)

**Унікальність:** event_id

**Інваріанти:**

- payload містить тільки замасковані PII (privacy-first)
- processed = false при створенні
- Подія обробляється P5 notify_parties

**Індекси:**

- PRIMARY KEY (event_id)
- INDEX (processed, created_at) - для вибірки необроблених
- INDEX (aggregate_id) - для пошуку подій конкретного бронювання

**Privacy-first:**

- email маскується: `user@example.com` → `u***@e***.com`
- customer_name може бути скорочений до ініціалів

---

### D4 logs (додаткове сховище)

**Опис:** Журнал подій для аудиту та аналітики.

**Ключові атрибути:**

- `log_id` (PK, auto-increment)
- `timestamp` (timestamp, indexed)
- `event_type` (string)
- `booking_id` (UUID, nullable)
- `user_context` (JSON) - IP, user-agent
- `payload` (JSON) - деталі події

**Індекси:**

- PRIMARY KEY (log_id)
- INDEX (timestamp DESC)
- INDEX (booking_id)
- INDEX (event_type)

---

## 6. Потоки даних (Data Flows)

### availability_query

**Склад атрибутів:**

- `date_from` (ISO 8601 date)
- `date_to` (ISO 8601 date)
- `meeting_room_id` (UUID, optional)

**Призначення:** Запит на пошук доступних слотів

---

### available_timeslots

**Склад атрибутів:**

- `timeslots` (array):
  - `timeslot_id` (UUID)
  - `meeting_room_id` (UUID)
  - `room_name` (string)
  - `start_time` (ISO 8601 UTC)
  - `end_time` (ISO 8601 UTC)
  - `current_bookings` (integer, 0-3)
  - `available_spots` (integer, 3 - current_bookings)

**Призначення:** Відповідь зі списком доступних слотів

**Performance-first:** Пагінація 20 елементів, розмір відповіді ≤ 50KB

---

### booking_request

**Склад атрибутів:**

- `customer_name` (string, 2-100 символів)
- `customer_email` (string, email format)
- `timeslot_id` (UUID)

**Призначення:** Запит на створення бронювання

**Валідація:** Виконується в P3.1

---

### booking_confirmation

**Склад атрибутів:**

- `booking_id` (UUID)
- `confirmation_code` (string, 8 символів)
- `booking_status` (enum: 'requested')
- `timeslot` (object):
  - `start_time` (ISO 8601 UTC)
  - `end_time` (ISO 8601 UTC)
  - `room_name` (string)
- `created_at` (ISO 8601 UTC)

**Призначення:** Підтвердження створеного бронювання

**HTTP статус:** 201 Created

---

### cancel_request

**Склад атрибутів:**

- `booking_id` (UUID) OR `confirmation_code` (string)

**Призначення:** Запит на скасування бронювання

---

### booking_cancelled

**Склад атрибутів:**

- `booking_id` (UUID)
- `status` (string: 'cancelled')
- `cancelled_at` (ISO 8601 UTC)

**Призначення:** Підтвердження скасування

---

### slot_unavailable

**Склад атрибутів:**

- `error_code` (string: "slot_unavailable")
- `message` (string: "This timeslot is fully booked")
- `details` (object):
  - `timeslot_id` (UUID)
  - `current_bookings` (integer: 3)
  - `max_capacity` (integer: 3)

**Призначення:** Помилка коли слот повністю зайнятий

**HTTP статус:** 409 Conflict

---

### error_response

**Склад атрибутів:**

- `error_code` (string: машинний код)
- `message` (string: людино-читабельне пояснення)
- `details` (object: контекст помилки)

**Можливі коди:**

- `invalid_email` (400)
- `missing_field` (400)
- `timeslot_not_found` (404)
- `slot_unavailable` (409)
- `booking_not_found` (404)

**Призначення:** Стандартизована структура помилок (з NFR checklist)

---

### notification_payload

**Склад атрибутів:**

- `event_type` (string)
- `recipient_email` (string: замаскований)
- `template_id` (string)
- `variables` (object):
  - `confirmation_code` (string)
  - `room_name` (string)
  - `start_time` (ISO 8601 UTC)

**Призначення:** Дані для відправки через E4 notification_gateway

**Privacy-first:** email завжди замаскований

---

## 7. Трасування до User Stories

### US-001: Переглядати доступні часові слоти

**DFD процеси:**

- Level 0: P1 discover_services_and_timeslots
- Потоки: availability_query → P1 → available_timeslots

**Сховища:**

- D1 meeting_rooms (читання)
- D2 timeslots (читання)
- D4 cache_or_view (читання/запис)

**Capacity відображення:**

- Показує current_bookings / 3 для кожного слоту
- Фільтрує слоти де current_bookings >= 3

---

### US-002: Створювати нове бронювання

**DFD процеси:**

- Level 0: P3 create_booking
- Level 1: P3.1 → P3.2 → P3.3 → P3.4 → P3.5 → P3.7
- Level 2: P3.3.1 → P3.3.2 → P3.3.3 → P3.3.4 → P3.3.5
- Потоки: booking_request → P3 → booking_confirmation | slot_unavailable

**Сховища:**

- D2 timeslots (читання - перевірка існування)
- D3 bookings (читання - capacity check, запис - створення)
- D4 cache_or_view (запис - інвалідація)
- D5 notification_outbox (запис - подія)

**Capacity логіка:**

- P3.3 / P3.3.1: COUNT(bookings WHERE timeslot_id = ? AND status != 'cancelled') < 3

---

### US-003: Переглядати мої бронювання

**DFD процеси:**

- (Не деталізовано в Level 0, але логічно належить до P1 або окремого P6)
- Потік: query_by_email → [process] → bookings_list

**Сховища:**

- D3 bookings (читання за customer_email)

---

## 8. NFR Compliance

### Performance-first вимоги:

**Кешування (D4 cache_or_view):**

- TTL = 30 секунд
- P1 спочатку перевіряє кеш
- Інвалідація при будь-яких змінах через P2, P3, P4

**Відповіді:**

- GET /timeslots за ≤ 200ms (з кешем)
- POST /bookings за ≤ 300ms
- Розмір відповіді ≤ 50KB

**Пагінація:**

- Limit = 20 елементів по замовчуванню
- Max limit = 50

**Індекси:**

- D3: (timeslot_id, booking_status) для capacity check
- D3: (customer_email) для пошуку бронювань
- D2: (start_time, end_time) для пошуку доступності

---

### Privacy-first реалізація:

**P3.7 mask_personal_data:**

- Маскує email перед D5: user@example.com → u**_@e_**.com
- Повний email тільки в D3 bookings

**D5 notification_outbox:**

- Зберігає лише замасковані PII
- Payload не містить повних персональних даних

---

## 9. Balancing Check

**Level 0 → Level 1 (P3):**

- ✅ booking_request (вхід) → P3.1 validate_request
- ✅ booking_confirmation (вихід) → P3.7 → E1
- ✅ timeslot_reference (D2) → P3.2 check_timeslot
- ✅ booking_created (D3) → P3.4 persist_booking
- ✅ booking_event (P5) → P3.6 emit_event
- ✅ error_response → P3.8 reject_with_errors

**Level 1 → Level 2 (P3.3):**

- ✅ timeslot_exists (вхід) → P3.3.1 validate_booking_data
- ✅ capacity_ok (вихід) → P3.3.3 → P3.4
- ✅ slot_unavailable → P3.3.1 → P3.8
- ✅ count_bookings (D3) → P3.3.1

Всі потоки збалансовані між рівнями.

---

**Кінець словника DFD**
