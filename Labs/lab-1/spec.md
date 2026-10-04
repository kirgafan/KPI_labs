# Spec - Прокат туристичного спорядження

Модель прокату "Споряджай": один пункт видачі й повернення; без платежів і перевезень.

## Сутності та атрибути

PK - ключ запису; FK - посилання; UK - унікальне значення. UUID - формат id; string - текст; decimal - число; timestamp - дата й час.

| Сутність | Атрибути |
| --- | --- |
| Customer - клієнт | id (UUID, PK); full_name (string); email (string, UK); phone (string) |
| PickupPoint - пункт | id (UUID, PK); name (string); address (string) |
| EquipmentType - тип | id (UUID, PK); name (string); description (string); daily_rate (decimal) |
| EquipmentItem - окремий предмет | id (UUID, PK); equipment_type_id (UUID, FK); inventory_code (string, UK); status (string) |
| Tag - тег | id (UUID, PK); name (string, UK) |
| Rental - оренда | id (UUID, PK); customer_id (UUID, FK); pickup_point_id (UUID, FK); starts_at (timestamp); due_at (timestamp); status (string) |
| RentalLine - позиція оренди | id (UUID, PK); rental_id (UUID, FK); equipment_item_id (UUID, FK); daily_rate_at_booking (decimal); returned_at (timestamp, nullable) |

Rental.status: reserved, active, closed, cancelled. EquipmentItem.status: ready, maintenance, retired - стан експлуатації. Ціни - грн; лише returned_at може бути порожнім.

## Зв'язки словами

0..* означає жодного або кілька, 1..* - щонайменше один.

| Зв'язок | Перший об'єкт має | Другий об'єкт має |
| --- | --- | --- |
| R1 Customer - Rental | 0..* оренд | 1 клієнта |
| R2 PickupPoint - Rental | 0..* оренд | 1 пункт видачі |
| R3 EquipmentType - EquipmentItem | 0..* предметів | 1 тип |
| R4 Rental - RentalLine | 1..* позицій | 1 оренду |
| R5 EquipmentItem - RentalLine | 0..* позицій в історії | 1 предмет |
| R6 EquipmentType - Tag | 1..* тегів | 0..* типів |

## Критерії прийняття

- C1: у схемі є всі 7 сутностей і поля з таблиці; назви та типи збігаються.
- C2: усі PK і FK - UUID. RentalLine.rental_id має той самий тип, що Rental.id; integer не приймається.
- C3: зв'язки відповідають R1-R6. Тип без тегів не приймається; для EquipmentType - Tag немає посередника.
- C4: поля мають окремі значення, модель у 3NF. Email лише в Customer; погоджена ціна й повернення - у RentalLine.
- C5: email, inventory_code і Tag.name унікальні. Пара (rental_id, equipment_item_id) у RentalLine не повторюється. Ціни > 0; погоджена ціна не змінюється після зміни каталогу.
- C6: due_at > starts_at, returned_at >= starts_at. Чинні бронювання предмета не перетинаються; неповернений предмет недоступний навіть після due_at. closed - після всіх повернень; maintenance/retired не видаються.
- C7: Mermaid erDiagram у model.mmd; diagram.png створено з нього. Є промпти, аудит і ADR; без SQL DDL/ORM.
