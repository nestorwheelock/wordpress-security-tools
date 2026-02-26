# WordPress Security Tools

Copyright 2026 Nestor Wheelock - South City Computer

## Your WordPress Site Can Be Compromised With One Click
There are two paths, and both are trivially easy:

**Path 1 — A link to your own website.** An attacker sends the site administrator a URL that points to the site's own domain. It looks like a normal link to their own content. One click from a logged-in admin executes arbitrary JavaScript in their browser, steals their session cookie, and gives the attacker full administrative control of the site. No suspicious domains, no attachments, no malware downloads. Just a crafted URL to a site the victim trusts because it's theirs. *(CVE-2026-1404, Ultimate Member <= 2.11.1)*

**Path 2 — "Can I write articles for your site?"** An attacker offers to contribute free content — guest blog posts, event listings, community updates. The site owner grants them a Contributor account, the lowest privilege role that can create content. This is routine. Guest bloggers, volunteers, community members, freelance writers — WordPress sites hand out Contributor accounts every day without a second thought. Once the attacker has that account, they inject invisible tracking beacons and phishing links directly into the site's login page. Every visitor to the login page is silently compromised. No further interaction with the site owner required. Or the attacker offers to pay the site owner for a reciprocal link — they'll write a free article, publish it, and "manage their link for SEO purposes." The site owner happily grants access because they're getting paid for a backlink. In reality, the attacker is using a legitimate, trusted website to phish every person who visits the login page. A small payment in Bitcoin for a backlink is a trivial cost compared to the potential gain: stolen credentials, financial data, and a platform to deploy ransomware or cryptoware against every visitor to a site they already trust.

The attacker doesn't need to hack anything. They just need to ask. *(CVE-2026-1512, Essential Addons for Elementor <= 6.5.9)*

---

## How We Found This

A passive review of a production WordPress site's public HTML source revealed outdated plugin versions with published CVEs. No scanning tools, no authenticated access — just reading the version strings that every visitor's browser receives from CSS and JavaScript files.

| CVE | Plugin | Vulnerable Version | Fixed In | Type |
|-----|--------|--------------------|----------|------|
| CVE-2026-1404 | Ultimate Member | <= 2.11.1 | 2.11.2 | Reflected XSS |
| CVE-2026-1512 | Essential Addons for Elementor | <= 6.5.9 | 6.5.10 | Stored XSS |
| — | Elementor | < 3.35.5 | 3.35.5 | Outdated (update recommended) |

A CVE number alone doesn't tell you whether a specific site is actually at risk. To answer that, we built an isolated lab that replicated the exact software stack, confirmed both vulnerabilities are exploitable, and built a remediation tool to fix them. That tool grew into a 112-check security scanner. The full investigation is documented in [docs/CVE-2026-1404.md](docs/CVE-2026-1404.md) and [docs/CVE-2026-1512.md](docs/CVE-2026-1512.md).

> ## Download the Free Whitepaper
>
> [WordPress XSS Vulnerability Assessment: A Lab-Based Case Study](docs/whitepaper/wordpress-xss-vulnerability-assessment.md)
>
> Methodology, exploitation proof-of-concept, full attack chain analysis, and remediation procedures for CVE-2026-1512 and CVE-2026-1404. Includes screenshots and architecture diagrams.

---

## Download

Precompiled binaries for all platforms. See [releases/](releases/) for checksums and detailed instructions.

| Platform | Download |
|----------|----------|
| Linux x86_64 | [wp-fix-linux-x86_64](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-linux-x86_64) |
| FreeBSD x86_64 | [wp-fix-freebsd-x86_64](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-freebsd-x86_64) |
| Windows x86_64 | [wp-fix-windows-x86_64.exe](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-windows-x86_64.exe) |
| Checksums | [SHA256SUMS](https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/SHA256SUMS) |

**All releases:** [github.com/nestorwheelock/wordpress-security-tools/releases](https://github.com/nestorwheelock/wordpress-security-tools/releases)

### Install (Linux / FreeBSD)

```sh
curl -LO https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-linux-x86_64
chmod +x wp-fix-linux-x86_64
sudo mv wp-fix-linux-x86_64 /usr/local/bin/wp-fix
```

For FreeBSD, replace `wp-fix-linux-x86_64` with `wp-fix-freebsd-x86_64`.

### Install (Windows)

Download from PowerShell:

```powershell
Invoke-WebRequest -Uri "https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/wp-fix-windows-x86_64.exe" -OutFile "wp-fix.exe"
```

Or download `wp-fix-windows-x86_64.exe` from the [releases page](https://github.com/nestorwheelock/wordpress-security-tools/releases) and rename to `wp-fix.exe`.

Run from PowerShell or Command Prompt:

```powershell
.\wp-fix.exe all --wp-path C:\xampp\htdocs\wordpress
```

For WAMP installations, use `--wp-path C:\wamp64\www\wordpress`.

### Verify Checksum

```sh
# Linux / FreeBSD
curl -LO https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/SHA256SUMS
sha256sum -c SHA256SUMS
```

```powershell
# Windows (PowerShell)
Invoke-WebRequest -Uri "https://github.com/nestorwheelock/wordpress-security-tools/releases/download/v0.1.0/SHA256SUMS" -OutFile "SHA256SUMS"
Get-FileHash wp-fix.exe -Algorithm SHA256
```

Free for personal and hobby use. Commercial or business use requires a license — contact [South City Computer](https://southcitycomputer.com) for terms.

---

## Projects

### wp-fix — Security Scanner and Remediation Tool

A cross-platform CLI binary that scans WordPress installations for 112 security issues across 5 categories and remediates the ones it finds. Produces human-readable terminal output (color-coded PASS/FAIL) and machine-readable JSON.

| Category | Checks | Examples |
|----------|--------|----------|
| Network | 35 | Version exposure, security headers, TLS configuration, DNS, CORS |
| Server | 25 | wp-config.php hardening, PHP configuration, file permissions |
| Plugin/Theme | 22 | Source code analysis for eval(), SQL injection, XSS patterns |
| Database | 18 | Default admin accounts, orphaned data, suspicious user accounts |
| Environment | 12 | Disk usage, memory, firewall rules, SSH configuration, suspicious processes |

```sh
# Run all checks
wp-fix all

# Individual commands
wp-fix check       # Report plugin versions, flag vulnerable
wp-fix audit       # Scan for XSS indicators
wp-fix update      # Update targeted vulnerable plugins
wp-fix update-all  # Update ALL outdated plugins
wp-fix verify      # Confirm patched versions installed
wp-fix scan        # Run full 112-check security scan
```

See [docs/USAGE.md](docs/USAGE.md) for the full command reference.

### wordpress — Vulnerable Lab Provisioner

An automation module that generates a 6-phase FreeBSD jail provisioning sequence: package installation, MariaDB setup, PHP-FPM and nginx configuration, WordPress core install, vulnerable plugin deployment, and test account creation. Used internally for reproducing and verifying CVEs in an isolated environment.

---

## Why a Compiled Binary Instead of Shell Scripts

- **Zero dependencies** — copy one file to the server and run it. No interpreter, no PATH issues, no "is jq installed," no "which version of bash."
- **Cross-platform from one codebase** — the same tool runs on Linux, FreeBSD, and Windows. A shell script that works on Linux needs rewriting for FreeBSD (different paths, service names, package managers) and rewriting from scratch for Windows.
- **Memory safe** — Rust prevents buffer overflows, null pointer crashes, and use-after-free at compile time. wp-fix parses untrusted data (plugin version strings, wp-cli CSV output, raw Elementor JSON). In a shell script, malformed input produces silent corruption. In Rust, it produces a handled error.
- **Tested before deployed** — 65 unit tests covering version comparison, CSV parsing, XSS pattern detection, and argument handling run on every build. Shell scripts have no equivalent.
- **Structured exit codes** — exit 0 (clean), 1 (vulnerabilities found), 2 (fatal error). Feed directly to cron and monitoring systems without wrapper scripts.
- **Automation-ready** — the same codebase can be deployed through our proprietary automation tooling for sandboxed remote execution. No SSH keys to distribute, no ansible inventory to maintain.

Read the full rationale in [docs/WHY-RUST.md](docs/WHY-RUST.md).

---

## Services

### You Have a WordPress Site. We Keep It Secure.

Whether you're running a personal blog or a business-critical website, WordPress requires ongoing security maintenance. We offer five tiers of service:

| Tier | Service | You Handle | We Handle |
|------|---------|------------|-----------|
| 1 | Advisory Report | Everything | Scan + documentation |
| 2 | One-Time Remediation | Ongoing maintenance | Backup + patch + verify |
| 3 | Ongoing Monitoring | Hosting infrastructure | Scanning + patching + alerts |
| 4 | Fully Managed | Nothing | Infrastructure + patching + monitoring + backups |
| 5 | Platform Migration | Content direction | Replace WordPress entirely |

**Tier 1 — Advisory Report:** We run the scanner against your site and deliver a detailed report covering all 112 checks. You receive the wp-fix binary and the report. You handle remediation.

**Tier 2 — One-Time Remediation:** We take a full backup, apply all patches, verify the fixes, and deliver a before/after report. One engagement, one invoice.

**Tier 3 — Ongoing Monitoring:** Your site is scanned on a recurring schedule. When vulnerabilities are detected, we patch them and send you a summary. You keep your hosting provider; we keep your plugins current.

**Tier 4 — Fully Managed:** Your server is enrolled in our automation platform. Sandboxed modules handle patching, monitoring, backups, and health checks — no SSH keys, no open ports, automatic snapshots before every change.

**Tier 5 — Platform Migration:** When you're ready to leave WordPress, we build a purpose-built Rust/Axum application tailored to your content and workflows. Same team, same toolchain, no WordPress.

---

## Reports and Research

| Document | Description |
|----------|-------------|
| [Whitepaper: WordPress XSS Vulnerability Assessment](docs/whitepaper/wordpress-xss-vulnerability-assessment.md) | Full case study — methodology, exploitation, and remediation for CVE-2026-1512 and CVE-2026-1404 |
| [docs/CVE-2026-1404.md](docs/CVE-2026-1404.md) | Ultimate Member reflected XSS analysis |
| [docs/CVE-2026-1512.md](docs/CVE-2026-1512.md) | Essential Addons stored XSS analysis |

## Documentation

| Document | Description |
|----------|-------------|
| [docs/USAGE.md](docs/USAGE.md) | Complete usage guide for all tools |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Component relationships and design decisions |
| [docs/WHY-RUST.md](docs/WHY-RUST.md) | Why compiled binaries instead of shell scripts |
| [docs/DEVELOPMENT-JOURNAL.md](docs/DEVELOPMENT-JOURNAL.md) | Iterative development story |

## License

Copyright 2026 Nestor Wheelock - South City Computer. All rights reserved.

Free for non-commercial use on hobby and personal websites. Commercial or business use requires a commercial license. See [LICENSE](LICENSE) for full terms.

This software is not released under the MIT License, Creative Commons, or any open-source license.
