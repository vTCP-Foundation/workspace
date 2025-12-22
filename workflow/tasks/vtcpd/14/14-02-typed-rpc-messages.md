# 14-02 - Typed RPC Messages

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Реалізувати типізовані RPC запити/відповіді для всіх методів Observer: GetBlockNumber, AcceptClaim, GetClaimStatus, SubmitClaimVotes, GetClaimStatuses, включно з полями, методами доступу та (де)серіалізацією JSON відповідно до протоколу з `observer/README.md`.

# Requirements and DOD
- Додати класи запитів у `src/core/network/rpc/requests/`: `GetBlockNumberRpcRequest`, `AcceptClaimRpcRequest`, `GetClaimStatusRpcRequest`, `SubmitClaimVotesRpcRequest`, `GetClaimStatusesRpcRequest`.
- Додати класи відповідей у `src/core/network/rpc/responses/`: `GetBlockNumberRpcResponse`, `AcceptClaimRpcResponse`, `GetClaimStatusRpcResponse`, `SubmitClaimVotesRpcResponse`, `GetClaimStatusesRpcResponse`.
- Кожен клас повертає відповідний `RpcMethod` і містить поля, перелічені в PRD (UUID-и, блок-номери, ключі, підписи, стани, повідомлення тощо).
- Реалізувати JSON серіалізацію/десеріалізацію згідно зі схемами `observer/README.md` та вимог PRD (масиви votes/participants, мапінг станів, поля success/message тощо).
- Забезпечити коректну роботу з типами ключів/підписів (sphincs) та PaymentNodeID.
- Код компілюється без попереджень; покриття тестами виконуватиметься в окремих задачах.

# Implementation Plan
- Додати потрібні заголовки і cpp файли для кожного запиту/відповіді у відповідні підкаталоги.
- Реалізувати конструктори, гетери та `method()` для кожного класу.
- Додати функції (де)серіалізації JSON для кожного запиту/відповіді, використовуючи nlohmann/json.
- Валідувати мапінг станів та полів за `observer/README.md` (votes, rejectionSignature, claims arrays).
- Перевірити інтеграцію з наявними типами (TransactionUUID, BlockNumber, PaymentNodeID, sphincs ключі/підписи).

# Test Plan
- Побудувати проєкт (CMake) і переконатися у відсутності помилок/попереджень.
- Юніт-тести на серіалізацію/десеріалізацію та базові гетери будуть покриті в тестових задачах цього PRD.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Файли розміщені у `requests/` та `responses/` згідно структури PRD.
- [ ] Використовують базові класи `RpcRequest`/`RpcResponse`.

## Security
- [ ] Коректне поводження з ключами/підписами без логування секретів.

## Performance
- [ ] Серіалізація/десеріалізація без зайвих копій (де це можливо).

## Scalability
- [ ] Можливість додавання нових методів без зміни існуючих класів.

## Reliability
- [ ] Гарантована коректність полів і мапінгів станів/помилок.

## Maintainability
- [ ] Чіткі інтерфейси і назви методів/полів, сумісні зі стилем проекту.

## Cost
- [ ] Використано існуючі залежності (nlohmann/json, sphincs типи), без нових бібліотек.

## Compliance
- [ ] Відповідає вимогам PRD і `observer/README.md`.

# Restrictions
- Commit changes тільки після успішної побудови (демо/тести для цієї задачі не передбачені). 
