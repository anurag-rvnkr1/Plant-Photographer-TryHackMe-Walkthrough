# ✈️ Plant Photographer — TryHackMe Penetration Testing Walkthrough

<p align="center">
  <img src="assets/room-completed.png" width="100%" alt="Plant Photographer room completion">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TryHackMe-Plant%20Photographer-red?style=for-the-badge&logo=tryhackme"/>
  <img src="https://img.shields.io/badge/Linux-Hard-orange?style=for-the-badge&logo=linux&logoColor=black"/>
  <img src="https://img.shields.io/badge/Category-Web%20Security-blue?style=for-the-badge"/>
</p>

---

## 📌 Overview

Plant Photographer is a Hard-difficulty web/Linux assessment built around a server-controlled download feature that can be transformed into local file disclosure and then chained into Flask debugger compromise.

## 📑 Table of Contents

- Executive Summary
- Reconnaissance
- Web Enumeration
- SSRF / LFI
- Source Disclosure
- Werkzeug PIN Reconstruction
- Debug Console
- RCE
- Security Findings
- Remediation
- Lessons Learned
- Conclusion

---

# Executive Summary

```text
Recon
 ↓
Werkzeug Fingerprint
 ↓
Download Endpoint
 ↓
Debug Traceback
 ↓
file:// LFI
 ↓
Source Disclosure
 ↓
Werkzeug Metadata Recovery
 ↓
PIN Reconstruction
 ↓
Debugger Authentication
 ↓
Python RCE
 ↓
Root Context
```

The public edition intentionally redacts all challenge flags.

---

# 1. Reconnaissance

The target exposes SSH and HTTP. Nmap identifies the HTTP service as Werkzeug 0.16.0 running on Python 3.10.7.

That development-server fingerprint makes debug functionality a high-priority hypothesis for validation.

---

# 2. Web Application Enumeration

The site exposes:

```text
/admin
/download
/console
```

The `/download` endpoint is particularly interesting because it accepts a `server` parameter that resembles a backend destination.

---

# 3. SSRF → LFI

A malformed backend destination produces a Werkzeug traceback. The error discloses the application's URL-building logic.

The backend accepts `file://`, and the application's appended path can be neutralized with a URL fragment encoded as `%23`.

This yields arbitrary local-file reads.

---

# 4. Source Disclosure

Reading the application source confirms:

```text
/usr/src/app/app.py
```

and:

```text
debug=True
```

The same source shows the localhost check protecting `/admin` and the filesystem location of its private artifact.

The localhost restriction becomes irrelevant once direct local-file reads are available.

---

# 5. Werkzeug PIN Reconstruction

The debugger is not simply open; it requires a generated PIN.

The required application and host values are recovered through the LFI primitive and matched against the exact Werkzeug 0.16.0 implementation.

The resulting PIN is withheld from this public portfolio.

---

# 6. Debug Console Authentication

A session-specific debugger secret is combined with the recovered PIN.

After PIN authentication, the debugger trust cookie must be preserved for later requests.

Without the cookie:

```text
correct PIN + correct secret
≠
authorized command execution
```

---

# 7. Remote Code Execution

With the trusted debugger session established, a harmless Python expression is used to confirm execution.

The execution context reports root privileges.

Therefore the final code-execution stage does not require a separate privilege-escalation exploit.

---

# 8. Final Enumeration

The application directory is listed directly through Python execution, revealing the randomized final file name.

The public portfolio does not disclose the resulting flag.

---

# 9. Security Findings

| Finding | Severity |
|---|---|
| `file://`-enabled server-side fetch | 🔴 Critical |
| Exposed Flask debugger | 🔴 Critical |
| Derivable Werkzeug PIN | 🔴 Critical |
| Source-code disclosure | 🟠 High |
| Debug traceback exposure | 🟠 High |
| Root Flask process | 🔴 Critical |

---

# 10. Remediation

- Allowlist backend destinations and URL schemes.
- Block local/internal resource access.
- Never concatenate attacker-controlled destinations into server URLs.
- Disable Flask debug mode in production.
- Remove public access to `/console`.
- Run the application as a non-root user.
- Protect source/configuration files.
- Rotate secrets exposed through tracebacks.

---

# 11. Evidence

<p align="center">
  <img src="assets/room-completed.png" width="95%" alt="TryHackMe completion evidence">
</p>

**Figure 01 — Supplied room-completion evidence.**

The `attack-chain.png` image in this repository is a conceptual visualization, not target evidence.

---

# 12. Flag Policy

```text
Flag 01 → [REDACTED]
Flag 02 → [REDACTED]
Flag 03 → [REDACTED]
```

---

# 13. Conclusion

The challenge is an example of chained application-security weaknesses rather than a single isolated bug.

The reusable methodology is:

**enumerate → validate → disclose → reconstruct → authenticate → execute → enumerate**

---

## Repository

[GitHub Repository](https://github.com/anurag-rvnkr1/Plant-Photographer-TryHackMe-Walkthrough)

**Author:** Anurag Ravankar
