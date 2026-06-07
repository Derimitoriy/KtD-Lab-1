# Laboratory Work 1: Designing a Messaging System

**Variant:** Variant 4 — Group Chat (Focus: scaling delivery logic)

---

## 🧱 Part 1 — Component Diagram

Component Diagram відображає основні елементи системи групового чату та забезпечує логіку розгалуження повідомлень (Fan-out) для багатьох отримувачів.

```mermaid
graph TD
    Client[Client Web/Mobile] --> API[Backend API]
    API --> MsgService[Message Service]
    MsgService --> DB[(Database)]
    MsgService --> FanOut[Fan-out / Queue Service]
    FanOut --> Delivery[WebSocket / Push Service]
    Delivery --> ClientB[Recipient User B]
    Delivery --> ClientC[Recipient User C]

sequenceDiagram
    participant UserA as Користувач А (Онлайн)
    participant API as Backend API
    participant DB as Database
    participant Queue as Fan-out / Queue
    participant UserB as Користувач Б (Офлайн)
    participant UserC as Користувач В (Онлайн)

    UserA->>API: Відправка повідомлення в Групу
    API->>DB: Збереження тексту
    API->>Queue: Ініціація Fan-out
    
    Queue->>UserC: Доставка через WebSocket
    UserC-->>API: Підтвердження
    API->>DB: Статус В -> Delivered
    
    Queue->>DB: Статус Б -> Sent в чергу
    Note over Queue, UserB: Очікування мережі

stateDiagram-v2
    [*] --> Created : Повідомлення створено
    Created --> SentToBackend : Надіслано на сервер

    state PerUser {
        [*] --> Sent : В черзі
        Sent --> Delivered : Доставлено
        Delivered --> Read : Прочитано
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

Performance: Повідомлення з'являються у користувачів онлайн із мінімальною затримкою.

Ізоляція статусів: Зручно відстежувати індивідуальні статуси (Delivered/Read) у персональних чергах.

Storage Overhead: Потребує більше оперативної пам'яті в момент відправки повідомлення у великі групи.
