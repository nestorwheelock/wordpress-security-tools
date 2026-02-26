# Architecture

## Overview

The project has two components that serve two purposes: deploying vulnerable WordPress instances for security education, and remediating those vulnerabilities in production.

```
  Lab Setup:        modules/wordpress/
                    (deploys vulnerable WP in FreeBSD jail)

  Remediation:      wp-fix binary
  (native binary)   (Linux/FreeBSD/Windows)

  Managed:          wp-fix deployed through proprietary automation tooling
                    (remote execution, no SSH required)
```

## Why Two Modes

The **native binary** is for direct use: copy it to a server, run it, get results. It's the simplest deployment path — one file, no dependencies, works on cron.

The **managed mode** uses our proprietary automation platform to run the same remediation logic remotely without SSH access. Execution is sandboxed, and the platform handles file I/O, command execution, and result collection.

Both share the same vulnerability definitions, version comparison logic, and remediation strategies. The difference is the execution boundary.

## Component Architecture

### WordPress Lab Module

```
modules/wordpress/
├── main.rs          Entry point + command dispatch (install|status|verify|info)
├── args.rs          Argument parsing with injection validation
├── configs.rs       Config generators (nginx, PHP-FPM, MariaDB, wp-config.php)
├── constants.rs     Package names, paths, service names, credentials, plugin URLs
├── cve_info.rs      CVE details, status/verify command handlers
├── generators.rs    6-phase installation orchestration
└── security.rs      Input validation (identifiers, passwords, URLs)
```

The module generates shell commands as instruction sets rather than executing them directly. This separation makes the installation logic testable without a live FreeBSD system.

#### Installation Phases

| Phase | Name | Operations |
|-------|------|------------|
| 0 | packages | `pkg install` MariaDB 11.4, PHP 8.2 + 13 extensions, nginx, utilities |
| 1 | mariadb | Enable service, create database and user, grant privileges |
| 2 | services | Write PHP-FPM pool config, nginx site config, PHP ini overrides, start services |
| 3 | wordpress | Download WordPress 6.9.1 and wp-cli, run `wp core install` |
| 4 | plugins | Download and activate vulnerable plugin versions (UM 2.11.1, EA 6.5.9) |
| 5 | finalize | Create contributor test account, disable auto-updates, print summary |

Each phase checks previous phase results before proceeding.

### wp-fix Tool

```
wp-fix/
├── main.rs          CLI argument parsing + command dispatch
├── check.rs         Plugin version scanning against vulnerable list
├── audit.rs         Elementor XSS payload detection in database
├── update.rs        Plugin update via wp-cli or direct download
├── verify.rs        Post-update version verification
├── http.rs          HTTP client with cascading fallback (fetch→curl→wget→ureq)
├── output.rs        ANSI colored terminal output
└── wp_cli.rs        wp-cli wrapper (preflight, exec, plugin operations, DB queries)
```

**Platforms:** Linux, FreeBSD, Windows
**Dependency:** `ureq` (pure Rust HTTP client, used as final fallback)

#### Command Pipeline

```
check ─────▶ audit ─────▶ update ─────▶ verify
  │            │             │             │
  │ Scan       │ Scan        │ Update      │ Confirm
  │ plugin     │ Elementor   │ vulnerable  │ patched
  │ versions   │ JSON for    │ plugins     │ versions
  │ against    │ XSS         │ via wp-cli  │ installed
  │ known-     │ patterns    │ or direct   │
  │ vulnerable │ (9 types)   │ download    │
  │ list       │             │             │
  ▼            ▼             ▼             ▼
Exit 0/1     Exit 0/1     Exit 0/1/2    Exit 0/1

           `wp-fix all` runs this entire pipeline
```

#### HTTP Fallback Chain

wp-fix needs to download plugin ZIPs when wp-cli update fails. Different platforms have different HTTP tools available:

```
FreeBSD:  fetch (native) → curl → wget → ureq (Rust)
Linux:    curl → wget → ureq (Rust)
Windows:  ureq (Rust)
```

The `ureq` crate is the final fallback — a pure Rust HTTP client that works everywhere without system dependencies. This is the only external dependency.

## Cross-Compilation

### Build Targets

| Target | Use |
|--------|-----|
| `x86_64-unknown-linux-gnu` | wp-fix native Linux binary |
| `x86_64-unknown-freebsd` | wp-fix native FreeBSD binary |
| `x86_64-pc-windows-gnu` | wp-fix native Windows binary |

### Platform Defaults

wp-fix selects platform-appropriate defaults at compile time using `#[cfg(target_os)]`:

| Setting | FreeBSD | Linux | Windows |
|---------|---------|-------|---------|
| WordPress root | `/usr/local/www/wordpress` | `/var/www/html` | `C:\xampp\htdocs\wordpress` |
| wp-cli path | `/usr/local/bin/wp` | `/usr/local/bin/wp` | `C:\wp-cli\wp.bat` |

Both paths are overridable via `--wp-path` and `--wp-cli` arguments.

## Dependency Graph

```
wp-fix
└── ureq 3 (crates.io)
    └── Pure Rust HTTP client (TLS via rustls)
```

Everything else uses the Rust standard library.

## Data Flow

### Vulnerability Check

```
wp-cli                  wp-fix                    Output
──────                  ──────                    ──────
plugin list --csv  ──▶  parse_plugin_csv()   ──▶  Plugin versions
                        version_gte() against     PASS/FAIL per plugin
                        VULNERABLE_PLUGINS        Exit code 0 or 1
```

### XSS Audit

```
wp-cli                  wp-fix                    Output
──────                  ──────                    ──────
db query (postmeta) ──▶ extract_subtitles()  ──▶  Subtitle values
                        detect_suspicious()       Pattern matches
                        (9 XSS patterns)          PASS/FAIL per post
```

### Managed Remediation

For Tier 4 clients, wp-fix is deployed through our proprietary automation platform. The binary runs in a sandboxed environment on the target server. The platform handles scheduling, result collection, and automatic snapshots before every change. No SSH access is required.
