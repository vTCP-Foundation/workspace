# 11-04 - OR-Tools Integration Update

# Links
- [PRD](../../prd/vtcpd/11-path-rebuilding-on-inaccessible-nodes.md)
- [Previous task 1](11-02-topology-sanitization.md)
- [Previous task 2](11-03-apply-existing-reservations.md)

# Description
Оновити частину `buildPathsAgain()` після очищення та застосування резервів: викликати `ExchangePathsManager::calculateMaxFlow()` із правильними ContractorID, додати нові шляхи до `mPathsStats`, забезпечити коректне журналювання. Відповідає пункту 5-7 алгоритму 4 PRD.

# Requirements and DOD

## Functional Requirements
1. Замінити частину коду, що викликає `calculateMaxFlow`, використовуючи `mContractorID` і `TopologyTrustLinesManager::kCurrentNodeID` як `senderID`.
2. Переконатися, що список еквівалентів передається згідно з PRD (`mCommand->equivalent()` + `mCommand->exchangeEquivalents()`).
3. Для кожного знайденого `OptimalPathResult` генерувати унікальний `PathID` через `generateNextPathID()` і додавати в `mPathsStats`.
4. Додати журналювання: кількість нових шляхів, їх `received_amount`, максимальний потік.
5. Обробити випадок, коли нових шляхів немає: логування `warning()` і повернення без змін.

## Definition of Done
- [ ] Виклик `calculateMaxFlow` оновлений та компілюється.
- [ ] Нові шляхи додаються з правильним журналюванням.
- [ ] Відсутність шляхів призводить до попередження, без винятків.
- [ ] Жодна інша частина транзакції не змінена.

# Implementation Plan
1. **Виклик OR-Tools**
   - Оновити аргументи виклику відповідно до PRD.
2. **Обробка результату**
   - Перевірити `optimalPaths` на порожність; додати журналювання `info`/`warning`.
   - Додати цикл, який викликає `generateNextPathID()` та додає `std::make_unique<OptimalPathResult>(path)`.
   - Логувати `received_amount` кожного шляху в debug.
3. **Error handling**
   - Забезпечити, що винятки OR-Tools (якщо виникають) обробляються існуючою логікою.
4. **Тестова збірка**
   - Переконатися, що код компілюється; запустити релевантні тести (див. task 11-07).

# Test Plan
- **Unit / Integration (завдання 11-07)**:
  - Мокнути `ExchangePathsManager` для повернення набору шляхів і перевірити, що вони додаються з правильними ID.
  - Перевірити випадок, коли `optimalPaths` порожній, — `mPathsStats` не змінюється, лог з попередженням.
- **Manual Smoke**: виконати сценарій, де є альтернативні шляхи; підтвердити в логах, що вони додані.

# Verification and Validation

## Architecture integrity
- Підтвердити відповідність вимогам PRD і відсутність побічних ефектів у інших стадіях.

## Security
- Переконатися, що логування не містить конфіденційних даних (використовувати лише ID та суми).

## Performance
- Визначити, що додані операції мають мінімальний оверхед (лише цикл додавання шляхів).

## Scalability
- Переконатися, що цикл обробляє велику кількість шляхів без зайвих копій.

## Reliability
- Зафіксувати, що у випадку відсутності шляхів код поводиться передбачувано (без винятків).

## Maintainability
- Код структурувати так, щоб легко було модифікувати логіку додавання шляхів.

## Cost
- Лише трудові витрати.

## Compliance
- Дотримуватися `policy.md`, особливо щодо логування та обробки помилок.

# Restrictions
- Не втручатися в інші стадії резервування.
- Комітити після успішних тестів/смоук-перевірок.
