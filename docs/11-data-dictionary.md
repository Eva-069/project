# Data Dictionary - CoworkSpace Booking System

**Дата:** 2025-01-15  
**Версія:** 1.0  
**Capacity:** 3 одночасних бронювання на timeslot  
**NFR акцент:** Performance-first

---

## Таблиця: meeting_rooms

Переговорні кімнати у коворкінг-просторі.

| Column           | Type         | Required | Default           | Domain/Enum | Keys   | PII | Description                               | Invariants/Notes                           |
| ---------------- | ------------ | -------- | ----------------- | ----------- | ------ | --- | ----------------------------------------- | ------------------------------------------ |
| meeting_room_id  | UUID         | YES      | gen_random_uuid() | -           | PK     | NO  | Унікальний ідентифікатор кімнати          | UUID v4                                    |
| room_name        | VARCHAR(100) | YES      | -                 | -           | UNIQUE | NO  | Назва переговорної кімнати                | Унікальна в межах системи; 2-100 символів  |
| capacity_persons | INT          | YES      | -                 | 1..50       | -      | NO  | Місткість кімнати (кількість осіб)        | Має бути більше 0; рекомендовано 2-20      |
| equipment        | JSONB        | NO       | NULL              | -           | -      | NO  | Обладнання кімнати (проектор, whiteboard) | Приклад: ["projector", "whiteboard", "tv"] |
| is_active        | BOOLEAN      | YES      | true              | -           | INDEX  | NO  | Чи активна кімната для бронювань          | true = доступна, false = архівована        |
| created_at       | TIMESTAMP    | YES      | NOW()             | -           | -      | NO  | Дата створення запису (UTC)               | ISO 8601, не може бути в майбутньому       |
| updated_at       | TIMESTAMP    | YES      | NOW()             | -           | -      | NO  | Дата останнього оновлення (UTC)           | ISO 8601, автоматично при UPDATE           |

**Індекси:**

- PRIMARY KEY: meeting_room_id
- UNIQUE INDEX: room_name
- INDEX: is_active (для фільтрації активних)

**Бізнес-правила:**

- room_name має бути унікальним
- capacity_persons > 0
- При створенні is_active = true по замовчуванню

---

## Таблиця: timeslots

Часові слоти для бронювання переговорних кімнат.

| Column                  | Type      | Required | Default           | Domain/Enum | Keys               | PII | Description                                | Invariants/Notes                               |
| ----------------------- | --------- | -------- | ----------------- | ----------- | ------------------ | --- | ------------------------------------------ | ---------------------------------------------- |
| timeslot_id             | UUID      | YES      | gen_random_uuid() | -           | PK                 | NO  | Унікальний ідентифікатор слоту             | UUID v4                                        |
| meeting_room_id         | UUID      | YES      | -                 | -           | FK → meeting_rooms | NO  | Посилання на переговорну кімнату           | NOT NULL, RESTRICT on delete                   |
| start_at                | TIMESTAMP | YES      | -                 | -           | INDEX              | NO  | Початок слоту (UTC)                        | ISO 8601, має бути в майбутньому при створенні |
| end_at                  | TIMESTAMP | YES      | -                 | -           | INDEX              | NO  | Кінець слоту (UTC)                         | ISO 8601, має бути більше start_at             |
| max_concurrent_bookings | INT       | YES      | 3                 | -           | -                  | NO  | Максимальна кількість одночасних бронювань | Завжди = 3 (capacity з варіанту)               |
| is_active               | BOOLEAN   | YES      | true              | -           | INDEX              | NO  | Чи активний слот для бронювань             | true = доступний, false = архівований          |
| created_at              | TIMESTAMP | YES      | NOW()             | -           | -                  | NO  | Дата створення запису (UTC)                | ISO 8601                                       |
| updated_at              | TIMESTAMP | YES      | NOW()             | -           | -                  | NO  | Дата останнього оновлення (UTC)            | ISO 8601, автоматично при UPDATE               |

**Індекси:**

- PRIMARY KEY: timeslot_id
- UNIQUE INDEX: (meeting_room_id, start_at, end_at) - унікальність слоту для кімнати
- INDEX: (meeting_room_id, start_at) - для пошуку доступних слотів
- INDEX: (start_at, end_at) - для діапазонних запитів

**Constraint:**

```sql
CHECK (end_at > start_at)
CHECK (max_concurrent_bookings > 0)
```

**Бізнес-правила:**

- start_at має бути менше end_at
- (meeting_room_id, start_at, end_at) унікальна комбінація
- max_concurrent_bookings = 3 (константа з варіанту P=1 непарне)
- Слоти не перекриваються для однієї кімнати (перевіряється на рівні додатку)

---

## Таблиця: bookings

Бронювання переговорних кімнат користувачами.

| Column             | Type         | Required | Default           | Domain/Enum  | Keys                  | PII | Description                         | Invariants/Notes                                          |
| ------------------ | ------------ | -------- | ----------------- | ------------ | --------------------- | --- | ----------------------------------- | --------------------------------------------------------- |
| booking_id         | UUID         | YES      | gen_random_uuid() | -            | PK                    | NO  | Унікальний ідентифікатор бронювання | UUID v4                                                   |
| timeslot_id        | UUID         | YES      | -                 | -            | FK → timeslots, INDEX | NO  | Посилання на часовий слот           | NOT NULL, RESTRICT on delete                              |
| customer_name      | VARCHAR(100) | YES      | -                 | -            | -                     | NO  | Ім'я користувача (організатора)     | 2-100 символів                                            |
| customer_email     | VARCHAR(255) | YES      | -                 | email format | INDEX                 | YES | Email користувача                   | RFC 5322 валідація; маскується в логах: u**_@e_**.com     |
| confirmation_code  | VARCHAR(8)   | YES      | random(8)         | [A-Z0-9]{8}  | UNIQUE                | NO  | Код підтвердження для перегляду     | Генерується автоматично; uppercase + digits               |
| booking_status     | VARCHAR(20)  | YES      | 'requested'       | enum         | INDEX                 | NO  | Статус бронювання                   | Дозволені значення: 'requested', 'confirmed', 'cancelled' |
| cancelled_at       | TIMESTAMP    | NO       | NULL              | -            | -                     | NO  | Дата скасування (UTC)               | Заповнюється тільки якщо status = 'cancelled'             |
| cancel_reason_code | VARCHAR(50)  | NO       | NULL              | enum         | -                     | NO  | Код причини скасування              | Null якщо не скасовано; див. код-сет cancel_reasons       |
| created_at         | TIMESTAMP    | YES      | NOW()             | -            | INDEX                 | NO  | Дата створення бронювання (UTC)     | ISO 8601                                                  |
| updated_at         | TIMESTAMP    | YES      | NOW()             | -            | -                     | NO  | Дата останнього оновлення (UTC)     | ISO 8601, автоматично при UPDATE                          |

**Індекси:**

- PRIMARY KEY: booking_id
- UNIQUE INDEX: confirmation_code
- INDEX: (timeslot_id, booking_status) - для capacity check
- INDEX: (customer_email, created_at DESC) - для пошуку бронювань користувача
- INDEX: created_at DESC - для сортування

**Constraint:**

```sql
CHECK (booking_status IN ('requested', 'confirmed', 'cancelled'))
CHECK (cancelled_at IS NULL OR booking_status = 'cancelled')
CHECK (cancel_reason_code IS NULL OR booking_status = 'cancelled')
```

**Бізнес-правила (app-level):**

- **Capacity limit:** COUNT(booking WHERE timeslot_id = X AND booking_status IN ('requested', 'confirmed')) ≤ 3
- Переходи status: requested → {confirmed, cancelled}; confirmed → cancelled
- cancelled_at заповнюється автоматично при зміні status на 'cancelled'
- confirmation_code генерується при створенні: 8 символів uppercase letters + digits
- customer*email валідується regex: `^[a-zA-Z0-9.*%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`

**PII маскування:**

- customer_email: `user@example.com` → `u***@e***.com` в логах та помилках

---

## Таблиця: users

Користувачі системи (адміністратори, менеджери коворкінгу).

| Column     | Type         | Required | Default           | Domain/Enum  | Keys   | PII | Description                          | Invariants/Notes                                |
| ---------- | ------------ | -------- | ----------------- | ------------ | ------ | --- | ------------------------------------ | ----------------------------------------------- |
| user_id    | UUID         | YES      | gen_random_uuid() | -            | PK     | NO  | Унікальний ідентифікатор користувача | UUID v4                                         |
| email      | VARCHAR(255) | YES      | -                 | email format | UNIQUE | YES | Email користувача                    | RFC 5322; унікальний; маскується: u**_@e_**.com |
| role       | VARCHAR(50)  | YES      | -                 | enum         | INDEX  | NO  | Роль користувача в системі           | Дозволені: 'space_manager', 'admin'             |
| name       | VARCHAR(100) | NO       | NULL              | -            | -      | YES | Повне ім'я користувача               | Опціонально; PII                                |
| created_at | TIMESTAMP    | YES      | NOW()             | -            | -      | NO  | Дата створення запису (UTC)          | ISO 8601                                        |
| updated_at | TIMESTAMP    | YES      | NOW()             | -            | -      | NO  | Дата останнього оновлення (UTC)      | ISO 8601                                        |

**Індекси:**

- PRIMARY KEY: user_id
- UNIQUE INDEX: email
- INDEX: role

**Constraint:**

```sql
CHECK (role IN ('space_manager', 'admin'))
```

**Бізнес-правила:**

- email має бути унікальним
- role обов'язкова, з дозволеного набору
- name опціонально (для privacy-first)

**Примітка:**

- meeting_organizer (E1 з DFD) НЕ зберігається в таблиці users
- Для бронювань використовується тільки customer_email (без створення акаунту)

---

## Таблиця: notification_events

Черга подій для відправки повідомлень (outbox pattern).

| Column       | Type         | Required | Default           | Domain/Enum | Keys                 | PII | Description                    | Invariants/Notes                          |
| ------------ | ------------ | -------- | ----------------- | ----------- | -------------------- | --- | ------------------------------ | ----------------------------------------- |
| event_id     | UUID         | YES      | gen_random_uuid() | -           | PK                   | NO  | Унікальний ідентифікатор події | UUID v4                                   |
| booking_id   | UUID         | YES      | -                 | -           | FK → bookings, INDEX | NO  | Посилання на бронювання        | NOT NULL, RESTRICT on delete              |
| event_type   | VARCHAR(50)  | YES      | -                 | enum        | -                    | NO  | Тип події                      | 'booking_created', 'booking_cancelled'    |
| masked_email | VARCHAR(255) | NO       | NULL              | -           | -                    | NO  | Замаскований email отримувача  | u**_@e_**.com (privacy-first)             |
| payload      | JSONB        | NO       | NULL              | -           | -                    | NO  | Додаткові дані події           | JSON з booking_id, confirmation_code      |
| processed    | BOOLEAN      | YES      | false             | -           | INDEX                | NO  | Чи оброблено подію             | false при створенні, true після відправки |
| created_at   | TIMESTAMP    | YES      | NOW()             | -           | INDEX                | NO  | Дата створення події (UTC)     | ISO 8601                                  |
| processed_at | TIMESTAMP    | NO       | NULL              | -           | -                    | NO  | Дата обробки події (UTC)       | NULL якщо не оброблено                    |

**Індекси:**

- PRIMARY KEY: event_id
- INDEX: (processed, created_at) - для вибірки необроблених подій
- INDEX: booking_id - для пошуку подій бронювання

**Constraint:**

```sql
CHECK (event_type IN ('booking_created', 'booking_cancelled'))
CHECK (processed_at IS NULL OR processed = true)
```

**Бізнес-правила:**

- processed = false при створенні
- processed_at заповнюється при зміні processed на true
- masked_email містить замасковану версію customer_email (privacy-first)
- payload не містить повних PII даних

---

## Код-сети (Enumerations)

### booking_status

Статуси бронювання.

| Code      | Description                               | Allowed Transitions    |
| --------- | ----------------------------------------- | ---------------------- |
| requested | Бронювання створено, очікує підтвердження | → confirmed, cancelled |
| confirmed | Бронювання підтверджено менеджером        | → cancelled            |
| cancelled | Бронювання скасовано                      | (кінцевий стан)        |

**Правила переходів:**

```
requested → confirmed (space_manager підтверджує)
requested → cancelled (user або manager скасовує)
confirmed → cancelled (user або manager скасовує)
```

**Машинні коди:**

- Використовуються в API, БД, логах
- UI-тексти зберігаються окремо (локалізація)

---

### event_type

Типи подій для notification_events.

| Code              | Description              |
| ----------------- | ------------------------ |
| booking_created   | Нове бронювання створено |
| booking_cancelled | Бронювання скасовано     |

---

### role

Ролі користувачів системи.

| Code          | Description                    |
| ------------- | ------------------------------ |
| space_manager | Менеджер коворкінгу (E2 з DFD) |
| admin         | Адміністратор системи          |

**Примітка:** meeting_organizer (E1) не є роллю user - це анонімні користувачі з customer_email.

---

### cancel_reason_code

Причини скасування бронювання.

| Code              | Description               |
| ----------------- | ------------------------- |
| user_request      | Користувач сам скасував   |
| room_unavailable  | Кімната стала недоступною |
| schedule_conflict | Конфлікт розкладу         |
| no_show           | Користувач не з'явився    |
| admin_action      | Скасовано адміністратором |

**Використання:**

- Заповнюється при booking_status = 'cancelled'
- Опціонально (NULL дозволено)
- Для аналітики та звітності

---

## Домени даних

### UUID

- Формат: UUID v4 (128-bit, gen_random_uuid())
- Приклад: `550e8400-e29b-41d4-a716-446655440000`
- Використання: всі PK

### TIMESTAMP

- Формат: ISO 8601 в UTC
- Приклад: `2025-01-15T10:30:00Z`
- Зберігання: TIMESTAMP WITHOUT TIME ZONE (завжди UTC)
- Відображення: з timezone клієнта на frontend

### Email

- Валідація: RFC 5322 simplified regex
- Regex: `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
- Max length: 255 символів
- PII: так, маскується в логах

### confirmation_code

- Формат: 8 символів [A-Z0-9]
- Генерація: random uppercase letters + digits
- Приклад: `AB12CD34`
- Унікальність: UNIQUE constraint

---

## PII Політики

### PII поля:

1. **bookings.customer_email** - email користувача
2. **bookings.customer_name** - ім'я користувача
3. **users.email** - email системного користувача
4. **users.name** - ім'я системного користувача

### Маскування:

- **Email:** `user@example.com` → `u***@e***.com`
- **Name:** опціонально, можна не зберігати

### Правила:

- PII не виводиться в логи в незамаскованому вигляді
- PII не передається в error responses
- notification_events.masked_email вже містить замасковану версію
- PII не копіюється в кеш-таблиці

---

## Аудит та часові мітки

### Обов'язкові поля для всіх таблиць:

- `created_at TIMESTAMP NOT NULL DEFAULT NOW()`
- `updated_at TIMESTAMP NOT NULL DEFAULT NOW()`

### Автоматичне оновлення:

```sql
CREATE TRIGGER update_updated_at_column
BEFORE UPDATE ON table_name
FOR EACH ROW
EXECUTE FUNCTION update_updated_at_column();
```

### Політика видалення:

- **Soft-delete:** НЕ використовується
- **Hard-delete:** з RESTRICT на FK
- Причина: простота моделі, не потрібна історія видалень

---

## Performance-first індекси

### Патерни доступу → Індекси:

| Use Case                       | Query Pattern                                          | Index                             |
| ------------------------------ | ------------------------------------------------------ | --------------------------------- |
| US-001: Пошук доступних слотів | WHERE meeting_room_id = ? AND start_at BETWEEN ? AND ? | (meeting_room_id, start_at)       |
| US-002: Capacity check         | WHERE timeslot_id = ? AND booking_status IN (?)        | (timeslot_id, booking_status)     |
| US-003: Бронювання користувача | WHERE customer_email = ? ORDER BY created_at DESC      | (customer_email, created_at DESC) |
| Перевірка confirmation_code    | WHERE confirmation_code = ?                            | UNIQUE (confirmation_code)        |
| Необроблені події              | WHERE processed = false ORDER BY created_at            | (processed, created_at)           |

### Composite індекси:

- Порядок важливий: найбільш селективне поле першим
- Покриваючі індекси для COUNT запитів

---

## Інваріанти (summary)

1. **timeslots.start_at < timeslots.end_at** (DB CHECK)
2. **bookings.booking_status IN ('requested', 'confirmed', 'cancelled')** (DB CHECK)
3. **bookings.cancelled_at NULL якщо status != 'cancelled'** (DB CHECK)
4. **Capacity limit: COUNT(bookings) ≤ 3 per timeslot** (app-level, P3.3.1)
5. **Status transitions:** requested → {confirmed, cancelled}; confirmed → cancelled (app-level)
6. **confirmation_code унікальний** (DB UNIQUE)
7. **customer_email валідний email** (app-level validation)
8. **PII маскування в логах** (app-level)

---

**Кінець Data Dictionary**
