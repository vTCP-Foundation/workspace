# 14-07 - RPC Message and Serialization Tests

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Реалізувати юніт-тести для базових типів RPC та типізованих повідомлень, включно з JSON серіалізацією/десеріалізацією за контрактами `observer/README.md`.

# Requirements and DOD
- Додати тести в `tests/unit/network/rpc/`:
  - `RpcMethodTest.cpp`, `RpcResponseStatusTest.cpp` (унікальні значення, наявність усіх enum пунктів).
  - `RpcRequestTest.cpp`, `RpcResponseTest.cpp` (конструктори, гетери, `isSuccess`).
  - Для кожного запиту/відповіді: конструктори, гетери, `method()` (GetBlockNumber, AcceptClaim, GetClaimStatus, SubmitClaimVotes, GetClaimStatuses).
  - Серіалізація/десеріалізація JSON для всіх методів згідно `observer/README.md` та PRD (включно зі станами approved/rejected, votes, messages).
- Оновити `tests/unit/CMakeLists.txt`, додавши всі нові тестові файли.
- Тести проходять локально; код компілюється без попереджень.

# Implementation Plan
- Створити тестові файли у `tests/unit/network/rpc/` за переліком PRD.
- Покрити enum-и, базові класи, запити/відповіді та їхні поля.
- Реалізувати серіалізаційні тести проти зразків JSON за `observer/README.md`.
- Додати файли до CMake списку юніт-тестів; побудувати та прогнати тести.

# Test Plan
- Виконати юніт-тести: `ctest` (або відповідна команда) після збірки.
- Переконатися, що всі тести проходять.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Тести розміщені в правильному каталозі і прив’язані до таргету unit_tests.

## Security
- [ ] Тести не розкривають секрети (ключі/підписи згенеровані тестово).

## Performance
- [ ] Тести швидкі, не потребують мережі/файлової системи.

## Scalability
- [ ] Легко додати нові тести при появі нових методів.

## Reliability
- [ ] Повне покриття полів/статусів/серіалізації забезпечує раннє виявлення регресій.

## Maintainability
- [ ] Тестові кейси читабельні, віддзеркалюють контракти PRD/observer README.

## Cost
- [ ] Використано існуючу інфраструктуру тестів.

## Compliance
- [ ] Відповідає переліку тестів у PRD.

# Restrictions
- Commit changes тільки після успішного проходження доданих юніт-тестів. 
