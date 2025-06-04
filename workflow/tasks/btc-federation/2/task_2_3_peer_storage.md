# BTC-FED-2-3 - Peer Storage & File Management

# Links
- [PRD](../../prd/btc-federation/02_p2p_network_stack.md)

# Description
Implement robust YAML-based peer storage system with atomic file operations, concurrent access protection, and conflict resolution. This task creates the persistent data layer for peer information that can handle manual edits, concurrent access, and data integrity challenges.

# Requirements and DOD
- [ ] YAML-based peer storage matching PRD schema (`peers.yaml`)
- [ ] Atomic file operations with conflict detection and resolution
- [ ] Thread-safe concurrent access protection
- [ ] Schema validation and error recovery mechanisms
- [ ] Backup creation and restoration for corrupted files
- [ ] Manual edit detection and integration
- [ ] Duplicate peer filtering and validation
- [ ] File locking with timeout mechanisms
- [ ] Checksum validation for data integrity
- [ ] Emergency in-memory peer list fallback
- [ ] Unit tests covering all storage scenarios (>90% coverage)
- [ ] Stress tests for concurrent access patterns

# Implementation Plan

## Step 1: Storage Schema & Data Structures 
1. Define peer storage data structures:
   - Peer struct (public_key, addresses, metadata)
   - PeerList struct with validation methods
   - Storage metadata (version, checksum, last_modified)
2. Implement YAML marshaling/unmarshaling with proper tags
3. Create peer validation logic:
   - Public key format validation
   - Address format and reachability checks
   - Duplicate detection algorithms
4. Add schema versioning for future compatibility

## Step 2: Atomic File Operations 
1. Implement atomic write operations:
   - Write to temporary file first
   - Atomic rename/move operation
   - Backup creation before modification
   - Rollback capability on failure
2. Create file locking mechanisms:
   - Exclusive locks for writes
   - Shared locks for reads
   - Timeout handling and deadlock prevention
   - Cross-platform file locking
3. Add checksum validation:
   - CRC32/SHA256 checksums for integrity
   - Validation on read operations
   - Corruption detection and recovery

## Step 3: Concurrent Access Management 
1. Implement thread-safe storage manager:
   - Read-write mutex for in-memory cache
   - Synchronized file access patterns
   - Operation queuing for writes
   - Deadlock prevention strategies
2. Create conflict resolution system:
   - Three-way merge for concurrent modifications
   - Last-seen timestamp as tiebreaker
   - Manual edit detection and integration
   - Conflict reporting and logging
3. Add emergency fallback mechanisms:
   - In-memory peer list backup
   - Recovery from corrupted files
   - Partial data recovery strategies

## Step 4: Storage Operations & Interface 
1. Implement core storage operations:
   - AddPeer, RemovePeer, UpdatePeer
   - GetPeers, GetPeer, PeerExists
   - LoadPeers, SavePeers, ReloadPeers
   - Bulk operations for efficiency
2. Create storage manager interface:
   - Event-driven notifications
   - Callback system for peer changes
   - Integration hooks for external systems
   - Configuration-driven behavior
3. Add monitoring and metrics:
   - Operation performance tracking
   - Error rate monitoring
   - Storage statistics collection

## Step 5: Testing & Validation
1. Write comprehensive unit tests:
   - Basic CRUD operations
   - Concurrent access scenarios
   - File corruption handling
   - Validation and error cases
   - Performance and memory usage
2. Create stress tests:
   - Multiple goroutines accessing storage
   - Rapid file modification scenarios
   - Large peer list handling
   - Recovery from various failure modes
3. Add integration tests:
   - File system interaction
   - Cross-platform compatibility
   - Real-world usage patterns
4. Performance optimization and profiling

# Test Plan

## Unit Tests
- **CRUD Operations**: Add, remove, update peers
- **Validation**: Schema validation, duplicate detection, invalid data
- **File Operations**: Atomic writes, backup/restore, checksum validation
- **Concurrency**: Multiple readers, exclusive writers, deadlock prevention
- **Error Handling**: Corrupted files, missing files, permission errors

## Stress Tests
- **Concurrent Access**: Multiple goroutines reading/writing simultaneously
- **Large Datasets**: Performance with thousands of peers
- **Rapid Modifications**: High-frequency peer updates
- **Resource Limits**: Low disk space, permission restrictions
- **Recovery Scenarios**: Corruption during write, power failure simulation

## Integration Tests
- **File System Integration**: Real file operations
- **Manual Edit Detection**: External file modifications
- **Backup and Recovery**: Complete recovery scenarios
- **Configuration Integration**: Storage location and behavior settings

# Verification and Validation

## Architecture integrity
- Clean separation between storage logic and business logic
- Interface-based design for easy testing and mocking
- Event-driven architecture for storage notifications
- Proper abstraction of file system operations

## Security
- File permission management and validation
- Secure handling of temporary files
- Protection against path traversal attacks
- Atomic operations prevent data corruption

## Scalability
- Handles large peer lists (1000+ peers) efficiently
- Storage operations scale appropriately with data size
- Memory usage remains bounded regardless of peer count
- Support for future storage backend abstraction

## Reliability
- Data integrity through checksums and validation
- Atomic operations prevent partial writes
- Comprehensive backup and recovery mechanisms
- Graceful degradation when storage is unavailable

## Maintainability
- Clear, well-documented storage interfaces
- Comprehensive error messages and logging
- Modular design for easy modification and extension
- Good test coverage for future changes