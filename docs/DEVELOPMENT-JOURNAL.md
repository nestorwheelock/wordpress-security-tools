# Development Journal

The iterative development story of the WordPress Security Tools project, from passive reconnaissance to cross-platform remediation toolchain.

---

## Phase 1: Lab Setup

### The Problem

A passive review of a production WordPress site's HTML source revealed plugin version strings in CSS and JavaScript URLs. Cross-referencing against the CVE database identified two unpatched XSS vulnerabilities: CVE-2026-1404 (Ultimate Member 2.11.1) and CVE-2026-1512 (Essential Addons for Elementor 6.5.9).

But CVE numbers are theoretical. We needed to verify: are these actually exploitable with this specific combination of plugins and WordPress version?

### The Approach

Build an isolated replica of the production software stack. An automation module that provisions a FreeBSD jail with the exact plugin versions observed in production.

The module generates a 6-phase installation sequence:
1. Install FreeBSD packages (MariaDB, PHP 8.2, nginx)
2. Configure database
3. Configure web services
4. Install WordPress core
5. Deploy vulnerable plugin versions
6. Create test accounts, disable auto-updates

### What We Built

- 7 Rust source files for the lab provisioner
- Multi-step orchestration protocol for phased installation
- Config generators for nginx, PHP-FPM, MariaDB, and wp-config.php
- Input validation on all user-supplied arguments (identifiers, passwords, URLs)
- 26 unit tests covering argument parsing, config generation, and security validation

### What We Learned

The instruction-generation approach means the lab provisioning logic is testable without a live FreeBSD system. The instruction generation is a pure function: given arguments, produce shell commands. The tests verify the commands are correct without executing them.

---

## Phase 2: Reality Meets Theory

### The Problem

Phase 1 worked in theory. Phase 2 is where theory met FreeBSD.

### What Broke

Three things were wrong:

1. **Service names.** The MariaDB rc.d service on FreeBSD is `mysql-server`, not `mariadb`. The PHP-FPM service is `php_fpm`, not `php-fpm`. These are not documented anywhere obvious — you discover them by running `service -e` on a live FreeBSD system.

2. **Socket paths.** PHP-FPM was configured to write its socket to `/var/run/php-fpm-wordpress.sock`, but nginx was looking for it at `/var/run/php-fpm.sock`. The lab deployed successfully but WordPress returned 502 Bad Gateway because nginx couldn't reach PHP.

3. **Missing phar package.** wp-cli requires the PHP `phar` extension to run. FreeBSD's `php82` package doesn't include phar by default — it's a separate `php82-phar` package. The WordPress install phase failed silently because `wp` couldn't execute.

### What We Fixed

- Updated constants with correct FreeBSD service names
- Aligned socket paths between PHP-FPM pool config and nginx upstream config
- Added `php82-phar` to the package list

### What We Learned

FreeBSD package naming and service naming conventions don't follow Linux patterns. The only reliable way to get these right is to deploy on real FreeBSD and fix what breaks. This is exactly why the lab exists — discover these issues in isolation, not in production.

---

## Phase 3: Remediation Tool

### The Problem

The lab confirmed both CVEs are exploitable. Now we need to fix them — not just on this one site, but in a way that can be reused. The manual remediation is straightforward (`wp plugin update`), but we want:

- Version checking against known-vulnerable thresholds
- XSS payload detection in existing Elementor data
- A full audit → update → verify pipeline
- Structured output and exit codes for automation

### The Approach

A standalone Rust binary with 8 source modules. No framework, no async runtime, no serde — just the standard library plus one HTTP crate for download fallback.

### What We Built

- `check` — Scans plugin versions against a known-vulnerable list. Parses wp-cli CSV output. Semantic version comparison handles variable-length version strings (6.5.9 vs 6.5.10).
- `audit` — Scans `_elementor_data` post metadata for 9 XSS indicator patterns (`<img>`, `<a href=`, `<script>`, `onerror=`, etc.). Extracts `login_form_subtitle` values from raw JSON without a parser — string scanning handles escaped quotes and nested structures.
- `update` — Updates vulnerable plugins via wp-cli. Falls back to direct download from wordpress.org if wp-cli update fails.
- `verify` — Post-update confirmation: checks each plugin version is at or above the safe threshold.
- `wp_cli` — Wrapper that handles `--path=` and `--allow-root` on every invocation. Preflight checks verify wp-cli exists and WordPress path is valid.
- `http` — HTTP download with cascading fallback: fetch → curl → wget → ureq.
- `output` — ANSI colored terminal output (PASS/FAIL/WARN/INFO).
- `main` — CLI parsing, command dispatch, platform-specific defaults.

### What We Learned

Parsing Elementor JSON without serde is a deliberate choice. The `_elementor_data` field contains deeply nested JSON with escaped strings inside escaped strings. A full JSON parser would work, but a targeted string scanner that finds `"login_form_subtitle"` and extracts the value handles the specific patterns we care about with zero dependencies. The tests verify it handles edge cases (escaped quotes, multiple values, empty values).

---

## Phase 4: Real-World Fixes

### The Problem

wp-fix worked perfectly in the lab. Then we ran it on a server that wasn't the lab.

### What Broke

1. **`--path=` syntax.** wp-cli requires `--path=/var/www/html` (with equals sign), not `--path /var/www/html` (with space). The initial implementation used space-separated arguments. wp-cli silently ignored the path and operated on the wrong directory.

2. **Preflight checks.** The tool assumed wp-cli was installed and WordPress existed at the default path. On a fresh server, it crashed with an unhelpful error message instead of telling the user what was missing.

3. **UTF-8 panic.** One server had non-UTF-8 bytes in its wp-cli output (a plugin with a Latin-1 encoded description). `String::from_utf8().unwrap()` panicked. Changed to `from_utf8_lossy()`.

4. **Source fallback.** When wp-cli's `plugin update` command failed (network issues, wordpress.org rate limiting), there was no fallback. Added direct download from `https://downloads.wordpress.org/plugin/{slug}.zip` as a secondary update path.

### What We Fixed

- Preflight: now checks for wp-cli binary existence, WordPress directory existence, and wp-config.php presence before running any commands
- Path syntax: all wp-cli invocations use `--path={value}` format
- UTF-8: all command output parsed with `from_utf8_lossy`
- Source fallback: tries wp-cli first, then direct ZIP download from wordpress.org

### What We Learned

The gap between "works in my lab" and "works on arbitrary servers" is always larger than expected. Preflight checks are not optional — they're the difference between a useful tool and a tool that crashes with a stack trace. And UTF-8 assumptions in Rust are the equivalent of shell script quoting bugs: they work until they encounter real-world data.

---

## Phase 5: Portability

### The Problem

wp-fix worked on FreeBSD and Linux. But some WordPress servers run Windows (XAMPP, WAMP). And some servers don't have curl or wget installed.

### The Approach

Add `ureq` as a pure Rust HTTP client dependency — the final fallback when no system HTTP tools are available. Add Windows as a cross-compilation target. Add `update-all` and `install-wpcli` commands.

### What We Built

- **ureq HTTP fallback**: now tries fetch → curl → wget → ureq in order. The ureq fallback works everywhere with zero system dependencies.
- **Windows target**: Platform defaults for Windows: `C:\xampp\htdocs\wordpress` and `C:\wp-cli\wp.bat`.
- **`update-all` command**: Instead of only updating known-vulnerable plugins, enumerate ALL installed plugins and update any that have available updates. Useful for general WordPress maintenance beyond the specific CVEs.
- **`install-wpcli` command**: Downloads and installs wp-cli if not present. Eliminates the "wp-cli not found" failure mode.

### What We Learned

The ureq crate is the right trade-off. It adds one dependency but eliminates the "no HTTP tool available" failure mode completely. The cascading fallback means we use fast system tools when available and fall back to the pure Rust implementation only when necessary.

---

## Phase 6: 112-Check Security Scanner

### The Problem

The initial wp-fix tool addressed three specific plugins. But WordPress security is broader than three CVEs — a site can have dozens of misconfigurations, weak server settings, and vulnerable patterns across the full stack.

### What We Built

Expanded wp-fix from a targeted patch tool to a comprehensive 112-check security scanner across five categories:

| Category | Checks | Scope |
|----------|--------|-------|
| Network | 35 | Version exposure, security headers, TLS, DNS, CORS |
| Server | 25 | wp-config.php, PHP config, file permissions |
| Plugin/Theme | 22 | Source code analysis for eval(), SQL injection, XSS |
| Database | 18 | Default admin, orphaned data, suspicious accounts |
| Environment | 12 | Disk, memory, firewall, SSH, suspicious processes |

Two output formats: human-readable (color-coded PASS/FAIL) and machine-readable JSON for integration with monitoring systems.

65 tests covering all scanner categories.

---

## Themes

**Lab first, production second.** Every feature was developed and tested in the isolated lab before touching a production server. The lab exists specifically to make this possible.

**Failure-driven development.** Each phase exists because the previous phase broke in a new environment. The tool got better by encountering real-world conditions.

**One codebase, many targets.** The investment in Rust means the same logic runs as a native binary on three platforms. A shell script approach would require maintaining separate implementations for each.
