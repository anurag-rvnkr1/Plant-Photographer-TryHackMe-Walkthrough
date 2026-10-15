<div class="ctf-hero">

  <h1>Plant Photographer</h1>

  <p>
    A multi-stage TryHackMe security assessment examining how unsafe
    server-side resource retrieval, local file disclosure, exposed
    development debugging, and excessive process privileges can combine
    to produce root-level code execution.
  </p>

  <div class="ctf-badges">
    <span class="ctf-badge">TryHackMe</span>
    <span class="ctf-badge">Hard</span>
    <span class="ctf-badge">Web Application Security</span>
    <span class="ctf-badge">SSRF</span>
    <span class="ctf-badge">Python / Flask</span>
    <span class="ctf-badge">Linux</span>
    <span class="ctf-badge">Remote Code Execution</span>
  </div>

</div>

## Executive Summary

Plant Photographer demonstrates how weaknesses across a Python-backed web application can be chained into a complete application compromise.

The assessment begins with TCP service discovery and web application enumeration. A download endpoint accepts a client-controlled `server` parameter, while malformed requests expose detailed Werkzeug tracebacks. Inspection of the disclosed implementation reveals unsafe URL construction and insufficient restrictions on the backend retrieval mechanism.

The supplied challenge evidence establishes that the backend accepts the `file://` scheme. By examining how the application appends a fixed path and filename, the assessment identifies a URL-fragment handling weakness that can turn server-side resource retrieval into local file disclosure.

The recovered Flask source reveals that debugging is enabled and identifies the local path of a protected challenge artifact. Additional application and host metadata support analysis of the Werkzeug 0.16.0 debugger PIN algorithm. The documented workflow then proceeds through debugger authentication, preservation of session state, Python expression evaluation, and verification of the execution context.

The application process runs with root privileges in the supplied challenge material. Consequently, successful code execution has a substantially greater impact than it would under a dedicated unprivileged service account. Directory enumeration from the established execution context provides the final challenge artifact location.

This report organizes the supplied challenge notes into a technical assessment record, emphasizing the observed behavior, attack-chain dependencies, security implications, and remediation measures.

> **Evidence boundary:** This document is based on the supplied challenge write-up. Where the source describes an observation or result without including a corresponding raw output or screenshot, that result is attributed to the supplied assessment rather than presented as independently reverified evidence.

### Quick Overview

<div class="ctf-card-grid">

  <div class="ctf-card">
    <div class="ctf-card-title">Platform</div>
    <div class="ctf-card-value">TryHackMe</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Room</div>
    <div class="ctf-card-value">Plant Photographer</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Difficulty</div>
    <div class="ctf-card-value">Hard</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Application Stack</div>
    <div class="ctf-card-value">Python / Flask / Werkzeug</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Initial Attack Surface</div>
    <div class="ctf-card-value">HTTP Download Endpoint</div>
  </div>

  <div class="ctf-card">
    <div class="ctf-card-title">Demonstrated Impact</div>
    <div class="ctf-card-value">Root-Level Code Execution</div>
  </div>

</div>

### Assessment at a Glance

| Attribute | Assessment Detail |
|---|---|
| Platform | TryHackMe |
| Room | Plant Photographer |
| Difficulty | Hard |
| Assessment type | Authorized Capture the Flag (CTF) |
| Target operating system | Linux |
| Web server fingerprint | Werkzeug |
| Framework | Flask |
| Programming language | Python |
| Reported runtime | Python 3.10.7 |
| Reported Werkzeug version | 0.16.0 |
| Primary entry point | `/download` |
| Related interfaces | `/admin`, `/console` |
| Initial vulnerability class | Unsafe server-side resource retrieval |
| File access impact | Local file disclosure |
| Escalation mechanism | Exposed Werkzeug debugger |
| Reported execution context | Root |
| Publication policy | Challenge-specific answers redacted |

### Key Findings

<div class="key-finding">
  <div class="key-finding-title">Primary Finding — Unsafe Resource Retrieval</div>

  The download endpoint constructs a backend resource reference using client-controlled input. The supplied challenge material demonstrates that local-file scheme handling and URL-fragment behavior expose resources that should not be reachable through the public download interface.
</div>

<div class="key-finding">
  <div class="key-finding-title">Escalation Finding — Exposed Debugger</div>

  The recovered Flask source confirms `debug=True`. The supplied assessment describes reconstructing the version-specific Werkzeug PIN from recovered application and host metadata, authenticating to the debugger, and establishing Python code execution.
</div>

<div class="key-finding">
  <div class="key-finding-title">Impact Finding — Excessive Process Privileges</div>

  The documented process identity indicates that the Flask application executes as root. This privilege configuration amplifies debugger compromise into root-level execution within the observed challenge environment.
</div>

### Attack Chain Overview

<div class="attack-chain">
  <div class="attack-step">Reconnaissance</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Web Enumeration</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Unsafe Retrieval</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Local File Disclosure</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Source Recovery</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Debugger Analysis</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Python RCE</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Root-Level Impact</div>
</div>

> **Public portfolio policy:** Challenge flags, the reconstructed debugger PIN, and other challenge-specific answer values are intentionally redacted. Technical findings and remediation guidance are retained.

---

## Navigation

<div class="ctf-toc">
<div class="ctf-toc-title">Documentation Map</div>
<ul>
<li><a href="#mission-and-assessment-overview">Mission and Assessment Overview</a></li>
<li><a href="#scope-and-responsible-use">Scope and Responsible Use</a></li>
<li><a href="#assessment-objectives">Assessment Objectives</a></li>
<li><a href="#tools-and-techniques">Tools and Techniques</a></li>
<li><a href="#target-and-attack-surface">Target and Attack Surface</a></li>
<li><a href="#1-reconnaissance">1. Reconnaissance</a></li>
<li><a href="#2-web-application-enumeration">2. Web Application Enumeration</a></li>
<li><a href="#3-validating-the-download-primitive">3. Validating the Download Primitive</a></li>
<li><a href="#4-debug-traceback-and-information-disclosure">4. Debug Traceback and Information Disclosure</a></li>
<li><a href="#5-local-file-disclosure">5. Local File Disclosure</a></li>
<li><a href="#6-application-source-recovery">6. Application Source Recovery</a></li>
<li><a href="#7-protected-administrator-artifact">7. Protected Administrator Artifact</a></li>
<li><a href="#8-identifying-the-final-challenge-stage">8. Identifying the Final Challenge Stage</a></li>
<li><a href="#9-werkzeug-debug-pin-analysis">9. Werkzeug Debug PIN Analysis</a></li>
<li><a href="#10-recovering-host-metadata">10. Recovering Host Metadata</a></li>
<li><a href="#11-matching-the-exact-hash-implementation">11. Matching the Exact Hash Implementation</a></li>
<li><a href="#12-pin-reconstruction">12. PIN Reconstruction</a></li>
<li><a href="#13-debugger-authentication">13. Debugger Authentication</a></li>
<li><a href="#14-debugger-session-state">14. Debugger Session State</a></li>
<li><a href="#15-establishing-code-execution">15. Establishing Code Execution</a></li>
<li><a href="#16-final-directory-enumeration">16. Final Directory Enumeration</a></li>
<li><a href="#end-to-end-attack-chain">End-to-End Attack Chain</a></li>
<li><a href="#security-findings-and-severity">Security Findings and Severity</a></li>
<li><a href="#root-cause-analysis">Root Cause Analysis</a></li>
<li><a href="#remediation-and-defensive-recommendations">Remediation and Defensive Recommendations</a></li>
<li><a href="#lessons-learned">Lessons Learned</a></li>
<li><a href="#evidence-and-publication-policy">Evidence and Publication Policy</a></li>
<li><a href="#conclusion">Conclusion</a></li>
<li><a href="#references-and-tools">References and Tools</a></li>
</ul>
</div>

---

## Mission and Assessment Overview

The objective of Plant Photographer is to evaluate a Python-backed web application and determine whether weaknesses in resource retrieval, error handling, debugging configuration, and process privilege separation can be chained into a full compromise.

The investigation follows the technical sequence documented in the supplied assessment:

1. Discover exposed TCP services.
2. Enumerate application routes and user-controlled parameters.
3. Investigate server-side download behavior.
4. Examine detailed errors for implementation information.
5. Validate the local-file disclosure behavior described in the challenge notes.
6. Recover application source and identify its runtime configuration.
7. Investigate the exposed Werkzeug debugger.
8. Recover the host and application metadata used by the debugger PIN algorithm.
9. Validate authentication and preserve debugger session state.
10. Verify Python expression evaluation and the effective process identity.
11. Enumerate the application directory to locate the remaining challenge artifact.

Each stage depends on observations from the preceding stage. The objective is not simply to demonstrate a successful endpoint request, but to understand how the initial weakness changes the security boundaries available to an attacker.

### Technical Domains

| Domain | Relevance to the Assessment |
|---|---|
| Network reconnaissance | Identifies exposed TCP services |
| Web application enumeration | Maps the routes and parameters that influence application behavior |
| SSRF analysis | Examines client-controlled server-side destination selection |
| URL parsing | Investigates scheme restrictions, appended paths, and fragment handling |
| Local file disclosure | Establishes the impact of unsafe backend resource retrieval |
| Python source analysis | Reveals application logic, file locations, and configuration |
| Flask debugging | Identifies development configuration exposed to untrusted users |
| Werkzeug analysis | Examines version-specific debugger PIN behavior |
| HTTP session handling | Explains preservation of debugger trust state |
| Python code execution | Validates the capabilities available through the debugger |
| Linux privilege analysis | Establishes the impact of root-level application execution |
| Remediation planning | Translates the observed weaknesses into relevant defensive controls |

---

## Scope and Responsible Use

All testing described here is restricted to the authorized TryHackMe Plant Photographer environment.

The material is intended for cybersecurity education, authorized penetration testing, and defensive security analysis. Commands and implementation details should not be used against systems without explicit permission.

### In Scope

- TCP service discovery and service fingerprinting.
- HTTP route and parameter enumeration.
- Investigation of server-side resource retrieval.
- Analysis of error responses and disclosed source information.
- Local-file disclosure validation within the assigned lab.
- Examination of Flask and Werkzeug configuration.
- Analysis of debugger PIN-generation inputs.
- Debugger authentication and session-state behavior.
- Verification of process identity and privilege level.
- Enumeration of challenge artifacts.

### Out of Scope

- Testing unrelated public-facing systems.
- Accessing production applications without authorization.
- Collecting third-party data or unrelated credentials.
- Publishing recovered challenge secrets.
- Establishing persistence or conducting activities outside the challenge objectives.

**Responsible testing principle:** Use the minimum access necessary to demonstrate impact, preserve evidence carefully, and remain within the authorized scope.

---

## Assessment Objectives

The assessment addresses the following questions:

- Which TCP services are exposed by the target?
- Does the web application accept a user-controlled backend destination?
- How does the download handler construct the final resource reference?
- Does the retrieval mechanism permit local-file access?
- Do error responses disclose useful implementation details?
- Does the local-file-read primitive expose application source or host metadata?
- Is the Flask/Werkzeug interactive debugger reachable?
- Which application and host attributes contribute to the version-specific debugger PIN?
- Does authenticated debugger interaction require preserved session state?
- What privileges does the application process possess?
- How can the documented vulnerability chain be disrupted through application and operating-system controls?

---

## Tools and Techniques

The following tools and techniques are identified in the supplied challenge notes. No additional tools are assumed to have been used.

| Tool or Technique | Documented Purpose |
|---|---|
| RustScan | Rapid TCP port discovery |
| Nmap | Service fingerprinting and version enumeration |
| Web browser | Application navigation and route discovery |
| HTTP request inspection | Analysis of download parameters and responses |
| Python source review | Examination of URL construction and debug configuration |
| Local file disclosure analysis | Recovery of application and host metadata |
| Werkzeug source inspection | Analysis of the target version's PIN-generation behavior |
| Python | Local PIN reconstruction and debugger interaction |
| Linux `/proc` inspection | Examination of process, environment, and container metadata |
| Linux filesystem inspection | Enumeration of application files |
| Debugger expression evaluation | Validation of code execution and process identity |

### Methodology

The assessment uses a progressive testing workflow:

1. Establish the attack surface.
2. Identify the application functionality influenced by untrusted input.
3. Observe abnormal behavior and use the resulting evidence to investigate the implementation.
4. Validate the scope and impact of the resource-disclosure primitive.
5. Correlate recovered source and host metadata with debugger behavior.
6. Verify the effective execution context.
7. Identify the appropriate defensive controls for each stage.

---

## Target and Attack Surface

### Target Services

The supplied service enumeration reports the following:

| Port | Service | Reported Information | Assessment Relevance |
|---:|---|---|---|
| 22/TCP | SSH | Open | Identified as an exposed remote administration service |
| 80/TCP | HTTP | Open | Primary web application entry point |
| 80/TCP | Werkzeug HTTP server | Werkzeug 0.16.0; Python 3.10.7 reported | Supports further investigation of the Python application stack |

No target IP address or hostname was supplied in the provided write-up, so neither is added here.

### Application Routes

| Endpoint | Observed Behavior | Security Relevance |
|---|---|---|
| `/admin` | Returns a localhost-only message when accessed remotely | Uses a source-address restriction |
| `/download` | Accepts a `server` parameter and an identifier | Influences backend resource retrieval |
| `/console` | Associated with the Flask/Werkzeug debugger | Exposes a high-risk diagnostic interface |

The route descriptions above reflect the supplied assessment. The `/admin` response establishes the observed restriction but does not independently demonstrate a bypass of that restriction.

---

# Technical Walkthrough

## 1. Reconnaissance

### Objective

Identify exposed TCP services and fingerprint the web service to establish the initial attack surface.

### Methodology

The supplied assessment uses RustScan to discover open TCP ports, followed by Nmap for service identification and additional fingerprinting.

The documented output is summarized as follows:

```text
22/tcp  open  ssh
80/tcp  open  http
        Werkzeug httpd 0.16.0
        Python 3.10.7
```

### Observations

The target exposes SSH and HTTP. The Werkzeug banner indicates that the HTTP service is associated with a Python-based web stack.

The banner is an investigative clue, not proof by itself that the interactive debugger is enabled. Debugger configuration and reachability require separate validation.

### Security Relevance

Service discovery identifies the interfaces that should be investigated first. In this case, HTTP is prioritized because it provides the application functionality and parameters examined in the later stages.

### Result

The HTTP service becomes the primary investigation target. SSH remains part of the documented attack-surface inventory.

---

## 2. Web Application Enumeration

### Objective

Map the exposed application routes and identify inputs that may influence backend behavior.

### Methodology

The supplied notes describe exploration of the main application interface and the following routes:

```text
/admin
/download
/console
```

### `/admin`: Localhost-Only Response

A remote request to `/admin` returns a message indicating that the route is restricted to localhost.

This suggests the application evaluates the request source address before serving the protected resource.

At this stage, the observed response establishes the presence of an address-based condition; it does not establish that the condition was bypassed.

### `/download`: User-Controlled Destination

The download endpoint accepts a `server` parameter and an identifier. Because the supplied destination influences backend processing, this endpoint becomes the priority for additional testing.

A secure implementation should not allow an untrusted caller to select arbitrary schemes, hosts, or filesystem resources.

### `/console`: Debugger Interface

The supplied assessment associates `/console` with the Flask/Werkzeug debugger. Its configuration and authentication requirements are investigated after the initial resource-disclosure behavior is established.

### Result

The download endpoint is selected for detailed analysis because its parameter structure indicates that client-controlled input may influence server-side resource retrieval.

---

## 3. Validating the Download Primitive

### Objective

Determine how the download endpoint processes the supplied destination and responds to malformed input.

### Methodology

The supplied assessment establishes a baseline using a legitimate download request. A subsequent request containing an invalid destination, including an invalid port, produces a backend exception.

Because detailed Werkzeug debugging behavior is enabled, the response contains a traceback rather than only a generic error message.

### Observation

The traceback reveals information about the application's implementation and shows that the supplied destination reaches backend processing.

### Why the Result Matters

Malformed input can reveal more than whether a request succeeds or fails. A detailed exception may disclose:

- Application source locations.
- Framework and library behavior.
- Backend URL construction.
- Internal file paths.
- Resource retrieval logic.

These details provide information that can guide subsequent testing.

### Result

The malformed request establishes a documented path from user-controlled input to observable backend processing and implementation disclosure.

---

## 4. Debug Traceback and Information Disclosure

### Objective

Determine how the application constructs the backend destination and identify the missing validation boundary.

### Source Evidence

The supplied traceback exposes a route expression equivalent to:

```python
crl.URL = server + '/public-docs-k057230990384293/' + filename
```

### Analysis

The expression demonstrates two relevant characteristics:

1. The `server` value originates from client-controlled input.
2. The application appends a fixed path and filename to that value without first establishing that the completed resource reference is safe.

The traceback also discloses source-location and implementation details.

### Finding

**Unsafe server-side URL construction combined with debug information disclosure.**

When a backend resource location is constructed from untrusted input, the accepted scheme, destination, redirects, and normalized resource must all be controlled. A simple string concatenation operation does not enforce these security boundaries.

### Security Impact

The vulnerability may allow a caller to influence which resource the server attempts to retrieve. In the supplied challenge, the later analysis establishes that the accepted URL handling also exposes local files.

### Defensive Recommendations

- Avoid concatenating untrusted strings into complete URLs.
- Parse and validate URL components using an established library.
- Restrict schemes and permitted destinations.
- Validate resolved addresses and redirect destinations.
- Return generic external error messages.
- Store detailed tracebacks only in access-controlled logs.
- Disable interactive debug tracebacks in production.

---

## 5. Local File Disclosure

### Objective

Determine whether server-side resource retrieval can access local files and assess how the application-generated suffix influences resource selection.

### Initial Observation

The supplied assessment identifies that the backend retrieval mechanism accepts the `file://` scheme.

A direct local-file reference initially fails because the application appends its own path and filename. The resulting reference resembles:

```text
file:///etc/passwd/public-docs-k057230990384293/1.pdf
```

The failed request indicates that the selected path includes the application-controlled suffix.

### URL Fragment Handling

The next part of the investigation examines URL-fragment processing. A fragment begins with `#`. In standard URI handling, the fragment identifies a portion of the resource reference and is not part of the HTTP request target.

The supplied challenge notes describe using fragment handling to prevent the appended suffix from changing the intended local resource.

The documented input form is:

```text
file:///etc/passwd#
```

When the fragment delimiter must pass through an outer query-string parser, it is encoded as `%23` so that it can reach the relevant URL-processing stage.

Conceptually:

```text
Client-controlled destination
            |
            v
Application appends a fixed suffix
            |
            v
URL parsing and fragment handling
            |
            v
The appended suffix no longer changes
the selected resource
            |
            v
Local file disclosure
```

This behavior depends on the complete parsing and reconstruction sequence and on the backend retrieval library's URL semantics. A fragment delimiter should not be assumed to behave identically in every application.

### Finding

**Local file disclosure through unsafe server-side resource retrieval.**

### Security Impact

The primitive permits access to files readable by the application process, subject to filesystem permissions. In the supplied assessment, the disclosed source and host metadata support later investigation of the debugger.

### Defensive Recommendations

- Reject `file://` when the application only requires remote HTTP or HTTPS retrieval.
- Do not pass arbitrary client-controlled URLs directly to backend retrieval libraries.
- Validate the complete destination after parsing and normalization.
- Apply strict host and scheme allowlists.
- Prevent the download service from accessing unrelated filesystem resources.
- Restrict the application's filesystem permissions.
- Test URL fragments, encodings, redirects, and alternate URL representations during security validation.

---

## 6. Application Source Recovery

### Objective

Recover the Flask source and verify the implementation of the administrator route and application runtime configuration.

### Source Location

The supplied assessment identifies the application source as:

```text
/usr/src/app/app.py
```

### Administrator Route

The recovered source contains the following route:

```python
@app.route("/admin")
def admin():
    if request.remote_addr == '127.0.0.1':
        return send_from_directory('private-docs', 'flag.pdf')
```

### Route Analysis

The route checks whether `request.remote_addr` equals `127.0.0.1` before serving `flag.pdf` from the `private-docs` directory.

This is a source-address restriction, not a complete authentication or authorization model. The assessment does not need to claim that the HTTP check was bypassed to establish the risk of the separate local-file disclosure primitive.

The important distinction is that an HTTP route restriction cannot protect a filesystem resource from an independent vulnerability that can read the underlying file directly.

### Application Startup Configuration

The supplied source contains:

```python
app.run(host='0.0.0.0', port=8087, debug=True)
```

This configuration binds the application server to all available interfaces and explicitly enables debugging.

### Findings

- The `/admin` route depends on a localhost address check.
- The protected file is stored on the local filesystem.
- Flask debugging is explicitly enabled.
- The local-file-read primitive exposes source-level information needed to analyze the runtime configuration.

### Security Impact

Source disclosure reveals assumptions about access control, resource locations, and debugging configuration. When combined with an accessible debugger, this information materially increases the potential impact of the original resource-retrieval vulnerability.

---

## 7. Protected Administrator Artifact

### Objective

Determine the location of the protected administrator artifact and explain why route-level restrictions do not mitigate arbitrary local-file access.

### Source-Identified Path

The supplied application source identifies the protected resource as:

```text
/usr/src/app/private-docs/flag.pdf
```

The `/admin` route serves this file through `send_from_directory`.

### Analysis

The localhost check is enforced at the application route. The underlying file remains part of the local filesystem.

If a separate vulnerability allows arbitrary local file reads, the route's source-address condition does not protect the file against that alternate access path.

This distinction illustrates the difference between restricting one interface and restricting access to the resource itself.

### Result

The protected artifact's location is established from the supplied source. Its challenge-specific contents remain redacted.

### Defensive Recommendations

- Store sensitive resources outside directories that do not require access to them.
- Enforce resource-level authorization.
- Apply restrictive filesystem permissions.
- Prevent untrusted input from controlling arbitrary filesystem access.
- Avoid relying on a single route-level condition as the sole protection for sensitive data.

---

## 8. Identifying the Final Challenge Stage

### Objective

Determine how the final challenge artifact can be located when its filename is randomized.

### Observation

The supplied notes describe a remaining text artifact with a randomized filename in the application directory. The filename contains a high-entropy numeric suffix.

### Analysis

Blind filename guessing is inefficient when filenames are randomized and there is no reliable naming convention.

The supplied assessment therefore changes direction: instead of continuing to guess the filename, it focuses on the application's debugger and the filesystem permissions available after authenticated code execution.

### Result

The next stage examines the Werkzeug debugger and the metadata required by its PIN-generation algorithm.

---

## 9. Werkzeug Debug PIN Analysis

### Objective

Determine whether the debugger's PIN can be reconstructed from application and host metadata exposed by the local-file-read primitive.

### Target Version

The supplied assessment identifies Werkzeug version `0.16.0`.

The installed version is important because the correct PIN reconstruction process must match the implementation on the target. The behavior of a different release should not be assumed to be identical.

### PIN Input Categories

The supplied notes identify two categories of values used in the reconstruction.

**Public bits**

```text
Application user
Module name
Application class
Absolute Flask application path
```

**Private bits**

```text
MAC-derived value associated with uuid.getnode()
Container or machine identifier
```

These inputs are considered in the version-specific PIN-generation process described by the assessment.

### Analysis

The PIN's effectiveness depends partly on the confidentiality and unpredictability of the inputs that contribute to its generation.

When a separate vulnerability exposes application and host metadata, those values may become recoverable. A PIN is not an adequate production security boundary for an interactive debugger that can execute server-side code.

### Finding

**Debugger authentication material can be reconstructed from exposed application and host metadata in the documented challenge workflow.**

### Defensive Recommendations

- Never expose the Werkzeug debugger to untrusted users.
- Do not treat its PIN as a replacement for network isolation.
- Remove development debugging from production deployments.
- Restrict access to diagnostic interfaces.
- Investigate exposure and rotate affected credentials or secrets when debugger compromise is suspected.

---

## 10. Recovering Host Metadata

### Objective

Identify the application and host metadata used by the target's version-specific PIN reconstruction process.

### Documented Resources

The supplied assessment identifies the following resources for inspection:

| Resource | Investigative Purpose |
|---|---|
| `/proc/self/environ` | Process environment information |
| `/proc/net/arp` | ARP table and interface information |
| `/sys/class/net/eth0/address` | MAC address of the `eth0` interface |
| `/etc/machine-id` | Potential machine identifier |
| `/proc/self/cgroup` | Process and container-related metadata |

These resources are listed because they appear in the supplied challenge notes; the report does not claim that each file returned identical information or was available in every environment.

### MAC-Derived Value

The documented workflow identifies a network interface's MAC address and converts its hexadecimal representation into the decimal integer associated with the value used by `uuid.getnode()`.

### Container Identifier

The assessment checks the conventional machine-ID location. Where that source is unavailable, it examines the relevant container-related information described in the supplied notes.

### Security Relevance

Host identifiers may not be secrets in isolation. However, values used as inputs to an authentication mechanism can become security-sensitive when combined with a vulnerable file-disclosure path.

This is an example of how an information-disclosure vulnerability can undermine a separate security mechanism.

---

## 11. Matching the Exact Hash Implementation

### Objective

Ensure that the PIN analysis corresponds to the installed Werkzeug implementation.

### Methodology

The supplied assessment inspects the source associated with Werkzeug `0.16.0` before performing the reconstruction.

The notes identify version-specific digest and salt behavior as relevant implementation details.

### Why Version Matching Matters

Library implementations can change across releases. A reconstruction based on a different version may use different inputs, hashing behavior, or formatting rules and may not match the target.

The installed implementation is therefore the appropriate reference for explaining the behavior observed in the challenge.

### Finding

**Version-specific debugger behavior must be verified against the target implementation.**

### Defensive Recommendations

- Maintain an inventory of Flask, Werkzeug, Python, and related dependencies.
- Keep supported dependencies patched.
- Use reproducible dependency files and controlled builds.
- Review relevant security advisories when upgrading.
- Avoid treating a version banner alone as proof of exploitability.

---

## 12. PIN Reconstruction

### Objective

Reconstruct the debugger authentication PIN from the application and host metadata described in the supplied assessment.

### Documented Process

The recovered values are combined using the algorithm associated with the target's Werkzeug version.

```text
Application metadata
          +
Host and container metadata
          |
          v
Version-specific PIN algorithm
          |
          v
Debugger authentication value
```

The challenge-specific PIN is intentionally omitted from this public portfolio document.

### Result

The supplied assessment reports that the reconstruction produces the authentication value used in the subsequent debugger interaction.

### Security Impact

This stage demonstrates that the PIN mechanism does not adequately protect an exposed debugger when its inputs can be recovered through a separate vulnerability.

The primary security failure remains the exposure of interactive debugging functionality to untrusted users, compounded by the underlying local-file disclosure.

---

## 13. Debugger Authentication

### Objective

Validate the reconstructed PIN through the debugger's authentication workflow.

### Methodology

The supplied assessment describes retrieving a session-specific secret from the debugger page and submitting that value together with the reconstructed PIN to the authentication endpoint.

The documented response indicates that the PIN authentication step completes successfully.

### Authentication Versus Authorization State

Successful PIN authentication and authorization of subsequent debugger commands are separate events.

A client must preserve the session state established by the debugger's authentication process. An authentication response alone does not prove that a later command will be recognized as part of a trusted session.

### Result

The assessment proceeds to the session-state analysis needed to explain the behavior of subsequent debugger requests.

---

## 14. Debugger Session State

### Objective

Explain why debugger commands may fail even after the PIN authentication step succeeds.

### Observation

The supplied assessment reports that early command attempts return ordinary application responses rather than the expected debugger output.

The documented explanation is that the later requests do not preserve the debugger's required trust state.

### Identified Request Parameters

The supplied notes identify the following debugger request parameters:

```text
__debugger__=yes
cmd=<command>
frm=<frame id>
s=<secret>
```

The debugger also relies on session state established during successful PIN authentication.

### Analysis

HTTP requests are independent unless the client retains and returns the relevant session information.

If the debugger's trust cookie is not preserved, a later request may not be recognized as part of the authenticated session.

This explains why a successful authentication response alone is insufficient to demonstrate code execution.

### Finding

**Debugger authorization depends on preservation of the authenticated session state.**

### Defensive Relevance

Testing should distinguish between authentication completion, session establishment, command authorization, and verified execution results.

The principal mitigation is to remove interactive debugging from untrusted environments. Session-state protections do not make an exposed code-execution-capable debugger safe for production use.

---

## 15. Establishing Code Execution

### Objective

Validate Python expression evaluation and determine the effective process identity.

### Expression Evaluation

The supplied assessment uses a harmless arithmetic expression to validate evaluation:

```python
1 + 1
```

The documented expected result is:

```text
2
```

### Process Identity

The source material then describes a process-identity check:

```python
__import__('os').popen('id').read()
```

The assessment reports that the resulting process identity indicates root privileges.

### Finding

**Authenticated debugger access enables Python code execution in the application process.**

### Impact Analysis

Remote code execution permits actions within the privileges and environmental constraints of the compromised process.

In this challenge, the application process runs as root. Successful execution therefore establishes a root-level execution context without requiring a separate Linux privilege-escalation vulnerability.

### Why Process Privileges Matter

An application running as an unprivileged service account may still expose sensitive resources and require urgent remediation. However, the possible consequences are typically more constrained than when the same code runs with root privileges.

### Defensive Recommendations

- Disable interactive debugging in production.
- Run the application under a dedicated unprivileged account.
- Restrict filesystem and operating-system permissions.
- Apply container security controls and capability restrictions where applicable.
- Limit access to sensitive local resources.
- Investigate possible compromise if a production debugger is found to be accessible.

---

## 16. Final Directory Enumeration

### Objective

Locate the remaining challenge artifact after the supplied assessment establishes authenticated code execution.

### Target Directory

The documented application directory is:

```text
/usr/src/app
```

### Methodology and Result

The supplied assessment describes enumerating the application directory from the authenticated Python execution context. This reveals the randomized filename that could not be identified efficiently through blind guessing.

The final artifact is then accessed through the established execution context.

The challenge-specific answer remains redacted from this public write-up.

### Analysis

Once code execution has been established, directory enumeration is a more direct way to identify accessible files than guessing a randomized filename.

The randomized filename was not an effective security boundary. The decisive condition was the ability to execute Python code within a process that could enumerate the directory.

---

## End-to-End Attack Chain

The documented assessment progresses through the following stages:

<div class="attack-chain">
  <div class="attack-step">TCP Discovery</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Route Enumeration</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Download Analysis</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Traceback Disclosure</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Local File Disclosure</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Source Recovery</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Debugger Analysis</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">PIN Authentication</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Python RCE</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Root-Level Impact</div>
  <div class="attack-arrow">→</div>
  <div class="attack-step">Artifact Discovery</div>
</div>

### Attack-Chain Analysis

The initial weakness is unsafe server-side resource retrieval. Its local-file disclosure impact exposes the application source and the metadata used during the subsequent debugger investigation.

The Flask debug configuration introduces the next escalation path. The supplied assessment describes recovering the version-specific PIN inputs, authenticating to the debugger, preserving trusted session state, and executing Python expressions.

Finally, the root execution context amplifies the impact and permits enumeration of the application directory.

The chain demonstrates how several distinct weaknesses can reinforce one another. Correcting the download endpoint would disrupt the initial entry point; disabling the debugger and removing unnecessary process privileges would provide additional, independent containment.

---

## Security Findings and Severity

The following ratings are qualitative assessments based on the supplied challenge description. They are not formal CVSS scores or independently validated production-system ratings.

| ID | Finding | Qualitative Severity | Primary Impact |
|---|---|---|---|
| F-01 | Client-controlled server-side destination | High | Allows influence over backend resource retrieval |
| F-02 | Local file disclosure through `file://` handling | Critical | Exposes local files readable by the application |
| F-03 | Detailed debug traceback disclosure | High | Reveals implementation details and source paths |
| F-04 | Exposed Flask/Werkzeug debugger | Critical | Provides a path to interactive code execution |
| F-05 | Reconstructable debugger PIN | Critical | Weakens the debugger's authentication barrier |
| F-06 | Root-privileged application process | Critical | Amplifies code execution into root-level impact |

### Severity Interpretation

- **Critical:** A weakness that directly enables or materially contributes to full application compromise or privileged execution in the documented chain.
- **High:** A weakness that exposes sensitive implementation details or creates a substantial stepping stone toward a more severe compromise.

A production assessment should determine final severity using verified impact, exposure, exploitability, required privileges, affected assets, available compensating controls, and business context.

---

## Root Cause Analysis

### F-01 — Client-Controlled Backend Destination

**Observed weakness:** The application concatenates a client-controlled `server` value with a fixed path and filename.

**Root cause:** Insufficient validation of the destination, scheme, and completed resource reference.

**Security consequence:** The user can influence the server-side retrieval process.

**Recommended control:** Parse and validate destinations using a strict allowlist instead of constructing complete URLs through direct string concatenation.

### F-02 — Local File Disclosure

**Observed weakness:** The backend accepts a local-file scheme, and fragment handling can prevent an appended suffix from changing the selected resource.

**Root cause:** Insufficient scheme restrictions combined with unsafe URL construction and parsing assumptions.

**Security consequence:** Files readable by the application process may be exposed through the download endpoint.

**Recommended control:** Reject local-file schemes, validate destinations and resource references, and apply least-privilege filesystem permissions.

### F-03 — Debug Traceback Disclosure

**Observed weakness:** Malformed input produces detailed tracebacks containing implementation information.

**Root cause:** Development debugging behavior is exposed externally.

**Security consequence:** Internal paths and application logic become available for further attack analysis.

**Recommended control:** Disable production debugging and return generic errors while preserving detailed diagnostics in protected logs.

### F-04 — Exposed Flask/Werkzeug Debugger

**Observed weakness:** The application runs with debugging enabled and exposes the debugger interface.

**Root cause:** Development configuration is used in an environment accessible to untrusted users.

**Security consequence:** An attacker who satisfies the debugger's authentication requirements may obtain interactive code execution.

**Recommended control:** Disable the debugger, restrict diagnostic interfaces, and deploy the application with a production-appropriate WSGI configuration.

### F-05 — Reconstructable Debugger PIN

**Observed weakness:** The supplied assessment describes reconstructing the PIN from recovered application and host metadata.

**Root cause:** The authentication inputs become recoverable after another vulnerability exposes local data and runtime details.

**Security consequence:** The PIN provides an inadequate barrier against an attacker who can reach the debugger and recover the relevant values.

**Recommended control:** Do not rely on the debugger PIN as a production security control. Remove the debugger from untrusted environments and fix the underlying disclosure vulnerability.

### F-06 — Root-Privileged Application Process

**Observed weakness:** The application executes under root privileges.

**Root cause:** Inadequate operating-system privilege separation.

**Security consequence:** Application code execution can have root-level effects within the environment.

**Recommended control:** Run the application using a dedicated unprivileged identity, restrict capabilities, and enforce filesystem and container isolation.

---

## Remediation and Defensive Recommendations

The recommendations below map directly to the vulnerabilities and execution conditions documented in the assessment.

### 1. Secure the Download Endpoint

The download endpoint is the initial entry point into the attack chain. Its input-handling behavior should be corrected first.

Recommended controls:

- Avoid accepting complete user-supplied URLs when a resource identifier would suffice.
- Restrict permitted URL schemes to those needed by the application.
- Use a strict allowlist of destination hosts.
- Validate destination addresses and resolved IP addresses.
- Prevent access to loopback, private, link-local, and other prohibited destinations where appropriate.
- Revalidate destination policy after redirects.
- Use established URL-parsing libraries rather than custom string concatenation.
- Reject malformed, ambiguous, or unexpected URL structures.
- Apply connection timeouts and response-size limits.
- Restrict outbound network access at the infrastructure layer.

If the feature only needs to retrieve a fixed set of documents, map client-supplied identifiers to server-controlled resources rather than accepting arbitrary destinations.

### 2. Prevent Local File Disclosure

- Reject the `file://` scheme when local filesystem access is not an explicit requirement.
- Separate public resources from sensitive application files.
- Validate filesystem paths against an explicit allowlist.
- Do not treat URL fragments as a security boundary.
- Apply consistent parsing and normalization before authorization checks.
- Restrict filesystem permissions for the application process.
- Add tests covering unexpected URL schemes, path manipulation, fragment handling, and encoded input.

The safest design is to prevent untrusted input from determining arbitrary filesystem paths in the first place.

### 3. Disable Flask Debug Mode in Production

An interactive debugger should not be exposed to untrusted users.

Recommended controls:

- Disable debugging in production configuration.
- Do not use the Flask development server as a production server.
- Deploy through an appropriately configured production WSGI server.
- Remove or restrict diagnostic endpoints.
- Return generic error messages to external clients.
- Store detailed tracebacks in access-controlled logs.
- Verify the deployed configuration rather than relying solely on development defaults.

Disabling the debugger removes a major escalation path but does not eliminate the need to remediate the initial local-file-disclosure weakness.

### 4. Harden the Application Stack

- Track Flask, Werkzeug, Python, and related dependencies.
- Maintain supported and patched versions.
- Use reproducible dependency files and controlled deployment builds.
- Review relevant framework and library security advisories.
- Avoid relying on undocumented implementation details for security guarantees.
- Revalidate configuration after dependency upgrades.

A version banner helps direct an investigation, but it does not by itself establish the presence or exploitability of a vulnerability.

### 5. Prevent Sensitive Information in Error Responses

- Disable interactive traceback pages in production.
- Prevent external error responses from exposing source paths and configuration details.
- Return consistent generic errors.
- Preserve diagnostic information in protected server logs.
- Review logs for accidental disclosure of secrets.
- Rotate credentials and tokens if exposure is confirmed.

Errors should support incident investigation without supplying attackers with a detailed map of the application's internals.

### 6. Protect Diagnostic Interfaces

The preferred control is to remove interactive debugging from production environments.

Where diagnostic interfaces are required in controlled development environments:

- Isolate them from public networks.
- Restrict access to authorized operators.
- Use appropriate access controls and network segmentation.
- Prevent unrelated application vulnerabilities from exposing diagnostic inputs.
- Monitor access to debugging endpoints.
- Treat unexpected debugger access as a potential security incident.

A PIN should not be treated as sufficient protection for an interface capable of executing server-side code.

### 7. Enforce Least Privilege

The root execution context substantially increases the impact of the debugger compromise.

Recommended controls:

- Run the web application under a dedicated, unprivileged service account.
- Restrict access to sensitive directories.
- Remove unnecessary Linux capabilities.
- Apply container security profiles where containers are used.
- Use read-only filesystem mounts where practical.
- Restrict access to host resources.
- Limit network egress to required destinations.
- Separate application processes from privileged administrative operations.

Least privilege will not prevent every application vulnerability, but it can substantially reduce the consequences of exploitation.

### 8. Implement Defense in Depth

A resilient deployment should combine multiple independent controls:

- Strict input validation.
- Safe URL and filesystem-path handling.
- Network egress restrictions.
- Protected application source and configuration.
- Secure error handling.
- Disabled production debugging.
- Least-privilege execution.
- Dependency management.
- Centralized logging and monitoring.
- Security testing of complete attack paths.

The objective is both to prevent the initial exploit and to ensure that an individual weakness does not automatically result in privileged compromise.

---

## Lessons Learned

### 1. A Download Feature Can Become an SSRF Entry Point

Any application functionality that retrieves resources on behalf of a user should be evaluated for control over the destination, URL scheme, redirects, and final resource selection.

An ordinary download workflow may expose internal services or local files when these boundaries are not enforced.

### 2. URL Parsing Must Be Consistent

The local-file-disclosure stage demonstrates the importance of understanding how untrusted input interacts with application-controlled path suffixes.

Security checks must operate on the parsed and validated resource reference, not merely on the raw input string.

### 3. Debug Tracebacks Can Reveal Critical Implementation Details

The exposed traceback revealed how the application assembled its backend resource reference.

Although detailed exceptions are useful during authorized development and testing, production systems should not disclose internal source paths or implementation details to external users.

### 4. Source Disclosure Changes the Nature of an Assessment

Once the application source is available, behavioral inference can be supplemented with direct analysis of actual implementation logic.

Source disclosure may reveal route conditions, file paths, configuration settings, and trust assumptions that would otherwise require more extensive investigation.

### 5. Debugger PINs Are Not a Production Security Boundary

The PIN-analysis stage demonstrates how authentication inputs can become recoverable when the application and host environment are exposed.

The correct defensive response is to disable the debugger in production, rather than rely on the obscurity of its authentication mechanism.

### 6. Authentication and Session State Must Be Distinguished

Successful authentication does not automatically prove that a subsequent command is authorized or executed.

Preserving the relevant session state is necessary to establish continuity across requests. Testing should verify the actual result rather than infer success from a single response.

### 7. Process Privileges Determine RCE Impact

The application ran as root, so successful code execution immediately provided a privileged execution context.

Running the same application as an unprivileged account would not eliminate the RCE vulnerability, but it would reduce the potential consequences.

### 8. Select Enumeration Methods Based on Established Capabilities

The randomized filename made blind guessing inefficient.

Once execution within the application context was available, directory enumeration provided a direct way to identify accessible files. The next assessment step should be chosen according to capabilities actually established by evidence.

---

## Evidence and Publication Policy

This report is based on the supplied Plant Photographer challenge documentation.

Evidence should demonstrate the relevant behavior while avoiding unnecessary disclosure of sensitive values.

### Evidence Standards

- Use screenshots captured from the actual authorized lab.
- Ensure each screenshot supports the finding associated with it.
- Redact challenge flags, debugger PINs, credentials, and other answer values.
- Preserve existing repository image paths.
- Do not present conceptual diagrams as screenshots of target activity.
- Do not claim independent verification beyond the evidence available in the supplied material.

### Evidence Checklist

The following list is a suggested evidence inventory, not a claim that every item was captured as a separate screenshot.

- [ ] RustScan or Nmap service enumeration.
- [ ] Application route discovery.
- [ ] Download endpoint behavior.
- [ ] Debug traceback disclosure.
- [ ] Local file disclosure evidence.
- [ ] Flask source showing the route and debug configuration.
- [ ] Werkzeug version or implementation evidence.
- [ ] Debugger authentication result with sensitive values redacted.
- [ ] Harmless expression evaluation.
- [ ] Process identity output demonstrating the execution context.
- [ ] Final challenge completion evidence, if available.

Only add screenshots that actually exist in the repository or that you have captured from the authorized lab.

### Public Flag Policy

Challenge-specific answers are intentionally redacted.

```text
Administrator artifact  → [REDACTED]
Final challenge answer   → [REDACTED]
Debugger PIN             → [REDACTED]
```

This policy keeps the portfolio focused on technical methodology, vulnerability analysis, and defensive lessons without publishing challenge answer values.

---

## Conclusion

Plant Photographer demonstrates how unsafe server-side resource retrieval can provide an entry point for a larger application compromise.

The assessment begins with service discovery and route enumeration. It then investigates a download endpoint that accepts a client-controlled destination, exposes detailed debug tracebacks, and permits local-file access through its backend resource-handling behavior.

The recovered Flask source confirms the debugging configuration and identifies a protected local artifact. The supplied assessment then uses recovered application and host metadata to analyze the target's Werkzeug PIN mechanism. After debugger authentication and session-state preservation, the assessment validates Python code execution within an application process reported to be running as root.

The central lesson is that **input validation, resource access, error disclosure, debugging configuration, and operating-system privileges must be secured as a connected system**. No single layer should be expected to compensate for weaknesses across all the others.

The assessment reinforces the following methodology:

**Enumerate → Inspect → Validate → Disclose → Correlate → Authenticate → Execute → Remediate**

For defenders, the highest-value actions are to eliminate arbitrary resource retrieval, prevent local-file disclosure, disable production debugging, restrict diagnostic interfaces, and run application processes with minimal privileges.

These controls address both the original vulnerability and the conditions that allow it to escalate into root-level execution.

---

## References and Tools

### Official Documentation and Security Resources

- [TryHackMe](https://tryhackme.com/) — Authorized cybersecurity learning environment.
- [Flask Documentation](https://flask.palletsprojects.com/) — Application configuration, deployment, and debugging guidance.
- [Werkzeug Documentation](https://werkzeug.palletsprojects.com/) — WSGI utilities and debugger behavior.
- [Python Documentation](https://docs.python.org/3/) — Python runtime and standard-library reference.
- [Nmap](https://nmap.org/) — Network discovery and service enumeration.
- [RustScan](https://github.com/RustScan/RustScan) — TCP port discovery.
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) — Web application testing methodology.
- [OWASP Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) — Defensive SSRF controls.
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal) — Background on unsafe filesystem resource access.

### Technical Topics Demonstrated

- TCP service enumeration.
- Web application reconnaissance.
- Server-side request forgery analysis.
- URL scheme validation.
- URL fragment handling.
- Local file disclosure.
- Python source-code analysis.
- Flask debug configuration.
- Werkzeug PIN reconstruction.
- HTTP session-state analysis.
- Python remote code execution.
- Linux process privilege assessment.
- Application-directory enumeration.
- Root-cause analysis and remediation planning.

---

## Responsible Use

> This documentation was prepared for authorized cybersecurity training and CTF environments. The techniques described here should only be used against systems for which explicit permission has been granted. Do not use these methods to access, modify, or extract data from unauthorized systems.

