# Usage Guide

Complete reference for the wp-fix remediation binary.

---

## Installation

Copy the precompiled binary to your server:

```sh
scp wp-fix user@server:/usr/local/bin/
chmod +x /usr/local/bin/wp-fix
```

Precompiled binaries are available for Linux (x86_64), FreeBSD (x86_64), and Windows (x86_64). Download from the [releases page](https://github.com/nestorwheelock/wordpress-security-tools/releases).

### Platform Defaults

| Setting | FreeBSD | Linux | Windows |
|---------|---------|-------|---------|
| WordPress root | `/usr/local/www/wordpress` | `/var/www/html` | `C:\xampp\htdocs\wordpress` |
| wp-cli path | `/usr/local/bin/wp` | `/usr/local/bin/wp` | `C:\wp-cli\wp.bat` |

Override with `--wp-path` and `--wp-cli`.

---

## Commands

### check

Report plugin versions and identify vulnerable ones:

```sh
wp-fix check [--wp-path <PATH>] [--wp-cli <PATH>]
```

Scans installed plugins via `wp plugin list --format=csv` and compares versions against the known-vulnerable list. Reports PASS for safe versions, FAIL for vulnerable versions.

### audit

Scan Elementor widget data for XSS indicators:

```sh
wp-fix audit [--wp-path <PATH>] [--wp-cli <PATH>]
```

Queries `_elementor_data` from the WordPress database and scans for 9 suspicious patterns:

| Pattern | Description |
|---------|-------------|
| `<img` | Image tag injection (beacon/tracking) |
| `<a href=` | Anchor tag injection (phishing) |
| `<iframe` | Iframe injection |
| `<script` | Script tag injection |
| `<svg` | SVG injection |
| `onerror=` | Event handler (XSS trigger) |
| `onload=` | Event handler (XSS trigger) |
| `onclick=` | Event handler (XSS trigger) |
| `javascript:` | Protocol injection |

Pattern matching is case-insensitive.

### update

Update targeted vulnerable plugins:

```sh
wp-fix update [--wp-path <PATH>] [--wp-cli <PATH>]
```

Updates only the plugins in the known-vulnerable list (Essential Addons, Ultimate Member, Elementor). Tries `wp plugin update` first, falls back to direct ZIP download from wordpress.org.

### update-all

Check and update ALL outdated plugins:

```sh
wp-fix update-all [--wp-path <PATH>] [--wp-cli <PATH>]
```

Enumerates all installed plugins, identifies those with available updates, and updates them. Useful for general WordPress maintenance beyond the specific CVEs.

### verify

Confirm updates succeeded:

```sh
wp-fix verify [--wp-path <PATH>] [--wp-cli <PATH>]
```

Checks current version of each vulnerable plugin against the minimum safe threshold. Reports PATCHED, STILL VULNERABLE, or NOT INSTALLED.

### scan

Run the full 112-check security scan:

```sh
wp-fix scan [--wp-path <PATH>] [--url <URL>] [--category <CATEGORY>] [--format <FORMAT>]
```

Scans across all five categories (network, server, plugin, database, environment). Use `--format json` for machine-readable output.

### all

Run the full pipeline:

```sh
wp-fix all [--wp-path <PATH>] [--wp-cli <PATH>]
```

Executes check → audit → update → verify in sequence. Stops on fatal errors but continues through non-fatal issues.

### install-wpcli

Download and install wp-cli:

```sh
wp-fix install-wpcli
```

Downloads wp-cli.phar from the official source and installs it. Useful when wp-fix is deployed to a server that doesn't have wp-cli installed yet.

### help

Print help message:

```sh
wp-fix help
```

---

## Exit Codes

| Code | Meaning | Action |
|------|---------|--------|
| 0 | All checks passed, no vulnerabilities found | No action needed |
| 1 | Vulnerabilities detected or updates applied | Review results, verify fixes |
| 2 | Fatal error (missing wp-cli, invalid path, command failure) | Fix configuration, re-run |

---

## HTTP Download Fallback

When wp-fix needs to download plugin files, it tries system tools in order before falling back to pure Rust:

1. `fetch` (FreeBSD native)
2. `curl`
3. `wget`
4. `ureq` (compiled-in Rust HTTP client)

---

## Examples

### Full Remediation

```sh
# Copy binary to server and run full pipeline
scp wp-fix user@server:/usr/local/bin/
ssh user@server wp-fix all

# Or with custom path
ssh user@server wp-fix all --wp-path /srv/wordpress
```

### Cron Monitoring

```crontab
# Run daily at 6am, log results
0 6 * * * /usr/local/bin/wp-fix check --wp-path /var/www/html >> /var/log/wp-fix.log 2>&1
```

---

## Safety Notes

- wp-fix validates all inputs before constructing shell commands
- wp-fix always adds `--allow-root` to wp-cli invocations (required in jail/container environments)
