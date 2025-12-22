# 14-03 - AsyncRpcSession

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Реалізувати `AsyncRpcSession`, що виконує один асинхронний цикл TCP RPC (resolve → connect → write → read_until) з таймаутом-константою 10 секунд, гарантією єдиного callback і мапінгом помилок у `RpcResponseStatus`.

# Requirements and DOD
- Конструктор приймає IOCtx, адресу Observer, `RpcRequest`, completion callback, Logger.
- `start()` ініціює ланцюг async_resolve → async_connect → async_write → async_read_until.
- Таймаут 10 с із класової константи (наприклад, `kObserverRpcTimeoutMs`); таймер скасовується після успіху.
- Callback викликається рівно один раз при успіху/таймауті/помилці; пізні події після таймауту не викликають повторний callback.
- Мапінг помилок: resolve/connect/write/read → NetworkError; JSON parse → ParseError; observer error → RpcError; таймаут → Timeout; успіх → Success; `errorMessage` містить деталі.
- Зберігає TransactionUUID у відповіді; додає TODO про потенційний ліміт одночасних сесій.
- Код компілюється без попереджень; покриття тестами буде в окремих задачах.

# Implementation Plan
- Додати `AsyncRpcSession.h/.cpp` у `src/core/network/rpc/`.
- Реалізувати конструктор, `start()`, приватні хендлери resolve/connect/write/read, таймер і функцію завершення.
- Забезпечити коректне скасування таймера та закриття сокета на помилці/таймауті.
- Формувати відповідний `RpcResponse` (у т.ч. RpcError/Timeout) із TransactionUUID і статусом.
- Додати TODO-коментар щодо майбутнього ліміту одночасних сесій.

# Test Plan
- Ручна збірка проєкту для перевірки компіляції.
- Юніт-тести на сценарії успіх/таймаут/NetworkError/ParseError/RpcError/late-response — у тестових задачах цього PRD.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Відповідає патернам Communicator (async, signals) і вимогам PRD.
- [ ] Чітке розділення транспортної логіки та формування RpcResponse.

## Security
- [ ] Коректне закриття сокета, без витоків ресурсів.

## Performance
- [ ] Асинхронний ланцюг без блокувань; таймаут не додає зайвого overhead.

## Scalability
- [ ] Структура дозволяє майбутній ліміт/бекпрешер (TODO зафіксовано).

## Reliability
- [ ] Single-callback гарантія; правильне завершення на всіх шляхах.

## Maintainability
- [ ] Код читається, обробка помилок централізована, логування інформативне.

## Cost
- [ ] Використано наявні залежності Boost.Asio/nlohmann_json.

## Compliance
- [ ] Відповідає PRD вимогам щодо таймауту і мапінгу статусів.

# Restrictions
- Commit changes тільки після успішної побудови (демо/тести для цієї задачі не передбачені). 
