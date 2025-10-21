# 07-02 - Create ExchangePathsResource and Signal Infrastructure

# Links
- [PRD](../../../prd/vtcpd/07-exchange-payment-topology-collection.md)
- Previous tasks: None (foundation task)

# Description
Create the resource and signal infrastructure needed for exchange payment path collection. This task implements:
1. `ExchangePathsResource` class - a resource object that signals completion of exchange path topology collection
2. `RequestExchangePathsResourceSignal` - a Boost.Signals2 signal for requesting exchange path collection
3. `ResourcesManager::requestExchangePaths()` method - a helper to emit the signal

This infrastructure enables asynchronous communication between `CoordinatorExchangePaymentTransaction` (requester) and `FindPathsByMaxFlowExchangeTransaction` (path collector), following the established resource-based communication pattern used for single-equivalent payments.

# Requirements and DOD

## Functional Requirements

### 1. ExchangePathsResource Class
- **Location**: `src/core/resources/resources/ExchangePathsResource.h` (new file)
- **Inheritance**: Extends `BaseResource`
- **Purpose**: Lightweight resource object carrying transaction UUID to route path collection completion notifications
- **Fields**:
  - `TransactionUUID mTransactionUUID` - UUID of requesting coordinator transaction
- **Methods**:
  - Constructor: `ExchangePathsResource(const TransactionUUID &transactionUUID)`
  - Getter: `const TransactionUUID& transactionUUID() const`
  - Override: `const byte_t resourceType() const` returns `BaseResource::ExchangePaths`

### 2. RequestExchangePathsResourceSignal
- **Location**: `src/core/resources/manager/ResourcesManager.h`
- **Type**: `boost::signals2::signal`
- **Signature**:
  ```cpp
  typedef signals::signal<void(
      const TransactionUUID&,                    // Requesting transaction UUID
      BaseAddress::Shared,                       // Contractor address
      const vector<SerializedEquivalent>&,       // Exchange equivalents (sender)
      const SerializedEquivalent)>               // Receiver equivalent
  RequestExchangePathsResourceSignal;
  ```
- **Visibility**: Public member of `ResourcesManager`
- **Naming**: `requestExchangePathsResourceSignal` (mutable, follows existing pattern)

### 3. ResourcesManager::requestExchangePaths() Method
- **Location**: `src/core/resources/manager/ResourcesManager.h/.cpp`
- **Purpose**: Helper method to emit `requestExchangePathsResourceSignal`
- **Signature**:
  ```cpp
  void requestExchangePaths(
      const TransactionUUID &transactionUUID,
      BaseAddress::Shared contractorAddress,
      const vector<SerializedEquivalent> &exchangeEquivalents,
      const SerializedEquivalent receiverEquivalent) const;
  ```
- **Implementation**: Simple signal emission with parameters

## Definition of Done
- [ ] `ExchangePathsResource.h` created and compiles
- [ ] `ExchangePathsResource` correctly inherits from `BaseResource`
- [ ] `ExchangePathsResource` resource type enum added to `BaseResource` (if needed)
- [ ] Constructor and getter implemented correctly
- [ ] `RequestExchangePathsResourceSignal` typedef added to `ResourcesManager.h`
- [ ] Signal instance `requestExchangePathsResourceSignal` declared in `ResourcesManager`
- [ ] `requestExchangePaths()` method declared in `ResourcesManager.h`
- [ ] `requestExchangePaths()` method implemented in `ResourcesManager.cpp`
- [ ] All files compile without errors or warnings
- [ ] Code follows existing patterns (`PathsResource`, `RequestPathsResourcesSignal`)
- [ ] No memory leaks (smart pointers used correctly)

# Implementation Plan

## Step 1: Check BaseResource for ExchangePaths Type
**Files**: `src/core/resources/resources/BaseResource.h`

1. Check if `BaseResource::ResourceType` enum contains `ExchangePaths` entry
2. If not, add it:
   ```cpp
   enum ResourceType : byte_t {
       Paths = 1,
       ObservingBlockNumber = 2,
       ExchangePaths = 3,  // Add this
       // ...
   };
   ```
3. This may already exist from PRD 05 work - verify first

## Step 2: Create ExchangePathsResource Class
**File**: `src/core/resources/resources/ExchangePathsResource.h` (new)

```cpp
#ifndef VTCPD_EXCHANGEPATHSRESOURCE_H
#define VTCPD_EXCHANGEPATHSRESOURCE_H

#include "BaseResource.h"
#include "../../transactions/transactions/base/TransactionUUID.h"

class ExchangePathsResource : public BaseResource {
public:
    static const byte_t kResourceType = ExchangePaths;

    ExchangePathsResource(const TransactionUUID &transactionUUID);

    const TransactionUUID& transactionUUID() const;

    const byte_t resourceType() const override {
        return kResourceType;
    }

private:
    TransactionUUID mTransactionUUID;
};

#endif // VTCPD_EXCHANGEPATHSRESOURCE_H
```

**Implementation**: `src/core/resources/resources/ExchangePathsResource.cpp` (new)

```cpp
#include "ExchangePathsResource.h"

ExchangePathsResource::ExchangePathsResource(
    const TransactionUUID &transactionUUID) :
    mTransactionUUID(transactionUUID)
{}

const TransactionUUID& ExchangePathsResource::transactionUUID() const {
    return mTransactionUUID;
}
```

**Key points**:
- Minimal implementation (no paths stored, paths already in `ExchangePathsManager` cache)
- Follows exact pattern of `PathsResource`
- UUID used for routing to correct coordinator transaction

## Step 3: Add RequestExchangePathsResourceSignal to ResourcesManager
**File**: `src/core/resources/manager/ResourcesManager.h`

1. **Add signal typedef** (around line 25, near `RequestPathsResourcesSignal`):
   ```cpp
   typedef signals::signal<void(
       const TransactionUUID&,
       BaseAddress::Shared,
       const vector<SerializedEquivalent>&,
       const SerializedEquivalent)>
   RequestExchangePathsResourceSignal;
   ```

2. **Add signal instance** (around line 42, in public section):
   ```cpp
   mutable RequestExchangePathsResourceSignal requestExchangePathsResourceSignal;
   ```

3. **Add requestExchangePaths() declaration** (around line 35, near `requestPaths()`):
   ```cpp
   void requestExchangePaths(
       const TransactionUUID &transactionUUID,
       BaseAddress::Shared contractorAddress,
       const vector<SerializedEquivalent> &exchangeEquivalents,
       const SerializedEquivalent receiverEquivalent) const;
   ```

## Step 4: Implement requestExchangePaths() Method
**File**: `src/core/resources/manager/ResourcesManager.cpp`

Add implementation (pattern identical to `requestPaths()`):

```cpp
void ResourcesManager::requestExchangePaths(
    const TransactionUUID &transactionUUID,
    BaseAddress::Shared contractorAddress,
    const vector<SerializedEquivalent> &exchangeEquivalents,
    const SerializedEquivalent receiverEquivalent) const
{
    requestExchangePathsResourceSignal(
        transactionUUID,
        contractorAddress,
        exchangeEquivalents,
        receiverEquivalent);
}
```

**Key points**:
- Simple signal emission wrapper
- No validation needed (handled by signal subscribers)
- `const` method (signal is mutable)

## Step 5: Add Include Directives
**File**: `src/core/resources/manager/ResourcesManager.h`

Ensure includes at top of file:
```cpp
#include <vector>
// ... existing includes ...
```

(May already be present, verify)

## Step 6: Verify Compilation
```bash
cd build-tests
cmake ..
make ResourcesManager ExchangePathsResource -j4
```

Check for:
- No compilation errors
- No warnings
- Clean build

## Expected Files Created/Modified
**Created**:
- `src/core/resources/resources/ExchangePathsResource.h`
- `src/core/resources/resources/ExchangePathsResource.cpp`

**Modified**:
- `src/core/resources/resources/BaseResource.h` (if `ExchangePaths` enum not present)
- `src/core/resources/manager/ResourcesManager.h`
- `src/core/resources/manager/ResourcesManager.cpp`

# Test Plan

## Test Approach
Unit tests will be created in separate testing task (Task 07-08). This task focuses solely on implementation.

## Demo Requirements (Before Commit)
Create a simple demonstration showing resource and signal functionality:

1. **Demo script location**: `workspace/demos/07-2-resource-signal-demo.cpp` (temporary)

2. **Demo scenarios**:
   - **Scenario 1: Resource creation and UUID retrieval**
     ```cpp
     TransactionUUID testUUID = TransactionUUID::generate();
     auto resource = make_shared<ExchangePathsResource>(testUUID);
     assert(resource->transactionUUID() == testUUID);
     assert(resource->resourceType() == BaseResource::ExchangePaths);
     ```

   - **Scenario 2: Signal emission and subscription**
     ```cpp
     ResourcesManager resourcesMgr;

     // Subscribe to signal
     bool signalReceived = false;
     TransactionUUID receivedUUID;
     BaseAddress::Shared receivedAddress;
     vector<SerializedEquivalent> receivedEquivs;
     SerializedEquivalent receivedReceiverEquiv;

     resourcesMgr.requestExchangePathsResourceSignal.connect(
         [&](const TransactionUUID &uuid, BaseAddress::Shared addr,
             const vector<SerializedEquivalent> &equivs, SerializedEquivalent recvEq) {
             signalReceived = true;
             receivedUUID = uuid;
             receivedAddress = addr;
             receivedEquivs = equivs;
             receivedReceiverEquiv = recvEq;
         });

     // Emit signal
     TransactionUUID testUUID = TransactionUUID::generate();
     auto testAddress = make_shared<IPv4WithPortAddress>("127.0.0.1", 8080);
     vector<SerializedEquivalent> testEquivs = {1, 2, 3};
     SerializedEquivalent testRecvEq = 5;

     resourcesMgr.requestExchangePaths(testUUID, testAddress, testEquivs, testRecvEq);

     assert(signalReceived);
     assert(receivedUUID == testUUID);
     assert(receivedAddress == testAddress);
     assert(receivedEquivs == testEquivs);
     assert(receivedReceiverEquiv == testRecvEq);
     ```

   - **Scenario 3: Multiple subscribers**
     ```cpp
     ResourcesManager resourcesMgr;
     int subscriber1Called = 0;
     int subscriber2Called = 0;

     resourcesMgr.requestExchangePathsResourceSignal.connect([&](...) { subscriber1Called++; });
     resourcesMgr.requestExchangePathsResourceSignal.connect([&](...) { subscriber2Called++; });

     resourcesMgr.requestExchangePaths(...);

     assert(subscriber1Called == 1);
     assert(subscriber2Called == 1);
     ```

3. **Demo execution**:
   ```bash
   # Compile
   cd build-tests
   cmake ..
   make 07-2-demo

   # Run
   ./bin/07-2-demo
   ```

4. **Success criteria**:
   - All scenarios pass assertions
   - No segmentation faults
   - No memory leaks (verify with valgrind if available)
   - Output confirms signal emission and reception

## Testing Notes
- Full unit test suite in Task 07-08
- Demo must pass before committing code
- Demo code not committed to repository

# Verification and Validation

## Architecture integrity
- **Pattern consistency**: Follows existing `PathsResource` / `RequestPathsResourcesSignal` pattern exactly
- **Separation of concerns**: Resource for data, signal for communication, manager for orchestration
- **No architectural violations**: Pure extension, no modifications to core architecture
- **Encapsulation maintained**: Resource is opaque container for UUID

## Security
- **No security implications**: Infrastructure components, no authentication/authorization logic
- **UUID security**: Transaction UUID prevents resource hijacking (UUID collision probability negligible)
- **No data exposure**: Resource contains only routing information (UUID), no sensitive path data

## Performance
- **Signal overhead**: Boost.Signals2 call overhead ~100-500ns (negligible)
- **Resource allocation**: Single heap allocation per resource (minimal overhead)
- **No performance regression**: Infrastructure only, no hot path impact

## Scalability
- **Signal subscribers**: Boost.Signals2 supports multiple subscribers efficiently
- **Resource lifetime**: Short-lived (created on path collection completion, consumed immediately)
- **No scalability bottlenecks**: Infrastructure scales with existing resource pattern

## Reliability
- **Signal safety**: Boost.Signals2 handles disconnection and exception safety
- **Resource lifecycle**: Managed by shared_ptr, no manual memory management
- **No new failure modes**: Infrastructure mirrors existing reliable patterns

## Maintainability
- **Code clarity**: Simple, minimal implementation following established patterns
- **Consistency**: Identical structure to `PathsResource` / `RequestPathsResourcesSignal`
- **Documentation**: Class and method purposes clear from names and structure
- **Future extensibility**: Easy to add additional resource fields if needed

## Cost
- **Development cost**: ~2 hours implementation + 1 hour demo
- **Testing cost**: Covered in Task 07-08
- **Maintenance cost**: Minimal (stable infrastructure)

## Compliance
- **Policy compliance**: Task-driven development (PRD 07)
- **Coding standards**: Follows existing C++ resource patterns
- **No external dependencies**: Uses existing Boost.Signals2

# Restrictions
- Commit changes only after successfully executing the demo
- Do not implement tests in this task (tests are in Task 07-08)
- Do not modify existing resource classes
- Follow exact pattern of PathsResource/RequestPathsResourcesSignal
- Do not add logic beyond resource creation and signal emission
