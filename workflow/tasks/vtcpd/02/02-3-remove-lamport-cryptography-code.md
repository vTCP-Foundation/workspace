# 02-3 - Remove Lamport Cryptography Code

# Links
- [PRD-02: SPHINCS+ Cryptography Implementation](../../prd/vtcpd/02_sphincs_plus_cryptography_implementation.md)
- [Task 02-2: Keychain Classes Migration to SPHINCS+](./02-2-keychain-classes-migration-to-sphincs.md)

# Description
Remove all Lamport signature scheme related code, classes, files, and dependencies from the project. This task involves cleaning up the codebase after the migration to SPHINCS+ is complete, removing obsolete cryptographic implementations and their dependencies.

# Requirements and DOD

## Requirements
1. **Remove Lamport Source Files**:
   - Delete `src/core/crypto/lamportscheme.h`
   - Delete `src/core/crypto/lamportscheme.cpp`
   - Delete `src/core/crypto/lamportkeys.h`
   - Delete `src/core/crypto/lamportkeys.cpp`
   - Remove any other Lamport-related files in crypto directory

2. **Update Build System**:
   - Remove Lamport source files from `src/core/crypto/CMakeLists.txt`
   - Remove any Lamport-specific build targets or dependencies
   - Update compilation rules to exclude deleted files
   - Ensure project builds successfully without Lamport files

3. **Remove Lamport Dependencies**:
   - Remove libsodium dependency (if only used for Lamport)
   - Update CMakeLists.txt to remove sodium linking
   - Remove any Lamport-specific library dependencies
   - Clean up package requirements

4. **Clean Up Remaining References**:
   - Search and remove any remaining `#include` statements for Lamport headers
   - Remove any unused Lamport namespace references
   - Clean up any Lamport-related constants or definitions
   - Remove Lamport-related comments and documentation

## Definition of Done (DOD)
- [ ] All Lamport source files deleted from the project
- [ ] CMakeLists.txt updated to remove Lamport file references
- [ ] libsodium dependency removed (if applicable)
- [ ] No remaining `#include` references to deleted Lamport files
- [ ] Project compiles successfully without Lamport code
- [ ] No unused Lamport namespace references remain
- [ ] All Lamport-related build targets removed
- [ ] Code follows project coding standards and style guidelines
- [ ] Changes properly documented

# Implementation Plan

## Phase 1: Dependency Analysis (Day 1)
1. **Identify Lamport Dependencies**:
   - Analyze which files depend on Lamport headers
   - Check if libsodium is used only for Lamport operations
   - Identify any Lamport-specific build configurations
   - Document all files and dependencies to be removed

2. **Verify Migration Completion**:
   - Confirm Task 02-2 (Keychain Migration) is completed
   - Ensure no active code still uses Lamport primitives
   - Check that all keychain classes use SPHINCS+ exclusively

## Phase 2: File and Dependency Removal (Day 2)
1. **Remove Lamport Source Files**:
   - Delete `lamportscheme.h` and `lamportscheme.cpp`
   - Delete `lamportkeys.h` and `lamportkeys.cpp`
   - Remove any other Lamport-related files
   - Update file organization if needed

2. **Update Build System**:
   - Remove Lamport files from CMakeLists.txt
   - Remove libsodium dependency if no longer needed
   - Update build targets and linking rules
   - Clean up any Lamport-specific build configurations

## Phase 3: Cleanup and Validation (Day 3)
1. **Clean Up References**:
   - Search for and remove any remaining Lamport includes
   - Remove unused namespace references
   - Clean up any leftover Lamport-related constants
   - Remove obsolete comments and documentation

2. **Build Validation**:
   - Verify project compiles successfully
   - Check that no undefined symbols remain
   - Test basic functionality to ensure nothing is broken
   - Validate that SPHINCS+ operations work correctly

# Test Plan

**Note**: This task does not include test implementation. Tests will be implemented in a separate dedicated testing task.

## Test Objectives
Verify that Lamport code removal does not break the project and all functionality remains intact with SPHINCS+.

## Test Scope
- Project compilation after file removal
- Basic functionality verification
- SPHINCS+ operations still work correctly
- No missing dependencies or undefined symbols

## Manual Validation
- Verify project builds without errors
- Check that no Lamport references remain in code
- Validate that SPHINCS+ keychain operations work
- Ensure all tests still pass (if any exist)

# Verification and Validation

## Architecture integrity
- Verify removal of Lamport code doesn't affect system architecture
- Confirm SPHINCS+ integration remains intact
- Validate clean separation between crypto implementations

## Security
- Ensure no security regressions from code removal
- Verify SPHINCS+ operations maintain security standards
- Confirm no sensitive data exposed during cleanup

## Performance
- Verify no performance degradation from dependency removal
- Check that build times improve without unused code
- Validate SPHINCS+ operations maintain expected performance

## Scalability
- Ensure system scalability is not affected by code removal
- Verify clean codebase supports future enhancements
- Check that dependency reduction improves maintainability

## Reliability
- Verify system reliability is maintained after cleanup
- Test that no critical functionality depends on removed code
- Ensure error handling remains robust

## Maintainability
- Confirm codebase is cleaner and more maintainable
- Verify documentation reflects current implementation
- Check that removed dependencies reduce maintenance burden

## Cost
- Assess reduction in build complexity and dependencies
- Evaluate maintenance cost savings from code removal
- Monitor any impact on development workflow

## Compliance
- Ensure compliance requirements are still met
- Verify no regulatory dependencies on removed code
- Check that security standards are maintained

# Restrictions
- Make commits only after successfully executing the demo (if it is in the task) or after successfully passing the tests (if they are provided for by the task)
- Do not remove any code until Task 02-2 (Keychain Migration) is fully completed and verified
- Preserve any shared dependencies that might be used by other parts of the system
- Follow secure coding practices when handling code removal