# 8. Concurrency Flows

## Вступ

У системі "Плай" використано різні патерни конкурентності для забезпечення коректної роботи при паралельних запитах. Оскільки система має монолітну архітектуру, патерни реалізовані на рівні бази даних та бекенд-логіки.

## 1. Обробка замовлень з використанням Database Transactions

```mermaid
sequenceDiagram
    participant К as Користувач
    participant API as Бекенд API
    participant БД as База даних
    participant П as Платіжний шлюз

    К->>API: Створення замовлення
    API->>БД: BEGIN TRANSACTION
    
    par Перевірка наявності
        API->>БД: SELECT quantity FROM books<br/>WHERE id = ? FOR UPDATE
        БД-->>API: кількість товару
    and Резервація товару
        API->>БД: UPDATE books SET quantity = quantity - 1<br/>WHERE id = ? AND quantity > 0
        БД-->>API: результат оновлення
    end
    
    alt Товар в наявності
        API->>БД: INSERT INTO orders (...)
        БД-->>API: ID замовлення
        API->>П: Запит на оплату
        П-->>API: результат оплати
        
        alt Оплата успішна
            API->>БД: UPDATE orders SET status = 'paid'
            API->>БД: COMMIT
            API-->>К: Замовлення підтверджено
        else Оплата невдала
            API->>БД: ROLLBACK
            API-->>К: Помилка оплати
        end
    else Товару немає
        API->>БД: ROLLBACK
        API-->>К: Товар закінчився
    end
```

Transaction Pattern з використанням FOR UPDATE та SELECT ... FOR UPDATE для блокування рядків. Забезпечує атомарність операцій резервації товару та створення замовлення.


## 2. Обробка платежів з використанням Idempotency Key

```mermaid
sequenceDiagram
    participant К as Користувач
    participant API as Бекенд API
    participant БД as База даних
    participant ПШ as Платіжний шлюз

    К->>API: Запит на оплату (idempotency_key=abc123)
    API->>БД: Перевірка: чи був такий запит?
    БД-->>API: Не знайдено
    
    API->>БД: INSERT INTO payment_requests<br/>(idempotency_key, status)
    API->>ПШ: Запит оплати з idempotency_key
    
    par Чекаємо відповіді
        ПШ-->>API: Результат оплати
    and Паралельний запит
        К->>API: Повторний запит (idempotency_key=abc123)
        API->>БД: Перевірка idempotency_key
        БД-->>API: Знайдено - запит в обробці
        API-->>К: Запит вже обробляється
    end
    
    API->>БД: Оновлення статусу оплати
    API-->>К: Фінальний результат
```

Idempotency Pattern забезпечує, що повторні виклики API з тим самим idempotency_key не призводять до повторної сплати. Критично для платіжних операцій.


## 3. Кешування каталогу з використанням Read-Through Cache

```mermaid
sequenceDiagram
    participant К as Користувач
    participant API as Бекенд API
    participant КЕШ as Redis Cache
    participant БД as База даних

    К->>API: Пошук книг
    API->>КЕШ: GET catalog:search:query
    alt Дані в кеші
        КЕШ-->>API: Кешовані результати
        API-->>К: Відповідь з кешу
    else Кеш промах
        API->>БД: SELECT * FROM books WHERE ...
        БД-->>API: Результати пошуку
        API->>КЕШ: SET catalog:search:query<br/>EX 300 (5 хв)
        API-->>К: Відповідь з БД
    end
    
    Note right of API: Інвалідація кешу<br/>при оновленні каталогу
    API->>КЕШ: DEL catalog:search:*
```

Cache-Aside Pattern з автоматичною інвалідацією кешу при змінах. Зменшує навантаження на БД при частому читанні каталогу.


## 4. Фонова обробка email-сповіщень

```mermaid
sequenceDiagram
    participant API as Бекенд API
    participant ЧЕРГА as Database Queue
    participant ВОРКЕР as Background Worker
    participant ПОШТА as Email Service

    API->>ЧЕРГА: INSERT INTO email_queue<br/>(type, user_id, data)
    ЧЕРГА-->>API: Успішно додано
    
    loop Фонова обробка
        ВОРКЕР->>ЧЕРГА: SELECT * FROM email_queue<br/>WHERE processed = false LIMIT 10<br/>FOR UPDATE SKIP LOCKED
        ЧЕРГА-->>ВОРКЕР: Завдання для обробки
        
        par Обробка сповіщень
            ВОРКЕР->>ПОШТА: Відправка email
            ПОШТА-->>ВОРКЕР: Результат
            ВОРКЕР->>ЧЕРГА: UPDATE email_queue SET processed = true
        end
    end
```

Background Job Pattern з використанням SKIP LOCKED для уникнення блокувань. Забезпечує асинхронну обробку без блокування основного потоку.


## 5. Обмеження запитів API (Rate Limiting) з Token Bucket
```mermaid
sequenceDiagram
    participant К as Клієнт
    participant API as Бекенд API
    participant РЛ as Rate Limiter
    participant Redis as Redis Cache

    К->>API: API Запит (/api/books/search)
    
    API->>РЛ: checkRateLimit(user_id, endpoint)
    
    РЛ->>Redis: GET rate_limit:user123:search
    alt Ключ не існує
        Redis-->>РЛ: null
        РЛ->>Redis: SET rate_limit:user123:search 1<br/>EX 60 EXNX
        РЛ-->>API: Ліміт: 1/100
        API->>API: Обробка запиту
        API-->>К: 200 OK + дані
    else Ключ існує
        Redis-->>РЛ: поточне значення (напр: 45)
        РЛ->>Redis: INCR rate_limit:user123:search
        Redis-->>РЛ: нове значення (46)
        
        alt Ліміт не перевищено (46 ≤ 100)
            РЛ-->>API: Дозвіл (46/100)
            API->>API: Обробка запиту
            API-->>К: 200 OK + дані
        else Ліміт перевищено (101 > 100)
            РЛ-->>API: Заборона
            API-->>К: 429 Too Many Requests<br/>Retry-After: 60
        end
    end
```

Token Bucket Algorithm реалізований через Redis. Кожен користувач має окремий лічильник для кожного endpoint. При першому запиті створюється ключ з TTL 60 секунд. Кожен наступний запит збільшує ліміт. Коли лічильник досягає максимуму (напр. 100 запитів за 60 сек), подальші запити блокуются з HTTP 429.

Переваги:

- Гнучкість: різні ліміти для різних endpoint
- Простота: легка реалізація через Redis
- Ефективність: мінімальне навантаження на API



## 6. Конкурентне оновлення рейтингів книг
```mermaid
sequenceDiagram
    participant К1 as Користувач 1
    participant К2 as Користувач 2
    participant API as Бекенд API
    participant БД as База даних

    К1->>API: Додати відгук (rating=5)
    К2->>API: Додати відгук (rating=4)
    
    par Оновлення 1
        API->>БД: SELECT rating_data FROM books WHERE id = ?
        БД-->>API: поточні дані
        API->>БД: UPDATE books SET rating_data = ?<br/>WHERE id = ? AND version = ?
    and Оновлення 2
        API->>БД: SELECT rating_data FROM books WHERE id = ?
        БД-->>API: поточні дані
        API->>БД: UPDATE books SET rating_data = ?<br/>WHERE id = ? AND version = ?
    end
    
    БД-->>API: Успіх (1 оновлення)
    БД-->>API: Помилка (версія змінена)
    
    API->>API: Retry logic для конфліктних оновлень
```

Optimistic Concurrency Control з використанням версій записів. Дозволяє паралельні оновлення з автоматичним вирішенням конфліктів.

Використання цих патернів конкурентності дозволяє системі "Плай":

- Забезпечувати консистентність даних при паралельних замовленнях
- Обробляти пікові навантаження без втрати даних
- Захищати API від зловживань та перевантажень
- Покращувати продуктивність через кешування та фонова обробку
- Забезпечувати ідемпотентність критичних операцій