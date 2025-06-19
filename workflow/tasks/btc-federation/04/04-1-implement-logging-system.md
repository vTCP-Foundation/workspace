# 04-1 - Implement Logging System with Zerolog

# Links
- [PRD-04: Logging System Implementation](../../prd/btc-federation/04_logging_system_implementation.md)

# Description
Implement a comprehensive logging system for the btc-federation project using Zerolog library. Replace all scattered fmt.Printf, fmt.Println, log.Printf, and log.Fatalf calls with a centralized, configurable logging system that supports file output with rotation, JSON formatting, and colored console output. Create a comprehensive demo module for validation.

# Requirements and DOD

## Core Implementation Requirements
1. **Logger Package Creation**:
   - Create `internal/logger` package with single-file implementation
   - Use Zerolog library with all external dependencies isolated within the logger file
   - Support four log levels: error, warn, info, debug
   - Implement configurable console output with optional colors
   - Implement configurable file output with JSON formatting  
   - Implement file rotation at configurable size limits
   - Design architecture to enable easy future library migration

2. **Configuration Extension**:
   - Extend existing `logging` configuration section in `internal/types/config.go`
   - Add `console_output: boolean` - controls console logging
   - Add `console_color: boolean` - controls colored console output
   - Add `file_output: boolean` - controls file logging
   - Add `file_name: string` - specifies log file name
   - Add `file_max_size: string` - specifies rotation size threshold
   - Update validation in `internal/config/config.go` for all new fields

3. **Legacy Code Elimination**:
   - Replace ALL fmt.Printf calls in cmd/ and internal/ directories
   - Replace ALL fmt.Println calls in cmd/ and internal/ directories
   - Replace ALL log.Printf calls in cmd/ and internal/ directories
   - Replace ALL log.Fatalf calls in cmd/ and internal/ directories
   - Preserve key messages: "BTC Federation Node starting with config", "Node started successfully", "Listening addresses"
   - Replace all TODO comments related to logging system

4. **Demo Module Implementation**:
   - Create `cmd/demo/logger/main.go` with package `main`
   - Implement comprehensive validation that performs ALL the following checks:
     a. **Binary Check**: Verify btc-federation binary exists in build directory
     b. **Config Check**: Verify conf.yaml exists in build directory  
     c. **Config Read**: Read log file name from configuration file
     d. **Cleanup**: Delete existing log file with name from step c
     e. **Execution**: Start btc-federation binary and wait exactly 5 seconds
     f. **File Creation**: Verify log file was created with name from step c
     g. **JSON Validation**: Verify log file contains valid JSON format entries
     h. **Message Validation**: Verify log file contains these required messages:
        - "BTC Federation Node starting with config"
        - "Node started successfully"
        - "Listening addresses"
     i. **Code Validation**: Verify zero instances of fmt.Printf/fmt.Println/log.Printf/log.Fatalf remain in cmd/ and internal/ directories
     j. **Success Criteria**: ALL checks must pass for demo to return success status

## Definition of Done
- **Git Branch**: All work completed in dedicated feature branch `task/04-1-implement-logging-system`
- Logger package implemented in single file using Zerolog
- All external dependencies isolated within logger implementation
- Configuration extended with all required logging fields
- Configuration validation updated for new fields
- Zero legacy logging calls remain in target directories
- All key messages preserved in new logging system
- Demo module created and successfully validates all requirements
- All demo validation steps pass without errors
- File rotation works at configured size limits
- JSON formatting works for file output
- Colored console output works for console output
- **Branch merge**: Feature branch successfully merged to main after all validations pass

# Implementation Plan

## Prerequisites
- **Git Branch Setup**: Create and checkout feature branch `task/04-1-implement-logging-system` from `main`
- **Branch Naming**: Follow project convention: `task/<PRD-ID>-<TASK-ID>-<short-description>`

## Phase 1: Logger Package and Configuration (Week 1)
1. **Create Logger Package Structure**:
   - Create `internal/logger/` directory
   - Implement single-file logger using Zerolog library
   - Design clean API that isolates Zerolog dependencies
   - Implement console output with color support
   - Implement file output with JSON formatting
   - Implement file rotation based on size configuration

2. **Extend Configuration**:
   - Update `internal/types/config.go` with new logging fields
   - Update `internal/config/config.go` with validation logic
   - Update default configuration values
   - Test configuration loading and validation

## Phase 2: Legacy Code Replacement (Week 2)
1. **Systematic Code Replacement**:
   - Audit all fmt.Printf/fmt.Println/log.Printf/log.Fatalf calls
   - Replace calls in cmd/btc-federation/main.go
   - Replace calls in internal/network/manager.go
   - Replace calls in internal/network/peer_handler.go
   - Replace calls in internal/network/events.go
   - Replace calls in internal/network/host.go
   - Replace calls in cmd/utils/keygen/keygen.go
   - Ensure appropriate log levels are used for each replacement
   - Preserve all critical messages

2. **Integration Testing**:
   - Test application startup with new logging
   - Test file creation and rotation
   - Test console output formatting
   - Test JSON file output formatting

## Phase 3: Demo Module and Validation (Week 2-3)
1. **Demo Module Development**:
   - Create `cmd/demo/logger/main.go`
   - Implement build directory constant
   - Implement binary and config file checks
   - Implement configuration reading logic
   - Implement log file cleanup
   - Implement binary execution with 5-second wait
   - Implement log file creation verification
   - Implement JSON format validation
   - Implement required message validation
   - Implement source code validation for legacy calls
   - Implement comprehensive error reporting

2. **Final Validation**:
   - Run demo module and verify all checks pass
   - Test with different configuration scenarios
   - Verify file rotation under load
   - Verify colored console output
   - Confirm zero legacy logging calls remain

## Phase 4: Git Branch Management
1. **Commit Management**:
   - Use commit message format: `[04-1] <description>`
   - Make atomic commits for each logical change
   - Link commits to task documentation

2. **Pull Request**:
   - Create pull request to merge feature branch to main
   - Include task completion validation
   - Ensure all demo module checks pass before merge

# Test Plan

## Objective
Validate complete logging system implementation including logger package functionality, configuration handling, legacy code replacement, and comprehensive demo module validation.

## Test Scope
- Logger package functionality (levels, console/file output, rotation, JSON formatting)
- Configuration extension and validation
- Legacy code elimination verification
- Demo module comprehensive validation
- File rotation behavior
- JSON format compliance
- Console color output

## Environment & Setup
- Development environment with Go compiler
- Build directory with btc-federation binary and conf.yaml
- File system with write permissions for log files
- Terminal supporting colored output for visual verification

## Mocking Strategy
- No mocking required - integration testing with real file system
- Demo module performs real validation against actual built binary
- Configuration testing uses real YAML parsing

## Key Test Scenarios

### Logger Package Testing
1. **Log Level Testing**:
   - Verify error, warn, info, debug levels work correctly
   - Test log level filtering based on configuration
   - Verify appropriate console colors for each level

2. **Output Destination Testing**:
   - Test console-only output (file_output: false)
   - Test file-only output (console_output: false)
   - Test dual output (both console and file enabled)
   - Test disabled output (both disabled)

3. **File Rotation Testing**:
   - Create logs exceeding configured file_max_size
   - Verify new log file creation with sequential naming
   - Verify original file preservation during rotation

4. **Format Testing**:
   - Verify JSON format in log files
   - Verify human-readable format in console
   - Verify color codes in console output

### Configuration Testing
1. **Field Validation**:
   - Test all new logging configuration fields
   - Test invalid values rejection
   - Test default values when fields omitted

2. **Integration Testing**:
   - Verify configuration loading with new fields
   - Test configuration validation error messages

### Legacy Code Replacement Testing
1. **Code Audit**:
   - Search for remaining fmt.Printf/fmt.Println/log.Printf/log.Fatalf
   - Verify zero instances in cmd/ and internal/ directories
   - Confirm all TODO logging comments replaced

2. **Message Preservation**:
   - Verify all critical messages still appear in logs
   - Test message content matches expected strings

### Demo Module Testing
1. **All Validation Steps**:
   - Test binary and config existence checks
   - Test configuration reading functionality
   - Test log file cleanup
   - Test binary execution and timing
   - Test log file creation verification
   - Test JSON format validation
   - Test required message detection
   - Test source code validation
   - Test overall success/failure reporting

2. **Error Scenarios**:
   - Test behavior when binary missing
   - Test behavior when config missing
   - Test behavior when log file creation fails
   - Test behavior when required messages missing

## Success Criteria
- All logger package tests pass
- Configuration validation works correctly
- Zero legacy logging calls found in source code audit
- Demo module successfully completes all validation steps
- File rotation works under various scenarios
- JSON format validation passes
- Console color output displays correctly
- All critical messages preserved and detectable

# Verification and Validation

This is a **Complex Task** requiring comprehensive validation across multiple domains:

## Architecture integrity
- **Single-file implementation**: Logger package implemented in one file with isolated dependencies
- **Clean API design**: Logger interface does not expose Zerolog-specific types to consumers
- **Configuration integration**: Logger properly integrates with existing configuration system
- **Modular design**: Easy to replace Zerolog with different library in future
- **Separation of concerns**: Logging logic separated from business logic throughout codebase

## Security
- **Log file permissions**: Log files created with secure permissions (644)
- **Sensitive data protection**: No private keys, passwords, or sensitive data logged
- **Input validation**: Configuration values properly validated to prevent injection
- **File path safety**: Log file paths constructed safely to prevent directory traversal

## Performance
- **Minimal overhead**: Logging does not significantly impact application performance
- **Efficient file I/O**: File operations optimized for production use
- **Memory usage**: Logger does not cause memory leaks or excessive allocation
- **Rotation efficiency**: File rotation performs atomically without blocking

## Scalability
- **High-volume logging**: System handles sustained high log volume without issues
- **File rotation scaling**: Rotation mechanism works effectively under load
- **Concurrent access**: Logger handles concurrent access from multiple goroutines safely
- **Resource management**: Proper cleanup of file handles and resources

## Reliability
- **Error handling**: Logging failures do not crash the application
- **Atomic operations**: File rotation and creation operations are atomic
- **Graceful degradation**: System continues functioning if logging fails
- **Recovery mechanisms**: Logger recovers from temporary file system issues

## Maintainability
- **Code organization**: Logger implementation is well-structured and readable
- **Documentation**: Clear API documentation and usage examples
- **Migration path**: Architecture supports easy future library changes
- **Testing coverage**: Comprehensive test coverage for all logging functionality

## Cost
- **Resource efficiency**: Minimal CPU and memory overhead
- **Storage optimization**: Efficient log file storage and rotation
- **Development efficiency**: Simple API reduces development time
- **Maintenance cost**: Architecture minimizes long-term maintenance burden

## Compliance
- **Policy adherence**: Implementation follows all project policies and standards
- **Code quality**: Code meets project quality standards and best practices
- **Configuration compliance**: All configuration changes follow established patterns
- **Documentation standards**: All documentation follows project templates and guidelines 