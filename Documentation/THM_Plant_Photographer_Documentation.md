# Plant Photographer — Security Assessment Walkthrough

> **Platform:** TryHackMe  
> **Room:** Plant Photographer  
> **Difficulty:** Hard  
> **Classification:** Web Application / Linux Security  
> **Primary Chain:** SSRF-style fetch → LFI → source disclosure → Werkzeug PIN reconstruction → debug-console RCE

## Executive Summary

Plant Photographer is a chained web-security exercise centered on a server-controlled file download mechanism. The endpoint accepts a destination supplied by the client and appends an application-controlled path to it. By manipulating the destination and terminating the appended path with a URL fragment, the server-side fetch client can be coerced into reading arbitrary local files.

The resulting file-read primitive exposes the application's source and confirms that Flask is running with debug mode enabled. The Werkzeug debugger can then be accessed after reconstructing its PIN from application and host metadata obtained through the same local-file primitive. Correct handling of the debugger's trust cookie enables Python execution.

The application process executes as root, so the debugger itself provides the final privileged execution context.

> **Public portfolio edition:** challenge flags and answer strings are intentionally redacted.

---

## 1. Scope and Objectives

Testing is restricted to the authorized TryHackMe target. The objectives are to identify the externally exposed services, validate the web attack surface, establish local file disclosure, recover application internals, authenticate to the debugger and validate code execution.

---

## 2. Reconnaissance

RustScan identifies TCP ports 22 and 80.

Nmap then fingerprints the services:

```text
22/tcp  open  ssh
80/tcp  open  http
        Werkzeug httpd 0.16.0
        Python 3.10.7
```

The Werkzeug banner is notable because it suggests a Flask development-server deployment. It does not by itself prove the debugger is exposed, but it provides a strong hypothesis to validate during web enumeration.

---

## 3. Web Application Enumeration

The main page exposes an administrator endpoint and a parameterized download feature.

Relevant routes include:

```text
/admin
/download
/console
```

The `/admin` route returns a localhost-only message when requested remotely.

The `/download` route accepts both a `server` parameter and an identifier. Because the parameter resembles a backend destination rather than a simple file identifier, it becomes the primary SSRF test candidate.

---

## 4. Validating the Download Primitive

A legitimate request demonstrates that the application genuinely performs a server-side retrieval.

The next step is intentionally malformed input. Supplying an invalid port causes the backend to throw an exception.

Because the service runs in Werkzeug debug mode, the exception is rendered as a detailed traceback instead of a generic error.

This reveals source-level implementation details.

---

## 5. Debug Traceback as an Information Disclosure

The exposed traceback reveals a route pattern equivalent to:

```python
crl.URL = server + '/public-docs-k057230990384293/' + filename
```

This is significant for two reasons:

1. the client controls the base `server` value;
2. the application appends its own suffix without first validating the scheme or canonical target.

The traceback also exposes the application path and other sensitive implementation information.

The response therefore turns a normal download feature into an SSRF/LFI investigation target.

---

## 6. Converting SSRF into Local File Disclosure

The backend client accepts the `file://` scheme.

A direct request for `/etc/passwd` initially fails because the application appends its own path suffix:

```text
file:///etc/passwd/public-docs-k057230990384293/1.pdf
```

That failure is useful evidence: it confirms the local-file scheme is being processed by the backend.

The path can be terminated using a URL fragment:

```text
file:///etc/passwd#
```

URL-encoding the fragment delimiter as `%23` allows it to survive the outer query-string parsing.

Conceptually:

```text
attacker-controlled path
        +
%
23 fragment delimiter
        +
application suffix

→ suffix no longer changes the selected local resource
```

The result is arbitrary local file disclosure.

---

## 7. Application Source Recovery

The application source is stored at:

```text
/usr/src/app/app.py
```

Retrieving it confirms:

```python
@app.route("/admin")
def admin():
    if request.remote_addr == '127.0.0.1':
        return send_from_directory('private-docs', 'flag.pdf')
```

and:

```python
app.run(host='0.0.0.0', port=8087, debug=True)
```

The source confirms two major facts:

- the `/admin` restriction is based on the actual peer address;
- the application is explicitly running with debug enabled.

Because local file disclosure already exists, the localhost restriction no longer needs to be bypassed. The protected artifact can be read directly from disk.

---

## 8. Protected Administrator Artifact

The administrator route serves:

```text
/usr/src/app/private-docs/flag.pdf
```

The file is retrieved through the established local-file primitive.

The challenge flag is intentionally redacted from this public portfolio.

---

## 9. Identifying the Final Stage

The remaining flag is stored in a randomized text filename within the application directory.

Blind guessing is ineffective because the filename contains a high-entropy numeric suffix.

This changes the objective:

> obtain code execution, then enumerate the directory directly.

---

## 10. Werkzeug Debug PIN Analysis

The exposed `/console` endpoint is the Flask/Werkzeug interactive debugger.

The debugger is protected by a generated PIN rather than being openly executable.

The target is running Werkzeug 0.16.0, so the PIN algorithm must be derived from the actual installed source rather than copied from a generic script.

The required information consists of:

### Public bits

```text
application user
module name
application class
absolute Flask app.py path
```

### Private bits

```text
uuid.getnode()-equivalent MAC value
container identifier
```

The existing local-file primitive provides access to these values.

---

## 11. Recovering Host Metadata

The application environment is inspected through:

```text
/proc/self/environ
```

The network interface is identified through:

```text
/proc/net/arp
```

The interface MAC address is then read from:

```text
/sys/class/net/eth0/address
```

and converted from hexadecimal to the decimal integer used by `uuid.getnode()`.

The normal `/etc/machine-id` path is also checked. When it is unavailable in the container, the relevant identifier is taken from:

```text
/proc/self/cgroup
```

This is an important implementation detail because the effective PIN inputs depend on how Werkzeug identifies the host/container.

---

## 12. Matching the Exact Hash Implementation

Before generating the PIN, the target's Werkzeug source is inspected.

The specific version uses its own PIN hashing behavior, including its legacy digest/salt construction.

This matters because a reconstruction script written for a newer Werkzeug release may produce a different result.

The exact target implementation is therefore treated as authoritative.

---

## 13. PIN Reconstruction

The recovered values are fed into a local reconstruction script.

The process is:

```text
public bits
    +
private bits
    ↓
Werkzeug hash state
    ↓
cookie identifier
    ↓
PIN integer
    ↓
formatted debugger PIN
```

The resulting PIN is intentionally omitted from the public documentation.

---

## 14. Debugger Authentication

The debugger exposes a session-specific secret in the console page.

The PIN-authentication endpoint is called with both the secret and reconstructed PIN.

A successful response indicates that PIN authentication has completed.

However, this does not mean every later request is automatically trusted.

The debugger also expects the trust cookie established by the successful PIN-authentication request.

---

## 15. Debugger Session-State Failure

Initial attempts to execute commands return ordinary application responses.

The reason is that independent HTTP requests do not carry forward the debugger's trust cookie.

The client-side debugger code shows that command requests are sent with:

```text
__debugger__=yes
cmd=<command>
frm=<frame id>
s=<secret>
```

The server-side dispatch logic additionally requires the debugger's trusted session state.

This is why a single persistent cookie jar is necessary.

---

## 16. Establishing Code Execution

The debugger is first validated with a harmless expression:

```python
1 + 1
```

which returns:

```text
2
```

A subsequent identity check demonstrates execution context:

```python
__import__('os').popen('id').read()
```

The process reports root privileges.

No separate local privilege-escalation exploit is required because the Flask application itself is already running as root.

---

## 17. Final Directory Enumeration

With authenticated Python execution, the application directory can be listed directly:

```text
/usr/src/app
```

The listing reveals the randomized final flag filename.

This confirms why earlier dictionary-based filename guessing failed.

The final artifact is then read through the debugger.

The answer is intentionally removed from this public repository.

---

## 18. Attack Chain

```text
Werkzeug Fingerprint
        ↓
Endpoint Enumeration
        ↓
/download Analysis
        ↓
Malformed Server Request
        ↓
Debug Traceback
        ↓
Source Disclosure
        ↓
file:// Scheme
        ↓
URL-Fragment Termination
        ↓
Arbitrary Local File Read
        ↓
Flask Source Recovery
        ↓
debug=True Confirmation
        ↓
Host Metadata Recovery
        ↓
Werkzeug PIN Reconstruction
        ↓
Debugger PIN Authentication
        ↓
Cookie-Preserved Requests
        ↓
Python RCE
        ↓
Root Execution Context
        ↓
Randomized Final File Discovery
```

---

## 19. Security Findings

| Finding | Severity | Impact |
|---|---:|---|
| Server-controlled backend destination | High | Allows attacker-influenced server-side retrieval |
| Local file disclosure via `file://` | Critical | Arbitrary filesystem reads |
| Debug traceback disclosure | High | Reveals source, paths and secrets |
| Exposed Flask debugger | Critical | Provides an interactive execution interface |
| Derivable Werkzeug PIN | Critical | Weakens debugger authentication |
| Root application process | Critical | Debugger RCE becomes root-level compromise |

---

## 20. Root Cause Analysis

### Unsafe URL construction

Attacker-controlled input is concatenated into a backend URL before scheme and destination validation.

### Unrestricted backend schemes

The backend client accepts the local-file scheme.

### Production debug mode

Flask/Werkzeug exposes detailed tracebacks and the interactive debugger.

### Predictable debugger authentication inputs

The PIN is reproducible from information obtainable through the file-read primitive.

### Excessive process privileges

The web process executes as root, amplifying the impact of debugger compromise.

---

## 21. Defensive Recommendations

### SSRF and LFI

- Permit only required URL schemes.
- Allowlist valid destination hosts.
- Reject local and internal network targets.
- Canonicalize resource paths.
- Avoid direct concatenation of user-controlled URLs.

### Flask / Werkzeug

- Never deploy with `debug=True`.
- Remove `/console` from production.
- Replace development servers with a properly configured production WSGI stack.
- Disable interactive error pages.

### Secrets

- Do not embed API keys in source.
- Rotate any credentials exposed through debug traces.
- Centralize secret storage.

### Privilege Separation

- Run the application under a dedicated unprivileged account.
- Apply filesystem permissions and container restrictions.
- Restrict egress and local resource access.

---

## 22. Lessons Learned

**Validate behavior, not labels.** A Werkzeug banner is a clue; the traceback and source confirmed the actual deployment state.

**Use controlled failure.** Deliberately invalidating a download request exposed implementation details that a successful request concealed.

**Read error messages as evidence.** The `pycurl` failure proved that the local-file scheme was being processed and explained why the first payload did not work.

**Match library versions.** Werkzeug's PIN algorithm is implementation-specific; reading the target version avoided a false reconstruction.

**Preserve session state.** Debugger authentication and debugger command authorization were tied to different pieces of state.

**Stop guessing after code execution.** Enumerating `/usr/src/app` immediately solved the randomized filename problem.

---

## 23. Evidence and Public Flag Policy

Only supplied evidence is treated as real evidence in this portfolio.

<p align="center">
  <img src="../docs/assets/room-completed.png" width="95%" alt="TryHackMe Plant Photographer room completion evidence">
</p>

**Figure 01 — Supplied TryHackMe room-completion evidence.**

The attack-chain graphic in the repository is a conceptual diagram and is explicitly not a screenshot of target activity.

Challenge answers remain redacted:

```text
Flag 01 → [REDACTED]
Flag 02 → [REDACTED]
Flag 03 → [REDACTED]
```

---

## 24. Conclusion

Plant Photographer demonstrates how a server-side download feature can become a complete application compromise when URL validation, error handling, debug configuration and process privilege boundaries are weak.

The attack did not depend on breaking cryptography or discovering an exotic kernel exploit. The decisive progression was:

**enumerate → validate → disclose → reconstruct → authenticate → execute → enumerate**

The strongest defensive lesson is architectural: security controls must remain secure across protocol boundaries, deployment modes, library versions and operating-system privilege contexts.

---

**Author:** Anurag Ravankar  
**Purpose:** Authorized TryHackMe research and cybersecurity portfolio
