# Bluefin LTS

Bluefin LTS is a container-based operating system image built on CentOS Stream 10 using bootc technology. It creates bootable container images that can be converted to disk images, ISOs, and VM images.

Always reference these instructions first and fallback to search or bash commands only when you encounter unexpected information that does not match the info here.

**Lazy-load rule:** The sections below marked with `@file` are reference documents. Read them on demand when the task requires it — do not load them unless relevant.

## Working Effectively

### Prerequisites and Setup
- **CRITICAL**: Ensure `just` command runner is installed.
  Check with `which just`. If missing, install to `~/.local/bin`:
  ```bash
  mkdir -p ~/.local/bin
  wget -qO- "https://github.com/casey/just/releases/download/1.34.0/just-1.34.0-x86_64-unknown-linux-musl.tar.gz" | tar --no-same-owner -C ~/.local/bin -xz just
  export PATH="$HOME/.local/bin:$PATH"
  ```
- Ensure podman is available: `which podman` (should be present)
- Verify git is available: `which git`

### Build Commands - NEVER CANCEL BUILDS
- **Build container image**: `just build [IMAGE_NAME] [TAG] [DX] [GDX] [HWE]`
  - Defaults: `just build` is equivalent to `just build bluefin lts 0 0 0`
  - Takes 45-90 minutes. NEVER CANCEL. Set timeout to 120+ minutes.
  - Example: `just build bluefin lts 0 0 0` (basic build)
  - Example: `just build bluefin lts 1 0 0` (with DX - developer tools)
  - Example: `just build bluefin lts 0 1 0` (with GDX - GPU/AI tools)
- **Build VM images**:
  - `just build-qcow2` - Converts *existing* container image to QCOW2 (45-90 minutes)
  - `just rebuild-qcow2` - Builds container image THEN converts to QCOW2 (90-180 minutes)
  - `just build-iso` - ISO installer image (45-90 minutes)
  - `just build-raw` - RAW disk image (45-90 minutes)
  - NEVER CANCEL any build command. Set timeout to 120+ minutes.

### Validation and Testing
- **ALWAYS run syntax checks before making changes**:
  - `just check` - validates Just syntax (takes <30 seconds)
  - `just lint` - runs shellcheck on all shell scripts (takes <10 seconds)
  - `just format` - formats shell scripts with shfmt (takes <10 seconds)
- **Build validation workflow**:
  1. Always run `just check` before committing changes
  2. Always run `just lint` before committing changes
  3. Test build with `just build bluefin lts` (120+ minute timeout)
  4. Test VM creation with `just build-qcow2` if modifying VM-related code

### Running Virtual Machines
- **Run VM from built images**:
  - `just run-vm-qcow2` - starts QCOW2 VM with web console on http://localhost:8006
  - `just run-vm-iso` - starts ISO installer VM
  - `just spawn-vm` - uses systemd-vmspawn for VM management
- **NEVER run VMs in CI environments** - they require KVM/graphics support

## Repository Structure

### Key Directories
- `build_scripts/` - Build automation and package installation scripts
- `system_files/` - Base system configuration files
- `system_files_overrides/` - Variant-specific overrides (dx, gdx, arch-specific)
- `.github/workflows/` - CI/CD automation (60-minute timeout configured)
- `Justfile` - Primary build automation (13KB+ file with all commands)

### Important Files
- `Containerfile` - Main container build definition
- `image.toml` - VM image build configuration
- `iso.toml` - ISO build configuration
- `Justfile` - Build command definitions (use `just --list` to see all)

## Build System Architecture

> Variants, core build process, timing expectations, common failures, and recovery steps.
> Read on demand: `@file docs/build-architecture.md`

## Common Development Tasks

> How to add packages, modify build scripts, and change system configuration.
> Read on demand: `@file docs/development-tasks.md`

## GitHub Actions CI/CD Architecture

> Workflow roles, branch/tag namespaces, promotion flow, `publish` logic, SBOM rules, signing conditions, and schedule ownership. **Read before touching any workflow file.**
> Read on demand: `@file docs/ci-architecture.md`

## Validation Scenarios

### After Making Changes
1. **Syntax validation**: `just check && just lint`
2. **Build test**: `just build bluefin lts` (full 120+ minute build)
3. **VM test**: `just build-qcow2` (if modifying VM components)
4. **Manual testing**: Run VM and verify basic OS functionality

### Code Quality Requirements
- All shell scripts must pass shellcheck (`just lint`)
- Just syntax must be valid (`just check`)
- CI builds must complete within 60 minutes
- Always test the specific variant you're modifying (dx, gdx, regular)

## Common Commands Reference

```bash
# Essential validation (run before every commit)
just check                    # <30 seconds
just lint                     # <10 seconds

# Core builds (NEVER CANCEL - 120+ minute timeout)
just build bluefin lts        # Standard build
just build bluefin lts 1 0 0  # With DX (developer tools)
just build bluefin lts 0 1 0  # With GDX (GPU/AI tools)

# VM images (NEVER CANCEL - 120+ minute timeout)
just build-qcow2              # QCOW2 VM image
just build-iso                # ISO installer
just build-raw                # Raw disk image

# Development utilities
just --list                   # Show all available commands
just clean                    # Clean build artifacts
git status                    # Check repository state
```

## Critical Reminders

- **NEVER CANCEL builds or long-running commands** - they may take 45-90 minutes
- **ALWAYS set 120+ minute timeouts** for build commands
- **ALWAYS run `just check && just lint`** before committing changes
- **This is an OS image project**, not a traditional application
- **Internet access may be limited** in some build environments
- **VM functionality requires KVM/graphics support** - not available in all CI environments
