# 3. Данные, интеграции и API

## 3.1. DFD (уровень 0)

```mermaid
flowchart LR
    B[/Покупатель/]:::ext
    P[/Сотрудник ПВЗ/]:::ext
    W[/Склад возвратов/]:::ext
    G[/Платёжный шлюз/]:::ext

    P1((1. Оформить<br/>заявку))
    P2((2. Принять<br/>товар в ПВЗ))
    P3((3. Вернуть<br/>деньги))
    P4((4. Проверить<br/>товар))

    D1[(D1 Заказы)]
    D2[(D2 Заявки на возврат)]
    D3[(D3 Выплаты)]

    B -- заявка, фото --> P1
    D1 -- данные заказа --> P1
    P1 -- заявка --> D2
    P1 -- QR-код, решение --> B
    P -- QR, результат осмотра --> P2
    D2 -- данные заявки --> P2
    P2 -- статус «принят» --> D2
    P2 -- квитанция --> B
    P2 -- распоряжение на выплату --> P3
    P3 -- запрос возврата --> G
    G -- подтверждение --> P3
    P3 -- выплата --> D3
    P3 -- уведомление --> B
    P2 -- товар, накладная --> P4
    W -- результат проверки --> P4
    P4 -- статус «проверен» --> D2

    classDef ext fill:#fff8ec,stroke:#333
```

## 3.2. Статусная модель заявки

```mermaid
stateDiagram-v2
    [*] --> Создана
    Создана --> Отклонена: не прошла автопроверку
    Создана --> Ожидает_сдачи: одобрена, выдан QR
    Ожидает_сдачи --> Отменена: покупатель отменил / QR истёк
    Ожидает_сдачи --> Принята_в_ПВЗ: осмотр пройден
    Ожидает_сдачи --> Отказ_ПВЗ: товар не соответствует
    Принята_в_ПВЗ --> Деньги_возвращены: выплата подтверждена
    Деньги_возвращены --> Закрыта: склад подтвердил
    Деньги_возвращены --> Спор: склад выявил следы использования
    Спор --> Закрыта
    Отклонена --> [*]
    Отменена --> [*]
    Отказ_ПВЗ --> [*]
    Закрыта --> [*]
```

## 3.3. Приёмка в ПВЗ и выплата

```mermaid
sequenceDiagram
    autonumber
    actor P as Сотрудник ПВЗ
    participant T as Терминал ПВЗ
    participant R as Сервис возвратов
    participant Pay as Платёжный шлюз
    participant N as Уведомления

    P->>T: сканирует QR
    T->>R: GET /returns/{id}
    R-->>T: товар, причина, фото
    P->>T: результат осмотра «принят»
    T->>R: POST /returns/{id}/acceptance
    R->>R: статус «Принята в ПВЗ», квитанция
    R->>Pay: POST /refunds (Idempotency-Key = return_id)
    Pay-->>R: 202 Accepted
    R-->>T: квитанция
    Pay-->>R: webhook refund.succeeded
    R->>R: статус «Деньги возвращены»
    R->>N: push покупателю
```

## 3.4. REST API (основные методы)

| Метод | Путь | Назначение | Кто вызывает |
|---|---|---|---|
| POST | `/returns` | создать заявку, получить решение | приложение покупателя |
| GET | `/returns/{id}` | данные и статус заявки | приложение, терминал ПВЗ |
| DELETE | `/returns/{id}` | отменить заявку до сдачи | приложение покупателя |
| POST | `/returns/{id}/acceptance` | результат осмотра в ПВЗ | терминал ПВЗ |
| POST | `/returns/{id}/inspection` | результат проверки на складе | система склада |
| GET | `/sellers/{id}/returns/report` | отчёт продавцу | кабинет продавца |

**Пример: `POST /returns`**

```jsonc
// запрос
{
  "order_id": 900123,
  "item_id": 55,
  "reason": "not_as_described",
  "comment": "Цвет отличается от фото",
  "photos": ["ph_81f2", "ph_81f3"]
}
// ответ 201
{
  "return_id": "RET-2026-000481",
  "status": "awaiting_dropoff",
  "qr_code": "https://.../qr/RET-2026-000481",
  "qr_expires_at": "2026-10-07T23:59:59+03:00"
}
// ответ 422 — отказ по бизнес-правилу
{ "error": "return_period_expired", "message": "Срок возврата истёк 21.09.2026" }
```

## 3.5. Логическая модель данных

```mermaid
erDiagram
    ORDER ||--|{ ORDER_ITEM : "содержит"
    ORDER_ITEM ||--o{ RETURN_REQUEST : "возвращается"
    RETURN_REQUEST ||--o{ RETURN_PHOTO : "имеет"
    RETURN_REQUEST ||--o| PVZ_ACCEPTANCE : "принята"
    RETURN_REQUEST ||--o| WAREHOUSE_INSPECTION : "проверена"
    RETURN_REQUEST ||--o| REFUND : "оплачена"
    RETURN_REQUEST ||--|{ STATUS_HISTORY : "история"
    PVZ ||--o{ PVZ_ACCEPTANCE : "выполняет"

    RETURN_REQUEST { string return_id PK
                     int order_item_id FK
                     string reason
                     string status
                     datetime created_at
                     datetime qr_expires_at }
    PVZ_ACCEPTANCE { string return_id PK
                     int pvz_id FK
                     int employee_id
                     string result
                     string reject_reason
                     datetime accepted_at }
    REFUND { string refund_id PK
             string return_id FK
             decimal amount
             string status
             datetime paid_at }
    WAREHOUSE_INSPECTION { string return_id PK
                           string result
                           string resolution }
```
