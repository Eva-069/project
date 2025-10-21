# Invariants Matrix - CoworkSpace Booking System

**Дата:** 2025-01-15  
**Версія:** 1.0

---

## 1. Правила між полями (Field-level Invariants)

### Таблиця: timeslots

| Інваріант                       | Рівень   | Опис                                              | Код помилки          |
| ------------------------------- | -------- | ------------------------------------------------- | -------------------- |
| start_at < end_at               | DB CHECK | Початок слоту має бути раніше за кінець           | `invalid_time_range` |
| max_concurrent_bookings > 0     | DB CHECK | Максимальна кількість бронювань має бути більше 0 | `invalid_capacity`   |
| start_at >= NOW() при створенні | APP      | Слоти створюються тільки для майбутнього часу     | `timeslot_in_past`   |

**SQL Constraint:**

```sql
ALTER TABLE timeslots
ADD CONSTRAINT check_start_before_end
CHECK (start_at < end_at);

ALTER TABLE timeslots
ADD CONSTRAINT check_capacity_positive
CHECK (max_concurrent_bookings > 0);
```

---

### Таблиця: bookings

| Інваріант                                                  | Рівень   | Опис                                            | Код помилки                 |
| ---------------------------------------------------------- | -------- | ----------------------------------------------- | --------------------------- |
| booking_status IN ('requested', 'confirmed', 'cancelled')  | DB CHECK | Дозволені тільки ці статуси                     | `invalid_enum_value`        |
| cancelled_at IS NULL OR booking_status = 'cancelled'       | DB CHECK | cancelled_at заповнюється тільки при скасуванні | `invariant_violation`       |
| cancel_reason_code IS NULL OR booking_status = 'cancelled' | DB CHECK | Причина скасування тільки для скасованих        | `invariant_violation`       |
| customer_email відповідає RFC 5322                         | APP      | Валідація email формату                         | `invalid_email`             |
| customer_name довжина 2-100                                | APP      | Мінімум 2 символи, максимум 100                 | `invalid_field_length`      |
| confirmation_code [A-Z0-9]{8}                              | APP      | 8 символів uppercase + digits                   | `invalid_confirmation_code` |

**SQL Constraint:**

```sql
ALTER TABLE bookings
ADD CONSTRAINT check_booking_status
CHECK (booking_status IN ('requested', 'confirmed', 'cancelled'));

ALTER TABLE bookings
ADD CONSTRAINT check_cancelled_at_when_cancelled
CHECK (cancelled_at IS NULL OR booking_status = 'cancelled');

ALTER TABLE bookings
ADD CONSTRAINT check_cancel_reason_when_cancelled
CHECK (cancel_reason_code IS NULL OR booking_status = 'cancelled');
```

---

## 2. Посилальна цілісність (Foreign Keys)

| From Table          | Column          | To Table      | Column          | ON DELETE | ON UPDATE | Опис                            |
| ------------------- | --------------- | ------------- | --------------- | --------- | --------- | ------------------------------- |
| timeslots           | meeting_room_id | meeting_rooms | meeting_room_id | RESTRICT  | CASCADE   | Слот посилається на кімнату     |
| bookings            | timeslot_id     | timeslots     | timeslot_id     | RESTRICT  | CASCADE   | Бронювання посилається на слот  |
| notification_events | booking_id      | bookings      | booking_id      | RESTRICT  | CASCADE   | Подія посилається на бронювання |

**Політика видалення: RESTRICT**

- Не можна видалити meeting_room якщо є timeslots
- Не можна видалити timeslot якщо є bookings
- Не можна видалити booking якщо є notification_events

**Обгрунтування:**

- Збереження історії даних
- Уникнення cascade-видалень
- Soft-delete НЕ використовується (немає deleted_at)

**Код помилки при порушенні FK:**

- `reference_not_found` - коли посилання на неіснуючий запис
- `fk_violation` - коли намагаємося видалити запис з залежностями

---

## 3. Business-унікальності

### Унікальність 1: meeting_rooms.room_name

| Constraint         | Рівень    | Опис                              | Код помилки           |
| ------------------ | --------- | --------------------------------- | --------------------- |
| UNIQUE (room_name) | DB UNIQUE | Назва кімнати унікальна в системі | `duplicate_room_name` |

```sql
CREATE UNIQUE INDEX idx_meeting_rooms_room_name
ON meeting_rooms(room_name);
```

---

### Унікальність 2: timeslots(meeting_room_id, start_at, end_at)

| Constraint                                 | Рівень    | Опис                                               | Код помилки          |
| ------------------------------------------ | --------- | -------------------------------------------------- | -------------------- |
| UNIQUE (meeting_room_id, start_at, end_at) | DB UNIQUE | Той самий слот часу не може бути двічі для кімнати | `duplicate_timeslot` |

```sql
CREATE UNIQUE INDEX idx_timeslots_room_time
ON timeslots(meeting_room_id, start_at, end_at);
```

**Додаткове правило (app-level):**

- Слоти однієї кімнати не повинні перекриватися
- Перевіряється при створенні через діапазонний запит

---

### Унікальність 3: bookings.confirmation_code

| Constraint                 | Рівень    | Опис                                   | Код помилки                   |
| -------------------------- | --------- | -------------------------------------- | ----------------------------- |
| UNIQUE (confirmation_code) | DB UNIQUE | Код підтвердження унікальний глобально | `duplicate_confirmation_code` |

```sql
CREATE UNIQUE INDEX idx_bookings_confirmation_code
ON bookings(confirmation_code);
```

---

## 4. Capacity Limit (Twist A: P=1 непарне → capacity=3)

### Інваріант: Максимум 3 активних бронювання на timeslot

| Параметр    | Значення                                                                                  |
| ----------- | ----------------------------------------------------------------------------------------- |
| Рівень      | APP-level (не DB constraint)                                                              |
| Правило     | COUNT(booking WHERE timeslot_id = X AND booking_status IN ('requested', 'confirmed')) ≤ 3 |
| Перевірка   | P3.3.1 validate_booking_data (DFD Level 2)                                                |
| Код помилки | `slot_unavailable`                                                                        |
| HTTP Status | 409 Conflict                                                                              |

**SQL для перевірки:**

```sql
SELECT COUNT(*) as current_bookings
FROM bookings
WHERE timeslot_id = :timeslot_id
  AND booking_status IN ('requested', 'confirmed');

-- Якщо current_bookings >= 3 → відмова
```

**Чому не DB constraint:**

- Складно реалізувати в SQL (потрібен тригер)
- Capacity може змінюватися (хоча зараз константа 3)
- Краща продуктивність при перевірці на app-level з індексом
- Race conditions вирішуються через транзакції

**Індекс для швидкої перевірки:**

```sql
CREATE INDEX idx_bookings_timeslot_status
ON bookings(timeslot_id, booking_status);
```

**Альтернатива для capacity=1:**
Якщо б P була парна (capacity=1), використовувався б:

```sql
CREATE UNIQUE INDEX idx_bookings_timeslot_unique
ON bookings(timeslot_id)
WHERE booking_status IN ('requested', 'confirmed');
```

---

## 5. Status Transitions (State Machine)

### booking_status переходи

```
       ┌─────────────┐
       │  requested  │ (початковий стан)
       └─────────────┘
         │         │
         │         │
    ┌────▼──┐   ┌──▼──────┐
    │confirmed│  │cancelled│
    └────┬──┘   └─────────┘
         │           ▲
         │           │
         └───────────┘
```

| From | To | Trigger | Виконавець | Валідація |
|------|----|---------|-----------||-----------|
| requested | confirmed | Підтвердження менеджером | space_manager | - |
| requested | cancelled | Скасування | meeting_organizer OR space_manager | - |
| confirmed | cancelled | Скасування | meeting_organizer OR space_manager | - |
| cancelled | \* | - | - | Заборонено (кінцевий стан) |

**Код помилки при неправильному переході:**

- `invalid_status_transition`

**Правила при зміні status:**

```
IF new_status = 'cancelled' THEN
  SET cancelled_at = NOW()
  SET cancel_reason_code = :reason (опціонально)
END IF

IF old_status = 'cancelled' AND new_status != 'cancelled' THEN
  RAISE ERROR 'invalid_status_transition'
END IF
```

---

## 6. Матриця помилок та їх походження

| Код помилки                   | HTTP Status | Джерело         | Інваріант                | Опис                                         |
| ----------------------------- | ----------- | --------------- | ------------------------ | -------------------------------------------- |
| `invalid_email`               | 400         | P3.1 validation | customer_email format    | Email не відповідає RFC 5322                 |
| `missing_field`               | 400         | P3.1 validation | required fields          | Обов'язкове поле не заповнено                |
| `invalid_field_length`        | 400         | P3.1 validation | length constraints       | Довжина поля поза межами                     |
| `timeslot_not_found`          | 404         | P3.2 check      | FK integrity             | timeslot_id не існує                         |
| `slot_unavailable`            | 409         | P3.3 capacity   | capacity <= 3            | Слот повністю зайнятий (3/3)                 |
| `duplicate_confirmation_code` | 409         | DB UNIQUE       | confirmation_code unique | Код вже існує (малоймовірно)                 |
| `invalid_time_range`          | 400         | DB CHECK        | start_at < end_at        | Неправильний часовий діапазон                |
| `invalid_capacity`            | 400         | DB CHECK        | max_concurrent > 0       | Capacity має бути > 0                        |
| `invariant_violation`         | 400         | DB CHECK        | cancelled_at logic       | cancelled_at без status=cancelled            |
| `invalid_enum_value`          | 400         | DB CHECK        | enum domains             | Значення поза дозволеним набором             |
| `reference_not_found`         | 404         | FK constraint   | FK integrity             | Посилання на неіснуючий запис                |
| `fk_violation`                | 409         | FK RESTRICT     | ON DELETE                | Спроба видалити запис з залежностями         |
| `invalid_status_transition`   | 409         | APP logic       | status state machine     | Неправильний перехід статусу                 |
| `duplicate_room_name`         | 409         | DB UNIQUE       | room_name unique         | Кімната з такою назвою вже є                 |
| `duplicate_timeslot`          | 409         | DB UNIQUE       | timeslot uniqueness      | Слот часу вже існує                          |
| `timeslot_in_past`            | 400         | APP validation  | business rule            | Не можна створити слот у минулому            |
| `booking_not_found`           | 404         | APP query       | -                        | booking_id або confirmation_code не знайдено |
| `out_of_range`                | 400         | APP validation  | range checks             | Значення поза допустимим діапазоном          |

---

## 7. Транзакційні інваріанти

### Створення бронювання (P3 create_booking)

**Атомарність:**

```sql
BEGIN TRANSACTION;

-- 1. Перевірка існування timeslot
SELECT timeslot_id FROM timeslots WHERE timeslot_id = :id FOR UPDATE;

-- 2. Перевірка capacity
SELECT COUNT(*) FROM bookings
WHERE timeslot_id = :id AND booking_status IN ('requested', 'confirmed')
FOR UPDATE;

-- 3. Створення booking (якщо count < 3)
INSERT INTO bookings (...) VALUES (...);

-- 4. Створення notification_event
INSERT INTO notification_events (...) VALUES (...);

COMMIT;
```

**FOR UPDATE:**

- Блокує рядки для запобігання race conditions
- Гарантує послідовну перевірку capacity

**Rollback при помилці:**

- Будь-яка помилка → ROLLBACK
- Гарантує консистентність даних

---

### Скасування бронювання (P4 modify_or_cancel)

**Атомарність:**

```sql
BEGIN TRANSACTION;

-- 1. Знайти booking
SELECT * FROM bookings WHERE booking_id = :id FOR UPDATE;

-- 2. Перевірити що status != 'cancelled'
IF status = 'cancelled' THEN RAISE ERROR 'already_cancelled';

-- 3. Оновити booking
UPDATE bookings
SET booking_status = 'cancelled',
    cancelled_at = NOW(),
    cancel_reason_code = :reason
WHERE booking_id = :id;

-- 4. Створити notification_event
INSERT INTO notification_events (...) VALUES (...);

COMMIT;
```

---

## 8. Concurrent Access Patterns

### Race Condition: Одночасне бронювання останнього місця

**Проблема:**

```
T1: SELECT COUNT(*) → 2 (< 3, OK)
T2: SELECT COUNT(*) → 2 (< 3, OK)
T1: INSERT booking
T2: INSERT booking
Result: 4 bookings (порушення capacity!)
```

**Рішення: FOR UPDATE**

```sql
BEGIN TRANSACTION;

SELECT COUNT(*) FROM bookings
WHERE timeslot_id = :id AND booking_status IN ('requested', 'confirmed')
FOR UPDATE OF bookings;

-- Тільки один transaction може виконати SELECT FOR UPDATE
-- Інші чекають

IF count < 3 THEN
  INSERT INTO bookings (...);
END IF;

COMMIT;
```

**Альтернатива: Optimistic Locking**

- Додати version field до timeslots
- Перевіряти version при INSERT booking
- Повторна спроба при conflict

---

## 9. Cascading Effects

### При зміні booking_status на 'cancelled':

1. **Автоматичні зміни:**

   - `cancelled_at` = NOW()
   - `cancel_reason_code` = заповнюється

2. **Тригерні ефекти:**

   - Створення notification_event
   - Інвалідація cache (D4 cache_or_view)
   - Звільнення capacity (COUNT зменшується)

3. **Що НЕ змінюється:**
   - timeslot залишається незмінним
   - created_at не змінюється
   - booking_id не змінюється

---

## 10. Data Integrity Checks (Summary)

### DB-level (SQL Constraints)

| Check              | Tables                                   | Constraint Type      |
| ------------------ | ---------------------------------------- | -------------------- |
| PK uniqueness      | Всі таблиці                              | PRIMARY KEY          |
| FK integrity       | timeslots, bookings, notification_events | FOREIGN KEY RESTRICT |
| Enum values        | bookings.booking_status                  | CHECK                |
| Time range         | timeslots.start_at < end_at              | CHECK                |
| Conditional fields | bookings.cancelled_at logic              | CHECK                |
| Unique codes       | bookings.confirmation_code               | UNIQUE INDEX         |
| Unique names       | meeting_rooms.room_name                  | UNIQUE INDEX         |
| Unique slots       | timeslots composite                      | UNIQUE INDEX         |

**SQL для перевірки integrity:**

```sql
-- Перевірка FK
SELECT b.booking_id
FROM bookings b
LEFT JOIN timeslots t ON b.timeslot_id = t.timeslot_id
WHERE t.timeslot_id IS NULL;

-- Перевірка cancelled_at logic
SELECT booking_id
FROM bookings
WHERE cancelled_at IS NOT NULL AND booking_status != 'cancelled';

-- Перевірка capacity (інформаційно)
SELECT timeslot_id, COUNT(*) as bookings_count
FROM bookings
WHERE booking_status IN ('requested', 'confirmed')
GROUP BY timeslot_id
HAVING COUNT(*) > 3;
```

---

### APP-level (Business Logic)

| Check                        | Component | When                         |
| ---------------------------- | --------- | ---------------------------- |
| Email validation             | P3.1      | Before INSERT booking        |
| Capacity limit               | P3.3.1    | Before INSERT booking        |
| Status transitions           | P4        | Before UPDATE booking.status |
| Timeslot overlap             | P2        | Before INSERT timeslot       |
| Past timeslots               | P2        | Before INSERT timeslot       |
| Confirmation code generation | P3.4      | During INSERT booking        |

---

## 11. Rollback Scenarios

### Коли потрібен ROLLBACK:

| Scenario                    | Transaction    | Action                         |
| --------------------------- | -------------- | ------------------------------ |
| Capacity exceeded           | CREATE booking | ROLLBACK + return 409          |
| Invalid FK                  | CREATE booking | ROLLBACK + return 404          |
| Duplicate confirmation_code | CREATE booking | ROLLBACK + retry with new code |
| Invalid status transition   | UPDATE booking | ROLLBACK + return 409          |
| CHECK constraint violation  | Any write      | ROLLBACK + return 400          |

---

## 12. Monitoring & Alerts

### Інваріанти для моніторингу:

```sql
-- Alert: Capacity порушено (критично!)
SELECT timeslot_id, COUNT(*) as count
FROM bookings
WHERE booking_status IN ('requested', 'confirmed')
GROUP BY timeslot_id
HAVING COUNT(*) > 3;

-- Alert: Orphaned notification_events
SELECT ne.event_id
FROM notification_events ne
LEFT JOIN bookings b ON ne.booking_id = b.booking_id
WHERE b.booking_id IS NULL;

-- Alert: Inconsistent cancelled_at
SELECT booking_id
FROM bookings
WHERE (cancelled_at IS NULL AND booking_status = 'cancelled')
   OR (cancelled_at IS NOT NULL AND booking_status != 'cancelled');

-- Alert: Timeslots in the past still active
SELECT timeslot_id
FROM timeslots
WHERE end_at < NOW() AND is_active = true;
```

**Рекомендація:** Запускати ці перевірки щоденно як частину health-check.

---

## 13. Migration Strategy для інваріантів

### Додавання нового інваріанту:

1. **Перевірити існуючі дані:**

```sql
-- Знайти порушення ПЕРЕД додаванням constraint
SELECT * FROM bookings
WHERE cancelled_at IS NOT NULL AND booking_status != 'cancelled';
```

2. **Виправити дані:**

```sql
-- Виправити некоректні записи
UPDATE bookings
SET cancelled_at = NULL
WHERE booking_status != 'cancelled';
```

3. **Додати constraint:**

```sql
ALTER TABLE bookings
ADD CONSTRAINT check_cancelled_at_when_cancelled
CHECK (cancelled_at IS NULL OR booking_status = 'cancelled');
```

4. **Оновити application code** для підтримки нового правила

---

## 14. Testing Invariants

### Unit Tests для інваріантів:

```javascript
// Test: Capacity limit
test("should reject booking when capacity exceeded", async () => {
  // Given: timeslot з 3 бронюваннями
  const timeslot = await createTimeslot({ capacity: 3 });
  await createBooking({ timeslot, status: "requested" }); // 1
  await createBooking({ timeslot, status: "confirmed" }); // 2
  await createBooking({ timeslot, status: "requested" }); // 3

  // When: спроба створити 4-те бронювання
  const result = await createBooking({ timeslot, status: "requested" });

  // Then: помилка slot_unavailable
  expect(result.error_code).toBe("slot_unavailable");
  expect(result.status).toBe(409);
});

// Test: Status transition
test("should not allow transition from cancelled to confirmed", async () => {
  // Given: скасоване бронювання
  const booking = await createBooking({ status: "cancelled" });

  // When: спроба змінити на confirmed
  const result = await updateBooking(booking.id, { status: "confirmed" });

  // Then: помилка invalid_status_transition
  expect(result.error_code).toBe("invalid_status_transition");
});

// Test: cancelled_at invariant
test("should set cancelled_at when status changed to cancelled", async () => {
  // Given: активне бронювання
  const booking = await createBooking({ status: "confirmed" });

  // When: скасування
  const result = await updateBooking(booking.id, {
    status: "cancelled",
    cancel_reason_code: "user_request",
  });

  // Then: cancelled_at заповнено
  expect(result.cancelled_at).not.toBeNull();
  expect(result.cancel_reason_code).toBe("user_request");
});
```

---

**Кінець Invariants Matrix**
