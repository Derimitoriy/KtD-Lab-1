# Laboratory Work 1: Designing a Messaging System
**Variant:** Variant 4 — Group Chat (Focus: scaling delivery logic)

---

## 🧱 Part 1 — Component Diagram

Component Diagram відображає основні елементи системи групового чату та забезпечує логіку розгалуження повідомлень (Fan-out) для багатьох отримувачів.

```mermaid
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
    Note over API: Перевірка, чи А є в цій групі
    API->>DB: Збереження тексту повідомлення (Msg_ID)
    API->>Queue: Ініціація Fan-out для учасників (Б і В)
    
    par Для Користувача В (Онлайн)
        Queue->>UserC: Доставка через WebSocket-з'єднання
        UserC-->>API: Підтвердження отримання (Ack)
        API->>DB: Оновлення статусу для Учасника В -> Delivered
    and Для Користувача Б (Офлайн)
        Queue->>DB: Збереження статусу для Учасника Б -> Sent (в чергу)
        Note over Queue, UserB: Очікування, поки Б з'явиться в мережі
    end

    stateDiagram-v2
    [*] --> Created : Повідомлення створено клієнтом
    Created --> SentToBackend : Надіслано на сервер

    state "Статуси для конкретного отримувача" as PerUser {
        [*] --> Sent : Збережено в базі / черзі доставки
        Sent --> Delivered : Доставлено на пристрій юзера (WebSocket/Push)
        Delivered --> Read : Прочитано користувачем (відкрито вікно чату)
    }

    SentToBackend --> PerUser : Ініціалізація статусів для кожного учасника групи
    Read --> [*] : Кінцевий стан життєвого циклу для цього отримувача
