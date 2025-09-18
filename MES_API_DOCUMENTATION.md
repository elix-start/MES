# Документация API МЭШ (Московская Электронная Школа)

## Оглавление
1. [Общие сведения](#общие-сведения)
2. [Базовые URL](#базовые-url)
3. [Авторизация и заголовки](#авторизация-и-заголовки)
4. [API расписания](#api-расписания)
5. [API домашних заданий](#api-домашних-заданий)
6. [API оценок](#api-оценок)
7. [API профиля](#api-профиля)
8. [Дополнительные API](#дополнительные-api)
9. [Структуры данных](#структуры-данных)
10. [Примеры запросов](#примеры-запросов)

## Общие сведения

API МЭШ предоставляет доступ к данным электронного дневника через REST API. Все запросы требуют авторизации через токен доступа, полученный через OAuth 2.0 с mos.ru.

### Версии API
- **Основная версия**: v1
- **Мобильная версия**: mobile/v1
- **Календарь событий**: eventcalendar/v1

## Базовые URL

```
MES_SCHOOL_API = "https://school.mos.ru/api/"
MES_DNEVNIK = "https://dnevnik.mos.ru/"
MES_AUTH = "https://login.mos.ru/"
```

## Авторизация и заголовки

### Обязательные заголовки для всех запросов:

```http
Authorization: Bearer {access_token}
# или
auth-token: {access_token}

X-Mes-Subsystem: familymp
X-Mes-Role: student
Client-Type: diary-mobile
Content-Type: application/json
```

### Описание заголовков:
- **Authorization/auth-token**: Токен доступа МЭШ (не mos.ru!)
- **X-Mes-Subsystem**: Подсистема МЭШ (обычно "familymp")
- **X-Mes-Role**: Роль пользователя ("student", "parent", "teacher")
- **Client-Type**: Тип клиента ("diary-mobile")

## API расписания

### Получение событий/уроков

**Эндпоинт:** `GET /api/eventcalendar/v1/api/events`

**Параметры:**
- `person_ids` (обязательный) - ID ученика
- `begin_date` (обязательный) - Начальная дата в формате YYYY-MM-DD
- `end_date` (обязательный) - Конечная дата в формате YYYY-MM-DD
- `expand` (опциональный) - Дополнительные данные: "homework,marks,materials"

**Пример запроса:**
```http
GET /api/eventcalendar/v1/api/events?person_ids=123456&begin_date=2025-09-18&end_date=2025-09-18&expand=homework,marks,materials
```

**Структура ответа:**
```json
{
  "response": [
    {
      "id": 12345678,
      "lesson_number": 1,
      "start_at": "2025-09-18T08:30:00Z",
      "finish_at": "2025-09-18T09:15:00Z",
      "subject_name": "Математика",
      "lesson_topic": "Квадратные уравнения",
      "teacher_name": "Иванов И.И.",
      "room_name": "201",
      "room_number": "201",
      "homework": {
        "descriptions": [
          "Решить задачи №15-20 на стр. 45"
        ]
      },
      "marks": [],
      "materials": []
    }
  ]
}
```

### Получение детальной информации об уроке

**Эндпоинт:** `GET /api/family/mobile/v1/lesson_schedule_items/{lesson_id}`

**Параметры:**
- `lesson_id` (в URL) - ID урока
- `student_id` - ID ученика

## API домашних заданий

### Получение списка домашних заданий

**Эндпоинт:** `GET /api/family/mobile/v1/homeworks`

**Параметры:**
- `student_id` (обязательный) - ID ученика
- `from` (обязательный) - Начальная дата в формате YYYY-MM-DD
- `to` (обязательный) - Конечная дата в формате YYYY-MM-DD
- `sort_column` (опциональный) - Поле сортировки (по умолчанию "date")
- `sort_direction` (опциональный) - Направление сортировки ("asc", "desc")

**Пример запроса:**
```http
GET /api/family/mobile/v1/homeworks?student_id=123456&from=2025-09-18&to=2025-09-25&sort_column=date&sort_direction=asc
```

**Структура ответа:**
```json
{
  "payload": [
    {
      "homework_id": 987654,
      "homework_entry_id": 111222,
      "subject_name": "Математика",
      "homework": "Решить задачи №15-20",
      "description": "Подробное описание задания",
      "date_assigned_on": "2025-09-18",
      "date_prepared_for": "2025-09-20",
      "is_done": false,
      "materials": [
        {
          "title": "Учебник математики",
          "type": "book",
          "url": "https://..."
        }
      ]
    }
  ]
}
```

### Отметить домашнее задание как выполненное

**Эндпоинт:** `POST /api/family/mobile/v1/homeworks/{homework_id}/done`

**Параметры:**
- `homework_id` (в URL) - ID домашнего задания

### Отменить выполнение домашнего задания

**Эндпоинт:** `DELETE /api/family/mobile/v1/homeworks/{homework_id}/done`

## API оценок

### Получение оценок за период

**Эндпоинт:** `GET /api/family/mobile/v1/marks`

**Параметры:**
- `student_id` (обязательный) - ID ученика
- `from` (обязательный) - Начальная дата в формате YYYY-MM-DD
- `to` (обязательный) - Конечная дата в формате YYYY-MM-DD

**Структура ответа:**
```json
{
  "payload": [
    {
      "date": "2025-09-18",
      "marks": [
        {
          "id": 555666,
          "value": "5",
          "weight": 1,
          "subject_name": "Математика",
          "control_form_name": "Контрольная работа",
          "comment": "Отличная работа",
          "created_at": "2025-09-18T10:00:00Z"
        }
      ]
    }
  ]
}
```

### Получение детальной информации об оценке

**Эндпоинт:** `GET /api/family/mobile/v1/marks/{mark_id}`

**Параметры:**
- `mark_id` (в URL) - ID оценки
- `student_id` - ID ученика

### Получение оценок по предметам (краткий формат)

**Эндпоинт:** `GET /api/family/mobile/v1/subject_marks/short`

**Параметры:**
- `student_id` - ID ученика

## API профиля

### Получение профиля пользователя

**Эндпоинт:** `GET /api/family/mobile/v1/profile`

**Структура ответа:**
```json
{
  "children": [
    {
      "id": 123456,
      "first_name": "Иван",
      "last_name": "Петров",
      "middle_name": "Сергеевич",
      "birth_date": "2010-05-15",
      "class_name": "9А",
      "class_unit_id": 789012,
      "school": {
        "id": 345678,
        "name": "ГБОУ Школа №1234",
        "short_name": "Школа №1234"
      }
    }
  ],
  "profile": {
    "id": 654321,
    "first_name": "Анна",
    "last_name": "Петрова",
    "middle_name": "Владимировна",
    "email": "parent@example.com"
  }
}
```

### Получение информации о школе

**Эндпоинт:** `GET /api/family/mobile/v1/school_info`

**Параметры:**
- `school_id` - ID школы
- `class_unit_id` - ID класса

## Дополнительные API

### Рейтинг в классе

**Эндпоинт:** `GET /api/ej/rating/v1/rank/class`

**Параметры:**
- `class_unit_id` - ID класса
- `date` - Дата в формате YYYY-MM-DD

### Посещаемость (только для МЭШ)

**Эндпоинт:** `GET /api/family/mobile/v1/visits`

**Параметры:**
- `contract_id` - ID договора
- `from` - Начальная дата
- `to` - Конечная дата

### Баланс питания

**Эндпоинт:** `GET /api/family/mobile/v1/day-balance-info/v2`

**Параметры:**
- `person_id` - ID ученика
- `from` - Начальная дата
- `limit` - Лимит записей (по умолчанию 40)
- `with_payments` - Включить платежи (true/false)

### Меню питания

**Эндпоинт:** `GET /api/meals/v2/menu/complexes`

**Параметры:**
- `personId` - ID ученика
- `onDate` - Дата в формате YYYY-MM-DD

## Структуры данных

### Урок (Event)
```typescript
interface Event {
  id: number;
  lesson_number: number;
  start_at: string; // ISO 8601 с Z
  finish_at: string; // ISO 8601 с Z
  subject_name: string;
  lesson_topic?: string;
  lesson_theme?: string;
  teacher_name?: string;
  room_name?: string;
  room_number?: string;
  place?: string;
  homework?: {
    descriptions: string[];
  };
  marks?: Mark[];
  materials?: Material[];
  cancelled?: boolean;
  replaced?: boolean;
}
```

### Домашнее задание (Homework)
```typescript
interface Homework {
  homework_id: number;
  homework_entry_id: number;
  subject_name: string;
  homework: string;
  description?: string;
  date_assigned_on: string; // YYYY-MM-DD
  date_prepared_for: string; // YYYY-MM-DD
  lesson_date_time: string; // ISO 8601
  is_done: boolean;
  materials: Material[];
  attachments: any[];
  comments: any[];
}
```

### Оценка (Mark)
```typescript
interface Mark {
  id: number;
  value: string; // "2", "3", "4", "5", "н/а"
  weight: number;
  subject_name: string;
  control_form_name: string;
  comment?: string;
  created_at: string; // ISO 8601
  updated_at: string; // ISO 8601
}
```

### Материал (Material)
```typescript
interface Material {
  id: string;
  title: string;
  type: string; // "book", "video", "link", etc.
  url?: string;
  description?: string;
}
```

## Примеры запросов

### 1. Получение расписания на сегодня

```bash
curl -X GET "https://school.mos.ru/api/eventcalendar/v1/api/events" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "X-Mes-Subsystem: familymp" \
  -H "X-Mes-Role: student" \
  -H "Client-Type: diary-mobile" \
  -G \
  -d "person_ids=123456" \
  -d "begin_date=2025-09-18" \
  -d "end_date=2025-09-18" \
  -d "expand=homework,marks,materials"
```

### 2. Получение домашних заданий на неделю

```bash
curl -X GET "https://school.mos.ru/api/family/mobile/v1/homeworks" \
  -H "auth-token: YOUR_ACCESS_TOKEN" \
  -H "X-Mes-Subsystem: familymp" \
  -H "X-Mes-Role: student" \
  -H "Client-Type: diary-mobile" \
  -G \
  -d "student_id=123456" \
  -d "from=2025-09-18" \
  -d "to=2025-09-25" \
  -d "sort_column=date" \
  -d "sort_direction=asc"
```

### 3. Получение оценок за месяц

```bash
curl -X GET "https://school.mos.ru/api/family/mobile/v1/marks" \
  -H "auth-token: YOUR_ACCESS_TOKEN" \
  -H "X-Mes-Subsystem: familymp" \
  -H "X-Mes-Role: student" \
  -H "Client-Type: diary-mobile" \
  -G \
  -d "student_id=123456" \
  -d "from=2025-08-18" \
  -d "to=2025-09-18"
```

### 4. Отметить домашнее задание как выполненное

```bash
curl -X POST "https://school.mos.ru/api/family/mobile/v1/homeworks/987654/done" \
  -H "auth-token: YOUR_ACCESS_TOKEN" \
  -H "X-Mes-Subsystem: familymp" \
  -H "X-Mes-Role: student" \
  -H "Client-Type: diary-mobile"
```

## Коды ошибок

- **200** - Успешный запрос
- **400** - Неверные параметры запроса
- **401** - Не авторизован или недействительный токен
- **403** - Доступ запрещен
- **404** - Ресурс не найден
- **429** - Превышен лимит запросов
- **500** - Внутренняя ошибка сервера

## Лимиты и ограничения

- **Частота запросов**: Рекомендуется не более 60 запросов в минуту
- **Размер ответа**: Максимум 1000 записей за один запрос
- **Период данных**: Можно запрашивать данные не старше 2 лет
- **Кэширование**: Рекомендуется кэшировать данные на 15-30 минут

## Примечания

1. Все даты и время возвращаются в UTC (с суффиксом Z)
2. Для получения токена доступа используйте OAuth 2.0 через mos.ru
3. Некоторые API доступны только для определенных ролей
4. Структура данных может незначительно отличаться в зависимости от региона
5. Рекомендуется обрабатывать пустые и null значения в ответах
