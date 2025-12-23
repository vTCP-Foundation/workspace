# 15-02 - CompletedPaymentsMonitoringDelayedTask

# Links
- [PRD](../../prd/vtcpd/15-completed-payments-observer-monitoring.md)

# Description
Create a new delayed task class `CompletedPaymentsMonitoringDelayedTask` that periodically triggers the monitoring of completed payment transactions on the Observer. This class uses `boost::asio::steady_timer` for scheduling and emits a signal that will be connected to `TransactionsManager` to launch the monitoring transaction.

The task starts 60 seconds after node startup and repeats every 300 seconds thereafter. These values are defined as class constants for easy configuration.

# Requirements and DOD

## Functional Requirements
1. Create new class `CompletedPaymentsMonitoringDelayedTask` in `src/core/delayed_tasks/`
2. Define class constant `kInitialDelaySeconds = 60` (first run delay after node start)
3. Define class constant `kMonitoringIntervalSeconds = 300` (interval between subsequent runs)
4. Constructor takes: `as::io_context&`, `Logger&`
5. Use `boost::asio::steady_timer` for scheduling (immune to system clock changes)
6. Define signal type: `signals::signal<void()>` named `monitoringSignal`
7. First timer expiration scheduled after `kInitialDelaySeconds`
8. After each expiration, reschedule timer for `kMonitoringIntervalSeconds`
9. Emit `monitoringSignal()` on each timer expiration
10. Implement proper logging for timer events

## Definition of Done
- [ ] Header file created: `CompletedPaymentsMonitoringDelayedTask.h`
- [ ] Implementation file created: `CompletedPaymentsMonitoringDelayedTask.cpp`
- [ ] CMakeLists.txt updated in `src/core/delayed_tasks/`
- [ ] Constants defined with correct values (60, 300)
- [ ] Timer scheduling works correctly
- [ ] Signal declared and emitted on timer expiration
- [ ] Code compiles without warnings
- [ ] Logging implemented for significant events

# Implementation Plan

## Step 1: Create header file
Create `src/core/delayed_tasks/CompletedPaymentsMonitoringDelayedTask.h`:

```cpp
#ifndef VTCPD_COMPLETEDPAYMENTSMONITORINGDELAYEDTASK_H
#define VTCPD_COMPLETEDPAYMENTSMONITORINGDELAYEDTASK_H

#include "../logger/Logger.h"

#include <boost/asio/steady_timer.hpp>
#include <boost/signals2.hpp>
#include <boost/asio.hpp>

using namespace std;
namespace as = boost::asio;
namespace signals = boost::signals2;

class CompletedPaymentsMonitoringDelayedTask
{
public:
    typedef signals::signal<void()> MonitoringSignal;

public:
    CompletedPaymentsMonitoringDelayedTask(
        as::io_context &ioCtx,
        Logger &logger);

public:
    mutable MonitoringSignal monitoringSignal;

private:
    void runMonitoring(
        const boost::system::error_code &error);

    LoggerStream info() const;
    LoggerStream warning() const;
    const string logHeader() const;

private:
    static const uint32_t kInitialDelaySeconds = 60;
    static const uint32_t kMonitoringIntervalSeconds = 300;

private:
    as::io_context &mIOCtx;
    unique_ptr<as::steady_timer> mMonitoringTimer;
    Logger &mLog;
};

#endif //VTCPD_COMPLETEDPAYMENTSMONITORINGDELAYEDTASK_H
```

## Step 2: Create implementation file
Create `src/core/delayed_tasks/CompletedPaymentsMonitoringDelayedTask.cpp`:

Reference `GatewayNotificationAndRoutingTablesDelayedTask.cpp` for implementation pattern:
- Constructor initializes timer and schedules first run
- `runMonitoring()` method:
  1. Log timer expiration
  2. Handle error code if present
  3. Cancel current timer
  4. Reschedule for next interval
  5. Emit signal

## Step 3: Update CMakeLists.txt
Add `CompletedPaymentsMonitoringDelayedTask.cpp` to `src/core/delayed_tasks/CMakeLists.txt`

## Step 4: Implement logging
- Log initial scheduling with delay value
- Log each monitoring cycle start
- Log any timer errors

# Test Plan

**Complexity**: Simple

Unit tests will be implemented in Task 06 (Unit Tests). This task focuses on implementation only.

Expected test coverage (to be implemented in Task 06):
- Constructor initializes timer correctly
- Constants have expected values (60, 300)
- Signal emitted on timer expiration (mock io_context)

# Verification and Validation

## Architecture integrity
- Follows existing delayed task patterns (reference: `TopologyEventDelayedTask`, `GatewayNotificationAndRoutingTablesDelayedTask`)
- Uses boost::signals2 for decoupled communication
- Single responsibility: only scheduling, no business logic

## Security
- No sensitive data handling
- Timer uses steady_clock (no time manipulation vulnerabilities)

## Performance
- Minimal resource usage: single timer, no polling
- No CPU usage between timer expirations
- Signal emission is O(1) to connected slots

## Scalability
- N/A - single instance per node

## Reliability
- Uses steady_timer (immune to system clock adjustments)
- Error handling for timer failures
- Automatic rescheduling ensures continuous operation

## Maintainability
- Constants defined at class level for easy modification
- Consistent with existing delayed task implementations
- Clear logging for debugging

## Cost
- N/A

## Compliance
- Follows project coding standards
- Consistent naming conventions

# Restrictions
- Do not implement transaction creation logic (done in TransactionsManager)
- Do not connect signal to slots (done in Core/TransactionsManager integration task)
- Commit changes only after code compiles without warnings
