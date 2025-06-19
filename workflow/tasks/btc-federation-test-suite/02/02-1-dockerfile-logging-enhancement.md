# 02-1 - Dockerfile Logging Enhancement

# Links
- [PRD-02: Log Rotation Testing for BTC Federation](../../prd/btc-federation-test-suite/02-log-rotation-testing.md)

# Description
Extend the existing Dockerfile to support additional logging environment variables (`LOGGING_FILE_NAME` and `LOGGING_FILE_MAX_SIZE`) and enhance the configuration generation script to include all required logging parameters for comprehensive test scenarios.

# Requirements and DOD

## Core Implementation Requirements
1. **Environment Variables Extension**:
   - Add `LOGGING_FILE_NAME` environment variable with default value `"btc-federation.log"`
   - Add `LOGGING_FILE_MAX_SIZE` environment variable with default value `"10MB"`
   - Maintain backward compatibility with existing environment variables
   - Ensure proper environment variable validation in generate-config.sh

2. **Configuration Generation Enhancement**:
   - Extend the logging section in conf.yaml generation to include:
     - `console_output: true` (fixed value)
     - `console_color: true` (fixed value) 
     - `file_output: true` (fixed value)
     - `file_name: $LOGGING_FILE_NAME` (from environment variable)
     - `file_max_size: $LOGGING_FILE_MAX_SIZE` (from environment variable)
   - Preserve existing `level: info` and `format: json` parameters
   - Ensure proper YAML formatting and indentation

3. **Default Value Handling**:
   - When `LOGGING_FILE_NAME` is not set, default to `"btc-federation.log"`
   - When `LOGGING_FILE_MAX_SIZE` is not set, default to `"10MB"`
   - Document default values in configuration generation script comments

## Definition of Done
- Environment variables `LOGGING_FILE_NAME` and `LOGGING_FILE_MAX_SIZE` supported
- Configuration generation script updated to include all logging parameters
- Default values properly handled when environment variables not provided
- Generated conf.yaml contains complete logging section with all required parameters
- Backward compatibility maintained with existing container functionality
- Documentation comments updated to reflect new environment variables

# Implementation Plan

## Phase 1: Environment Variables Setup
1. **Add Environment Variables Declaration**:
   - Add `ENV LOGGING_FILE_NAME=""` to Dockerfile
   - Add `ENV LOGGING_FILE_MAX_SIZE=""` to Dockerfile
   - Position new variables after existing logging-related environment variables

2. **Update Configuration Generation Script**:
   - Modify `/generate-config.sh` creation in Dockerfile
   - Add default value assignment logic for new environment variables
   - Implement validation for proper variable handling

## Phase 2: Configuration Generation Enhancement
1. **Extend conf.yaml Generation**:
   - Update the logging section template in generate-config.sh
   - Add all required logging parameters with proper variable substitution
   - Ensure YAML syntax correctness and proper indentation

2. **Default Value Implementation**:
   - Add conditional logic to set defaults when variables are empty
   - Test default value assignment in container environment
   - Verify configuration generation with and without environment variables

## Phase 3: Testing and Validation
1. **Container Build Testing**:
   - Build container image with updated Dockerfile
   - Test container startup with default values
   - Test container startup with custom environment variables
   - Validate generated conf.yaml content

2. **Integration Verification**:
   - Verify compatibility with existing container infrastructure
   - Test with different combinations of environment variables
   - Ensure no regression in existing functionality

# Test Plan

## Objective
Validate that Dockerfile properly supports new logging environment variables and generates correct configuration files under various scenarios.

## Test Scope
- Environment variable handling and default value assignment
- Configuration file generation accuracy
- Container startup with various environment variable combinations
- Backward compatibility with existing functionality

## Environment & Setup
- Docker environment with build capabilities
- Test scenarios with different environment variable combinations
- Configuration file validation tools (YAML parsers)

## Mocking Strategy
- No mocking required - integration testing with real Docker environment
- Use actual container builds and startup sequences
- Test with real configuration file generation

## Key Test Scenarios

### Environment Variable Testing
1. **Default Values Test**:
   - Build and run container without setting new environment variables
   - Verify `LOGGING_FILE_NAME` defaults to `"btc-federation.log"`
   - Verify `LOGGING_FILE_MAX_SIZE` defaults to `"10MB"`

2. **Custom Values Test**:
   - Set `LOGGING_FILE_NAME="custom-test.log"`
   - Set `LOGGING_FILE_MAX_SIZE="1kB"`
   - Verify configuration generation uses custom values

3. **Mixed Scenarios Test**:
   - Test with only one variable set (other should use default)
   - Test with empty string values (should use defaults)
   - Test with various file size formats

### Configuration Generation Testing
1. **YAML Structure Validation**:
   - Verify generated conf.yaml has proper YAML syntax
   - Verify all logging section parameters are present
   - Verify proper indentation and formatting

2. **Parameter Value Validation**:
   - Verify `console_output: true` is set correctly
   - Verify `console_color: true` is set correctly
   - Verify `file_output: true` is set correctly
   - Verify `file_name` matches environment variable value
   - Verify `file_max_size` matches environment variable value
   - Verify existing `level` and `format` parameters preserved

### Integration Testing
1. **Container Startup Testing**:
   - Test container starts successfully with new configuration
   - Verify no errors in container logs during startup
   - Test with btc-federation binary execution

2. **Backward Compatibility Testing**:
   - Verify existing container functionality unchanged
   - Test with existing environment variables
   - Verify no breaking changes to container behavior

## Success Criteria
- All environment variable scenarios work correctly
- Configuration generation produces valid YAML with all required parameters
- Container starts successfully with various environment variable combinations
- No regression in existing container functionality
- Default values work as specified when environment variables not provided

# Verification and Validation

This is a **Simple Task** requiring basic functional validation with focus on configuration accuracy and container integration.

## Architecture integrity
- **Configuration Management**: Dockerfile changes maintain clear separation between environment setup and configuration generation
- **Environment Variable Handling**: Proper validation and default value assignment follows Docker best practices
- **YAML Generation**: Configuration generation maintains proper structure and formatting standards
- **Container Architecture**: Changes integrate seamlessly with existing container startup sequence

## Security
- **Environment Variable Security**: No sensitive data exposure through new environment variables
- **Configuration File Security**: Generated configuration files maintain proper permissions and content security
- **Default Value Security**: Default values do not introduce security vulnerabilities
- **Container Security**: No additional security risks introduced through configuration changes

## Performance
- **Container Startup Impact**: New environment variable processing does not significantly impact container startup time
- **Configuration Generation Efficiency**: YAML generation remains efficient with additional parameters
- **Memory Usage**: No additional memory overhead from new environment variables
- **Build Time Impact**: Dockerfile changes do not significantly increase build time

## Reliability
- **Configuration Consistency**: Generated configuration files are consistent across different environment scenarios
- **Error Handling**: Proper handling of missing or invalid environment variable values
- **Default Value Reliability**: Default values are applied consistently when environment variables not provided
- **Container Stability**: Container startup remains stable with new configuration parameters

## Maintainability
- **Code Organization**: Dockerfile changes are well-structured and clearly documented
- **Configuration Logic**: Environment variable handling logic is easy to understand and modify
- **Documentation**: Clear comments explain new environment variables and their usage
- **Testing Support**: Changes support easy testing of different configuration scenarios

## Cost
- **Resource Efficiency**: No additional resource consumption from new environment variables
- **Build Efficiency**: Dockerfile changes do not increase container image size significantly
- **Development Efficiency**: Implementation is straightforward and does not require complex tooling
- **Maintenance Cost**: Changes are simple and do not increase maintenance burden

## Compliance
- **Docker Best Practices**: Implementation follows Docker environment variable and configuration management best practices
- **YAML Standards**: Generated configuration files comply with YAML formatting standards
- **Project Policies**: All changes follow project policies for configuration management and documentation
- **Container Standards**: Changes maintain compliance with container security and operational standards 