# Common Development Tasks

## Making Changes to Build Scripts

1. Edit files in `build_scripts/` for package changes
2. Edit `system_files_overrides/[variant]/` for variant-specific changes
3. Always run `just lint` before committing
4. Test with full build: `just build bluefin lts` (120+ minute timeout)

## Adding New Packages

- Edit `build_scripts/20-packages.sh` for base packages
- Use variant-specific overrides in `build_scripts/overrides/[variant]/`
- Use architecture-specific overrides in `build_scripts/overrides/[arch]/`
- Use combined overrides in `build_scripts/overrides/[arch]/[variant]/`
- Package installation uses dnf/rpm package manager

## Modifying System Configuration

- Base configs: `system_files/`
- Variant configs: `system_files_overrides/[variant]/`
- Architecture-specific: `system_files_overrides/[arch]/`
- Combined: `system_files_overrides/[arch]-[variant]/`
