# 11-07 - Path Rebuild Tests

# Links
- [PRD](../../prd/vtcpd/11-path-rebuilding-on-inaccessible-nodes.md)
- [Previous task 1](11-01-topology-api-extensions.md)
- [Previous task 2](11-02-topology-sanitization.md)
- [Previous task 3](11-03-apply-existing-reservations.md)
- [Previous task 4](11-04-ortools-integration-update.md)
- [Previous task 5](11-05-capacity-helper.md)
- [Previous task 6](11-06-capacity-validation.md)

# Description
Створити та/або оновити unit-тести, що покривають нові можливості перебудови шляхів: API менеджера топології, очищення топології, застосування резервів, інтеграцію з OR-Tools, обчислення пропускної здатності та перевірку у `tryProcessNextPath()`. Цей таск реалізує тестову стратегію з PRD (розділ "Testing Strategy").

# Requirements and DOD

## Functional Requirements
1. Додати тести для `removeTrustLine` та `participantsIDs()` (Tasks 11-01).
2. Додати тести для очищення топології в `buildPathsAgain()` при недоступних нодах і відхилених лініях (Task 11-02).
3. Додати тести для застосування Incoming резервів, зокрема часткових та повних (Task 11-03).
4. Додати тести, що перевіряють зменшення доступної місткості після застосування резервів при повторному виклику OR-Tools (Task 11-04).
5. Додати тести для `calculateTotalPathCapacityForReceive()` з різними сценаріями (Task 11-05).
6. Додати тести для поведінки `tryProcessNextPath()` при достатній/недостатній місткості (Task 11-06).
7. Забезпечити, щоб нові тести вкладалися в існуючу структуру (`tests/unit/transactions/...` та/або `tests/unit/topology/...`).

## Definition of Done
- [ ] Написані та задокументовані всі тести зі списку вище.
- [ ] Усі тести проходять локально (`build-tests`).
- [ ] Тести додають покриття ключових сценаріїв (incoming-only резерви, альтернативні шляхи, дефіцит місткості).
- [ ] Немає флейковості: повторний запуск дає однаковий результат.

# Implementation Plan
1. **Topology Manager Tests**
   - Створити/оновити файл для тестів менеджера топології, перевірити `removeTrustLine` і `participantsIDs()`.
2. **CoordinatorExchangePaymentTransaction Tests**
   - Створити новий файл `CoordinatorExchangePaymentPathRebuildingTest.cpp` (якщо ще немає) або оновити існуючий.
   - Додати набір тестів, описаних у PRD (тести 1-19, узгоджені зі змінами).
3. **Mocking / Fixtures**
   - За потреби створити допоміжні фікстури для побудови топології й шляхів.
4. **Test Execution**
   - Запустити `build-tests` і відповідні бінарні юніт-тести; переконатися в зеленому статусі.
5. **Документація в коді тестів**
   - Додавати коментарі з посиланнями на конкретні розділи PRD для майбутньої підтримки.

# Test Plan
- **Automated**: усі нові тести виконуються через CI/локально (`build-tests`).
- **Manual**: у разі складних сценаріїв проаналізувати логи для підтвердження коректності.

# Verification and Validation

## Architecture integrity
- Перевірити, що покриття тестами відповідає архітектурним вимогам PRD.

## Security
- Переконатися, що тести не потребують реальних секретів/ключів.

## Performance
- Оцінити, чи не додають тести надмірного часу виконання; оптимізувати фікстури.

## Scalability
- Тести мають охоплювати сценарії з великою кількістю нод/ліній (відповідно до PRD цільових значень).

## Reliability
- Забезпечити детермінізм тестів, ізоляцію стану між кейсами.

## Maintainability
- Структура тестів має бути зрозумілою, з чітким групуванням сценаріїв.

## Cost
- Тільки час на розробку та виконання тестів.

## Compliance
- Відповідає вимогам розділу "Testing Strategy" PRD та політикам репозиторію.

# Restrictions
- Не змінювати продакшн-код у цьому таску.
- Вимагати зелене виконання `build-tests` перед комітом.
