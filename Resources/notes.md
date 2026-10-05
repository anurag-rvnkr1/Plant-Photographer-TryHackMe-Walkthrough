# 🌿 Plant Photographer — Technical Notes

## Core Attack Chain

```text
Werkzeug discovery
→ endpoint enumeration
→ SSRF-style download
→ file:// LFI
→ source disclosure
→ debug=True
→ host/application metadata recovery
→ Werkzeug PIN reconstruction
→ debugger authentication
→ Python RCE
→ root context
→ randomized file discovery
```

## Core Concepts

### SSRF / LFI Boundary
A server-side fetch destination must be strictly controlled. Permitting local-file schemes can convert SSRF into arbitrary file disclosure.

### Debug Tracebacks
Verbose Werkzeug errors disclose application source, paths and configuration.

### Debug PIN Reconstruction
Use the exact target Werkzeug implementation; library-version differences matter.

### Session State
Debugger authorization also depends on a trust cookie established after successful PIN authentication.

### Privilege Context
The Flask process runs as root, so debugger RCE already represents maximum privilege inside the environment.

## Public Flag Policy

`[FLAG REDACTED]`
