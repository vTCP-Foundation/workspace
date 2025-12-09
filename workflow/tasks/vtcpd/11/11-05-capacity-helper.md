# 11-05 - Capacity Helper

# Links
- [PRD](../../prd/vtcpd/11-path-rebuilding-on-inaccessible-nodes.md)
- [Previous task 1](11-04-ortools-integration-update.md)

# Description
Реалізувати та інтегрувати метод `calculateTotalPathCapacityForReceive()` у `CoordinatorExchangePaymentTransaction`, який підраховує сумарну пропускну здатність нових шляхів після перебудови. Логіка відповідає алгоритму 5 PRD.

# Requirements and DOD

## Functional Requirements
1. Додати метод (константний або з відповідною допоміжною функцією) `calculateTotalPathCapacityForReceive()`.
2. Розраховувати `remainingNeeded = mAmount - calculateTotalReservedAmount()` із дотриманням const-коректності.
3. Ігнорувати шляхи, де `pathID <= mCurrentAmountReservingPathIdentifier`.
4. Пропускати шляхи з `!isValid()` або `isLastIntermediateNodeProcessed()`.
5. Сумувати `received_amount` кожного релевантного шляху та логувати внесок кожного шляху і загальну суму.

## Definition of Done
- [ ] Метод реалізований і доступний у `CoordinatorExchangePaymentTransaction`.
- [ ] Логування включає інформацію про пропущені та враховані шляхи.
- [ ] Код компілюється, виклики методу виконуються без побічних ефектів.
- [ ] Підготовлено юніт-тести (task 11-07), що покривають основні сценарії.

# Implementation Plan
1. **Сигнатура і розміщення**
   - Додати декларацію в `.h` та реалізацію в `.cpp` відповідно до стилю класу.
2. **Реалізація логіки**
   - Отримати `alreadyReserved` через існуючий метод.
   - Ітерувати `mPathsStats`, застосовуючи фільтри за ID, валідністю, станом вузлів.
   - Накопичувати суму `TrustLineAmount` без переповнення.
   - Логувати `debug()` для кожного внеску, `info()` для фінального значення.
3. **Інтеграція**
   - Уточнити місця виклику (task 11-06) та переконатися, що метод доступний.
4. **Перевірка**
   - Запустити релевантні тести після реалізації.

# Test Plan
- **Unit Tests (moderate, покриття у task 11-07)**:
  - Сценарій із шляхами з обох боків від межі `mCurrentAmountReservingPathIdentifier`.
  - Сценарій із невалідним шляхом.
  - Сценарій із шляхом, що вже оброблено.
  - Сценарій без доступних шляхів (повернення 0).
- **Manual Debug Log Review**: перевірити, що журнали відображають правильні суми.

# Verification and Validation

## Architecture integrity
- Підтвердити, що метод відповідає вимогам PRD і не змінює стан класу.

## Security
- Необхідності в додаткових перевірках безпеки немає; переконатися, що лог не містить приватних даних.

## Performance
- Переконатися, що ітерація по `mPathsStats` лінійна без зайвих копій.

## Scalability
- Оцінити поведінку при великій кількості шляхів; сума має залишатися точною.

## Reliability
- Метод повертає коректний результат навіть при відсутності шляхів.

## Maintainability
- Код зрозумілий, з коментарями (коли потрібно), легко розширити.

## Cost
- Жодних додаткових витрат.

## Compliance
- Дотримання вимог PRD та політик репозиторію.

# Restrictions
- Не змінювати логіку розрахунку `calculateTotalReservedAmount()`.
- Комітити після підтвердження тестами з task 11-07.
