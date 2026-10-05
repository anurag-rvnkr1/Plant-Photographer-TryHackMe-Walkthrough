# Remediation

## Critical

1. Disable Flask debug mode in production.
2. Remove or restrict the interactive debugger.
3. Block non-HTTP URL schemes in server-side fetchers.
4. Allowlist backend destinations.
5. Run the application as a dedicated non-root user.

## High

6. Prevent user-controlled URL concatenation.
7. Suppress detailed production tracebacks.
8. Protect source/configuration files.
9. Rotate any secret disclosed through debug output.
10. Implement egress network controls.
