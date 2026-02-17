# Приложение D. Документация REST API

**Базовый URL:** `https://api.technosfera.ru/api/v1`  
**Формат данных:** JSON  
**Аутентификация:** Bearer JWT Token (в заголовке `Authorization: Bearer <access_token>`)  
**Кодировка:** UTF-8  

---

## Общие правила

### Заголовки запроса

| Заголовок | Обязательность | Описание |
|-----------|---------------|----------|
| `Content-Type` | Да (для POST/PUT/PATCH) | `application/json` |
| `Authorization` | Условно | `Bearer <access_token>` - для защищённых эндпоинтов |
| `X-Session-Id` | Условно | Идентификатор сессии гостя (для работы с корзиной без авторизации) |
| `Accept-Language` | Нет | `ru` (по умолчанию) |

### Формат ответа об ошибке

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Ошибка валидации",
    "details": [
      {
        "field": "email",
        "message": "Некорректный формат email"
      }
    ]
  }
}
```

### Коды ошибок

| HTTP-код | Код ошибки | Описание |
|----------|-----------|----------|
| 400 | BAD_REQUEST | Некорректный запрос |
| 401 | UNAUTHORIZED | Требуется авторизация |
| 403 | FORBIDDEN | Доступ запрещён (недостаточно прав) |
| 404 | NOT_FOUND | Ресурс не найден |
| 409 | CONFLICT | Конфликт (email занят, товар закончился) |
| 422 | UNPROCESSABLE_ENTITY | Ошибка валидации / недопустимое действие |
| 429 | TOO_MANY_REQUESTS | Превышен лимит запросов (100 req/min для гостей, 300 для авторизованных) |
| 500 | INTERNAL_ERROR | Внутренняя ошибка сервера |

### Формат пагинации

```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "per_page": 24,
    "total": 156,
    "total_pages": 7
  }
}
```

---

## 1. Аутентификация (Auth)

### 1.1 POST /auth/register - Регистрация

**Доступ:** Гость  
**Rate limit:** 5 запросов в минуту с одного IP

**Тело запроса:**
```json
{
  "first_name": "Иван",
  "last_name": "Петров",
  "email": "ivan@example.com",
  "phone": "+79991234567",
  "password": "SecurePass1", 
  "password_confirmation": "SecurePass1"
}
```

**Валидация:**

| Поле | Правила |
|------|---------|
| first_name | Обязательное, 2-100 символов, только буквы (кириллица/латиница) |
| last_name | Обязательное, 2-100 символов, только буквы |
| email | Обязательное, RFC 5322, уникальный |
| phone | Обязательное, формат +7XXXXXXXXXX, уникальный |
| password | Обязательное, минимум 8 символов, 1 заглавная, 1 цифра |
| password_confirmation | Обязательное, должен совпадать с password |

**Ответ 201:**
```json
{
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "first_name": "Иван",
      "last_name": "Петров",
      "email": "ivan@example.com",
      "phone": "+79991234567",
      "role": "customer",
      "created_at": "2026-02-09T10:30:00Z"
    },
    "tokens": {
      "access_token": "eyJhbGciOiJSUzI1NiIs...",
      "refresh_token": "dGhpcyBpcyBhIHJlZnJl...",
      "expires_in": 1800
    }
  }
}
```

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 409 CONFLICT | Email или телефон уже зарегистрирован |
| 422 UNPROCESSABLE_ENTITY | Ошибка валидации полей |

---

### 1.2 POST /auth/login - Авторизация

**Доступ:** Гость  
**Rate limit:** 10 запросов в минуту с одного IP

**Тело запроса:**
```json
{
  "email": "ivan@example.com",
  "password": "SecurePass1"
}
```

**Ответ 200:**
```json
{
  "data": {
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "first_name": "Иван",
      "last_name": "Петров",
      "email": "ivan@example.com",
      "phone": "+79991234567",
      "role": "customer"
    },
    "tokens": {
      "access_token": "eyJhbGciOiJSUzI1NiIs...",
      "refresh_token": "dGhpcyBpcyBhIHJlZnJl...",
      "expires_in": 1800
    }
  }
}
```

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 401 UNAUTHORIZED | Неверный email или пароль |
| 403 FORBIDDEN | Аккаунт заблокирован |
| 429 TOO_MANY_REQUESTS | Слишком много попыток (блокировка на 15 мин после 5 неудач) |

---

### 1.3 POST /auth/refresh - Обновление токена

**Доступ:** Авторизованный  

**Тело запроса:**
```json
{
  "refresh_token": "dGhpcyBpcyBhIHJlZnJl..."
}
```

**Ответ 200:**
```json
{
  "data": {
    "access_token": "eyJhbGciOiJSUzI1NiIs...",
    "refresh_token": "bmV3IHJlZnJlc2ggdG9r...",
    "expires_in": 1800
  }
}
```

---

### 1.4 POST /auth/logout - Выход

**Доступ:** Авторизованный (Bearer Token)

**Ответ 204:** No Content (refresh_token инвалидируется)

---

### 1.5 POST /auth/reset-password - Запрос восстановления пароля

**Тело запроса:**
```json
{
  "email": "ivan@example.com"
}
```

**Ответ 200:**
```json
{
  "message": "Если аккаунт с таким email существует, мы отправили инструкции по восстановлению"
}
```

> Одинаковый ответ вне зависимости от существования email (безопасность).

---

## 2. Каталог товаров (Products)

### 2.1 GET /products - Список товаров

**Доступ:** Все  
**Кешируется:** 5 минут (CDN + Redis)

**Query-параметры:**

| Параметр | Тип | По умолч. | Описание |
|----------|-----|-----------|----------|
| page | Integer | 1 | Номер страницы |
| per_page | Integer | 24 | Количество на странице (макс. 100) |
| category_id | UUID | - | Фильтр по категории |
| brand | String | - | Фильтр по бренду (точное, можно несколько через запятую) |
| price_min | Decimal | - | Минимальная цена |
| price_max | Decimal | - | Максимальная цена |
| in_stock | Boolean | - | true - только товары в наличии |
| sort | Enum | relevance | relevance, price_asc, price_desc, newest, popularity |

**Ответ 200:**
```json
{
  "data": [
    {
      "id": "prod-uuid-001",
      "sku": "TP-X15-128",
      "name": "Смартфон TechPhone X15 128GB",
      "slug": "smartfon-techphone-x15-128gb",
      "price": 29990.00,
      "old_price": 34990.00,
      "brand": "TechPhone",
      "category": {
        "id": "cat-uuid-001",
        "name": "Смартфоны",
        "slug": "smartfony"
      },
      "image_url": "https://cdn.technosfera.ru/products/tp-x15-main.jpg",
      "in_stock": true,
      "total_available": 42,
      "rating_avg": 4.5,
      "reviews_count": 128
    }
  ],
  "filters": {
    "brands": [
      {"name": "TechPhone", "count": 15},
      {"name": "SoundMax", "count": 8}
    ],
    "price_range": {"min": 990, "max": 199990},
    "categories": [
      {"id": "cat-uuid-001", "name": "Смартфоны", "count": 42}
    ]
  },
  "pagination": {
    "page": 1,
    "per_page": 24,
    "total": 156,
    "total_pages": 7
  }
}
```

---

### 2.2 GET /products/{slug} - Карточка товара

**Доступ:** Все  
**Кешируется:** 5 минут

**Ответ 200:**
```json
{
  "data": {
    "id": "prod-uuid-001",
    "sku": "TP-X15-128",
    "name": "Смартфон TechPhone X15 128GB",
    "slug": "smartfon-techphone-x15-128gb",
    "description": "Флагманский смартфон с 6.7\" AMOLED экраном...",
    "price": 29990.00,
    "old_price": 34990.00,
    "brand": "TechPhone",
    "category": {
      "id": "cat-uuid-001",
      "name": "Смартфоны",
      "slug": "smartfony",
      "breadcrumbs": [
        {"name": "Каталог", "slug": "catalog"},
        {"name": "Электроника", "slug": "elektronika"},
        {"name": "Смартфоны", "slug": "smartfony"}
      ]
    },
    "images": [
      {"url": "https://cdn.technosfera.ru/products/tp-x15-1.jpg", "alt": "TechPhone X15 вид спереди", "sort_order": 1},
      {"url": "https://cdn.technosfera.ru/products/tp-x15-2.jpg", "alt": "TechPhone X15 вид сзади", "sort_order": 2}
    ],
    "specifications": [
      {"name": "Экран", "value": "6.7\"", "unit": "AMOLED, 120Hz"},
      {"name": "Процессор", "value": "Snapdragon 8 Gen 3", "unit": ""},
      {"name": "Оперативная память", "value": "8", "unit": "ГБ"},
      {"name": "Память", "value": "128", "unit": "ГБ"},
      {"name": "Аккумулятор", "value": "5000", "unit": "мАч"}
    ],
    "in_stock": true,
    "total_available": 42,
    "pickup_points_available": 3,
    "rating_avg": 4.5,
    "reviews_count": 128,
    "created_at": "2026-01-15T08:00:00Z"
  }
}
```

---

### 2.3 GET /products/search - Поиск товаров

**Доступ:** Все  
**Все query-параметры из п. 2.1 +**:

| Параметр | Тип | Описание |
|----------|-----|----------|
| q | String | Поисковый запрос (минимум 2 символа) |

**Ответ 200:** Аналогичен GET /products с дополнительным полем:
```json
{
  "data": [...],
  "meta": {
    "query": "iphone",
    "corrected_query": null,
    "suggestions": ["iphone 15", "iphone case"]
  },
  "filters": {...},
  "pagination": {...}
}
```

---

### 2.4 GET /search/suggest - Подсказки поиска

**Доступ:** Все

| Параметр | Тип | Описание |
|----------|-----|----------|
| q | String | Запрос (минимум 3 символа) |
| limit | Integer | Максимум подсказок (по умолч. 5, макс. 10) |

**Ответ 200:**
```json
{
  "data": [
    {
      "name": "Смартфон TechPhone X15 128GB",
      "slug": "smartfon-techphone-x15-128gb",
      "price": 29990.00,
      "image_url": "https://cdn.technosfera.ru/products/tp-x15-thumb.jpg"
    }
  ]
}
```

---

## 3. Категории (Categories)

### 3.1 GET /categories - Дерево категорий

**Доступ:** Все  
**Кешируется:** 30 минут

**Ответ 200:**
```json
{
  "data": [
    {
      "id": "cat-uuid-root-01",
      "name": "Электроника",
      "slug": "elektronika",
      "icon_url": "https://cdn.technosfera.ru/icons/electronics.svg",
      "products_count": 1250,
      "children": [
        {
          "id": "cat-uuid-001",
          "name": "Смартфоны",
          "slug": "smartfony",
          "products_count": 342,
          "children": []
        },
        {
          "id": "cat-uuid-002",
          "name": "Ноутбуки",
          "slug": "noutbuki",
          "products_count": 215,
          "children": [
            {
              "id": "cat-uuid-003",
              "name": "Игровые ноутбуки",
              "slug": "igrovye-noutbuki",
              "products_count": 78,
              "children": []
            }
          ]
        }
      ]
    }
  ]
}
```

---

## 4. Корзина (Cart)

### 4.1 GET /cart - Получить корзину

**Доступ:** Все (по `Authorization` или `X-Session-Id`)

**Ответ 200:**
```json
{
  "data": {
    "id": "cart-uuid-001",
    "items": [
      {
        "id": "ci-uuid-001",
        "product": {
          "id": "prod-uuid-001",
          "sku": "TP-X15-128",
          "name": "Смартфон TechPhone X15 128GB",
          "slug": "smartfon-techphone-x15-128gb",
          "price": 29990.00,
          "image_url": "https://cdn.technosfera.ru/products/tp-x15-thumb.jpg",
          "in_stock": true,
          "available": 42
        },
        "quantity": 1,
        "price_at_add": 29990.00,
        "price_changed": false,
        "item_total": 29990.00
      }
    ],
    "items_count": 3,
    "subtotal": 45960.00,
    "has_unavailable_items": false
  }
}
```

---

### 4.2 POST /cart/items - Добавить товар в корзину

**Доступ:** Все

**Тело запроса:**
```json
{
  "product_id": "prod-uuid-001",
  "quantity": 1
}
```

**Ответ 200:** Обновлённая корзина (формат как GET /cart)

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 404 | Товар не найден или неактивен |
| 422 | Количество превышает доступный остаток |

---

### 4.3 PATCH /cart/items/{item_id} - Изменить количество

**Доступ:** Все

**Тело запроса:**
```json
{
  "quantity": 3
}
```

| Поле | Правила |
|------|---------|
| quantity | 1-99, не более available |

**Ответ 200:** Обновлённая корзина

---

### 4.4 DELETE /cart/items/{item_id} - Удалить товар из корзины

**Доступ:** Все

**Ответ 200:** Обновлённая корзина

---

### 4.5 DELETE /cart - Очистить корзину

**Доступ:** Все

**Ответ 204:** No Content

---

## 5. Пункты выдачи (Pickup Points)

### 5.1 GET /pickup-points - Список ПВЗ

**Доступ:** Все

**Query-параметры:**

| Параметр | Тип | Описание |
|----------|-----|----------|
| city | String | Фильтр по городу |
| lat | Decimal | Широта (для сортировки по расстоянию) |
| lng | Decimal | Долгота |
| product_ids | String | Comma-separated список product_id (для проверки наличия) |

**Ответ 200:**
```json
{
  "data": [
    {
      "id": "pp-uuid-001",
      "name": "ПВЗ «Центральный»",
      "address": "ул. Ленина, 42",
      "city": "Москва",
      "latitude": 55.753215,
      "longitude": 37.622504,
      "working_hours": [
        {"day_of_week": 1, "open": "09:00", "close": "21:00"},
        {"day_of_week": 2, "open": "09:00", "close": "21:00"},
        {"day_of_week": 6, "open": "10:00", "close": "18:00"},
        {"day_of_week": 7, "open": "10:00", "close": "18:00"}
      ],
      "phone": "+74951234567",
      "product_availability": {
        "all_available": true,
        "items": [
          {"product_id": "prod-uuid-001", "available": 5},
          {"product_id": "prod-uuid-002", "available": 12}
        ]
      }
    }
  ]
}
```

---

## 6. Заказы (Orders)

### 6.1 POST /orders - Создание заказа

**Доступ:** Все (авторизованный или гость с `X-Session-Id`)

**Тело запроса:**
```json
{
  "contact": {
    "first_name": "Иван",
    "last_name": "Петров",
    "email": "ivan@example.com",
    "phone": "+79991234567"
  },
  "delivery_type": "pickup",
  "pickup_point_id": "pp-uuid-001",
  "delivery_address": null,
  "delivery_date": null,
  "delivery_time_slot": null,
  "payment_type": "cash_on_delivery",
  "promo_code": null,
  "bonus_points_used": 0,
  "comment": "Позвоните за 30 минут"
}
```

**Валидация:**

| Поле | Правила |
|------|---------|
| contact.first_name | Обязательное, 2-100 символов |
| contact.last_name | Обязательное, 2-100 символов |
| contact.email | Обязательное, RFC 5322 |
| contact.phone | Обязательное, +7XXXXXXXXXX |
| delivery_type | Обязательное: `pickup` (V1), `courier` (V2) |
| pickup_point_id | Обязательное если delivery_type = pickup |
| delivery_address | (V2) Обязательное если delivery_type = courier |
| payment_type | Обязательное: `cash_on_delivery` (V1), `card_online` (V2), `sbp` (V2) |
| promo_code | (V2) Строка, 4-50 символов |
| bonus_points_used | (V3) 0 - целое, не более 30% от суммы, не более баланса |
| comment | До 1000 символов |

**Ответ 201:**
```json
{
  "data": {
    "id": "order-uuid-001",
    "order_number": "TS-20260209-00042",
    "status": "new",
    "contact": {
      "first_name": "Иван",
      "last_name": "Петров",
      "email": "ivan@example.com",
      "phone": "+79991234567"
    },
    "delivery_type": "pickup",
    "pickup_point": {
      "id": "pp-uuid-001",
      "name": "ПВЗ «Центральный»",
      "address": "ул. Ленина, 42"
    },
    "payment_type": "cash_on_delivery",
    "items": [
      {
        "product_id": "prod-uuid-001",
        "product_name": "Смартфон TechPhone X15 128GB",
        "product_sku": "TP-X15-128",
        "price": 29990.00,
        "quantity": 1,
        "total": 29990.00
      }
    ],
    "subtotal": 45960.00,
    "discount_amount": 0.00,
    "delivery_cost": 0.00,
    "total": 45960.00,
    "comment": "Позвоните за 30 минут",
    "payment_url": null,
    "created_at": "2026-02-09T14:30:00Z"
  }
}
```

> Если `payment_type` = `card_online` или `sbp`, поле `payment_url` будет содержать URL для редиректа на платёжный шлюз. Статус заказа будет `awaiting_payment`.

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 409 CONFLICT | Товар закончился между добавлением в корзину и оформлением |
| 422 UNPROCESSABLE_ENTITY | Ошибка валидации, корзина пуста, ПВЗ не найден |

---

### 6.2 GET /orders - Мои заказы (Покупатель)

**Доступ:** Авторизованный (customer)

| Параметр | Тип | Описание |
|----------|-----|----------|
| page | Integer | Номер страницы |
| per_page | Integer | 20 по умолчанию |
| status | String | Фильтр по статусу |

**Ответ 200:**
```json
{
  "data": [
    {
      "id": "order-uuid-001",
      "order_number": "TS-20260209-00042",
      "status": "new",
      "status_label": "Новый",
      "items_count": 3,
      "total": 45960.00,
      "created_at": "2026-02-09T14:30:00Z"
    }
  ],
  "pagination": {...}
}
```

---

### 6.3 GET /orders/{id} - Детали заказа

**Доступ:** Авторизованный (владелец заказа или менеджер/админ)

**Ответ 200:**
```json
{
  "data": {
    "id": "order-uuid-001",
    "order_number": "TS-20260209-00042",
    "status": "assembling",
    "status_label": "Собирается",
    "contact": {...},
    "delivery_type": "pickup",
    "pickup_point": {...},
    "payment_type": "cash_on_delivery",
    "items": [...],
    "subtotal": 45960.00,
    "discount_amount": 0.00,
    "delivery_cost": 0.00,
    "total": 45960.00,
    "comment": "Позвоните за 30 минут",
    "status_history": [
      {
        "status": "new",
        "status_label": "Новый",
        "created_at": "2026-02-09T14:30:00Z",
        "comment": null
      },
      {
        "status": "assembling",
        "status_label": "Собирается",
        "created_at": "2026-02-09T15:10:00Z",
        "comment": null
      }
    ],
    "can_cancel": false,
    "created_at": "2026-02-09T14:30:00Z"
  }
}
```

---

### 6.4 POST /orders/{id}/cancel - Отмена заказа

**Доступ:** Авторизованный (владелец заказа)

**Тело запроса:**
```json
{
  "reason": "changed_mind",
  "comment": "Нашёл дешевле в другом магазине"
}
```

| Поле | Правила |
|------|---------|
| reason | Обязательное: `changed_mind`, `found_cheaper`, `long_wait`, `other` |
| comment | До 500 символов |

**Ответ 200:** Обновлённый заказ (статус `cancelled`)

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 422 | Текущий статус заказа не позволяет отмену |

---

### 6.5 POST /orders/{id}/retry-payment - Повторная оплата (V2)

**Доступ:** Авторизованный (владелец заказа)  
**Предусловие:** Заказ в статусе `awaiting_payment`

**Ответ 200:**
```json
{
  "data": {
    "payment_url": "https://pay.gateway.ru/payment/xyz123"
  }
}
```

---

### 6.6 GET /order-status - Проверка статуса (для гостей)

**Доступ:** Все

| Параметр | Тип | Описание |
|----------|-----|----------|
| number | String | Номер заказа (TS-XXXXXXXX-XXXXX) |
| email | String | Email, указанный при оформлении |

**Ответ 200:** Сокращённая информация о заказе (статус, дата, сумма)

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 404 | Заказ не найден или email не совпадает |

---

## 7. Промокоды (V2)

### 7.1 POST /promo-codes/validate - Проверка промокода

**Доступ:** Все

**Тело запроса:**
```json
{
  "code": "SALE2026",
  "cart_subtotal": 45960.00
}
```

**Ответ 200:**
```json
{
  "data": {
    "code": "SALE2026",
    "discount_type": "percent",
    "discount_value": 10,
    "discount_amount": 4596.00,
    "new_total": 41364.00,
    "valid": true
  }
}
```

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 404 | Промокод не найден |
| 422 | Промокод истёк / исчерпан / минимальная сумма не достигнута |

---

## 8. Отзывы (Reviews, V2)

### 8.1 GET /products/{product_id}/reviews - Отзывы на товар

**Доступ:** Все

| Параметр | Тип | Описание |
|----------|-----|----------|
| page | Integer | Страница |
| per_page | Integer | 10 по умолчанию |
| sort | Enum | newest (по умолч.), rating_desc, rating_asc, helpful |

**Ответ 200:**
```json
{
  "data": {
    "summary": {
      "avg_rating": 4.5,
      "total_reviews": 128,
      "distribution": {
        "5": 72,
        "4": 31,
        "3": 15,
        "2": 7,
        "1": 3
      }
    },
    "reviews": [
      {
        "id": "rev-uuid-001",
        "user_name": "Иван П.",
        "rating": 5,
        "text": "Отличный смартфон, камера супер!",
        "is_verified_purchase": true,
        "created_at": "2026-02-01T12:00:00Z"
      }
    ]
  },
  "pagination": {...}
}
```

---

### 8.2 POST /products/{product_id}/reviews - Оставить отзыв

**Доступ:** Авторизованный (customer, с подтверждённой покупкой)

**Тело запроса:**
```json
{
  "rating": 5,
  "text": "Отличный смартфон, камера супер!"
}
```

| Поле | Правила |
|------|---------|
| rating | Обязательное, 1-5 |
| text | До 5000 символов |

**Ответ 201:**
```json
{
  "data": {
    "id": "rev-uuid-002",
    "status": "pending",
    "message": "Ваш отзыв отправлен на модерацию"
  }
}
```

**Ошибки:**
| Код | Ситуация |
|-----|----------|
| 403 | Пользователь не покупал данный товар |
| 409 | Отзыв от этого пользователя уже существует |

---

## 9. Админ-панель (Admin)

### 9.1 GET /admin/orders - Список заказов

**Доступ:** manager, admin

| Параметр | Тип | Описание |
|----------|-----|----------|
| page, per_page | Integer | Пагинация (per_page по умолч. 50) |
| status | String | Фильтр по статусу |
| date_from | Date | От даты (YYYY-MM-DD) |
| date_to | Date | До даты |
| search | String | Поиск по номеру заказа, имени или email |
| pickup_point_id | UUID | Фильтр по ПВЗ |

---

### 9.2 PATCH /admin/orders/{id}/status - Смена статуса заказа

**Доступ:** manager, admin

**Тело запроса:**
```json
{
  "status": "assembling",
  "comment": "Начал сборку"
}
```

**Валидация:** Проверка допустимости перехода по матрице из диаграммы состояний (см. Приложение C).

**Ответ 200:** Обновлённый заказ

---

### 9.3 CRUD Товары (admin)

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | /admin/products | Список товаров с расширенными фильтрами |
| GET | /admin/products/{id} | Детали товара (включая остатки по ПВЗ) |
| POST | /admin/products | Создание товара |
| PUT | /admin/products/{id} | Полное обновление товара |
| PATCH | /admin/products/{id} | Частичное обновление |
| DELETE | /admin/products/{id} | Архивирование товара |

### 9.4 CRUD Категории (admin)

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | /admin/categories | Плоский список с parent_id |
| POST | /admin/categories | Создание категории |
| PUT | /admin/categories/{id} | Обновление |
| DELETE | /admin/categories/{id} | Удаление (только если нет товаров) |

### 9.5 CRUD ПВЗ (admin)

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | /admin/pickup-points | Список ПВЗ |
| POST | /admin/pickup-points | Создание ПВЗ |
| PUT | /admin/pickup-points/{id} | Обновление |
| DELETE | /admin/pickup-points/{id} | Деактивация (soft delete) |

### 9.6 CRUD Промокоды (admin, V2)

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | /admin/promo-codes | Список промокодов |
| POST | /admin/promo-codes | Создание промокода |
| PUT | /admin/promo-codes/{id} | Обновление |
| PATCH | /admin/promo-codes/{id}/deactivate | Деактивация |

### 9.7 Управление пользователями (admin)

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | /admin/users | Список пользователей с фильтрами |
| GET | /admin/users/{id} | Детали пользователя (+ его заказы) |
| PATCH | /admin/users/{id}/role | Смена роли |
| PATCH | /admin/users/{id}/block | Блокировка |
| PATCH | /admin/users/{id}/unblock | Разблокировка |

### 9.8 Управление остатками (admin)

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | /admin/stock | Все остатки с фильтрами (по товару, по ПВЗ) |
| PUT | /admin/stock/{product_id}/{pickup_point_id} | Установка остатка |

**Тело запроса PUT /admin/stock:**
```json
{
  "quantity": 100
}
```

---

## 10. Webhook (входящие, V2)

### 10.1 POST /webhooks/payment - Уведомление от платёжного шлюза

**Доступ:** Только с IP-адресов платёжного шлюза (whitelist)  
**Подпись:** Проверка HMAC SHA-256 в заголовке `X-Signature`

**Тело запроса (от шлюза):**
```json
{
  "payment_id": "pay-ext-001",
  "order_id": "order-uuid-001",
  "status": "success",
  "amount": 45960.00,
  "currency": "RUB",
  "paid_at": "2026-02-09T14:35:00Z"
}
```

**Ответ 200:** `{"status": "ok"}`

> При получении `status: success` - заказ переводится в статус `paid`.  
> При получении `status: failed` - транзакция помечается как `failed`, заказ остаётся в `awaiting_payment`.
