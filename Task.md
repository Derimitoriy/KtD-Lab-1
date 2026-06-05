graph TD
    Client[Client: Web / Mobile] -->|1. Send Group Msg| API[Backend API]
    API -->|2. Authorize & Save| MsgService[Message Service]
    MsgService -->|3. Persist Msg & Group Metadata| DB[(Database)]
    MsgService -->|4. Trigger Fan-out| FanOut[Fan-out / Queue Service]
    FanOut -->|5. Push to Active Users| Delivery[WebSocket / Push Service]
    Delivery -->|6. Deliver| ClientB[Recipient User B]
    Delivery -->|7. Deliver| ClientC[Recipient User C]

    sequenceDiagram
    autonumber
    actor UserA as Користувач А (Онлайн)
    participant API as Backend API / Msg Service
    participant DB as Database
    participant Queue as Fan-out / Queue
    actor UserB as Користувач Б (Офлайн)
    actor UserC as Користувач В (Онлайн)

    UserA->>API: Відправка повідомлення в Групу Х
    Note over API: Перевірка, чи А є в групі
    API->>DB: Збереження тексту повідомлення (Msg_ID)
    API->>Queue: Ініціація Fan-out для учасників (Б і В)
    
    par Для Користувача В (Онлайн)
        Queue->>UserC: Доставка через WebSocket
        UserC-->>API: Підтвердження отримання (Delivered для В)
        API->>DB: Оновлення статусу повідомлення для Учасника В -> Delivered
    and Для Користувача Б (Офлайн)
        Queue->>DB: Збереження статусу для Учасника Б -> Sent (в чергу)
        Note over Queue, UserB: Очікування, поки Б вийде в мережу
    end

    stateDiagram-v2
    [*] --> Created : Повідомлення створено (User A)
    Created --> SentToBackend : Надіслано на сервер
    
    state "Статуси для конкретного отримувача" as PerUser {
        [*] --> Sent : Збережено в базі / черзі
        Sent --> Delivered : Доставлено на пристрій (WebSocket/Push)
        Delivered --> Read : Прочитано (Відкрито чат)
    }
    
    SentToBackend --> PerUser : Ініціалізація для кожного учасника
    Read --> [*] : Кінцевий стан для цього юзера

    # ADR-002: Fan-out on Write for Group Chat Architecture

## Status
Accepted

## Context
В нашому месенджері є групові чати. Повідомлення, надіслане в групу, має швидко доставлятися багатьом користувачам одночасно, а статус доставки (Sent/Delivered/Read) повинен відстежуватися для кожного учасника окремо.

## Decision
Ми приймаємо рішення використовувати підхід **Fan-out on Write** за допомогою брокера повідомлень (Message Queue, наприклад, RabbitMQ/Kafka). 
Коли повідомлення потрапляє на бекенд, спеціальний сервіс визначає список усіх учасників групи і миттєво створює задачу/копію вказівника на повідомлення в персональну чергу доставки кожного активного користувача.

## Alternatives
- **Fan-out on Read:** Зберігати повідомлення в одній таблиці групи, а клієнти самі будуть постійно опитувати (polling) базу даних на наявність нових ID. (Відхилено через величезне навантаження на читання БД при зростанні кількості користувачів).

## Consequences
- **+ Performance (Швидкість доставки):** Мінімальна затримка для кінцевих користувачів, які зараз онлайн.
- **+ Ізоляція статусів:** Легко трекати індивідуальні статуси (Delivered/Read) у персональних чергах/таблицях зв'язків.
- **- Storage/Complexity Overhead:** Потребує більше пам'яті в момент відправки повідомлення у великі групи (якщо в групі 10 000 людей, буде згенеровано 10 000 подій доставки).
