# 02-1 - Dockerfile OpenSSL 3.5+ Integration

# Links
- [PRD-02: SPHINCS+ Compatibility Update](../../prd/vtcpd-test-suite/02-sphincs-plus-compatibility.md)

# Description
Update Dockerfile to include OpenSSL 3.5+ libraries in both Ubuntu and Manjaro container variants to support SPHINCS+ cryptographic operations. The containers must have OpenSSL 3.5+ available at runtime for the VTCPD binary to function correctly with SPHINCS+ implementation.

# Requirements and DOD

## Requirements
1. **Ubuntu Container OpenSSL Integration**:
   - Add OpenSSL 3.5+ runtime and development libraries to Ubuntu 24.04 container
   - Use appropriate apt packages: `libssl3`, `libssl-dev`, `openssl`
   - Ensure compatibility with existing boost and postgresql libraries
   - Verify no package conflicts during installation

2. **Manjaro Container OpenSSL Integration**:
   - Add OpenSSL 3.5+ packages to Manjaro container using pacman
   - Use appropriate packages: `openssl` (latest available version ≥3.5)
   - Ensure compatibility with existing boost and postgresql libraries
   - Verify no package conflicts during installation

3. **Container Build Verification**:
   - Both container variants must build successfully without errors
   - OpenSSL version must be 3.5+ when queried inside container
   - SPHINCS+ algorithm EVP_SIGNATURE-SLH-DSA-SHA2-256s must be available
   - Container size increase should be reasonable (document final sizes)

4. **Backwards Compatibility**:
   - All existing functionality must continue to work
   - No breaking changes to container startup scripts
   - Environment variables and configuration remain unchanged
   - Existing library dependencies preserved

## Definition of Done
- [ ] Ubuntu container includes OpenSSL 3.5+ packages
- [ ] Manjaro container includes OpenSSL 3.5+ packages  
- [ ] Both containers build successfully without errors
- [ ] OpenSSL version verification shows ≥3.5 in both containers
- [ ] SPHINCS+ algorithm availability confirmed in both containers
- [ ] Container size impact documented
- [ ] No regression in existing container functionality
- [ ] Updated Dockerfile passes linting/validation

# Implementation Plan

## Step 1: Analyze Current Dockerfile Structure
- Review existing multi-stage Dockerfile architecture
- Identify Ubuntu and Manjaro runtime stages that need OpenSSL
- Document current package installation patterns
- Check for any existing OpenSSL references

## Step 2: Research Package Requirements
- Verify OpenSSL 3.5+ package availability in Ubuntu 24.04 repositories
- Verify OpenSSL 3.5+ package availability in Manjaro repositories
- Identify exact package names and dependencies
- Check for any PPA or additional repository requirements

## Step 3: Update Ubuntu Runtime Stage
- Modify `runtime-ubuntu` stage to include OpenSSL packages
- Add packages to existing `apt-get install` command
- Ensure proper package cleanup with `rm -rf /var/lib/apt/lists/*`
- Test build isolation for Ubuntu variant

## Step 4: Update Manjaro Runtime Stage
- Modify `runtime-manjaro` stage to include OpenSSL packages
- Add packages to existing `pacman -S` command
- Ensure compatibility with existing package installation
- Test build isolation for Manjaro variant

## Step 5: Add OpenSSL Verification
- Add verification commands to confirm OpenSSL version during build
- Include SPHINCS+ algorithm availability check
- Document verification output in build logs
- Add debugging information for troubleshooting

## Step 6: Test Container Builds
- Build both container variants separately
- Verify successful builds without errors
- Test container startup and basic functionality
- Document container sizes before/after changes

## Step 7: Integration Testing
- Test containers with existing VTCPD binary
- Verify SPHINCS+ cryptographic operations work
- Run basic smoke tests for both database variants
- Validate no regression in existing functionality

# Test Plan

## Unit Testing
- **Container Build Tests**: Verify both Ubuntu and Manjaro containers build successfully
- **Package Installation Tests**: Confirm OpenSSL packages are correctly installed
- **Version Verification Tests**: Validate OpenSSL version is 3.5+ in both containers

## Integration Testing
- **VTCPD Binary Compatibility**: Test that VTCPD binary can load and use OpenSSL 3.5+
- **SPHINCS+ Algorithm Tests**: Verify EVP_SIGNATURE-SLH-DSA-SHA2-256s is available
- **Database Connectivity Tests**: Ensure PostgreSQL and SQLite functionality unchanged
- **Container Startup Tests**: Verify containers start correctly with new OpenSSL

## Regression Testing
- **Existing Functionality**: All current container features must continue working
- **Environment Variables**: Verify all existing env vars work correctly
- **Volume Mounts**: Ensure file system mounts work as before
- **Network Connectivity**: Confirm container networking unchanged

## Performance Testing
- **Build Time Impact**: Measure and document build time changes
- **Container Size Impact**: Document size increase from OpenSSL packages
- **Runtime Performance**: Verify no significant performance degradation

## Test Environment Setup
```bash
# Test Ubuntu container build
docker build --target final-ubuntu -t vtcpd-test-ubuntu .

# Test Manjaro container build  
docker build --target final-manjaro -t vtcpd-test-manjaro .

# Verify OpenSSL version in Ubuntu container
docker run --rm vtcpd-test-ubuntu openssl version

# Verify OpenSSL version in Manjaro container
docker run --rm vtcpd-test-manjaro openssl version

# Test SPHINCS+ algorithm availability
docker run --rm vtcpd-test-ubuntu openssl list -signature-algorithms | grep -i sphincs
```

## Success Criteria
- Both containers build without errors or warnings
- OpenSSL version output shows 3.5+ in both containers
- SPHINCS+ algorithms are listed in available signature algorithms
- Container size increase is reasonable (≤50MB per container)
- All existing tests pass with new containers

# Verification and Validation

## Architecture integrity
- **Multi-stage Build**: Preserve existing multi-stage Dockerfile architecture
- **Container Separation**: Maintain clear separation between Ubuntu and Manjaro variants
- **Dependency Management**: Follow existing patterns for package installation and cleanup
- **Build Optimization**: Maintain efficient layer caching and minimize image size

## Security
- **Package Sources**: Use official Ubuntu and Manjaro repositories only
- **Package Verification**: Rely on package manager signature verification
- **Minimal Attack Surface**: Install only required OpenSSL components
- **No Privilege Escalation**: Maintain existing user permissions and security model

## Performance
- **Build Time**: Document build time impact (target: ≤20% increase)
- **Image Size**: Document size impact (target: ≤50MB increase per variant)
- **Runtime Performance**: No significant performance degradation in cryptographic operations
- **Memory Usage**: OpenSSL memory footprint should be reasonable for container environment

## Scalability
- **Multi-Container**: Changes must work in multi-container test scenarios
- **Parallel Builds**: Support concurrent building of both container variants
- **CI/CD Integration**: Compatible with existing build pipelines
- **Resource Usage**: Reasonable resource consumption during builds

## Reliability
- **Consistent Builds**: Reproducible builds across different environments
- **Dependency Stability**: Use stable package versions to avoid breaking changes
- **Error Handling**: Proper error handling during package installation
- **Rollback Capability**: Changes can be easily reverted if issues arise

## Maintainability
- **Code Clarity**: Clear, well-documented Dockerfile changes
- **Package Management**: Follow best practices for package installation and cleanup
- **Version Pinning**: Consider pinning OpenSSL version for stability
- **Documentation**: Update relevant documentation and comments

## Cost
- **Build Resources**: Minimal impact on build server resources
- **Storage**: Reasonable increase in image registry storage requirements
- **Bandwidth**: Consider impact on image download times
- **Development Time**: Estimated 1-2 days for implementation and testing

## Compliance
- **License Compatibility**: OpenSSL license compatible with project requirements
- **Security Standards**: Meet any applicable security compliance requirements
- **Container Standards**: Follow Docker best practices and security guidelines
- **Documentation Standards**: Maintain documentation quality and completeness