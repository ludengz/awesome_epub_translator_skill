# Installation Requirements

This skill requires `zip` and `unzip` commands available in your shell.

## Platform-specific instructions

### Windows
- **Git Bash** (bundled with Git for Windows): `zip` and `unzip` are included
- **Chocolatey**: `choco install zip unzip`
- **Manual**: Download from GnuWin32 and add to PATH

### macOS
- Pre-installed on all macOS versions

### Linux
- Usually pre-installed. If not:
  - Debian/Ubuntu: `sudo apt install zip unzip`
  - Fedora/RHEL: `sudo dnf install zip unzip`
  - Arch: `sudo pacman -S zip unzip`

## Verification

Run these commands to verify:
```
zip --version
unzip --version
```

Both should print version info without errors.
