# Взаємодія під час оформлення оренди

Сценарії UC-02 з [завдання 2](../lab-2/requirements.md#3-сценарії-gherkin). Сховище працює із сутностями [ER-моделі](../lab-1/model.mmd).

## Сценарій 2. Бронювання вільного спальника

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    C->>U: confirmRental(SLP-01, Поділ, 2026-10-23, 2026-10-25)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: checkAvailability(itemIds, dates)
    activate A
    A->>S: loadAvailability(itemIds)
    activate S
    S-->>A: ready, відкритих позицій немає, daily_rate = 80.00
    deactivate S
    A-->>R: available(80.00)
    deactivate A
    opt Предмет доступний
        R->>S: createReservation(customerId, pointId, dates, lines)
        activate S
        Note over R,S: Rental.status = reserved<br/>RentalLine.daily_rate_at_booking = 80.00
        S-->>R: rentalId
        deactivate S
    end
    R-->>U: confirmed(rentalId)
    deactivate R
    U-->>C: showConfirmation(rentalId)
    deactivate U
```

## Сценарій 3. Неповернений намет після due_at

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    C->>U: confirmRental(TENT-01, 2026-10-26, 2026-10-28)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: checkAvailability(itemIds, dates)
    activate A
    A->>S: findOpenLines(TENT-01)
    activate S
    S-->>A: active, due_at минув, returned_at = null
    deactivate S
    opt Строк active минув, предмет не повернуто
        A->>A: markUnavailable(TENT-01)
    end
    A-->>R: unavailable(overdueRental)
    deactivate A
    R-->>U: rejected(overdueRental)
    deactivate R
    U-->>C: showUnavailable(overdueRental)
    deactivate U
```

## Сценарій 4. Предмет у ремонті

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    C->>U: confirmRental(TENT-02, 2026-10-23, 2026-10-25)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: checkAvailability(itemIds, dates)
    activate A
    A->>S: loadItemStatus(TENT-02)
    activate S
    S-->>A: maintenance
    deactivate S
    opt Статус відрізняється від ready
        A->>A: markUnavailable(TENT-02)
    end
    A-->>R: unavailable(notReady)
    deactivate A
    R-->>U: rejected(notReady)
    deactivate R
    U-->>C: showUnavailable(notReady)
    deactivate U
```

## Сценарій 9. Груднева бронь і жовтневі дати

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    C->>U: confirmRental(TENT-01, 2026-10-20, 2026-10-22)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: checkAvailability(itemIds, dates)
    activate A
    A->>S: loadAvailability(itemIds)
    activate S
    S-->>A: ready, reserved 2026-12-01..2026-12-05, daily_rate = 200.00
    deactivate S
    loop Відкриті позиції предмета
        A->>A: checkOverlap(requestedDates, reservedDates)
    end
    A-->>R: available(200.00)
    deactivate A
    R->>S: createReservation(customerId, pointId, dates, lines)
    activate S
    Note over R,S: Rental з жовтневими датами і статусом reserved
    S-->>R: rentalId
    deactivate S
    R-->>U: confirmed(rentalId)
    deactivate R
    U-->>C: showConfirmation(rentalId)
    deactivate U
```

## Сценарій 10. Неправильні дати

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    C->>U: confirmRental(SLP-01, starts_at, due_at)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: validateInterval(starts_at, due_at)
    activate A
    opt due_at не пізніше starts_at
        A->>A: setInvalidInterval()
    end
    A-->>R: invalidInterval
    deactivate A
    R-->>U: rejected(INVALID_INTERVAL)
    deactivate R
    U-->>C: showInvalidDates()
    deactivate U
```

## Сценарій 12. Один зайнятий предмет відхиляє весь запит

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    C->>U: confirmRental(TENT-01 і SLP-01, 2026-12-02, 2026-12-04)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: checkAvailability(itemIds, dates)
    activate A
    loop Кожен обраний предмет
        A->>S: loadAvailability(itemId)
        activate S
        S-->>A: стан, відкриті позиції
        deactivate S
        A->>A: evaluateAvailability(itemId, dates)
    end
    Note right of A: TENT-01 зайнятий<br/>SLP-01 вільний
    A-->>R: unavailable(overlappingRental)
    deactivate A
    R-->>U: rejected(overlappingRental)
    deactivate R
    U-->>C: showUnavailable(overlappingRental)
    deactivate U
```
