# WordPress Security Research

![License](https://img.shields.io/badge/license-MIT-blue)
![Focus](https://img.shields.io/badge/focus-wordpress%20plugins-informational)
![Status](https://img.shields.io/badge/status-active-brightgreen)

A collection of WordPress plugin vulnerability writeups with reproducible
lab environments, root-cause analysis, and documented exploitation paths.

## Purpose

Most public CVE writeups for WordPress plugins stop at "run this script."
They don't explain why the bug exists, what the patch changed, or how the
design decision that caused it could have been avoided.

This repository is the opposite. Each writeup covers:

- The vulnerable code path and the design decision behind it
- The vendor's patch, diffed against the vulnerable version
- A local Docker lab that reproduces the exact version
- Exploitation with captured evidence
- Realistic impact assessment and mitigation

## Contents

| CVE | Component | Class | Severity | Status |
|-----|-----------|-------|----------|--------|
| [CVE-2020-25213](CVE-2020-25213-wp-file-manager/) | WP File Manager < 6.9 | Unauthenticated file upload → RCE | 9.8 Critical | Complete |

Additional writeups are in progress. Each directory is self-contained and
does not depend on the others.

## Methodology

Every writeup follows the same structure so the reasoning is comparable
across CVEs:

1. **Root cause** — the specific code and the reason it exists
2. **Patch analysis** — what the fix changed and why
3. **Exploitation** — reproduction against the local lab, with evidence
4. **Impact** — what an attacker actually gets in a realistic deployment
5. **Mitigation** — fix, compensating controls, and detection opportunity

Failed approaches and dead ends are documented in each `METHODOLOGY.md`.
The record includes what didn't work.

## Scope and Ethics

> [!WARNING]
> All research here is conducted against locally hosted, intentionally
> vulnerable environments. No live or third-party systems are targeted.
> Where a CVE has an established public disclosure, that disclosure is
> cited and the timeline is documented in the relevant README.
