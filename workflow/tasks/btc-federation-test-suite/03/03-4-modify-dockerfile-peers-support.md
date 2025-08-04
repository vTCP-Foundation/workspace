# 03-4 - Modify Dockerfile for peers.yaml Support

# Links
- [PRD](../../prd/btc-federation-test-suite/03-p2p-connection-testing.md)

# Description
Modify the existing Dockerfile to support peers.yaml file generation from environment variables. This enables dynamic peer configuration during container startup while maintaining compatibility with existing configuration generation patterns.

# Requirements and DOD
## Requirements
- Add `PEERS_YAML_CONTENT` environment variable to Dockerfile
- Implement peers.yaml generation script integrated with existing startup process
- Maintain compatibility with existing conf.yaml generation
- Handle empty or missing peers configuration gracefully
- Generate peers.yaml in the same directory as conf.yaml
- Follow existing environment variable substitution patterns

## Definition of Done
- [x] `PEERS_YAML_CONTENT` environment variable added to Dockerfile
- [x] Peers.yaml generation integrated into existing startup script
- [x] Generation script handles empty peers configuration
- [x] Generated peers.yaml follows proper YAML format
- [x] Integration with existing configuration generation process
- [x] No breaking changes to existing container startup
- [x] Documentation updated for new environment variable
- [x] Container startup tested with and without peers configuration

# Implementation Plan
## Step 1: Add environment variable
- Add `PEERS_YAML_CONTENT` environment variable to Dockerfile
- Set default empty value to ensure variable exists
- Document variable purpose and usage

## Step 2: Modify generation script
- Update `/generate-config.sh` to handle peers configuration
- Add peers.yaml generation logic after conf.yaml generation
- Use environment variable substitution pattern like conf.yaml

## Step 3: Peers.yaml generation logic
- Check if `PEERS_YAML_CONTENT` is set and non-empty
- Generate peers.yaml file when content is provided
- Skip generation when content is empty (graceful handling)
- Use proper file permissions and location

## Step 4: Integration with startup process
- Ensure peers.yaml is generated before btc-federation-node starts
- Add logging for peers.yaml generation process
- Handle generation errors gracefully

## Step 5: Testing and validation
- Test container startup with peers configuration
- Test container startup without peers configuration
- Validate generated peers.yaml format
- Verify btc-federation-node can read generated file

# Test Plan
## Objective
Verify that Dockerfile modifications enable proper peers.yaml generation while maintaining existing functionality and container startup reliability.

## Test Scope
- Environment variable handling in Dockerfile
- Peers.yaml generation script functionality
- Integration with existing configuration generation
- Container startup with various configuration scenarios
- Error handling for malformed or missing peers data

## Environment & Setup
- Docker build environment
- Test containers with various environment variable configurations
- YAML validation tools for generated file verification

## Mocking Strategy
- Use test environment variables for different scenarios
- Mock various peers configuration content
- Test with empty and malformed environment variables

## Key Test Scenarios
1. **Container Startup Scenarios**:
   - Container with peers configuration provided
   - Container without peers configuration
   - Container with empty peers configuration
   - Container with malformed peers configuration

2. **File Generation**:
   - Verify peers.yaml is created when content is provided
   - Verify no peers.yaml is created when content is empty
   - Validate generated YAML format and content
   - Check file permissions and location

3. **Integration**:
   - Both conf.yaml and peers.yaml generated correctly
   - Startup process handles both configurations
   - btc-federation-node can access both files

4. **Error Handling**:
   - Malformed PEERS_YAML_CONTENT environment variable
   - File system errors during generation
   - Startup continues even if peers.yaml generation fails

## Success Criteria
- Container builds successfully with Dockerfile modifications
- Peers.yaml generation works correctly when peers configured
- Container startup works with and without peers configuration
- No breaking changes to existing functionality
- Generated peers.yaml files are valid YAML format

# Verification and Validation

## Architecture integrity
- Modifications follow existing Dockerfile patterns
- Integration with existing startup script is clean
- No changes to container base image or core functionality
- Consistent with existing environment variable usage

## Security
- No security vulnerabilities introduced
- Environment variables handled securely
- Generated files have appropriate permissions
- No exposure of sensitive peer data

## Performance
- Container startup time not significantly impacted
- File generation completes quickly
- No resource overhead from peers.yaml generation

## Scalability
- Environment variable size reasonable for typical peer configurations
- Generation script handles varying peer configuration sizes
- No limitations on number of peers within practical bounds

## Reliability
- Container startup is reliable with various configuration scenarios
- Graceful handling of missing or malformed peers configuration
- Error scenarios don't prevent container from starting
- Generated files are consistent and properly formatted

## Maintainability
- Script modifications are clear and well-documented
- Integration with existing patterns is clean
- Future extensions to peer configuration are supported
- Debugging and troubleshooting are straightforward

## Cost
- No additional infrastructure or runtime costs
- Minimal impact on container image size
- No additional dependencies required

## Compliance
- Follows Docker best practices
- Adheres to existing Dockerfile patterns and conventions
- Environment variable naming consistent with project standards

# Implementation Results

## Changes Made
1. **Environment Variable**: Added `PEERS_YAML_CONTENT=""` environment variable to Dockerfile
2. **Generate Script Enhancement**: Modified `/generate-config.sh` to include peers.yaml generation logic:
   - Checks if `PEERS_YAML_CONTENT` is provided and non-empty
   - Generates `/btc-federation/peers.yaml` when content is available
   - Skips generation gracefully when no peers configured
   - Provides appropriate logging for both scenarios
3. **Startup Script Enhancement**: Updated `/start-node.sh` to log peers configuration status
4. **Integration**: Peers.yaml generation integrated seamlessly with existing conf.yaml generation

## Testing Results
All test scenarios passed successfully:

### ✅ Dockerfile Build
- Dockerfile builds successfully with new environment variable
- No breaking changes to existing functionality
- Binary and scripts created correctly

### ✅ Script Functionality
- `generate-config.sh` contains proper PEERS_YAML_CONTENT handling
- Peers.yaml generation logic correctly implemented
- Script works with and without peers configuration

### ✅ Configuration Generation
- **With peers**: peers.yaml created with correct YAML format and content
- **Without peers**: peers.yaml correctly not generated, appropriate message logged
- conf.yaml generation unaffected in both scenarios

### ✅ Container Integration
- Container startup works correctly with peers configuration
- Container startup works correctly without peers configuration
- Startup script shows appropriate peers status: `[PROVIDED]` or `[NOT PROVIDED]`
- Configuration files generated in correct location (`/btc-federation/`)

### ✅ Backward Compatibility
- Existing containers without PEERS_YAML_CONTENT work unchanged
- No impact on existing environment variables or functionality
- All existing Dockerfile patterns preserved

## Generated Files Example
When `PEERS_YAML_CONTENT` is provided, the generated `peers.yaml` follows this format:
```yaml
peers:
- public_key: test-peer-key
  addresses:
  - /ip4/192.168.1.101/tcp/9001
  connection_timeout: 10s
  max_retries: 3
```

## Usage Documentation
To use peers configuration in containers:
```bash
docker run -e PEERS_YAML_CONTENT="peers:
- public_key: peer1-key
  addresses:
  - /ip4/192.168.1.101/tcp/9001" btc-federation-node
```

Task 03-4 completed successfully with all requirements met and full backward compatibility maintained.