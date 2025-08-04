# 02-2 - NodeConfig Extension for Logging

# Links
- [PRD-02: Log Rotation Testing for BTC Federation](../../prd/btc-federation-test-suite/02-log-rotation-testing.md)

# Description
Extend the NodeConfig structure in `pkg/testsuite/node.go` to support all logging configuration parameters, enabling comprehensive programmatic configuration of logging settings for containerized test scenarios.

# Requirements and DOD

## Core Implementation Requirements
1. **NodeConfig Structure Extension**:
   - Add `ConsoleOutput bool` field to NodeConfig struct
   - Add `ConsoleColor bool` field to NodeConfig struct  
   - Add `FileOutput bool` field to NodeConfig struct
   - Add `FileName string` field to NodeConfig struct
   - Add `FileMaxSize string` field to NodeConfig struct
   - Maintain backward compatibility with existing `LogLevel` and `LogFormat` fields
   - Add appropriate YAML tags for configuration serialization

2. **Default Value Enhancement**:
   - Update `validateAndSetDefaults` function to handle new logging fields
   - Set `ConsoleOutput: true` as default when not specified
   - Set `ConsoleColor: true` as default when not specified
   - Set `FileOutput: true` as default when not specified
   - Set `FileName: "btc-federation.log"` as default when not specified
   - Set `FileMaxSize: "10MB"` as default when not specified
   - Preserve existing default logic for `LogLevel` and `LogFormat`

3. **Environment Variable Generation**:
   - Update `GetEnvironmentVariables` method to include new logging parameters
   - Generate `LOGGING_FILE_NAME` environment variable from `FileName` field
   - Generate `LOGGING_FILE_MAX_SIZE` environment variable from `FileMaxSize` field
   - Maintain existing environment variable generation for other parameters
   - Ensure proper string conversion and validation

## Definition of Done
- NodeConfig struct extended with all required logging configuration fields
- Default value assignment implemented for new fields in `validateAndSetDefaults`
- Environment variable generation updated to support new logging parameters
- Backward compatibility maintained with existing NodeConfig functionality
- YAML serialization/deserialization works correctly for new fields
- All new fields properly documented with comments

# Implementation Plan

## Phase 1: Structure Definition
1. **NodeConfig Struct Extension**:
   - Add new logging fields to NodeConfig struct in `pkg/testsuite/node.go`
   - Add appropriate YAML tags for each field
   - Add documentation comments explaining field purpose and default values
   - Position new fields logically with existing logging fields

2. **Constants Definition**:
   - Add constants for default logging values to maintain consistency
   - Define `DefaultConsoleOutput = true`
   - Define `DefaultConsoleColor = true`
   - Define `DefaultFileOutput = true`
   - Define `DefaultFileName = "btc-federation.log"`
   - Define `DefaultFileMaxSize = "10MB"`

## Phase 2: Default Value Implementation
1. **Update validateAndSetDefaults Function**:
   - Add validation and default assignment for `ConsoleOutput` field
   - Add validation and default assignment for `ConsoleColor` field
   - Add validation and default assignment for `FileOutput` field
   - Add validation and default assignment for `FileName` field
   - Add validation and default assignment for `FileMaxSize` field
   - Validate `FileMaxSize` format (e.g., "1kB", "10MB")

2. **Validation Logic Enhancement**:
   - Add file name validation (non-empty string, valid file name format)
   - Add file size format validation (number + unit: kB, MB, GB)
   - Ensure boolean fields have proper default values
   - Add error handling for invalid values

## Phase 3: Environment Variable Integration
1. **Update GetEnvironmentVariables Method**:
   - Add `LOGGING_FILE_NAME` environment variable generation
   - Add `LOGGING_FILE_MAX_SIZE` environment variable generation
   - Maintain existing environment variable generation
   - Ensure proper string formatting and escaping

2. **Integration Testing**:
   - Test environment variable generation with various field combinations
   - Verify integration with container infrastructure
   - Test with both default and custom logging values

## Phase 4: Documentation and Testing
1. **Documentation Updates**:
   - Update struct field comments with usage examples
   - Document default values and validation rules
   - Add examples of typical logging configurations

2. **Backward Compatibility Testing**:
   - Test with existing code that uses NodeConfig
   - Verify no breaking changes to existing functionality
   - Test YAML serialization/deserialization

# Test Plan

## Objective
Validate that NodeConfig structure properly supports new logging configuration fields and integrates correctly with existing test infrastructure.

## Test Scope
- NodeConfig struct field handling and default value assignment
- Environment variable generation accuracy
- YAML serialization/deserialization functionality
- Backward compatibility with existing code

## Environment & Setup
- Go development environment with test capabilities
- YAML parsing and validation tools
- Container environment for integration testing

## Mocking Strategy
- Unit testing for NodeConfig struct functionality
- No mocking required for struct field operations
- Integration testing with real environment variable generation

## Key Test Scenarios

### Struct Functionality Testing
1. **Default Value Assignment**:
   - Create NodeConfig without specifying new logging fields
   - Verify all logging fields get appropriate default values
   - Test `validateAndSetDefaults` function with various input scenarios

2. **Custom Value Assignment**:
   - Create NodeConfig with custom logging field values
   - Verify custom values are preserved and validated
   - Test with various file size formats and file names

3. **Field Validation Testing**:
   - Test with invalid file size formats
   - Test with invalid file names (empty, special characters)
   - Verify proper error handling for invalid values

### Environment Variable Generation Testing
1. **Complete Environment Variable Set**:
   - Test `GetEnvironmentVariables` with all logging fields populated
   - Verify `LOGGING_FILE_NAME` and `LOGGING_FILE_MAX_SIZE` variables generated
   - Verify existing environment variables still generated correctly

2. **Mixed Configuration Testing**:
   - Test with some logging fields using defaults, others custom
   - Verify environment variable generation accuracy
   - Test with various file size units (kB, MB, GB)

### YAML Serialization Testing
1. **Serialization Testing**:
   - Serialize NodeConfig with logging fields to YAML
   - Verify proper YAML structure and field representation
   - Test with both default and custom values

2. **Deserialization Testing**:
   - Deserialize YAML to NodeConfig struct
   - Verify all logging fields properly populated
   - Test with missing fields (should use defaults)

### Integration Testing
1. **Container Integration**:
   - Test NodeConfig with container creation
   - Verify environment variables passed correctly to container
   - Test with btc-federation binary execution

2. **Backward Compatibility**:
   - Test with existing test code that uses NodeConfig
   - Verify no breaking changes to existing functionality
   - Test with existing YAML configuration files

## Success Criteria
- All new logging fields work correctly with default and custom values
- Environment variable generation includes new logging parameters
- YAML serialization/deserialization works for all fields
- No regression in existing NodeConfig functionality
- Integration with container infrastructure works correctly

# Verification and Validation

This is a **Simple Task** requiring basic structural enhancement with focus on data handling and backward compatibility.

## Architecture integrity
- **Data Structure Design**: New fields logically integrated with existing NodeConfig structure
- **Default Value Management**: Consistent default value handling across all configuration fields
- **Environment Variable Generation**: Clean separation between struct fields and environment variable creation
- **API Consistency**: New fields follow existing naming and documentation patterns

## Security
- **Data Validation**: Proper validation of file names and sizes to prevent injection attacks
- **Environment Variable Security**: No sensitive data exposure through new environment variables
- **Input Sanitization**: File name validation prevents directory traversal attempts
- **Default Value Security**: Default values do not introduce security vulnerabilities

## Performance
- **Struct Operations**: Minimal performance impact from additional fields
- **Environment Variable Generation**: Efficient string operations for variable creation
- **Memory Usage**: Negligible memory overhead from new string and boolean fields
- **Validation Performance**: Efficient validation logic without complex operations

## Reliability
- **Default Value Consistency**: Reliable default value assignment across different scenarios
- **Field Validation**: Robust validation prevents invalid configurations
- **Error Handling**: Proper error messages for validation failures
- **Backward Compatibility**: Existing functionality remains stable with new fields

## Maintainability
- **Code Organization**: New fields clearly documented and logically grouped
- **Validation Logic**: Clear and maintainable validation rules
- **Testing Support**: Structure supports easy testing of different configuration scenarios
- **Documentation**: Comprehensive field documentation with usage examples

## Cost
- **Development Efficiency**: Straightforward implementation without complex dependencies
- **Memory Efficiency**: Minimal memory overhead from additional fields
- **Maintenance Cost**: Simple structure changes with low maintenance requirements
- **Testing Cost**: Basic testing requirements for struct functionality

## Compliance
- **Go Best Practices**: Implementation follows Go struct and method naming conventions
- **YAML Standards**: YAML tags comply with standard serialization practices
- **Project Policies**: All changes follow project policies for struct design and documentation
- **API Standards**: New methods maintain consistency with existing NodeConfig API 