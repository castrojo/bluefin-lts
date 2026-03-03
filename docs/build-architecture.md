# Build System Architecture

## Key Build Variants

- **Regular**: Basic Bluefin LTS (`just build bluefin lts 0 0 0`)
- **DX**: Developer Experience with VSCode, Docker, development tools (`just build bluefin lts 1 0 0`)
- **GDX**: GPU Developer Experience with CUDA, AI tools (`just build bluefin lts 0 1 0`)
- **HWE**: Hardware Enablement for newer hardware (`just build bluefin lts 0 0 1`)

## Core Build Process

1. **Container Build**: Uses Containerfile with CentOS Stream 10 base
2. **Build Scripts**: Located in `build_scripts/` directory
3. **System Overrides**: Architecture and variant-specific files in `system_files_overrides/`
4. **Bootc Conversion**: Container images converted to bootable formats via Bootc Image Builder

## Build Timing Expectations

- **Container builds**: 45-90 minutes (timeout: 120+ minutes)
- **VM image builds**: 45-90 minutes (timeout: 120+ minutes)
- **Syntax checks**: <30 seconds
- **Linting**: <10 seconds
- **Git operations**: <5 seconds

## Build Failures and Debugging

### Common Issues

- **Network timeouts**: Build pulls packages from CentOS repositories
- **Disk space**: Container builds require significant space (clean with `just clean`)
- **Permission errors**: Some commands require sudo/root access
- **Missing dependencies**: Ensure just, podman, git are installed

### Recovery Steps

1. Clean build artifacts: `just clean`
2. Verify tools: `which just podman git`
3. Check syntax: `just check && just lint`
4. Retry with full timeout: `just build bluefin lts` (120+ minutes)

Never attempt to fix builds by canceling and restarting - let them complete or fail naturally.
