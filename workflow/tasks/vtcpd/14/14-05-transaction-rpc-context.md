# 14-05 - Transaction RPC Context

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Розширити `TransactionState` та `BaseTransaction` для підтримки очікування async RPC: новий wait-state, сигнал відправки RPC, черга відповідей з лімітом 50 і синтетичною RpcError-відповіддю при overflow.

# Requirements and DOD
- `TransactionState`: додати `waitForRpcResponse(RpcMethod, timeout)`, `isWaitingForRpcResponse()`, `requiredRpcMethod()`, `mustBeAwakenedOnRpcResponse()`, підтримка кількох паралельних RPC (пробудження за порядком прибуття).
- `BaseTransaction`: додати `outgoingRpcRequestSignal`, `sendRpcRequest()`, чергу `mRpcContext` (FIFO) з константою ліміту 50 (`kMaxRpcResponsesPerTransaction`), `pushRpcResponse()/popRpcResponse()/hasRpcResponse()`, `resultWaitForRpcResponse()`.
- При overflow `pushRpcResponse()` повертає false і генерує RpcError-відповідь про переповнення, що негайно передається транзакції без додавання в чергу.
- Зберегти TransactionUUID у всіх операціях; підтримати паралельні RPC без гонок в однопотоковому циклі.
- Код компілюється без попереджень.

# Implementation Plan
- Оновити `TransactionState` для нового стану очікування RPC (поля/методи згідно PRD).
- У `BaseTransaction` додати сигнал, чергу з лімітом, методи push/pop/has, логіку overflow RpcError.
- Реалізувати `resultWaitForRpcResponse()` для повернення правильного `TransactionResult`.
- Додати константи ліміту та таймауту в відповідних класах (якщо потрібно).
- Переконатися в узгодженні з існуючими сигналами/чергами.

# Test Plan
- Побудова проєкту.
- Юніт-тести на чергу/overflow/паралельні RPC будуть у тестових задачах цього PRD.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Узгоджено з існуючою моделлю станів і сигналів транзакцій.

## Security
- [ ] Немає витоків секретів; логування без ключів/підписів.

## Performance
- [ ] Черга з лімітом 50; операції O(1); без блокувань.

## Scalability
- [ ] Підтримка кількох RPC без зміни API.

## Reliability
- [ ] Overflow обробляється синтетичною RpcError-відповіддю; пробудження транзакції гарантується.

## Maintainability
- [ ] Код/коментарі відповідають патернам існуючих станів/сигналів.

## Cost
- [ ] Без нових залежностей.

## Compliance
- [ ] Відповідає вимогам PRD щодо черги, ліміту й поведінки при overflow.

# Restrictions
- Commit changes тільки після успішної побудови (демо/тести для цієї задачі не передбачені). 
