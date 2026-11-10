# Взаємодія під час оформлення оренди

UC-02 «Оформити оренду» включає UC-03 «Перевірити доступність предмета». Номери і назви нижче взято з [Gherkin](../lab-2/requirements.md#3-сценарії-gherkin) та [матриці](../lab-2/requirements.md#4-матриця-трасування).

Форма передає `customerId` з автентифікованого контексту. Сервіс оренди керує запитом, а сервіс доступності виконує UC-03 у два кроки: `validateInterval` та `checkItems` для валідних дат. Сховище на початку транзакції блокує рядки `EquipmentItem` за порядком ID, а перевірка читає актуальні дані після блокування. Активація сховища позначає виконання окремого виклику. Транзакція `tx` і блокування тривають до `commit` або `rollback`.

Відкрита позиція має `returned_at = null`. Доступність залежить від `EquipmentItem.status` і відкритих позицій `reserved` та `active`. Інтервали мають вигляд `[starts_at, due_at)`: спільна межа не створює перетину. Прострочена відкрита `active` займає предмет навіть за межами свого інтервалу.

## Сценарій 2. Бронювання вільного спальника записує погоджену ціну

REQ-03. Бронь Олени Коваль у пункті «Поділ» отримує статус `reserved`. У позицію `SLP-01` копіюється ціна 80.00.

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    Note over C,U: Олена Коваль, SLP-01, Поділ<br/>2026-10-23 до 2026-10-25
    C->>U: confirmRental(itemCodes, pointId, dates)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: validateInterval(starts_at, due_at)
    activate A
    A-->>R: validInterval
    deactivate A
    R->>S: beginBooking(itemIds)
    activate S
    S-->>R: tx, предмети заблоковано
    deactivate S
    R->>A: checkItems(tx, itemIds, dates)
    activate A
    A->>S: loadAvailability(tx, itemIds)
    activate S
    S-->>A: ready, відкритих позицій немає<br/>EquipmentType.daily_rate = 80.00
    deactivate S
    A->>A: evaluateItem(SLP-01, dates)
    A-->>R: available(SLP-01, 80.00)
    deactivate A
    R->>S: createReservation(tx, customerId, pointId, dates, itemsWithRates)
    activate S
    S->>S: tryInsertRental(customer_id, pickup_point_id,<br/>starts_at, due_at, status = reserved)
    opt Rental отримав id
        S->>S: tryInsertLines(rental_id, equipment_item_id,<br/>daily_rate_at_booking, returned_at = null)
    end
    alt Rental і всі позиції записано
        S->>S: commit(tx)
    else Збій запису до commit
        S->>S: rollback(tx)
    end
    S-->>R: confirmed(rentalId, reserved) або BOOKING_FAILED
    deactivate S
    R-->>U: bookingResult
    deactivate R
    U-->>C: showBookingResult(bookingResult)
    deactivate U
```

## Сценарій 3. Неповернений намет не бронюють після due_at

REQ-02. Прострочена відкрита позиція `TENT-01` блокує навіть запит на дати після її `due_at`.

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    Note over C,U: TENT-01<br/>2026-10-26 до 2026-10-28
    C->>U: confirmRental(itemCodes, pointId, dates)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: validateInterval(starts_at, due_at)
    activate A
    A-->>R: validInterval
    deactivate A
    R->>S: beginBooking(itemIds)
    activate S
    S-->>R: tx, предмети заблоковано
    deactivate S
    R->>A: checkItems(tx, itemIds, dates)
    activate A
    A->>S: loadAvailability(tx, itemIds)
    activate S
    S-->>A: відкрита позиція active<br/>due_at минув, returned_at = null
    deactivate S
    opt active, due_at минув і returned_at порожній
        A->>A: markUnavailable(TENT-01, overdueRental)
    end
    A-->>R: unavailable(overdueRental)
    deactivate A
    R->>S: rollback(tx)
    activate S
    S-->>R: rolledBack
    deactivate S
    R-->>U: rejected(ITEM_UNAVAILABLE)
    deactivate R
    U-->>C: showUnavailable(ITEM_UNAVAILABLE)
    deactivate U
```

## Сценарій 4. Предмет у ремонті не додають до оренди

REQ-02. `TENT-02` недоступний через `EquipmentItem.status = maintenance`. Записів броні немає.

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    Note over C,U: TENT-02<br/>2026-10-23 до 2026-10-25
    C->>U: confirmRental(itemCodes, pointId, dates)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: validateInterval(starts_at, due_at)
    activate A
    A-->>R: validInterval
    deactivate A
    R->>S: beginBooking(itemIds)
    activate S
    S-->>R: tx, предмети заблоковано
    deactivate S
    R->>A: checkItems(tx, itemIds, dates)
    activate A
    A->>S: loadAvailability(tx, itemIds)
    activate S
    S-->>A: EquipmentItem.status = maintenance
    deactivate S
    opt status відрізняється від ready
        A->>A: markUnavailable(TENT-02, notReady)
    end
    A-->>R: unavailable(notReady)
    deactivate A
    R->>S: rollback(tx)
    activate S
    S-->>R: rolledBack
    deactivate S
    R-->>U: rejected(ITEM_UNAVAILABLE)
    deactivate R
    U-->>C: showUnavailable(ITEM_UNAVAILABLE)
    deactivate U
```

## Сценарій 9. Бронь на грудень не заважає жовтню

REQ-03. Груднева бронь `TENT-01` лишається чинною. Нова `reserved` має `starts_at = 2026-10-20` і `due_at = 2026-10-22`.

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    Note over C,U: TENT-01<br/>2026-10-20 до 2026-10-22
    C->>U: confirmRental(itemCodes, pointId, dates)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: validateInterval(starts_at, due_at)
    activate A
    A-->>R: validInterval
    deactivate A
    R->>S: beginBooking(itemIds)
    activate S
    S-->>R: tx, предмети заблоковано
    deactivate S
    R->>A: checkItems(tx, itemIds, dates)
    activate A
    A->>S: loadAvailability(tx, itemIds)
    activate S
    S-->>A: ready, відкрита позиція reserved<br/>2026-12-01 до 2026-12-05, daily_rate = 200.00
    deactivate S
    loop Кожна відкрита позиція TENT-01
        A->>A: checkOverlapAndOverdue(line, dates)
        Note right of A: Напіввідкриті інтервали не перетинаються<br/>Позиція reserved, прострочення active відсутнє
    end
    A-->>R: available(TENT-01, 200.00)
    deactivate A
    R->>S: createReservation(tx, customerId, pointId, dates, itemsWithRates)
    activate S
    S->>S: tryInsertRental(customer_id, pickup_point_id,<br/>starts_at, due_at, status = reserved)
    opt Rental отримав id
        S->>S: tryInsertLines(rental_id, equipment_item_id,<br/>daily_rate_at_booking, returned_at = null)
    end
    alt Rental і всі позиції записано
        S->>S: commit(tx)
    else Збій запису до commit
        S->>S: rollback(tx)
    end
    S-->>R: confirmed(rentalId, reserved) або BOOKING_FAILED
    deactivate S
    R-->>U: bookingResult
    deactivate R
    U-->>C: showBookingResult(bookingResult)
    deactivate U
```

## Сценарій 10. Оренду з due_at не пізніше starts_at відхиляють

REQ-14. Один сценарій Gherkin містить два незалежні запити. Кожен завершується на перевірці інтервалу.

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    loop Дві незалежні спроби зі сценарію 10
        Note over C,U: SLP-01, спроба 1: 2026-10-25 до 2026-10-23<br/>SLP-01, спроба 2: 2026-10-25 до 2026-10-25
        C->>U: confirmRental(itemCodes, pointId, dates)
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
    end
```

## Сценарій 12. Один зайнятий предмет відхиляє всю оренду

REQ-02. Перевірки `TENT-01` і `SLP-01` можуть виконуватися незалежно на вже прочитаних даних. Сервіс приймає рішення після обох перевірок. Для вільного `SLP-01` позиція теж не створюється.

```mermaid
sequenceDiagram
    actor C as Customer
    participant U as Форма оренди
    participant R as RentalService
    participant A as AvailabilityService
    participant S as Сховище
    Note over C,U: TENT-01 і SLP-01<br/>2026-12-02 до 2026-12-04
    C->>U: confirmRental(itemCodes, pointId, dates)
    activate U
    U->>R: submitRental(customerId, pointId, itemIds, dates)
    activate R
    R->>A: validateInterval(starts_at, due_at)
    activate A
    A-->>R: validInterval
    deactivate A
    R->>S: beginBooking(itemIds)
    activate S
    S-->>R: tx, предмети заблоковано
    deactivate S
    R->>A: checkItems(tx, itemIds, dates)
    activate A
    A->>S: loadAvailability(tx, itemIds)
    activate S
    S-->>A: обидва ready, відкрита позиція TENT-01<br/>reserved з 2026-12-01 до 2026-12-05
    deactivate S
    par Оцінка TENT-01
        A->>A: evaluateItem(TENT-01, dates)
        Note right of A: Інтервали перетинаються
    and Оцінка SLP-01
        A->>A: evaluateItem(SLP-01, dates)
        Note right of A: Предмет вільний
    end
    A-->>R: unavailable(TENT-01, overlappingRental)
    deactivate A
    opt Хоча б один предмет недоступний
        R->>S: rollback(tx)
        activate S
        S-->>R: rolledBack
        deactivate S
    end
    R-->>U: rejected(ITEM_UNAVAILABLE)
    deactivate R
    U-->>C: showUnavailable(ITEM_UNAVAILABLE)
    deactivate U
```

## Звірка з ER-моделлю

| Зв'язок | Як видно у взаємодії |
| --- | --- |
| R1, R2 | `Rental.customer_id` і `pickup_point_id` посилаються на вибраного клієнта та пункт. |
| R3 | `EquipmentItem.equipment_type_id` визначає, з якого `EquipmentType` взяти `daily_rate`. |
| R4, R5 | `RentalLine.rental_id` вказує на нову оренду, `equipment_item_id` - на обраний предмет. Записується одна позиція на предмет. У ній фіксується `daily_rate_at_booking` і порожній `returned_at`. |
| R6 | Зв'язок `EquipmentType` з `Tag` використовується в каталозі й не бере участі в підтвердженні броні. |

Сховище працює із сутностями з [ER-моделі завдання 1](../lab-1/model.mmd). Форма і сервіси є учасниками взаємодії, а `tx` і коди відповіді існують під час виклику. Незмінність погодженої ціни після зміни каталогу (REQ-09) перевіряє сценарій 7 завдання 2 у UC-08.

## Нефункціональні вимоги

| Критерій | Де видно в моделі | Як перевірити після реалізації |
| --- | --- | --- |
| NFR-01 | `beginBooking` перед читанням, спільний `tx` до запису або відмови. Сценарії 2, 3, 4, 9, 12. | Одночасно підтвердити два запити різних клієнтів на той самий вільний предмет на дати з перетином. Очікувати одну нову бронь з однією позицією та одну відповідь `ITEM_UNAVAILABLE`. |
| NFR-02 | `commit` після всіх позицій, `rollback` при збої запису. Сценарії 2, 9. | Спричинити збій запису другої позиції після запису першої. Очікувати `BOOKING_FAILED` і 0 нових `Rental` та `RentalLine` для спроби. |
| NFR-03 | `overdueRental`, `notReady` та `overlappingRental` лишаються між сервісами. Сценарії 3, 4, 12. | Перевірити відповідь форми для трьох причин. Очікувати `ITEM_UNAVAILABLE` у всіх випадках і 0 полів іншого `Rental` та `Customer`. |
| NFR-04 | Від виклику `submitRental` до відповіді RentalService формі, включно з очікуванням блокування. Сценарії 2, 3, 4, 9, 12. | Виміряти p95 за навантаженням зі [специфікації](spec.md#нефункціональні-вимоги). Ціль - ≤ 500 мс. |
