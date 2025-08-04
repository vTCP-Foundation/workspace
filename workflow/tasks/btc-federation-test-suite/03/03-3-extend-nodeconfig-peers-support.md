# 03-3 - Extend NodeConfig for peers.yaml Support

# Links
- [PRD](../../prd/btc-federation-test-suite/03-p2p-connection-testing.md)

# Description
Extend NodeConfig structure and related functionality to support peers.yaml configuration generation through environment variables. This enables dynamic peer configuration for testing different network topologies.

# Requirements and DOD
## Requirements
- Add peers configuration fields to NodeConfig struct
- Implement methods to generate peers.yaml environment variables
- Support structured peer configuration (public keys, addresses, connection parameters)
- Maintain backward compatibility with existing NodeConfig usage
- Integration with existing environment variable generation system
- Support for multiple peer configurations per node

## Definition of Done
- [ ] NodeConfig struct extended with peers configuration fields
- [ ] `PeerConfig` struct defined for individual peer settings
- [ ] Methods for environment variable generation for peers
- [ ] Integration with existing `GetEnvironmentVariables()` method
- [ ] Validation of peer configuration data
- [ ] Unit tests for new configuration structures and methods
- [ ] Integration tests with environment variable generation
- [ ] Code documentation updated
- [ ] No breaking changes to existing NodeConfig functionality

# Implementation Plan
## Step 1: Define peer configuration structures
- Create `PeerConfig` struct for individual peer settings
  - PublicKey field
  - Addresses slice
  - Additional connection parameters
- Add `Peers` field to NodeConfig
- Define validation methods for peer data

## Step 2: Extend environment variable generation
- Modify `GetEnvironmentVariables()` method to include peers data
- Implement `generatePeersYAMLContent()` helper method
- Convert peer configuration to YAML format
- Add `PEERS_YAML_CONTENT` environment variable

## Step 3: Configuration validation
- Implement peer configuration validation
- Check for required fields (PublicKey, Addresses)
- Validate address formats
- Handle empty or invalid configurations

## Step 4: Integration with existing patterns
- Ensure consistency with current NodeConfig patterns
- Maintain existing method signatures
- Follow established validation approaches

## Step 5: Testing and validation
- Unit tests for peer configuration structures
- Tests for YAML generation from peer configs
- Integration tests with environment variable system
- Validation of generated YAML format

# Test Plan
## Objective
Verify that NodeConfig correctly supports peers configuration and generates proper environment variables for peers.yaml creation.

## Test Scope
- `PeerConfig` struct functionality
- NodeConfig extension with peers support
- Environment variable generation for peers
- YAML format generation and validation
- Integration with existing configuration patterns

## Environment & Setup
- Go test environment for struct and method testing
- Mock peer configuration data for testing
- YAML validation tools for format verification

## Mocking Strategy
- Mock peer configuration data for various scenarios
- Use test structs for validation testing
- No external dependencies to mock

## Key Test Scenarios
1. **Configuration Creation**:
   - Create NodeConfig with single peer
   - Create NodeConfig with multiple peers
   - Handle empty peers configuration
   
2. **Environment Variable Generation**:
   - Generate PEERS_YAML_CONTENT from peer configs
   - Validate generated YAML format
   - Test integration with existing environment variables

3. **Validation**:
   - Valid peer configurations
   - Invalid peer configurations (missing fields, bad formats)
   - Empty or nil peer configurations

4. **Backward Compatibility**:
   - Existing NodeConfig usage without peers
   - Environment variable generation without peers
   - Default values and optional fields

## Success Criteria
- All unit tests pass
- Generated YAML content is valid and well-formed
- Environment variable integration works seamlessly
- No breaking changes to existing functionality
- Peer configuration validation works correctly

# Verification and Validation

## Architecture integrity
- Extension follows existing NodeConfig patterns
- New structures integrate cleanly with existing code
- No modifications to existing public interfaces
- Consistent naming conventions and method signatures

## Security
- No security implications for configuration extension
- Peer configuration data handled securely
- No exposure of sensitive data in environment variables

## Performance
- Configuration generation completes quickly (<1 second)
- No performance degradation in existing functionality
- Efficient YAML generation for peer configurations

## Scalability
- Support for reasonable number of peers per node
- Configuration structure scales with peer count
- Environment variable size within practical limits

## Reliability
- Robust validation prevents invalid configurations
- Graceful handling of missing or malformed peer data
- Consistent behavior across different usage patterns

## Maintainability
- Clear structure definitions with appropriate documentation
- Extensible design for future peer configuration needs
- Well-organized code following project conventions

## Cost
- No additional runtime costs
- Minimal memory overhead for peer configuration storage

## Compliance
- Follows Go struct and method naming conventions
- Adheres to project coding standards
- Consistent with existing configuration patterns 