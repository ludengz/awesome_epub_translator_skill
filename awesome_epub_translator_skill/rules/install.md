# Installation Requirements

This skill requires `unzip` for extracting ePub files, and either `zip` or Python 3 for repackaging.

## Platform-specific instructions

### Windows
- **Git Bash** (bundled with Git for Windows): `unzip` is included; `zip` may NOT be included
- **Python fallback**: If `zip` is not available, Python 3 is used automatically for repackaging (Python is commonly pre-installed or available via `python` command)
- **Chocolatey** (optional): `choco install zip unzip`

### macOS
- `zip` and `unzip` are pre-installed on all macOS versions

### Linux
- Usually pre-installed. If not:
  - Debian/Ubuntu: `sudo apt install zip unzip`
  - Fedora/RHEL: `sudo dnf install zip unzip`
  - Arch: `sudo pacman -S zip unzip`

## Verification

Run these commands to verify:
```
unzip --version
zip --version || python --version
```

`unzip` is required. For repackaging, either `zip` or Python 3 must be available.
