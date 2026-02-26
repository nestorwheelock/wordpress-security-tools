# Releases

Precompiled wp-fix binaries are available on the [GitHub Releases](https://github.com/nestorwheelock/wordpress-security-tools/releases) page.

## Latest: v0.1.0

| Platform | Binary | Download |
|----------|--------|----------|
| Linux x86_64 | `wp-fix-linux-x86_64` | [Download](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-linux-x86_64) |
| FreeBSD x86_64 | `wp-fix-freebsd-x86_64` | [Download](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-freebsd-x86_64) |
| Windows x86_64 | `wp-fix-windows-x86_64.exe` | [Download](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-windows-x86_64.exe) |
| Checksums | `SHA256SUMS` | [Download](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/SHA256SUMS) |

## Installation

### Linux / FreeBSD

```sh
# Download
curl -LO https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-linux-x86_64

# Verify checksum
curl -LO https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/SHA256SUMS
sha256sum -c SHA256SUMS

# Make executable and install
chmod +x wp-fix-linux-x86_64
sudo mv wp-fix-linux-x86_64 /usr/local/bin/wp-fix
```

For FreeBSD, replace `wp-fix-linux-x86_64` with `wp-fix-freebsd-x86_64`.

### Windows

Download `wp-fix-windows-x86_64.exe` from the link above and run from PowerShell or Command Prompt:

```powershell
.\wp-fix-windows-x86_64.exe all --wp-path C:\xampp\htdocs\wordpress
```

## Verify Checksums

Every release includes a `SHA256SUMS` file. Always verify before deploying to production:

```sh
sha256sum -c SHA256SUMS
```
