# WordPress Security Research

![License](https://img.shields.io/badge/license-MIT-blue)
![Focus](https://img.shields.io/badge/focus-wordpress%20plugin%20vulnerabilities-informational)
![Status](https://img.shields.io/badge/status-active-brightgreen)
![Writeups](https://img.shields.io/badge/writeups-3-blueviolet)

Reproducible vulnerability research on WordPress plugins. Every entry in this repository is a complete research artifact: source-level root cause analysis, vendor patch diffing, captured reproduction evidence, and a realistic impact assessment. The goal is methodology that a reader can audit and repeat, not a payload collection.

---

## Table of Contents

- [Repository Structure](#repository-structure)
- [Research Methodology](#research-methodology)
- [Evidence Standard](#evidence-standard)
- [Getting Started](#getting-started)
- [Running the Code](#running-the-code)
- [Reading Order](#reading-order)
- [Scope](#scope)
- [Ethics and Legal](#ethics-and-legal)
- [Disclosure Policy](#disclosure-policy)
- [Contributing](#contributing)
- [License](#license)

---

## Repository Structure

Every CVE directory follows the same layout. The consistency is deliberate — once a reader has navigated one writeup, every other writeup is immediately familiar.

    wordpress-security-research/
    ├── CVE-<year>-<id>-<component>/
    │   ├── README.md              # Summary, exploitation, impact, mitigation
    │   ├── ANALYSIS.md            # Root cause, vulnerable code path, patch diff
    │   ├── METHODOLOGY.md         # How the reproduction was performed
    │   ├── requirements.txt       # Pinned Python dependencies
    │   ├── src/
    │   │   ├── exploit.py         # Reproduction script
    │   │   └── lib/               # Support modules (HTTP client, logger)
    │   ├── tests/
    │   │   └── test_exploit.py    # Unit tests for reproducer logic
    │   └── evidence/
    │       └── exploitation.log   # Terminal transcript
    ├── .github/workflows/lint.yml # CI: ruff + black on push
    ├── CONTRIBUTING.md
    ├── SECURITY.md
    ├── LICENSE
    └── README.md

Each CVE's `README.md` is the primary writeup. The root README you are reading now is a navigational and methodological overview — it does not duplicate the technical content of the individual writeups.

---

## Research Methodology

The structure of each writeup is fixed. This is not a stylistic choice — it forces the analysis to answer the questions that actually matter and prevents the writeup from drifting into vague description.

### 1. Scope

Affected versions, fixed version, CVSS vector, CWE classification, and the public disclosure timeline. Anyone reading the writeup knows immediately what the boundaries of the research are.

### 2. Root Cause

The vulnerable code, cited by file and function. This section answers: *what specific line runs, with what input, and why is that a problem?* It is not enough to say "the plugin allows file upload" — the writeup names the handler, traces the request through it, and identifies the check that should have been there and was not.

### 3. Patch Analysis

The vendor's fix, extracted by diffing the vulnerable and patched versions. This section answers: *what did the vendor change, and what does that reveal about how they understood the bug?* A patch that adds input validation and a patch that removes the endpoint entirely are different statements about the severity of the issue, and the writeup distinguishes between them.

### 4. Reproduction

The exact steps to trigger the finding. Every command is provided. Expected output is captured. If the reproduction depends on a non-obvious detail — a specific header, a particular parameter name, a required ordering — that detail is called out.

### 5. Impact

What an attacker actually gains in a realistic deployment. This section deliberately avoids the theoretical maximum and describes the practical case: default configuration, common hosting environment, the attacker's actual position on the network. A CVSS 9.8 that requires local access and a non-default configuration is not the same as one that works against shared hosting from the public internet.

### 6. Mitigation

The vendor fix, plus compensating controls for environments where patching is not immediate, plus detection opportunities for environments where the fix may never be applied. A writeup that ends at "update the plugin" is only useful to people who can update the plugin.

---

## Evidence Standard

Every technical claim in this repository is backed by one of the following. This is the rule that keeps the writeups honest.

| Claim Type | Required Backing |
|-----------|------------------|
| Vulnerable code path | Source excerpt with file and line reference |
| Patch behavior | `diff` output between the two versions |
| Exploitation | Captured request/response pair from a live test |
| Execution confirmation | Terminal transcript with the unique marker in the response |
| Impact | Cited advisory or documented deployment context |

If a claim cannot be backed by one of those, it does not appear in the writeup. Speculation is not included, and no statement is made about a system that was not tested.

---

## Getting Started

### Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.9+ | Required to run the reproducers |
| Git | any recent | Only needed if cloning rather than downloading |

Each writeup's reproduction section documents any additional requirements specific to that CVE. Most reproducers have no dependencies beyond the standard library and `requests`.

### Clone the Repository

    git clone https://github.com/aprnx/wordpress-security-research.git
    cd wordpress-security-research

---

## Running the Code

Each CVE directory has its own isolated Python environment:

    cd CVE-2020-25213-wp-file-manager
    python3 -m venv .venv
    source .venv/bin/activate
    pip install -r requirements.txt
    python3 src/exploit.py --help

Dependencies are pinned to specific versions. Updates to dependencies are made deliberately and tested before being merged, not pulled automatically.

### Running Tests

    cd CVE-2020-25213-wp-file-manager
    pip install pytest
    pytest tests/

Unit tests cover the reproducer's helper logic — argument parsing, version comparison, session configuration. Integration testing is performed manually and captured in the `evidence/` directory of each CVE.

---

## Reading Order

If you are new to the repository, read the entries in the order they appear in the [Contents](#contents) table.

If you are looking for a specific vulnerability class:

- **Unauthenticated file upload → RCE:** all three entries
- **Third-party library integration flaws:** CVE-2020-25213
- **Missing authorization on state-changing endpoints:** CVE-2026-82970 and CVE-2026-3891

If you are looking for a specific plugin:

- **WP File Manager:** CVE-2020-25213
- **WP Cookie Consent (gdpr-cookie-consent):** CVE-2026-82970
- **Pix for WooCommerce:** CVE-2026-3891

---

## Scope

This repository covers WordPress plugin vulnerabilities only. The scope is deliberate: a focused repository with well-documented entries is more useful than a broad one with shallow entries. The analysis methodology is transferable to other plugin families and to other web application stacks, but the entries here stay within WordPress.

Related work outside this repository's scope is available in a separate repository. See the profile for links.

---

## Ethics and Legal

> [!WARNING]
> **All research in this repository is conducted against locally hosted, intentionally vulnerable environments.** No live, third-party, or production systems are targeted at any point. Where a CVE has an existing public disclosure, that disclosure is cited and the timeline is documented in the relevant writeup. Nothing here is zero-day.

### Rules for Use

- Do not run any code in this repository against a system you do not own or do not have explicit written authorization to test.
- Do not open issues, pull requests, or discussions describing how to adapt the code to attack live targets.
- Do not redistribute the code with the intent of enabling its use against systems without authorization.

The reproducer code is intentionally minimal and non-destructive. The payloads used in the reproductions echo a marker string — they do not open shells, exfiltrate data, or perform any other weaponized behavior. This is a deliberate design choice: the point is to demonstrate the finding, not to produce a usable offensive tool.

---

## Disclosure Policy

Every CVE covered in this repository was disclosed publicly before this repository was published. No original vulnerability disclosure is claimed here.

The disclosure timeline for each CVE is documented in that CVE's `README.md` and includes:

- The original disclosure date
- The vendor fix release date
- The date the local reproduction was performed
- The date the writeup was published

If you believe any writeup in this repository covers a vulnerability that has not been publicly disclosed, contact the maintainer before publishing anything.

---

## Contributing

Corrections, additional detection signatures, and portability fixes are welcome. Full guidance is in [CONTRIBUTING.md](CONTRIBUTING.md).

The short version:

- Open an issue first for anything beyond typo fixes
- Corrections to technical claims require a source or a reproduction
- New CVE writeups are not accepted — every entry in this repository is original work by the maintainer
