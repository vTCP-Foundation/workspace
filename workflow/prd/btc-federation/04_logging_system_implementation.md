# [PRD-04] Logging System Implementation

## Document Information
- **Project Name**: BTC Federation
- **Phase/Iteration**: Phase 4
- **Document Version**: 1.0
- **Date**: December 2024
- **Author(s)**: System
- **Stakeholders**: Development Team
- **Status**: Draft
- **Previous PRD**: [PRD-03] MuSig2 FROST Comparative Analysis
- **Related Documents**: [Internal Config Module](../../internal/config/config.go)

## Executive Summary
This PRD defines the implementation of a comprehensive logging system for the btc-federation project. The current implementation uses basic `fmt.Printf`, `fmt.Println`, `log.Printf`, and `log.Fatalf` calls scattered throughout the codebase. This iteration will replace all these calls with a structured, configurable logging system that supports file output with rotation, JSON formatting, and colored console output.

### Current project state
- Basic logging through fmt and log packages
- No centralized logging configuration
- No file-based logging capability
- No log rotation functionality

### This iteration's focus
- Create centralized logger package in `internal/logger`
- Extend existing `logging` configuration section
- Replace all fmt/log calls with structured logging
- Implement file rotation based on configurable size limits
- Create comprehensive demo module for validation

### Connection to overall vision
This logging system provides essential observability infrastructure that will support debugging, monitoring, and operational oversight of the btc-federation network components.

## Iteration Context

### Previous Iterations Summary
- **PRD-01**: Established Taproot multisig foundation
- **PRD-02**: Implemented P2P network stack with basic console output
- **PRD-03**: Conducted MuSig2 FROST comparative analysis
- **Completed Features**: Core network functionality with primitive logging via fmt/log

### Lessons Learned
- Current logging approach lacks consistency and configurability
- Debug information is mixed with user-facing output
- No persistent logging for post-mortem analysis
- TODO comments in code indicate recognized need: "TODO: Use proper logging system"

### Current State Analysis
- **What's working well**: Basic information output to console
- **Pain points identified**: 
  - Inconsistent logging formats across modules
  - No way to control log levels or output destinations
  - No persistent logging for operational analysis
  - Debug output clutters user interface

## Problem Statement

### Background
The btc-federation project currently uses scattered `fmt.Printf`, `fmt.Println`, `log.Printf`, and `log.Fatalf` calls throughout the codebase for information output. This approach lacks the structured logging capabilities needed for a production network component.

### Problem Description
The current logging approach presents several critical issues:
- **Unstructured output**: Mixed use of fmt and log packages without consistent formatting
- **No configurability**: Cannot control what gets logged or where it goes
- **No persistence**: All output goes to console only, no file-based logging
- **No rotation**: No mechanism to manage log file sizes
- **Poor operational visibility**: Difficult to analyze system behavior post-mortem
- **Inconsistent error handling**: Some errors use log.Fatalf while others use fmt.Printf

### Success Metrics
- **Primary KPIs**: 
  - Zero remaining fmt.Printf/fmt.Println/log.Printf/log.Fatalf calls in cmd/ and internal/ directories
  - Successful file logging with rotation at configured size limits
  - JSON-formatted log entries in files
  - Colored console output differentiated by log level
- **Secondary KPIs**:
  - Demo module successfully validates all requirements
  - Configuration validation for new logging parameters
- **Target Values**: 100% replacement of legacy logging calls

## Project Scope

### This Iteration's Scope

#### New Features/Enhancements
1. **Logger Package**: Create `internal/logger` package with Zerolog-based implementation contained in a single file
2. **Configuration Extension**: Extend existing `logging` section in configuration with:
   - `console_output: boolean` - enable/disable console output
   - `console_color: boolean` - enable/disable colored console output  
   - `file_output: boolean` - enable/disable file output
   - `file_name: string` - log file name (placed in same directory as binary)
   - `file_max_size: string` - maximum file size before rotation (e.g., "10MB")
3. **Legacy Code Replacement**: Replace all fmt.Printf, fmt.Println, log.Printf, log.Fatalf calls
4. **Demo Module**: Create `cmd/demo/logger/main.go` for comprehensive validation

#### Technical Improvements
- Structured logging with consistent format
- Configurable log levels (error, warn, info, debug) 
- File rotation mechanism
- JSON formatting for file output
- Human-readable colored output for console

#### Modifications to Existing Features
- Update `internal/types/config.go` with new logging configuration fields
- Update `internal/config/config.go` with validation for new fields
- Replace all TODO comments related to logging system

### Explicitly Out of Scope
- Log aggregation or forwarding to external systems
- Log retention policies (unlimited rotated files for now)
- Performance benchmarking of logging system
- Integration with external monitoring systems

### Dependencies from Previous Iterations
- Configuration system from existing implementation
- Network manager and peer handler modules that contain logging calls to be replaced

### Future Roadmap Impact
This logging foundation will support future monitoring, alerting, and observability requirements as the federation network scales.

## User Stories & Requirements

### User Personas

#### Primary User: Node Operator
- **Role**: Runs btc-federation nodes in production
- **Goals**: Monitor node health, debug connectivity issues, maintain operational logs
- **Pain Points**: Currently has no persistent logs, difficult to troubleshoot issues after they occur
- **Technical Proficiency**: Intermediate

#### Secondary User: Developer
- **Role**: Develops and maintains btc-federation codebase
- **Goals**: Debug code issues, understand system behavior, maintain code quality
- **Pain Points**: Inconsistent logging makes debugging difficult
- **Technical Proficiency**: Advanced

### Functional Requirements

#### New Features for This Iteration

1. **Logger Package Creation**
   - **Description**: Create Zerolog-based logging package in `internal/logger` with single-file implementation
   - **User Story**: As a developer, I want a centralized logging system so that all components use consistent logging
   - **Rationale**: Eliminates scattered fmt/log calls and provides structured logging with future-proof architecture
   - **Builds Upon**: Existing configuration system
   - **Acceptance Criteria**:
     - Package implemented using Zerolog library
     - All implementation contained in a single file
     - All external library dependencies isolated within logger implementation
     - Package supports error, warn, info, debug levels
     - Configurable console and file output
     - JSON formatting for file output
     - Colored console output by level
     - File rotation at configurable size
     - Easy migration path for future library changes
   - **Priority**: High
   - **Dependencies**: Configuration system extension

2. **Configuration Extension**
   - **Description**: Extend existing `logging` configuration section
   - **User Story**: As a node operator, I want to configure logging behavior so that I can control where logs go and how they're formatted
   - **Rationale**: Provides operational control over logging behavior
   - **Builds Upon**: Existing configuration validation system
   - **Acceptance Criteria**:
     - `console_output` boolean field controls console logging
     - `console_color` boolean field controls colored output
     - `file_output` boolean field controls file logging
     - `file_name` string field specifies log file name
     - `file_max_size` string field specifies rotation size
     - All fields validated in configuration loading
   - **Priority**: High
   - **Dependencies**: None

3. **Legacy Code Replacement**
   - **Description**: Replace all fmt.Printf, fmt.Println, log.Printf, log.Fatalf calls
   - **User Story**: As a developer, I want consistent logging throughout the codebase so that all output follows the same format and configuration
   - **Rationale**: Eliminates inconsistent logging and provides unified control
   - **Builds Upon**: New logger package
   - **Acceptance Criteria**:
     - Zero fmt.Printf calls remain in cmd/ and internal/ directories
     - Zero fmt.Println calls remain in cmd/ and internal/ directories  
     - Zero log.Printf calls remain in cmd/ and internal/ directories
     - Zero log.Fatalf calls remain in cmd/ and internal/ directories
     - All replaced calls use appropriate log levels
     - Key messages preserved: "BTC Federation Node starting with config", "Node started successfully", "Listening addresses"
   - **Priority**: High
   - **Dependencies**: Logger package, configuration extension

4. **Demo Module Implementation**
   - **Description**: Create comprehensive validation module at `cmd/demo/logger/main.go`
   - **User Story**: As a developer, I want automated validation so that I can verify the logging system works correctly
   - **Rationale**: Provides comprehensive testing of all logging requirements
   - **Builds Upon**: Complete logging system implementation
   - **Acceptance Criteria**:
     - Checks for btc-federation binary and conf.yaml in build directory
     - Reads log file name from configuration
     - Cleans up existing log file before test
     - Starts btc-federation binary and waits 5 seconds
     - Verifies log file creation and JSON format
     - Verifies presence of required log messages
     - Verifies no legacy logging calls remain in source code
     - All checks must pass for demo to succeed
   - **Priority**: High
   - **Dependencies**: Complete logging system implementation

### Non-Functional Requirements

#### Performance
- Logging should not significantly impact application performance
- File I/O should be efficient and non-blocking where possible

#### Security
- Log files should be created with appropriate permissions (644)
- No sensitive information should be logged (private keys, passwords)

#### Scalability
- File rotation should handle high-volume logging scenarios
- Log file naming should support multiple rotated files

#### Reliability
- Logging failures should not crash the application
- File rotation should be atomic to prevent log loss

## Technical Specifications

### Architecture Evolution
- **Current Architecture**: Scattered fmt/log calls throughout modules
- **Proposed Changes**: Centralized Zerolog-based logger package with configuration-driven behavior and dependency isolation
- **Backwards Compatibility**: Maintains all existing log message content
- **Migration Requirements**: Systematic replacement of all logging calls
- **Future-proof Design**: Single-file implementation with isolated external dependencies enables easy library migration

### Technology Stack Updates

#### New Technologies/Libraries
- **Zerolog**: Use Zerolog library for structured logging with JSON support, colors, and rotation capabilities
- **Integration approach**: Implement logger in a single file within internal/logger package with all external library dependencies isolated to enable easy future library migration
- **Dependency isolation**: All external logging library dependencies must be contained within the logger implementation file only

#### Version Updates
- No version updates required for core dependencies

### Integration Requirements

#### New Integrations
- Logger package integration with configuration system
- Logger initialization in main application startup

#### Modified Integrations
- All existing modules updated to use new logger instead of fmt/log

### Data Requirements

#### Data Models
- Log entry structure with timestamp, level, message, and contextual fields
- Configuration structure extension for logging parameters

#### Data Storage
- Log files stored in same directory as binary
- Rotated log files with sequential naming
- JSON format for file storage, human-readable for console

## Implementation Plan

### This Iteration Timeline
- **Duration**: 2-3 weeks
- **Key Deliverables**: 
  - Logger package implementation
  - Configuration updates
  - Legacy code replacement
  - Demo module creation and validation

### Iteration Milestones

| Milestone | Date | Description | Dependencies | Risk Level |
|-----------|------|-------------|--------------|------------|
| Logger Package | Week 1 | Create internal/logger with all required features | Configuration extension | Medium |
| Config Updates | Week 1 | Extend logging configuration and validation | None | Low |
| Code Replacement | Week 2 | Replace all fmt/log calls systematically | Logger package | Medium |
| Demo Module | Week 2-3 | Create and validate demo module | Complete implementation | Low |
| Final Validation | Week 3 | Ensure all requirements met | Demo module | Low |

### Dependencies on Other Teams/Projects
- None - self-contained within btc-federation project

### Integration Points with Previous Work
- Builds upon existing configuration system
- Maintains all existing log message content for operational continuity

### Resource Requirements

#### Team Structure
- **Developer**: 1 developer for implementation
- **Testing**: Demo module provides comprehensive validation

## Risk Management

### Technical Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Logging library performance impact | Medium | Low | Performance testing, efficient library selection |
| File rotation implementation complexity | Medium | Medium | Use proven libraries, thorough testing |
| Missing log calls during replacement | High | Medium | Systematic approach, demo validation |
| Configuration validation complexity | Low | Low | Follow existing patterns |

### Business Risks

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|-------------------|
| Operational disruption from logging changes | Medium | Low | Demo validation, careful message preservation |

## Testing Strategy

### Testing Approach for This Iteration

#### New Feature Testing
- Unit testing for logger package functionality
- Configuration validation testing
- File rotation testing under various conditions

#### Integration Testing  
- End-to-end testing via demo module
- Validation of all logging calls replacement
- JSON format validation
- Console color output validation

#### Regression Testing
- **Scope**: Ensure all existing functionality continues to work
- **Critical User Journeys**: Node startup, network connectivity, graceful shutdown
- **Demo Module**: Serves as comprehensive regression test

### Quality Gates
- All demo module checks must pass
- Zero remaining legacy logging calls in target directories
- Configuration validation tests pass
- Log file creation and rotation work correctly

## Deployment & Release Strategy

### Release Approach
- **Release Type**: Minor release (logging infrastructure)
- **Rollout Strategy**: Direct replacement approach
- **Rollback Plan**: Revert to previous logging approach if critical issues arise

### Feature Flags & Gradual Rollout
- Configuration-based enabling of file logging
- Console output can be disabled independently

### Database Migrations
- No database changes required

### Communication Plan
- **Internal**: Update team on logging configuration changes
- **Documentation Updates**: Update README with new logging configuration options

## Success Metrics & Monitoring

### Iteration-Specific KPIs
- **Primary Metrics**: 
  - Demo module success rate (target: 100%)
  - Legacy logging call elimination (target: 0 remaining)
  - Log file creation success (target: 100%)
  - JSON format validation (target: 100%)
- **Leading Indicators**: Successful logger package tests, configuration validation
- **Baseline Values**: Currently 50+ legacy logging calls across codebase
- **Target Values**: 0 legacy logging calls, 100% demo validation success

### Monitoring Plan
- **New Monitoring**: Demo module provides comprehensive validation
- **Enhanced Monitoring**: Log file presence and format validation

### Review Schedule
- **Daily**: Progress on code replacement
- **Weekly**: Demo module validation runs
- **Post-Release Review**: Evaluate logging system effectiveness

## Conditions of Satisfaction (CoS)

1. **Logger Package**: A complete `internal/logger` package exists with support for:
   - Implementation using Zerolog library
   - Single-file implementation with isolated external dependencies
   - Four log levels: error, warn, info, debug
   - Configurable console output with optional colors
   - Configurable file output with JSON formatting
   - File rotation at configurable size limits
   - Architecture enabling easy future library migration

2. **Configuration Extension**: The existing `logging` configuration section includes:
   - `console_output: boolean` - controls console logging
   - `console_color: boolean` - controls colored console output
   - `file_output: boolean` - controls file logging  
   - `file_name: string` - specifies log file name
   - `file_max_size: string` - specifies rotation size threshold

3. **Legacy Code Elimination**: Zero instances of the following remain in `cmd/` and `internal/` directories:
   - `fmt.Printf` calls
   - `fmt.Println` calls
   - `log.Printf` calls
   - `log.Fatalf` calls

4. **Message Preservation**: The following key messages are preserved in the new logging system:
   - "BTC Federation Node starting with config"
   - "Node started successfully"  
   - "Listening addresses"

5. **Demo Module Success**: The `cmd/demo/logger/main.go` module successfully executes all validation steps:
   - Verifies btc-federation binary and conf.yaml exist in build directory
   - Reads log file name from configuration
   - Cleans up existing log file
   - Starts binary and waits 5 seconds
   - Verifies log file creation and JSON format
   - Verifies presence of required log messages
   - Verifies elimination of legacy logging calls
   - Returns success status only when all checks pass

## Scope and Boundaries

### Included in This PRD
- Complete logging system implementation
- Configuration system extensions
- Systematic replacement of legacy logging calls
- Comprehensive demo validation module
- File rotation functionality
- JSON and colored console output

### Explicitly Excluded from This PRD
- External log forwarding or aggregation
- Log retention policies or cleanup
- Performance optimization beyond basic efficiency
- Integration with monitoring systems
- Structured logging fields beyond basic message content

## Dependencies
- Existing configuration system in `internal/config`
- Existing type definitions in `internal/types`  
- Build process that creates btc-federation binary in build directory

## Assumptions and Constraints

### Key Assumptions
- Build directory contains btc-federation binary and conf.yaml before demo execution
- File system supports file rotation operations
- Standard Go logging libraries provide sufficient functionality
- Current log message content is operationally important and should be preserved

### Constraints
- Must maintain backward compatibility with existing configuration structure
- Log files created in same directory as binary (no separate log directory)
- No limit on number of rotated log files (for now)
- Demo module must be self-contained validation
- Logger implementation must be contained in a single file with isolated external dependencies
- Must use Zerolog library specifically for implementation

## Task List
Reference to the task list file for this PRD: `workflow/tasks/btc-federation/04/tasks.md`

---

**Document History**
| Version | Date | Author | Changes | Iteration |
|---------|------|--------|---------|-----------|
| 1.0 | December 2024 | System | Initial draft for logging system implementation | Phase 4 |

**Related Documents**
- **Previous PRD**: [PRD-03] MuSig2 FROST Comparative Analysis
- **Configuration Module**: internal/config/config.go
- **Type Definitions**: internal/types/config.go