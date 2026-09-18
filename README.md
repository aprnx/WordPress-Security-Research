# WordPress Security Research

![License](https://img.shields.io/badge/license-MIT-blue)
![Focus](https://img.shields.io/badge/focus-wordpress%20plugin%20vulnerabilities-informational)
![Status](https://img.shields.io/badge/status-active-brightgreen)
![Writeups](https://img.shields.io/badge/writeups-3-blueviolet)

Reproducible vulnerability research on WordPress plugins. Every entry in this repository is a complete research artifact: source-level root cause analysis, vendor patch diffing, captured reproduction evidence, and a realistic impact assessment. The goal is methodology that a reader can audit and repeat, not a payload collection.

---

## Table of Contents

- [Contents](#contents)
- [Writeups](#writeups)
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

## Contents

| CVE | Component | Class | CVSS | CWE | Status |
|-----|-----------|-------|------|-----|--------|
| [CVE-2020-25213](CVE-2020-25213-wp-file-manager/) | WP File Manager < 6.9 | Unauthenticated file upload → RCE | 9.8 Critical | CWE-434 | Complete |
| [CVE-2026-82970](CVE-2026-82970-wp-cookie-notice/) | WP Cookie Consent ≤ 4.4.1 | Unauthenticated arbitrary file upload | 9.8 Critical | CWE-434 | Complete |
| [CVE-2026-3891](CVE-2026-3891-pix-for-woocommerce/) | Pix for WooCommerce | Unauthenticated file upload via exposed nonce endpoint | TBD | CWE-434 / CWE-862 | Complete |

Each directory is fully self-contained. Reproduction steps for one writeup do not depend on another.

---

## Writeups

### CVE-2020-25213 — WP File Manager Unauthenticated File Upload

**Component:** WP File Manager (plugin slug `wp-file-manager`)
**Affected versions:** < 6.9
**Fixed version:** 6.9
**Class:** Unauthenticated arbitrary file upload leading to remote code execution
**CVSS v3.1:** 9.8 (Critical) — AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
**CWE:** CWE-434 (Unrestricted Upload of File with Dangerous Type)

The plugin bundles the elFinder file manager library and deploys its reference connector script as a directly reachable PHP file inside the plugin directory:

    /wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php

The connector accepts file operations via a `cmd` request parameter. For `cmd=upload`, it writes the multipart body into the configured volume root — which, in vulnerable versions, resolves to a directory inside the plugin itself:

    /wp-content/plugins/wp-file-manager/lib/files/

Because that path sits inside the plugin directory, and WordPress web servers execute PHP in that path by default, an uploaded `.php` file becomes immediately executable. No authentication check runs anywhere in the request. No capability check runs. The connector is reachable by any HTTP client with network access to the site.

The design flaw is that elFinder's connector was written to be embedded behind a host application that performs authorization. The plugin deployed the reference connector directly, without wrapping it in WordPress's permission layer — so the assumption the library was built on was silently removed.

The fix in 6.9 moved the connector behind WordPress context and constrained both the upload destination and the file types accepted.

**What the writeup covers:** the connector's request-routing logic, the default volume configuration that placed uploaded files inside the plugin directory, the specific patch hunks extracted via SVN diff, the exact multipart request that triggers the upload, and the follow-up GET that confirms code execution.

---

### CVE-2026-82970 — WP Cookie Consent Unauthenticated Arbitrary File Upload

**Component:** WP Cookie Consent (plugin slug `gdpr-cookie-consent`)
**Affected versions:** ≤ 4.4.1
**Fixed version:** 4.4.2
**Class:** Unauthenticated arbitrary file upload
**CVSS v3.1:** 9.8 (Critical) — AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
**CWE:** CWE-434 (Unrestricted Upload of File with Dangerous Type)

The plugin's SaaS connector feature exposes a file upload handler intended to receive branding assets — logos and banner images — pushed from the vendor's own dashboard to customer WordPress sites. The handler decodes a base64-encoded request body and writes the result to `wp-content/uploads/` under an attacker-supplied filename.

Nothing in the handler validates the incoming data. The filename is not checked against an extension allowlist. The decoded content is not inspected. The destination path is not sanitized. The intended security boundary — "only the vendor's SaaS service calls this endpoint" — was never actually enforced. The endpoint is reachable from the public internet, and the request it accepts is an ordinary HTTP POST.

The CVSS vector carries a Scope change (S:C) because the vulnerability crosses a privilege boundary: the upload handler runs in the application's security context, but the payload it writes executes in the server's.

The fix in 4.4.2 was not to patch the endpoint but to remove it and rebuild the connector's trust model around JWT ownership binding and HMAC request signing. That response is itself a statement about how severe the vendor judged the issue to be.

**What the writeup covers:** the vulnerable upload handler and its base64 decode path, the difference between the intended trust boundary and the enforced one, the patch's replacement of the endpoint with a signed-and-bound connector, and the specific impact on default WordPress deployments where `wp-content/uploads/` permits PHP execution.

---

### CVE-2026-3891 — Pix for WooCommerce Unauthenticated File Upload via Exposed Nonce Endpoint

**Component:** Pix for WooCommerce (plugin slug `payment-gateway-pix-for-woocommerce`)
**Affected versions:** TBD
**Class:** Unauthenticated file upload chained through an exposed nonce-generation endpoint
**CWE:** CWE-434 (Unrestricted Upload of File with Dangerous Type), CWE-862 (Missing Authorization)

This vulnerability is a two-request chain, and the first request is the interesting part.

The plugin registers an AJAX action named `lkn_pix_for_woocommerce_generate_nonce`. The action accepts an `action_name` parameter and returns a WordPress nonce for whichever action name is supplied. It does not verify that the caller has any capability. It does not check whether the requested nonce is for an action the caller should be allowed to invoke. It hands out a valid nonce for any action name the attacker names, to any unauthenticated request that reaches `admin-ajax.php`.

The second AJAX action, `lkn_pix_for_woocommerce_c6_save_settings`, is the upload handler. It accepts a multipart POST with a file field named `certificate_crt_path`. The file is written to:

    /wp-content/plugins/payment-gateway-pix-for-woocommerce/Includes/files/certs_c6/

No extension allowlist is applied. No MIME validation is performed. The handler trusts the nonce check as the sole gate — which would be fine if the nonce could only be obtained by an authorized user. Because the first action exposes nonce generation to anyone, the second action's nonce check provides no real protection. A valid nonce is trivially obtained, and once it is, the upload succeeds.

The result is unauthenticated remote code execution: an attacker uploads a PHP file, requests it, and PHP executes in the web server's context. The exploit is two POST requests against `admin-ajax.php` with no cookies, no credentials, and no user interaction.

The class of bug is the same as the other two entries in this repository — an endpoint designed around a trust assumption that was never enforced — but the mechanism is different. CVE-2020-25213 exposes the upload handler directly. CVE-2026-82970 exposes it because the intended authentication was never implemented. This one exposes it because the authentication mechanism it relies on is itself available to unauthenticated callers.

**What the writeup covers:** the nonce-generation action and why exposing it nullifies the upload endpoint's protection, the multipart upload request and its parameters, the plugin's files directory as an executable path, and the patch that restricts nonce generation to capable users.

---

## Repository Structure

    wordpress-security-research/
    ├── CVE-2020-25213-wp-file-manager/
    │   ├── README.md
    │   ├── ANALYSIS.md
    │   ├── METHODOLOGY.md
    │   ├── requirements.txt
    │   ├── src/
    │   │   ├── exploit.py
    │   │   └── lib/
    │   │       ├── __init__.py
    │   │       ├── http_client.py
    │   │       └── logger.py
    │   ├── tests/
    │   │   └── test_exploit.py
    │   └── evidence/
    │       └── exploitation.log
    ├── CVE-2026-82970-wp-cookie-notice/
    │   ├── README.md
    │   ├── ANALYSIS.md
    │   ├── METHODOLOGY.md
    │   ├── requirements.txt
    │   ├── src/
    │   │   ├── exploit.py
    │   │   └── lib/
    │   │       ├── __init__.py
    │   │       ├── http_client.py
    │   │       └── logger.py
    │   └── evidence/
    ├── CVE-2026-3891-pix-for-woocommerce/
    │   ├── README.md
    │   ├── ANALYSIS.md
    │   ├── METHODOLOGY.md
    │   ├── requirements.txt
    │   ├── src/
    │   │   ├── exploit.py
    │   │   └── lib/
    │   │       ├── __init__.py
    │   │       ├── http_client.py
    │   │       └── logger.py
    │   └── evidence/
    ├── .github/workflows/lint.yml
    ├── CONTRIBUTING.md
    ├── SECURITY.md
    ├── LICENSE
    └── README.md

Each CVE's own `README.md` is the primary writeup for that finding. The sections above are summaries — the full technical detail, patch diffs, and reproduction steps live in the individual writeups.

---

## Research Methodology

The structure of each writeup is fixed. This is not a stylistic choice — it forces the analysis to answer the questions that actually matter and prevents the writeup from drifting into vague description.

### 1. Scope

Affected versions, fixed version, CVSS vector, CWE classification, and the public disclosure timeline.

### 2. Root Cause

The vulnerable code, cited by file and function. This section answers: *what specific line runs, with what input, and why is that a problem?* The writeup names the handler, traces the request through it, and identifies the check that should have been there and was not.

### 3. Patch Analysis

The vendor's fix, extracted by diffing the vulnerable and patched versions. A patch that adds input validation and a patch that removes the endpoint entirely are different statements about the severity of the issue, and the writeup distinguishes between them.

### 4. Reproduction

The exact steps to trigger the finding. Every command is provided. Expected output is captured. Non-obvious details — a specific header, a particular parameter name, a required ordering — are called out.

### 5. Impact

What an attacker actually gains in a realistic deployment: default configuration, common hosting environment, the attacker's actual network position.

### 6. Mitigation

The vendor fix, plus compensating controls for environments where patching is not immediate, plus detection opportunities for environments where the fix may never be applied.

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

If a claim cannot be backed by one of those, it does not appear in the writeup.

---

## Getting Started

### Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.9+ | Required to run the reproducers |
| Git | any recent | Only needed if cloning rather than downloading |

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

Dependencies are pinned to specific versions. Updates are made deliberately and tested before being merged.

### Running Tests

    cd CVE-2020-25213-wp-file-manager
    pip install pytest
    pytest tests/

Unit tests cover helper logic — argument parsing, version comparison, session configuration. Integration testing is performed manually and captured in the `evidence/` directory of each CVE.

---

## Reading Order

Read the entries in the order they appear in the [Contents](#contents) table.

By vulnerability class:

- **Directly exposed upload handler:** CVE-2020-25213
- **Missing authorization on the upload endpoint itself:** CVE-2026-82970
- **Missing authorization on an endpoint that gates the upload endpoint:** CVE-2026-3891

By plugin:

- **WP File Manager:** CVE-2020-25213
- **WP Cookie Consent (`gdpr-cookie-consent`):** CVE-2026-82970
- **Pix for WooCommerce (`payment-gateway-pix-for-woocommerce`):** CVE-2026-3891

---

## Scope

This repository covers WordPress plugin vulnerabilities only. The scope is deliberate: a focused repository with well-documented entries is more useful than a broad one with shallow entries. The analysis methodology is transferable to other plugin families and other web application stacks, but the entries here stay within WordPress.

Related work outside this repository's scope is available in a separate repository. See the profile for links.

---

## Ethics and Legal

> [!WARNING]
> **All research in this repository is conducted against locally hosted, intentionally vulnerable environments.** No live, third-party, or production systems are targeted at any point. Where a CVE has an existing public disclosure, that disclosure is cited and the timeline is documented in the relevant writeup. Nothing here is zero-day.

### Rules for Use

- Do not run any code in this repository against a system you do not own or do not have explicit written authorization to test.
- Do not open issues, pull requests, or discussions describing how to adapt the code to attack live targets.
- Do not redistribute the code with the intent of enabling its use against systems without authorization.

The reproducer code is intentionally minimal and non-destructive. The payloads echo a marker string — they do not open shells, exfiltrate data, or perform any other weaponized behavior. This is a deliberate design choice.

---

## Disclosure Policy

Every CVE covered in this repository was disclosed publicly before this repository was published. No original vulnerability disclosure is claimed here.

The disclosure timeline for each CVE is documented in that CVE's own `README.md` and includes the original disclosure date, the vendor fix release date, the date the reproduction was performed, and the date the writeup was published.
