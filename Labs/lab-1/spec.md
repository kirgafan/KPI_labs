# Spec - Прокат туристичного спорядження

Версія 1. Побудувати логічну ER-модель каталогу, бронювань і повернень спорядження у Mermaid. Межі - один пункт видачі на оренду, повернення в той самий пункт, без платежів і складських переміщень.

## Сутності та атрибути

| Сутність | Атрибути |
| --- | --- |
| Customer | id (UUID, PK); full_name (string); email (string, UK); phone (string) |
| PickupPoint | id (UUID, PK); name (string); address (string) |
| EquipmentType | id (UUID, PK); name (string); description (string); daily_rate (decimal) |
| EquipmentItem | id (UUID, PK); equipment_type_id (UUID, FK); inventory_code (string, UK); condition (string) |
| Tag | id (UUID, PK); name (string, UK) |
| Rental | id (UUID, PK); customer_id (UUID, FK); pickup_point_id (UUID, FK); starts_at (timestamp); due_at (timestamp); status (string) |
| RentalLine | id (UUID, PK); rental_id (UUID, FK); equipment_item_id (UUID, FK); daily_rate_at_booking (decimal); returned_at (timestamp, nullable) |

Статуси оренди: reserved, active, closed, cancelled. Стани предмета: ready, maintenance, retired. Лише returned_at може бути порожнім. Грошові ставки - у гривнях.

## Зв'язки словами

- R1 Customer - Rental: клієнт має 0..* оренд; кожна оренда належить одному клієнту. `Customer ||..o{ Rental`.
- R2 PickupPoint - Rental: пункт має 0..* оренд; кожна оренда має один пункт. `PickupPoint ||..o{ Rental`.
- R3 EquipmentType - EquipmentItem: тип має 0..* предметів; кожен предмет має один тип. `EquipmentType ||..o{ EquipmentItem`.
- R4 Rental - RentalLine: оренда містить 1..* позицій; кожна позиція належить одній оренді. `Rental ||..|{ RentalLine`.
- R5 EquipmentItem - RentalLine: предмет має 0..* позицій в історії; позиція містить один предмет. `EquipmentItem ||..o{ RentalLine`.
- R6 EquipmentType - Tag: тип має 1..* тегів; тег описує 0..* типів. Прямий M:N, без посередника.

## Критерії прийняття

- C1: є рівно 7 сутностей і наведені атрибути; назви збігаються зі spec.
- C2: кожна сутність має id як PK; усі PK і FK мають тип UUID.
- C3: усі 6 зв'язків відповідають обом кінцям кардинальності; для R6 немає сполучної сутності.
- C4: атрибути атомарні, модель відповідає 3NF; RentalLine має власні дані про ставку й повернення.
- C5: email, inventory_code і Tag.name унікальні; пара (rental_id, equipment_item_id) у RentalLine не повторюється. Ставки > 0; історична ставка не змінюється разом із каталогом.
- C6: due_at > starts_at, returned_at >= starts_at; бронювання предмета не перетинаються, неповернений активний предмет недоступний; closed лише після всіх повернень; maintenance/retired не видаються.
- C7: model.mmd містить erDiagram без SQL DDL/ORM; SVG і PNG створено з нього. Промпти, аудит і ADR збережено.
