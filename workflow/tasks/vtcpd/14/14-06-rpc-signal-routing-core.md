# 14-06 - RPC Signal Routing and Core Integration

# Links
- [PRD 14: Async Observer RPC Communication](../../../prd/vtcpd/14-async-observer-rpc-communication.md)

# Description
Підключити сигнальний ланцюг RPC: `TransactionsScheduler`, `TransactionsManager`, `Core`, `ObserverRpcCommunicator`. Забезпечити доставку RpcResponse транзакціям із обробкою overflow (синтетичний RpcError) та пробудженням у порядку надходження.

# Requirements and DOD
- `TransactionsScheduler`: реалізувати `tryAttachRpcResponseToTransaction` (пошук транзакції за UUID, перевірка очікуваного RpcMethod, виклик pushRpcResponse, overflow → RpcError і повернення транзакції, пробудження при `mustBeAwakenedOnRpcResponse`).
- `TransactionsManager`: додати `rpcRequestSignal`, `onRpcResponseReceived`, підписку на `outgoingRpcRequestSignal` транзакцій; форвардинг RPC запитів до Core.
- `Core`: створити `ObserverRpcCommunicator`, підключити сигнал `rpcRequestSignal` → `onRpcRequestSlot` → `sendRequest`, і `rpcResponseSignal` → `onRpcResponseSlot` → `TransactionsManager::onRpcResponseReceived`; метод `connectObserverRpcSignals`.
- Порядок пробудження визначається фактичним порядком прибуття відповідей в однопотоковому циклі.
- Код компілюється без попереджень.

# Implementation Plan
- Оновити `TransactionsScheduler` з новою логікою доставки/overflow RpcError.
- Оновити `TransactionsManager` для нового сигналу/слотів і реєстрації транзакційних сигналів.
- Оновити `Core` для створення `ObserverRpcCommunicator`, конфіг адреси, та підключення сигналів.
- Додати необхідні include/forward declarations, уникнути циклічних залежностей.
- Перевірити, що синхронний клієнт залишається, але не потребує фічефлагу.

# Test Plan
- Побудова проєкту.
- Юніт-тести маршрутизації/overflow/пробудження будуть у тестових задачах цього PRD.

# Verification and Validation
<Architect approval that validation criteria appropriate to task complexity have been met>

## Architecture integrity
- [ ] Сигнальний ланцюг відповідає схемі в PRD; немає нових глобальних залежностей.

## Security
- [ ] Без небезпечних операцій з пам’яттю; коректна життєдіяльність об’єктів сигналів.

## Performance
- [ ] Без блокувань; мінімальна обробка при доставці відповідей.

## Scalability
- [ ] Підтримує кілька RPC у черзі, порядок за надходженням.

## Reliability
- [ ] Overflow обробляється RpcError; пробудження транзакції гарантовано при відповідному методі.

## Maintainability
- [ ] Код узгоджений з існуючими сигналами/менеджерами; зрозуміле логування.

## Cost
- [ ] Без нових залежностей.

## Compliance
- [ ] Відповідає вимогам PRD для інтеграції ядра.

# Restrictions
- Commit changes тільки після успішної побудови (демо/тести для цієї задачі не передбачені). 
