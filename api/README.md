# CoworkSpace Booking API - Практичне 4

## 📋 Зміст

1. [Швидкий старт](#швидкий-старт)
2. [Запуск Mock Server](#запуск-mock-server)
3. [Робота з Postman](#робота-з-postman)
4. [Тестові сценарії](#тестові-сценарії)
5. [Структура файлів](#структура-файлів)

---

## 🚀 Швидкий старт

### Необхідні інструменти

1. **Node.js** (v14+) - [Скачати](https://nodejs.org/)
2. **Postman** - [Скачати](https://www.postman.com/downloads/)
3. **Prism** (для mock server)

```bash
# Встановіть Prism глобально (один раз)
npm install -g @stoplight/prism-cli
```

---

## 🎯 Запуск Mock Server

### Метод 1: Через Prism (Рекомендовано)

```bash
# 1. Перейдіть у папку з openapi.yaml
cd api/

# 2. Запустіть mock server
prism mock openapi.yaml

# Mock server буде доступний на http://127.0.0.1:4010
```

### Метод 2: Онлайн (Swagger Editor)

1. Відкрийте https://editor.swagger.io/
2. Скопіюйте вміст `openapi.yaml`
3. Вставте у лівій панелі
4. Натисніть "Try it out" для тестування

---

## 📮 Робота з Postman

### Імпорт колекції

1. Відкрийте Postman
2. Натисніть **Import**
3. Перетягніть файл `CoworkSpace.postman_collection.json`
4. Колекція з'явиться у лівій панелі

### Налаштування Environment

**Варіант A: Використати змінні з колекції (вже налаштовано)**

Змінні вже є в колекції:

- `baseUrl`: http://localhost:4010/v1
- `meetingRoomId`: 550e8400-e29b-41d4-a716-446655440000
- `timeslotId`: 770e8400-e29b-41d4-a716-446655440002

**Варіант B: Створити власний Environment (опціонально)**

1. Натисніть **Environments** → **Create Environment**
2. Додайте змінні:

| Variable      | Value                                     |
| ------------- | ----------------------------------------- |
| baseUrl       | http://localhost:4010/v1                  |
| meetingRoomId | 550e8400-e29b-41d4-a716-446655440000      |
| timeslotId    | 770e8400-e29b-41d4-a716-446655440002      |
| managerToken  | eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.mock |

---

## 🧪 Тестові сценарії

### Сценарій 1: Перегляд доступних кімнат (US-001)

```bash
# 1. Запустіть Prism mock
prism mock openapi.yaml

# 2. У новому терміналі виконайте:
curl http://127.0.0.1:4010/meeting_rooms
```

**Очікуваний результат:**

```json
{
  "data": [
    {
      "meeting_room_id": "550e8400-e29b-41d4-a716-446655440000",
      "room_name": "Conference Room A",
      "location": "Floor 2, Room 201",
      "capacity": 8
    }
  ],
  "meta": {
    "total_count": 12,
    "current_page": 1,
    "total_pages": 1
  }
}
```

**У Postman:**

1. Відкрийте `Meeting Rooms → List Meeting Rooms`
2. Натисніть **Send**
3. Перевірте статус: **200 OK**

---

### Сценарій 2: Перегляд доступних слотів (US-001)

```bash
curl "http://127.0.0.1:4010/timeslots?meeting_room_id=550e8400-e29b-41d4-a716-446655440000&date_from=2025-01-15&date_to=2025-01-20"
```

**У Postman:**

1. Відкрийте `Timeslots → US-001: Get Available Timeslots`
2. Натисніть **Send**
3. Перевірте `availability_status`: "1/3 occupied"

---

### Сценарій 3: Створення бронювання (US-002) ✅ Success

```bash
curl -X POST http://127.0.0.1:4010/bookings \
  -H "Content-Type: application/json" \
  -d '{
    "timeslot_id": "770e8400-e29b-41d4-a716-446655440002",
    "customer_name": "Jane Smith",
    "customer_email": "jane.smith@example.com"
  }'
```

**У Postman:**

1. Відкрийте `Bookings → US-002: Create Booking (Success)`
2. Натисніть **Send**
3. Перевірте:
   - Статус: **201 Created**
   - Час відповіді: **< 300ms** (Performance NFR)
   - `confirmation_code`: формат `BK-2025-XXXXXX`
   - `customer_email_masked`: `j***@e***.com` (Privacy)
4. Автоматично збережеться `bookingId` та `confirmationCode`

**Автотести (Tests tab):**

```javascript
pm.test("Status code is 201", function () {
  pm.response.to.have.status(201);
});

pm.test("Response time < 300ms", function () {
  pm.expect(pm.response.responseTime).to.be.below(300);
});
```

---

### Сценарій 4: Capacity Full (US-002) ❌ Error

```bash
curl -X POST http://127.0.0.1:4010/bookings \
  -H "Content-Type: application/json" \
  -d '{
    "timeslot_id": "880e8400-e29b-41d4-a716-446655440003",
    "customer_name": "John Doe",
    "customer_email": "john.doe@example.com"
  }'
```

**У Postman:**

1. Відкрийте `Bookings → US-002: Create Booking (Capacity Full - 409)`
2. Натисніть **Send**
3. Перевірте:
   - Статус: **409 Conflict**
   - `error_code`: `"slot_unavailable"`
   - `details.current_bookings`: 3
   - `details.max_concurrent_bookings`: 3

**Очікувана помилка:**

```json
{
  "error_code": "slot_unavailable",
  "message": "This timeslot has reached maximum capacity (3/3)",
  "details": {
    "timeslot_id": "880e8400-e29b-41d4-a716-446655440003",
    "current_bookings": 3,
    "max_concurrent_bookings": 3
  }
}
```

---

### Сценарій 5: Перегляд моїх бронювань (US-003)

```bash
curl "http://127.0.0.1:4010/bookings?customer_email=jane.smith@example.com"
```

**У Postman:**

1. Відкрийте `Bookings → US-003: List My Bookings`
2. Натисніть **Send**
3. Перевірте хронологічний порядок (старі → нові)

---

### Сценарій 6: Підтвердження бронювання (US-006)

```bash
curl -X PATCH http://127.0.0.1:4010/bookings/{booking_id} \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.mock" \
  -d '{
    "booking_status": "confirmed"
  }'
```

**У Postman:**

1. Спочатку створіть бронювання (Сценарій 3) - `bookingId` збережеться автоматично
2. Відкрийте `Bookings → US-006: Confirm Booking (Manager)`
3. Натисніть **Send**
4. Перевірте: `booking_status`: `"confirmed"`

---

### Сценарій 7: Помилки валідації

**Invalid Email (400):**

```bash
curl -X POST http://127.0.0.1:4010/bookings \
  -H "Content-Type: application/json" \
  -d '{
    "timeslot_id": "770e8400-e29b-41d4-a716-446655440002",
    "customer_name": "Test",
    "customer_email": "invalid-email"
  }'
```

**У Postman:**

1. Відкрийте `Error Scenarios → Invalid Email Format (400)`
2. Натисніть **Send**
3. Перевірте:
   - Статус: **400 Bad Request**
   - `error_code`: `"invalid_email"`

---

## 📁 Структура файлів для здачі

```
api/
├── openapi.yaml                       # OpenAPI специфікація
├── CoworkSpace.postman_collection.json # Postman колекція
├── README.md                          # Ця інструкція
└── screenshots/                       # Папка зі скріншотами
    ├── 01-list-rooms.png
    ├── 02-get-timeslots.png
    ├── 03-create-booking-success.png
    ├── 04-create-booking-capacity-full.png
    ├── 05-list-my-bookings.png
    ├── 06-confirm-booking.png
    └── 07-error-invalid-email.png

docs/
├── 05-traceability-skeleton.md        # Оновлений з API endpoints
└── 14-api-error-codes.md              # Таблиця кодів помилок
```

---

## 📸 Як зробити скріншоти

### У Postman:

1. Виконайте запит
2. Відкрийте **Response** панель
3. Переконайтеся що видно:
   - HTTP статус (200, 201, 400, 409)
   - Body з JSON відповіддю
   - Headers (X-Response-Time для performance)
4. Зробіть скріншот (Windows: `Win+Shift+S`, Mac: `Cmd+Shift+4`)
5. Збережіть у папку `screenshots/`

### Назви файлів:

- `01-list-rooms.png` - GET /meeting_rooms (200 OK)
- `02-get-timeslots.png` - GET /timeslots (200 OK, показує 1/3 occupied)
- `03-create-booking-success.png` - POST /bookings (201 Created)
- `04-create-booking-capacity-full.png` - POST /bookings (409 Conflict)
- `05-list-my-bookings.png` - GET /bookings?customer_email=...
- `06-confirm-booking.png` - PATCH /bookings/{id} (200 OK)
- `07-error-invalid-email.png` - POST /bookings (400 Bad Request)

---

## ✅ Чеклист здачі

- [ ] `openapi.yaml` - валідний (перевірити на https://editor.swagger.io/)
- [ ] `CoworkSpace.postman_collection.json` - імпортується в Postman
- [ ] Mock server запускається (`prism mock openapi.yaml`)
- [ ] Всі 7+ запитів працюють у Postman
- [ ] Скріншоти зроблені та збережені
- [ ] `05-traceability-skeleton.md` оновлений з API endpoints
- [ ] `14-api-error-codes.md` створений з таблицею помилок
- [ ] README.md містить інструкції запуску

---

## 🎓 NFR Requirements виконані

### ✅ Performance-first (Twist B)

- Заголовок `X-Response-Time` у відповідях
- Пагінація всіх списків (`page`, `limit` max 50)
- Автотести перевіряють час відповіді < 300ms
- Mock імітує швидкі відповіді (< 200ms для GET)

### ✅ Capacity = 3 (Twist A)

- `max_concurrent_bookings: 3` у всіх timeslots
- `current_bookings_count` відображається в GET /timeslots
- Помилка `slot_unavailable` при 3/3 bookings
- Автотести перевіряють capacity logic

### ✅ Стабільні коди помилок

- Машинні коди: `snake_case` (slot_unavailable, invalid_email)
- Структура: `{error_code, message, details}`
- HTTP статуси: 200, 201, 400, 404, 409, 500

### ✅ ISO 8601 UTC

- Всі timestamp: `"2025-01-15T10:00:00Z"`
- Дати у query: `YYYY-MM-DD`
- Duration у хвилинах: `duration_minutes: 60`

---

## 🆘 Troubleshooting

### Prism не запускається

```bash
# Перевстановіть Prism
npm uninstall -g @stoplight/prism-cli
npm install -g @stoplight/prism-cli

# Або використайте npx
npx @stoplight/prism-cli mock openapi.yaml
```

### Postman не бачить змінні

1. Перевірте що **environment** обраний (праворуч зверху)
2. Або використайте змінні з колекції (вони вже є)

### Mock повертає не ті дані

Prism генерує дані з `example:` у OpenAPI. Якщо хочете інші дані:

1. Змініть `example:` у `openapi.yaml`
2. Перезапустіть `prism mock`

---

## 📚 Корисні посилання

- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [Prism Documentation](https://stoplight.io/open-source/prism)
- [Postman Learning Center](https://learning.postman.com/)
- [Swagger Editor (Online)](https://editor.swagger.io/)

---

**Автор:** Савранська Єва
**Студентський квиток:** КВ14736512  
**Варіант:** L=2, P=1, Capacity=3, Performance-first
