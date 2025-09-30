# STARTUP-1 - Bitcoin Fastnet Local Node Setup

# Links
- [01 Taproot Multisig](/workflow/prd/federation/01_taproot_multisig.md)

# Description
Deploy a local Bitcoin regtest node with fast block generation (~10 seconds) for development. Includes node setup, miner configuration, and RPC interface.

# Requirements and DOD

## Core Requirements
- Bitcoin Core node installed in `.btc/node/` project directory
- Node operates in regtest mode (isolated local network)
- RPC interface accessible only on `127.0.0.1`
- Miner configured with minimal system load
- Blocks generated every ~10 seconds
- Solution named "fastnet"
- Env setup must be reproducible: same flow each time, same miner keys and address, etc.

## File System Layout
- Install Bitcoin Core node in `.btc/node/` directory within project
- Makefile placed in project root directory

## Makefile Commands
Create Makefile in project root with:
- `make btc-start` - start node
- `make btc-stop` - stop node
- `make btc-reset` - restart with new genesis block
- `make btc-miner-start` - start miner
- `make btc-miner-stop` - stop miner

## Documentation Requirements
Create instructions for:
- Wallet creation
- Transaction execution
- Balance checking
- Transaction status verification

## Definition of Done
- Node starts from genesis block
- RPC responds on 127.0.0.1 only
- Miner generates blocks at ~10 second intervals
- All Makefile commands work
- Documentation tested and verified
- Miner CPU usage <5% during normal operation
- Support only Linux OS (Ubuntu)

# Implementation Plan

## Phase 1: Setup and Configuration
1. Download and install Bitcoin Core to `.btc/node/`
2. Create bitcoin.conf for regtest mode with localhost-only RPC
3. Configure fast block generation parameters
4. Test node startup and RPC connectivity

## Phase 2: Mining and Automation
1. Configure CPU miner with minimal resource usage (1-2 threads)
2. Create Makefile with all required commands
3. Implement genesis block reset functionality
4. Test all automation commands

## Phase 3: Documentation and Validation
1. Create user interaction documentation
2. Test all documented procedures
3. Verify performance requirements (<5% CPU)
4. Validate security (localhost-only access)

# Test Plan

## Key Test Scenarios
- Node starts successfully and responds to RPC commands
- Block generation maintains ~10 second intervals
- CPU usage remains below 5% during operation
- All Makefile commands execute correctly
- Documentation procedures work as described
- RPC interface only accessible from localhost

# Verification and Validation

## Security
- Node binds only to localhost (127.0.0.1)
- No external network connections in regtest mode

## Architecture
- File system layout follows project structure (.btc/node/ directory)
- Configuration separates node and miner concerns
- Makefile commands follow consistent naming