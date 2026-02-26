# Why a Compiled Binary Instead of Shell Scripts

The standard approach to WordPress remediation is a bash script: `wp plugin update`, some `grep` calls, maybe a `curl`. It works until it doesn't. This document explains why wp-fix is a compiled Rust binary instead — and why the same reasoning extends to every server management problem.

## Shell Scripts Break

Every WordPress administrator has a collection of bash scripts. They work on the machine where they were written. Then they don't work somewhere else.

**Quoting bugs silently corrupt data.** A path with a space, a plugin name with a hyphen in the wrong context, a variable expansion inside single quotes — bash will happily execute the wrong command and report success. The script author discovers the bug when a production site breaks.

**Portability is a fiction.** bash 3.2 (macOS default for years) behaves differently from bash 5.x. `sh` on FreeBSD is not bash. `sed -i` takes a mandatory argument on macOS but not on Linux. `grep -P` doesn't exist on FreeBSD. Every "portable" shell script is actually a collection of platform-specific workarounds.

**Error handling is opt-in.** `set -e` helps but doesn't catch errors in pipelines, subshells, or command substitutions. A failed `curl` inside a `$()` substitution returns empty string, not an error. The script continues with empty data and produces wrong results.

**There are no tests.** You can write tests for shell scripts (bats, shunit2), but nobody does. The feedback loop is: write script → run it on production → discover the bug → fix the script → repeat.

## A Single Binary Just Works

```sh
scp wp-fix user@server:/usr/local/bin/
ssh user@server wp-fix all
```

No interpreter. No dependencies. No PATH issues. No "which version of bash is installed." No "does this server have jq." No "is curl compiled with SSL support."

The binary contains everything it needs. Copy it to any Linux, FreeBSD, or Windows server and run it. The same binary that passed 38 tests on the build machine runs identically on the target.

## Cron and Automation

A shell script on cron requires a wrapper script to handle errors, logging, and exit codes. The wrapper needs its own error handling. It's wrappers all the way down.

wp-fix returns structured exit codes:

| Exit Code | Meaning | Cron / Monitoring Action |
|-----------|---------|--------------------------|
| 0 | All checks passed, no vulnerabilities | No action needed |
| 1 | Vulnerabilities found or updates applied | Alert: review results |
| 2 | Fatal error (missing wp-cli, bad path) | Alert: tool misconfigured |

```crontab
0 6 * * * /usr/local/bin/wp-fix all --wp-path /var/www/html >> /var/log/wp-fix.log 2>&1
```

Feed the exit code to your monitoring system. No wrapper script needed.

## Memory Safety

Rust's ownership system prevents the classes of bugs that shell scripts discover in production at 3am:

- **No buffer overflows** — string handling is bounds-checked at compile time
- **No use-after-free** — the compiler tracks every value's lifetime
- **No null pointer crashes** — Rust has no null; optional values use `Option<T>` which must be explicitly handled
- **No data races** — the type system prevents concurrent mutation

wp-fix parses untrusted data: plugin version strings, CSV output from wp-cli, raw JSON from Elementor widget metadata. In a shell script, a malformed version string or unexpected CSV format produces silent corruption. In Rust, it produces a compile error or a handled `Result::Err`.

## Tested Before Deployed

38 unit tests run before every build:

- **Version comparison**: `6.5.9` < `6.5.10`, `6.6` > `6.5.10`, equal versions, different-length versions
- **CSV parsing**: Normal output, empty output, malformed rows
- **XSS pattern detection**: `<img>`, `<a href=`, `<script>`, `onerror=`, `javascript:`, case-insensitive matching
- **Subtitle extraction**: Basic HTML, escaped quotes, multiple values, empty values, no values
- **Argument handling**: Custom paths, flag ordering, unknown arguments, missing values

Every code path that processes external data has a test. The tests run in milliseconds. They run on every `cargo build`. A shell script has none of this.

## Cross-Compilation

One codebase. Three platforms:

- Linux (`x86_64-unknown-linux-gnu`)
- FreeBSD (`x86_64-unknown-freebsd`)
- Windows (`x86_64-pc-windows-gnu`)

A shell script that works on Linux needs rewriting for FreeBSD (different paths, different package managers, different service names). A shell script for Windows needs rewriting from scratch in PowerShell. wp-fix uses compile-time `#[cfg(target_os)]` to select platform-appropriate defaults while sharing all logic.

---

# Managed Remediation at Scale

The native binary handles direct remediation: SSH to the server, run wp-fix, done. But what about 10 servers? 50? What about servers behind firewalls with no inbound SSH?

## Proprietary Automation Platform

For Tier 4 (Fully Managed) clients, wp-fix is deployed through our proprietary automation platform. The same remediation logic runs remotely in a sandboxed environment on the target server.

No SSH keys to distribute. No ansible inventory to maintain. No playbooks to debug. The automation platform handles module delivery, sandboxed execution, and structured result collection.

## Sandboxed Execution

Modules run in a sandboxed runtime with declared capabilities:

- The module declares what filesystem paths it needs access to
- The module declares what network access it requires
- The runtime enforces these boundaries
- A compromised or buggy module cannot escape its sandbox

Compare this to a shell script run via SSH or ansible: it executes with the full privileges of the SSH user, can read any file, can write anywhere, can make arbitrary network connections.

## Snapshot → Test → Rollback

Our automation platform integrates with ZFS snapshots:

1. Before remediation: `zfs snapshot zroot/jails/wordpress@pre-fix`
2. Run wp-fix: update plugins, verify versions
3. Test: confirm site loads, login works, no broken pages
4. If broken: `zfs rollback zroot/jails/wordpress@pre-fix` — instant, complete rollback

This workflow is impossible to brick a production server. The snapshot captures the entire filesystem state. The rollback is atomic and instant. No "restore from backup" that takes hours and might be stale.

---

# AI-Ready Architecture

The wp-fix tool already collects structured, machine-readable data. The architecture is designed to evolve toward AI-assisted remediation.

## Data Collection

wp-fix currently produces structured output:

- Plugin names, versions, update availability
- XSS pattern matches with payload classification
- Version comparison results (vulnerable / patched / not installed)
- Exit codes for automation

This is not log output — it's structured audit data that machines can consume.

## API Integration (Future)

Future versions send collected audit data to an analysis API:

```
wp-fix audit --report-to https://api.example.com/analyze
```

The API evaluates risk across multiple sites, identifies patterns ("this plugin has had 4 XSS CVEs in 2 years — recommend replacement"), and suggests prioritized remediation steps.

## Pre-Built Action Library

The binary contains a fixed set of remediation actions:

- Update specific plugin to latest version
- Update all outdated plugins
- Download and install wp-cli
- Scan Elementor data for XSS indicators
- Verify patched versions

An AI analysis service selects from these actions rather than generating arbitrary shell commands. The binary executes deterministic, tested operations. There is no prompt → shell execution chain where a malformed AI response could run `rm -rf /`.

## Human-in-the-Loop

The intended workflow:

1. **wp-fix collects** — structured audit data from the target server
2. **AI analyzes** — evaluates risk, suggests prioritization, identifies patterns
3. **Human reviews** — approves or modifies the suggested remediation plan
4. **wp-fix executes** — runs the approved actions (tested, deterministic, sandboxed)

The compiled binary is the execution layer. It does exactly what it's told, nothing more. The AI is the analysis layer. It suggests, it doesn't execute. The human is the approval layer. This architecture prevents the failure mode where an AI hallucinates a remediation command and a shell script blindly executes it.
