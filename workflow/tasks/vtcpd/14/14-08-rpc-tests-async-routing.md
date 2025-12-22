# 14-08 - RPC Async Session and Routing Tests

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Реалізувати юніт-тести для асинхронної частини RPC: `AsyncRpcSession`, маршрутизації в `TransactionsScheduler`, черги `BaseTransaction`, включно зі сценріями таймауту, помилок та overflow.

# Requirements and DOD
- Додати тести у `tests/unit/network/rpc/`:
  - ~~`AsyncRpcSessionTest.cpp`: успіх (resolve/connect/write/read), таймаут (10 с константа), NetworkError (resolve/connect/write/read), ParseError (невалідний JSON), RpcError (observer error), пізня відповідь після таймауту не викликає повторний callback; таймер скасовано після успіху.~~ **ВИКЛЮЧЕНО**: тести async TCP є інтеграційними за природою (потребують mock-сервера або мокування boost::asio), не юніт-тестами.
  - `TransactionsSchedulerRpcRoutingTest.cpp`: доставка відповіді транзакції, пробудження за порядком прибуття, overflow → синтетичний RpcError, очікуваний RpcMethod перевіряється. *(Примітка: обхід помилки лінкування в основному коді через явне додавання `transactions__trustlines` і `features` в правильному порядку в tests/unit/CMakeLists.txt)*
  - `BaseTransactionRpcQueueTest.cpp`: ліміт 50, 51-й push повертає false і генерує RpcError, FIFO порядок, `hasRpcResponse` коректний.
- Оновити `tests/unit/CMakeLists.txt` додавши ці файли.
- Тести проходять локально; код компілюється без попереджень.

# Implementation Plan
- Реалізувати `BaseTransactionRpcQueueTest.cpp` з тестовою реалізацією транзакції для перевірки черги RPC.
- Реалізувати `TransactionsSchedulerRpcRoutingTest.cpp` з mock-транзакціями для перевірки маршрутизації відповідей.
- Переконатися, що overflow шляхи формують RpcError і пробуджують транзакцію.
- Додати тести до CMake; зібрати і виконати.
- **Примітка**: `AsyncRpcSessionTest.cpp` виключено - тести async TCP потребують інтеграційного тестування.

# Test Plan
- Виконати `ctest` після побудови таргету unit_tests.
- Переконатися у проходженні всіх нових тестів.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Тести віддзеркалюють схеми сигналів/черг із PRD.

## Security
- [ ] Використовуються тестові дані; немає витоків секретів.

## Performance
- [ ] Тести швидкі, не блокують event loop (імітації/моки).

## Scalability
- [ ] Легко додати нові сценарії при розширенні RPC.

## Reliability
- [ ] Перевіряють критичні сценарії: таймаут, помилки, overflow, порядок пробудження.

## Maintainability
- [ ] Тести структуровані, зрозумілі, без прихованих залежностей.

## Cost
- [ ] Без додаткових бібліотек; використовують існуючу тестову інфраструктуру.

## Compliance
- [ ] Відповідає тестовим вимогам PRD для асинхронних компонентів.

# Restrictions
- Commit changes тільки після успішного проходження доданих юніт-тестів. 
