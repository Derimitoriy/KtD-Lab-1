# Laboratory Work 1: Designing a Messaging System
**Variant:** Variant 4 — Group Chat (Focus: scaling delivery logic)

---

## 🧱 Part 1 — Component Diagram

Component Diagram відображає основні елементи системи групового чату та забезпечує логіку розгалуження повідомлень (Fan-out) для багатьох отримувачів.

```mermaid
graph TD
    Client["Client: Web / Mobile"] -->|"1. Send Group Msg"| API["Backend API"]
    API -->|"2. Authorize & Save"| MsgService["Message Service"]
    MsgService -->|"3. Persist Msg & Group Metadata"| DB[("Database")]
    MsgService -->|"4. Trigger Fan-out"| FanOut["Fan-out / Queue Service"]
    FanOut -->|"5. Push to Active Users"| Delivery["WebSocket / Push Service"]
    Delivery -->|"6. Deliver"| ClientB["Recipient User B"]
    Delivery -->|"7. Deliver"| ClientC["Recipient User C"]

sequenceDiagram
    autonumber
    actor UserA as "Користувач А (Онлайн)"
    participant API as "Backend API / Msg Service"
    participant DB as "Database"
    participant Queue as "Fan-out / Queue"
    actor UserB as "Користувач Б (Офлайн)"
    actor UserC as "Користувач В (Онлайн)"

    UserA->>API: Відправка повідомлення в Групу Х
    Note over API: Перевірка, чи А є в цій групі
    API->>DB: Збереження тексту повідомлення
    API->>Queue: Ініціація Fan-out для учасників (Б і В)
    
    par Для Користувача В (Онлайн)
        Queue->>UserC: Доставка через WebSocket
        UserC-->>API: Підтвердження отримання (Ack)
        API->>DB: Оновлення статусу В -> Delivered
    and Для Користувача Б (Офлайн)
        Queue->>DB: Збереження статусу Б -> Sent (в чергу)
        Note over Queue, UserB: Очікування, поки Б з'явиться в мережі
    end

stateDiagram-v2
    [*] --> Created : "Повідомлення створено"
    Created --> SentToBackend : "Надіслано на сервер"

    state "Статуси для конкретного отримувача" as PerUser {
        [*] --> Sent : "Збережено в базі/черзі"
        Sent --> Delivered : "Доставлено на пристрій"
        Delivered --> Read : "Прочитано користувачем"
    }

    SentToBackend --> PerUser
    Read --> [*]

📚 Part 4 — Architecture Decision Record (ADR)
# ADR-001: Use Fan-out on Write for Group Chat Architecture
Status
Accepted

Context
У системі месенджера реалізуються групові чати. Повідомлення, надіслане в групу, має миттєво доставлятися багатьом користувачам одночасно. При цьому, відповідно до вимог варіанта, статус доставки (Sent/Delivered/Read) повинен відстежуватися для кожного учасника групи індивідуально.

Decision
Ми приймаємо рішення використовувати підхід Fan-out on Write із залученням брокера повідомлень (Message Queue). Коли повідомлення надходить на бекенд, сервіс розгалуження (Fan-out) запитує список учасників групи та створює персональну задачу доставки в чергу кожного користувача.

Alternatives
Fan-out on Read: Клієнти самостійно опитують (polling) базу даних на наявність нових повідомлень у групі. (Відхилено через створення критичного навантаження на читання БД при великій кількості користувачів).

Consequences
+ Performance: Повідомлення з'являються у користувачів онлайн із мінімальною затримкою.

+ Ізоляція статусів: Зручно відстежувати індивідуальні статуси (Delivered/Read) у персональних чергах.

- Storage Overhead: Потребує більше оперативної пам'яті в момент відправки повідомлення у великі групи.
