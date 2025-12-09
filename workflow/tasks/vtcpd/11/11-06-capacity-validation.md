# 11-06 - Capacity Validation

# Links
- [PRD](../../prd/vtcpd/11-path-rebuilding-on-inaccessible-nodes.md)
- [Previous task 1](11-04-ortools-integration-update.md)
- [Previous task 2](11-05-capacity-helper.md)

# Description
Додати в `CoordinatorExchangePaymentTransaction::tryProcessNextPath()` перевірку, що сумарна пропускна здатність нових шляхів достатня для закриття залишкової суми. Реалізує алгоритм 6 PRD і гарантує коректне завершення транзакції при дефіциті.

# Requirements and DOD

## Functional Requirements
1. У блоці `catch (NotFoundError &e)` після успішного `buildPathsAgain()` викликати `calculateTotalPathCapacityForReceive()` та порівнювати результат із `remainingReceive = mAmount - calculateTotalReservedAmount()`.
2. При достатній місткості логувати `info` і переходити до `runAmountReservationStage()` (наявна логіка).
3. При дефіциті: логувати `warning`, викликати `reject("Rebuilt paths have insufficient capacity")` і повертати `resultInsufficientFundsError()`.
4. Забезпечити, що лічильники `mPreviousInaccessibleNodesCount`, `mPreviousRejectedTrustLinesCount` оновлюються тільки коли перевірка пройдена та транзакція продовжується.
5. Додати інформаційне логування про `remainingReceive` і `totalCapacity`.

## Definition of Done
- [ ] Код перевірки доданий та компілюється.
- [ ] Відповідні гілки логують очікувані повідомлення.
- [ ] Поведінка при дефіциті коректно завершує транзакцію.
- [ ] Жодних побічних змін у логіці резервування.

# Implementation Plan
1. **Додати обчислення**
   - Використати `calculateTotalReservedAmount()` і `calculateTotalPathCapacityForReceive()`.
2. **Логування**
   - Додати `debug` або `info` повідомлення про значення before/after.
3. **Оновлення лічильників**
   - Пересвідчитись, що оновлення `mPrevious*` відбувається лише після позитивного результату.
4. **Обробка дефіциту**
   - Виконати `reject` і повернення `resultInsufficientFundsError()`; переконатися, що інші колбеки (audit) не запускаються.
5. **Збірка й тестування**
   - Переконатися, що всі тест-кейси (task 11-07) та попередні перевірки проходять.

# Test Plan
- **Unit Tests (у межах task 11-07)**:
  - Сценарій достатньої місткості (перехід до наступної стадії).
  - Сценарій дефіциту (warning + resultInsufficientFundsError).
  - Сценарій точного збігу (місткість == залишок).
- **Manual Review**: Перевірити журнали, що значення залишку й сумарна місткість виводяться.

# Verification and Validation

## Architecture integrity
- Підтвердити, що логіка повністю відповідає PRD і не порушує попередні стадії.

## Security
- Немає впливу; логування містить лише числові значення.

## Performance
- Додаткові обчислення мінімальні (виклик уже наявних методів).

## Scalability
- Метод базується на попередньо підготовленій сумі й не додає нових ітерацій.

## Reliability
- Забезпечує детерміноване завершення транзакції при нестачі місткості.

## Maintainability
- Логіка сконцентрована в одному блоці `catch`, легко супроводжувати.

## Cost
- Тільки трудові витрати.

## Compliance
- Дотримання політик PRD і проекту щодо перевірок і журналювання.

# Restrictions
- Не змінювати інші гілки `tryProcessNextPath()`.
- Комітити лише після підтвердження тестами (task 11-07).
