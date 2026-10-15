---
title: "Plant Photographer — TryHackMe Security Assessment"
description: "Professional security assessment documenting SSRF, local file disclosure, Flask debug exposure, Werkzeug PIN reconstruction, Python remote code execution, and root-level compromise."
platform: "TryHackMe"
room: "Plant Photographer"
difficulty: "Hard"
author: "Anurag Revankar"
---

# Plant Photographer — Security Assessment & CTF Walkthrough

> **Platform:** TryHackMe  
> **Room:** Plant Photographer  
> **Difficulty:** Hard  
> **Assessment Type:** Authorized Capture the Flag (CTF)  
> **Primary Domains:** Web Application Security, SSRF, Local File Disclosure, Python, Flask, Werkzeug, Remote Code Execution, Linux Security  
> **Author:** Anurag Revankar

---

## Executive Summary

Plant Photographer is a multi-stage web security challenge demonstrating how an inadequately validated server-side download mechanism can escalate into arbitrary local file disclosure and ultimately privileged remote code execution.

The investigation begins with network reconnaissance, which identifies an HTTP service running Werkzeug. Subsequent application enumeration reveals a download endpoint that accepts a client-controlled destination. Analysis of the endpoint's behavior and an exposed debug traceback reveals unsafe URL construction and insufficient restrictions on the backend retrieval mechanism.

The application accepts the `file://` scheme. By exploiting the way the application appends a path to the supplied destination, the assessment demonstrates how a URL fragment can prevent the appended suffix from changing the selected local resource. This converts a server-side request capability into a local file disclosure primitive.

The disclosed application source confirms that Flask is running with debug mode enabled. The exposed Werkzeug debugger becomes the next attack surface. By recovering the relevant application and host metadata through the local-file-read primitive, identifying the exact Werkzeug version, and reconstructing its PIN algorithm, the assessment establishes authenticated access to the interactive debugger.

Preserving the debugger's session state then enables Python code execution. Because the application process runs with root privileges, the resulting execution occurs in a privileged context. This makes further application-directory enumeration possible and reveals the location of the final challenge artifact.

### Key Findings

- A server-controlled download mechanism accepts attacker-influenced destinations.
- The backend supports local-file access through the `file://` scheme.
- URL construction and fragment handling enable arbitrary local file disclosure.
- Detailed debug tracebacks disclose application implementation details.
- The Flask application runs with debugging enabled.
- Werkzeug debugger authentication can be reconstructed using exposed host metadata and version-specific implementation details.
- Authenticated debugger access enables Python code execution.
- The application process runs as root, amplifying the impact of remote code execution.

### Attack Chain Overview

```text
Network Reconnaissance
        |
        v
Web Application Enumeration
        |
        v
Download Endpoint Analysis
        |
        v
Debug Traceback Disclosure
        |
        v
SSRF / Local File Disclosure
        |
        v
Flask Source Recovery
        |
        v
Debug Mode Confirmation
        |
        v
Host Metadata Recovery
        |
        v
Werkzeug PIN Reconstruction
        |
        v
Debugger Authentication
        |
        v
Session-State Preservation
        |
        v
Python Remote Code Execution
        |
        v
Root-Level Execution
        |
        v
Final Artifact Discovery
```

> **Public portfolio policy:** Challenge flags, recovered debugger PINs, and other challenge-specific answer values are intentionally redacted. The document focuses on the technical methodology, vulnerability chain, impact, and defensive recommendations.

---

## Navigation

**Documentation Map**

<div class="ctf-toc">
<div class="ctf-toc-title">Documentation Map</div>
<ul>
<li><a href="#mission-and-assessment-overview">Mission and Assessment Overview</a></li>
<li><a href="#scope-and-responsible-use">Scope and Responsible Use</a></li>
<li><a href="#assessment-objectives">Assessment Objectives</a></li>
<li><a href="#tools-and-techniques">Tools and Techniques</a></li>
<li><a href="#1-reconnaissance">1. Reconnaissance</a></li>
<li><a href="#2-web-application-enumeration">2. Web Application Enumeration</a></li>
<li><a href="#3-validating-the-download-primitive">3. Validating the Download Primitive</a></li>
<li><a href="#4-debug-traceback-and-information-disclosure">4. Debug Traceback and Information Disclosure</a></li>
<li><a href="#5-converting-ssrf-into-local-file-disclosure">5. Converting SSRF into Local File Disclosure</a></li>
<li><a href="#6-application-source-recovery">6. Application Source Recovery</a></li>
<li><a href="#7-protected-administrator-artifact">7. Protected Administrator Artifact</a></li>
<li><a href="#8-identifying-the-final-stage">8. Identifying the Final Stage</a></li>
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

The objective of Plant Photographer is to investigate the security of a Python-backed web application and determine how weaknesses in server-side resource retrieval, error handling, debugger configuration, and operating-system privilege separation can be chained into a full application compromise.

The assessment follows an evidence-driven methodology. Each stage builds on observations collected during the preceding stage, reducing reliance on assumptions and avoiding unnecessary testing.

The principal stages are:

1. Identify exposed network services.
2. Enumerate application routes and user-controlled parameters.
3. Investigate server-side retrieval behavior.
4. Analyze error responses for implementation details.
5. Establish the scope of local file disclosure.
6. Recover the application source and confirm its runtime configuration.
7. Identify the exposed debugger and its authentication requirements.
8. Reconstruct the version-specific debugger PIN inputs.
9. Preserve authenticated debugger session state.
10. Validate code execution and determine the effective process privileges.
11. Enumerate the application directory to locate the remaining challenge artifact.

### Assessment Context

| Attribute | Description |
|---|---|
| Platform | TryHackMe |
| Room | Plant Photographer |
| Difficulty | Hard |
| Target type | Intentionally vulnerable lab environment |
| Web server | Werkzeug |
| Application framework | Flask |
| Programming language | Python |
| Operating system | Linux |
| Primary vulnerability chain | SSRF-style retrieval → local file disclosure → debugger authentication → RCE |
| Final execution context | Root, as observed in the supplied challenge material |
| Documentation purpose | Technical portfolio and security assessment record |

---

## Scope and Responsible Use

All testing described in this document is restricted to the authorized TryHackMe environment.

The techniques are documented for educational purposes, authorized penetration testing, and defensive security analysis.

### In Scope

- TCP service discovery and service fingerprinting.
- HTTP endpoint enumeration.
- Analysis of the download endpoint's input handling.
- Examination of debug tracebacks and application source.
- Local file disclosure testing within the assigned lab.
- Analysis of Flask and Werkzeug configuration.
- Debugger PIN reconstruction using challenge-provided evidence.
- Authenticated debugger interaction.
- Verification of process identity and privilege level.
- Enumeration of challenge artifacts.

### Out of Scope

- Testing unrelated public-facing systems.
- Accessing production applications without authorization.
- Using the discovered technique against third-party servers.
- Extracting real users' files, credentials, or sensitive data.
- Publishing challenge secrets or recovered authentication values.
- Attempting persistence or unnecessary actions outside the challenge objectives.

**Responsible testing principle:** Use the minimum access necessary to demonstrate impact. Protect sensitive evidence, document the authorized scope, and avoid actions that could affect systems outside the assigned lab.

---

## Assessment Objectives

The assessment aims to answer the following questions:

- Which services are exposed by the target?
- Does the web application expose functionality that performs server-side resource retrieval?
- Can a client-controlled destination influence the backend's resource selection?
- Does the application permit unsupported URL schemes?
- Can malformed input expose detailed exception traces or source code?
- Can the retrieval mechanism access arbitrary local files?
- Is the Flask debugger enabled and reachable?
- Which application and host attributes contribute to Werkzeug PIN generation?
- Does successful PIN authentication establish a persistent trusted session?
- What privileges does the application process possess?
- How can the discovered weaknesses be mitigated at both the application and infrastructure layers?

---

## Tools and Techniques

| Tool / Technique | Purpose |
|---|---|
| RustScan | Rapid TCP port discovery |
| Nmap | Service fingerprinting and version enumeration |
| Web browser | Application navigation and route discovery |
| HTTP request inspection | Analysis of download parameters and responses |
| Python source review | Identification of unsafe URL construction and debug configuration |
| Local file disclosure analysis | Retrieval of application and host metadata |
| Werkzeug source inspection | Version-specific debugger PIN algorithm analysis |
| Python | PIN reconstruction and debugger interaction |
| Linux `/proc` inspection | Recovery of process and container metadata |
| Linux filesystem inspection | Application-directory enumeration |
| Debugger expression evaluation | Validation of code execution and process identity |

The investigation prioritizes evidence-driven testing: each tool is used to answer a specific question raised by the preceding stage.

---

# Technical Walkthrough

## 1. Reconnaissance

### Objective

Identify the exposed TCP services and fingerprint the web server to establish the initial attack surface.

### Methodology

RustScan is used to discover open TCP ports. Nmap is then used to identify the services and collect additional version information.

The supplied assessment identifies the following services:

```text
22/tcp  open  ssh
80/tcp  open  http
        Werkzeug httpd 0.16.0
        Python 3.10.7
```

### Observations

| Port | Service | Observation |
|---|---|---|
| 22/TCP | SSH | Remote administration service |
| 80/TCP | HTTP | Web application |
| 80/TCP | Werkzeug | Python-based development server fingerprint |
| Application runtime | Python 3.10.7 | Runtime version reported by service fingerprinting |

### Analysis

The Werkzeug banner is a useful investigative clue because Werkzeug is commonly associated with Flask applications.

However, identifying Werkzeug does not independently prove that the interactive debugger is enabled or externally accessible. These conditions must be validated through subsequent application testing.

The initial reconnaissance therefore establishes two priorities:

- Investigate the exposed HTTP application.
- Determine whether development-oriented debugging functionality is present.

### Result

The HTTP service becomes the primary investigation target, while SSH remains an identified service in the attack-surface inventory.

---

## 2. Web Application Enumeration

### Objective

Map the available application routes and identify parameters that influence backend behavior.

### Methodology

The web application is explored through its main interface and exposed endpoints.

The supplied assessment identifies the following routes:

```text
/admin
/download
/console
```

### Route Analysis

| Endpoint | Observed Behavior | Investigative Relevance |
|---|---|---|
| `/admin` | Returns a localhost-only message when accessed remotely | Indicates an address-based access restriction |
| `/download` | Accepts a `server` parameter and an identifier | Potential server-side retrieval primitive |
| `/console` | Associated with the Flask/Werkzeug debugger | Potentially critical if debugging functionality is exposed |

### `/admin` Observation

A remote request to `/admin` produces a response indicating that the endpoint is restricted to localhost.

This suggests that the application checks the request's source address before serving the protected resource.

At this stage, the restriction is an observation rather than a demonstrated bypass.

### `/download` Observation

The download endpoint accepts a `server` parameter in addition to an identifier.

A parameter that influences a backend destination deserves additional scrutiny because it may allow user input to influence a server-side network request or resource lookup.

### Result

The download endpoint is selected for further analysis because its input structure suggests that client-controlled data may influence server-side resource retrieval.

---

## 3. Validating the Download Primitive

### Objective

Determine whether the download endpoint performs server-side retrieval and understand how it handles invalid destinations.

### Methodology

A legitimate download request is used to establish the endpoint's expected behavior.

The next step involves examining how the application responds when supplied with an invalid destination, including a destination containing an invalid port.

The supplied evidence indicates that this malformed request causes the backend to raise an exception.

Because the application is running with Werkzeug debugging enabled, the exception produces a detailed traceback instead of a generic error response.

### Observation

The error response exposes implementation details that would normally remain hidden from an unauthenticated user.

This provides evidence that the application performs server-side processing of the supplied destination.

### Security Relevance

Detailed exceptions can expose:

- Application source locations.
- Framework and library behavior.
- Backend request-processing logic.
- Internal file paths.
- Potentially sensitive implementation details.

Error handling is therefore part of the application's attack surface, not merely a diagnostic feature.

### Result

The malformed request establishes a path from a user-controlled download parameter to observable backend behavior and source-level information disclosure.

---

## 4. Debug Traceback and Information Disclosure

### Objective

Analyze the exposed traceback to determine how the application constructs the backend destination.

### Evidence

The supplied traceback reveals a route pattern equivalent to:

```python
crl.URL = server + '/public-docs-k057230990384293/' + filename
```

### Analysis

The expression reveals two important characteristics:

1. The client influences the base `server` value.
2. The application appends a fixed path and a filename without first establishing that the destination and resource are safe.

The traceback also discloses the application's source path and other implementation details.

### Finding

**Unsafe server-side URL construction combined with debug information disclosure.**

The application constructs a backend resource location using client-controlled input. The absence of demonstrated scheme and destination validation creates an opportunity to investigate server-side request forgery and local resource access.

### Security Impact

An attacker may be able to influence the backend's resource selection and potentially reach resources that are not intended to be accessible through the public interface.

The actual impact depends on the accepted URL schemes, the underlying retrieval library, filesystem permissions, and the application’s URL-handling behavior.

### Defensive Recommendation

- Avoid concatenating untrusted strings into complete URLs.
- Parse and validate destinations using established URL-handling libraries.
- Restrict the accepted URL schemes.
- Validate destination hosts and resolved addresses.
- Return generic error messages to external clients.
- Record detailed diagnostic information only in access-controlled server logs.
- Disable interactive debug tracebacks in production.

---

## 5. Converting SSRF into Local File Disclosure

### Objective

Determine whether the backend retrieval mechanism can access local files through an unsupported or insufficiently restricted URL scheme.

### Methodology

The supplied assessment establishes that the backend client accepts the `file://` scheme.

A direct request for `/etc/passwd` initially fails because the application appends its own fixed path and filename.

The resulting resource path resembles:

```text
file:///etc/passwd/public-docs-k057230990384293/1.pdf
```

This failure provides useful evidence: the backend is processing the local-file scheme, but the appended suffix changes the resource path.

### URL Fragment Analysis

The next stage examines URL-fragment behavior.

A fragment begins with `#`. In standard URL handling, the fragment identifies a client-side portion of a resource reference and is not sent as part of the HTTP request target.

In the challenge's local-file retrieval context, fragment handling can prevent an appended path suffix from affecting the selected local resource.

The supplied assessment uses a URL of this form:

```text
file:///etc/passwd#
```

When the fragment delimiter must pass through an outer query-string parser, it is URL-encoded as `%23`.

Conceptually, the application constructs:

```text
attacker-controlled destination
          +
fragment delimiter
          +
application-controlled suffix
          |
          v
resource selection occurs before the fragment
          |
          v
appended suffix no longer changes the selected resource
```

This behavior is dependent on how the application parses and reconstructs the URL and how the backend retrieval library interprets it.

### Finding

**Arbitrary local file disclosure through unsafe server-side resource retrieval.**

### Security Impact

The file-read primitive provides access to local resources readable by the application process, subject to filesystem permissions.

In the supplied assessment, this capability exposes application source and host metadata required for the subsequent debugger analysis.

### Defensive Recommendation

- Reject the `file://` scheme when only remote HTTP or HTTPS retrieval is required.
- Use a strict allowlist of permitted schemes and destinations.
- Avoid passing complete client-controlled URLs directly to backend retrieval libraries.
- Resolve and validate destinations before use.
- Prevent access to sensitive local files through the download mechanism.
- Run the application with minimal filesystem privileges.
- Test URL parsing and normalization behavior across the complete request-processing chain.

---

## 6. Application Source Recovery

### Objective

Recover the Flask application source and confirm the access-control logic and runtime configuration.

### Methodology

The established local-file-read primitive is used to retrieve the application source.

The source file is located at:

```text
/usr/src/app/app.py
```

The supplied source contains the following administrator route:

```python
@app.route("/admin")
def admin():
    if request.remote_addr == '127.0.0.1':
        return send_from_directory('private-docs', 'flag.pdf')
```

The application startup configuration also contains:

```python
app.run(host='0.0.0.0', port=8087, debug=True)
```

### Analysis: Administrator Access Control

The `/admin` route checks whether `request.remote_addr` equals `127.0.0.1`.

This is an IP-based access restriction. It does not establish a complete authentication or authorization mechanism.

In the supplied challenge, direct local file disclosure means that the protected artifact can be investigated without relying on a bypass of the HTTP route itself.

This distinction matters: the assessment does not need to claim that the localhost check was bypassed when the underlying file can already be read through a separate vulnerability.

### Analysis: Debug Configuration

The application explicitly enables Flask debugging.

The debugger is intended to assist development by providing detailed exception information and interactive debugging functionality. Exposing it to untrusted users creates a serious security risk.

### Findings

- The administrator endpoint relies on a source-address restriction.
- The protected file resides on the local filesystem.
- Flask is configured with `debug=True`.
- The local-file-read primitive can expose information needed to investigate the debugger.

### Security Impact

The source disclosure reveals both the application's trust assumptions and its debugging configuration.

The combination of arbitrary local file access and an exposed interactive debugger substantially increases the potential impact of the initial download vulnerability.

---

## 7. Protected Administrator Artifact

### Objective

Determine the location of the protected administrator artifact and validate the impact of the local-file-read primitive.

### Evidence

The application source identifies the protected resource as:

```text
/usr/src/app/private-docs/flag.pdf
```

The `/admin` route serves the corresponding file through `send_from_directory`.

### Analysis

The access restriction on `/admin` is enforced at the HTTP route level, but the file itself remains accessible to the application process.

When a separate vulnerability allows arbitrary local file reads, the route's restriction does not protect the underlying file from that independent access path.

This is an example of why authorization controls should be considered alongside the security of the underlying resource-access mechanisms.

### Result

The protected artifact's location is established from application source. Its challenge-specific contents are intentionally excluded from this portfolio document.

### Defensive Recommendation

- Store sensitive files outside generally accessible application directories where appropriate.
- Enforce access controls at the resource-access layer.
- Restrict filesystem permissions.
- Prevent untrusted input from influencing arbitrary filesystem reads.
- Avoid relying on a single application-route condition to protect sensitive resources.

---

## 8. Identifying the Final Stage

### Objective

Determine how to locate the remaining challenge artifact when its filename is not predictable.

### Observation

The supplied assessment indicates that the remaining flag is stored in a text file with a randomized filename in the application directory.

The filename contains a high-entropy numeric suffix.

### Analysis

Blind filename guessing is inefficient when the search space is large and the filename does not follow a predictable naming convention.

The investigation therefore changes direction: instead of continuing to guess the filename, it focuses on establishing authenticated code execution and using the application's existing filesystem permissions to enumerate the directory.

### Result

The next stage focuses on the exposed Werkzeug debugger and the information required to authenticate to it.

---

## 9. Werkzeug Debug PIN Analysis

### Objective

Determine whether the exposed Werkzeug debugger's PIN can be reconstructed from information available through the local-file-read primitive.

### Methodology

The `/console` endpoint is identified as the Flask/Werkzeug interactive debugger.

The debugger is protected by a PIN-based authentication mechanism. The supplied assessment identifies Werkzeug version `0.16.0`.

The relevant algorithm must therefore be matched to the version installed on the target rather than assumed to match a modern Werkzeug release.

### PIN Input Categories

The assessment identifies two broad categories of inputs.

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

These values contribute to the debugger's PIN-generation process.

### Security Analysis

The security of the PIN mechanism depends partly on the confidentiality and unpredictability of its inputs.

If an attacker can obtain the relevant application and host metadata through another vulnerability, the PIN may no longer provide a meaningful barrier against debugger access.

### Finding

**Debugger authentication material can be reconstructed using exposed application and host metadata.**

### Defensive Recommendation

- Never expose the Werkzeug interactive debugger to untrusted users.
- Do not treat the debugger PIN as a substitute for network isolation.
- Avoid deploying development servers in production.
- Restrict access to diagnostic interfaces.
- Rotate credentials and investigate exposure following debugger compromise.

---

## 10. Recovering Host Metadata

### Objective

Identify the process and container metadata required by the version-specific PIN reconstruction process.

### Methodology

The local-file-read primitive is used to inspect selected process and network metadata files.

The supplied assessment identifies the following resources.

| Resource | Investigative Purpose |
|---|---|
| `/proc/self/environ` | Process environment information |
| `/proc/net/arp` | ARP table and interface information |
| `/sys/class/net/eth0/address` | MAC address of the `eth0` interface |
| `/etc/machine-id` | Potential machine identifier |
| `/proc/self/cgroup` | Process and container-related metadata |

### MAC-Derived Value

The supplied procedure identifies the network interface's MAC address and converts its hexadecimal representation into the decimal integer corresponding to the value used by `uuid.getnode()`.

### Container Identifier

The assessment also checks the conventional machine-ID location.

When that source is unavailable in the container, the investigation uses relevant information from `/proc/self/cgroup`, as described in the supplied material.

### Important Implementation Detail

The effective inputs depend on the target's runtime environment and the exact implementation of the PIN-generation algorithm.

A reconstruction must therefore use evidence from the target rather than assume that every container exposes identical metadata.

### Security Relevance

Information that is not individually a secret can become security-sensitive when it contributes to an authentication mechanism.

The combination of local file disclosure and predictable or discoverable PIN inputs is more significant than either condition considered in isolation.

---

## 11. Matching the Exact Hash Implementation

### Objective

Ensure that the PIN reconstruction follows the target's installed Werkzeug implementation.

### Methodology

The supplied assessment inspects the source corresponding to Werkzeug `0.16.0` before generating the PIN.

The relevant implementation uses a version-specific hashing and formatting process, including legacy digest and salt behavior.

### Why Version Matching Matters

Debugger implementations can change between library releases.

A reconstruction based on a newer version may use different inputs, hashing behavior, or formatting rules and therefore produce an incorrect result.

The installed target version is the authoritative source for understanding the behavior observed in the challenge.

### Finding

**Version-specific debugger behavior must be validated against the target implementation.**

### Defensive Recommendation

- Keep framework and library versions supported and patched.
- Remove development debugging facilities from production deployments.
- Use reproducible dependency management.
- Track framework versions as part of the application's software inventory.
- Treat version fingerprinting as an indicator for further verification, not as proof of a vulnerability by itself.

---

## 12. PIN Reconstruction

### Objective

Reconstruct the debugger PIN from the recovered application and host metadata.

### Methodology

The supplied assessment feeds the recovered values into a local reconstruction script that matches the target's Werkzeug implementation.

The conceptual sequence is:

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

The resulting PIN is intentionally omitted from this public write-up.

### Result

The reconstruction provides the value required for the next authentication stage.

### Security Impact

The result demonstrates that the PIN mechanism does not adequately protect an exposed debugger when its inputs can be recovered through a separate vulnerability.

The underlying security failure is the availability of the debugger to untrusted users, compounded by information disclosure that enables reconstruction of its authentication value.

---

## 13. Debugger Authentication

### Objective

Validate the reconstructed PIN against the debugger's authentication workflow.

### Methodology

The debugger exposes a session-specific secret through its console page.

The supplied assessment describes submitting the secret and reconstructed PIN to the debugger's authentication endpoint.

A successful response indicates that the PIN authentication step has completed.

### Important Distinction

Successful PIN authentication does not imply that every subsequent HTTP request is automatically trusted.

The debugger uses session state to determine whether later command requests are authorized.

### Security Relevance

Authentication state may be represented through cookies and other session-specific values. Clients that do not preserve these values can fail to maintain an authenticated debugger session.

The supplied assessment identifies this behavior as a significant part of the exploitation process.

### Result

The debugger authentication step is completed, but maintaining the appropriate session state remains necessary for subsequent interactions.

---

## 14. Debugger Session State

### Objective

Understand why debugger command requests initially fail despite successful PIN authentication.

### Observation

The supplied assessment reports that early command attempts return ordinary application responses rather than the expected debugger output.

The investigation attributes this behavior to missing debugger trust state between independent HTTP requests.

### Request Parameters

The supplied material identifies the following debugger request parameters:

```text
__debugger__=yes
cmd=<command>
frm=<frame id>
s=<secret>
```

The server also expects the trust state established during successful PIN authentication.

### Analysis

HTTP requests are independent unless the client maintains the relevant session state.

If a client does not preserve and return the debugger's trust cookie, a subsequent request may not be recognized as part of the authenticated debugging session.

This explains why a successful authentication response alone is insufficient to demonstrate command execution.

### Finding

**Debugger authorization depends on preserved session state.**

### Security Relevance

This stage illustrates the importance of distinguishing between:

- Successfully submitting authentication material.
- Establishing trusted session state.
- Sending an authorized command request.
- Verifying the actual execution result.

### Defensive Recommendation

The primary mitigation is to remove the debugger from untrusted environments. Session-state protections should not be treated as an adequate defense for an interactive console that can execute server-side code.

---

## 15. Establishing Code Execution

### Objective

Validate that the authenticated debugger can evaluate Python expressions and determine the resulting execution context.

### Methodology

The supplied assessment first validates expression evaluation with a harmless arithmetic expression:

```python
1 + 1
```

The expected result is:

```text
2
```

A subsequent process-identity check uses:

```python
__import__('os').popen('id').read()
```

The supplied assessment reports that the process identity indicates root privileges.

### Finding

**Authenticated debugger access enables Python code execution in the application process.**

### Impact Analysis

Remote code execution is a critical security condition because it permits actions within the privileges and environmental constraints of the compromised process.

In this challenge, the application process already runs as root. As a result, successful debugger execution provides root-level execution without requiring a separate local privilege-escalation vulnerability.

### Why Process Privileges Matter

The same remote-code-execution vulnerability can have substantially different consequences depending on the identity under which the application runs.

A process operating as an unprivileged account may still expose sensitive application data, but a process running as root can have much broader filesystem and system access.

### Defensive Recommendation

- Never expose an interactive debugger to untrusted users.
- Run applications under dedicated, unprivileged service accounts.
- Restrict filesystem and operating-system permissions.
- Apply container security controls and capability restrictions.
- Limit network egress and access to sensitive local resources.
- Investigate potential compromise if a production debugger is found to be accessible.

---

## 16. Final Directory Enumeration

### Objective

Locate the final challenge artifact after establishing authenticated code execution.

### Methodology

The supplied assessment uses the authenticated Python execution context to enumerate the application directory:

```text
/usr/src/app
```

The directory listing reveals the randomized filename that could not be identified efficiently through blind guessing.

The final artifact is then accessed through the established debugger execution context.

### Analysis

This stage demonstrates a practical advantage of code execution over filename guessing: authenticated execution allows direct inspection of the filesystem resources accessible to the process.

The randomized filename was not the primary security boundary. The decisive issue was the ability to execute code within a process that could enumerate the directory.

### Result

The final challenge artifact is located. Its answer is intentionally redacted from this public portfolio document.

---

# Security Assessment Results

## End-to-End Attack Chain

The complete progression can be summarized as follows:

```text
[Network Reconnaissance]
          |
          v
[Werkzeug Fingerprinting]
          |
          v
[Application Route Enumeration]
          |
          v
[Download Endpoint Analysis]
          |
          v
[Malformed Request / Debug Traceback]
          |
          v
[Unsafe URL Construction Identified]
          |
          v
[Local-File Scheme Accepted]
          |
          v
[Arbitrary Local File Disclosure]
          |
          v
[Flask Source Recovery]
          |
          v
[debug=True Confirmed]
          |
          v
[Application and Host Metadata Recovery]
          |
          v
[Werkzeug Version-Specific PIN Reconstruction]
          |
          v
[Debugger PIN Authentication]
          |
          v
[Trusted Session State Preserved]
          |
          v
[Python Code Execution]
          |
          v
[Root-Level Execution Context]
          |
          v
[Application Directory Enumeration]
          |
          v
[Final Challenge Artifact Discovery]
```

### Attack-Chain Analysis

The initial download vulnerability is the enabling weakness. It exposes local resources that reveal the application implementation and metadata required for later stages.

The debug configuration then introduces a separate high-impact weakness. The debugger provides a path to code execution, while recoverable PIN inputs reduce the effectiveness of its authentication barrier.

Finally, the root execution context amplifies the compromise and allows direct enumeration of application files.

The result is a chain in which several weaknesses reinforce one another.

---

## Security Findings and Severity

The following severity ratings are qualitative assessments based on the behavior described in the supplied challenge material. They are not formal CVSS scores and should not be interpreted as independently validated production-system ratings.

| ID | Finding | Qualitative Severity | Primary Impact |
|---|---|---|---|
| F-01 | Client-controlled server-side destination | High | Allows influence over backend resource retrieval |
| F-02 | Local file disclosure through `file://` | Critical | Enables access to local files readable by the application |
| F-03 | Debug traceback information disclosure | High | Reveals source paths and implementation details |
| F-04 | Exposed Flask/Werkzeug debugger | Critical | Exposes an interactive debugging interface with code-execution capability |
| F-05 | Reconstructable Werkzeug PIN | Critical | Weakens the debugger's authentication barrier |
| F-06 | Root-privileged application process | Critical | Amplifies debugger compromise into root-level execution |

### Severity Interpretation

- **Critical:** A weakness that directly enables or substantially contributes to full application compromise or privileged code execution in the demonstrated chain.
- **High:** A weakness that exposes sensitive implementation details or enables a meaningful step toward a more severe compromise.

In a professional production assessment, final ratings should account for exposure, exploitability, required privileges, affected assets, compensating controls, and verified business impact.

---

## Root Cause Analysis

### F-01: Client-Controlled Backend Destination

**Observed weakness:** The application constructs a backend URL by concatenating a client-controlled value with a fixed path and filename.

**Root cause:** Insufficient validation of the destination, scheme, and resulting resource reference.

**Security consequence:** The user can influence the backend resource retrieval process.

**Recommended remediation:** Parse and validate destinations using a strict allowlist and avoid constructing complete URLs through direct string concatenation.

### F-02: Local File Disclosure

**Observed weakness:** The backend retrieval mechanism accepts the local-file scheme, and URL construction can be manipulated so that an appended suffix does not change the selected local resource.

**Root cause:** Unsupported scheme acceptance combined with unsafe URL construction and fragment-handling assumptions.

**Security consequence:** Files readable by the application process may be exposed through the download endpoint.

**Recommended remediation:** Reject local-file schemes, restrict destinations, validate canonical resource references, and minimize filesystem permissions.

### F-03: Debug Traceback Disclosure

**Observed weakness:** Invalid requests produce detailed tracebacks that reveal source-level implementation information.

**Root cause:** Development debugging behavior is exposed to external users.

**Security consequence:** Internal paths, source code details, and application logic become available for further attack analysis.

**Recommended remediation:** Disable debug mode in production and return generic errors while preserving detailed diagnostics in protected logs.

### F-04: Exposed Flask/Werkzeug Debugger

**Observed weakness:** The application runs with debugging enabled and exposes the debugger interface.

**Root cause:** Development configuration is used in an environment accessible to untrusted users.

**Security consequence:** An attacker who satisfies the debugger's authentication requirements may gain interactive code execution.

**Recommended remediation:** Disable the debugger, remove public access to diagnostic interfaces, and deploy through a production-appropriate WSGI configuration.

### F-05: Reconstructable Debugger PIN

**Observed weakness:** The PIN can be reconstructed from application and host metadata available through the local-file-read primitive.

**Root cause:** Debugger authentication inputs are recoverable after another vulnerability exposes the required metadata.

**Security consequence:** The PIN provides an inadequate barrier against an attacker who can access the debugger and retrieve its inputs.

**Recommended remediation:** Do not rely on the debugger PIN as a production security control. Remove the debugger from untrusted environments and remediate the underlying information-disclosure vulnerability.

### F-06: Root-Privileged Application Process

**Observed weakness:** The Flask process runs with root privileges.

**Root cause:** Inadequate operating-system privilege separation.

**Security consequence:** Application-level code execution becomes root-level execution, substantially increasing the potential impact.

**Recommended remediation:** Run the application under a dedicated unprivileged account, restrict capabilities, and enforce filesystem and container isolation.

---

# Remediation and Defensive Recommendations

## 1. Secure the Download Endpoint

The download endpoint is the initial entry point into the attack chain. Its input-handling behavior should be corrected before considering downstream defenses sufficient.

Recommended controls:

- Avoid accepting arbitrary complete URLs when a resource identifier would suffice.
- Restrict supported URL schemes to those required by the application.
- Use a strict allowlist of permitted destination hosts.
- Validate destination addresses and their resolved IP addresses.
- Prevent access to loopback, private, link-local, and other prohibited destinations where appropriate.
- Revalidate destinations after redirects.
- Use established URL-parsing libraries rather than custom string concatenation.
- Reject malformed, ambiguous, or unexpected URL structures.
- Apply connection and response-size limits.
- Restrict outbound network access at the infrastructure layer.

If the feature only needs to retrieve a fixed set of documents, use server-controlled resource mappings rather than arbitrary user-supplied URLs.

## 2. Prevent Local File Disclosure

- Reject the `file://` scheme when local filesystem access is not an explicit requirement.
- Separate public files from sensitive application resources.
- Validate filesystem paths against a narrowly defined allowlist.
- Avoid treating URL fragments as a reliable security boundary.
- Ensure path normalization and resource selection are performed consistently.
- Apply least-privilege filesystem permissions.
- Add automated tests for unexpected URL schemes and path-manipulation attempts.

The safest design is to avoid allowing untrusted input to determine arbitrary filesystem paths.

## 3. Disable Flask Debug Mode in Production

The interactive debugger should never be exposed to untrusted users.

Recommended controls:

- Set `debug=False` in production.
- Do not use the Flask development server as a production server.
- Deploy through an appropriately configured production WSGI server.
- Remove or restrict diagnostic endpoints.
- Return generic error responses to external clients.
- Keep detailed tracebacks in access-controlled logs.
- Verify the deployed configuration rather than relying solely on development settings.

Disabling the debugger removes a major escalation path, but it does not replace fixing the original local-file-disclosure vulnerability.

## 4. Harden Werkzeug and Application Dependencies

- Track Flask, Werkzeug, Python, and other runtime dependencies.
- Keep supported dependencies patched.
- Use reproducible dependency files and controlled deployment builds.
- Review release-specific security advisories.
- Avoid relying on undocumented implementation details for security guarantees.
- Revalidate deployment settings after dependency upgrades.

Library version information is useful evidence, but a version banner alone does not establish the presence or exploitability of a particular vulnerability.

## 5. Remove Sensitive Information from Error Responses

- Disable interactive traceback pages in production.
- Prevent exceptions from exposing filesystem paths and internal configuration.
- Return consistent external error messages.
- Store diagnostic information in protected application logs.
- Review logs for accidental disclosure of secrets.
- Rotate credentials or tokens if they have been exposed.

Error messages should support incident investigation without providing an attacker with a detailed map of the application's internals.

## 6. Strengthen Debugger and Diagnostic Access Controls

The preferred control is to remove interactive debugging from production environments.

Where diagnostic interfaces are required in controlled development environments:

- Keep them isolated from public networks.
- Restrict access to authorized operators.
- Use strong access controls and network segmentation.
- Avoid exposing host metadata through unrelated application vulnerabilities.
- Monitor access to diagnostic endpoints.
- Treat unexpected debugger access as a potential security incident.

A PIN should not be treated as sufficient protection for an interface capable of executing code.

## 7. Apply Least Privilege

The root execution context significantly increases the impact of the debugger compromise.

Recommended controls:

- Run web applications under dedicated, unprivileged service accounts.
- Restrict access to sensitive directories.
- Remove unnecessary Linux capabilities.
- Apply container security profiles where containers are used.
- Use read-only filesystem mounts where practical.
- Restrict access to host resources.
- Limit network egress to required destinations.
- Separate application processes from privileged administrative operations.

Least privilege does not prevent every application vulnerability, but it reduces the consequences when one is exploited.

## 8. Implement Defense in Depth

The attack chain demonstrates why a single security control is insufficient.

A resilient deployment should combine:

- Strict input validation.
- Safe URL and path handling.
- Network egress restrictions.
- Protected application source and configuration.
- Secure error handling.
- Disabled production debugging.
- Least-privilege execution.
- Dependency management.
- Centralized logging and monitoring.
- Security testing of complete attack paths.

The objective is to prevent the initial exploit while ensuring that additional weaknesses do not automatically result in privileged compromise.

---

## Lessons Learned

### 1. A Download Feature Can Become an SSRF Entry Point

Any application feature that retrieves resources on behalf of a user should be evaluated for control over the destination, scheme, redirects, and final resource selection.

A seemingly ordinary download operation may expose internal services or local resources if those boundaries are not enforced.

### 2. URL Parsing Must Be Consistent

The local-file disclosure stage demonstrates the importance of understanding how the application combines user input with application-controlled suffixes.

Security controls must operate on the parsed and validated resource reference, not merely on a raw input string.

### 3. Debug Tracebacks Are Valuable Evidence but Dangerous in Production

The detailed exception revealed how the application constructed its backend URL.

Although such information is valuable during authorized testing, production systems should avoid disclosing internal implementation details to external users.

### 4. Source Disclosure Changes the Nature of an Assessment

Once application source is available, testing can move from behavioral inference toward direct analysis of the actual implementation.

Source disclosure can expose security assumptions, file locations, route conditions, and configuration values that would otherwise require substantially more investigation.

### 5. Debugger PINs Are Not a Production Security Boundary

The PIN reconstruction stage demonstrates how authentication inputs can become recoverable when the host and application environment are exposed.

The correct defensive response is to disable the debugger in production, not to rely on the obscurity of its authentication mechanism.

### 6. Session State Must Be Verified Explicitly

Successful authentication and successful command execution are distinct events.

Preserving the appropriate session state is necessary to establish that later requests belong to the authenticated interaction. Testing should verify the actual outcome rather than infer success from a single response.

### 7. Process Privileges Determine the Impact of RCE

The application was running as root, so code execution immediately provided a privileged context.

Running the same application as an unprivileged user would not eliminate the RCE vulnerability, but it would reduce the potential consequences.

### 8. Enumerate After Establishing the Required Access

The randomized final filename made blind guessing inefficient.

Once code execution was available, directory enumeration provided a direct way to identify the artifact. The lesson is to select the next action based on the capabilities actually established during testing.

---

## Evidence and Publication Policy

This portfolio document is based on the supplied Plant Photographer challenge notes.

Evidence should demonstrate the relevant behavior without exposing unnecessary sensitive information.

### Evidence Standards

- Use screenshots captured from the actual authorized lab.
- Ensure each screenshot supports the finding associated with it.
- Redact challenge flags, PINs, credentials, and other answer values.
- Keep image paths consistent with the repository structure.
- Avoid presenting conceptual diagrams as screenshots of actual target activity.
- Do not claim that a finding has been independently verified beyond the evidence available.

### Suggested Evidence Checklist

- [ ] RustScan or Nmap service enumeration.
- [ ] Application route discovery.
- [ ] Download endpoint behavior.
- [ ] Debug traceback disclosure.
- [ ] Local file disclosure evidence.
- [ ] Flask source showing the relevant route and debug configuration.
- [ ] Werkzeug version or implementation evidence.
- [ ] Debugger authentication result with sensitive values redacted.
- [ ] Harmless expression evaluation.
- [ ] Process identity output demonstrating the execution context.
- [ ] Final challenge completion evidence, if available.

The supplied source material does not establish that every item in this checklist exists as a separate screenshot. Include only evidence that you actually captured.

### Public Flag Policy

Challenge answers are intentionally omitted.

```text
Administrator artifact → [REDACTED]
Final challenge answer  → [REDACTED]
Debugger PIN             → [REDACTED]
```

This keeps the repository focused on the technical work and defensive lessons rather than publishing challenge-specific secrets.

---

## Conclusion

Plant Photographer demonstrates how a vulnerable server-side download mechanism can become the starting point for a complete application compromise.

The investigation begins with service discovery and endpoint enumeration. It then establishes that the application accepts a client-controlled destination, exposes detailed debug tracebacks, and permits local-file access through its backend retrieval mechanism.

The resulting file disclosure reveals the Flask source and confirms that debugging is enabled. Host and application metadata obtained through the same primitive allow the assessment to investigate the version-specific Werkzeug PIN mechanism. After debugger authentication and session-state preservation, the application permits Python code execution in a process running as root.

The central security lesson is that **input validation, URL handling, error disclosure, debugger configuration, and operating-system privileges must be secured as a connected system**.

No single layer should be expected to compensate for weaknesses across all the others.

The assessment reinforces a repeatable methodology:

**Enumerate → Inspect → Validate → Disclose → Correlate → Authenticate → Execute → Remediate**

For defenders, the highest-value actions are to eliminate arbitrary resource retrieval, prevent local file disclosure, disable production debugging, restrict diagnostic interfaces, and run application processes with minimal privileges.

These controls address both the initial vulnerability and the conditions that allowed it to escalate into root-level execution.

---

## References and Tools

### Official Documentation

- [TryHackMe](https://tryhackme.com/) — Authorized cybersecurity learning environment.
- [Flask Documentation](https://flask.palletsprojects.com/) — Application configuration, deployment, and debugging guidance.
- [Werkzeug Documentation](https://werkzeug.palletsprojects.com/) — WSGI utilities and debugger behavior.
- [Python Documentation](https://docs.python.org/3/) — Python runtime and standard-library reference.
- [Nmap](https://nmap.org/) — Network discovery and service enumeration.
- [RustScan](https://github.com/RustScan/RustScan) — Fast TCP port discovery.
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
- Session-state analysis.
- Python remote code execution.
- Linux process privilege assessment.
- Application-directory enumeration.
- Root-cause analysis and remediation planning.

---

**Author:** Anurag Revankar  
**Platform:** TryHackMe  
**Room:** Plant Photographer  
**Purpose:** Authorized security research, technical documentation, and cybersecurity portfolio  
**Disclosure:** Challenge-specific flags, PINs, and answer values are intentionally redacted.

