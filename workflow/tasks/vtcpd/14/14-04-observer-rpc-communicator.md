# 14-04 - ObserverRpcCommunicator

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Створити `ObserverRpcCommunicator`, який інкапсулює відправку RpcRequest через `AsyncRpcSession`, керує адресою Observer, виконує JSON (де)серіалізацію за `observer/README.md` та емісує `rpcResponseSignal`.

# Requirements and DOD
- Конструктор приймає IOCtx, адресу Observer, Logger; зберігає адресу з геттерами/сеттерами.
- `sendRequest(RpcRequest::Shared)` створює `AsyncRpcSession` і запускає її.
- Внутрішньо виконує серіалізацію запиту в JSON і десеріалізацію відповіді згідно `observer/README.md`/PRD (усі методи).
- Емітує `rpcResponseSignal(RpcResponse::Shared)` при завершенні сесії; сигнал підходить для підписки Core.
- Мапить помилки transport/timeout/parse/RpcError у `RpcResponseStatus` з відповідним `errorMessage`.
- Код компілюється без попереджень.

# Implementation Plan
- Додати `ObserverRpcCommunicator.h/.cpp` у `src/core/network/rpc/`.
- Інкапсулювати логіку формування JSON з типізованих RpcRequest та парсинг JSON у відповідні RpcResponse.
- Під’єднати completion callback `AsyncRpcSession` до емісії `rpcResponseSignal`.
- Забезпечити setter/getter для адреси Observer (із валідацією форматів, якщо потрібно).
- Додати логування ключових подій (відправка, відповідь, помилки).

# Test Plan
- Побудова проєкту (CMake).
- Юніт-тести на серіалізацію/десеріалізацію та сигнал будуть у тестових задачах цього PRD.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Дотримується патерну Communicator та сигналів/слотів.

## Security
- [ ] Не логувати секрети (ключі/підписи); безпечна обробка помилок мережі.

## Performance
- [ ] Мінімальні копії при (де)серіалізації; не блокує event loop.

## Scalability
- [ ] Підтримує додавання нових RpcMethod без зміни існуючого коду.

## Reliability
- [ ] Коректне формування/розбір JSON та мапінг статусів/помилок.

## Maintainability
- [ ] Чітке відокремлення транспорту (AsyncRpcSession) і (де)серіалізації.

## Cost
- [ ] Використовує існуючі залежності; без нових бібліотек.

## Compliance
- [ ] Відповідає вимогам PRD та протоколу `observer/README.md`.

# Restrictions
- Commit changes тільки після успішної побудови (демо/тести для цієї задачі не передбачені). 
